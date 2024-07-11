---
layout: single
title: 分布式锁的设计与实现(PART I)——Chubby
categories: [Distributed Systems]
toc: true
toc_label: "目录"
toc_icon: "cog"
---
本文的写作目的是将分布式锁这一重要的分布式基础组件的几种典型设计与实现「剔肤见骨」地展示给各位读者（老毛病了，开头先自吹自擂一下)。写作背景是笔者在过去一段时间从事过分布式锁的研发工作，期间调研过多篇分布式锁典型实现的论文，在这里挑出比较优秀的三篇介绍一下。

本文接下来的内容会围绕三篇论文涉及的三个系统展开，它们是两个为 coarse-grained locks 设计的系统：Chubby 和 Zookeeper，一个为 fine-grained locks 设计的系统：VAX/VMS DLM（DLM stands for Distributed lock manager。补充一下，据说 Oracle DLM 就是参考这个系统实现的）。

Q：coarse-grained 和 fine-grained locks 是什么鬼？

A：在本文的语境下，coarse-grained locks 指的是持有时间长，加锁频率低的锁，e.g., 在 primary-backup 系统中用于实现 primary election 的锁；fine-grained locks 恰恰与之相反，e.g., 数据库系统中的 relation lock，基于 shared disk 架构的系统中的 page lock。

# 2 Chubby

是 Google 在 2006 年公开的一个分布式锁系统。

## 2.1 Motivation

用来解决分布式系统中 coarse-grained synchronization 问题以及 low-volume 的 metadata 存储问题。

在 Google，需要 coarse-grained synchronization 的场景有：GFS 中的 master election，Map Reduce 中的 rendezvous（又叫 barrier/CountDownLatch)。需要 low-volume metadata 存储的场景有：GFS，BigTable（它们需要一个存放 group membership，ACL 等信息的地方)。

## 2.2 Methods

**2.2.1 Design Goals**

1. coarse-grained locking service，low-volume storage

   * 实现成一个 service，而不是 library，更解耦易用。
   * storage 用来存 metadata，不会有很大的 volume 需求。
2. 侧重于 reliability 和 availability，其次才考虑 throughput 和 storage volume

   * coarse-grained locks 在 failure 后丢失对 client 的影响会非常大，必须保证 reliable。
     * e.g., client application 用 Chubby 来实现 primary election，一旦锁丢失，就不得不重新发起 election，client application 还得做一遍费劲的 recovery 工作，这是一项耗时的巨大工程。
   * coarse-grained locks 的频率很低，这就导致
     * throughput 不会很高。
     * 短暂的 unavailable 带来的影响相对 fine-grained locks 更小，允许我们用一堆**普通**机器组成一个 lock servers 集群。（尽管 Chubby 侧重 available，也只是实现到 high available，而不是 always available。）
3. easy-to-understand

   * 笔者强烈赞成这一点，做过系统或者用过系统的人都懂得

**2.2.2 Architecture**

Chubby 支持多集群部署，单个集群叫 Chubby cell。Chubby cell 有自己的 name，client 指定 Chubby cell name，在 DNS lookup 时解析到具体的地址，然后去访问对应的 Chubby cell（下文 「2.2.3 Data Model」一节会介绍 Chubby cell name 如何指定）。

![1652599659546.png]({{ site.url }}{{ site.baseurl }}/assets/images/2022-05-15-dlm-all-in-one/1652599659546.png "System structure")

上图是一个典型 Chubby cell 的系统架构。cell 内部有 5 个 server，基于 Paxos 做 replication，用来实现 high available，其中一个 replica 是 master。 为了实现 Linearizability，Chubby 的所有读写操作都会走 master。Chubby client 是一堆进程。client applicaiton 通过 Chubby library 来和 Chubby replicas 进行 RPC 交互。由于读写操作只能走 master，client 在第一次访问 Chubby cell 时，会先根据 DNS 提供的 replicas 列表，向它们发送 master location 请求获取 master 的地址，然后和 master 建立连接，再进入到正常读写流程。

此外，Chubby 是支持 partitioning 的，一个 Chubby cell 内可以有多组 Paxos group，即多个 master，数据基于 hash 分布打散到各个 master 上（下文「Performance Stuff」一节会详细介绍 partitioning）。

**2.2.3 Data Model**

Chubby 的 data model 是一个极简版的文件系统，有 directory 和 file 的概念，二者亦统称为 **node**。

Chubby 不支持 move directory，因为跨 partition 的 move 不好做；不记录 directory modified time，不记录 node 的 last-access time，因为记了这些东西就不好做 client 端的 cache 了（下文「Performance Stuff」一节会详细介绍 client cache)。

每一个 file 会包含一段可用的空间（原文：Each file contains a sequence of un-interpreted bytes)，可用来存一些数据。

给一个具体的例子：

```
/ls/cellfoo/appfoo/filefoo
```

在这个例子中，`ls` 表示 lock service，固定前缀。`cellfoo` 就是上文提到的 Chubby cell name，会在 DNS lookup 时解析成 Chubby cell 的地址。`/appfoo/filefoo` 是这个 Chubby cell 中具体的一个 application（`appfoo`） 下的一个 file（`filefoo`），`filefoo` 中可以存一些 metadata。

Chubby 中 node 的类型有两种：ephemeral 和 permanent。ephemeral nodes 在  client 的 session 断开或过期（下文「2.2.4 API」一节会详细介绍 session）的情况下会被 master 自动清理，而 permanent nodes 必须由 client 显式地清理。（前者通常用于实现 pirmary election，挂掉的 primary 持有的锁会被自动释放，replica 就有机会拿到锁成为新的 primary。）

任意一个 node 都可以作为 read/write lock。（下文「2.2.4 API」一节会详细介绍锁的用法。）

此外，Chubby 自身也有一些 metadata，包括：

1. ACL 信息。记录每个 node 的 read/write/modify 的权限信息。这些信息会存在 Chubby cell 中一个 well-known 的目录下。
2. 4 个 64-bit 单调递增整数
   * Instance number。创建同名 node 时递增，用于区分时间上先后的同名 node（创建一个 node，删除它，然后再创建一个同名 node，这两个 node 是不一样的）。
   * Content generation number(file only)。文件内容更新时递增。用于对文件执行 CAS 操作，实现 optimistic locking 语义。
   * Lock generation number。加锁成功时递增。用于实现 sequencer 机制（下文「2.2.4 API」一节会详细介绍）。
   * ACL generation number。ACL 修改时递增。用途类似 Conntent generation number。
3. 64-bit file-content checksum。用于校验文件数据的正确性。

**2.2.4 API**

笔者为保留原（懒）汁（病）原（犯）味（了），直接从论文中把 API 介绍复制过来了：

- **Open()** opens a named file or directory to produce a handle, analogous to a UNIX file descriptor.
- **Close()** closes an open handle.
- **Poison()** causes outstanding and subsequent operations on the handle to fail without closing it;
- **GetContentsAndStat()** returns both the contents and meta-data of a file.
- **GetStat()** returns just the meta-data.
- **ReadDir()** returns the names and meta-data for the children of a directory.
- **SetContents()** writes the contents of a file.（可以基于上文提到的 conent generation number 来更新，实现 optimistic locking 语义）
- **SetACL()** performs a similar operation on the ACL names associated with the node
- **Delete()** deletes the node if it has no children.
- **Acquire(), TryAcquire(), Release()** acquire and release locks.
- **GetSequencer()** returns a sequencer.（下文「2.2.4 API」一节会详细介绍 sequencer)
- **SetSequencer()** associates a sequencer with a handle. Subsequent operations on the handle fail if the sequencer is no longer valid.
- **CheckSequencer()** checks whether a sequencer is valid.

我们可以看到，API 的用法思路大致和文件系统一样，通过 Open() 获得文件的 handle，然后调用其他 API 对这个 handle 进行操作，最后使用完毕对 handle 进行 Close()。

client 通过 Acquire()/TryAcquire() 针对 handle 进行加锁，通过 Release() 主动释放锁。注意 Chubby 中的锁是 advisory lock，不是 mandatory lock，因此即使对某一个 file 加了写锁，也不会禁止其他 client 读写这个 file。做成这样的一个重要原因是，Chubby 锁常见的使用场景不是来实现 Chubby file 的并发控制，而是实现 client application 侧资源的并发控制，做成 mandatory lock 必须改 client application 代码，这是做不到的。

------------------------------------------------------------------------

这里有一个分布式锁必须面对的**一个问题**：client 加锁成功后，挂掉/联系不上了怎么办？如何释放锁？谁来释放锁？

Chubby 引入 session 和 KeepAlive 机制来解决这个问题。

- session：client 连接上 master 后会建立一个 session（master 会在内存中记下来），client 在运行结束或者 session idle（没有 open handles 且一分钟内没发起过 RPC) 后关闭 session。只要 session 是有效的，client 持有的所有 handles，locks，以及 cache 的所有数据（下文「Performance Stuff」一节会详细介绍 client cache）都是有效的。此外，session 会关联一个 lease，master 保证在 lease 的有效期内不会清理这个 session。
- KeepAlive：实现上就是一个 RPC，client 通过发送 KeepAlive RPC 到 master 来续约 lease，维持 session 的有效性。KeepAlive RPC 的实现可以说**非常有意思**，master 收到这个 RPC 请求后不会立刻回复，而是会 block 住这个 RPC，直到 client 的 lease 快要过期才会回复。client 收到 KeepAlive RPC 的回复后，续约自己的 lease，然后立即发送下一个 KeepAlive RPC，一段时间后 client 的 lease 快到期前又会收到 master 的回复，再接再厉续约 lease。为什么 master 要 block 住 KeepAlive RPC 一段时间才回复？因为想要 master 利用 KeepAlive RPC 的回包顺带传递 event notification 和 cache invalidation 给 client，这样就能做到只允许 client 向 master 单向发起连接，防火墙可配置成单向策略提升网络安全性。由于 event notification 和 cache invalidation 必须及时传递，因此当需要传递这两种信息时，master 会提前回复 KeepAlive RPC，不会再 block 它直到 client 的 lease 快过期。

session + KeepAlive 这个方案实际上还存在问题：client A 的 session lease 过期，master 就会清理这个 session，并释放这个 session 持有的锁，接下来另外一个 client B 就有机会拿到这把锁（说白了，就是要保证锁的 liveness property）。如果 client A 确实挂掉了，这样做没啥问题。但是，如果 client A 实际还活着（可能由于 GC 或者进程暂停导致一段时间没有发送 KeepAlive RPC），就会出现 client A 和 client B 同时拿到了锁的异象（锁的 safety property 被打破)。设想这样一个场景，这把锁是用来保护某个共享的资源服务器不会被并发操作的，结果 client A 和 B 都拿到了锁，都去操作资源服务器，很大可能导致资源服务器的数据被写坏。

Chubby 为解决这个问题引入了两种解决方案：

1. sequencer（思路同 DDIA 中介绍的 fencing token 方案)。sequencer 是一个描述锁状态的对象，状态包括锁的 name，mode，以及 lock generation number。client 可以通过 GetSequencer() 获取一个锁的 sequencer，去资源服务器操作的时候带上这个 sequencer，资源服务器当且仅当 sequencer 有效（lock generation number 是最新的）的情况下才会接受这个请求，否则直接拒绝。资源服务器如何确认 sequencer 的有效性？有两种方式：

   * a. 维护一个 sequencer cache，记录自己见到过的最大的 sequencer。当一个请求中携带的 sequencer（的 lock generation number） 比 cache 中 sequencer 的小时，认为该 sequencer 无效。
   * b. 直接调用 Chubby 提供的 CheckSequencer() 向 master 询问 sequencer 的有效性。

   针对上面的问题，如果资源服务器收到了 client B 的请求，然后又收到了 client A 的请求，client A 的请求会被直接拒绝。（实际上这个方案还存在问题，下文「My Analysis」一节会解释。）
2. lock-delay。尽管 sequencer 很好用，但需要改资源服务器的代码。Chubby 又提供了一个简单的 lock-delay 方案，即 master 检测到 session lease 过期，会再等 lock-delay 时长才释放过期 session 的锁，可以减少上面这个问题发生的概率。

---

Chubby 的 API 都支持同步和异步两种调用方式，可以通过 API 参数指定。如果是异步调用，client 会告知 master 自己关注哪些 event，master 会在这些 event 触发后通知 client（通过提前回复 KeepAlive RPC）。

Chubby 支持的 event 主要分为两类：

1. session event: Chubby client library 产生的 event（下文「2.2.5 Reliability & Availability Stuff」会详细介绍）。有以下几种：
   * jeopardy：client 的 session lease 过期后，grace period 开始时。
   * safe：session 从 grace period 幸存了下来（换句话说，KeepAlive RPC 联系上了 master 或者新 master)。
   * expired：与 safe 相反，session 没有从 grace period 幸存下来。
2. handle event: Chubby master 产生的 event。有以下几种：
   * file contents modified：用来监控文件更新。
   * child node added/removed/modified：用来监控子 node 变化。
   * Chubby master failed over：用来警告 client， event 可能丢失了，需要重新扫描数据。
   * a handle (and its lock) has become invalid：用来告知 client 发生了连接问题。
   * lock acquired：用来告知 client 某个 node 被加上锁了。很少使用，一般都用 file contents modified 而不是这个。
   * conflicting lock request from another client：用来支持锁的 cache。client A 加锁后，可以 cache 住锁一直用不释放，直到有其他 client 想要这把锁，这个 event 会被发送给 client A，client A 收到通知后尽快释放锁。很有意思的功能，非常适合锁冲突少的场景，不过很少使用。

最后举个用 Chubby 来实现 primary election 的例子来更加具体的说明下 API 的用法，parimary election 的主要流程如下：

1. 所有 replica 先尝试 open lock file（Open()），并加锁（TryAcquire()）。唯一一个成功加锁的成为 primary。
2. primary 将自己的信息写入 lock file（SetContent()）。
   * 其他的 replica 读取 lock file 获悉 primary 信息（GetContentAndStat()），也可以通过 file-modification event 做到这一点。
3. （可选）primary 获取一个 sequencer（GetSequencer()），primary 会将 sequencer 传给它访问的某个资源服务器。
4. （可选）资源服务器的 server 检查 sequencer 的有效性（CheckSequencer()）。

**2.2.5 Reliability & Availability Stuff**

Chubby 侧重于 reliability 和 availability，上文已经提到过的 replication 就是一个提升 reliability 和 availability 的有效手段。这里我们再看一下，Chubby 如何通过的优雅（grace）的 fail-over 机制来提升 reliability。

![1652673093622.png]({{ site.url }}{{ site.baseurl }}/assets/images/2022-05-15-dlm-all-in-one/1652673093622.png){: .align-center .width-half}

上图描述了当 master 发生 fail-over（从 old master 切到 new master）时发生了什么。时间线是从左到右的，我们也从左到右看。在 lease C1 的有效期内 client 发送 KeepAlive 请求（1）到 old master，old master block 住这个请求直到 client lease 快要过期才回复（2），同时 old master 视角的 client lease（这个 lease 是从回复 KeepAlive RPC 的时刻算起的，考虑到 RPC 传输的时间，实际会大于真实的 client lease 一点点） 变成了 M2，收到 KeepAlive 回复后，client lease 变成了 C2，紧接着 client 立即发起下一个 KeepAlive 请求（3）。不幸的是，old master 挂了，KeepAlive 请求很久没有得到回复，client lease C2 过期后进入 grace period（默认时长为 45s），同时会产生一个 jeopardy event（client application 可以观测到这个 event，可按需做一些处理逻辑）。在 grace period 期间 client 的所有请求都被 pending（相当于我们的静态*管理）。万幸，new master 在此期间选举成功了，new master 直接为 client 分配上它认为的 old master 可能为 client 分配的最大 lease（lease 时长是由 master 控制的，下文「 2.2.6 Performance Stuff」会详细介绍）。一段时间后，client 再次发起的 KeepAlive 请求（4）终于联系上了 new master。不过由于 epoch number （下面会介绍）错误，请求被拒绝并告知最新的 epoch number（5）。接着 client 拿着新的 epoch number 再次发起 KeepAlive 请求（6），收到回复（7）后，grace period 结束，产生一个 safe event（同样，client application 可按需处理。如果没有从 grace period 幸存下来，就会产生一个 expired event。）。这时 client lease 变成了 C3，然后立即发起 下一个KeepAlive 请求（8），后续就是正常流程了。

通过引入上面这个 grace period 机制，很大程度上减小了 master fail-over 对 client 影响，虽然搞了一通静态*管理，但是 client 没有报错啊！可以说很 reliable！

这里说一下上面提到的 epoch number，这个值其实就是 Paxos 里面的 round（aka ballot） number，每选举一轮递增一下。new master 的 epoch number 会大于 old master 的 epoch number。new master 通过拒绝小于自己的 epoch number 的请求，实现拒绝发送给 old master 的太老的请求（太老是多老？没能在 grace period 期间通过 KeepAlive RPC「上车」new master，拿不到新 epoch number 的请求都算）。

补充一下 handle 和 session 的 fail-over recovery：

1. handle 在 master 上不持久化，只记在内存里。master 检测到 client 用了一个 old master 的 handle（通过检测 client 请求中 handle 的 sequence number，笔者推测 sequence number 用的也是 epoch number)，就在自己的内存里面重建这个 handle；当这种**重建的** handle 被 close 时，也在内存里面记录一下，防止一个 delayed 或者 duplicated 的请求包不会意外地重建一个已经被 close 了的 handle。
2. session 原本的实现是持久化的（这也是为什么上面的 fail-over 图里面 new master 起来就能直接对 client 分配 lease，因为它是知道 session 信息的）。因为 master 持久化 session 很影响性能，Chubby 后来改了一版，改版后 session 和 handle 类似，也是只记在内存里面，通过 client 上报过来的信息来重建。在这种方式下，new master 需要等待一个 lease 确保所有 client 都「上车」，重建完它们的 session，才能对外提供服务。

**2.2.6 Performance Stuff**

Chubby 通过让读写请求都走 master 来实现 Lineariziablity，显然 master 干活太多，容易成为瓶颈。尽管 Chubby 更侧重 reliability 和 availability，这不妨碍用各种「骚操作」去优化 performance。

总结一下优化点，主要有：

1. Master lease。master 选举成功后会获得一个 lease，followers 保证在 lease 有效期的不会发起选举。读请求在 master lease 有效的情况下，可直接读 master，不需要走一遍 Paxos 流程。
2. Event-driven。event 的概念上文已经提及，利用 event-driver 机制可避免 Chubby client 轮询 master，减轻 master 的负担。
3. Chubby client cache。通过 client cache 进一步减轻 master 的负担。cache 的目标对象有：file content/node metadata(include file absense)/open handles/locks。Chubby client cache 是 strict consistency 的，写请求在真正写之前要**同步地**通知（通过提前返回 KeepAlive RPC）所有 client 做 invalidation。在确认所有 client invalidation 成功之前，将要写的目标对象会被标记为 uncachable，所有 client 会被暂时性地禁止 cache 这个对象直接读 master（这个方案会导致 master 负载增加。另一种可选方案，block 住所有 client 的读请求直到 cache invalidation 完成，这会带来读延时。当然也可以用混合方案，即先使用第一种方案，master 负载过高时自动切到第二个方案）。此外，出于性能考虑，master 只发送 invalidation 通知，不会推送更新后的数据。
4. KeepAlives/lease 自动调整。Chubby 的 coarse-grained locks 场景下，master 处理最多的 RPC 其实是 KeepAlive RPC。为了减轻处理 KeepAlive RPC 的负担，master 会根据负载动态扩展 lease 的时长（client 会跟随调整），减小 KeepAlive RPC 的频率。
5. Proxies。为进一步减轻 master 的负担，可以搞一堆 proxy server。proxy server 可以聚合 KeepAlive RPC，同时支持 read cache。
6. Protocol-conversion servers。用来减小 Chubby 作为 name service 时，对接使用 DNS protocol 的 client 的开销。client 使用 DNS protocol，Chubby 使用 Chubby protocol，要想对接，必须实现协议互转。互转的逻辑让 master  来做会带来额外的负担，可以专门搞一堆 protocol-conversion servers 来做。
7. Partitioning。用来解决一个 Chubby cell 在用尽了上面 1-6 的这些优化手段后还扛不住的问题。一个 Chubby cell 内可以搞多个 partition。每一个 node 按照其父 node 的 hash 值做 partition。大多数 client 操作只涉及一个 partition，性能很好。少数涉及多个 partition 的操作有：a. ACL 操作（只有 Open() 和 Delete() 操作涉及，不过 ACL files 有 cache)） b. 删除 directory，需要清空这个 directory 下的所有东西。
8. Mirroring。Chubby 支持把小文件 mirror 到各个 partition，可以降低读延迟。通过 event-driven 机制来解决 mirroring file 的更新同步问题。

## 2.4 My Analysis

笔者认为 Chubby 是**一个成功的系统**，针对分布式系统「coarse-grained synchronization，low-volume metadata storage」问题给出了实用的解决方案，它在 Big Table，Map Reduce，GFS 这三驾马车上都应用得不错也侧面证明了它的价值。

笔者看到的 Chubby 的**一个缺点**：资源利用率不够高，master 一个人在干活（尤其是以一己之力对抗所有 KeepAlive RPC，令人起敬），replicas 都在看戏。

笔者想到的 Chubby 的**一个优化点**：其实 locks 可以不持久化，只记在内存里，做成像 session 一样，fail-over 时让 client 上报过来，在内存里重建。如果搞了这个优化，Chubby 甚至有能力支持中轻度的 fine-grained locks，不过，这好像背离了 Chubby 的 design goals :)

笔者发现的 Chubby paper 中的**一个问题**：sequencer 其实不能彻底解决上文提到的 client A 和 B 同时拿到锁去操作资源服务器的问题。如果资源服务器先处理 client B 的请求，再处理 client A 的请求，由于 B 的 sequencer（中的 lock generation number）大于 A 的 sequencer，因此 client A 请求被拒绝，这种场景是没问题的。如果资源服务器先处理 client A 的请求，处理过程中又收到了 client B 的请求，client B 的请求不会被拒绝，client A 和 B 并发操作，照样导致资源服务器的数据被写坏。要解决这种情况，笔者能想到的解决方案，要么把资源服务器上的操作放在 critical section 中，保证不会并发，要么资源服务器收到 client B 的请求时，检查发现 client A 的请求还在处理，就等 client A 请求执行完毕或者把它直接 abort 掉，再处理 client B 的请求。

## 2.5 Take-away

笔者个人从 Chubby 这里学习到的系统设计小技巧都是关于 KeepAlive RPC 的骚操作的：

- KeepAlive RPC 不只可以用来续约 session lease，还可以用来 piggyback 其他信息（event notification && cache invalidation)。这样 client 与 master 可以实现单向发起连接。（单向发起连接这个特性让笔者感到莫名的舒服。)
- KeepAlive RPC(session lease) 可以根据负载自动调整，减小 master 的负担。

# 3 FBI Warning

警（敬）告一下读到此处的读者，非常抱歉，本文尚未写完，笔者后续会补上本文的 PART II 和 PART III 介绍一下 Zookeeper 和 VAX/VMS DLM。

在这里提前预告下 PART II 和 PART III 的内容：

**PART II**：介绍同样是为 coarse-grained locks 设计的分布式锁系统 Zookeeper，相较于 Chubby，Zookeeper 是一个更加通用的 coordination service，不止于分布式锁。此外，Zookeeper 采用了更弱的 consistency 解决了上文提到了「master 干活，replicas 看戏」的问题。详情见 PART II 为您分解。

**PART III**：介绍为 fine-grained locks 设计的分布式锁系统 VAX/VMS DLM，它的独特之处在于它是一个**纯内存**（volatile）的高性能分布式锁系统。高性能是多高？简直是丧心病狂，令人发指。详情见 PART III 为您分解。

本文首发于 [hhwyt](https://hhwyt.xyz/)，在知乎亦有发布，毫无疑问，来自**知乎点赞**的鼓励是笔者迅速码字的重要原因之一。

# 4 References

1. [[The Chubby lock service for loosely-coupled distributed systems](https://research.google.com/archive/chubby-osdi06.pdf)](https://www.google.com.hk/search?q=chubby+paper&oq=chubby+paper&aqs=chrome..69i57j0i512l3j46i175i199i512j0i512j46i175i199i512j0i512l3.1573j0j7&sourceid=chrome&ie=UTF-8#:~:text=The%20Chubby%20lock,%E2%80%BA%20chubby%2Dosdi06)



<br />

> 本文作者：hhwyt 
>
> 本文链接：[https://hhwyt.xyz/2022-05-15-dlm-all-in-one](https://hhwyt.xyz/2022-05-15-dlm-all-in-one)
>
> 版权声明：本博客所有原创文章均采用 [CC BY-NC-SA 4.0](https://creativecommons.org/licenses/by-nc-sa/4.0/) 许可协议。转载请注明出处！

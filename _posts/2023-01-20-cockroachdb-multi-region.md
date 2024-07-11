---
layout: single
title: 为什么 CockroachDB 的跨地域性能远超同行
categories: [Database]
toc: true
toc_label: "目录"
toc_icon: "cog"
---

之前有听说 CockroachDB 在跨地域部署（multi-region）方面的能力还不错，前段时间就花了几天时间研究了一下。主要是看了看 CockroachDB 在 SIGMOD 2022 上的一篇文章「Enabling the Next Generation of Multi-Region Applications with CockroachDB」以及他们的官方文档。看完之后，我的评价是：远超同行（哈哈，此处或有夸张；这里也要排除它开物理外挂的爸爸 Spanner）。本文就来趁热(我)打(没)铁(忘)介绍一下 CockroachDB 跨地域的设计。

在介绍 CockroachDB 的跨地域设计之前，我们先来看一下数据库跨地域的部署形态。跨地域的部署形态有很多种，可以根据客户的需求灵活变化。客户主要是两方面的需求：

1. （region 级别的）容灾。
2. 多活。

这两个方面是两个正交的维度，两个维度组合一下就会有非常多的可能性，对应不同的部署形态。

|              | 容灾能力低：region 失效丢失少量数据                          | 容灾能力中：region 失效不丢数据                              | 容灾能力高：region 失效仍然可用                              |
| ------------ | ------------------------------------------------------------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 多活能力：无 | 同城双中心（三副本）+ 异地异步复制只读副本                   | 两地三中心（五副本）                                         | 三地五中心（五副本）                                         |
| 多活能力：有 | 表数据按地域分区（geo-partitioned），每个分区的数据都同城双中心（三副本）+ 异地异步复制只读副本 | 表数据按地域分区（geo-partitioned），每个分区的数据都两地三中心（五副本） | 表数据按地域分区（geo-partitioned），每个分区的数据都三地五中心（五副本） |

注意：上面的 6 种部署形态只是例子，实践中可以组合出来更多的部署形态。

上面通过容灾和多活两个维度抽象出来部署形态是我的理解，CockroachDB 中对部署形态的抽象和我的抽象类似。在 CockroachDB 中，这两个维度叫做 survivability goals 和 table locality。CockroachDB 允许客户通过简单的 SQL 语法指定 database 的 survivability goals 和 table 的 table locality。示意如下：

```
ALTER DATABASE movr SURVIVE REGION FAILURE; 
ALTER DATABASE movr SURVIVE ZONE FAILURE;

CREATE TABLE west_coast_users ( ... ) LOCALITY REGIONAL BY TABLE IN "us-west1";
CREATE TABLE users ( ... ) LOCALITY REGIONAL BY ROW;
ALTER TABLE promo_codes SET LOCALITY GLOBAL;
```

调整 survivability goals 的语法比较简单，这里就不说了。调整 table localility 的语法，出现了三种 locality，这里解释一下：

1. regional by table：即这张表不做多活，表数据不做 geo-partitioned，只在一个地域写。
2. regional by row：即这张表做多活，表数据做 geo-partitioned，不同的地域写不同地域的数据。
3. global：与 regional by table 一样，不做多活，但允许所有 region 在本地的 follower 副本上低延时强一致读取。**这个是 CockroachDB 的全球独家黑科技**，下文会详细介绍。

如果客户事先已经在不同的 region 启动好了 CockroachDB 节点，通过执行上述 SQL 命令，就可以方便的调整数据库的容灾能力（database 级别）和多活能力（table 级别），生成不同的部署形态。（如此简单易用，有没有远超同行？)

说完了部署形态，我们终于要进入本文的重点了（好激动），接下来我们要揭开（脱掉） CockroachDB 跨地域高性能的面纱（底裤）。

众所周知，数据库跨地域性能最大的挑战就是物理学定律：网络传输的速度受光速限制。在这个定律下，数据库中跨地域的通信延迟必然大大增加，严重影响数据库性能。这个定律显然是打不过的，既然打不过，那我们就绕过。只要在数据库的关键链路，不产生或者尽量少产生跨地域通信，那性能就不会差啦。

我认为 CockroachDB 跨地域性能好，也是因为它在减少跨地域网络通信方便，做的事情还真不少，我们就来一个一个看一下到底做了啥。

**1. 时间戳采用 HLC** 

相对于全局 TSO 来说，这是一个大胆激进的方案（CockroachDB 家的技术方案一直很大胆，我对它们大胆采用 write snapshot isolation 实现且只实现 serializable 隔离级别印象深刻）。全局 TSO 方案实现很简单，但分配时间戳需要网络通信延迟，尤其是跨地域多活的部署形态下，如果所有地域的事务分配时间戳都要与一个特定地域的全局 TSO 通信，将会带来非常高的事务延迟。HLC 很好的解决了这个问题，虽然 HLC 会因为时钟不精确，在某个情况下需要重分配时间戳重试操作影响性能（CockroachDB 中的 Read within uncertainty window），不过这个情况也不是特别常见，对性能影响可以忍受。对于这个 trade-off， 我个人是非常赞同的。

**2. 全局唯一索引检查** 

CockroachDB 会针对全局唯一约束索引的特点，做到能不去所有地域都检查一遍是否违反约束就不去。如果客户定义的全局唯一索引符合以下几种情况，就会用上这个优化：

1. 全局唯一约束索引列是 UUID 类型。这种情况天然全局唯一，不需要去任何地域检查。
2. 全局唯一约束索引是多列组合，第一列是 crdb_region 列（crdb_region 列是隐藏列，用来描述一行数据所在的地域，由用户写入数据时指定，或者按写入所在地域自动生成）。这种情况只在 crdb_region 所在的地域检查是否违反全局唯一。
3. crdb_region 列是基于全局唯一索引列的 computed column。比如全局唯一索引列是身份证号，crdb_region 列是根据身份证号前缀算出来的 。这种情况也只在 crdb_region 对应的检查是否违反全局唯一。

**3. 尽量避免跨地域读数据** 

CockroachDB 支持 Locality Optimized Search 功能。查询优化器感知地域信息，支持 region local join，join 过程不需要跨地域通信。当查询只要求返回特定行数的结果时（如查找唯一列的一行或带 LIMIT 的查询），优先从本地查找，本地如果查够了就直接返回结果集。本地查不够，再去其他地域查找。由于本地读的延迟（几 ms）相对于跨地域读（几十~几百 ms）非常低，这么做的确是一个好办法！

**4. 不投票的 follower**

CockroachDB 允许在不同的地域放置不参与共识投票的 foloower 副本。这样既不影响写性能，又能让不同地域享受到低延迟的本地 follower read 功能，详情见下面的 5 和 6。

**5. Follower stale read** 

Follower 支持 Exact staleness read 和 Bounded staleness read 两种方式。

Leader 会维护一个 closed timestamp（类似于 Spanner 的 safe time），定期推动这个时间戳。leader 会禁止这个时间戳以下的写操作执行，但如果写操作是先于推动 closed timestamp 前写下去，不会禁止它对应的事务将来提交。Leader 通过 raft 日志以及周期性地通信双通道告知 follower 最新的 closed timestamp。

Exact staleness read 允许查询指定具体的时间点，如果这个时间点小于 closed timestamp，并且读到的数据不是 intent 数据（CockroachDB 中的 write intent，已写入存储但尚未提交的数据），那么就可以直接在 follower 上读。否则就要去 leader 读。

```
SELECT * FROM t AS OF SYSTEM TIME '2021-01-02 03:04:05' 
```

Bounded staleness read 允许查询指定一个宽松的时间点下界，系统自己计算出一个合适的时间戳，尽可能的做到 follower read，尽可能读新一点的数据，并保证这个合适的时间点不会低于这个下界。

```
SELECT * FROM t AS OF SYSTEM TIME with_min_timestamp('2021-01-02 03:04:05')
```

Bounded staleness read 具体的实现开销比较大，需要把事务的读集合先读出来，选取 min{读集合所有intent的写入时间戳, closed timestamp} 作为时间戳来读数据。时间戳确定好后，需要用这个时间戳重新读取一遍读集合。Bounded staleness read 的实现要求读集合需要能事先确定，因此应用场景有限，只支持单条语句的事务。

相对于 Exact staleness read，虽然 Bounded staleness read 开销更大，但由于 Bounded staleness read 指定的时间点更为宽松，往往更容易走 follower read 而不是 leader read，很多场景性能会更好一些。

**6. 独家黑科技：follower 低延时强一致读** 

CockroachDB 中的 GLOBAL table（上文提到的 table locality 为 reginal by global 表） 专为一些读多写少还不能做 geo-partitioned 的表设计（e.g., 一些全局配置表），支持在所有地域的 follower 上强一致读取，且不需要与 leader 有任何交互。

这个听起来很神奇，因为我们通常的认识是直接在 follower 上读数据是不能做到强一致的。如果非要做到强一致，就得用到 Raft 博士论文中提到的 Read Index 方案，然而该方案是需要与 leader 交互的。

那 CockroachDB 是怎么实现的呢？

CockroachDB 对于 GLOBAL table 的写事务，会分配一个未来的时间戳写。follower 上的读事务用当下时间戳读。

GLOBAL table 的 closed timestamp，会推进的更激进一些，leader 打个提前量，预估一个时间戳（考虑传播到 follower 的时延，时钟漂移等因素）发送给 follower（换句话说，closed timestamp 其实也是一个未来的时间戳）。

未来的时间戳与 closed timestamp 和当下现实世界的时间戳的关系是：

**未来时间戳 > closed timestamp > 当下时间戳**

所以，follower 上读数据，对于非 intent 数据（已提交的数据），可以直接读取，天然强一致。

对于 intent 数据，通过两个手段保证强一致：

1. 写事务提交会有 commit wait。写事务会等到现实世界的时间戳推进到自己用的未来时间戳，才能提交成功返回。保证了后续的读事务一定能读到已提交的写。
2. 读事务大部分情况下是非阻塞读。但如果发现写事务的提交时间戳落在了自己的 uncertainty interval 内（这个区间是 HLC 的时钟漂移区间，范围是：[自己的时间戳，自己认为集群中其他节点可能的最大时间戳，时间区间大概是数百 ms]），也需要 commit wait，而且还必须把自己的时间戳推到对方的时间戳并重试，保证能读到对方写的数据。这么做的原因，我这里先就不解释了，读者可以自行思考一下。

![]({{ site.url }}{{ site.baseurl }}/assets/images/2023-01-29-cockroachdb-multi-region/global-transaction.png){: .align-center .width-half}

总结一下， 不得不说，CockroachDB 的跨地域性能确实很强，通过各种花里胡哨的手段，尽量减少了跨地域通信的开销，跨地域能力可圈可点，值得我们学习！至于到底有没有远超同行，相信大家读了本文后心里已经有 B-树 了，也就不用我多说了。

## **参考**

1. https://www.cockroachlabs.com/pdf/SIGMOD2022.pdf
2. https://www.cockroachlabs.com/docs/stable/multiregion-overview.html
# 分布式事务解决方案：从了解到放弃
## 导语
***
让我们聊聊微服务的老大难：分布式事务。这是个已经被无数次讨论的问题，网上文章多如牛毛。本文从业务底层视角出发，探讨分布式事务究竟难在何处，以及务实的解决之路走向何方，再加一根牛毛... 不过希望本文是比较不一样的视角，能给到读者不同的启发。
在微服务架构流行的背景下，分布式事务的文章多如牛毛，虽然很多将事务一致性与副本一致性混为一谈，也仍不可否认其中相当一部分文章、开源代码，也还是不错的。

然而当你跃跃欲试，期待将业界所谓成熟方案落地，可能很快就会发现现实的骨感 —— 对于大量互联网业务，尤其是在大并发、大量使用 nosql 数据库的微服务架构下，很难落地。

## 一、问题简述
分布式事务，即在分布式环境下，对于同一个事务下的几件事情，要么一起完成，要么就和什么都没发生一样。也就是我们一般最关注的，是 ACID 中 的 A，因为拆分了服务之后，每一步远程 RPC 都可能故障。

例如张三给李四转账，张三的钱扣了，李四的钱加了，这两件事情，显然就应该一起完成，或者就都没做。

注意这里不要发散到 paxos、raft 等一致性协议上去，因为他们的领域本来就不同。

paxos 针对的是多个节点重复做相同的一件事情，主要处理的是多副本之间的一致性：
* 节点1完成任务1
* 节点2完成任务1
* 节点3完成任务1

分布式事务则显然不同，它针对的是有几个不同的任务，要捆绑一起完成或失败，重点在整体的原子性：
* 节点1完成任务1
* 节点1完成任务2
* 节点1完成任务3
二、分析探讨
1. 超时的难题
这里开门见山，让我们不妨抛开具体的分析过程，直击问题要害，那就是 —— 超时的处理。

明确的成功 / 失败都好说，但是超时呢，有可能成功了，也有可能没成功。在大量现实场景下，只有明确地知道 A 和 B 的真实执行情况，才能采取对应的后续措施，这点比较显而易见，此处不做展开。

那我们要怎么知道 A 和 B 的真实执行情况呢？“立字据” 成了必然的选择，也就是应该要能对账反查。

2. 对账反查能力
A 和 B 这两件事，是关联到一个事务的，这就要求，要有一个唯一事务 ID（这个 ID 还可以绑定一些其他附属信息），我们可以通过这个 ID 来做对账。如果能对账，那么幂等也会顺利成章就支持了。

唯一的事务 ID 生成很简单，如何通过这个 ID 来对账才是关键所在。

为了安全可靠，我们必须使用落地的信息来 “立字据”，ID 要体现在落地的数据中，这样我们就可以事后查询相关数据得知真正执行的情况，从而可以继续将事务推进下去。

接下来我们看不同情况下，落地信息具体如何记录。

3. 通过本地日志落地
采用进程本地打日志的方式落地，可以配合网盘或者可靠日志采集等方式做集中化收集处理，这种做法其实也只能从事务角度记录一些信息，至于业务数据本身是否明确修改 ok，在 DB timeout 时并不得而知，需要通过 WAL 日志冗余多记录一些东西（DB 修改前后的预期值都可以记录），这样无论 DB 操作成功与否，在一定的前提下，也能继续推进处理（例如以锁定玩家为前提，可以再将 db 数据查回来做核对，就知道是否操作 ok 了，更多讨论参考本章第 9 小节）。

4. SQL 存储的落地反查
SQL 存储通常可以基于 “本地事务” 来做反查，类似 seata 之类的解决方案，建个 undo log 表，操作业务数据库表 sql 的同时，会拦截绑定同库内 undo log 表的 sql 插入，一起作为一个本地事务来提交。也就是业务 sql 与 undo log 表的 sql，能通过本地事务进行关联，保证一起成功和失败，这样通过反查 undo log 表，即可得知超时的 sql 到底是否执行成功了。

开启本地事务
  add 业务sql：具体业务逻辑对应的 sql
  add undolog 表sql：insert 一条 undo log，key 包含事务 ID // 基于该表反查
提交本地事务
5. NoSQL 存储的落地反查
NoSQL 存储不一定有本地事务能力，如果支持本地事务的话，那和 sql 差不多，绑定批量操作做事务就好了（redis 有支持，如果对一致性要求不是很苛刻，或许凑合能用）。

如果不支持的话，这里提供两种对账的思路：

1). 通过落地日志的方式，将 ID 和操作结果体现到流水日志中，然后通过准实时的接口支持查日志来判断真正执行的情况。这种做法，侵入了存储访问的 API，写请求都带上 ID，且需要存储服务支持流水查询，自研的 kv 存储可以考虑该方案。

2). 直接将 ID 附着在存储记录里，例如 NoSQL 的 value 里面加一个事务 ID 数组（然后按大小滚动，条数上限够用就行），写操作会留下 ID 的 “痕迹”。这种做法，侵入了业务表结构，表本身要挂这么一个数组。用 proto 描述示例如下：

// 业务表示例
message ServiceTable{
  repeated string txids = 1; // 事务 id 数组，按一定 size 滚存
  int32 field1 = 2;
  int32 field2 = 3;
  int32 field3 = 4;
  int32 field4 = 5;
  // ...
}
上述这两种做法，显然都谈不上优雅，但这样也算是能建立基本的对账能力，尤其值得一提的是方案 2)，理论上适用于所有类型的 kv 存储。

6. 外部接口
如果分布式事务还关联了外部接口调用，那外部接口的超时，则需要依赖合作方提供对账、幂等能力。

7. 微服务典型复杂场景
对于典型复杂的微服务场景，这里举例说明：

事务开启
  任务1：改数据 A // sql 存储
  任务2：改数据 B // nosql 存储
  任务3：调用系统内服务接口 C // -- 被调服务可能是类似情况
  任务4：调用外部门服务接口 D
事务结束
假如我们通过一些手段，使得底层事务原子任务终于支持了对账反查能力，采用哪种应用层的事务方案，依然要三思。小规模业务或许本来对吞吐没啥要求，可以积极尝试。但大规模业务，请求量大，数据复杂，事务底层涉及到的底层数据资源一排列组合，情况就非常复杂了，例如：

// A B C D 四种数据
并行事务一：改 A B C
并行事务二：改 B D
并行事务三：改 B A
并行事务四：改 C D
不难想象，这里事务锁怎么加都是个问题，如果一把大锁，吞吐显然受限，如果细粒度锁，那该如何设计对应的粒度？所以对于大规模业务，可能得特事特办，例如仅仅是 cover 一下真的有强烈事务诉求的场景，其他场景下就直接有损处理。对应上述伪码，可能只有事务一继续是事务，其他三个就干脆不作为事务了。还有锁的时效性问题，以及是否存在并发锁，并发锁要不要和事务锁用同一把锁，都要考虑。

8. 为什么 DB 层能做而业务不好做
前面说了做好分布式事务的一些难处和前置条件，了解 NewSQL 的读者可能会有这样的疑问：TDSQL、TiDB 他们不都已经号称支持了跨库跨表的分布式事务了么，凭什么他们数据库系统能做，怎么我们业务层反而为难了，不是总说数据库才是高精尖么，他们不是应该更难么？是啊，凭什么？

这里的关键在于，数据库系统将跨表事务这个问题收拢到它内部了，关进了它自己的笼子里，然后它自己基于 XA 或其他协议，结合 MVCC 事务锁之类的机制，就可以解决好这个问题。

而在业务层面，潜在的参与方耦合面实在是太广了，无法简单收拢，同一个数据的变更触发源太多了，很难参考 DB 层的解决方案。可能有读者会认为，对于一个具体的数据表，最终将写入操作也只收拢到一个点，确实如此，但是我们关注点并不仅仅在这个点，而是在上层的事务。例如 SvrA 管理了数据表 A，但是 SvrB、SvrC、SvrD 都可以通过 RPC 接口来调用 SvrA 进行数据表 A 的修改，还可以存在级联的情况，例如 SvrE 先调用 RPC 访问 SvrB，然后再由 SvrB 发起 RPC 调用 SvrA 做了数据表 A 的修改。

SvrE ---|
       SvrB----\
       SvrC--- SvrA 数据表A
       SvrD----/
为了努力在业务层面解决事务问题，我们只好又祭出万能大法，加一层或多层抽象，这也正是目前大量上层应用解决方案做的：

增加外在的事务存储（主 key 为事务 ID，记录事务的参与方、进度、子事务等信息）。
要求事务参与方遵循某些约束（如本地事务 / 对账能力），调用特定的一些 API 将事务信息进行上报关联。
引入事务协调者，由它来根据参与方上报的信息做一些事务的驱动逻辑。
具体执行的话，可能还要考虑事务锁的具体实现策略（时效、粒度、与并发锁的关系等），于是问题就会变得更加复杂。

9. 那牺牲一点可用性呢
前面已经提到，存储层支持对账反查，是分布式事务得以应用的基本前提，那万一存储层就是不好支持对账呢，还能怎么办？

在做分布式系统设计的时候，有个 trick：一致性与可用性之间，有时是可以做置换的，当想要保障一致性的时候，实在没办法，可以想想是否能用一些可用性来置换，反之亦然。

那么我们在产生 timeout 类悬而未决异常的时候，可以牺牲一点可用性来加强事务的一致性保障么，粗想确实是可以的。例如 TCC 方案，try 已经都 ok 了，到了 confirm commit 阶段，有一个参与者其实并没有及时完成 commit 提交，它超时了。通过事务锁，限制事务所涉及用户的数据操作，在 try 阶段加好锁，直到这个悬而未决的事务完整执行完再释放即可。这里对用户加锁，本质上就是牺牲可用性的做法，解锁之前，其他写该用户数据的请求都处理不了。

1. try 阶段 lock 好
2. confirm commit... 直至确定 ok 为止
3. 明确完成后才 unlock
假如任由未 confirm 的数据继续提供写接口，则可能导致 confirm 时违反 try ok 的条件（例如字段 A 符合一定条件），那不锁定的话，在重试 confirm 的时候有可能字段 A 都已经被改写得不符合条件了，导致 confirm 实质上失败。

有些特殊场合下，还可以不加锁。例如对于金融转账而言，TCC 还有一种预留冻结金额的做法，就可以不加锁，但这显然和业务强相关，在很多业务场景下并不是一个简单的整数字段操作，常常无法 “预留资源” ，不具普适性。

简而言之，就是要对事务关联的用户，规定一个明确的执行序列（confirm、cancel 都可以），只要事务参与者活着，最终就一定会执行完毕。同时这里要遵循可用性约束，就是该事务关联用户在完成相关执行序列之前，不能放开别的请求修改对应的数据，否则可能会破坏数据一致性，即要锁住玩家直至完成事务。

我们再补充考虑下 crash-safe 以及 restart-safe，为了能可靠的按执行序列推进，显然还需要考虑 WAL 日志（如果重启有节点漂移，得用网盘），以及重启后基于 WAL 的逻辑检查和运行，而且这些其实都是很侵入代码的工作。此外如果事务框架还想做一些重试，事务协调者让事务参与者重试 commit，那相关数据的操作逻辑得有幂等能力。还需要指出的是，在分布式环境下，可能出现参与者没收到 try 就收到了 cancel（空回滚），或者收到了 cancel 之后才收到了 try（悬挂）的情况，也都需要处理，也就是无论收到 try 还是 cancel，都得先检查一下相关事务的本地执行情况。

综上来看，这样做确实不一定要求 DB 层支持对账，但必然要考虑锁、WAL、幂等之类的问题，因此想要正确严谨地实现事务也并非易事，换言之，做是能做的，但要考虑细致，然后因为比较侵入，代码复用性可能也不佳。如果不是很重要的业务场合，用上之后性价比不高，且任何机制叠加，还不可避免会带来一些新的依赖和异常，维护成本也会增加。

10. 特定场景下的简易实现
有些业务场景，对事务隔离性没什么要求，不必加锁，最终一致即可，对时效性也要求不高，同时也能从数据本身的最新值体现出操作是否已成功（例如设置 vip，查回来发现还没设置好就是没成功的），那么此时还是有相对简单的解决方案的。

这类场景下，恰巧是天然的可反查、有幂等，例如对于设置 vip 这步而言：

反查：直接查回来数据看下是否已经 vip 了，就知道执行的结果。
幂等：重新设置很多次 vip 效果也依然一样是个 vip。
具体细节此处就不再更多展开，无非就是结合反查持续重试直至成功即可。除了上面举例的场景，也还会有各种其他的业务场景，都是可能采用折中有损方案应对解决的，但因为太业务相关，不具普适意义，这里不再发散。

三、事务小结
1. 分布式事务的关键
要有个唯一的事务 ID，否则无法将各个子任务进行关联。
通常要基于事务 ID 实现幂等或对账，否则 timeout 没法正确处理，也无法安全重试。
一致性要求高的场景，会有对资源做锁定或预留的做法，最终一致性要求的场景，则只要最终符合预期即可。基于对资源要求的不同，会有一些常见的解决方案，例如多阶段协商提交、TCC、事务消息等。
现有的各种框架级别的解决方案，一般也就是：

创建事务，主 key 不必多说，肯定是事务 ID 了，然后关联一些事务信息的存储。
由协调者负责跟踪推进完整个事务。
各参与者遵从一定的规范约束，以幂等、对账能力为基础，实现相应的 API。
对于锁不锁，锁的粒度和时效性，是否能预留资源，则需要具体问题具体分析。这里提一种相对不侵入的解决方案 —— 异步对账补偿，参与者将事务信息汇报给协调者之后，就不再紧耦合，协调者可以自己独立旁路慢慢跑对账，然后按需 callback 回调业务进行补偿处理，当然其适用场景也是有限的。

2. 基于事务 ID 的幂等或对账能力
如果没有相关能力，盲目引进事务上层应用解决方案，只会让系统更加复杂，而并没有解决实际问题。

如果是 nosql 存储，可在 kv 的 value 结构中增加事务 ID 数组留痕来实现对账。
如果是 sql 存储，就要看 sql 服务方有没有提供基本的本地事务能力，如果是大规模 sql 存储，还要确定好 sharding 策略。
如果是外部接口，那就得外部接口支持相关能力。
如果真的明确存储层无法对账，还是想基于 TCC 之类的方式去做，那务必小心谨慎，具体细节已于前一章的第 9 小节阐释过。

3. 使用分布式事务的前置工作
明确各个任务原子的幂等、对账能力。
明确需要采用怎样的事务锁机制（还要同时考虑并发、吞吐等要素）。
明确业务本身对一致性的要求程度，是否可接受有损实现，进程优雅退出、故障重启是否要考虑残留事务的处理。
充分选型，优先考虑低侵入性的解决方案。
4. 结论
虽然前文花了很大的篇幅，论述了分布式事务处理的复杂性 —— 有时着实复杂到让人想要放弃，或许考虑做点异步对账补偿就是最经济实惠的解决方案。但如果业务场景真的需要，大家还是从实际出发，该用就用。

那些广为流传的方案能不能用呢，答案是看情况，只要场景匹配就行，但是在使用的时候，一定要清楚其背后的机制，明确其可能对吞吐带来的影响，并遵循好必要的约定或规范。至于是否必要引入第三方的事务框架，也要具体情况具体分析，只要评估引入后确有价值，同时成本也能承受即可。

总而言之，具体业务能不能使用分布式事务，以及使用什么样的解决方案，可以参考本文做充分的分析，实事求是，因地制宜。而好不好做（问题本身的难度复杂度），需不需要做（业务本身的核心诉求），想不想做（技术人员的主观意愿），始终是几个不同层面的问题。

四、参考资料
seata: https://seata.io/zh-cn/

https://seata.io/zh-cn/blog/seata-at-tcc-saga.html

TIDB事务：https://pingcap.com/zh/blog/tikv-source-code-reading-12

Google Percolator：https://research.google/pubs/pub36726/

https://www.bilibili.com/video/BV1FJ411A7mV

https://km.woa.com/group/51744/articles/show/498269

公司关于分布式事务的 OTEAM

更多资料可直接 google，km 上也有很多文章

~~~~~~~~~~~

最近在和公司外同行交流的时候，讨论起分布式事务这个话题，笔者发表了以自身业务视角切入的一些观点，有一些争论探讨，后仍觉意犹未尽，又与公司内同学做了一些探讨。讨论过后，越发觉得有必要再完善一下本文，遂抽时间做了内容的大幅调整，希望能给读者以拨云见日之感。

一家之言，欢迎讨论 ~

广告区，这里再介绍几篇其他笔者自己觉得还不错的文章：

ServiceMesh 服务网格和 zero-based thinking
跨学科通识好书推荐：《控制论与科学方法论》
关于技术债务的一点思考（参加腾讯学堂技术债务讨论有感）
从golang goroutine GMP调度的作者视角探究其设计之道
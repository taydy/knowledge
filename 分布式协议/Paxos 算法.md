# Paxos 算法

Paxos 算法是 Lamport 宗师提出的一种基于消息传递的分布式一致性算法，该算法包含两个部分：

- **Basic Paxos 算法** 其描述的是多节点之间如何就某个值达成共识；
- **Multi-Paxos 思想** 其描述的是执行多个 Basic Paxos 实例，就一系列值达成共识。

## Basic Paxos 算法

Paxos 将系统中的角色分为**提议者(Proposer)**、**决策者(Acceptor)** 和 **最终决策学习者(Learner)**。

- **Proposer** 提出提案(Proposal)，Proposal 信息中包含提案编号(Proposal ID)和提议的值(Value)；
- **Acceptor** 参与决策，并回应 Proposers 的提案。收到 Proposal 后可以接受提案，若 Proposal 获得多数 Acceptor 的接受，则称该 Proposal 被批准；
- **Learner** 不参与决策，从 Proposers/Acceptors 学习最新达成的一致的提案。

> 其实，这三种角色，在本质上代表的是三种功能：
>
> - 提议者代表的是接入和协调功能，收到客户端请求后，发起二阶段提交，进行共识协商；
> - 决策者代表投票协商和存储数据，对提案的值进行投票，并接受达成共识的值，存储保存；
> - 学习者代表存储数据，不参与共识协商，只接受达成共识的值，存储保存。

在多副本状态机中，每个副本同时具有Proposer、Acceptor、Learner三种角色。

![](https://tva1.sinaimg.cn/large/0082zybpgy1gc8k5lrv5nj30be075mx9.jpg)

Paxos算法通过一个决议分为两个阶段（Learn阶段之前决议已经形成）:

1. 第一阶段：**Prepare 阶段**。Proposer 向 Acceptors 发出 Prepare 请求， Acceptors 针对收到的 Prepare 请求进行 Promise 承诺。
2. 第二阶段：**Accept 阶段**。Proposer 收到多数 Acceptors 承诺的 Promise 后，向 Acceptors 发出 Propose 请求，Acceptors 针对收到的 Propose 请求进行 Accept 处理。
3. 第三阶段：**Learn 阶段**。Proposer 在收到多数 Acceptors 的 Accept 之后，标志着本次 Accept 成功，决议形成，将形成的决议发送给所有 Learners。

![](https://tva1.sinaimg.cn/large/0082zybpgy1gc8k9t3hovj30k00800t4.jpg)

Paxos算法流程中的每条消息描述如下：

- **Prepare** Proposer 生成全局唯一且递增的 Proposal ID (可使用时间戳加Server ID)，向所有 Acceptors 发送 Prepare 请求，这里无需携带提案内容，只携带 Proposal ID 即可。

- **Promise** Acceptors 收到 Prepare 请求后，做出“两个承诺，一个应答”。

  `两个承若`:

  1. 不再接受 Proposal ID 小于等于（注意：这里是 <= ）当前请求的 Prepare 请求。
  2. 不再接受 Proposal ID 小于（注意：这里是 < ）当前请求的 Propose 请求。

  `一个应答`：

  1. 在不违背以前作出的承诺的前提下，回复已经 Accept 过的提案中 Proposal ID 最大的那个提案的 Value 和 Proposal ID，没有则返回空值。

- **Propose** Proposer 收到多数 Acceptors 的 Promise 应答后，从应答中选择 Proposal ID 最大的提案的 Value，作为本次要发起的提案。如果所有应答的提案 Value 均为空值，则可以自己随意决定提案 Value。然后携带当前 Proposal ID，向所有 Acceptors 发送 Propose 请求。

- **Accept** Acceptor 收到 Propose 请求后，在不违背自己之前作出的承诺下，接受并持久化当前 Proposal ID 和提案 Value。

- **Learn** Proposer 收到多数 Acceptors 的 Accept 后，决议形成，将形成的决议发送给所有 Learners。

---

**示例：**

首先客户端 1、2 作为提议者，分别向所有接受者发送包含提案编号的准备请求：

![](https://tva1.sinaimg.cn/large/0082zybpgy1gc8kmxkm68j30vq0co42w.jpg)

**在准备请求中是不需要指定提议的值的，只需要携带提案编号就可以了**

接着，当节点 A、B 收到提案编号为 1 的准备请求，节点 C 收到提案编号为 5 的准备请求后，将进行这样的处理：

![](https://tva1.sinaimg.cn/large/0082zybpgy1gc8knv1q5yj30vq0cdwjn.jpg)

- 由于之前没有通过任何提案，所以节点 A、B 将返回一个 “尚无提案”的响应。也就是说节点 A 和 B 在告诉提议者，我之前没有通过任何提案呢，并承诺以后不再响应提案编号`小于等于` 1 的准备请求，不会通过编号`小于` 1 的提案。
- 节点 C 也是如此，它将返回一个 “尚无提案”的响应，并承诺以后不再响应提案编号`小于等于` 5 的准备请求，不会通过编号`小于` 5 的提案。

另外，当节点 A、B 收到提案编号为 5 的准备请求，和节点 C 收到提案编号为 1 的准备请求的时候，将进行这样的处理过程：

![](https://tva1.sinaimg.cn/large/0082zybpgy1gc8kpd1az3j30vq0c7djx.jpg)

- 当节点 A、B 收到提案编号为 5 的准备请求的时候，因为提案编号 5 大于它们之前响应的准备请求的提案编号 1，而且两个节点都没有通过任何提案，所以它将返回一个 “尚无提案”的响应，并承诺以后不再响应提案编号`小于等于` 5 的准备请求，不会通过编号`小于` 5 的提案。
- 当节点 C 收到提案编号为 1 的准备请求的时候，由于提案编号 1 小于它之前响应的准备请求的提案编号 5，所以`丢弃`该准备请求，不做响应。

客户端 1、2 在收到大多数节点的准备响应之后，会分别发送接受请求：

![](https://tva1.sinaimg.cn/large/0082zybpgy1gc8kr5m8stj30vq0c2n1h.jpg)

- 当客户端 1 收到大多数的接受者（节点 A、B）的准备响应后，根据响应中提案编号最大的提案的值，设置接受请求中的值。因为该值在来自节点 A、B 的准备响应中都为空（也就是图中的“尚无提案”），所以就把自己的提议值 3 作为提案的值，发送接受请求[1, 3]。
- 当客户端 2 收到大多数的接受者的准备响应后（节点 A、B 和节点 C），根据响应中提案编号最大的提案的值，来设置接受请求中的值。因为该值在来自节点 A、B、C 的准备响应中都为空（也就是图中的“尚无提案”），所以就把自己的提议值 7 作为提案的值，发送接受请求[5, 7]。

当三个节点收到 2 个客户端的接受请求时，会进行这样的处理：

![](https://tva1.sinaimg.cn/large/0082zybpgy1gc8ksh14sjj30vq0byadr.jpg)

- 当节点 A、B、C 收到接受请求[1, 3]的时候，由于提案的提案编号 1 小于三个节点承诺能通过的提案的最小提案编号 5，所以提案[1, 3]将被拒绝。
- 当节点 A、B、C 收到接受请求[5, 7]的时候，由于提案的提案编号 5 不小于三个节点承诺能通过的提案的最小提案编号 5，所以就通过提案[5, 7]，也就是接受了值 7，三个节点就 X 值为 7 达成了共识。

---

> Basic Paxos 是通过二阶段提交的方式来达成共识的。
>
> 除了共识，Basic Paxos 还实现了容错，在少于一半的节点出现故障时，集群也能工作。它不像分布式事务算法那样，必须要所有节点都同意后才提交操作，因为“所有节点都同意”这个原则，在出现节点故障的时候会导致整个集群不可用。也就是说，“大多数节点都同意”的原则，赋予了 Basic Paxos 容错的能力，让它能够容忍少于一半的节点的故障。



## Multi-Paxos 算法

Basic Paxos 只能就单个值（Value）达成共识，一旦遇到为一系列的值实现共识的时候，它就不管用了。虽然 Lamport 提到可以通过多次执行 Basic Paxos 实例（比如每接收到一个值时，就执行一次 Basic Paxos 算法）实现一系列值的共识。

但是 Lamport 并没有把 Multi-Paxos 讲清楚，只是介绍了大概的思想，缺少算法过程的细节和编程所必须的细节（比如缺少选举领导者的细节）。这也就导致每个人实现的 Multi-Paxos 都不一样。不过从本质上看，大家都是在 Lamport 提到的 Multi-Paxos 思想上补充细节，设计自己的 Multi-Paxos 算法，然后实现它（比如 Chubby 的 Multi-Paxos 实现、Raft 算法、ZAB 协议等）。

> Lamport 提到的 Multi-Paxos 是一种思想，不是算法。而 Multi-Paxos 算法是一个统称，它是指基于 Multi-Paxos 思想，通过多个 Basic Paxos 实例实现一系列值的共识的算法（比如 Chubby 的 Multi-Paxos 实现、Raft 算法等）

### 关于 Multi-Paxos 的思考

Basic Paxos 是通过二阶段提交来达成共识的。在第一阶段，也就是准备阶段，接收到大多数准备响应的提议者，才能发起接受请求进入第二阶段（也就是接受阶段）：

![](https://tva1.sinaimg.cn/large/0082zybpgy1gc8lb4uylrj30vq0hvq8x.jpg)

而如果我们直接通过多次执行 Basic Paxos 实例，来实现一系列值的共识，就会存在这样几个问题：

- 如果多个提议者同时提交提案，可能出现因为提案冲突，在准备阶段没有提议者接收到大多数准备响应，协商失败，需要重新协商。
- 两轮 RPC 通讯（准备阶段和接受阶段）往返消息多、耗性能、延迟大。

针对上述两个问题，我们可以通过`引入领导者`和`优化 Basic Paxos 执行`来解决。

### 引入领导者(Leader)

引入领导者节点后，使领导者节点作为唯一的提议者，这样就不存在多个提议者勇士提交提案的情况，也就不存在提案冲突的情况了。

![](https://tva1.sinaimg.cn/large/0082zybpgy1gc8lfzl1pvj30vq0i5jw3.jpg)

> **在论文中，Lamport 没有说如何选举领导者，需要我们在实现 Multi-Paxos 算法的时候自己实现。** 比如在 Chubby 中，主节点（也就是领导者节点）是通过执行 Basic Paxos 算法，进行投票选举产生的。

### 优化 Basic Paxos 执行

领导者节点上，序列中的命令是最新的，不再需要通过准备请求来发现之前被大多数节点通过的提案，领导者可以独立指定提案中的值。

因此当领导者处于稳定状态时，我们可以省掉准备阶段，直接进入接受阶段。

![](https://tva1.sinaimg.cn/large/0082zybpgy1gc8lk9n26lj30vq0hp0x8.jpg)



和重复执行 Basic Paxos 相比，Multi-Paxos 引入领导者节点之后，因为只有领导者节点一个提议者，只有它说了算，所以就不存在提案冲突。另外，当主节点处于稳定状态时，就省掉准备阶段，直接进入接受阶段，所以在很大程度上减少了往返的消息数，提升了性能，降低了延迟。

### Chubby 的 Multi-Paxos 实现

首先，Chubby 通过引入主节点，实现了 Lamport 提到的领导者（Leader）节点的特性。也就是说，主节点作为唯一提议者，这样就不存在多个提议者同时提交提案的情况，也就不存在提案冲突的情况了。

另外，在 Chubby 中，主节点是通过执行 Basic Paxos 算法，进行投票选举产生的，并且**在运行过程中，主节点会通过不断续租的方式来延长租期（Lease）**。比如在实际场景中，几天内都是同一个节点作为主节点。如果主节点故障了，那么其他的节点又会投票选举出新的主节点，也就是说主节点是一直存在的，而且是唯一的。

其次，在 Chubby 中实现了 Lamport 提到的，“当领导者处于稳定状态时，省掉准备阶段，直接进入接受阶段”这个优化机制。

最后，在 Chubby 中，实现了成员变更（Group membership），以此保证节点变更的时候集群的平稳运行。

**在 Chubby 中，为了实现了强一致性，读操作也只能在主节点上执行。** 也就是说，只要数据写入成功，之后所有的客户端读到的数据都是一致的。

- 所有的读请求和写请求都由主节点来处理。当主节点从客户端接收到写请求后，作为提议者，执行 Basic Paxos 实例，将数据发送给所有的节点，并且在大多数的服务器接受了这个写请求之后，再响应给客户端成功：、
- 当主节点接收到读请求后，处理就比较简单了，主节点只需要查询本地数据，然后返回给客户端就可以了。


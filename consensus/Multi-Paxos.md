# 5.3.4 Multi Paxos

:::tip 额外知识
lamport 提到的 Multi Paxos 是一种思想，所以 Multi Paxos 算法实际上是个统称，Multi Paxos 算法是指基于 Multi Paxos 思想，通过多个 Basic Paxos 实例实现的一系列值的共识算法。
:::

在分布式环境中，如果我们要让一个服务具有容错能力，那么最常用最直接的办法就是让一个服务的多个副本同时运行在不同的节点上。但是，当一个服务的多个副本都在运行的时候，我们如何保证它们的状态都是同步的呢，或者说，如果让客户端看起来无论请求发送到哪一个服务副本，最后都能得到相同的结果？实现这种同步方法就是所谓的状态机复制（State Machine Replication）。

:::tip 额外知识

状态机（State Machine）来源于数学领域，它被定义为一组状态和一个转换函数。状态机从一个初始状态，每次输入都要传入转换函数，使得状态机进入一个新的状态，在下一个输入来之前，保持状态不变。

状态机具有确定性，一个确定的输入只会转化到一个确定的状态。这个特性可以保证多个状态机从相同的初始状态开始，按照相同的顺序将一系列输入传给转换函数之后，都会达到一致的状态。
:::

分布式系统为了实现多副本状态机（Replicated state machine），常常需要一个多副本日志（Replicated log）系统，如果日志的内容和顺序都相同，多个进程从同一状态开始，并且以相同的顺序获得相同的输入，那么这些进程将会生成相同的输出，并且结束在相同的状态。


<div  align="center">
	<img src="../assets/Replicated-state-machine.webp" width = "600"  align=center />
	<p></p>
</div>

问题是如何保证日志数据在每台机器上都一样？当然是一直在讨论的 Paxos。一次独立的 Paxos 代表日志中的一条记录，重复运行 Paxos 即可创建一个 Replicated log。

首先，Replicated log 类似一个数组，我们需要知道当次请求是在写日志的第几位。因此，Multi-Paxos 做的第一个调整就是要添加一个日志的 index 参数到 Prepare 和 Accept 阶段，表示这轮 Paxos 正在决策哪一条日志记录。

现在流程大致如下，当收到客户端带有提案值的请求时：

- 找到第一个没有 chosen 的日志记录
- 运行 Basic Paxos，对这个 index 用客户端请求的提案值进行提案
- Prepare 是否返回 acceptedValue？
    - 是：用 acceptedValue 跑完这轮 Paxos，然后回到步骤 1 继续处理
    - 否：chosen 客户端提案值

我们举个例子，如图所示，首先，服务器上的每条日志记录可能存在三种状态：

- 已经保存并知道被 chosen 的日志记录，例如 S1 方框加粗的第 1、2、6 条记录（后面会介绍服务器如何知道这些记录已经被 chosen）
- 已经保存但不知道有没有被 chosen，例如 S1 第 3 条 cmp 命令。观察三台服务器上的日志，cmp 其实已经存在两台上达成了多数派，只是 S1 还不知道
- 空的记录，例如 S1 第 4、5 条记录，S1 在这个位置没有接受过值，但可能在其它服务器接受过：例如 S2 第 4 条接受了 sub，S3 第 5 条接受了 cmp

我们知道三台机可以容忍一台故障，为了更具体的分析，我们假设此时是 S3 宕机的情况。同时，这里的提案值是一条具体的命令。当 S1 收到客户端的请求命令 jmp 时，：

- 找到第一个没有 chosen 的日志记录：图示中是第 3 条 cmp 命令。
- 这时候 S1 会尝试让 jmp 作为第 3 条的 chosen 值，运行 Paxos。
- 因为 S1 的 Acceptor 已经接受了 cmp，所以在 Prepare 阶段会返回 cmp，接着用 cmp 作为提案值跑完这轮 Paxos，s2 也将接受 cmp 同时 S1 的 cmp 变为 chosen 状态，然后继续找下一个没有 chosen 的位置——也就是第 4 位。
- S2 的第 4 个位置接受了 sub，所以在 Prepare 阶段会返回 sub，S1 的第 4 位会 chosen sub，接着往下找。
- 第 5 位 S1 和 S2 都为空，不会返回 acceptedValue，所以第 5 个位置就确定为 jmp 命令的位置，运行 Paxos，并返回请求。


值得注意的是，这个系统是可以并行处理多个客户端请求，比如 S1 知道 3、4、5、7 这几个位置都是未 chosen 的，就直接把收到的 4 个命令并行尝试写到这四个位置。但是，如果是状态机要执行日志时，必须是按照日志顺序逐一输入，如果第 3 条没有被 chosen，即便第 4 条已经 chosen 了，状态机也不能执行第 4 条命令。


通过上面的流程可以看出，每个提案在最优情况下需要 2 个 RTT。当多个节点同时进行提议的时候，对于 index 的争抢会比较严重，会造成 Split Votes。为了解决 Split Votes，节点需要进行随机超时回退，这样写入延迟就会增加。针对这个问题，一般通过如下方案进行解决：

- 选择一个 Leader，任意时刻只有一个 Proposer，这样可以避免冲突，降低延迟
- 消除大部分 Prepare，只需要对整个 Log 进行一次 Prepare，后面大部分 Log 可以通过一次 Accept 被 chosen






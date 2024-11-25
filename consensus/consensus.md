# 6.1 什么是共识

受翻译影响，网上讨论 Paxos 或 Raft 的内容多使用“分布式一致性协议”或者“分布式一致性算法”这样的描述。如 Google Chubby 系统的作者 Mike Burrows[^1]，他对 Paxos 的评价原话是：“There is only one consensus protocol...”，很多文章翻译成“世界上只有一种一致性算法...”。

虽然汉语中的“共识”和“一致”是一个意思，但它们在计算机领域有明显的区别：
- 共识（consensus）：所有的节点就某一项提议（如选举、分布式锁、全局 ID、数据复制等等）达成一致的**过程及其算法**；
- 一致（consistency）：描述多个节点中存储的数据之间不自相矛盾，侧重于节点共识过程最终达成稳定状态的**结果**。

Paxos、Raft、ZAB 等等属于 consensus 算法，明显使用“共识”描述更准确，而 CAP 定理中的 C 和数据库 ACID 的 C 才是真正的“一致性” —— consistency 问题。

那么为什么要研究共识呢？这是因为在分布式系统中，为了消除单点风险，会使用一种名为“副本模式”的容错模型。副本容错模型的系统由多个节点组成，节点之间数据完全一致，可以保证在（≤ (N-1)/2）个节点故障的情况下，系统仍然能正常对外提供服务。

副本模型使用**复制状态机**实现不同节点的数据一致性，共识算法（Consensus Algorithm）是实现复制状态机最关键的角色。

:::tip 复制状态机
复制状态机（Replicated State Machine）是指多台机器具有完全相同的状态，运行完全相同的确定性状态机。它让多台机器协同工作犹如一个强化的组合，其中少数机器宕机不影响整体的可用性。
:::

如图 6-1 所示，使用复制状态机模型的工作流程：
1. 客户端发起一个写请求。
2. 共识模块执行共识算法进行日志复制，将日志复制至集群内各个节点。
3. 日志应用到状态机。
4. 服务端返回请求结果。

:::center
  ![](../assets/raft-state-machine.png) <br/>
  图 6-1 复制状态机工作过程 [图片来源](https://raft.github.io/raft.pdf)
:::

实现共识有哪些难点？搞清楚这个问题从 Leslie Lamport[^2] 发表的 Paxos 论文开始最合适不过。

[^1]: Mike Burrows 是 Google Chubby 的作者。
[^2]: Lamport 在分布式系统理论方面有非常多的成就，比如 Lamport 时钟、拜占庭将军问题、Paxos 算法等等。除了计算机领域之外，其他领域的无数科研工作者也要成天和 Lamport 开发的一套软件打交道，目前科研行业应用最广泛的论文排版系统 —— LaTeX (名字中的 La 就是指 Lamport)
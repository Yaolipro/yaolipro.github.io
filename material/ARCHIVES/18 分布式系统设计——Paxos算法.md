> 分布式一致性

### Paxos算法
* Prepare阶段
	- 争取提议权，争取到了提议权才能在Accept阶段发起提议，否则需要重新争取
	- 学习之前已经提议的值
* Accept阶段
	- 使提议形成多数派，提议一旦形成多数派则决议达成，可以开始学习达成的决议
	- 若被拒绝需要重新走Prepare阶段

#### Multi-Paxos
* Multi-Paxos选举一个Leader，提议由Leader发起，没有竞争，解决了活锁问题。提议都由Leader发起的情况下，Prepare阶段可以跳过，将两阶段变为一阶段，提高效率。Multi-Paxos并不假设唯一Leader，它允许多Leader并发提议，不影响安全性，极端情况下退化为Basic Paxos

### Raft算法
![img](assets/v2-5b683e2fa70a3a61e370be22ca492717_1440w.jpg)

* 不同于Paxos直接从分布式一致性问题出发推导出来，Raft则是从多副本状态机的角度提出，使用更强的假设来减少需要考虑的状态，使之变的易于理解和实现。
* Raft限制具有最新已提交的日志的节点才有资格成为Leader，Multi-Paxos无此限制。

### EPaxos算法


### 适用场景
* EPaxos更适用于跨AZ跨地域场景，对可用性要求极高的场景，Leader容易形成瓶颈的场景；Multi-Paxos和Raft本身非常相似，适用场景也类似，适用于内网场景，一般的高可用场景，Leader不容易形成瓶颈的场景
* Paxos、Raft等分布式一致性算法则可在一致性和可用性之间取得很好的平衡，在保证一定的可用性的同时，能够对外提供强一致性，因此Paxos、Raft等分布式一致性算法被广泛的用于管理副本的一致性，提供高可用性

### 思考
* Paxos的Proposal ID需要唯一吗，不唯一会影响正确性吗？
* Paxos如果不区分max Proposal ID和Accepted Proposal ID，合并成一个max Proposal ID，过滤Proposal ID小于等于max Proposal ID的Prepare请求和Accept请求，会影响正确性吗？
* Raft的PreVote有什么作用，是否一定需要PreVote？
* Raft的Leader必须在Quorum里面吗？如果不在会有什么影响？


















### 附录
* https://mp.weixin.qq.com/s/KSpsa1viYz9K_-DYYQkmKA
* 

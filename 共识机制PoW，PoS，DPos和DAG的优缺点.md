近期，随着区块链技术在社区中的声音越来越大，业界已经开始从技术角度对区块链进行全方位的解读。作为第一批区块链技术的实现，传统比特币与以太坊在共识机制、存储机制、智能合约机制、跨链通讯机制等领域并没有非常严密的设计，从而引发了一些在数据库与存储领域比较常见的问题，导致其数据规模无法无限增加（当前仅几百GB就产生了严重的性能瓶颈，几乎不可能到达上百TB规模），吞吐量极为有限，使其不可能适应通用分布式数据存储或通用结算体系的要求。

从产品功能的角度看，当前的区块链产品与数据库相比存在极大的差距。尤其是对于在业界存在了几十年的关系型数据库，其主要核心功能包括增删改查，而主要结构则包括SQL解析、日志、数据管理、以及索引管理几大模块。

而大数据技术兴起后，业界开始使用PC服务器替代传统小型机，为了避免服务器掉电导致的数据页损坏，分布式数据库或存储普遍使用三副本对数据进行冗余保存。

尽管从功能上看，当前区块链技术仅仅是数据库的一个微小子集，但是其一系列设计机制，与传统数据库的内核理念极为相似。譬如，从其传输和存储的数据结构上来看，区块链的链式结构来源于传统数据库的事务日志。任何数据库的DBA都知道，数据库的事务日志本质上就是不可更改的链式结构，事务中的每一条操作记录都会有一个反向指针指向该事务中的上一条记录。因此，区块链的链式结构本质上脱胎于数据库事务日志，同时增加了区块之间的反向哈希值作为指针，且引入了默克尔树结构进行快速数据校验。因而，我们可以安全地进行认为：区块链的链式结构在存储体系中等价于数据库的事务日志。本质上数据库的任何操作同样是不可篡改的，只不过当前大部分数据库不会对外暴露事务日志的解析工具，仅保存每一条记录的最终状态而已。

![image](https://raw.githubusercontent.com/FerminPan/Doc/master/static/4_11_1.jpeg)

（图1：数据库体系结构，黄色部分代表区块链同样包含的组件）

下面我们就从一致性原理出发，对分布式数据库与区块链技术进行对比。并在此基础上，详细聊聊当前区块链领域流行的几个共识算法各有什么优劣势。

## 一致性原理对比

在分布式数据库中，当前普遍采用PAXOS或RAFT算法进行数据多份冗余的一致性协商。一般来说，在分布式数据库体系中，每个数据分片由至少3个互相冗余备份的节点构成，而在正常运行时的数据库每个分片都会存在一个主节点与两个从节点。其中主节点负责数据的读写操作，从节点进行只读操作。当主节点写入数据时，其事务日志会被实时同步给其他从节点进行回放，以达到主从节点之间数据一致性的目标。

![image](https://raw.githubusercontent.com/FerminPan/Doc/master/static/4_11_2.jpeg)

（图2：数据库主从节点同步）

那么对比区块链的体系，可以认为数据库领域的主节点即日志生成节点，其每次生成事务日志的功能，与区块链中每次出块时矿工的功能完全等价。唯一不同的是，数据库在每次操作时对日志实时广播到从节点中，并且在事务提交时进行一致性判断。而区块链则采用检查点方式，每个节点接收自己的交易请求，并将请求广播到其他节点中，而每一次出块操作即产生一个检查点，该检查点包含的信息即出块节点向区块中写入的所有记录。这些记录被发送到其他节点后，每个节点对数据块中的记录进行验证并永久写入自身的交易日志（即区块文件）。

![image](https://raw.githubusercontent.com/FerminPan/Doc/master/static/4_11_3.jpeg)

（图3：区块链节点互相对等）

但是，区块链和数据库在一致性选择上最大的不同在于哪个节点成为检查点发起的节点。

数据库由于采用了主从机制，主节点永远是日志的发起节点，而从节点永远是日志回放与验证节点。

但是区块链则不同，其采用某些算法（例如PoW、PoS、DPoS等）在多个参与节点之间定期选取一个节点进行检查点确认，这也是区块链号称自身安全的一个理由所在：在全网大量的节点中攻击者无法确定下一个检查点确认的节点是谁（当然，就算攻击者确定了下一个出块节点，还有一系列的数字签名机制保障事务不被伪造和篡改）。


因此我们可以安全地认为，从检查点节点选择的领域来看，传统分布式数据库确定主节点生成事务日志的机制，是区块链共识机制的一种简单实现。也就是说，如果区块链共识机制每次都选取同一个节点作为出块节点，其机制基本等价于分布式数据库的主从复制原理（数据库按照事务提交进行一致性验证，区块链不存在事务的概念，因此按照数据块进行一致性验证）。

![image](https://raw.githubusercontent.com/FerminPan/Doc/master/static/4_11_4.jpeg)

（图4：数据库以提交回滚操作作为检查点，区块链以生成区块作为检查点）

对比了区块链的一致性原理，我们再详细看看当前区块链领域流行的几个共识算法以及对比。

## 区块链共识算法探析

由于区块链体系中并不存在某个节点永久作为检查点确认的节点，而是每个参与节点都有机会被选举成为该角色，因此在每个节点都能够进行读写操作时，整个区块链体系从功能上等价于一个不支持事务机制的多活数据库。而具体使用哪种算法选择出块节点（PoW与PoS之争）、哪些节点在接收到数据块时该如何验证（PoS与DPoS之争）、节点之间的数据以什么方式进行传播（DAG与链式结构之争）、以及如何确保一条交易被大多数参与节点所接受（PBFT、Paxos、RAFT、以及各种分叉解决方案等算法之争，Hyperledger 1.0甚至直接使用中央Kafka做排序也是醉了），则是区块链共识算法需要回答的问题。不同的解决方式制约着区块链的一致性、性能、吞吐量、以及可靠性。

### 1)  挖矿

挖矿是来自于比特币的一种说法，其本质在于多个节点通过PoW算法选举出一致性检查节点。关于PoW的说明业界已有无数文章分析，这里笔者不再赘述细节。实际上，从数据管理的角度来看，PoW是一种效率极为低下的暴力机制，通过不停地循环生成随机数并进行散列，通过网络预先广播的规则（复杂度），让每个参与的节点自证明其是否符合成为检查点的资格。

对比分布式数据库的Paxos或RAFT算法，每个参与节点默认自身有资格成为主节点，在原本的主节点无法连通的情况下通过最新事务号或其他原则相互投票，从而选举出新的主节点。而由于竞争节点过多，区块链作为一个拥有几万甚至几十万复制节点的多活数据库，继续采用Raft或Paxos算法一方面复杂度太高，另一方面无法解决拜占庭问题，因此比特币采用PoW机制，通过大家公认的某种机制，让每个参与节点首先自己判断是否符合要求（即生成了随机数后自己进行散列并验证）。当节点自身认为符合条件后，将之前生成的随机数以及打包好的日志（数据块）广播给集群中其他节点，从而大幅度减少了节点间相互投票所需的复杂度。

节点通过循环生成随机数并自我验证的过程，即PoW中所谓的“挖矿”阶段。

因此，如果把挖矿的概念扩展，不论是PoS、PoW或DPoS算法中，节点间竞争成为检查点的过程即挖矿过程。

### 2)  PoW与PoS的选择

PoW是一种极为粗暴原始，但却又及其有效防止恶意攻击的选举算法。该算法与计算机内核中多线程协作的自旋锁有异曲同工之处，自旋锁的原理在于通过线程自身不停循环判断一个内存地址状态，直到该状态设置为空闲后，通过CPU原子操作将其置为锁定状态，以此和其他线程进行互斥的机制。这种机制和PoW极为相似。

而PoS更倾向于类似Raft投票机制，通过固定时间协调所有节点参与投票，根据某种规则（例如持有代币数量、或提供存储空间大小等）判断每个节点的权重，最后选取权重最高的节点作为检查点节点。而在数据库一致性选择的Raft算法中，普遍会根据最新事务号作为权重，多个节点之间优先选择包含最新事务记录的节点作为主节点。

因此，可以看到PoW与PoS最大的区别在于，PoW在算法复杂度足够高的前提下，基本不需要太多的节点间互相通讯和确认，对代码的实现要求极低。而PoS对于多节点间一致性验证、防伪等要求较高，但是很大程度上可以沿用传统一致性选举的思路进行一定程度的优化即可。

![image](https://raw.githubusercontent.com/FerminPan/Doc/master/static/4_11_5.jpeg)

（图5：PoW与PoS流程对比）

但是PoW的缺点与自旋锁一样，对于计算资源的要求极高。一个被错误应用的自旋锁可以轻易消耗掉计算机中所有的CPU资源，同样PoW当前被人们诟病的最大问题也在于资源消耗。PoS在这方面则没有任何问题。

### 3)  PoS与DPoS的选择

类似Paxos与Raft，集群内参与的节点越多则效率越慢。一个典型的分布式数据库，使用单副本的效率可能会是三副本的两倍，而三副本的效率则又是七副本的两至三倍。因此，为了满足足够的吞吐量，使用PoS在进行选举时务必不能在成千上万个节点之间进行投票选举，而是应当在有限的集合范围内进行投票验证。这就是DPoS的核心原理。

DPoS给出一种思路，将成千上万个PoS节点，通过某种机制（例如持有代币的数量）选举出若干（奇数个）节点，在这几个节点之间进行投票选举（在一些实现中甚至会在这些节点间以令牌环的方式进行轮询，进一步减少投票开销）出每次的检查点（出块）节点，而不用在网络中全部节点之间进行选择。

这种机制能够大幅度提升选举效率。在几十个最多上百节点之间进行一致性投票一般来说可以在秒级完成并达到共识，因此DPoS机制可以将检查点（事务确认时间）提升到秒级，通过减少投票节点的数量或采用令牌环机制甚至可以降低到毫秒级。

![image](https://raw.githubusercontent.com/FerminPan/Doc/master/static/4_11_6.jpeg)

（图6：PoS对比DPoS）

但是，DPoS的性能无法无限提升。在一个完美的软件实现中，其性能与吞吐量则物理制约于节点间通讯的网络带宽。一般来说，对于公网环境中两个节点之间的带宽能够维持在上下行均5MB/s（50兆带宽）则相当优秀了，大部分情况下远远无法达到该数值。而如果每条交易日志需要100字节，由于网络即需要广播交易也需要广播日志，则网络带宽消耗加倍，因此在两个节点的单链中最大吞吐量不超过2.5万每秒（5MB/100字节/2=25000），假设集群中包含更多节点，则最大吞吐量需要根据其使用的P2P同步机制成比例缩减。如果需要进一步提升则需要进行分链（类似于数据库分片的概念），该主题将会在下面的章节详细讨论。

### 4）DAG与链式结构的选择

DAG与链式结构的本质区别在于异步与同步通讯。在前文中已经讨论过链式结构的本质等同于数据库事务日志，而出块操作则为检查点操作，那么链式结构体系可以看做是定期同步检查点的数据库事务同步机制。

而DAG则通过将事务操作进行异步处理来增加网络吞吐量，采用谣言传播算法在节点间发送操作日志，并通过某种机制（IOTA每次验证前两条交易，并计算一个PoW代表权重）将一个权重赋给该操作。

相比起同步操作的链式结构，DAG结构与任何异步机制一样，能够带来的提升在于吞吐量（真的么？后文会有描述），但是可能产生的问题则在于无法有效预测交易被确认的时间与周期，并且操作之间的顺序无法最终在多个节点间确认保持一致。

由于当前市面上DAG的实现相对较新，暂时还存在一些理论上未突破的局限性，包括：



1.    在对历史交易验证时采用随机方式，而没有任何先后规则，那么有可能产生某些交易在极端情况下没有任何其他节点对其验证，从而永远不会被确认。这个问题在IOTA中通过多次重试的方式解决，但是是否存在更好的一次性确认机制能更有效地解决该问题值得探讨；

2.    为了追踪每一笔交易与之前交易的关系，整个DAG图谱需要被随时检索和访问。在一个较大规模的系统中其交易图谱溯源会非常复杂，同时几乎不可能被全部保存在内存中以进行实时更新。而如果将这些数据保存在磁盘上，那么实时刷新每个Tangle的权重会造成大量随机I/O（也许可以通过大量部署SSD解决），因此从工程实现上来看优化难度较大；

3.    由于DAG的操作记录写入顺序不存在“区块”或“日志”这类检查点机制，因此每个节点各自为政，对于全局顺序无法得到保障。在这种情况下，在非等阶操作时可能存在不一致的问题（例如对同一条记录，在两个不同的节点同时执行转账（加减法）和计息（乘法）操作，两个节点得到的操作顺序有区别，导致账户余额最终结果不一致）。

如今从DAG衍生出一些其他数据结构（例如哈希树等），基本上只是从存储方式上有一些特定的优化，但是整体上与DAG所带来的问题保持一致。

笔者认为，DAG的异步数据分发思路完全可以与链式结构相辅相成。在最终理论完善之前其应用场景应当被谨慎选择，避免过早将其直接应用于通用化范式的场景。

## 结论

在区块链的共识机制中，其本质与分布式数据库的一致性算法存在极多的相似之处。拜占庭问题的引入仅仅从算法和选举节点数量上对网络结构做出一些调整，但是并不从本质上改变分布式系统一致性选举的机制。

![image](https://raw.githubusercontent.com/FerminPan/Doc/master/static/4_11_7.jpeg)

（图7：区块链共识机制对比）

PoW采用简单粗暴但极为有效的方式，通过节点首先自证其资质后才进行广播的方式，大幅度减少了网络间的通讯压力，但与之带来的问题则在于自证资质的计算资源消耗极大。

PoS采用与传统分布式一致性验证类似的机制，通过代币数量（或存储容量等指标）作为权重依据，使用某种分布式算法选举出每次的检查点节点。这种机制的好处在于没有消耗计算资源的自证资质过程，但是带来的问题在于每次选举时在大量节点的网络中对网络压力极大。



DPoS作为PoS的变形，通过缩小选举节点的数量以减少网络压力，是一种典型的分治策略：将所有节点分为领导者与跟随者，只有领导者之间达成共识后才会通知跟随者。该机制能够在不增加计算资源的前提下有效减少网络压力，在优秀的软件实现中将会具有较强的应用价值。

DAG则采用异步机制替代链式检查点的同步策略，但是由于其核心不存在一个标准的一致性确认机制（即账本或日志体系），同时无法对操作顺序进行全局统一排序，因此短期看来理论基础还有待突破。但是，从长期看来，DAG是一种非常新颖且有前景的机制，为传统数据管理领域的思维打开了新的大门。
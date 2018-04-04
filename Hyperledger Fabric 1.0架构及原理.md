# <center>Hyperledger Fabric 1.0架构及原理</center>

如果说以比特币为代表的货币区块链技术为 1.0，以以太坊为代表的合同区块链技术为 2.0，那么实现了完备的权限控制和安全保障的 Hyperledger 项目毫无疑问代表着区块链技术 3.0时代的到来。

Hyperledger 项目目前主要包括Fabric,Sawtooth Lake,Iroha,Blockchain-explorer四个子项目。下面我们来了解一下核心子项目Fabric最新版本是1.0的架构，原理及一个典型的交易过程，最后总结一下Fabric的优点。

 

# Fabric 1.0架构简介

![image](https://raw.githubusercontent.com/FerminPan/Static/master/doc_1.jpeg)


如上图所示，Fabric架构的核心包括三部分：Identity, Ledger及Transactions, Smart Contact.

# Identity
Identity，也就是身份管理，Fabric是目前为止在设计上最贴近联盟链思想的区块链。联盟链考虑到商业应用对安全、隐私、监管、审计、性能的需求，提高准入门槛，成员必须被许可才能加入网络。Fabric成员管理服务为整个区块链网络提供身份管理、隐私、保密和可审计的服务。成员管理服务通过公钥基础设施PKI和去中心化共识机制使得非许可的区块链变成许可制的区块链。

# Smart Contract
Fabric的智能合约smart contract称为链码chaincode，是一段代码，它处理网络成员所同意的业务逻辑。和以太坊相比，Fabric链码和底层账本是分开的，升级链码时并不需要迁移账本数据到新链码当中，真正实现了逻辑与数据的分离。

链码可采用Go、Java、Node.js语言编写。链码被编译成一个独立的应用程序，fabric用Docker容器来运行chaincode，里面的base镜像都是经过签名验证的安全镜像，包括OS层和开发chaincode的语言、runtime和SDK层。一旦chaincode容器被启动，它就会通过gRPC与启动这个chaincode的Peer节点连接。

# Ledger | Transcations
Fabric使用建立在HTTP/2上的P2P协议来管理分布式账本。采取可插拔的方式来根据具体需求来设置共识协议，比如PBFT，Raft，PoW和PoS等。

# Ledger

![image](https://raw.githubusercontent.com/FerminPan/Static/master/doc_2.jpeg)

如上图所示，账本Ledger主要包含两块：blockchain和state。blockchain就是一系列连在一起的block，用来记录历史交易。state对应账本的当前最新状态，它是一个key-value数据库，Fabric默认采用Level DB, 可以替换成其他的Key-value数据库，如Couch DB。举个例子。我们采用区块链实现一个弹珠交易的系统。我们开发了一个Chaincode,每个弹珠有以下几个属性：Name, owner, color, size.  可以定义一个JSON对象，用name做KEY, JSON对象做Value，存储在Level DB或者CouchDB中。

# transcation
Fabric上的transction交易分两种，部署交易和调用交易。

## 部署交易：

把Chaincode部署到peer节点上并准备好被调用，当一个部署交易成功执行时，Chaincode就被部署到各个peer节点上。好比把一个web service或者EJB部署到应用服务器上的不同实例上。

## 调用交易：

客户端应用程序通过Fabric提供的API调用先前已部署好的某个chaincode的某个函数执行交易，并相应地读取和写入KV数据库，返回是否成功或者失败。

# APIs, Events, SDKs
Fabric提供API方便应用开发，对服务端的ChainCode，目前支持用Go、Java或者Node.js开发。对客户端应用，Fabric目前提供Node.js和Java SDK。未来计划提供Python 和Go SDK，Fabric还提供RESTAPI。对于开发者，还可以通过CLI快速去测试chaincode，或者去查询交易状态。在区块链网络里，节点和chaincode会发送events来触发一些监听动作，方便与其他外部系统的集成。

# Fabric 1.0应用开发流程
如下图所示，开发者创建客户端应用和智能合约（chaincode），Chaincode被部署到区块链网络的Peer节点上面。通过chaincode来操作账本，当你调用一个交易transaction时，你实际上是在调用Chaincode中的一个函数方法，它实现业务逻辑，并对账本进行get, put, delete操作。客户端应用提供用户交互界面，并提交交易到区块链网络上。

 

![image](https://raw.githubusercontent.com/FerminPan/Static/master/doc_3.jpeg)

 

# Fabric 1.0业务网络
业务网络，也叫共识网络或区块链网络，由不同的节点构成。节点是区块链的通信实体，节点是一个逻辑概念，不同类型的节点可以运行在同一台物理服务器上。这些节点可能部署在云上面或者本地。可能来自不同的公司或者组织。在区块链网络中有两种类型的节点：Peer节点和Orderer节点，如下图所示。

![image](https://raw.githubusercontent.com/FerminPan/Static/master/doc_4.jpeg)

Peer节点：chaincode部署在Peer节点上，它对账本进行读写操作。一个Peer节点可以充当多种角色，如背书者endorser,提交者committer。一个区块链网络中会有多个Peer节点。

Orderer节点：对交易进行排序，批量打包，生成区块，发给Peer节点。一个区块链网络中会有多个Orderer节点，它们共同提供排序服务。排序服务可以别实现为多种不同的方式，从一个中心化的服务（被用于开发和测试，如Solo）,到分布式协议（如Kafka）。

排序服务提供了通向客户端和Peer节点的共享通信通道。提供了包含交易的消息广播服务（broadcast和deliver）。客户端可以通过这个通道向所有的节点广播（broadcast）消息。通道可以向连接到该通道的节点投递(deliver)消息。

排序服务支持多通道，类似于发布/订阅消息系统中的主题topic。客户端和Peer节点可以连接到一个给点的通道，并通过给定的通道发送和接收消息。多通道使得Peer节点可以基于应用访问控制策略来订阅任意数量的通道;也就是说，应用程序在指定Peer节点的子集中架设通道。这些peer组成提交到该通道交易的相关者集合，而且只有这些peer可以接收包含相关交易的区块，与其他交易完全隔离，实现数据隔离和保密。

此外，peers的子集将这些私有块提交到不同的账本上，允许它们保护这些私有交易，与其他peers子集的账本隔离开来。应用程序根据业务逻辑决定将交易发送到1个或多个通道。


![image](https://raw.githubusercontent.com/FerminPan/Static/master/doc_5.jpeg)
 

例如，如上图所示，peer 1,2和N订阅红色通道，并共同维护红色账本; peer 1和N订阅蓝色通道并维护蓝色账本;类似地，peer 2和peer N在黑色通道上并维护黑色账本。

在这个例子中，peer N在订阅了所有通道，我们看到每个通道都有一个相关的账本。一般来说，我们称不涉及所有peer的账本为子账本，另一种是系统账本，即全账本。

 

# Fabric 1.0交易流程
 

Fabric1.0一个典型的交易流程如下图所示：

![image](https://raw.githubusercontent.com/FerminPan/Static/master/doc_6.jpeg)

# 1.客户端构造交易提案
客户端应用程序利用任意SDK（Node.js，java，python）构造交易提案propose。该提案是一个调用智能合约功能函数的请求，用来确认哪些数据可以读取或写入账本。
客户端把交易提案发送给一个或多个Peer节点，交易提案中包含本次交易要调用的合约标识、合约方法和参数信息以及客户端签名等。

SDK将交易提案打包为可识别的格式（如gRPC上的protocolbuffer），并使用用户的加密凭证为该交易提案生成唯一的签名。

![image](https://raw.githubusercontent.com/FerminPan/Static/master/doc_7.jpeg)

# 2.背书节点模拟执行交易
背书节点endorser收到交易提案后，验证签名并确定提交者是否有权执行操作。背书节点将交易提案的参数作为输入，在当前状态KV数据库上执行交易，生成包含执行返回值、读操作集合和写操作集合的交易结果（此时不会更新账本），这些值的集合、背书节点的签名和背书结果（YES / NO）作为提案的结果返回给客户端SDK，SDK解析这些信息判断是否应用于后续的交易。

 

# 3.客户端把交易发送到共识服务
应用程序（SDK）验证背书节点签名，并比较各节点返回的提案结果，判断提案结果是否一致以及是否参照指定的背书策略执行。

客户端收到各个背书节点的应答后，打包到一起组成一个交易并签名，发送给Orderers。

# 4.共识排序，生成新区块，提交交易
Orderers对接收到的交易进行共识排序，然后按照区块生成策略，将一批交易打包到一起，生成新的区块，调用deliver API投递消息，发送给提交节点。

提交节点收到区块后，会对区块中的每笔交易进行校验，检查交易依赖的输入输出是否符合当前区块链的状态，完成后将区块追加到本地的区块链，并修改K-V状态数据库。

 

# Fabric 1.0优势总结
完备的权限控制和安全保障
成员必须被许可才能加入网络，通过证书，加密，签名等手段保证安全。通过多通道功能，保证只有参与交易的节点能访问到数据，其他的节点看不到。满足数据保护方面的法律法规要求。如有些行业，需要知道谁访问了特定的数据。

# 模块化设计，可插拔架构
如状态数据库可采用Level DB或者Couch DB，或其他的key-value数据库。

身份管理（identity management）可以采用自己的。共识机制和加密算法也是可插拔的，可以根据实际情况选择替换。

# 高性能，可扩展，较低的信任要求
Fabric采用模块化架构把交易处理划分为3个阶段：通过Chaincode进行分布式业务逻辑处理和协商（endorsers）；交易排序(orderders)；交易的验证和提交(committers)。这样划分带来的好处：不同的阶段由不同的节点（角色endorsers, orderders, committers）参与，不需要全网的节点都参与。网络的性能和扩展性得到优化。Peer节点和Orderder节点可以独立扩展，并可以动态增加。

因为只有endorsers和committers能真正交易的内容。只需要较低的信任要求就可以保证安全。

在不可更改的分布式账本上提供丰富的查询功能
可以在Level DB上进行按key查询，按复合KEY查询，按KEY的范围查询。如果采用Couch DB，Couch DB是文档数据库，数据是JSON格式的。除了支持按key查询，按复合KEY查询，按KEY的范围查询外，还支持全文搜索。


文章来源：书生老徐
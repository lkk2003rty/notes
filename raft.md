# raft 学习笔记

## 概览

raft 是一种分布式一致性算法，因此易于理解、实现而被广泛应用。之前一直久闻大名，而不知其细节，为了满足好奇心特地学习了下，做些笔记于此。

raft 从何学起呢？工欲善其事，必先利其器。当然首先看 paper 啦。主要的 paper 有两个 [raft](https://raft.github.io/raft.pdf) 和 [thesis](https://ramcloud.stanford.edu/~ongaro/thesis.pdf)。相关的资料站点为 [https://raft.github.io/](https://raft.github.io/)。要是嫌弃这个站点 raft 的可视化做得不带好，还有一个比较详细的 [http://thesecretlivesofdata.com/raft/](http://thesecretlivesofdata.com/raft/) 可以看。 可是光看 paper 的话毕竟还是赶脚有些距离，毕竟理论是理论，实现起来说不定还会遇到什么坑呢。所以呢，还是 RTFSC 吧。要看代码那就看个靠谱的，哪个靠谱呢？etcd 啊！在 etcd 中 raft 被实现成了一个包，那么只要看这个包的内部实现就行了。那么从哪里入手呢？稍微看了下 etcd 的代码，赶脚就是一脸蒙蔽，无从下手。但是别急，为了说明这包怎么用 etcd 的童鞋提供了一个 demo。这个 demo 其实就是一个简单的分布式 KV 数据库。这个 demo 位于 etcd 代码的 contrib/raftexample 目录下。这次就从这里入手。

### raftexample 主要代码速读

这里 main 函数比较简单就调用了 3 个函数 `newRaftNode`、`newKVStore`、`serveHttpKVAPI`。下面分开说说这三个函数。

`newRaftNode` 主要干了什么事呢？进去一看主要是生成了一个 `raftNode` 实例，然后启动一个 goroutine 调用该实例的 startRaft 方法。

`newKVStore` 主要实例化了一个 `kvstore` 实例，调用一次该实例的 readCommits 方法之后启动一个 goroutine 继续调用该实例的 readCommits 方法。结构体 `kvstore` 长下面这样。注意其中 `kvStore` 这个成员变量就是存储实际的键值对的。

```go
 // a key-value store backed by raft
 type kvstore struct {
     proposeC    chan<- string // channel for proposing updates
     mu          sync.RWMutex
     kvStore     map[string]string // current committed key-value pairs
     snapshotter *snap.Snapshotter
 }
```

`serveHttpKVAPI` 启动 http 对外提供服务。其中可以看到请求可以分为两类，一类是 kv 操作，一类是集群中节点的增加/删除。对于 kv 操作，GET 请求用于获取 key 对应的 value，这个简单直接从 `kvstore` 结构体中的 `kvStore` 成员变量中搂就是了；PUT 请求用于设置键值对，这个直接调用 `kvstore` 结构体的 Propose 方法就是了。而 Propose 只是将键值对编码下往 `kvstore` 结构体的 `proposeC` 扔就完事了。而对于节点操作更简单，拼装成一个消息往 confChangeC 这个 channel 扔。经过简单的扒拉代码可以发现，这里 confChangeC 就是上面 `raftNode` 实例中的 confChangeC 这个成员变量。

由此，我们需要关注的就两个地方，一个是 `raftNode` 的 startRaft 方法，一个是 `kvstore` 的 readCommits 方法。再查看一下 readCommits 方法，可以看到它做的事情也比较简单，就是不断从 commitC 这个 channel 中获取数据，反序列化之后存入 `kvstore` 结构体的成员变量 `kvStore`。如果数据是 nil 的话则尝试加载 snapshot，替换整个 `kvstore` 结构体的成员变量 `kvStore`。而 commitC 哪蹦达出来的呢？实际上它就是上面 `raftNode` 实例中的 commitC 这个成员变量。至此一切都和上面的 `raftNode` 实例联系上了。而其它的代码我们也浏览完了，知道了它们各自的作用，那么就只要好好看一下 `raftNode` 的 startRaft 方法就行了。
 
### startRaft 方法
 
在开始围观 startRaft 方法之前需要注意一点，提供 startRaft 方法的 `raftNode` 实例实际并不是 etcd 的 raft 包里面定义的。这个结构体是==自己定义==的。在 etcd/raft/doc.go 中对 raft 包的使用方式进行了说明，这里只是进行了封装。

为了能够直白的理解 raft 算法，startRaft 方法中有些地方可以先暂时忽略，只需要知道其作用即可。startRaft 方法中做了如下这么几件事。

1. 加载 wal，如果不存在则创建对应的目录
2. 初始化 rc.node 成员变量，这里需要根据是否存在 wal 日志来区分这个节点是否是第一次创建。 
3. 初始化 rc.transport 成员变量。
4. 开启 rc.serveRaft goroutine。
5. 开启 rc.serveChannels goroutine。

这里所说的 rc 就是上面说的 `raftNode` 实例。在上面的步骤中，需要关注的只有 2、4、5 这三个步骤中，下面依次分析。

首先查看的是 rc.serveRaft 这个方法。这个方法开启了一个 http server 用来处理 raft 相关的请求。简单的扒拉代码可以看到对于收到的 raft 相关的请求最终调用的是 h.r.Process(context.TODO(), m) 去处理的(etcd/rafthttp/http.go:117)。而这里的 h.r 就是上面提到的 `raftNode` 实例。其它细节暂时也不深究了。

再来看看 rc.serveChannels 方法这个方法主要处理 rc 这个结构体中各种 channel 传来的消息，外加了一个定时器的定时事件处理。具体细节涉及到 rc.node 所以需要结合上面第 2 步的代码一起看。

初始化 rc.node 成员变量需要区分节点是旧节点重启还是新节点。简单起见，这里只考虑新节点的情况，也就是只需要看 StartNode 方法就行了。需要注意的是 StartNode 是在 raft 包里定义的。在 StartNode 方法中有几个主要结构体是需要注意的，总结如下。

![struct](https://github.com/lkk2003rty/notes/raw/master/images/raft_struct.png)

StartNode 方法干的事儿也挺简单的，创建 raft 实例，创建 node 实例，然后将 raft 实例作为参数传递给 node.run 调用。这里 node.run 是作为一个 goroutine 跑起来的。

所以，干活的 goroutine 也就是 node.run 这个方法了，核心的数据结构也就是 raft 和 node 这两结构体了。可以看到这两结构体恰好也就是 raft 包里面定义的。

仔细查看 node 和 raft 这两个结构体就会发现 node 结构体的成员变量基本都是 channel 类型的，而 raft 结构体成员变量的类型比较多。可以看出来实际上 raft 表示的是该节点当前的状态等信息。而这些状态的变更维护实际上就是 node.run 里面做的事情，通过往 node 的各种 channel 放入不同的消息来触发对应的操作。

![state](https://github.com/lkk2003rty/notes/raw/master/images/raft_state.png)

按照 raft 论文的描述每个节点都是一个状态机，之间状态的变更如上图所示。对应到代码的话 raft.state 存的就是该节点当前的状态。在一个集群中，只会有一个 StateLeader 状态的节点，其它节点都是 StateFollower。

StateLeader 节点就相当于集群的老大，任何需要同步的信息都需要老大批准才可以生效，否则就会被认为是无效的。那么同步的信息放哪呢？当然是放日志里啦。这里日志可不是程序打印的那种日志，而是会有描述集群有几个节点，追加了什么信息（对于 raftexample 这个例子来说就是新增/删除键值对），新增/删除哪些节点这样信息的日志。所以，StateLeader 节点要干的事情就是尽量让集群所有节点的日志都长得一样。下面就按照每个状态切换的顺序来阅读代码。

所有节点一开始的状态都是 StateFollower。这一点从 node.go 181 行就能看出来。那么一个节点怎么从  StateFollower 变成 StateLeader 的呢？这就需要选举了。下面就看一下选举的过程。
  
## 选举

raft 的选举其实就是发起投票，然后检查自己获得了多少张票，如果票数超过了集群节点数的一半，那么这个节点就华丽转身成了 StateLeader 了。
	
### 触发时机

每个节点本身有一个定时器，如果在一定时间内没有收到需要处理的靠谱消息的话，这个节点就会发起选举。这里有两个地方需要明确：超时时间和哪些是靠谱的消息。

首先来看超时时间的选取，在论文中说了这个超时时间是随机生成的。这是作者试了半天试出来的。从之前提到的 node.go 181 行的 becomeFollower 函数也可以看出来这一点。

其次是靠谱消息的定义。由于是分布式的环境，并不是我们收到的所有消息都是需要响应的，比如这条消息是很早以前发送的但由于网络阻塞现在才收到等。因为就需要在消息中添加一些字段来供我们判断，我们是否要处理。由于 raft 是基于日志的，那么明显需要一个字段来标识当前日志的版本号。也就是说比如节点 A 给节点 B 发送一条消息，那么这条消息就会带有节点 A 本地日志的最新版本号。这个版本号对应到报文中就是 index 这个字段。通过阅读代码，对比论文可以知道这个字段是单调增加的。除此以外，对于一个分布式环境而言，任何一个节点任何时候都有可能 crash 或者失联。如果身为 StateLeader 的节点 crash 了，那么势必会引起新一轮的选举。再考虑到网络延迟的问题，我们还需要另外一个字段来标识这条消息是属于哪一个任期的。报文中就用 term 这个字段来表示任期的。基于这两个字段我们就可以得出靠谱消息的定义。只有消息中的 term 大于或等于本地且消息中的 index 大于或等于本地的日志版本号，这样的消息才是靠谱的。

那么哪些靠谱消息会打断节点发起选举呢？有下面几种

- MsgApp
- MsgHeartbeat
- MsgSnap
- MsgTimeoutNow

看到这里可能会有些懵逼，这到底有多少种消息呢？见下图

![msg](https://github.com/lkk2003rty/notes/raw/master/images/raft_msg.png)

上面说的超时发起选举对应到代码是在哪呢？入口就在于上面提到的 rc.serveChannels 这个 goroutine 里面。在这个 goroutine 里面启动了一个 timer，每次到点调用 rc.node.Tick()，这里的 rc.node 就是 node 结构体实例。node 结构体的 Tick 方法也啥事没干就往自己的 tickc 这个 channel 扔了一个 struct{}{} 就完事了。既然是 channel 有往里扔的，那么就有往外取的，之前作为 goroutine 跑起来的 node.run 里面就是干取这件事的。可以看到最后调用的是 raft 结构体的 tick 方法，但是查看 raft 结构体的定义就能发现，tick 只是个成员变量而已。那它在啥时候被赋值的呢？还记得之前我们调用过一次 becomeFollower 函数么？就在这里面被赋值的。其实，这个成员变量都是在 raft 的 becomeXXX 这样的函数里面被赋值的。

### 选举流程

从上面的状态变迁图可以看出来从 StateFollower 变成 StateLeader 可谓是路漫漫啊，其中还需要先变成 StateCandidate 然后才能变成 StateLeader。其中还有条路线更是曲折，StateFollower -> StatePreCandidate -> StateCandidate -> StateLeader。

下面先分析第一条路线，即 StateFollower ->  StateCandidate -> StateLeader。

#### StateFollower -> StateCandidate

这个状态变迁比较好理解，就是节点自身把任期增加 1，投票给自己，然后将拉票信息(MsgVote)广播给其它节点。

对应的代码只要接着上面说到的往下跟就行了。需要注意的是这里的设计思路。这里为了触发节点状态切换，自己组装了一条消息在 tickElection 中直接调用 Step 函数去处理。而这个函数恰好就是节点接收到消息的处理函数。为了把这条消息和其它的消息区分开来，这条消息的 Term 被置为 0，且类型为 MsgHub。

#### StateCandidate -> StateLeader

当节点已经切换到 StateCandidate 状态之后需要做的事情就是等待。等待其它的节点投票同意其为 StateLeader 节点。那要多少票才能算数呢？当然是过半啦。一旦收到的票数过半了，那么该节点就会切换到 StateLeader 状态。这里注意的是节点状态虽然切换到但是其它节点并不知道啊，其它节点投票并不会重设定时器。如果投票后过一段时间啥事没发生还是自己变成 StateCandidate 会发起拉票请求的。因此节点一旦收到如果多的票数变成 StateLeader 后徐需要广播 MsgApp 消息让其它节点乖乖变成 StateFollower。

对应的代码其实直接从 Step 函数入手开始看就行了。具体来说就是 r.step(r, m) 里面对 MsgVoteResp 消息的处理。

#### StateCandidate -> StateCandidate

这个状态变迁其实就是上面说的等待中的状态。比如一个超过 3 个节点的集群，其中某个节点已经发出去了拉票信息，然后过了一会儿收到了一条其它节点发来的赞同消息(MsgVoteResp)。这个节点一看，这才两票(有一票是自己投给自己的)还没过半数呢，于是状态不变，继续等着。但是也不能老是等着吧，身处 StateCandidate 状态的节点也是有超时时间的，超过了超时时间就会再次发起选举，这时候状态依旧不会变(但是既然是重新发起选举，之前收到的票统统不算数啦)。这里需要注意的是每次调用 becomeXXX 函数来切换状态(哪怕之后的状态和之前的一样)都会重新设置选举超时时间。

对应的代码可以看 stepCandidate 里面对 MsgVoteResp 消息的处理，或者 tickElection 函数重新发起选举。

#### StateCandidate -> StateFollower

这个状态变迁可以参考以下两种场景。

场景一：节点在等待赞同票的过程中，收到其它节点发来的我是 StateLeader 的消息(MsgApp)，或者其它节点已经变成 StateLeader 了定期心跳检查的消息(MsgHeartbeat)。这时候我们只能乖乖认怂，赶紧拜山头。具体代码见 stepCandidate 函数对 MsgApp 和 MsgHeartbeat 消息的处理。需要注意的是其实 Step 里面对消息中的 Term 和节点的 Term 不一致的情况做了下处理，但是在这个场景中并没有什么影响，主要的流程还是在 stepCandidate 函数里面。其实不止 MsgHeartbeat 和 MsgApp 这两种消息会造成  StateCandidate -> StateFollower 这样的效果，MsgSnap 或者收到超过半数的反对票也会有同样的行为。

其实这里描述的有点小问题，MsgApp 类型的消息不一定说明我是 StateLeader，只能说明这样的消息是从 StateLeader 节点发出来的。在成为 StateLeader 节点的时候会广播一条 MsgApp 消息。除此以外，一旦日志有新的数据进来，StateLeader 节点将数据添加完之后也会将需要添加的数据附在 MsgApp 消息中广播给其它节点。具体到 raftexample 这个例子，就比如说新增了一个键值对啦，就会有这种消息出现。

场景二：节点在等待赞同票的过程中，收到其它节点发来的拉票消息(MsgVote)，并且消息中的 Term 大于节点的 Term。这时候对应的代码就在 Step 函数里面了(raft.go:784)，stepCandidate 函数没有对这种消息进行处理。这里需要注意的是此时节点虽然切换成了 StateFollower 状态，但是其并不知道 StateLeader 节点是哪个，也没有对这样的消息给出回应。这与场景一是不一样的。

那么场景二是怎么发生的呢？考虑一个由 A、B、C、D、E 五个节点构成的集群，A 节点是 StateLeader。跑着跑着发生了 partition。也就是假设，A、B、C 这三个节点之间能够两两进行通信，D、E 这两个节点之间能够互相通信。但是 A、B、C 与 D、E 之间是无法通信的。比如 A 要给 D 发条消息，那就是死活不成功。这样的话持续一段时间，按照 raft 的流程来说，D、E 这两个节点会因为收到不到心跳检查的消息(MsgHeartbeat)而导致超时发起选举。但是由于 partition 的缘故，D、E 都不可能成为 StateLeader 的(票数没过半)，于是乎就只能不断的超时、发起选举死循环了。但是注意到发起选举的时候会将节点的 Term 加 1。久而久之，D 节点的 Term 就会比 A 节点的要大(假设 A、B、C 这三个节点一直稳定运行，那么它们的 Term 是不会变的)。这时候，partition 的问题被修复了，恰巧 D 节点又发起选举了，于是乎一条 Term 明显比较大的拉票消息(MsgVote)就发送到了 A、B、C 节点了。

#### StateLeader -> StateFollower

这状态切换的就比较诡异了，原来领导也不好干啊，还有变成 StateFollower 的一天啊。实际的场景是怎么样的呢？查看下 Step 函数以及后续的 stepLeader 函数就知道在 StateCandidate -> StateFollower 中我们描述的场景二就会发生这样的情况。对应的代码和场景二的是一致的。

以上就是成为 StateLeader 的第一条路线了。那第二条路线呢？可以看到，第二条路线有很大一部分是重合的，只是在变成 StateCandidate 状态之前引入了一个新的状态 StatePreCandidate。那么为什么要引入这个新的状态呢？从代码注释中可以找到答案(raft.go:164)。这代码注释的位置是一个配置项，只有这个配置项被设置为 true 的时候我们才有可能看到这第二条路线。结合论文可以看到它是为了解决我们在 `StateCandidate -> StateFollower` 中描述的场景二的问题。在场景二中，由于之前存在 partition 的缘故会导致在 partition 的问题得到修复之后的一段时间内可能会收到 MsgVote 导致其它的节点一起回退到 StateFollower 状态，然后重启发起新一轮选举。仔细考虑下，其实这是完全没有必要的。当 partition 存在的时候其中有一部分还是可以正常 work 的，也就是说 StateLeader 节点还活得好好的。那么，当 partition 消失之后只要同步下本地日志，重新加入到已有的集群就可以了啊，没必要整体回退到 StateFollower 状态，发起新一轮选举啊。为了达到这样的效果，这里引入了 StatePreCandidate 状态。

上面问题的根源就在于 MsgVote 中的 Term 是是比节点本身的 Term 要来得大，导致收到消息的节点不得不调整自身的 Term 来同步。而消息中的 Term 来源与节点本身的 Term。考察下从 `StateFollower -> StateCandidate` 过程中节点是先将自身的 Term 加 1 然后再去发拉票消息。那么，不要增加 Term 不就行了么？本质上来说 `StateFollower -> StatePreCandidate` 就是这么干的。

#### StateFollower -> StatePreCandidate

这个状态切换的时机和 `StateFollower -> StateCandidate` 是一样的，都是由于定时器到期。但是这里变成 StatePreCandidate 状态的时候需要注意的地方如下有两点

- 节点自身的 Term 并没有增加 1，但是发出去的预拉票消息(MsgPreVote)中的 Term 是加 1 的。
- 节点并没有重设定时器的时长

其它的部分基本和 `StateFollower -> StateCandidate` 是一样的，stepXXX 使用的函数都是一样的。

#### StatePreCandidate -> StatePreCandidate

这个状态切换和 `StateCandidate -> StateCandidate` 类似，就是收到 MsgPreVoteResp 消息但是没有过半数只能傻等了。

#### StatePreCandidate -> StateCandidate

这个状态切换发生的时机就是收到了超过半数的 MsgPreVoteResp 消息(包含自己投给自己的)。


#### StatePreCandidate -> StateFollower

这个状态切换其实直接看 `StateCandidate -> StateFollower` 就可以了，毕竟 stepXXX 函数都是一样的。

以上就是节点所有的状态切换了，虽然本来是描述一个节点怎么变成 StateLeader 的，但是顺带也把其它的状态变迁都描述了一遍。但是这里我们漏说了一点，节点是可以投反对票的。举例来说节点 A 先收到了节点 B 发过来的(预)拉票请求(MsgPreVote/MsgVote)，然后投票给了它。过了一段时间又收到节点 C 发过来的(预)拉票请求(MsgPreVote/MsgVote)，这时候就会发送反对票的请求，也就是在消息中 Reject 的值为 true。这里假设在这个收到这两个(预)拉票请求的过程中，节点 A 的选举定时器未超时。

接下来还有什么需要考虑呢？当然是集群中一个节点变成 StateLeader，其它节点是  StateFollower 的情况下，怎么维持稳定。毕竟每个 StateFollower 都有个定时器，超时会重新发起选举的，因此需要有一套机制去重设 StateFollower 节点的定时器，让它老老实实的保持在 StateFollower 状态。这套机制就是发送心跳啦。下面来查看下。

## 心跳

当一个节点状态变更为 StateLeader 后会定期发送心跳消息(MsgHeartbeat)从而让其它节点重置定时器。那么首先需要解决的问题是这个定期时间多长合适。显然，如果这个时间大于选举时间的话是没用的。所以心跳的间隔时间小于选举超时时间。再结合 thesis 3.9 中的内容，心跳间隔时间也不能太短，至少要大于广播时间，这里广播时间指的是并行发送消息到集群中每个节点并收到响应的耗时。当然选举时间也是有上限的，不能超过一个节点 crash 的平均时间。

心跳消息的类型为 MsgHeartbeat，其它节点收到这一类型的消息回复 MsgHeartbeatResp 类型的消息即可。

这里还有一个选项可以配置，查看 tickHeartbeat 函数的话就会发现如果设置了 CheckQuorum 的话，那么身为 StateLeader 状态的节点每隔一个选举超时时间的话就会检查一下在这一时间间隔内收到 MsgHeartbeatResp 的节点数量是否过半，如果没有过半的话，就回退到 StateFollower 状态。这样的话就能提早发现 partition 了。

## 节点变动

在实际使用中，可能会发生往集群中添加或删除节点的情况。此时，节点的数量会发生变更，而且新添加进来的节点还需要与 StateLeader 的节点同步日志，raft 对这种情况是怎么处理的呢？下面就来考察下。

幸运的是在 raftexample 这个示例中已经考虑到了这个情况。在其 README 中可以看到添加/删除节点的操作。

### 增加节点

新增节点包含两个操作，一个是告知当前集群的老大也就是 StateLeader 节点，再一个就是启动新增的节点。首先来看一下启动新增节点的操作。

在 raftexample 示例中，启动新增节点和启动普通节点的唯一区别就是启动的时候多了个 --join 参数。而这个参数具体的影响见 raftexample/raft.go 的 287 行。其直接影响就是 raft 结构体的 prs 字段是一个空字典，而不是像正常启动的节点会有其它节点的默认信息。粗看起来这并没有什么影响，超时了照样会发起选举，其实不然。我们来看一下超时发起选举的函数。

```Go
 // tickElection is run by followers and candidates after r.electionTimeout.
 func (r *raft) tickElection() {
     r.electionElapsed++

     if r.promotable() && r.pastElectionTimeout() {
         r.electionElapsed = 0
         r.Step(pb.Message{From: r.id, Type: pb.MsgHup})
     }
 }
 
 // promotable indicates whether state machine can be promoted to leader,
 // which is true when its own id is in progress list.
 func (r *raft) promotable() bool {
     _, ok := r.prs[r.id]
     return ok
 }
```

其中，promotable 函数中用到了 raft 结构体的 prs 字段。显然，在这样的流程下，新增的这个节点在收到其它节点给他发送的请求之前将会一直处于 StateFollower 状态。

这里既然说到了 prs 这个字段，那就顺带好好来看一下这个字段的信息。首先需要明确的是这个字段的用途。raft 实质上一个基于日志的一致性算法，为此需要尽量保证集群中每个节点的日志是一致的。为了达成这个目的，处于 StateLeader 状态的节点用 prs 这个字段记录其它节点的当前的日志状态。StateLeader 节点就根据这里面记录的信息来决定发送哪些日志给对方。prs 是一个字典，其 value 类型为 Progress。

再来看一下之前的告知操作。从 raftexample/httpapi.go 中很容易就追踪到 node.go 的 ProposeConfChange 函数了。这里需要区分接受到告知操作的节点当前是什么状态，如果当前节点是 StateFollower 状态的话那么，节点会将这样的消息转发给 StateLeader 节点处理。这里需要注意的是不是只要是 StateFollower 状态的节点都会转发消息的，raft 中有个配置项目(DisableProposalForwarding，raft.go:193)一旦设置为 true，那么这样的消息会被忽略掉的。这里先不考虑这种情况，那么只要查看 StateLeader 状态的节点是怎么处理 MsgProp 消息的就可以了。

StateLeader 状态的节点收到这样的消息后，要做的操作其实很简单，其实就是往 raft 结构体的 raftlog.unstable.entries 里面扔然后广播给整个集群就是了。但是这里要注意的是，使用这种方式的话一次只能添加一个节点，这一点可以从在处理 MsgProp 的时候会通过设置 r.pendingConf 的值可以看出来。那如果要一次添加多个节点要怎么处理呢？下面会再分析。此外还有就是广播给集群的消息类型可就变了，不再是 MsgProp 消息了而是 MsgApp 消息。那么什么时候才会真正把这个节点加进来呢？首先需要确定什么才能算是把节点添加进来的判断标准。对于 StateLeader 状态的节点而言所有的其它节点都记录在 prs 这个字段里，所以只要新增的节点被添加到这个字段中就可以认为是节点添加成功了。根据这个标准去阅读代码的话很快就能够梳理出了下面的流程。在带有节点变动的 MsgApp 消息广播之后，只要得到了超过半数的响应，StateLeader 状态节点会调整自身日志的 Commit 的值即 raft 结构体的 raftLog.committed，从而在 node 的 run 方法填充 Ready 结构体的时候这部分消息会被添加到 CommittedEntries 中，在应用层处理的时候调用 rc.node.ApplyConfChange 把新加节点信息添加到 raft 结构体的 prs 中。但是，这还没完，新添加的节点还需要与 StateLeader 节点同步日志。触发同步日志的就是 StateLeader 节点的心跳机制。

那么如果要添加多个节点该怎么办呢？一种办法就是一个一个的加，当一个添加成功之后再添加下一个。那有没有一次性搞定的方法呢？在 thesis 4.3 中对这种批量修改操作的描述，不过比较复杂。因为会存在一个中间的过渡阶段要处理。对应到 raft 库代码的话，没有看到现成的实现，只是在 raft 结构体中用 learnerPrs 来标记批量添加的节点，但是没有看到完整的流程。暂时就不深究了。

### 删除节点

删除节点的流程和添加节点基本上一样的。在上面的描述中只有在真正处理的时候才会区分是添加节点还是删除节点。在真正删除节点的时候会调整 raft 结构体 raftLog.committed 的值。

## 一致性读

在 stepXXX 函数中还可以看到一类名为 MsgReadIndex 的消息，这一类型的消息是干啥用的呢？很简单，就是用来处理一致性读的。raftexample 代码中明显是没有这一类消息的。那么就只能活生生扒拉 etcd 的代码了。那么 etcd 涉及到 MsgReadIndex 消息的代码在哪呢？当然是从请求入手啦，etcd 用来获取键值对的请求是 Range 请求，就从这入手。Range 请求的处理见 etcdserver/v3_server.go 中的 Range 函数。默认情况下 Range 请求的 Serializable 为 false，这时候 ectd 就会使用 MsgReadIndex 消息来保证读取的键值对是一致的。其主要操作就两步，首先通过 raft 协议确认本地的数据是和集群中大多数节点是一致的，然后从本地查数据出来返回。具体代码分析如下。

etcd 在启动的时候会启动一堆 goroutine 其中有一项目就和一致性读相关，具体代码位于 etcdserver/server.go：528 Start 函数，其中用于一致性读的函数是 s.linearizableReadLoop。那么再看看一眼 上面说道的 Range 请求的处理函数。在这里处理函数里面通过两个 channel 和 s.linearizableReadLoop 通信，一个用来告知需要进行一致性读(s.readwaitc)，一个用来接受处理结果(s.readNotifier.c)。可以看到真正干活的实际上是 s.linearizableReadLoop 这个函数。在这个函数里面进一步调用 raft 库的相关函数去处理，而后通过一个 channel 等待处理结果。在 s.linearizableReadLoop 函数中会为每一次一致性读操作分配一个自增的 id 作为这次操作的唯一标识。这个 id 也会在后续 raft 库发起 raft 消息的时候用到。此外，s.linearizableReadLoop 当 raft 库处理完了之后并不是说本地的数据就已经跟上了。还可能需要等待本地数据同步。这里下面会说到。

在 raft 库中具体实现的话会根据配置项的不同而有不同的处理流程。这个配置项实际上就是 raft 结构体的 readOnly.option 字段，一共有 ReadOnlySafe 和 ReadOnlyLeaseBased 这两种选择。 

### ReadOnlySafe

此时如果接受 Range 请求的节点是 StateFollower 状态的，那么其处理流程如下。

![read_index](https://github.com/lkk2003rty/notes/raw/master/images/raft_read_index.png)

如果接受 Range 请求的节点是 StateLeader 状态的，那么只要去除上面时序图中虚线的两次交互即可。

在上面流程中有几个地方是需要注意的

- 在当前 Term 中需要有日志 commit 才可以进行后续的操作，这一步校验就是图中第一步中的合法校验。
- 在上面的交互流程中 StateLeader 收到了 MsgHeartbeatResp 消息后有可能会触发 StateLeader 和 StateFollower 之间同步日志的操作。而同步日志与这里 raft 库处理读一致性的操作是异步的，因而需要在 s.linearizableReadLoop 函数中等待本地数据完成同步，在 StateFollower 日志的 Index 追上 StateLeader 日志 commit Index 才能进行进一步操作。

### ReadOnlyLeaseBased

相比 ReadOnlySafe 这就简单多了，其实就是上面的时序图去掉实线的两次交互。

可以看到这个明显就是比上面的水多了，如果当前集群出现 partition，没有配置 CheckQuorum 的话，是有可能出现两个 StateLeader 节点的，这样的话这里的读一致性是没法保证的。

## leader transfer

什么时候需要做 leader transfer 呢？在 thesis 3.10 中给出了几个实际的场景，对应到 etcd 的代码的话可以看到一个比较明显的场景就是当前跑着的 etcd 进程恰好是 StateLeader 节点，而我们却需要停止这个进程。代码的话只要从 etcdserver/server.go:1019 的 `func (s *EtcdServer) Stop()` 开始围观就行了。下面就记录一下 leader transfer 流程吧。

StateLeader 节点收到 leader transfer 的消息后，检查继任者是否已经是最新的日志。如果不是则发起 MsgApp 消息同步日志。如果是的话则发送 MsgTimeoutNow 消息给继任者。这里需要注意的是 StateLeader 节点会将 raft 结构体的 leadTransferee 字段赋值，这样如果在 leader transfer 整个过程还没有完成的情况下收到 MsgProp 消息就不会处理。这里的 MsgProp 消息是干啥用的呢？在 raftexample 中 StateFollower 节点转发的新增节点请求啦删除键值对啦新增键值对啦都是通过这一类型的消息通告  StateLeader 节点的。其次，按照论文中的要求 leader transfer 需要在一个选举超时时间内完成，所以这里也会重置定时器的值，也就是 raft 结构体的 electionElapsed 字段。再者，如果是先同步日志的话，同步日志的过程会一直持续直到继任 StateLeader 节点的日志已经同步了才会向其发送 MsgTimeoutNow 消息。

话分两头说，继任者节点收到 MsgTimeoutNow 消息后会直接将本身的 Term 值自增，切换成 StateCandidate 状态，而后发起选举流程。这里哪怕设置了 PreVote 也会直接切换成 StateCandidate 状态。后面的流程就是之前描述的选举流程了。

由于 leader transfer 要求在一个选举超时时间内完成，如果没有完成原来的 StateLeader 节点需要中断这一过程，继续维持 StateLeader 状态。

扒拉了下这部分代码，唯一的问题就是这个继任 StateLeader 节点是怎么选出来的。在 etcd 中是选取连接时间最长的集群中的节点。不过是否有其它的更好的选法呢？  

## 消息发送

### raft 状态机之外的状态机

这里说的 raft 状态机之外的状态机指的实际上是 StateLeader 节点维护的集群中各个节点的状态，也就是藏在 raft 结构体的 prs 字段中的 State。这个状态会影响到消息的发送。首先来看一下其状态的变迁图。

![progress_state](https://github.com/lkk2003rty/notes/raw/master/images/raft_progress_state.png)

- ProgressStateProbe：这个状态其实是低速模式，也就是 StateLeader 节点发送一条消息之后必须等到收到对方给出响应之后才发送下一条消息。
- ProgressStateReplicate：这状态相当于高速模式，自带一个发送窗口，已经发送了但是还没收到响应的的消息都会被记录在这个窗口中，收到了就从这个窗口中移除。其记录的队列是一个环状队列。这里有点类似 TCP 的发送窗口。
- ProgressStateSnapshot：这个状态其实是高速模式的升级版，也就是发现对方的日志和自己的差太多了，只能发个快照过去。

### 状态机切换

#### ProgressStateProbe -> ProgressStateReplicate

所有的节点一开始都是 ProgressStateProbe 状态，一旦发送了一条消息而后接受到了其响应报文后就会切换成 ProgressStateReplicate 状态，查看 stepLeader 函数中对 MsgAppResp 消息的处理就知道了

#### ProgressStateReplicate -> ProgressStateProbe

当收到拒绝的 MsgAppResp 消息并且需要更新 progress 结构体中数据的时候触发。实际上也就是出现了本地 progress 和对方节点实际存储的日志 Index 不一致的时候，需要将本地 progress 中的 Next 回退时候会发生这样的事情。此外还有在处理其它类型消息也有可能会触发这一状态变迁，这里就暂时先忽略。

#### ProgressStateProbe -> ProgressStateSnapshot || ProgressStateReplicate -> ProgressStateSnapshot

把这两状态切换放一块是因为只有在 raft.go 的 sendAppend 函数中会切换到 ProgressStateSnapshot 状态。而切换到一状态之前并没有考虑之前是什么状态的，单纯是由于对方落下的日志太多了只能发快照了。

#### ProgressStateSnapshot -> ProgressStateProbe

这个切换的一样是在收到 MsgAppResp 消息处理的时候，发现节点的日志已经追上了之前发送快照中记录的 Index 了，就切换到 ProgressStateProbe 状态了。


## 日志记录

由于 raft 是基于日志的，那么日志的记录的记录就尤为关键，这里就来看一下 raft 在这方面是怎么处理的。

### 日志类别

raft 本身的日志有两种 wal 和 snapshot。wal 日志记录的是 raft 消息，但是并不是所有的 raft 消息都会被记录下来，比如 MsgVote 这类的消息就不会被记录在案。而 snapshot 记录的消息消息则可分为两部分。一部分是节点本身的状态，比如 Term 啦，Index 啦等，另外一部分则是业务层面的数据，在 raftexample 这个例子中指就是所有的键值对啦。具体 snapshot 的结构可以看 raft 消息那种图中 Snapshot 结构体。

### 日志处理流程

并不是所有的 raft 消息都是用于保证一致性的，可以看到 raft 消息中的 Entry 字段是可以携带私有数据的(Data 字段)。对于 raftexample 这个例子而言这里的私有数据就是键值对的变动。因此对于 raft 消息的处理有两方面的需求，一方面来源于业务层面，一方面则是 raft 协议自身的需求。raft 协议层面需要做两件事情，其一是根据消息的类别进行处理（比如投票、自身状态切换等），其二则是将部分类别的消息记录在案(比如当前节点是 StateFollower 状态，那么这时候收到 MsgApp 消息就需要将这条消息存下来)。在上面状态切换等操作可以在 Step 函数和 stepXXX 函数中完成，但是并没有进行写入文件等存储操作，也没有业务层面的操作，那么这些操作是在哪里进行的呢？仔细查看 node.go 中的 run 函数的话很快就能够发现其中的猫腻。

在 run 函数中，首先会比对节点当前的状态，看看是否有需要发送的消息，是否有需要后续记录和交给业务层面处理的数据，如果有的话把这些数据包填充到一个 Ready 结构体中再通过 node 结构体的 readyc 传送出去。而应用层面则通过从 readyc 获取数据做进一步的处理，待处理完成后往 node 结构体的 advancec 扔个 struct{}{} 表示处理完成(对应到就是 raftexample/raft.go 中的 serveChannels 函数)。同样的，在 node.go 中的 run 方法也会接收 advancec 的数据，再进行后续的处理。

首先看眼 Ready 结构体长什么样。

```go
// Ready encapsulates the entries and messages that are ready to read,
// be saved to stable storage, committed or sent to other peers.
// All fields in Ready are read-only.
type Ready struct {
	// The current volatile state of a Node.
	// SoftState will be nil if there is no update.
	// It is not required to consume or store SoftState.
	*SoftState

	// The current state of a Node to be saved to stable storage BEFORE
	// Messages are sent.
	// HardState will be equal to empty state if there is no update.
	pb.HardState

	// ReadStates can be used for node to serve linearizable read requests locally
	// when its applied index is greater than the index in ReadState.
	// Note that the readState will be returned when raft receives msgReadIndex.
	// The returned is only valid for the request that requested to read.
	ReadStates []ReadState

	// Entries specifies entries to be saved to stable storage BEFORE
	// Messages are sent.
	Entries []pb.Entry

	// Snapshot specifies the snapshot to be saved to stable storage.
	Snapshot pb.Snapshot

	// CommittedEntries specifies entries to be committed to a
	// store/state-machine. These have previously been committed to stable
	// store.
	CommittedEntries []pb.Entry

	// Messages specifies outbound messages to be sent AFTER Entries are
	// committed to stable storage.
	// If it contains a MsgSnap message, the application MUST report back to raft
	// when the snapshot has been received or has failed by calling ReportSnapshot.
	Messages []pb.Message

	// MustSync indicates whether the HardState and Entries must be synchronously
	// written to disk or if an asynchronous write is permissible.
	MustSync bool
}

// SoftState provides state that is useful for logging and debugging.
// The state is volatile and does not need to be persisted to the WAL.
type SoftState struct {
	Lead      uint64 // must use atomic operations to access; keep 64-bit aligned.
	RaftState StateType
}
```

其中，Entries 的来源为 raft 结构体的 raftlog.unstable.entries 而这一字段内容是什么时候被填充上的呢？这就得看情况了，下面列出了几种情况。

- 如果节点当前是 StateFollower 状态且收到了条 MsgApp 的消息那势必要往  raftlog.unstable.entries 里追加数据。
- 如果当前节点是 StateLeader 状态且收到了条 MsgProp 消息，也会追加数据(什么时候会收到这样的消息呢？比如节点的增加删除，StateFollower 节点转发过来的消息都有可能。比如当 StateFollower 节点要增加一个键值对就会发送这样的消息过来)。
- 如果当前节点是 StateCandidate 状态且刚好收到了足够多的票数在华丽转成为 StateLeader 的时候也需要追加数据。

其实这些追加操作都躲在 stepXXX 函数和 Step 函数里，只要细细检查就能发现。

再来看下 Ready 结构体的其它字段，Messages 是存储要发送的消息，具体的发送也是在业务层面负责调用的。

CommittedEntries 表示的是已经被集群中超过半数成员认可的消息，但是还没有具体进行处理，业务层面需要进一步处理。这又是什么意思呢？比如说 raftexample 中有一个节点需要新增键值对，那么这样的请求首先要告知给 StateLeader 节点，StateLeader 再进一步广播给整个集群，只有当整个集群超过半数的节点给出了响应，StateLeader 节点才会把这个新增的键值对增加到自身的 map 中。这里添加到自身 map 的新增键值对就是从 CommittedEntries 中找出来的。那 CommittedEntries 又是怎么选出来的呢？其来源是 raft 结构体的 raftLog 字段，这个字段中有两个地方存储了 raft 消息，storage 和 unstable。前面已经说了消息都是追加到 unstable.entries 中的，在业务层面调用 `rc.raftStorage.Append(rd.Entries)` 把原来 unstable.entries 的数据追加到了 storage 中(注意这里并不会修改 raft 结构体 unstable.entries 的值，rd.Entries 是拷贝出来的)。业务层面的操作完事后，node.go 的 run 方法从 advancec 得到通知，把这部分数据从 raft 结构体 unstable.entries 中清除掉。因此可以看出，storage 和 unstable 就构成了全部的消息列表。CommittedEntries 呢就是从这个列表中找到 `[raftLog.applied + 1, raftLog.committed]` 的消息。raftLog.applied 实际上就是上一次 raftLog.committed 的值(node.go:378)。换句话说就是，raft 结构体的 Step 函数中处理了一条消息，检查了下历史上有哪些消息已经被集群认可了，赶紧调整下 raftLog.committed 的值，然后在填充 Ready 结构体的时候把这些消息捞出来，交给应用层面去处理。

Ready 结构体的其它字段就暂时不关注了，接下来转而看下 raftexample 怎么处理 Ready 结构体的。主要就是写入 wal，发送待发送的消息，处理 CommittedEntries，检查是否需要生成新的 snapshot 文件等操作。 

## TODO

MsgSnapStatus、MsgUnreachable 这两类消息还没有研究是用来干啥的。



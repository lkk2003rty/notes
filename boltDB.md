# boltDB 学习

## 缘起

之前为了学习扒拉了下 etcd 的代码，结果在看代码的过程中好奇心起顺便看了下 etcd 是怎么存储 KV 数据的，于是就入了 [boltDB](https://github.com/coreos/bbolt) 的坑。

简单来说 boltDB 是一款嵌入式持久化的微型 KV 数据库。你可以把它当成一个第三方库在你的程序中来使用。boltDB 的主要原理就是通过 mmap 将文件内容加载到内存中，而后惰性加载内存中的数据。而对 db 的写操作最后转化成不同类型的页大小的字符串再写入到硬盘上。其 KV 在实际中是以 B+ 树的形式存储的，但是正常的 B+ 树在插入/删除节点的时候需要有些自平衡的操作，但是 boltDB 的画风却不是这样的。在使用的时候可以随意插入/删除 KV，任意折腾，只有在最后 commit 的时候才会去做自平衡的操作。由于 boltDB 是使用 B+ 树来存储数据的，因此 boltDB 还可以支持根据前缀遍历。

## 使用

具体的使用方法就不赘述了，看 README 就能明白。简而言之，首先先加载文件，创建出 DB 实例，而后在其基础上创建出 transaction 实例。然后在 transaction 实例中打开/创建 bucket 实例。最后在 bucket 实例上对 KV 对进行增删改查操作。在 boltDB 中，一个文件就是一个 DB，一个 DB 中可以有多个 bucket，一个 bucket 中可以存储多个 KV 对。这里需要注意的是 bucket 可以存在嵌套关系。也就是 bucket 中可以有 bucket 也可以有 KV 对。不过个人觉得应该没有什么人会这么用吧。

## 源码分析

### 主要数据结构

boltDB 主要数据结构见下图。其中 Tx 结构体就对应上面说的 transaction，而 B+ 树的根节点就是 Bucket 结构体的 rootNode 字段。此外还有些辅助的结构体。Cursor 结构体是在搜索 KV 对和 Bucket 的时候被用到的。Stats 结构体则是负责记录一些统计信息。freelist 结构体则是负责重复利用之前分配的 page 大小的 byte 数组。

![boltDB struct](https://github.com/lkk2003rty/notes/raw/master/images/bolt_struct.png)

### 主要流程分析

#### 创建 DB 实例

首先看一下创建 DB 实例的创建过程(db.go 的 Open 函数)。通过简单的代码阅读可以发现 boltDB 存储的文件大小是操作系统页的整数倍，且文件的前面两页为元信息页。再结合后续的代码阅读中发现文件中一共有如下四种页，其中 freelist 只会有一页，meta 则是固定位于文件的前两页，其它类型的页则是可有可无的，但是至少会有一页是 leafPage。

![boltDB file struct](https://github.com/lkk2003rty/notes/raw/master/images/bolt_file_struct.png)

下面是一个可能的文件内容。这里 page 的大小被设置成了 128。


&{magic:3977042669 version:2 pageSize:128 flags:0 root:{root:10 sequence:0} freelist:3 pgid:12 txid:4 checksum:2217789981381900154}

```
 ----------------------------------------------------------------------------------------------------------------------------------------
|id                              0|id                              1|id                              2|id                              3|
|---------------------------------|---------------------------------|---------------------------------|---------------------------------|
|flags                metaPageFlag|flags                metaPageFlag|flags            freelistPageFlag|flags            freelistPageFlag|
|---------------------------------|---------------------------------|---------------------------------|---------------------------------|
|count                           0|count                           0|count                           2|count                           2|
|---------------------------------|---------------------------------|---------------------------------|---------------------------------|
|overflow                        0|overflow                        0|overflow                        0|overflow                        0|
|---------------------------------|---------------------------------|---------------------------------|---------------------------------|
|ptr  |magic            0xED0CDAED|ptr  |magic            0xED0CDAED|ptr  |                          3|ptr  |                          2|
|     |---------------------------|     |---------------------------|     |---------------------------|     |---------------------------|
|     |version                   2|     |version                   2|     |                         11|     |                         11|
|     |---------------------------|     |---------------------------|     |---------------------------|     |---------------------------|
|     |pageSize                128|     |pageSize                128|                                 |                                 |
|     |---------------------------|     |---------------------------|                                 |                                 |
|     |flags                     0|     |flags                     0|                                 |                                 |
|     |---------------------------|     |---------------------------|                                 |                                 |
|     |root            |root    10|     |root            |root    10|                                 |                                 |
|     |                |----------|     |                |----------|                                 |                                 |
|     |                |sequence 0|     |                |sequence 0|                                 |                                 |
|     |---------------------------|     |---------------------------|                                 |                                 |
|     |freelist                  3|     |freelist                  2|                                 |                                 |
|     |---------------------------|     |---------------------------|                                 |                                 |
|     |pgid                     12|     |pgid                     12|                                 |                                 |
|     |---------------------------|     |---------------------------|                                 |                                 |
|     |txid                      4|     |txid                      3|                                 |                                 |
|     |---------------------------|     |---------------------------|                                 |                                 |
|     |checksum 0X1EC72C3CDAB0377A|     |checksum 0X64699C81A3096BFC|                                 |                                 |
 --------------------------------- --------------------------------- --------------------------------------------------------------------

-----------------------------------------------------------------------------------------------------------------------------------------
|id                              4|id                              6|id                              9|id                             10|
|---------------------------------|---------------------------------|---------------------------------|---------------------------------|
|flags                leafPageFlag|flags                leafPageFlag|flags                leafPageFlag|flags                leafPageFlag|
|---------------------------------|---------------------------------|---------------------------------|---------------------------------|
|count                           2|count                           4|count                           2|count                           2|
|---------------------------------|---------------------------------|---------------------------------|---------------------------------|
|overflow                        1|overflow                        2|overflow                        0|overflow                        0|
|---------------------------------|---------------------------------|---------------------------------|---------------------------------|
|ptr  |flags                     0|ptr  |flags                     0|ptr  |pos                      32|ptr  |flags        bucketLeafFlag|
|     |pos                      32|     |pos                      64|     |ksize                     4|     |pos                       4|
|     |ksize                     4|     |ksize                     4|     |pgid                      4|     |ksize                     6|
|     |vsize                    64|     |vsize                    64|     |---------------------------|     |vsize                    16|
|     |---------------------------|     |---------------------------|     |pos                      20|     |---------------------------|
|     |flags                     0|     |flags                     0|     |ksize                     4|     |     bucket                |
|     |pos                      84|     |pos                     116|     |pgid                      6|     |---------------------------|
|     |ksize                     4|     |ksize                     4|     |---------------------------|     |bucket |root              9|
|     |vsize                    64|     |vsize                    64|     |      key0                 |     |       |sequence          0|
|     |---------------------------|     |---------------------------|     |---------------------------|     |---------------------------|
|     |       key0                |     |flags                     0|     |      key2                 |                                 |
|     |---------------------------|     |pos                     168|     |---------------------------|                                 |
|     | sdfsfsfsfsdfsfsfdfsf...   |     |ksize                     4|                                 |                                 |
|     |---------------------------|     |vsize                    64|                                 |                                 |
|     |       key1                |     |---------------------------|                                 |                                 |
|     |---------------------------|     |flags                     0|                                 |                                 |
|     | fsfsfsfsfsfsfswerwrw...   |     |pos                     220|                                 |                                 |
|     |---------------------------|     |ksize                     4|                                 |                                 |
|                                 |     |vsize                    64|                                 |                                 |
|                                 |     |---------------------------|                                 |                                 |
|                                 |     |       key2                |                                 |                                 |
|                                 |     |---------------------------|                                 |                                 |
|                                 |     | sdfsfsfsfsdfsfsfdfsf...   |                                 |                                 |
|                                 |     |---------------------------|                                 |                                 |
|                                 |     |       key3                |                                 |                                 |
|                                 |     |---------------------------|                                 |                                 |
|                                 |     | fsfsfsfsfsfsfswerwrw...   |                                 |                                 |
|                                 |     |---------------------------|                                 |                                 |
|                                 |     |       key4                |                                 |                                 |
|                                 |     |---------------------------|                                 |                                 |
|                                 |     | sdfsfsfsfsdfsfsfdfsf...   |                                 |                                 |
|                                 |     |---------------------------|                                 |                                 |
|                                 |     |       key5                |                                 |                                 |
|                                 |     |---------------------------|                                 |                                 |
|                                 |     | fsfsfsfsfsfsfswerwrw...   |                                 |                                 |
|                                 |     |---------------------------|                                 |                                 |
|                                 |                                 |                                 |                                 |
----------------------------------------------------------------------------------------------------------------------------------------- 
```

#### 创建 Tx 实例

在 boltDB 中有两种 Tx 实例，一种是可读可写的，另外一种是只读的。这里需要注意的是当创建一个可都可写的 Tx 实例的时候会对 DB 结构体中的 rwlock 进行加锁操作，而对应的解锁操作只有在创建出来的 Tx 实例 Commit 或者 Rollback 的时候才会出现。也就是说在同一时间最多只能有一个可读可写的 Tx 实例出现。而对于可读 Tx 而言则没有这样的限制，在同一时间可以有多个可读 Tx 出现。但是无论是可读可写 Tx 还是只读 Tx 对应的元信息使用的都是数据库文件中那两 meta 页中 txid 比较大的那一个。

#### 创建 Bucket 实例

创建 Bucket 实例的过程分为两步。

- 查找 Bucket 名字对应的 Bucket 是否存在
- 不存在则创建个新的，存在则返回 error

首先来看一下查找的过程。在后面查阅对 KV 对增删改查代码的时候会发现这一查找过程是通用的。其实不管是 Bucket 也好 KV 对也罢，都是拿着一个 key 去找对应的 value，因此可以统一成通用的流程。下面先来看眼查找的流程。

##### 查找流程

查找流程实际是一个递归的过程，沿着 B+ 树的根节点往下找，一直找到的对应的 KV 对为止。首先需要明确的是操作的起点，在创建 Tx 实例的时候会将 root 字段填充上，这也是所有 KV 对/Bucket 操作的起点。但是 root 中的 rootNode 此时却是空的，那么该怎么查找呢？仔细查看下 cursor.go 的 seek 函数就会发现其查找的来源有两个，一个是当场加载之前 mmap 出来的数据，一个利用已有的 node 节点。这里不管是哪种来源查找的依据都是 page id。具体的查找顺序请看 bucket.go 的 pageNode 函数。这里对 page id 为 0 的情况进行了特殊处理。那 page id 为 0 的情况是怎么出来的呢。试想下这样的流程。

1. 创建一个新的 Bucket
2. 在上面创建的 Bucket 中插入新的 KV 对

在这里，第一步创建出来的 Bucket 其 page id 为 0。这样的 page 在代码中被称为 inline page

可以看到查找的过程中并不会创建任何的 node 节点。其实这里要是创建节点的话也是可以的，不过之所以没有这么做的我觉得应该是为了代码清晰以及节省内存的考虑吧。毕竟这里只是查找但并不能保证一定就能找到，那么这时候鲁莽的把节点创建出来的话其实不妥。那么什么时候创建节点呢？当然是往 Bcuket 里写新数据的时候啦。创建的时候会沿着查找过程从树的根节点开始创建 node 节点。

#### KV 对操作

KV 对的操作也是类似的流程，先查找再操作。需要注意的是删除操作会将 node 节点的 unbalanced 字段置为 true。

#### Commit

Commit 做的事儿如下：

- 调整树结构，使之成为一棵 B+ 树
- 数据落盘

首先来看一下调整树结构的过程。这里分为如下两步：

1. 节点合并，对应到代码中为 rebalance 的过程
2. 节点分裂，对应到代码中为 spill 的过程

首先来看一下 rebalance 的过程。rebalance 的操作只会针对 unbalanced 字段被设置为 true 的 node 节点。因此 rebalance 实际上是一个自底向上的过程，从包含被删除 KV 对的节点开始递归向上处理。每次的处理流程如下：

1. 如果当前节点没有父节点且只有一个子节点则将这一个子节点的内容完全拷贝到当前节点，然后回收子节点的资源。
2. 如果当前节点没有任何的子节点(对叶子节点而言其包含的一个 KV 对就是一个子节点)，那么回收当前节点的资源，在其父节点上进行 rebalance 操作(n.parent.del(n.key) 这一代码会将其父节点的 unbalanced 字段置为 true)。
3. 根据当前节点在其父节点的位置选择与其左兄弟节点或右兄弟节点合并，回收兄弟节点的资源，在其父节点上进行 rebalance 操作。

然后再查看下 spill 的过程。与 rebalance 操作类似，spill 也是自底向上的过程，从 Bucket 的 rootNode 节点开始处理。但是与 rebalance 不同的是，spill 有可能会创建出新的节点。spill 的做的事儿其实是这样的，检查当前节点的 inodes 是不是太多了，如果太多了那么就分裂成多个并且看情况创建出新的父节点出来。如果有新的父节点创建出来，则需要对新的父节点做 spill 操作。这样一番折腾之后还要调整原来 Bucket 的 rootNode 指针。这里有一点需要注意的是分割之后节点的大小还是有可能超过一个 page 的大小的(比如node.go:326中要求 1 个节点至少要有 2 个子节点或 KV 对，那么这些数据的大小累加起来就有可能超过一个 page 的大小)。这时候 page 结构体中的 overflow 字段就会派上用场，用来记录当前的 page 是连续几个 page 的。

剩下操作就是落盘了，这一部分没啥值得说的了。要注意的是 Tx 结构体中 meta 字段和 Bucket 结构体中的 bucket 字段是一个指针，有任何变动会把 meta 信息给改了，所以 Commit 的时候写入的 meta 信息和最开始读入的时候有可能不一致哦。

### 总结

以上就是 boltDB 的代码分析，其中还有些内容没有涉及。比如为了复用 page 而记录了哪些 page 还可以用的 freelist 啦，transaction 的 rollback 操作啦统计数据的记录啦等，这些操作基本瞄眼代码就能看明白，这里就不展开说明了。其中也有一些细节值得学习，比如使用 sync.Pool(db.go:901)来分配内存等。总之，boltDB 的代码还是值得一读的。



















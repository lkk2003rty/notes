# 一致性 hash

关于一致性 hash 的文章网上可谓是一搜一大把，比如[这篇](https://www.jianshu.com/p/05b3b12c111a) 就很清楚的说明白了一致性 hash 的基本原理。但是对于后续出来的几种关于一致性 hash 的算法却没有一个比较综合的总结，直到本咸鱼看到了[这篇](https://medium.com/@dgryski/consistent-hashing-algorithmic-tradeoffs-ef6b8e2fcae8)，下面就对这里面提到的几种 hash 算法做个笔记。

## Jump Hash

论文在[这里](https://arxiv.org/abs/1406.2294)。其主要精髓就是提出了一种 hash 算法，能够直接找到对应节点。在原始的环状的情况下，查找节点的过程分两步：第一步，计算 hash 值；第二步，拿着算出来的 hash 值找到离它最近的节点。而使用了 Jump Hash 的朋友直接两步合成了一步，直接就能找到对应节点。论文也是很明白的把 hash 函数的代码给 show 出来了，如下所示。

```C++
int32_t JumpConsistentHash(uint64_t key, int32_t num_buckets) {
    int64_t b = ­1, j = 0;
    while (j < num_buckets) {
        b = j;
        key = key * 2862933555777941757ULL + 1;
        j = (b + 1) * (double(1LL << 31) / double((key >> 33) + 1)); 
    }
    return b; 
}
```

使用 Jump Hash 的优点很明显，实现简单，基本没什么内存占用，快。但是要删除/添加节点就不好处理了。

## Multi-Probe Consistent Hashing

论文在[这里](https://arxiv.org/abs/1505.00062)。讲真这篇论文写得实在是简略，看了也懵逼，可以直接看博主的实现。本咸鱼总结下就是本算法支持节点的添加/删除却牺牲了查找的速度。

## Maglev Hashing

论文在[这里](http://research.google.com/pubs/pub44824.html)。如果英文看懵逼的话可以参考这篇中文的[博文](http://www.evanlin.com/maglev/)，还是比较清晰易懂的。论文中说明了这种一致性 hash 想要解决的问题如下。

- 尽量均匀分布
- 当删除/添加节点的时候尽可能少的引起扰动。

其实这和最原始的环状一致性 hash 引入 virtual node 的目的是一致的。不同的地方在于这种 hash 算法在查找节点的时候是直接查表，而环状的则是取按照一定顺序的下一个节点。所以当有新节点添加进来的话就需要重新打表，而删除的话却可以少算一步。为什么呢？因为其打表的过程有两步，先计算出一个中间结果，然后依照这个中间结果再生成最终查找用的数组。如果是新增的话，第一步就需要重新计算了，而如果是删除的话在第二步的时候则可以做一个绕过处理。在原博主的[实现](https://github.com/dgryski/go-maglev)中可以看到这步绕过处理。而在另外一位童鞋的[实现](https://github.com/kkdai/maglev)中则是老老实实的走了全部的重新打表流程。论文中也说明了之所以会设计成这样的一个原因就在于可以认为后面的节点是比较稳定的，不会经常发生变动。

## Consistent Hashing with Bounded Loads

论文在[这里](https://arxiv.org/abs/1608.01350)。对于负载均衡来说，使用传统的环状的一致性 hash 还是会存在问题的。比如对于一些热门资源来说过，使用一致性 hash 的话会导致大量的请求都转发给了同样的一台后端服务，这样就会导致某些后端服务的负载比较高而其它的却比较低。因此我们希望通过某种算法来平衡一下，让那些比较负载低的后端来分担一些请求。其原理也不难，就是为每个后端设置一个上限，而这个是每个后端正在处理的请求数乘上一个倍数。对于一个新进请求而言，经过一致性哈希选出的后端如果超过这个上限那么，就继续采用原始的一致性哈希的方式选择最近的下一个，一直到遇到一个没有超过上限的为止。

对于这个算法的实现可以查看 haproxy 的代码。lb_chash.c 文件实现了一致性哈希。其中 chash_init_server_tree 函数用于初始化信息，为每一个后端分配和其权重等量的虚拟节点加入到 haproxy 作者自创的 ebtree 中。当有新进请求时则调用 chash_get_server_hash 来选取后端。在这个函数中一旦发现配置了 hash-balance-factor 这一配置项的话就会调用 chash_server_is_eligible 去检查选出来的后端负载是否超过了上限。




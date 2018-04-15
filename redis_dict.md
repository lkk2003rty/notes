# Redis 数据结构 map 实现

为了表示我没放弃阅读 redis 代码，继续水一篇。这一次围观下 redis 是怎么实现 KV 存储的。所谓 KV 说白了就是一个巨大的 map，而在 redis 里这个 map 就是用下图中的 dict 这个结构体表示的。与 map 相关的代码都位于 dict.c/dict.h 这两文件中，下面的代码都是出自这两文件。

![GitHub set up](https://github.com/lkk2003rty/notes/raw/master/images/redis_map_struct.png)

下面就分别根据对 KV 的不同操作来看一下是怎么实现的。其实主要关注新增操作就可以了，其余的查询、删除操作基本和新增的套路差不多。

## 新增

从 dict 这个结构体中我们不难猜到真正存储 KV 数据的其实是 dict 结构体中的 ht 这个成员变量，但是这却是个数组。何以如此呢？当一开始 dict 初始化的的时候只分配的小块的内存用于存储数据，随着存储数据的增加，hash 冲突就会越来越严重。为了解决 hash 冲突，redis 这里明显采用了 separate chaining 的方式来处理冲突(dictEntry 有个名为 next 类型为 *dictEntry 的成员变量就是用来干这事的)。但是这只是一定程度上缓解了问题，更好的方式是扩大 hash table 的长度，将现有的键值对重新 hash 一遍。对应到 redis 就是把 dictht 结构体中 table 这个成员变量只想的数组长度变大，再将原有的数据挪到这个变大的数组中来。为此，dict 结构体中的 ht 这个成员变量就变成了一个数组，ht[0] 是原有的 hash table，ht[1] 是新的变大的 hash table。当所有数据重新 hash 一遍之后再让 ht[0] = ht[1] 就完事了。具体的重新 hash 的过程下面会再具体描述。

新增键值对调用的是 dictAdd 这个函数。对了，在使用 dict 这个结构体之前做的初始化操作在 dictCreate 这个函数，可以先瞄一眼。dictAdd 本身也没啥可以说的，很标准的对 key 做 hash 找到 bucket 然后将键值对的信息包装成 dictEntry 结构体写入就是了。这里可以提的就是如果 hash 冲突了，那么后来新增的键值对是放在链表的最前面。此外，dictht 中之所以会有 sizemask 这个成员变量是因为其 table 的长度都是 2 的指数(也就是 dictht 中 size 这个成员变量的值), 所以可以用 sizemask 快速确定数组下标。

## 查询

调用的是 dictFind 这个函数。

## 删除

这里的删除看起来是 dictDelete 这个函数，除此以外其实还有一个函数 dictUnlink。从 dictUnlink 的描述可以看出，dictUnlink 会返回键值对的信息，然后调用者可以拿着这键值对做些操作，而后再调用 dictFreeUnlinkedEntry 来真正销毁这对键值对(也就是真正调用 free 去释放内存)。为啥会有这样的操作呢？其一为了支持 redis 中 unlink 这个命令，其二为了支持 lazyfree-lazy-expire 这一配置项。也就是说主线程执行 dictUnlink 函数，另外的线程去执行 dictFreeUnlinkedEntry 函数。

## rehash

大概瞄完了增删查的操作，剩下的就是在新增的时候说到的 rehash 情况的处理了。

### 触发条件

要寻找 rehash 的触发条件肯定只能从新增键值对的操作中去寻找。很快就能找到触发 rehash 的如下代码。

```C
/* Using dictEnableResize() / dictDisableResize() we make possible to
 * enable/disable resizing of the hash table as needed. This is very important
 * for Redis, as we use copy-on-write and don't want to move too much memory
 * around when there is a child performing saving operations.
 *
 * Note that even when dict_can_resize is set to 0, not all resizes are
 * prevented: a hash table is still allowed to grow if the ratio between
 * the number of elements and the buckets > dict_force_resize_ratio. */
static int dict_can_resize = 1;
static unsigned int dict_force_resize_ratio = 5;

static int _dictExpandIfNeeded(dict *d)
{
    /* Incremental rehashing already in progress. Return. */
    if (dictIsRehashing(d)) return DICT_OK;

    /* If the hash table is empty expand it to the initial size. */
    if (d->ht[0].size == 0) return dictExpand(d, DICT_HT_INITIAL_SIZE);

    /* If we reached the 1:1 ratio, and we are allowed to resize the hash
     * table (global setting) or we should avoid it but the ratio between
     * elements/buckets is over the "safe" threshold, we resize doubling
     * the number of buckets. */
    if (d->ht[0].used >= d->ht[0].size &&
        (dict_can_resize ||
         d->ht[0].used/d->ht[0].size > dict_force_resize_ratio))
    {
        return dictExpand(d, d->ht[0].used*2);
    }
    return DICT_OK;
}
```

很明显只有当 hash table 中键值对的数量大于 hash table 的大小的时候才会触发 rehash。这里 hash table 的大小指的就是 dictht 结构体中 table 这个数组的长度啦。简单的搜索可以发现，在 redis 代码中没有其它地方会修改 dict_force_resize_ratio 这个变量的值。而 dict_can_resize 却是有可能被修改的。唯一修改 dict_can_resize 这一变量的地方只有 dictEnableResize 和 dictDisableResize 这两个函数，而调用这两函数的只有 updateDictResizePolicy(server.c:736) 这一函数。那么首先查看下 updateDictResizePolicy 干了哪些事情吧。

```C
/* This function is called once a background process of some kind terminates,
 * as we want to avoid resizing the hash tables when there is a child in order
 * to play well with copy-on-write (otherwise when a resize happens lots of
 * memory pages are copied). The goal of this function is to update the ability
 * for dict.c to resize the hash tables accordingly to the fact we have o not
 * running childs. */
void updateDictResizePolicy(void) {
    if (server.rdb_child_pid == -1 && server.aof_child_pid == -1)
        dictEnableResize();
    else
        dictDisableResize();
}
```

很明显，只有当 redis 起了子进程去生成 RDB 文件或 AOF 文件的时候才会将 dict_can_resize 置 0(继续逆推代码也能印证这一点)。再回头看看触发 rehash 的条件判断，其实就是在当有子进程的时候 hash table 中键值对的数量与 hash table 的大小要更大才能允许进行 rehash。这是为什么呢？ updateDictResizePolicy 函数的注释对此进行了说明。在操作系统层面子进程是 COW 的，因此为了避免过多的内存拷贝作此处理。所以总结如下，触发条件只要满足如下任一一个即可

- 当前未在后台生成 rdb 文件或 aof 文件，只要比例大于等于 1
- 当前正在后台生成 rdb 文件或 aof 文件，只要比例大于 5

这里的比例指的是 hash table 中键值对的数量与 hash table 大小的比值。那么怎么才能在后台生成 rdb 文件或者 aof 文件呢？比如 redis 的定期备份，或者执行 BGSAVE/BGREWRITEAOF 则样的命令啦。

==注意==，这里只是从 dict 的角度去考察在哪些场景下会触发 rehash。但是实际上在 redis 中还有一个地方有可能会触发 dict 的 rehash。redis 有个定期任务会定期执行其中就会检查每个 db 中 dict 的大小，如果 dict 的利用率太低，就会调整 dict 的大小，从而隐式的触发 rehash。而且如果配置了 `activerehashing
 yes` 的话，就会对需要 rehash 的 dict 进行 rehash 操作。具体代码流程见 serverCron 中的 databasesCron 函数。serverCron 是在 initServer 中被注册到定时器列表中的(server.c:1884)。

### 具体操作

rehash 的操作总共就两步。

1. 生成一个更大的 hash table
2. 将原来 hash table 上面的数据重新 hash 到新的 hash table 上

在上面贴出来的代码中 `dictExpand(d, d->ht[0].used*2)` 很明显做的是第一步的事情，那么第二步呢？在 redis 中，这第二步藏在了每一个增查删操作中。试想一下，如果现在 hash table 里面已经有了几十万个键值对了，那么要将这几十万个键值对重新 hash 到新的 hash table 上要消耗多久时间。而且 redis 是单进程的，这样的操作只能在主线程做，那么就有可能会阻塞其它的命令处理导致有一段时间 redis 没法用。那对应这第二步的代码是啥呢？在增查删操作中，下面的代码就是干这事的。

```C
if (dictIsRehashing(d)) _dictRehashStep(d);
```

阅读代码可以发现，dict 结构体的 rehashidx 用于表示当前正在 rehash 的 hash table 的下标，-1 则表示当前没有 rehash 操作。每次 _dictRehashStep 都会将一部分键值对重新 hash 到新的 hash table 上。具体来说，就是 rehashidx 对应的这一链表的所有键值对。

这里需要注意的是，要执行 rehash 的第二步在 redis 中还需要有个条件就是当前没有迭代器在对 hash table 进行遍历。

```C
/* This function performs just a step of rehashing, and only if there are
 * no safe iterators bound to our hash table. When we have iterators in the
 * middle of a rehashing we can't mess with the two hash tables otherwise
 * some element can be missed or duplicated.
 *
 * This function is called by common lookup or update operations in the
 * dictionary so that the hash table automatically migrates from H1 to H2
 * while it is actively used. */
static void _dictRehashStep(dict *d) {
    if (d->iterators == 0) dictRehash(d,1);
}
```

## 其它杂项

### 键值对过期呢

redis 的 SET 命令长下面这样

```
SET key value [EX seconds] [PX milliseconds] [NX|XX]
```

这里多了控制键值对过期的参数，可是通过上面的分析完全没有涉及到 key 过期的处理。那如果执行包含了过期参数的 set 命令的话 redis 是怎么处理的呢？从 redisCommandTable (server.c:127)这个大数组可以看到 set 命令对应的处理函数是 setCommand。跟进去一路找就会发现在 setGenericCommand 函数里面有如下一行

```C
if (expire) setExpire(c,c->db,key,mstime()+milliseconds);
```

跟进去一看就明白了，原来代表数据库的 redisDb 这个结构体还有一个 dict 类型的成员变量 expires，如果 key 有过期时间的话，就把==到期==时间当作 value，存到 expires 里啦。

```C
/* Redis database representation. There are multiple databases identified
 * by integers from 0 (the default database) up to the max configured
 * database. The database number is the 'id' field in the structure. */
typedef struct redisDb {
    dict *dict;                 /* The keyspace for this DB */
    dict *expires;              /* Timeout of keys with a timeout set */
    dict *blocking_keys;        /* Keys with clients waiting for data (BLPOP)*/
    dict *ready_keys;           /* Blocked keys that received a PUSH */
    dict *watched_keys;         /* WATCHED keys for MULTI/EXEC CAS */
    int id;                     /* Database ID */
    long long avg_ttl;          /* Average TTL, just for stats */
} redisDb;
```

存的事情搞定了，那么什么时候删 key 呢？在 redis 中有两种策略删除过期的 key。

- redis 定期主动删除过期的 key
- GET 等命令的时候发现过期删除

很明显，第一种策略的存在是为了弥补第二种策略的不足。假设有一堆带有过期时间的键值对，但是始终没有 GET 请求。在这种场景下如果没有启用第一种策略的话，这堆过期的键值对将一直存在于内存中，白白浪费了空间。那么第一种策略的对应的代码在哪呢？该策略对应的函数是 activeExpireCycle。有两个地方调用了这个函数，一个是上面提到的 databasesCron，另外一处是 beforeSleep。这个 beforeSleep 是干啥的呢？看了 aeMain 函数就明白了。

activeExpireCycle 干的事也简单，就是遍历每一个 DB 随机找 key，看看是不是到期了，如果到期了就删除，否则就留着。只是每次不能删太多，会根据传入的参数设置删除 key 的个数、整体删除操作的耗时等限制。




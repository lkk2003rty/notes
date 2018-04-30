# redis 内存满了怎么办

资源总是有限的，那么对于 redis 来说要是内存满了或者超过了配置文件中设置的上限(maxmemory 配置项)该怎么办呢？一个显而易见的操作当然是删除没用的键值对。毕竟 redis 的键值对是可以设置超时的，如果在使用的内存超过限制的时候删除些到期的键值对(如果有的话)说不定就能让使用的内存不超标了。那如果不幸所有的键值对都没有超时时间的话又该如何处理呢？这次就来关注下这个问题。

首先看眼[配置文件](https://raw.githubusercontent.com/antirez/redis/4.0/redis.conf)内存超限后删键值对策略有如下几种：

- volatile-lru -> Evict using approximated LRU among the keys with an expire set.
- allkeys-lru -> Evict any key using approximated LRU.
- volatile-lfu -> Evict using approximated LFU among the keys with an expire set.
- allkeys-lfu -> Evict any key using approximated LFU.
- volatile-random -> Remove a random key among the ones with an expire set.
- allkeys-random -> Remove a random key, any key.
- volatile-ttl -> Remove the key with the nearest expire time (minor TTL)
- noeviction -> Don't evict anything, just return an error on write operations.

其中最好理解的就是 noeviction，说白了就是我就是不删，除非你告诉我删，新增一律报错。这也是默认的策略。而对于剩下的几种策略可以发现有的名字叫 volatile-XXX 有的则是 allkeys-XXX，在后续阅读完代码后可以得知 volatile-XXX 针对的是带有过期时间的键值对。换句话说，也就是如果要删除的话也只会删除有过期时间的键值对。而 allkeys-XXX 则是针对所有的键值对。如果抛开选择的键值对类型，那么删除策略也就如下几种：

- LRU
- LFU
- TTL
- random

其中，最好理解的就是 random，也就是从指定类型的键值对中随机选一个出来删除。LRU 呢就是选一个很久没被人用到的键值对删除。如果要做这件事情的话应该要维护一个双向链表，按最后一次使用的时间从近及远排个序，这样要删除的话只要删除链表尾指针指向的那个键值对就可以了。参考 leetcode 的[这一题](https://leetcode.com/problems/lru-cache/description/)。但是看一眼 redis 中用来表示一个数据库的结构体 redisDb 里面似乎并没有这样的链表存在，那 redis 是怎么实现 LRU 的呢？同样 LFU 也是类似的，只是 LFU 需要记录的是被使用的次数而不是最后一次被使用的时间。对于 redis，同样好奇是怎么实现的。TTL 亦是如此。总之，LRU/LFU/TTL 如果想要操作的时间复杂度低的话都需要额外的数据结构来存储相关信息，否则只能每次都傻乎乎的遍历所有的键值对来找出适合删除的那一对。

让我们暂时先不关注 redis 是怎么实现这些策略，先来考察下什么时候会触发 redis 删除操作。 

## 删除时机

一开始俺也不知道 redis 啥时候会执行删除策略，开始瞎蒙以为在分配内存的时候会检查下，如果超限了就执行策略，实际上并不是。于是只能老老实实从一个命令的执行过程开始跟，这才发现原来 redis 在处理每一个命令的时候都会去检查是否超限了是否要执行删除策略。具体来说，freeMemoryIfNeeded 这个函数就是执行删除策略的，只要在代码中 grep 这个函数调用的地方就是删除的时机啦。

## 策略执行

现在回头关注下 redis 是怎么实现 LRU/LFU/TTL 的，至于 allkeys-random 和 volatile-random 基本就没什么值得看的了。注意到 LRU 需要记录键值对最后被使用的时间，LFU 需要记录键值对被使用的次数，因此首先来看下这些数据是怎么存的。首先看一下键值对的结构体，长下面这样。

```C
typedef struct dictEntry {
    void *key;
    union {
        void *val;
        uint64_t u64;
        int64_t s64;
        double d;
    } v;
    struct dictEntry *next;
} dictEntry;
```

其中 key 是一个 void 类型的指针，value 也有可能是一个 void 类型的指针，看来还需要深扒下才知道实际上这 void 指针指向的是什么东东。于是开始跟命令处理流程。大概流程是这样的，main 函数调用 initServer。在 initServer 中注册了监听 socket 读事件的处理函数 acceptTcpHandler(server.c:1893)。acceptTcpHandler 调用 acceptCommonHandler，acceptCommonHandler 调用 createClient。在 createClient 中处理了连接 socket 的读事件处理函数 readQueryFromClient(networking.c:82)。readQueryFromClient 读取报文后调用 processInputBuffer 做进一步处理，processInputBuffer 开始分析报文内容了，由于 redis 的报文有两种类型，一种是批量操作的一种是单一操作的需要分别处理(networking.c:1314)，不过无论是哪种处理，redis 都是用下面这样的代码来处理参数的。这里 createObject 函数具体做了啥回头再看。

```C
c->argv[c->argc] = createObject(OBJ_STRING,argv[j]);
```

我们继续看 processInputBuffer 函数，解析完报文之后，调用 processCommand 处理这次请求的操作。在 processCommand 中我们就可以发现 freeMemoryIfNeeded 这个执行删除策略的函数，继续往下，可以发现最终调用 call(server.c:2471)去执行具体的操作。在 call 函数中，一句 `c->cmd->proc(c);` 就去干正事了。大概看眼 processCommand 函数就知道它其实就是根据参数去找对应的操作处理函数，然后一顿检查，没问题就调用 call 函数了。简单来说就是下面这两行代码。

```C
c->cmd = c->lastcmd = lookupCommand(c->argv[0]->ptr);
call(c,CMD_CALL_FULL);
```

再简单扒拉下就能够知道 call 里面调用的 c->cmd->proc 实际上就是全局变量 redisCommandTable 每个命令对应的处理函数啦。以 SET 命令为例，很快就能发现最后真正存进去的键值对，key 就是 `c->argv[1]`，value 是 `c->argv[2]`。这下可以回答上面的疑问了，key 这个 void 类型的指针实际上就是 `createObject(OBJ_STRING,argv[j])` 返回的东东啦。

那么就看眼 createObject 函数吧。

```C
#define LFU_INIT_VAL 5

typedef struct redisObject {
    unsigned type:4;
    unsigned encoding:4;
    unsigned lru:LRU_BITS; /* LRU time (relative to global lru_clock) or
                            * LFU data (least significant 8 bits frequency
                            * and most significant 16 bits decreas time). */
    int refcount;
    void *ptr;
} robj;

robj *createObject(int type, void *ptr) {
    robj *o = zmalloc(sizeof(*o));
    o->type = type;
    o->encoding = OBJ_ENCODING_RAW;
    o->ptr = ptr;
    o->refcount = 1;

    /* Set the LRU to the current lruclock (minutes resolution), or
     * alternatively the LFU counter. */
    if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
        o->lru = (LFUGetTimeInMinutes()<<8) | LFU_INIT_VAL;
    } else {
        o->lru = LRU_CLOCK();
    }
    return o;
}

/* This function is used to obtain the current LRU clock.
 * If the current resolution is lower than the frequency we refresh the
 * LRU clock (as it should be in production servers) we return the
 * precomputed value, otherwise we need to resort to a system call. */
unsigned int LRU_CLOCK(void) {
    unsigned int lruclock;
    if (1000/server.hz <= LRU_CLOCK_RESOLUTION) {
        atomicGet(server.lruclock,lruclock);
    } else {
        lruclock = getLRUClock();
    }
    return lruclock;
}

/* Return the current time in minutes, just taking the least significant
 * 16 bits. The returned time is suitable to be stored as LDT (last decrement
 * time) for the LFU implementation. */
unsigned long LFUGetTimeInMinutes(void) {
    return (server.unixtime/60) & 65535;
}
```

这下很容易就发现了，这个 lru 成员变量明显和删除策略有关系啊。但是其中用到了两个取时间的函数 LFUGetTimeInMinutes 和 LRU_CLOCK 分别对应 server.unixtime 和 server.lruclock 这两个值。所以在查看删除策略实现之前还是要先看一眼这两个值是怎么更新的。

## 时间更新

很明显能想到时间更新肯定是一个定期操作，于是直接看 serverCron 函数就是了。可以看到在 serverCron 中调用 updateCachedTime 更新了 server.unixtime 的值，调用 getLRUClock 更新了 server.lruclock 的值。那么还有个问题就是 serverCron 函数的调用频率是怎样的呢？在 serverCron 函数的最后可以看到 `return 1000/server.hz;`，而这返回值恰好就是下次调用 serverCron 要隔多少 ms。而 server.hz 来源于 hz 这一配置项。也就是说 hz 这一配置项的意思就是每 1000ms 更新几次时间。

除了对 server.unixtime 和 server.lruclock 这两个全局变量更新以外，如果访问一个已有的键值对，对 key 中的 lru 的值也是要更新的，以 GET 操作为例，具体代码如下。

```C
/* Low level key lookup API, not actually called directly from commands
 * implementations that should instead rely on lookupKeyRead(),
 * lookupKeyWrite() and lookupKeyReadWithFlags(). */
robj *lookupKey(redisDb *db, robj *key, int flags) {
    dictEntry *de = dictFind(db->dict,key->ptr);
    if (de) {
        robj *val = dictGetVal(de);

        /* Update the access time for the ageing algorithm.
         * Don't do it if we have a saving child, as this will trigger
         * a copy on write madness. */
        if (server.rdb_child_pid == -1 &&
            server.aof_child_pid == -1 &&
            !(flags & LOOKUP_NOTOUCH))
        {
            if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
                unsigned long ldt = val->lru >> 8;
                unsigned long counter = LFULogIncr(val->lru & 255);
                val->lru = (ldt << 8) | counter;
            } else {
                val->lru = LRU_CLOCK();
            }
        }
        return val;
    } else {
        return NULL;
    }
}

/* Logarithmically increment a counter. The greater is the current counter value
 * the less likely is that it gets really implemented. Saturate it at 255. */
uint8_t LFULogIncr(uint8_t counter) {
    if (counter == 255) return 255;
    double r = (double)rand()/RAND_MAX;
    double baseval = counter - LFU_INIT_VAL;
    if (baseval < 0) baseval = 0;
    double p = 1.0/(baseval*server.lfu_log_factor+1);
    if (r < p) counter++;
    return counter;
}
```

可以看到，对于 LRU/TTL 而言只要更新下 lru 的值为最新时间就是了，而 LFU 却比较复杂。下面再结合选删除键值对的流程具体分析。

## freeMemoryIfNeeded

背景知识基本都铺陈好了，可以直击核心了。瞄一眼 freeMemoryIfNeeded 可以粗略将其干的事可以总结成下面几步。

1. 是否超限？否啥事不干直接返回。
2. 计算超了多少，需要删多大才能满足要求
3. 根据不同的策略选对键值对出来，如果是 noeviction 那么直接到第 6 步
4. 删除键值对
5. 计算减少了多少内存，判断是否满足要求，不满足的话跳到第 3 步
6. 如果删除线程有待删键值对则等待，直到删除线程完成了任务链表中的所有任务

这里我们主要关注第 3 步即可。对于 LRU/LFU/TTL 而言选择键值对的流程如下：

1. 遍历每个 db 调用 evictionPoolPopulate 选出一堆候选的 key 存放在 pool 数组中
2. 从后往前遍历 pool 数组，排在最后面的就是要删除的 key

而 evictionPoolPopulate 又是怎么做的呢？从 db 中随机选取 server.maxmemory_samples (对应到 maxmemory-samples 配置项)个 key 放入 pool 数组中，并且保证在 pool 数组中有序的。这里有两个地方需要注意，其一选 key 的策略是随机的，其次是有序。先说第一个，选 key 调用的是 dictGetSomeKeys 函数，这个函数选 key 是随意选一个 bucket 然后把这个 bucket 里的 key 全搂出来，而不是反复调用 dictGetRandomKey 一次随机选一个 key 出来。当然关注细节的话这里需要考虑下当前 dict 正在 rehash 的情况。其次就是有序这事了。不同策略排序的依据肯定不一样了。具体到代码就是先算出每个 key 的 idle 值，然后按照 idle 值的从小到大排就是了。计算 idle 的代码如下。

```C
/* Calculate the idle time according to the policy. This is called
 * idle just because the code initially handled LRU, but is in fact
 * just a score where an higher score means better candidate. */
if (server.maxmemory_policy & MAXMEMORY_FLAG_LRU) {
    idle = estimateObjectIdleTime(o);
} else if (server.maxmemory_policy & MAXMEMORY_FLAG_LFU) {
    /* When we use an LRU policy, we sort the keys by idle time
     * so that we expire keys starting from greater idle time.
     * However when the policy is an LFU one, we have a frequency
     * estimation, and we want to evict keys with lower frequency
     * first. So inside the pool we put objects using the inverted
     * frequency subtracting the actual frequency to the maximum
     * frequency of 255. */
    idle = 255-LFUDecrAndReturn(o);
} else if (server.maxmemory_policy == MAXMEMORY_VOLATILE_TTL) {
    /* In this case the sooner the expire the better. */
    idle = ULLONG_MAX - (long)dictGetVal(de);
} else {
    serverPanic("Unknown eviction policy in evictionPoolPopulate()");
}
```

先来看 LRU。LRU 没啥说的就是按照键值对被使用的时间由近及远排列了。计算函数如下。

```C
/* Given an object returns the min number of milliseconds the object was never
 * requested, using an approximated LRU algorithm. */
unsigned long long estimateObjectIdleTime(robj *o) {
    unsigned long long lruclock = LRU_CLOCK();
    if (lruclock >= o->lru) {
        return (lruclock - o->lru) * LRU_CLOCK_RESOLUTION;
    } else {
        return (lruclock + (LRU_CLOCK_MAX - o->lru)) *
                    LRU_CLOCK_RESOLUTION;
    }
}
```

这里可能会有疑问，什么情况下会 `lruclock < o->lru` 呢？当然是溢出的情况下啦，lruclock 毕竟是不断变大的总有溢出的一天嘛。

再来看看 TTL 怎么处理的。简单粗暴啊，直接按照键值对 value 的值递减排就是了。因为是 TTL 嘛，所以这里选的键值对都是 db 里面 expires 存的键值对，值都是到期时间。恩，没毛病。

最后就是 LFU 了。对于 LFU 而言，key 中 lru 的值分为如下两部分。低 8 为实际上记录的是次数，而高 16 位记录的是 key 创建的时间。因此可以看到如果要严格的算的话，如果一个键值对被使用超过了 255 次，那么我们就没法记录其具体的次数了，毕竟只有 8 位可以用。但是在上面 GET 操作中对 lru 值的更新中可以看到，并不是每一次键值对被使用都会计算次数的。server.lfu_log_factor 会控制这次操作使用算不算数，server.lfu_log_factor 的值越大，一次操作被认为是算数的可能性越小。而 server.lfu_log_factor 恰恰是配置项 lfu-log-factor 设置的。

```
 *          16 bits      8 bits
 *     +----------------+--------+
 *     + Last decr time | LOG_C  |
 *     +----------------+--------+
```

再看眼怎么计算 idle 的，由于是 LFU 所以需要降序排列，因此 `idle = 255 - 使用次数`，但是仔细看一下这里使用次数是函数 LFUDecrAndReturn 计算的，它并不是直接把 lru 的低 8 位取出来就完事了。而是做了一步控制。如果直接把低 8 位取出来，那么我们计算的明显就是从 redis 启动以来到现在该键值对的使用次数。而在 LFUDecrAndReturn 函数中，它计算的是最近 server.lfu_decay_time 分钟内该键值对的使用次数。这里同样的 LFUTimeElapsed 处理了溢出的问题。

```C
/* If the object decrement time is reached, decrement the LFU counter and
 * update the decrement time field. Return the object frequency counter.
 *
 * This function is used in order to scan the dataset for the best object
 * to fit: as we check for the candidate, we incrementally decrement the
 * counter of the scanned objects if needed. */
#define LFU_DECR_INTERVAL 1

/* Given an object last decrement time, compute the minimum number of minutes
 * that elapsed since the last decrement. Handle overflow (ldt greater than
 * the current 16 bits minutes time) considering the time as wrapping
 * exactly once. */
unsigned long LFUTimeElapsed(unsigned long ldt) {
    unsigned long now = LFUGetTimeInMinutes();
    if (now >= ldt) return now-ldt;
    return 65535-ldt+now;
}

unsigned long LFUDecrAndReturn(robj *o) {
    unsigned long ldt = o->lru >> 8;
    unsigned long counter = o->lru & 255;
    if (LFUTimeElapsed(ldt) >= server.lfu_decay_time && counter) {
        if (counter > LFU_INIT_VAL*2) {
            counter /= 2;
            if (counter < LFU_INIT_VAL*2) counter = LFU_INIT_VAL*2;
        } else {
            counter--;
        }
        o->lru = (LFUGetTimeInMinutes()<<8) | counter;
    }
    return counter;
}
```

## 补充说明

在 evictionPoolPopulate 函数中会把一堆候选的 key 扔到 pool 这个数组中。这个数组的每个元素指向的结构体长下面这样。

```C
struct evictionPoolEntry {
    unsigned long long idle;    /* Object idle time (inverse frequency for LFU) */
    sds key;                    /* Key name. */
    sds cached;                 /* Cached SDS object for key name. */
    int dbid;                   /* Key DB number. */
};
```

注意到这里有个 cached 成员变量，这个 cached 是==复用==的。啥意思呢？也就是实际上 pool 这个数组在 evictionPoolAlloc(initServer 中调用)中就为 cached 分配好内存空间了，但是 key 的值却是 NULL。为啥呢？看眼 evictionPoolPopulate 函数中怎么赋值的就知道了。

```C
/* Try to reuse the cached SDS string allocated in the pool entry,
 * because allocating and deallocating this object is costly
 * (according to the profiler, not my fantasy. Remember:
 * premature optimizbla bla bla bla. */
int klen = sdslen(key);
if (klen > EVPOOL_CACHED_SDS_SIZE) {
    pool[k].key = sdsdup(key);
} else {
    memcpy(pool[k].cached,key,klen+1);
    sdssetlen(pool[k].cached,klen);
    pool[k].key = pool[k].cached;
}
```

看到了吧，如果 key 的长度不是很长的话，用的是 cached，这样避免了重复分配内存。看来为了省内存不遗余力啊。







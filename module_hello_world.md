# Redis 模块研究 - 第 1 回

[TOC]

## 缘起

想来至今没有完整的看过一个比较大型的开源项目，甚是失败。眼瞅着这半年过去，也没折腾出来啥，那干脆就好好利用下半年的时间努力看完一个开源项目的代码。那么选哪个项目好呢？首先列出了几个选择条件。

- 在业界被广泛使用的，也就是要接地气的
- 代码风格、注释比较全的
- 最好是 C/Python/Golang 的，因为其它语言真心不熟
- 涉及到分布式的

于是乎就想到了 Redis，那么就看它了。开动。。。。

## 上手

怎么上手呢？毕竟 Redis 的功能辣么多。想想旧的功能已经被各位大牛研究得差不多，那咱就从新功能开始吧，所以这次从 Redis 新加的模块功能开始。但是由于模块功能是 Redis 4.0 才会有的，但是 4.0 还出于 RC 状态，遂只能查看最近的代码了。以下所看的 Redis 代码 commit 为 `ab9d398835dca1187f190b28786cd9cc28e1fea1`，涉及到的开发测试环境为 `CentOS Linux release 7.3.1611`

### hello world

既然是看其模块怎么实现的，那么第一步当然就是自己写一个模块体验一下啦。模块咋写捏？在 Redis 代码的 `src/modules` 目录中就提供了编写模块的相关说明与例子，但是这 `helloworld.c` 实在是太长了，于是本菜鸟就精简了下，写个 `hello.c`，代码如下。

```C
#include "../redismodule.h"
#include <stdlib.h>
#include <time.h>

int HelloSimple_RedisCommand(RedisModuleCtx *ctx, RedisModuleString **argv, int argc) {
    REDISMODULE_NOT_USED(argv);
    REDISMODULE_NOT_USED(argc);
    srand(time(NULL));
    RedisModule_ReplyWithLongLong(ctx, rand());
    return REDISMODULE_OK;
}

int RedisModule_OnLoad(RedisModuleCtx *ctx, RedisModuleString **argv, int argc) {
    if (RedisModule_Init(ctx,"helloworld",1,REDISMODULE_APIVER_1)
        == REDISMODULE_ERR) return REDISMODULE_ERR;

    /* Log the list of parameters passing loading the module. */
    for (int j = 0; j < argc; j++) {
        const char *s = RedisModule_StringPtrLen(argv[j],NULL);
        printf("Module loaded with ARGV[%d] = %s\n", j, s);
    }

    if (RedisModule_CreateCommand(ctx,"hello",
        HelloSimple_RedisCommand,"random",0,0,0) == REDISMODULE_ERR)
        return REDISMODULE_ERR;

    return REDISMODULE_OK;
}
```
这代码就干两件事。

- 在 Redis 加载模块的时候打印配置后面跟着的参数
- 添加一个叫做 hello 的命令

原本 `src/module` 目录下就有 Makefile，在此基础上加上如下几行

```
hello.xo: ../redismodule.h

hello.so: hello.xo
        $(LD) -o $@ $< $(SHOBJ_LDFLAGS) $(LIBS) -lc
```
然后执行一把 `make hello.so` 咱这模块就编出来了。
模块编译完了就试一把吧。编辑 redis 配置文件加上这么一行。注意，这里为了观察需要配置 Redis 为非守护进程。

```
loadmodule /root/redis/src/modules/hello.so hello world
```

然后直接启动 Redis。就会发现在启动时打印出来了

```
Module loaded with ARGV[0] = hello
Module loaded with ARGV[1] = world
8380:M 30 Jun 06:09:23.527 * Module 'helloworld' loaded from /root/redis/src/modules/hello.so
```

看起来似乎咱这模块加载上了，这不把咱配置模块模块后面跟着的参数打印出来了。再连上 Redis 输入咱添加的命令 hello 试试。

```
[root@devlop redis]# redis-cli
127.0.0.1:6379> hello
(integer) 708870949
127.0.0.1:6379> hello
(integer) 1482576064
127.0.0.1:6379> hello
(integer) 1165567901
```

OK，没有问题。咱的 hello 模块运行良好。下面开始结合 Redis 代码走读下模块加载、运行的流程。

## 代码走读

拍屁屁想也知道模块嘛，无非是调用 Redis 提供的一套 API 进行操作。Redis 本身呢用 `dlopen` 加载模块然后在对应的 hook 调用模块里定义的函数。总体来说就是这个路数。

### Redis 代码 - 第一回

在 Redis 代码中是怎么处理模块的呢？咱顺着代码流程往下讲。

首先遇到的是 `moduleInitModulesSystem` 这个函数，位于 `server.c:3689`。这函数干了这么几件事儿。

1. 初始化 `server.loadmodule_queue`，这是一个链表 
2. 初始化 `modules` 这个 dict 类型那个的全局变量，将其相关信息设置为 `modulesDictType`。
3. 初始化 `server.moduleapi`，将其相关信息设置为 `moduleAPIDictType`。并用 `REGISTER_API` 这个宏添加了一堆键值对。
4. `pipe(server.module_blocked_pipe`，并设置为非阻塞。
5. 加锁 `pthread_mutex_lock(&moduleGIL)`。

这里有必要说一下 `REGISTER_API` 这个宏，比如 `REGISTER_API(Calloc)` 这样调用这个宏的话就会往 `server.moduleapi` 这个 hash 表里面插入一条 key 为 RedisModule_Calloc，value 为 RM_Calloc 的键值对。其中 key 为字符串，value 为函数指针。其实这里插入的 key 都是模块可以调用的 API 函数名。在后面会我们再结合 hello 模块看一下在模块中是怎么通过 API 函数真正调用到对应的函数的。比如，在 hello 模块中我们使用了 RedisModule_CreateCommand 这个 API 来添加一个新的命令，但是实际上它真正调用的是 RM_CreateCommand 这个函数。这是怎么发生的捏？后续揭晓。

然后呢就是解析配置这块了。Redis 既可以跟参数也有配置文件。同样的配置项目既可以在配置文件里面指定也可以在后面参数指定，那么那边为准呢？Redis 是这么操作的，先读配置文件内容到一字符串中，然后再把参数的配置项目添加到这字符串的后面，最后统一解析这字符串。***如此看来，如果在参数和配置文件中有同样配置项的以参数为准***。

对于配置项而言，与模块有关的只有一项 `loadmodule`。我们来看一下 Redis 是怎么处理的(`config.c:716`)。它直接把配置的模块信息封装成结构体 `moduleLoadQueueEntry` 添加到 `server.loadmodule_queue` 这一链表上。

继续往下就是加载模块了，位于 `server.c：3809`。这里依次遍历 `server.loadmodule_queue` 链表中每一项，dlopen 打开，调用各自模块的 `RedisModule_OnLoad` 方法。看，这里和咱 hello 模块的代码对上了。在调用完 `RedisModule_OnLoad` 方法之后，把模块相关信息添加到 `modules` 这个全局变量中并执行一些清理操作（这需要另起一篇写了）。通过这里可以看出如下几点

- `RedisModule_OnLoad` 是模块的入口，任何模块都***必须***实现这个函数
- 由于 Redis 后续会将模块相关信息添加到 `modules` 这个全局变量中，所以在 `RedisModule_OnLoad` 函数中一定需要对 `ctx->module` 做初始化。这个怎么做呢？调用 `RedisModule_Init` 函数。
- `RedisModule_OnLoad` 函数返回值***必须***是 REDISMODULE_OK，不然 Redis 就会认为模块加载失败，直接 `exit(1)` 退出了。

下面就是一些其它的设置，然后进入到 `epoll_wait` 等待请求过来啦。这里暂时先高一段落了。我们来看看咱们 `hello` 模块的代码。

### hello 模块

####  RedisModule_Init

咱们先来看看 `RedisModule_Init` 函数有啥名堂。上面说道这个函数对 `ctx->module` 做了初始化，这里咱也研究下它是怎么做的。

先说一下这个 `ctx` 是个什么鬼。`ctx` 是在 `module.c：3528` 定义的变量(值为 `REDISMODULE_CTX_INIT`)，然后作为第一个参数传递给 `RedisModule_OnLoad`。在 `hello` 模块中我们看到在 `RedisModule_OnLoad` 又把它传递给了 `RedisModule_Init`。

下面贴出来 `RedisModule_Init` 函数的实现，其中省略了部分代码。

```C
#define REDISMODULE_GET_API(name) \
     RedisModule_GetApi("RedisModule_" #name, ((void **)&RedisModule_ ## name))
     
#define REDISMODULE_API_FUNC(x) (*x)

void *REDISMODULE_API_FUNC(RedisModule_Alloc)(size_t bytes);

int REDISMODULE_API_FUNC(RedisModule_GetApi)(const char *, void *);

static int RedisModule_Init(RedisModuleCtx *ctx, const char *name, int ver, int apiver) {
     void *getapifuncptr = ((void**)ctx)[0];
     RedisModule_GetApi = (int (*)(const char *, void *)) (unsigned long)getapifuncptr;
     REDISMODULE_GET_API(Alloc);
     REDISMODULE_GET_API(Calloc);
     /* 省略部分代码，都是类似 REDISMODULE_GET_API(xxx) 这样的 */
     RedisModule_SetModuleAttribs(ctx,name,ver,apiver);
     return REDISMODULE_OK;
 }
```

可以看到 `RedisModule_GetApi` 是一个函数指针，其值为 ctx 这个结构体的第一个成员变量，翻阅 `REDISMODULE_CTX_INIT` 的定义便可发现其实就是 `RM_GetApi` 函数。下面紧跟着一堆 `REDISMODULE_GET_API(xxx)` 其实翻译一下就是调用 `RM_GetApi("RedisModule_xxx", (void **)&RedisModule_xxx)` 而这里的 `RedisModule_xxx` 一个个都是函数指针。那么很容易就能猜到 `RM_GetApi` 干的事情就是根据名字从 `server.moduleapi` 找到对应的函数复制给对应的函数指针。所以总结一下，这里干的事情就是给 `RedisModule_xxx` 类似这样的一堆函数指针赋值。其实这就是 Redis 暴露给我们模块的 API(哼，骗我们是 API，其实是函数指针)。

下面的 `RedisModule_SetModuleAttribs` 函数就是初始化 `ctx->module` 的内容啦，没啥好说的。

#### RedisModule_CreateCommand

`hello` 模块还使用 `RedisModule_CreateCommand` 这个 API 来添加新的命令，咱们来看看这是怎么做的。同样 `RedisModule_CreateCommand` 就是个函数指针，对应的函数就是 `RM_CreateCommand`。

熟悉 Redis 代码的话就知道 Redis 每个命令对应的处理函数等信息都在一个结构体数组(变量 redisCommandTable，位于`server.c:127`)中，然后被添加到 `server.commands` 这个 dict 中(main -> initServerConfig -> populateCommandTable)。后续处理请求的时候再根据命令查找相应的处理函数进一步调用。(入口函数为 acceptTcpHandler，这个函数调用 accept 并设置后续可读事件的回调函数为 readQueryFromClient， readQueryFromClient 里面的流程为 processInputBuffer -> processCommand -> lookupCommand -> call(c,CMD_CALL_FULL))。因此，`RM_CreateCommand` 肯定会往 `server.commands` 插入我们的命令的。


```
...
int RM_CreateCommand(RedisModuleCtx *ctx, const char *name, RedisModuleCmdFunc cmdfunc, const char *strflags, int firstkey, int lastkey, int keystep) {
    /* 省略部分代码 */
    /* Check if the command name is busy. */
    if (lookupCommand((char*)name) != NULL) {
        sdsfree(cmdname);
        return REDISMODULE_ERR;
    }
    /* 省略部分注释 */

    cp = zmalloc(sizeof(*cp));
    cp->module = ctx->module;
    cp->func = cmdfunc;
    cp->rediscmd = zmalloc(sizeof(*rediscmd));
    /* 省略部分代码 */
    cp->rediscmd->proc = RedisModuleCommandDispatcher;
    /* 省略部分代码 */
    dictAdd(server.commands,sdsdup(cmdname),cp->rediscmd);
    dictAdd(server.orig_commands,sdsdup(cmdname),cp->rediscmd);
    return REDISMODULE_OK;
}
```

可以看到在 `RM_CreateCommand` 中先进行了一系列检查比如 strflags 是否正确(这部分代码这里没有列出来)、命令的名字是否重复。然后就生成一个新的结构体保存相关信息，最后通过 `dictAdd(server.commands,sdsdup(cmdname),cp->rediscmd);` 这一行添加到了 `server.commands` 中。

但是，这里要注意的是添加到 `server.commands` 中的是 `cp->rediscmd` 也就是说当我们执行 hello 命令是调用的处理函数是 `RedisModuleCommandDispatcher`，而不是我们自己定义的 `HelloSimple_RedisCommand`。

那么，`RedisModuleCommandDispatcher` 函数要做了哪些事情呢？

```C
void RedisModuleCommandDispatcher(client *c) {
    RedisModuleCommandProxy *cp = (void*)(unsigned long)c->cmd->getkeys_proc;
    RedisModuleCtx ctx = REDISMODULE_CTX_INIT;

    ctx.module = cp->module;
    ctx.client = c;
    cp->func(&ctx,(void**)c->argv,c->argc);
    moduleHandlePropagationAfterCommandCallback(&ctx);
    moduleFreeContext(&ctx);
}
```

很明显，在 `RedisModuleCommandDispatcher` 中填充一下上下文，就直接通过 `cp->func(&ctx,(void**)c->argv,c->argc)` 调用到了我们的 `HelloSimple_RedisCommand` 函数了。

### RedisModule_ReplyWithLongLong

这个函数很明显就是用来给客户端返回数据的，查看下 `RM_ReplyWithLongLong` 函数就发现其实调用的是 `addReplyLongLong`。但是注意这里并没有真正的给客户端写数据哦，因为还没注册写事件的回调函数。这个回调函数是在 `beforeSleep` 里面调用 `handleClientsWithPendingWrites` 给加上的。

## 总结

至此，Redis 模块的编写，其中涉及到的大致流程也比较清晰了，接下来可以进一步分析其中的细节了，比如 `RedisModule_CreateCommand` 中作为参数的 `strflags` 都启什么作用，在模块想知道 Redis 的数据要调用哪些 API 等。




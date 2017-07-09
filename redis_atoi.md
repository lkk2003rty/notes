# Redis 模块研究 - 第 1 回 SP

[TOC]

##  缘起

在上一回中我们通过一个 hello 模块初步了解了怎么写一个 Redis 模块，但是在仔细扒代码的过程中还发现了一个 Redis 一个优化的地方，故在这里记录下。

在 hello 模块中我们调用 `RedisModule_ReplyWithLongLong(ctx, rand())` 来将生成的随机数返回给客户端。注意到这里 `rand` 函数的返回值是 int 类型的，但是我们都知道往 socket fd 写数据肯定传的都是一个字符串。所以，这里必然涉及到一个类似 itoa 的函数将数字转换成字符串。这次我们就来看看 Redis 里面是怎么做 itoa 的。

## 普通做法

在看 Redis 里面的实现之前，先想想如果是我们来实现的话会怎么写这个函数(貌似这也是一个经典的面试题)。其算法无比简单，无非就是先 malloc 一定长度的 char 数组，然后对数字求 10 的余数，将余数转成对应的字符填充到 char 数组，最后将数组一翻转完事。其中还需要考虑的数字正负问题，有可能最后还需要填上负号。

## Redis 中的做法

Redis 中对应的代码为 `ll2string` 函数。他的步骤其实也不复杂，就下面几步

- 判断是否为负数，并转成无符号的类型。
- 求出数字长度，比如数字是 999，那么长度就是 3。这里的数字用的上一步转成的无符号的。
- 将数字每次求 100 的余数，每次填充两个字符到数组中。注意这里是从后往前填写，因此最后不需要翻转。
- 如果需要的话，填上负号。

我们再来看一下第 1 步中求数字长度是怎么做的。普通做法无非将数字不断除以 10 直到结果为 0 为止，得到的除以 10 的次数即为数字的长度。但是 Redis 却是根据不同的数字大小走不同的 if-else 分支，返回对应的值，代码如下。

```C
/* Return the number of digits of 'v' when converted to string in radix 10.
 * See ll2string() for more information. */
uint32_t digits10(uint64_t v) {
    if (v < 10) return 1;
    if (v < 100) return 2;
    if (v < 1000) return 3;
    if (v < 1000000000000UL) {
        if (v < 100000000UL) {
            if (v < 1000000) {
                if (v < 10000) return 4;
                return 5 + (v >= 100000);
            }
            return 7 + (v >= 10000000UL);
        }
        if (v < 10000000000UL) {
            return 9 + (v >= 1000000000UL);
        }
        return 11 + (v >= 100000000000UL);
    }
    return 12 + digits10(v / 1000000000000UL);
}
```

因此，可以看到在 Redis 版的 atoi 实际上有两个优化的地方

- 计算数字长度用 if-else 逻辑的替代
- 转成字符串的时候每次除以 100，根据余数查表填充字符

为什么要设计成这样呢，难不成这有啥黑魔法不成。在 `ll2string` 函数的注释中说明这样的算法来源于 [Three Optimization Tips for C++](https://www.facebook.com/notes/facebook-engineering/three-optimization-tips-for-c/10151361643253920) 这篇文章。

## 寻根问底

链接中的文章主要说了几个优化的技巧，其中对于 atoi 有用的有这么几点。

### 使用更快速的操作

不同的操作速度是不一样的，文章中给出的由快到慢分别是

-	comparisons
-	(u)int add, subtract, bitops, shift
-	floating point add, sub (separate unit!)
-	indexed array access (caveat: cache effects)
-	(u)int32 mul
-	FP mul
-	FP division, remainder
-	(u)int division, remainder

其中 FP 指的是 float point。这样就不难理解了为什么求数字长度要用一堆 if-else 而不是普通的不断除以 10 的做法了。一方面，比较是最快的操作，另一方面使用比较的话，对于某些数字能减少操作的个数，比如对于一个长度为 11 的数字，只要 6 次比较就能知道长度了。

不过得吐槽下，这排名是咋出来的？难道是翻 Intel Architectures Developer's Manual 翻出来的？

### 减少写次数

在普通版 atoi 中涉及到写的操作有两次，第 1 次是往 char 数组填充字符，第 2 次是翻转。而由于写操作的对象是数组，也就是说这不是直接修改寄存器中值的操作，因此是一个比较慢的操作，因此我们要想法设法避免多余的写操作。在普通版中唯一能避免的就是翻转时候涉及到的操作了。但是为此付出的代价就是预先计算出数字的长度。其次，在上面列出来的操作速度排名中也看到了整数的除法是最慢的，因此要尽量少用。所以在 Redis 版的 atoi 中每次都是除以 100 而不是除以 10。不过，按照这么说的话是不是可以除以 1000，然后加大打表的长度？可能是为了写代码方便所以除以 100 吧，不然得先写一个有 1000 个元素的表也是件麻烦事啊。

## 总结

至此，Redis 版的 aoti 就分析完了，可以看到所谓的优化还是需要对计算机有深刻的理解的啊。



# gofail 学习笔记

## 缘起

花了一周时间看完了[Chaos Engineering](https://book.douban.com/subject/27075046/)，对 Chaos Engineering 有了个初步的了解。其实说白了所谓的 Chaos Engineering 就是在系统中“捣乱”，检查系统在各种异常情况下的行为，从而发现潜在的 bug，同时也让开 RD & OP 知晓怎么排查问题，怎么处理。只有经过 Chaos Engineering 检验过的系统才能够让 RD & OP 有足够的自信部署上线。

其中 Chaos Engineering 必做的一步操作就是 fail injection。而 fail injection 又有多个层面，网络、IO、代码都可以做 fail injection。网络方面可以通过 tc、iptables、systemTap 等控制（丢包、延迟、乱序），IO 方面可以用 [libfuse](https://github.com/libfuse/libfuse) 等模拟（读写失败，读写 delay），至于代码嘛就可以用本文所说的 [gofail](https://github.com/coreos/gofail) 来操作了。当然了，看这名字就知道这货只适用于 Go 代码。

gofail 其实是一个 golang 的库。它实现 fail injection 的方式也很简单。首先，在需要 fail injection 的地方写上注释以及注入之后执行的代码。其次，通过 gofail 这一工具找到注释的地方，将它替换成其它代码，同时生成一个 go 文件只包含注入点的信息。
最后，正常编译运行即可。只是运行的时候可以通过环境变量激活特点的注入点，从而执行注入的代码，最终实现 fail injection。下面就结合 gofail 的代码来学习下。

## gofail 使用

gofail 本身就带有一个示例，位于项目的 examples 目录下，只是它的 README 文档写的是错误的。第一步 gofail enable 后面要跟的是包路径，这里可以写成`gofail enable example`。剩余部分就没有其它需要更正的地方了。下面都以这个作为示例进行分析。

## gofail 源码分析

从上面的步骤中可以看到 gofail 的使用就两步，第一步做的事情是代码的修改生成，第二步做的事是根据环境变量要么猫敲的启动一个 http server 要么直接修改注入错误。下面就分开来分析。

### 代码修改

代码修改主要是 `gofail enable|disable <path>` 做的事，其中 enable 是把原来的代码改成注入之后的代码，disable 则是还原成原来的代码。这里需要注意的是 enable 不仅是修改代码也会生成一个新的代码文件。比如 examples.go 中有需要注入的代码注释，那么 gofail 不仅会修改 examples.go 这个文件，还会在同目录下生成一个名为 examples.fail.go 的文件。需要注入的代码注释长下面这样。

```go
	// gofail: var ExampleString string
	// return ExampleString

```

`// gofail: var ExampleString string` 这行是必须的，但是类型 string 则是随你喜好只要是 Go 的类型就行了，`// return ExampleString` 这行就随便写了可以有多行也可以没有，都行。执行完了 `gofail enable example` 之后就会发现上面的这段注释被修改成了下面这样，同时还生成了 examples.fail.go 文件。

```go
     if vExampleString, __fpErr := __fp_ExampleString.Acquire(); __fpErr == nil { defer __fp_ExampleString.Release(); ExampleString, __fpTypeOK := vExampleString.(string); if !__fpTypeOK { goto            __badTypeExampleString}
         return ExampleString; __badTypeExampleString: __fp_ExampleString.BadType(vExampleString, "string"); };
```

其中 `__fp_ExampleString` 恰恰就是生成的 examples.fail.go 文件中定义的变量。

```go
 import "github.com/coreos/gofail/runtime"

 var __fp_ExampleString *runtime.Failpoint = runtime.NewFailpoint("github.com/coreos/gofail/examples", "ExampleString")
```
gofail 的 main 函数就位于 gofail.go 文件中，流程也清晰没啥好说的，找到注入点修改代码就是了。

### 行为控制

根据上面的替换结果很明显只需要查看下 `github.com/coreos/gofail/runtime` 这个库干了哪些事情以及 `runtime.Failpoint` 这一类结构体对应的 `Acquire`、`Release`、`BadType` 这三方法就行了。

首先是 `github.com/coreos/gofail/runtime` 这个库，注意这个库有 init 方法。init 方法做了两件事情，如果有设置环境变量 `GOFAIL_HTTP` 那么启动 http server，如果设置了环境变量 `GOFAIL_FAILPOINTS`，按照 `；` 拆分成多项，每项按照 `=` 拆分成 key 和 value 存入全局变量 envTerms 中。

接下来，来看看 `runtime.Failpoint` 相关的代码。`runtime.Failpoint` 这一结构体长下面这样。

```
 type Failpoint struct {
     mu sync.RWMutex
     t  *terms
 }
 
  // terms encodes the state for a failpoint term string (see fail(9) for examples)
 // <fp> :: <term> ( "->" <term> )*
 type terms struct {
     // chain is a slice of all the terms from desc
     chain []*term
     // desc is the full term given for the failpoint
     desc string
     // fpath is the failpoint path for these terms
     fpath string

     // mu protects the state of the terms chain
     mu sync.Mutex
 }
 
  // term is an executable unit of the failpoint terms chain
 type term struct {
     desc string

     mods mod
     act  actFunc
     val  interface{}

     parent *terms
 }

 type mod interface {
     allow() bool
 }
 
 type actFunc func(*term) interface{}

 var actMap = map[string]actFunc{
     "off":    actOff,
     "return": actReturn,
     "sleep":  actSleep,
     "panic":  actPanic,
     "break":  actBreak,
     "print":  actPrint,
 }
```

那么，让我们来看一下啊始化一个 `runtime.Failpoint` 实例的 `NewFailpoint` 方法都干了些什么吧。其实很简单，从全局变量 envTerms 中找到设置的值，解析后创建 `runtime.Failpoint` 实例，存入全局变量 failpoints 中。通过代码分析可以看到可设置的 fail injection 动作的语法如下。

```
<fp> :: <term> ( "->" <term> )*
<term> :: <mod> <act> [ "(" <val> ")" ]
<mod> :: ((<float> "%")|(<int> "*" ))*
<act> :: "off" | "return" | "sleep" | "panic" | "break" | "print"
<val> :: <int> | <string> | <bool> | <nothing>
```

翻译成人话就是你可以写成注入下面这样的语句出来

```
50.0%return("hello")comment->8*sleep(1000)
```

这其中的 comment 会被当成字符串存入到 `term` 这一结构体的 desc 这一成员变量中。以上还有个地方要注意 50.0% 中的小数点不能少，代码里面就是这样做 parse 的。

结合这里罗列出来的代码就能猜出来了，按照 `->` 拆分字符串变成一个个 `term`， 每个 `term` 对应一项操作。对每一项操作而言有其对应的激活条件以及激活后执行的动作。激活条件由这里的 `50.0%` 和 `8*` 来控制，执行的动作则是 `return("hello")` 这样的语句。那我们来看一下 `runtime.Failpoint` 结构体对应的 `Acquire` 方法验证下是不是这样的呢？果然是这样的。`Acquire` 方法中会遍历这一系列操作，只要有一个可以满足激活条件的，那么就执行对应的操作。如果有多个都能满足的话就按照顺序，先满足的执行，排在后面的就不管了。那么对应的这些 `return` 等一系列操作都代表啥意思呢？进一步查看了下代码，总结如下。

```
off： 注入不生效，继续走原有代码流程
print： 打印一串字符串，继续走原有代码流程
panic： 直接调用 Go 的 panic 函数
sleep： sleep 对应的时间，然后继续走原有代码流程。这个跟的参数可以是数字和字符串，数字的话单位是毫秒，字符串的话就调用 time.ParseDuration 去解析拉
return： 返回后面跟的参数的值，走注入的代码流程
break：用 exec.Command 执行 gdb <execute> <pid>，其中 <execute> 和 <pid> 分别是自身可执行程序和进程 pid。
```

对了，这里还漏了说明上面例子中的 `50.0%` 和 `8*` 分别代表啥意思，在怎样的条件下才会执行后面的操作。`50.0%` 其实就是指概率，有百分之几的可能性执行后面的操作。`8*` 则表示只执行 8 次，以后就再也不执行了。

至此，gofail 的总体代码就看完了。








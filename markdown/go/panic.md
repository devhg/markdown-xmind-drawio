# 深度细节 | Go 的 panic 的三种诞生方式



https://mp.weixin.qq.com/s/sGdTVSRxqxIezdlEASB39A

![图片](https://mmbiz.qpic.cn/mmbiz_png/4UtmXsuLoNdtbJoXjWrOTJUKsljrickDho4nf1XZLSPTmmDZc01s8crUexw4cgeXAMkITvzZ0lPVX3fic303cMpA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)







## 1. 为什么 panic 值得思考？

初学 Go 的时候，心里常常很多疑问，有时候看似懂了的问题，其实是是而非。

panic 究竟是啥？看似显而易见的问题，但是却回答不出个所以然来。奇伢分两个章节来彻底搞懂 panic 的知识：

- 姿势篇：摸清楚 panic 的诞生，它不是石头里蹦出来的，总结有三种姿势；
- 原理篇：彻底搞明白 panic 的内部原理，理解 panic 的深层原理；



## 2. panic 的三种姿势

什么时候会产生 panic ？

我们先从“形”来学习。从程序猿的角度来看，可以分为主动和被动方式，被动的方式有两种，如下：

**主动方式**：

- 程序猿主动调用 `panic( )` 函数；

**被动的方式**：

- 编译器的隐藏代码触发；
- 内核发送给进程信号触发 ；



### 2.1 编译器的隐藏代码



Go 之所以简单又强大，编译器居功至伟。非常多的事情是编译器帮程序猿做了的，逻辑补充，内存的逃逸分析等等。

**包括 panic 的抛出！**

举个非常典型的例子：整数算法**除零**会发生 panic，怎么做到的？

看一段极简代码：

```go
func divzero(a, b int) int {
    c := a/b
    return c
}
```

上面函数就会有除零的风险，当 b 等于 0 的时候，程序就会触发  panic，然后退出，如下：

```
root@ubuntu:~/code/gopher/src/panic# ./test_zero 

panic: runtime error: integer divide by zero

goroutine 1 [running]:
main.zero(0x64, 0x0, 0x0)
 /root/code/gopher/src/panic/test_zero.go:6 +0x52
```

**问题来了：程序怎么触发的 panic ？**

**代码面前无秘密。**

可代码看不出啥呀，不就是一行 `c := a/b` 嘛？

奇伢说的是**汇编代码**。**因为这段隐藏起来的逻辑，是编译器帮你加的。**

用 dlv 调试断点到 `divzero` 函数，然后执行 `disassemble` ，你就能看到秘密了。奇伢截取部分汇编，并备注了下：

```shell
 (dlv) disassemble
TEXT main.zero(SB) /root/code/gopher/src/panic/test_zero.go

    // 判断 b 是否等于 0 
    test_zero.go:6  0x4aa3c1    4885c9          test rcx, rcx
    // 不等于 0 就跳转到 0x4aa3c8 执行指令，否则就往下执行
    test_zero.go:6  0x4aa3c4    7502            jnz 0x4aa3c8
    // 执行到这里，就说明 b 是 0 值，就跳转到 0x4aa3ed ，也就是 call $runtime.panicdivide
=>  test_zero.go:6  0x4aa3c6    eb25            jmp 0x4aa3ed
    test_zero.go:6  0x4aa3c8    4883f9ff        cmp rcx, -0x1
    test_zero.go:6  0x4aa3cc    7407            jz 0x4aa3d5
    test_zero.go:6  0x4aa3ce    4899            cqo
    test_zero.go:6  0x4aa3d0    48f7f9          idiv rcx
    // ...
    test_zero.go:7  0x4aa3ec    c3              ret
    // 看到神奇的函数了嘛 ！
    test_zero.go:6  0x4aa3ed    e8ee27f8ff      call $runtime.panicdivide
```

**编译器偷偷加上了**一段 `if/else` 的判断逻辑，并且还给加了 `runtime.panicdivide` 的代码。

1. 如果 b == 0 ，那么跳转执行函数 `runtime.panicdivide` ；

再来看一眼 `panicdivide` 函数，这是一段极简的封装：

```
// runtime/panic.go
func panicdivide() {
    panicCheck2("integer divide by zero")
    panic(divideError)
}
```

看到了不，这里面调用的就是 `panic()` 函数。

除零触发的 panic 就是这样来的，它不是石头里蹦出来的，而是编译器多加的逻辑判断保证了除数为 0 的时候，触发 panic 函数。

**划重点：编译器加的隐藏逻辑，调用了抛出 panic 的函数。Go 的编译器才是真大佬！**



### 2.2 进程信号触发



最典型的是**非法地址访问**，比如， nil 指针 访问会触发 panic，怎么做到的？

看一个极简的例子：

```go
func nilptr(b *int) int {
    c := *b
    return c
}
```

当调用 `nilptr( nil )` 的时候，将会导致进程异常退出：

```
root@ubuntu:~/code/gopher/src/panic# ./test_nil 

panic: runtime error: invalid memory address or nil pointer dereference
[signal SIGSEGV: segmentation violation code=0x1 addr=0x0 pc=0x4aa3bc]

goroutine 1 [running]:
main.nilptr(0x0, 0x0)
 /root/code/gopher/src/panic/test_nil.go:6 +0x1c
```

**问题来了：这里的 panic 又是怎么形成的呢？**

在 Go 进程启动的时候会注册默认的信号处理程序（ `sigtramp` ）

在 cpu 访问到 0 地址会触发 `page fault` 异常，这是一个非法地址，内核会发送 `SIGSEGV` 信号给进程，所以当收到 `SIGSEGV` 信号的时候，就会让 `sigtramp` 函数来处理，最终调用到 `panic` 函数 ：

```
// 信号处理函数回调
sigtramp （纯汇编代码）
  -> sigtrampgo （ signal_unix.go ）
    -> sighandler  （ signal_sighandler.go ）
       -> preparePanic （ signal_amd64x.go ）

          -> sigpanic （ signal_unix.go ）
            -> panicmem 
              -> panic (内存段错误)
```

在 `sigpanic` 函数中会调用到 `panicmem` ，在这个里面就会调用 panic 函数，从而走上了 Go 自己的 panic 之路。

`panicmem` 和 `panicdivide` 类似，都是对 `panic()` 的极简封装：

```go
func panicmem() {
    panicCheck2("invalid memory address or nil pointer dereference")
    panic(memoryError)
}
```

**划重点：这种方式是通过信号软中断的方式来走到 Go 注册的信号处理逻辑，从而调用到 `panic()` 的函数。**

童鞋可能会好奇，信号处理的逻辑什么时候注册进去的？

在进程初始化的时候，创建 M0（线程）的时候用系统调用 `sigaction` 给信号注册处理函数为 `sigtramp` ，调用栈如下：

```
mstartm0 （proc.go）
    -> initsig (signal_unix.go:113)
        -> setsig （os_linux.go）
```

这样的话，以后触发了信号软中断，就能调用到 Go 的信号处理函数，从而进行语言层面的 panic 处理 。

总的来说，这个是从系统层面到特定语言层面的处理转变。



### 2.3 程序猿主动



第三种方式，就是程序猿自己主动调用 `panic` 抛出来的。

```
func main() {
    panic("panic test")
}
```

简单的函数调用，这个超简单的。



## 3. 聊聊 panic 到底是什么？



现在我们摸透了 panic 产生的姿势，以上三种方式，无论哪一种都归一到 `panic( )` 这个函数调用。**所以有一点很明确：panic 这个东西是语言层面的处理逻辑。**

![图片](https://mmbiz.qpic.cn/mmbiz_png/4UtmXsuLoNdtbJoXjWrOTJUKsljrickDh6fvZXp0cuwibNCQ7pN1d68qHvkqG8x6O6VcpsyHf2Q4xia2v15iaicMp2w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



panic 发生之后，如果 Go 不做任何特殊处理，默认行为是**打印堆栈，退出程序**。

现在回到最本源的问题：panic 到底是什么？

这里不纠结概念，只描述几个简单的事实：

1. `panic()` 函数内部会产生一个关键的数据结构体 `_panic` ，并且挂接到 goroutine 之上；
2. `panic()` 函数内部会执行 `_defer` 函数链条，并针对 `_panic` 的状态进行**对应的处理**；

什么叫做 `panic()` 的对应的处理？

循环执行 goroutine 上面的 `_defer` 函数链，如果执行完了都还没有恢复 `_panic` 的状态，那就没得办法了，**退出进程，打印堆栈**。

如果在 goroutine 的 `_defer` 链上，有个朋友 `recover()` 了一下，把这个 `_panic` 标记成恢复，那事情就到此为止，就从这个 `_defer` 函数执行后续正常代码即可，走 `deferreturn` 的逻辑。

**所以，panic 是什么 ？**

**它就是个特殊函数调用，仅此而已。**



几个问题

- panic 究竟是啥？是一个结构体？还是一个函数？
- 为什么 panic 会让 Go 进程退出的 ？
- 为什么 recover 一定要放在 defer 里面才生效？
- 为什么 recover 已经放在 defer 里面，但是进程还是没有恢复？
- 为什么 panic 之后，还能再 panic ？有啥影响？





总结

1. panic 产生的三大姿势：**程序猿主动，编译器辅助逻辑，软中断信号触发**；
2. 无论哪一种姿势，最终都是归一到 `panic()` 函数的处理，**panic 只是语言层面的处理逻辑**；
3. panic 发生之后，如果不做处理，默认行为是打印 panic 原因，打印堆栈，进程退出；







<hr>







# 深度细节 | Go 的 panic 的秘密都在这



![图片](https://mmbiz.qpic.cn/mmbiz_png/4UtmXsuLoNdthCmFP2fSGlB6kCkOgRGLY2TSibVYqtHQROXh2PQGNTVWMgQiaMIBHNakR7rPOTyuT8icztjQcZdMQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



## 1. 前情提要



关于 panic 的时机，在上篇 [深度细节 | Go 的 panic 的三种诞生方式](http://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247493867&idx=1&sn=9fef8e55b8220976d6362658d5188618&chksm=cf3df82ef84a7138f21b6fea9e719b204b0f7adaab5632cc8aff8b6966c985300d4af2a71bcd&scene=21#wechat_redirect) 对 panic 总结三种诞生方式：

- 程序猿主动：调用 `panic( )` 函数；
- 编译器的隐藏代码：比如除零场景；
- 内核发送给进程信号：比如非法地址访问 ；

**三种都归一到** **`panic( )` 函数的调用，指出 Go 的 panic 只是一个特殊的函数调用，是语言层面的处理。**初学 Go 的时候，奇伢心里也常常有些疑问：

- panic 究竟是啥？是一个结构体？还是一个函数？
- 为什么 panic 会让 Go 进程退出的 ？
- 为什么 recover 一定要放在 defer 里面才生效？
- 为什么 recover 已经放在 defer 里面，但是进程还是没有恢复？
- 为什么 panic 之后，还能再 panic ？有啥影响？

今天便是深入到代码原理，明确解答以上问题。Go 源码版本声明

> Go 1.13.5





## 2. _panic 数据结构



看看 `_panic` 的数据结构：

```go
// runtime/runtime2.go
// 关键结构体
type _panic struct {
    argp      unsafe.Pointer
    arg       interface{}    // panic 的参数
    link      *_panic        // 链接下一个 panic 结构体
    recovered bool           // 是否恢复，到此为止？
    aborted   bool           // the panic was aborted
}
```

**重点字段关注**：

- `link` 字段：一个指向 `_panic` 结构体的指针，表明 `_panic` 和 `_defer` 类似，`_panic` 可以是一个单向链表，就跟 `_defer` 链表一样；
- `recovered` 字段：重点来了，所谓的 `_panic` 是否恢复其实就是看这个字段是否为 true，`recover( )` 其实就是修改这个字段；

![图片](https://mmbiz.qpic.cn/mmbiz_png/4UtmXsuLoNdthCmFP2fSGlB6kCkOgRGLZhwXTx2kQjwFsh0GzeNM7sUCL8JkMVZQ0XQWdUtv8hia6De5869YeRw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



再看一下 goroutine 的两个重要字段：

```go
type g struct {
    // ...
    _panic         *_panic // panic 链表，这是最里的一个
    _defer         *_defer // defer 链表，这是最里的一个；
    // ...
}
```

从这里我们看出：`_defer` 和 `_panic` 链表都是挂在 goroutine 之上的。什么时候会导致 `_panic` 链表上多个元素？

**`panic()` 的流程下，又调用了 `panic()` 函数。**

这里有个细节要注意了，怎么才能做到 `panic()` 流程里面再次调用 `panic()` ？

**划重点：只能是在 defer 函数上，才有可能形成一个 `_panic` 链表。因为 `panic( )` 函数内只会执行 `_defer` 函数 ！**



## 3. recover 函数



为了方便讲解，我们由简单的开始分析，先看 recover 函数究竟做了什么？

```go
defer func() {
    recover()
}()
```

`recover` 对应了 `runtime/panic.go` 中的 `gorecover` 函数实现。



 **gorecover 函数**

```go
func gorecover(argp uintptr) interface{} {
    // 只处理 gp._panic 链表最新的这个 _panic；
    gp := getg()
    p := gp._panic
    if p != nil && !p.recovered && argp == uintptr(p.argp) {
        p.recovered = true
        return p.arg
    }
    return nil
}
```



这个函数可太简单了：

1. 取出当前 goroutine 结构体；
2. 取出当前 goroutine 的 `_panic` 链表最新的一个 `_panic`，如果是非 nil 值，则进行处理；
3. 该 `_panic` 结构体的 `recovered` 赋值 true，程序返回；

这就是 recover 函数的全部内容，只给 `_panic.recovered` 赋值而已，不涉及代码的神奇跳转。而 `_panic.recovered` 的赋值是在 `panic` 函数逻辑中发挥作用。



![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

## 4. panic 函数



panic 的实现在一个叫做 gopanic 的函数，位于 `runtime/panic.go` 文件。panic 机制最重要最重要的就是 gopanic 函数了，所有的 panic 细节尽在此。为什么 panic 会显得晦涩，主要有两个点：

1. 嵌套 panic 的时候，gopanic 会有**递归执行**的场景；
2. 程序指令跳转并不是常规的函数压栈，弹栈，在 recovery 的时候，是**直接修改指令寄存器的结构体**，从而直接越过了 gopanic 后面的逻辑，甚至是多层 gopanic 递归的逻辑；

**一切秘密都在下面这个函数：**

```go
// runtime/panic.go
func gopanic(e interface{}) {
    // 在栈上分配一个 _panic 结构体
    var p _panic
    // 把当前最新的 _panic 挂到链表最前面
    p.link = gp._panic
    gp._panic = (*_panic)(noescape(unsafe.Pointer(&p)))
    
    for {
        // 取出当前最近的 defer 函数；
        d := gp._defer
        if d == nil {
            // 如果没有 defer ，那就没有 recover 的时机，只能跳到循环外，退出进程了；
            break
        }

        // 进到这个逻辑，那说明了之前是有 panic 了，现在又有 panic 发生，这里一定处于递归之中；
        if d.started {
            if d._panic != nil {
                d._panic.aborted = true
            }
            // 把这个 defer 从链表中摘掉；
            gp._defer = d.link
            freedefer(d)
            continue
        }

        // 标记 _defer 为 started = true （panic 递归的时候有用）
        d.started = true
        // 记录当前 _defer 对应的 panic
        d._panic = (*_panic)(noescape(unsafe.Pointer(&p)))

        // 执行 defer 函数
        reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))

        // defer 执行完成，把这个 defer 从链表里摘掉；
        gp._defer = d.link
        
        // 取出 pc，sp 寄存器的值；
        pc := d.pc
        sp := unsafe.Pointer(d.sp)
        // 如果 _panic 被设置成恢复，那么到此为止；
        if p.recovered {
            // 摘掉当前的 _panic
            gp._panic = p.link
            // 如果前面还有 panic，并且是标记了 aborted 的，那么也摘掉；
            for gp._panic != nil && gp._panic.aborted {
                gp._panic = gp._panic.link
            }
            // panic 的流程到此为止，恢复到业务函数堆栈上执行代码；
            gp.sigcode0 = uintptr(sp)
            gp.sigcode1 = pc
            // 注意：恢复的时候 panic 函数将从此处跳出，本 gopanic 调用结束，后面的代码永远都不会执行。
            mcall(recovery)
            throw("recovery failed") // mcall should not return
        }
    }

    // 打印错误信息和堆栈，并且退出进程；
    preprintpanics(gp._panic)
    fatalpanic(gp._panic) // should not return
    *(*int)(nil) = 0      // not reached
}
```

上面逻辑可以拆分为**循环内**和**循环外**两部分去理解：

- 循环内：程序执行 defer，是否恢复正常的指令执行，一切都在循环内决定；
- 循环外：一旦走到循环外，说明 `_panic` 没人处理，认命吧，程序即将退出；





### 4.1  for 循环内



循环内的事情拆解成：

1. 遍历 goroutine 的 defer 链表，获取到一个 `_defer` 延迟函数；

2. 获取到 `_defer` 延迟函数，设置标识 `d.started`，绑定当前 `d._panic`（用以在递归的时候判断）；

3. **执行 `_defer` 延迟函数**；

4. 摘掉执行完的 `_defer` 函数；

5. 判断 `_panic.recovered` 是否设置为 true，进行相应操作；

6. 1. 如果是 true 那么重置 pc，sp 寄存器（一般从 deferreturn 指令前开始执行），goroutine 投递到调度队列，等待执行；

7. 重复以上步骤；



 **1**  **思考问题有答案了！**

你会发现，更改 `recovered` 这个字段的时机只有在第三个步骤的时候。在任何地方，你都改不到 `_panic.recovered` 的值。

**问题一：为什么 recover 一定要放在 defer 里面才生效？**

**因为，这是唯一的修改 _panic.recovered 字段的时机 ！**

举几个对比的栗子：

```go
func main() {
    panic("test")
    recover()
}
```

上面的例子调用了 `recover( )` 为什么还是 panic ？

因为**根本执行不到 `recover` 函数**，执行顺序是：

```
panic 
  gopanic
    执行 defer 链表 
    exit
```

有童鞋较真，那我把 `recover()` 放 `panic("test")` 前面呗？

```go
func main() {
    recover()
    panic("test")
}
```

不行，因为执行 `recover` 的时候，还没有 `_panic` 挂在 goroutine 上面呢，`recover` 了个寂寞。



**问题二：为什么 `recover` 已经放在 `defer` 里面，但是进程还是没有恢复？**

回忆一下上面 for 循环的操作：

```go
// 步骤：遍历 _defer 链表
d := gp._defer
// 步骤：执行 defer 函数
reflectcall(nil, unsafe.Pointer(d.fn), deferArgs(d), uint32(d.siz), uint32(d.siz))
// 步骤：执行完成，把这个 defer 从链表里摘掉；
gp._defer = d.link
```

**划重点：在 `gopanic` 里，只遍历执行当前 goroutine 上的 `_defer` 函数链条。所以，如果挂在其他 goroutine 的 `defer` 函数做了 `recover` ，那么没有丝毫用途。**

举一个栗子：

```go
func main() { // g1
    go func() { // g2
        defer func() {
            recover()
        }()
    }()
    panic("test")
}
```

因为，`panic` 和 `recover` 在两个不同的 goroutine，`_panic` 是挂在 g1 上的，`recover` 是在 g2 的 `_defer` 链条里。

**`gopanic` 遍历的是 g1 的 `_defer` 函数链表，跟 g2 八杆子打不着**，g2 的 `recover` 自然拿不到 g1 的 `_panic` 结构，自然也不能设置 `recovered` 为 true ，所以程序还是崩了。



**问题三：为什么 panic 之后，还能再 panic ？有啥影响？**

这个其实很容易理解，有些童鞋可能想复杂了。`gopanic` 只是一个函数调用而已，那函数调用为啥不能嵌套递归？

**当然可以。**

触发的场景一般是：

- `gopanic` 函数调用 `_defer` 延迟函数；
- `defer` 延迟函数里面又调用了 `panic/gopanic` 函数；

这不就有了嘛，就是个简单的函数嵌套而已，没啥不可以的，并且在这种场景下，`_panic` 结构体就会从 `gp._panic` 开始形成了一个链表。

而 `gopanic` 函数指令执行的特殊在于两点：

1. `_panic` 被人设置成 recovered 之后，重置 pc，sp 寄存器，直接跨越 gopanic （还有嵌套的函数栈），跳转到正常业务流程中；
2. 循环之外，等到最后，没人处理 `_panic` 数据，那就 exit 退出进程，终止后续所有指令的执行；

举个嵌套的栗子：

```go
func main() {
    defer func() { // 延迟函数
        panic("panic again")
    }()
    panic("first")
}
```

函数执行：

```
    gopanic
        defer 延迟函数 
            gopanic
                无 defer 延迟函数（递归往上），终止条件达成
                
     // 打印堆栈，退出程序
    fatalpanic
```

童鞋你理解了吗？下面就来考考你哦。看一个栗子：

```go
func main() {
    println("=== begin ===")
    defer func() { // defer_0
        println("=== come in defer_0 ===")
    }()
    defer func() { // defer_1
        recover()
    }()
    defer func() { // defer_2
        panic("panic 2")
    }()
    panic("panic 1")
    println("=== end ===")
}
```

上面的函数会出打印堆栈退出进程吗？

**答案是：不会。**终端输出结果如下：

```
➜  panic ./test_panic
=== begin ===
=== come in defer_0 ===
```

你猜对了吗？给你梳理了一下完整的路线：

```
main
    gopanic // 第一次
        1. 取出 defer_2，设置 started
        2. 执行 defer_2 
            gopanic // 第二次
                1. 取出 defer_2，panic 设置成 aborted
                2. 把 defer_2 从链表中摘掉
                3. 执行 defer_1
                    - 执行 recover
                4. 摘掉 defer_1
                5. 执行 recovery ，重置 pc 寄存器，跳转到 defer_1 注册时候，携带的指令，一般是跳转到 deferreturn 上面几个指令

    // 跳出 gopanic 的递归嵌套，直接到执行 deferreturn 的地方；
    defereturn
        1. 执行 defer 函数链，链条上还剩一个 defer_0，取出 defer_0；
        2. 执行 defer_0 函数
    // main 函数结束
```

再来一个对比的例子：

```go
func main() {
    println("=== begin ===")
    defer func() { // defer_0
        println("=== come in defer_0 ===")
    }()
    defer func() { // defer_1
        panic("panic 2")    
    }()
    defer func() { // defer_2
        recover()
    }()
    panic("panic 1")
    println("=== end ===")
}
```

上面的函数会打印堆栈，并且退出吗？

**答案是：会。**输出如下：

```
➜  panic ./test_panic
=== begin ===
=== come in defer_0 ===
panic: panic 2

goroutine 1 [running]:
main.main.func2()
 /Users/code/gopher/src/panic/test_panic.go:9 +0x39
main.main()
 /Users/code/gopher/src/panic/test_panic.go:11 +0xf7
```

奇伢给你梳理的执行路径如下：

```
main
    gopanic // 第一次
        1. 取出 defer_2，设置 started
        2. 执行 defer_2 
            - 执行 recover，panic_1 字段被设置 recovered
        3. 把 defer_2 从链表中摘掉
        4. 执行 recovery ，重置 pc 寄存器，跳转到 defer_1 注册时候，携带的指令，一般是跳转到 deferreturn 上面几个指令

    // 跳出 gopanic 的递归嵌套，执行到 deferreturn 的地方；
    defereturn

        1. 遍历 defer 函数链，取出 defer_1   
        2. 摘掉 defer_1
        2. 执行 defer_1
            gopanic // 第二次
                1. defer 链表上有个 defer_0，取出来；
                2. 执行 defer_0 （defer_0 没有做 recover，只打印了一行输出）
                3. 摘掉 defer_0，链表为空，跳出 for 循环
                3. 执行 fatalpanic
                    - exit(2) 退出进程
```

你猜对了吗？



 ### 4.2 recovery 函数

最后，看一下关键的 recovery 函数。在 `gopanic` 函数中，在循环执行 defer 函数的时候，如果发现 `_panic.recovered` 字段被设置成 true 的时候，调用 `mcall(recovery)` 来执行所谓的**恢复**。

看一眼 `recovery` 函数的实现，这个函数极其简单，就是恢复 pc，sp 寄存器，重新把 Goroutine 投递到调度队列中。

```go
// runtime/panic.go
func recovery(gp *g) {
    // 取出栈寄存器和程序计数器的值
    sp := gp.sigcode0
    pc := gp.sigcode1
    // 重置 goroutine 的 pc，sp 寄存器；
    gp.sched.sp = sp
    gp.sched.pc = pc
    // 重新投入调度队列
    gogo(&gp.sched)
}
```

**重置了 pc，sp 寄存器代表什么意思？**

pc 寄存器指向指令所在的地址，换句话说，就是**跳转到其他地方执行指令去了**。而不是顺序执行 gopanic 后面的指令了，补回来了。

**`_defer.pc` 的指令行，这个指令是哪里？**

这个要回忆一下 `defer` 的章节，`defer` 注册延迟函数的时候对应一个 `_defer` 结构体，在 new 这个结构体的时候，`_defer.pc` 字段赋值的就是 new 函数的下一行指令。这个在 [Golang 最细节篇 — 解密 defer 原理，究竟背着程序猿做了多少事情？](http://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247486776&idx=1&sn=acd0b940bd4dd939552cfc410b021949&chksm=cf3e1dfdf84994ebccd479c23945508dfcfa43e6c089fe0bf852a4a6ba5001f9abfd2258eabb&scene=21#wechat_redirect) 详细说过。

举个例子，如果是栈上分配的话，那么在 `deferprocStack` ，所以，`mcall(recovery)` 跳转到这个位置，其实后续就走 `deferreturn` 的逻辑了，执行后续的 `_defer` 函数链。

本次 panic 就到此为止，相当于就恢复了程序的正常运行。

当然，如果后续在 defer 函数里面又出现 panic ，那可能形成一个 `_panic` 的链条，但是每一个的处理还是一样的。

**划重点：函数的 call，ret 是最常见的指令跳转。最本源的就是 pc 寄存器，函数压栈，出栈的时候，修改的也是 pc 寄存器，在 recovery 流程里，则来的更直接一点，直接改 pc ，sp。**





 ### 4.3 for 循环外



走到 for 循环外，那程序 100% 要退出了。因为 fatalpanic 里面打印一些堆栈信息之后，直接调用 exit 退出进程的。到这已经没有任何机会了，只能乖乖退出进程。

退出的调用就在 `fatalpanic` 里：

```go
func fatalpanic(msgs *_panic) {
    // 1. 打印协程堆栈

    // 2. 退出进程
    systemstack(func() {
        exit(2)
    })

    *(*int)(nil) = 0 // not reached
}
```

所以这个问题清楚了嘛：为什么 panic 会让 Go 进程退出的 ？

**还能为啥，因为调用了 exit(2) 嘛。**



## 5. 总结



1. `panic()` 会退出进程，是因为调用了 exit 的系统调用；
2. `recover()` 并不是说只能在 defer 里面调用，而是**只能在 defer 函数中才能生效**，**只有在 defer 函数里面，才有可能遇到 `_panic` 结构**；
3. `recover()` 所在的 defer 函数必须和 panic 都是挂在同一个 goroutine 上，**不能跨协程**，因为 `gopanic` **只会执行当前 goroutine 的延迟函数；**
4. panic 的恢复，就是重置 pc 寄存器，直接跳转程序执行的指令，跳转到原本 defer 函数执行完该跳转的位置（`deferreturn` 执行），从 `gopanic` 函数中跳出，不再回来，自然就不会再 `fatalpanic` ；
5. panic 为啥能嵌套？这个问题就像是在问**为什么函数调用可以嵌套**一样，因为这个本质是一样的。


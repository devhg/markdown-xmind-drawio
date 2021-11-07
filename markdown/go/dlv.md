# 用 dlv 调试

那有同学问了，有没有其他可以调试 Go、以及和 Go 程序互动的方法呢？其实是有的！这就是我们要介绍的 dlv 调试工具，目前它对调试 Go 程序的支持是最好的。

之前没我怎么研究它，只会一些非常简单的命令，这次学会了几个进阶的指令，威力挺大，也进一步加深了对 Go 的理解。

下面我们带着一个任务来讲解 dlv 如何使用。

我们知道，向一个 nil 的 slice append 元素，不会有任何问题。但是向一个 nil 的 map 插入新元素，马上就会报 panic。这是为什么呢？又是在哪 panic 呢？

首先写出让 map 产生 panic 的示例程序：

```go
package main

func main() {
    var m map[int]int
    m[1] = 1
}
```

接着用 `go build` 命令编译生成可执行文件：

```bash
go build a.go
```

然后，使用 dlv 进入调试状态：

```bash
dlv exec ./a
```

使用 `b` 这个命令打断点，有三种方法：

1. b + 地址
2. b + 代码行数
3. b + 函数名

我们要在对 map 赋值的地方加个断点。先找到代码位置：

```
cat -n a.go
```

看到：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)hello.go

赋值的地方在第 5 行，加断点：

```bash
(dlv) b a.go:5
Breakpoint 1 set at 0x45e55d for main.main() ./a.go:5
```

执行 `c` 命令，直接运行到断点处：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)运行到断点处

执行 `disass` 命令，可以看到汇编指令：

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)disass

这时使用 `si` 命令，执行单条指令，多次执行 `si`，就会执行到 map 赋值函数 `mapassign_fast64`：

![图片](https://mmbiz.qpic.cn/mmbiz_png/ASQrEXvmx62n8lLzhfvCN5TicbWguicZWjQpficE3PsWQFwsibVdQrl1MySDvwJD2uNDIXSry7wO1GkOYgRL2ibZvEw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)mapassign_fast64

这时再用单步命令 `s`，就会进入判断 h 的值为 nil 的分支，然后执行 `panic` 函数：

![图片](https://mmbiz.qpic.cn/mmbiz_png/ASQrEXvmx62n8lLzhfvCN5TicbWguicZWjg1lbybdsaRcXrlXkbEq1HicccCzsODh9jNj8Ctzh3jVS3kDPk5j7kiag/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)panic

至此，向 nil 的 map 赋值时，产生 panic 的代码就被我们找到了。接着，按图索骥找到对应 runtime 源码的位置，就可以进一步探索了。

除此之外，我们还可以使用 `bt` 命令看到调用栈：

![图片](https://mmbiz.qpic.cn/mmbiz_png/ASQrEXvmx62n8lLzhfvCN5TicbWguicZWjbCouVhaibWKChY3PiaZRCWvtfWoCf3xcd7APQ08TgUp9y6tbQyhQq0oA/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)调用栈

使用 `frame 1` 命令可以跳转到相应位置。这里 `1` 对应图中的 `a.go:5`，也就是我们前面打断点的地方，是不是非常酷炫。

上面这张图里我们也能清楚地看到，用户 goroutine 其实是被 goexit 函数一路调用过来的。当用户 goroutine 执行完毕后，就会回到 goexit 函数做一些收尾工作。当然，这是题外话了。

另外，用 dlv 也能干第二部分“找到 runtime 源码”活。

# 总结

今天系统地讲了几招通过命令和工具查看用户代码对应的 runtime 源码或者汇编代码的方法，非常实用。最后再汇总一下：

1. go tool compile
2. go tool objdump
3. dlv

使用这些命令和工具，可以让你在看 Go 源码的过程中事半功倍。
# CpuCache

## 前言

代码都是由 CPU 跑起来的，我们代码写的好与坏就决定了 CPU 的执行效率，特别是在编写计算密集型的程序，更要注重 CPU 的执行效率，否则将会大大影响系统性能。

CPU 内部嵌入了 CPU Cache（高速缓存），它的存储容量很小，但是离 CPU 核心很近，所以缓存的读写速度是极快的，那么如果 CPU 运算时，直接从 CPU Cache 读取数据，而不是从内存的话，运算速度就会很快。

但是，大多数人不知道 CPU Cache 的运行机制，以至于不知道如何才能够写出能够配合 CPU Cache 工作机制的代码，一旦你掌握了它，你写代码的时候，就有新的优化思路了。

那么，接下来我们就来看看，CPU Cache 到底是什么样的，是如何工作的呢，又该写出让 CPU 执行更快的代码呢？



## 正文

### CPU Cache 有多快？

你可能会好奇为什么有了内存，还需要 CPU Cache？根据摩尔定律，CPU 与内存的访问性能的差距不断拉大。到现在，一次内存访问所需时间是 `200~300` 多个时钟周期，这意味着 CPU 和内存的访问速度已经相差 `200~300` 多倍。

为了弥补 CPU 与内存两者之间的性能差异，就在 CPU 内部引入了  CPU Cache，也称高速缓存。

CPU Cache 通常分为大小不等的三级缓存，分别是 **L1 Cache、L2 Cache 和 L3 Cache**。

由于 CPU Cache 所使用的材料是 SRAM，价格比内存使用的 DRAM 高出很多，在当今每生产 1 MB 大小的 CPU Cache 需要 7 美金的成本，而内存只需要 0.015 美金的成本，成本方面相差了 466 倍，所以 CPU Cache 不像内存那样动辄以 GB 计算，它的大小是以 KB 或 MB 来计算的。

在 Linux 系统中，我们可以使用下图的方式来查看各级 CPU Cache 的大小，比如我这手上这台服务器，离 CPU 核心最近的 L1 Cache 是 32KB，其次是 L2 Cache 是 256KB，最大的 L3 Cache 则是 3MB。

![图片](cpu_cache.assets/640-20211121195516369)

其中，**L1 Cache 通常会分为「数据缓存」和「指令缓存」**，这意味着数据和指令在 L1 Cache 这一层是分开缓存的，上图中的 `index0` 也就是数据缓存，而 `index1` 则是指令缓存，它两的大小通常是一样的。

另外，你也会注意到，L3 Cache 比 L1 Cache 和 L2 Cache 大很多，这是因为 **L1 Cache 和 L2 Cache 都是每个 CPU 核心独有的，而 L3 Cache 是多个 CPU 核心共享的。**

程序执行时，会先将内存中的数据加载到共享的 L3 Cache 中，再加载到每个核心独有的 L2 Cache，最后进入到最快的 L1 Cache，之后才会被 CPU 读取。它们之间的层级关系，如下图：

![图片](cpu_cache.assets/640-20211121200953749)

越靠近 CPU 核心的缓存其访问速度越快，CPU 访问 L1 Cache 只需要 `2~4` 个时钟周期，访问 L2 Cache 大约 `10~20` 个时钟周期，访问 L3 Cache 大约 `20~60` 个时钟周期，而访问内存速度大概在 `200~300` 个 时钟周期之间。

时钟周期是 CPU 主频的倒数，比如主频为 2ghz 的 CPU，一个时钟周期是 0.5ns。

**所以，CPU 从 L1 Cache 读取数据的速度，相比从内存读取的速度，会快 `100` 多倍。**

------

### CPU Cache 的数据结构和读取过程

CPU Cache 的数据是从内存中读取过来的，它是以一小块一小块的读取数据的，而不是按照单个数组元素来读取数据的，在 CPU Cache 中的，这样一小块一小块的数据，称为 **Cache Line（缓存块）**。

你可以在你的 Linux 系统，用下面这种方式来查看 CPU 的 Cache Line，你可以看我服务器的 L1 Cache Line 大小是 64 字节，也就意味着 **L1 Cache 一次载入数据的大小是 64 字节**。

![图片](cpu_cache.assets/640-20211121195516292)

比如，有一个 `int array[100]` 的数组，当载入 `array[0]` 时，由于这个数组元素的大小在内存只占 4 字节，不足 64 字节，CPU 就会**顺序加载**数组元素到 `array[15]`，意味着 `array[0]~array[15]` 数组元素都会被缓存在 CPU Cache 中了，因此当下次访问这些数组元素时，会直接从 CPU Cache 读取，而不用再从内存中读取，大大提高了 CPU 读取数据的性能。

事实上，CPU 读取数据的时候，无论数据是否存放到 Cache 中，CPU 都是先访问 Cache，只有当 Cache 中找不到数据时，才会去访问内存，并把内存中的数据读入到 Cache 中，CPU 再从 CPU Cache 读取数据。

这样的访问机制，跟我们使用「内存作为硬盘的缓存」的逻辑是一样的，如果内存有缓存的数据，则直接返回，否则要访问龟速一般的硬盘。

![图片](cpu_cache.assets/640-20211121195516364)



那 CPU 怎么知道要访问的内存数据，是否在 Cache 里？如果在的话，如何找到 Cache 对应的数据呢？我们从最简单、基础的**直接映射 Cache（\*Direct Mapped Cache\*）** 说起，来看看整个 CPU Cache 的数据结构和访问逻辑。

前面，我们提到 CPU 访问内存数据时，是一小块一小块数据读取的，具体这一小块数据的大小，取决于 `coherency_line_size` 的值，一般 64 字节。在内存中，这一块的数据我们称为**内存块（\*Bock\*）**，读取的时候我们要拿到数据所在内存块的地址。

对于直接映射 Cache 采用的策略，就是把内存块的地址始终「映射」在一个 CPU Line（缓存块） 的地址，至于映射关系实现方式，则是使用「取模运算」，取模运算的结果就是内存块地址对应的 CPU Line（缓存块） 的地址。

举个例子，内存共被划分为 32 个内存块，CPU Cache 共有 8 个 CPU Line，假设 CPU 想要访问第 15 号内存块，如果 15 号内存块中的数据已经缓存在 CPU Line 中的话，则是一定映射在 7 号 CPU Line 中，因为 `15 % 8` 的值是 7。

机智的你肯定发现了，使用取模方式映射的话，就会出现多个内存块对应同一个 CPU Line，比如上面的例子，除了 15 号内存块是映射在 7 号 CPU Line 中，还有 7 号、23 号、31 号内存块都是映射到 7 号 CPU Line 中。

![图片](data:image/gif;base64,iVBORw0KGgoAAAANSUhEUgAAAAEAAAABCAYAAAAfFcSJAAAADUlEQVQImWNgYGBgAAAABQABh6FO1AAAAABJRU5ErkJggg==)

因此，为了区别不同的内存块，在对应的 CPU Line 中我们还会存储一个**组标记（Tag）**。这个组标记会记录当前 CPU Line 中存储的数据对应的内存块，我们可以用这个组标记来区分不同的内存块。

除了组标记信息外，CPU Line 还有两个信息：

- 一个是，从内存加载过来的实际存放**数据（\*Data\*）**。
- 一个是，**有效位（\*Valid bit\*）**，它是用来标记对应的 CPU Line 中的数据是否是有效的，如果有效位是 0，无论 CPU Line 中是否有数据，CPU 都会直接访问内存，重新加载数据。

CPU 在从 CPU Cache 读取数据的时候，并不是读取 CPU Line 中的整个数据块，而是读取 CPU 所需要的一个数据片段，这样的数据统称为一个**字（\*Word\*）**。那怎么在对应的 CPU Line 中数据块中找到所需的字呢？答案是，需要一个**偏移量（Offset）**。

因此，一个内存的访问地址，包括**组标记、CPU Line 索引、偏移量**这三种信息，于是 CPU 就能通过这些信息，在 CPU Cache 中找到缓存的数据。而对于 CPU Cache 里的数据结构，则是由**索引 + 有效位 + 组标记 + 数据块**组成。

![图片](cpu_cache.assets/640-20211121195516422)

如果内存中的数据已经在 CPU Cahe 中了，那 CPU 访问一个内存地址的时候，会经历这 4 个步骤：

1. 根据内存地址中索引信息，计算在 CPU Cahe 中的索引，也就是找出对应的 CPU Line 的地址；
2. 找到对应 CPU Line 后，判断 CPU Line 中的有效位，确认 CPU Line 中数据是否是有效的，如果是无效的，CPU 就会直接访问内存，并重新加载数据，如果数据有效，则往下执行；
3. 对比内存地址中组标记和 CPU Line 中的组标记，确认 CPU Line 中的数据是我们要访问的内存数据，如果不是的话，CPU 就会直接访问内存，并重新加载数据，如果是的话，则往下执行；
4. 根据内存地址中偏移量信息，从 CPU Line 的数据块中，读取对应的字。

到这里，相信你对直接映射 Cache 有了一定认识，但其实除了直接映射 Cache 之外，还有其他通过内存地址找到 CPU Cache 中的数据的策略，比如全相连 Cache （*Fully Associative Cache*）、组相连 Cache （*Set Associative Cache*）等，这几种策策略的数据结构都比较相似，我们理解流直接映射 Cache 的工作方式，其他的策略如果你有兴趣去看，相信很快就能理解的了。

------

### 如何写出让 CPU 跑得更快的代码？

#### 如何提升数据缓存的命中率？

假设要遍历二维数组，有以下两种形式，虽然代码执行结果是一样，但你觉得哪种形式效率最高呢？为什么高呢？

```go
package benchmark

import "testing"

func createMatrix(size int) [][]int64 {
	matrix := make([][]int64, size)
	for i := 0; i < size; i++ {
		matrix[i] = make([]int64, size)
	}
	return matrix
}

const matrixLength = 6400

func BenchmarkMatrixCombination(b *testing.B) {
	matrixA := createMatrix(matrixLength)
	matrixB := createMatrix(matrixLength)

	for n := 0; n < b.N; n++ {
		for i := 0; i < matrixLength; i++ {
			for j := 0; j < matrixLength; j++ {
				matrixA[i][j] = matrixA[i][j] + matrixB[i][j]
			}
		}
	}
}

func BenchmarkMatrixReversedCombination(b *testing.B) {
	matrixA := createMatrix(matrixLength)
	matrixB := createMatrix(matrixLength)

	for n := 0; n < b.N; n++ {
		for i := 0; i < matrixLength; i++ {
			for j := 0; j < matrixLength; j++ {
				matrixA[i][j] = matrixA[i][j] + matrixB[j][i]
			}
		}
	}
}

```



经过测试，形式一 `matrixB[i][j]` 执行时间比形式二 `matrixB[j][i]` 快好几倍。

```
goos: darwin
goarch: amd64
pkg: github.com/devhg/playgroud.go/benchmark
cpu: Intel(R) Core(TM) i7-9750H CPU @ 2.60GHz
BenchmarkMatrixCombination
BenchmarkMatrixCombination-12            	      16	  62682152 ns/op	45894662 B/op	     800 allocs/op
BenchmarkMatrixReversedCombination
BenchmarkMatrixReversedCombination-12    	       2	 875145025 ns/op	367157252 B/op	    6401 allocs/op
```

之所以有这么大的差距，是因为二维数组 `array` 所占用的内存是连续的。那么操作 `array[j][i]` 时，访问的方式跳跃式的，是没办法把 `array[j+1][i]` 也读入到 CPU Cache 中的，既然 `array[j+1][i]` 没有读取到 CPU Cache，那么就需要从内存读取该数据元素了。很明显，这种不连续性、跳跃式访问数据元素的方式，可能不能充分利用到了 CPU Cache 的特性，从而代码的性能不高。

**因此，遇到这种遍历数组的情况时，按照内存布局顺序访问，将可以有效的利用 CPU Cache 带来的好处，这样我们代码的性能就会得到很大的提升，**

#### 如何提升指令缓存的命中率？

提升数据的缓存命中率的方式，是按照内存布局顺序访问，那针对指令的缓存该如何提升呢？

在回答这个问题之前，我们先了解 CPU 的**分支预测器**。对于 if 条件语句，意味着此时至少可以选择跳转到两段不同的指令执行，也就是 if 还是 else 中的指令。那么，**如果分支预测可以预测到接下来要执行 if 里的指令，还是 else 指令的话，就可以「提前」把这些指令放在指令缓存中，这样 CPU 可以直接从 Cache 读取到指令，于是执行速度就会很快**。

当数组中的元素是随机的，分支预测就无法有效工作，而当数组元素都是顺序的，分支预测器会动态地根据历史命中数据对未来进行预测，这样命中率就会很高。

因此，先排序再遍历速度会更快，这是因为排序之后，数字是从小到大的，那么前几次循环命中 `if < 50` 的次数会比较多，于是分支预测就会缓存 `if` 里的 `array[i] = 0` 指令到 Cache 中，后续 CPU 执行该指令就只需要从 Cache 读取就好了。

如果你肯定代码中的 `if` 中的表达式判断为 `true` 的概率比较高，我们可以使用显示分支预测工具，比如在 C/C++ 语言中编译器提供了 `likely` 和 `unlikely` 这两种宏，如果 `if` 条件为 `ture` 的概率大，则可以用 `likely` 宏把 `if` 里的表达式包裹起来，反之用 `unlikely` 宏。

实际上，CPU 自身的动态分支预测已经是比较准的了，所以只有当非常确信 CPU 预测的不准，且能够知道实际的概率情况时，才建议使用这两种宏。



#### 如果提升多核 CPU 的缓存命中率？

在单核 CPU，虽然只能执行一个进程，但是操作系统给每个进程分配了一个时间片，时间片用完了，就调度下一个进程，于是各个进程就按时间片交替地占用 CPU，从宏观上看起来各个进程同时在执行。

而现代 CPU 都是多核心的，进程可能在不同 CPU 核心来回切换执行，这对 CPU Cache 不是有利的，虽然 L3 Cache 是多核心之间共享的，但是 L1 和 L2 Cache 都是每个核心独有的，**如果一个进程在不同核心来回切换，各个核心的缓存命中率就会受到影响**，相反如果进程都在同一个核心上执行，那么其数据的 L1 和 L2 Cache 的缓存命中率可以得到有效提高，缓存命中率高就意味着 CPU 可以减少访问 内存的频率。

当有多个同时执行「计算密集型」的线程，为了防止因为切换到不同的核心，而导致缓存命中率下降的问题，我们可以把**线程绑定在某一个 CPU 核心上**，这样性能可以得到非常可观的提升。

在 Linux 上提供了 `sched_setaffinity` 方法，来实现将线程绑定到某个 CPU 核心这一功能。Go语言中也要类似的函数`runtime.LockOSThread()`

------

### 总结

由于随着计算机技术的发展，CPU 与 内存的访问速度相差越来越多，如今差距已经高达好几百倍了，所以 CPU 内部嵌入了 CPU Cache 组件，作为内存与 CPU 之间的缓存层，CPU Cache 由于离 CPU 核心很近，所以访问速度也是非常快的，但由于所需材料成本比较高，它不像内存动辄几个 GB 大小，而是仅有几十 KB 到 MB 大小。

当 CPU 访问数据的时候，先是访问 CPU Cache，如果缓存命中的话，则直接返回数据，就不用每次都从内存读取速度了。因此，缓存命中率越高，代码的性能越好。

但需要注意的是，当 CPU 访问数据时，如果 CPU Cache 没有缓存该数据，则会从内存读取数据，但是并不是只读一个数据，而是一次性读取一块一块的数据存放到 CPU Cache 中，之后才会被 CPU 读取。

内存地址映射到 CPU Cache 地址里的策略有很多种，其中比较简单是直接映射 Cache，它巧妙的把内存地址拆分成「索引 + 组标记 + 偏移量」的方式，使得我们可以将很大的内存地址，映射到很小的 CPU Cache 地址里。

要想写出让 CPU 跑得更快的代码，就需要写出缓存命中率高的代码，CPU L1 Cache 分为数据缓存和指令缓存，因而需要分别提高它们的缓存命中率：

- 对于数据缓存，我们在遍历数据的时候，应该按照内存布局的顺序操作，这是因为 CPU Cache 是根据 CPU Cache Line 批量操作数据的，所以顺序地操作连续内存数据时，性能能得到有效的提升；
- 对于指令缓存，有规律的条件分支语句能够让 CPU 的分支预测器发挥作用，进一步提高执行的效率；

另外，对于多核 CPU 系统，线程可能在不同 CPU 核心来回切换，这样各个核心的缓存命中率就会受到影响，于是要想提高进程的缓存命中率，可以考虑把线程绑定 CPU 到某一个 CPU 核心。











#  缓存一致性与伪共享问题

> https://mp.weixin.qq.com/s/2O8SDJTf6LRcv_9Qc2Tkrw



**Cache Coherency & False Sharing**

假设有一个双核CPU，两个核心上并行运行着不同的线程，它们同时从内存中读取两个不同的数据A和B，如果这两个数据在物理内存上是连续的（或者非常接近），那么就会出现在两个核心的L1 Cache中均存在var1和var2的情况。

![图片](cpu_cache.assets/640-20211121204932796)

我们知道，为了提高数据访问效率，每个CPU核心上都内嵌了一个容量小，但速度快的缓存体系，用于保存最常访问的那些数据。因此当数据被修改时，处理器也会首先只更改缓存中的内容，并不会马上将更改写回到内存中去，那么这样就会产生问题。

以上图为例，如果此时两个处于不同核心的线程1和线程2都试图去修改数据，例如线程1修改数据A，线程2修改数据B，这样就造成了在各缓存之间，缓存与内存之间数据均不一致。此时在线程1中看到的数据B和线程2中看到的数据A不再一样（或者如果有更多核上搭载的线程，它们从内存中取的还是老数据），即存在脏数据，这给程序带来了巨大隐患。因此有必要维护多核的缓存一致性。

缓存一致性的朴素解决思想也比较简单：只要在多核共享缓存行上有数据修改操作，就通知所有的CPU核更新缓存，或者放弃缓存，等待下次访问的时候再重新从内存中读取。

但很明显，这样的约束条件会对程序性能有所影响，目前有很多维护缓存一致性的协议，其中，最著名的是Intel CPU中使用的MESI缓存一致性协议。

### MESI协议

理解MESI协议前，我们需要知道的是：所有cache与内存，cache与cache之间的数据传输都发生在一条共享的数据总线上，所有的cpu核都能看到这条总线。

MESI协议是一种监听协议，cahce不但与内存通信时和总线打交道，同时它会不停地监听总线上发生的数据交换，跟踪其他cache在做什么。所以当一个cache代表它所属的cpu核去读写内存，或者对数据进行修改，其它cpu核都会得到通知，它们以此来使自己的cache保持同步。

MESI的四个独立字母是代表Cache line的四个状态，每个缓存行只可能是四种状态之一。在缓存行中占用两比特位，其含义如下。

- Modified（被修改的）：处于这一状态的数据只在本核处理器中有缓存，且其数据已被修改，但还没有更新到内存中。
- Exclusive（独占的）：处于这一状态的数据只在本核处理器中有缓存，且其数据没有被修改，与内存一致。
- Shared（共享的）：处于这一状态的数据在多核处理器中都有缓存。
- Invalid（无效的）：本CPU中的这份缓存已经无效了。

还是通过上述例子，一起来看处理器是如何通过MESI保证缓存一致性的。

![图片](cpu_cache.assets/640-20211121204932737)

1. 假设线程1首先读取数据A，因为按缓存行读取，且A和B在物理内存上是相邻的，所以数据B也会被加载到Core 1的缓存行中，此时将此缓存行标记为**Exclusive**状态。

![图片](cpu_cache.assets/640-20211121204932951)

1. 接着线程2读取数据B，它从内存中取出了数据A和数据B到缓存行中。由于在Core 1中已经存在当前数据的缓存行，那么此时处理器会将这两个缓存行标记为**Shared**状态。

![图片](cpu_cache.assets/640-20211121204932805)

1. Core1 上的线程1要修改数据A，它发现当前缓存行的状态是**Shared**，所以它会先通过数据总线发送消息给Core 2，通知Core 2将对应的缓存行标记为**Invalid**，然后再修改数据A，同时将Core 1上当前缓存行标记为**Modified**。

![图片](cpu_cache.assets/640-20211121204932840)

1. 此后，线程2想要修改数据B，但此时Core2 中的当前缓存行已经处于**Invalid**状态，且由于Core 1当中对应的缓存行也有数据B，且缓存行处于**Modified**状态。因此，Core2 通过内存总线通知Core1 将当前缓存行数据写回到内存，然后Core 2再从内存读取缓存行大小的数据到Cache中，接着修改数据B，当前缓存行标记为**Modified**。最后，通知Core1将对应缓存行标记为**Invalid**。

所以，可以发现，如果Core 1和 Core2 上的线程持续交替的对数据A和数据B作修改，就会重复 3 和 4 这两个步骤。这样，Cache 并没有起到缓存的效果。

虽然变量 A 和 B 之间其实并没有任何的关系，但是因为归属于一个缓存行 ，这个缓存行中的任意数据被修改后，它们都会相互影响。因此，这种因为多核线程同时读写同一个 Cache Line 的不同变量，而导致 CPU 缓存失效的现象就是**伪共享**。

### 内存填充

那有没有什么办法规避这种伪共享呢？**答案是有的：内存填充（Memory Padding）**。它的做法是在两个变量之间填充足够多的空间，以保证它们属于不同的缓存行。下面，我们看一个具体的例子。

https://github.com/devhg/playgroud.go/blob/master/benchmark/mem_padding_test.go

```go
// 这里M需要足够大，否则会存在goroutine 1已经执行完成，而goroutine 未启动的情况
const M = 1000000

type SimpleStruct struct {
	n int
}

func BenchmarkStructureFalseSharing(b *testing.B) {
	structA := SimpleStruct{}
	structB := SimpleStruct{}
	wg := sync.WaitGroup{}

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		wg.Add(2)
		go func() {
			for j := 0; j < M; j++ {
				structA.n += 1
			}
			wg.Done()
		}()
		go func() {
			for j := 0; j < M; j++ {
				structB.n += 1
			}
			wg.Done()
		}()
		wg.Wait()
	}
}
```

在该例中，我们相继实例化了两个结构体对象structA和structB，因此，这两个结构体应该会在内存中被连续分配。之后，我们创建两个goroutine，分别去访问这两个结构体对象。

structA上的变量n被goroutine 1访问，structB上的变量n被goroutine 2访问。然后，由于这两个结构体在内存上的地址是连续的，所以两个n会存在于两个CPU缓存行中（假设两个goroutine会被调度分配到不同CPU核上的线程，当然，这不是一定保证的），压测结果如下。

```
BenchmarkStructureFalseSharing-12    	     552	   2082212 ns/op	      75 B/op	       2 allocs/op
```

下面我们使用内存填充：在结构体中填充一个为缓存行大小的占位对象CacheLinePad。

```go
// 使用内存填充
type PaddedStruct struct {
	n int
	_ CacheLinePad
}

type CacheLinePad struct {
	_ [CacheLinePadSize]byte
}

const CacheLinePadSize = 64
```

然后，我们实例化这两个结构体，并继续在单独的goroutine中访问两个变量。

```go
func BenchmarkStructurePadding(b *testing.B) {
	structA := PaddedStruct{}
	structB := SimpleStruct{}
	wg := sync.WaitGroup{}

	b.ResetTimer()
	for i := 0; i < b.N; i++ {
		wg.Add(2)
		go func() {
			for j := 0; j < M; j++ {
				structA.n += 1
			}
			wg.Done()
		}()
		go func() {
			for j := 0; j < M; j++ {
				structB.n += 1
			}
			wg.Done()
		}()
		wg.Wait()
	}
}
```

在CPU Cache中，内存分布应该如下图所示，因为两个变量之间有足够多的内存填充，所以它们只会存在于不同CPU核心的缓存行。

![图片](cpu_cache.assets/640-20211121204932792)

下面是两种方式的压测结果对比

```
BenchmarkStructureFalseSharing-12    	     560	   2054443 ns/op	      79 B/op	       2 allocs/op
BenchmarkStructurePadding-12         	     855	   1412661 ns/op	      48 B/op	       2 allocs/op
```

可以看到，在该例中使用填充的比最初的实现会快30%左右，这是一种以空间换时间的做法。需要注意的是，内存填充的确能提升执行速度，但是同时会导致更多的内存分配与浪费。



### 结语

机械同理心（Mechanical sympathy）是软件开发领域的一个重要概念，其源自三届世界冠军 F1赛车手 Jackie Stewart 的一句名言：

**You don’t have to be an engineer to be a racing driver, but you do have to have Mechanical Sympathy. （若想成为一名赛车手，你不必成为一名工程师，但必须要有机械同理心。）**

了解赛车的运作方式能让你成为更好的赛车手，同样，理解计算机硬件的工作原理能让程序员写出更优秀的代码。你不一定需要成为一名硬件工程师，但是你确实需要了解硬件的工作原理，并在设计软件时考虑这一点。

现代计算机为了弥补CPU处理器与主存之间的性能差距，引入了多级缓存体系。有了缓存的存在，CPU就不必直接与主存打交道，而是与响应更快的L1 Cache进行交互。根据局部性原理，缓存与内存的交换数据单元为一个缓存行，缓存行的大小一般是64个字节。

因为缓存行的存在，我们需要写出缓存命中率更高的程序，减少从主存中交换数据的频率，从而提高程序执行效率。同时，在多核多线程当中，为了保证缓存一致性，处理器引入了MESI协议，这样就可能会存在CPU 缓存失效的伪共享问题。最后，我们介绍了一种以空间换时间的内存填充做法，它虽然提高了程序执行效率，但也造成了更多的内存浪费。
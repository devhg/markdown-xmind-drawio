实际编程时，需要同步保护的临界区中通常会有数十数百条指令，甚至更多。锁需要将所有线程（或协程）对临界区的访问做串行化处理，就要同时保证两点要求：

（1）同时只能有一个线程获得锁，持有锁才能进入临界区；

（2）当线程离开临界区释放锁后，线程在临界区内做的所有操作都要全局可见。



## 1. 原子指令

软件层面的锁通常被实现为内存中的一个共享变量，加锁的过程至少需要3个步骤，按顺序依次是Load、Compare和Store。

（1）Load操作从内存中读取锁最新的状态；

（2）Compare操作检测是否处于未加锁状态；

（3）如果未加锁的话就通过Store操作进行修改来实现加锁。如果Compare发现已经处于加锁状态了，就不能执行后续的Store操作了。



如果用一般的x86汇编指令来实现Load-Compare-Store操作，至少需要两条指令，比如CMP和MOV：

- `CMP`可以接受一个内存地址操作数，所以实质上包含了Load和Compare两步。
- `MOV`指令用来向指定内存地址写入数据，也就是Store操作。

但是这样实现会有一个问题，我们知道线程用完时间片之后会被打断，假如线程a执行完CMP指令后发现未加锁，但是在执行MOV之前被打断了，然后线程b开始执行并获得了锁，接下来线程b在临界区中被打断，线程a恢复执行后也获得了锁，这样一来就会出现错误。

![图片](https://mmbiz.qpic.cn/mmbiz_png/ibjI8pEWI9L6ClKxkrzakUYcKB89GzxnnuYmYdMhknYzSJVSc0duSuYrTxFkz0icb7sicW111oeicLSBOYHyicUiabpw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

所以我们需要在一条指令中完成整个Load-Compare-Store操作，就必须要从硬件层面提供支持，例如x86就提供了CMPXCHG指令。



`CMPXCHG`是Compare andExchange的缩写，该指令有两个操作数，用来实现锁的时候，第一操作数通常是个内存地址，也称为目的操作数，第二操作数是个通用寄存器。

CMPXCHG会将`AX`寄存器和第一操作数进行比较，相等的话就把第二操作数，在下面代码存在DX寄存器中，拷贝到目的操作数中；若不相等就把目的操作数拷贝到AX寄存器中。基于这个指令来实现锁的话，一条指令是不会在中间被打断的，所以就解决了之前的问题。

`CAS(&addr, old, new)` ：`CX`：存了第一操作数地址         `AX`：存了old        `DX`：存了new

`CMPXCHGQ  DX, (CX)`  ：首先，CX和AX比较，如果相等，把DX拷贝到CX；否则把CX拷贝到AX

```go
package main

import (
  _ "embed"
  "sync/atomic"
)

func main() {
  var a int64 = 1
  atomic.CompareAndSwapInt64(&a, 1, 2)
}

```

```assembly
0x0020 00032 (main.go:9)        CALL    runtime.newobject(SB)
0x0025 00037 (main.go:9)        MOVQ    AX, "".&a+16(SP)
0x002a 00042 (main.go:9)        MOVQ    $1, (AX)
0x0031 00049 (main.go:10)       MOVQ    "".&a+16(SP), CX
0x0036 00054 (main.go:10)       MOVL    $1, AX
0x003b 00059 (main.go:10)       MOVL    $2, DX
0x0040 00064 (main.go:10)       LOCK
0x0041 00065 (main.go:10)       CMPXCHGQ        DX, (CX)
```



在单核环境下，任何能够通过一条指令完成的操作都可以称为“原子操作”，但是这也只适用于单核场景，在多核环境下，运行在不同CPU核心上的线程可能会并行加锁，不同核心同时执行CMPXCHG又会造成多个线程同时获得锁。**如何解决这个问题呢？**

一种思路是，在当前核心执行CMPXCHG时，阻止其他核心执行CMPXCHG，x86汇编中的`LOCK`前缀就是用来实现这一目的的。

LOCK前缀能够应用于部分内存操作指令，最简单的解释就是LOCK前缀会让当前CPU核心在当前指令执行期间**独占总线**，这样其他的CPU核心就不能同时操作内存了。事实上，只有对于不在高速缓存中的数据才会这样，对于高速缓存中的数据，LOCK前缀会通过MESI协议处理多核间缓存一致性。不管怎么说，加上LOCK前缀的CMPXCHG就无懈可击了。在多核环境下，这种带有LOCK前缀的指令也被称为**原子指令**。



至此，针对锁的两点要求，其中第一个可以通过原子指令实现了。


**那么如何做到第二点要求呢？也就是释放锁之后，临界区内所有的操作要全局可见。**

事实上，锁本身的状态变化就必须是全局可见的，而且必须要很及时，以保证高性能。因此，在x86 CPU上，LOCK前缀同时具有内存排序的作用，相当于在应用LOCK前缀的指令之后紧接着执行了一条MFENCE指令。综上所述，原子指令既能保证只允许一个线程进入临界区，又具有内存排序的作用，能够保证在锁的状态发生变化时，临界区中所有的修改随锁的状态一起变成全局可见。



## 2. 自旋锁

自旋锁得以实现的基础就是原子性的CAS操作，CAS即Compare And Swap，在x86平台上对应带有LOCK前缀的CMPXCHG指令。之所以叫自旋锁，是因为它会一直循环尝试CAS操作直到成功，看起来就像是一直在自旋等待。

接下来我们就尝试一下用汇编语言基于CMPXCHG指令实现一把自旋锁，首先在Go语言中基于int32创建一个自定义类型Spin，并为它实现Lock方法和Unlock方法：

```go
type Spin int32

func (l *Spin) Lock() { lock((*int32)(l), 0, 1) }

func (l *Spin) Unlock() { unlock((*int32)(l), 0) }

func lock(ptr *int32, o, n int32)

func unlock(ptr *int32, n int32)
```

实际的加锁和解锁操作在lock和unlock这两个函数中实现，Go代码中只包含这两个函数的原型声明，这两个函数是用汇编语言实现的，具体代码在spin_amd64.s文件中：

```assembly
#include "textflag.h"

// func lock(ptr *int32, old, new int32)
TEXT ·lock(SB), NOSPLIT, $0-16
    MOVQ  ptr+0(FP), BX
    MOVL  old+8(FP), AX
    MOVL  new+12(FP), CX
again:
    LOCK
    CMPXCHGL  CX, 0(BX)
    JE    ok
    JMP   again
ok:
    RET

// func unlock(ptr *int32, val int32)
TEXT ·unlock(SB), NOSPLIT, $0-12
    MOVQ  ptr+0(FP), BX
    MOVL  val+8(FP), AX
    XCHGL AX, 0(BX)
    RET

```

lock函数把锁的地址放在了BX寄存器中，把用来比较的旧值old放到了AX寄存器中，把要写入的新值new放到了CX寄存器中。

从标签again处开始，是一个循环，不断尝试通过CMPXCHG进行加锁，成功后会通过JE指令跳出循环。因为Go的汇编风格有点类似AT&T，操作数书写顺序与Intel汇编相反，所以CMPXCHG的两个操作数中BX出现在了CX右边。能够通过JE跳出循环，是因为CMP操作会影响标志寄存器。

unlock函数通过XCHG指令将锁清零，实现了解锁操作。细心的读者可能会注意到这里没有LOCK前缀，根据Intel开发者手册所讲，XCHG指令隐含了LOCK前缀，所以代码中不用写，依然能够起到独占总线和内存排序的作用。

事实上，atomic包中的CompareAndSwapInt32和StoreInt32这两个函数，就是基于CMPXCHG和XCHG这两条汇编指令实现的。所以上述的自旋锁可以改成完全用Go来实现：

```go
import "sync/atomic"

type Spin int32

func (l *Spin) Lock() { for !atomic.CompareAndSwapInt32((*int32)(l), 0, 1) {} }
func (l *Spin) Unlock() { atomic.StoreInt32((*int32)(l), 0) }
```

这样一来，我们确实实现了自旋锁，但是这跟生产环境中实际使用的自旋锁比起来还是有些差距。在锁竞争比较激烈的场景下，这种自旋会造成CPU使用率很高，所以还要进行优化。x86专门为此提供了PAUSE指令，它一方面能够提示处理器当前正处于自旋循环中，从而在退出循环的时候避免因检测到内存乱序而造成性能损失。另一方面，PAUSE能够大幅减小自旋造成的CPU功率消耗，节能减排减少发热。

可以把PAUSE指令加入到我们汇编版本的lock函数实现中，修改后的代码如下所示：

```assembly
// func lock(ptr *int32, old, new int32)
TEXT ·lock(SB), NOSPLIT, $0-16
    MOVQ  ptr+0(FP), BX
    MOVL  old+8(FP), AX
    MOVL  new+12(FP), CX
again:
    LOCK
    CMPXCHGL  CX, 0(BX)
    JE    ok
    PAUSE
    JMP   again
ok:
    RET

```

也可以把PAUSE指令单独放在一个函数里，这样就能够跟atomic包中的函数结合使用了：

```assembly
#include "textflag.h"

// func pause()
TEXT pause(SB), NOSPLIT, $0-0
    PAUSE
    RET
```



这样就对Go代码实现的自旋锁进行优化了：

```go
func (l *Spin) Lock() {
    for !atomic.CompareAndSwapInt32((*int32)(l), 0, 1) {
        pause()
    }
}
```

自旋锁的优点是比较轻量，不过它对适用的场景也是有要求的。首先，在单核心的环境下不适合使用自旋锁，因为单核系统上任一时刻只能有一个线程在运行，当前线程一直在自旋等待而持有锁的线程得不到运行，锁不可能被释放，等也是白等，纯属浪费CPU。这种情况下及时切换到其他可运行的线程会更高效一些，因此在单核环境下更适合用调度器对象。其次，即使是在多核环境下，也要考虑平均持有锁的时间，以及程序的并发程度等因素。在持有锁的时间占比很小，且活跃线程数接近CPU核心数量时，自旋锁比较高效，也就是自旋的代价小于线程切换的代价。其他情况就不一定了，要结合实际场景分析再加上充分的测试。



## 3. 调度器对象

使用调度器对象这个名字，主要是受Windows NT内核的影响。更通俗的讲，应该说是操作系统提供的线程间同步原语，一般以一组系统调用的形式存在。比如Win32的Event，以及Linux的futex等。基于这些同步原语，可以实现锁及更复杂的同步工具。



这些调度器对象与自旋锁的不同，主要是有一个等待队列。当线程获取锁失败时不会一直在那里自旋，而是挂起后进入等待队列中等待，然后系统调度器会切换到下一个可运行的线程。等到持有锁的线程释放锁的时候，会按照一定的算法从等待队列中取出一个线程并唤醒它，被唤醒的线程会获得所有权，然后继续执行。



这些同步原语是由内核提供的，直接与系统的调度器交互，能够挂起和唤醒线程，这一点是自旋锁做不到的。等待队列可以实现成支持FIFO、FILO，甚至支持某种优先级策略。但是也正是由于是在内核中实现的，所以应用程序需要以系统调用的方式来使用它，这就造成了一定的开销。获取锁失败的情况下还会发生线程切换，进一步增大开销。调度器对象和自旋锁各自有适用的场景，具体如何选用还要结合具体场景来分析。



**自旋锁**主要适用于多核环境下，且持有锁的时间占比较小的情况。这种情况下，往往在几次自旋之后就能拿到锁，比起发生一次线程切换的代价要小得多。

**调度器对象**主要适用于加锁失败就要挂起线程的场景，比如说单核环境，或者持有锁的时间占比较大的情况。

而实际的业务逻辑中，持有锁的时间往往不是很确定，有可能较短也有可能较长，我们不好一概用一种策略进行处理，将自旋锁和调度器对象结合的话，理论上就能得到一把优化的锁了：

加锁时首先经过自旋锁，但是限制最大自旋次数，如果在有限次数内加锁成功也就成功了，否则就进一步通过调度器对象将当前线程挂起。等到持有锁的线程释放锁的时候，会通过调度器对象将挂起的线程唤醒。

这样就结合了二者的优点，既避免了加锁失败立即挂起线程造成过多的上下文切换，又避免了无限制的自旋空耗CPU，这也是如今主流的锁实现思路，当然，Go语言也不例外。
最近我也在学习如何在开发中让代码运行更加高效，然后在浏览各种优秀的日志设计的时候看到 uber 有一个叫 zap 的日志库引起了我的注意，它主要特性是对性能和内存分配都做到了极致的优化。

对于我来说，原本在项目中是使用 logrus 来作为日志输出，但是看到 zap 的 benchmark，感觉在性能方面甩 logrus 不知道多少条街，所以这也是驱使我来看看它是如何进行优化的原因。

当然除了这个问题以外，还是想更有把握地熟悉它，以便能更高效地利用这个库，以及避免在出现问题的时候无法定位。

下面先放一下比较唬人的 benchmark ，给大家提供一下看下去的动力：

| Package         | Time        | Time % to zap | Objects Allocated |
| :-------------- | :---------- | :------------ | :---------------- |
| ⚡ zap           | 862 ns/op   | +0%           | 5 allocs/op       |
| ⚡ zap (sugared) | 1250 ns/op  | +45%          | 11 allocs/op      |
| zerolog         | 4021 ns/op  | +366%         | 76 allocs/op      |
| go-kit          | 4542 ns/op  | +427%         | 105 allocs/op     |
| apex/log        | 26785 ns/op | +3007%        | 115 allocs/op     |
| logrus          | 29501 ns/op | +3322%        | 125 allocs/op     |
| log15           | 29906 ns/op | +3369%        | 122 allocs/op     |

## zap 设计

### log 的实例化

在开始使用的时候，我们可以通过官方的例子来了解 zap 内部的组件：

```go
log := zap.NewExample()
```

NewExample 函数里面展示了要通过 NewCore 来创建一个 Core 结构体，根据名字我们应该也能猜到这个结构体是 zap 的核心。

对于一个日志库来说，最主要是无非是这三类：

1. 对于输入的数据需要如何序列化；
2. 将输入的数据序列化后存放到哪里，是控制台还是文件，还是别的地方；
3. 然后就是日志的级别，是 Debug、Info 亦或是 Error；

同理 zap 也是这样，在使用 NewCore 创建 Core 结构体的时候需要传入的三个参数分别对应的就是：输入数据的编码器 Encoder、日志数据的目的地 WriteSyncer，以及日志级别 LevelEnabler。

除了 NewExample 这个构造方法以外，zap 还提供了 NewProduction、NewDevelopment 来构造日志实例：

```go
log, _  = zap.NewProduction()
log, _  = zap.NewDevelopment()
```

这两个函数会通过构建一个 Config 结构体然后调用 Build 方法来创建 NewCore 所需要的参数，然后实例化日志实例。

![图片](https://mmbiz.qpic.cn/mmbiz_png/47hDWI1Fd96KV2LiblMCPrx9mUCcV6t3haOj9QzXBuFz0MibBTrkbNbOsYqKjLSicnkvPxlkEwQQFMCjd9EK2OzYQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

### 日志数据的输出

在初始化 log 实例之后，可以用 Info、Debug、Error等方法打印日志：

```
 log  = zap.NewExample()
 url := "http://example.org/api"
 log.Info("failed to fetch URL",
  zap.String("url", url),
  zap.Int("attempt", 3),
  zap.Duration("backoff", time.Hour),
 )
```

我们再来看一下 zap 打印一条结构化的日志的实现步骤：

1. 首先会校验一下日志配置的等级，例如 Error 日志配置等级肯定是不能输出 Debug 日志出来；

2. 然后会将日志数据封装成一个 Entry 实例；

3. 因为在 zap 中可以传入 multiCore，所以会把多个 Core 添加到  CheckedEntry 实例中；

4. 遍历 CheckedEntry  实例中 Cores，

5. 1. 根据 Core 中的 Encoder 来序列化日志数据到 Buffer 中；
   2. 再由 WriteSyncer 将 Buffer 的日志数据进行输出；

### 接口与框架设计

![图片](https://mmbiz.qpic.cn/mmbiz_png/47hDWI1Fd96KV2LiblMCPrx9mUCcV6t3hWkwTDgUIdXOlj3bsHX3oQUHkL9fGA3AMvhy8eiczKiahXicMe1mjiaCSyw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

在代码结构设计上，通过简单的接口封装，实现了多种样式的配置组合，从而满足各种需求。在最上层的设计上实现了三种 log 用来实现不同的功能：

**Logger**：使用较为繁琐，只能使用结构化输出，但是性能更好；

**SugaredLogger**：可以使用 Printf 来输出日志，性能较 Logger 相比差 40% 左右；

**zapgrpc**：用做 grpc 的日志输出；

在设计上 Logger 可以很方便的转化为 SugaredLogger 和 zapgrpc。这几个 Logger 需要传入一个 Core 接口的实现类才能创建。

**Core 接口**：zap 也提供了多种实现的选择：NewNopCore 、ioCore、multiCore 、hook。

最常用的是 ioCore、multiCore ，从名字便可看出来 multiCore 是可以包含多个 ioCore 的一种配置，比方说可以让 Error 日志输出一种日志格式以及设置一个日志输出目的地，让 Info 日志以另一种日志格式输出到别的地方。

在上面也说了，对于 Core 的实现类 ioCore 来说它需要传入三个对象：输入数据的编码器 Encoder、日志数据的目的地 WriteSyncer，以及日志级别 LevelEnabler。

**Encoder 接口**：zap 提供了 consoleEncoder、jsonEncoder 的实现，分别提供了 console 格式与 JSON 格式日志输出，这些 Encoder 都有自己的序列化实现，这样可以更快的格式化代码；

**EncoderConfig**：上面所说的 Encoder 还可以根据 EncoderConfig 的配置允许使用者灵活的配置日志的输出格式，从日志消息的键名、日志等级名称，到时间格式输出的定义，方法名的定义都可以通过它灵活配置。

**WriteSyncer 接口**：zap 提供了 writerWrapper 的单日志输出实现，以及可以将日志输出到多个地方的 multiWriteSyncer 实现；

**Entry** ：配置说完了，到了日志数据的封装。首先日志数据会封装成一个 Entry，包含了日志名、日志时间、日志等级，以及日志数据等信息，没有 Field 信息，然后经验 Core 的 Check 方法对日志等级校验通过之后会生成一个 CheckedEntry 实例。

**CheckedEntry** 包含了日志数据所有信息，包括上面提到的 Entry、调用栈信息等。

### 性能

#### 使用对象池

zap 通过 sync.Pool 提供的对象池，复用了大量可以复用的对象，如果对 sync.Pool 不是很了解的同学，可以看这篇文章：《多图详解Go的sync.Pool源码 https://www.luozhiyun.com/archives/416 》。

zap 在实例化 CheckedEntry 、Buffer、Encoder 等对象的时候，会直接从对象池中获取，而不是直接实例化一个新的，这样复用对象可以降低 GC 的压力，减少内存分配。

#### 避免反射

如果我们使用官方的 log 库，像这样输出日志：

```
log.Printf("%s login, age:%d", "luoluo", 19)
```

log 调用的 Printf 函数实际上会调用 `fmt.Sprintf`函数来格式化日志数据，然后进行输出：

```
func Printf(format string, v ...interface{}) {
 std.Output(2, fmt.Sprintf(format, v...))
}
```

但是`fmt.Sprintf`效率实际上是很低的，通过查看fmt.Sprintf源码， 可以看出效率低有两个原因：

1. `fmt.Sprintf` 接受的类型是 interface{}，内部使用了反射；
2. `fmt.Sprintf` 的用途是格式化字符串，需要去解析格式串，比如 `%s`、 `%d`之类的，增加了解析的耗时。

但是在 zap 中，使用的是内建的 Encoder，它会通过内部的 Buffer 以 byte 的形式来拼接日志数据，减少反射所带来性能损失；以及 zap 是使用的结构化的日志，所以没有 `%s`、 `%d`之类的标识符需要解析，也是一个性能提升点。

#### 更高效且灵活的序列化器

在 zap 中自己实现了 consoleEncoder、jsonEncoder 两个序列化器，这两个序列化器都可以根据传入的 EncoderConfig 来实现日志格式的灵活配置，这个灵活配置不只是日志输出的 key 的名称，而是通过在 EncoderConfig 中传入函数来调用到用户自定义的 Encoder 实现。

而像 logrus 在序列化 JSON 的时候使用的是标准库的序列化工具，效率也是更低。







## zap 代码分析

由于我感觉 zap 的代码还是写的比较优雅的，所以这里稍微分析一些源码。

### 初始化

#### 初始化 Core

我们在上面的图中也了解到，Core 是有 4个实现类，我们这里以最常用的 ioCore 作为例子来进行讲解。

```
type ioCore struct {
 LevelEnabler
 enc Encoder
 out WriteSyncer
}
```

ioCore 里面非常简单，总共需要三个字段，分别是：输入数据的编码器 Encoder、日志数据的目的地 WriteSyncer，以及日志级别 LevelEnabler。

```
func NewCore(enc Encoder, ws WriteSyncer, enab LevelEnabler) Core {
 return &ioCore{
  LevelEnabler: enab,
  enc:          enc,
  out:          ws,
 }
}
```

在使用 NewCore 函数创建 ioCore 的时候也是返回一个对象指针。

#### 初始化 Logger

zap 会通过 New 函数来实例化一个 Logger：

```
func New(core zapcore.Core, options ...Option) *Logger {
 if core == nil {
  return NewNop()
 }
 log := &Logger{
  core:        core,
  errorOutput: zapcore.Lock(os.Stderr),
  addStack:    zapcore.FatalLevel + 1,
  clock:       _systemClock,
 }
 return log.WithOptions(options...)
}
```

New 函数会设置好相应的默认字段，包括 core 实例、错误日志输出地、堆栈日志的输出级别、日志时间等，然后实例化一个 Logger 对象返回指针。

Logger 结构的信息如下：

```
type Logger struct {
 core zapcore.Core
 // 是否是开发模式
 development bool
 // 是否打印行号
 addCaller   bool
 onFatal     zapcore.CheckWriteAction // default is WriteThenFatal

 name        string
 // 错误日志输出
 errorOutput zapcore.WriteSyncer
 // 输出调用堆栈
 addStack zapcore.LevelEnabler
 // 打印调用者的行号
 callerSkip int

 clock Clock
}
```

Logger 结构体中会包含很多配置信息，我们在开发中可以通过 WithOptions 来添加相应的参数。如添加日志行号：

```
log := zap.New(core).WithOptions(zap.AddCaller())
```

AddCaller 函数会创建一个回调钩子给 WithOptions 执行，这也是函数式编程的魅力所在：

```
func (log *Logger) WithOptions(opts ...Option) *Logger {
 c := log.clone()
 for _, opt := range opts {
    // 调用 Option 接口的方法 
  opt.apply(c)
 }
 return c
}
```

WithOptions 可以传入 Option 数组，然后遍历数组并调用  apply 方法，Option 是一个接口，只提供了 apply 方法：

```
type optionFunc func(*Logger)

func (f optionFunc) apply(log *Logger) {
 f(log)
}
// 定义 Option 接口
type Option interface {
 apply(*Logger)
}

func AddCaller() Option {
  // 返回 Option
 return WithCaller(true)
}

func WithCaller(enabled bool) Option {
  // 将 func 强转成 optionFunc 类型
 return optionFunc(func(log *Logger) {
  log.addCaller = enabled
 })
}
```

这里这个代码写的非常有意思，在 go 中一个函数也是一种类型，和 struct 一样可以用有一个方法。

在这里 optionFunc 作为一种函数类型，它实现了 apply 方法，所以相当于继承了 Option 这个接口。然后在 WithCaller 中使用 optionFunc 将一个函数包了一层，看起来有些奇妙，但是实际上和 `int64(123)`没有本质区别。

然后在 WithOptions 函数中会获取到 WithCaller 返回的这个转成 optionFunc 类型的函数，并传入  log 执行，这样就相当于改变了 log 的 addCaller 属性。

没看懂的可以自己在编译器上试一下。

### 打印日志

整个打印日志的过程如下：

![图片](https://mmbiz.qpic.cn/mmbiz_png/47hDWI1Fd96KV2LiblMCPrx9mUCcV6t3hzcSbUm2wBPpt5jQWyYdickhAONicVk6ukJic4ZDdVbKtZNOgsdWLO355w/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

1. 首先是获取 CheckedEntry 实例，封装相应的日志数据；
2. 然后根据 core 里面封装的 encoder 进行编码，将编码的内容放入到 buffer 中；
3. 将 buffer 中的内容输出到 core 里封装的 WriteSyncer 中。

在我们初始化完成 Logger 后，就可以使用它来调用相应的 Info、Warn、Error 等方法打印日志输出。由于所有的日志级别的输出方法是一样的，所以这里通过 Info 方法来进行分析。

```
func (log *Logger) Info(msg string, fields ...Field) {
    // 检查该日志是否应该被打印
 if ce := log.check(InfoLevel, msg); ce != nil {
        // 打印日志
  ce.Write(fields...)
 }
}
```

这个方法首先会调用 check 方法进行校验，主要是查看在配置的日志等级下当前日志数据是否应该被打印。

对于 Info 日志级别来说会传入 InfoLevel，Error 日志级别来说会传入 ErrorLevel，在 zap 里面日志级别是通过这几个常量来进行定义：

```
type Level int8

const ( 
 DebugLevel Level = iota - 1 
 InfoLevel 
 WarnLevel 
 ErrorLevel 
 DPanicLevel 
 PanicLevel 
 FatalLevel

 _minLevel = DebugLevel
 _maxLevel = FatalLevel
)
```

最小的 DebugLevel 从 -1 开始。

#### check 检查

```
func (log *Logger) check(lvl zapcore.Level, msg string) *zapcore.CheckedEntry { 
 const callerSkipOffset = 2
 // 判断传入的日志等级是否应该打印
 if lvl < zapcore.DPanicLevel && !log.core.Enabled(lvl) {
  return nil
 }
 
 // 将日志数据封装成一个 Entry
 ent := zapcore.Entry{
  LoggerName: log.name,
  Time:       log.clock.Now(),
  Level:      lvl,
  Message:    msg,
 }
 //如果能写日志则返回一个 CheckedEntry 实例指针
 ce := log.core.Check(ent, nil)
 willWrite := ce != nil 
 ... 
 if !willWrite {
  return ce
 }
 
 ce.ErrorOutput = log.errorOutput
 // 判断是否打印调用行号
 if log.addCaller {
  // 获取调用者的栈帧
  frame, defined := getCallerFrame(log.callerSkip + callerSkipOffset)
  if !defined {
   fmt.Fprintf(log.errorOutput, "%v Logger.check error: failed to get caller\n", ent.Time.UTC())
   log.errorOutput.Sync()
  }
  // 设值调用者 entry
  ce.Entry.Caller = zapcore.EntryCaller{
   Defined:  defined,
   PC:       frame.PC,
   File:     frame.File,
   Line:     frame.Line,
   Function: frame.Function,
  }
 }
 if log.addStack.Enabled(ce.Entry.Level) {
  // 封装调用栈信息
  ce.Entry.Stack = StackSkip("", log.callerSkip+callerSkipOffset).String
 } 
 return ce
}
```

这里首先会调用 core 的 Enabled 方法判断一下该日志是否应该被打印。这个判断由于日志等级实际上是一个 int8 类型的，所以直接根据大小直接就可以判断。

```
func (l Level) Enabled(lvl Level) bool {
 return lvl >= l
}
```

判断完没有问题会调用 Check 方法获取 CheckedEntry 实例指针。获取完 CheckedEntry 实例指针后会根据配置信息设值，然后返回。

下面看看是如何获取 CheckedEntry 实例指针。

```
func (c *ioCore) Check(ent Entry, ce *CheckedEntry) *CheckedEntry {
 // 检查该 level 日志是否应该被打印
 if c.Enabled(ent.Level) {
  // 获取 CheckedEntry
  return ce.AddCore(ent, c)
 }
 return ce
}
```

这里会通过 CheckedEntry 的 AddCore 方法获取，需要主要的是传入的 ce 是个 nil 指针，但是这样也不方便 Go 调用其 AddCore 方法（要是放在 java 上该报错了）。

```
var (
 _cePool = sync.Pool{New: func() interface{} {
  // Pre-allocate some space for cores.
  return &CheckedEntry{
   cores: make([]Core, 4),
  }
 }}
)

func (ce *CheckedEntry) AddCore(ent Entry, core Core) *CheckedEntry {
 if ce == nil {
  // 从 _cePool 里面获取 CheckedEntry 实例
  ce = getCheckedEntry()
  ce.Entry = ent
 }
    // 因为可能为 multi core 所以这里需要 append 一下
 ce.cores = append(ce.cores, core)
 return ce
}

func getCheckedEntry() *CheckedEntry {
 // 从 pool 中获取对象
 ce := _cePool.Get().(*CheckedEntry)
 // 重置对象的属性
 ce.reset()
 return ce
}
```

AddCore 方法也是十分简洁，大家应该看一眼就明白了，不多说。

#### Write 日志打印

```
func (ce *CheckedEntry) Write(fields ...Field) {
 if ce == nil {
  return
 }
 ... 
 var err error
 // 遍历所有 core 写入日志数据
 for i := range ce.cores {
  err = multierr.Append(err, ce.cores[i].Write(ce.Entry, fields))
 }
 ...
 // 将 CheckedEntry 放回到缓存池中
 putCheckedEntry(ce)
 ...
}
```

这里就是调用 core 的 Write 方法写日志数据，继续往下。

```
func (c *ioCore) Write(ent Entry, fields []Field) error {
 // 调用 Encoder 的 EncodeEntry 方法将日志数据编码
 buf, err := c.enc.EncodeEntry(ent, fields)
 if err != nil {
  return err
 }
 // 将日志数据通过 WriteSyncer 写入
 _, err = c.out.Write(buf.Bytes())
 // 将buffer放回到缓存池中
 buf.Free()
 if err != nil {
  return err
 }
 if ent.Level > ErrorLevel {
  c.Sync()
 }
 return nil
}
```

Write 方法会根据编码器的不同，然后调用相应编码器的 EncodeEntry 方法。无论是 jsonEncoder 还是 consoleEncoder 都会在 EncodeEntry 方法中从 bufferpool 获取一个 Buffer 实例，然后将数据按照一定的格式封装到 Buffer 实例中。

获取到数据后，会调用 WriteSyncer 的 Write 方法将日志数据写入。

最后将  Buffer 实例释放回 bufferpool 中。

## 总结

这篇文章主要讲解了 zap 的设计原理以及代码的实现。我们可以看到它通过编码结构上的设计使得可以通过简单的配置从而实现丰富的功能。在性能方面，主要是通过使用对象池，减少内存分配的开销；内置高性能序列化器，减少在序列化上面的开销；以及通过结构化的日志格式，减少 `fmt.Sprintf`格式化日志数据的开销。

## Reference

https://mp.weixin.qq.com/s/i0bMh_gLLrdnhAEWlF-xDw

https://segmentfault.com/a/1190000022461706

https://github.com/uber-go/zap
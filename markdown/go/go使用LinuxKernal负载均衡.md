# Go 如何利用 Linux 内核的负载均衡能力

在测试 HTTP 服务时，如果该进程我们忘记关闭，而重新尝试启动一个新的服务进程，那么将会遇到类似以下的错误信息：

```bash
$ go run main.go
listen tcp :8000: bind: address already in use
```

这是由于默认情况下，操作系统不允许我们打开具有相同源地址和端口的套接字 socket。**但如果我们想开启多个服务进程去监听同一个端口，这可以吗？如果可以，这又能给我们带来什么？**

## socket 五元组

socket 编程是每位程序员都应该掌握的基础知识。因此，大家应该知道，socket 连接通过五元组唯一标识。任意两条连接，它的五元组不能完全相同。

```
{<protocol>, <src addr>, <src port>, <dest addr>, <dest port>}
```

`protocol` 指的是传输层 TCP/UDP 协议，它在 socket 被创建时就已经确定。`src addr/port` 与 `dest addr/port` 分别标识着请求方与服务方的地址信息。

因此，只要请求方的 `dest addr/port` 信息不相同，那么服务方即使是同样的 `src addr/port` ，它仍然可以标识唯一的 socket 连接。

基于这个理论基础，那实际上，我们可以在同一个网络主机复用相同的 IP 地址和端口号。

## Linux SO_REUSEPORT

为了满足复用端口的需求，Linux 3.9 内核引入了 `SO_REUSEPORT`选项（实际在此之前有一个类似的选项 `SO_REUSEADDR`，但它没有做到真正的端口复用，详细可见参考链接1）。

`SO_REUSEPORT` 支持多个进程或者线程绑定到同一端口，用于提高服务器程序的性能。它的特性包含以下几点：

- 允许多个套接字 bind 同一个TCP/UDP 端口

- - 每一个线程拥有自己的服务器套接字
  - 在服务器套接字上没有了锁的竞争

- 内核层面实现负载均衡

- 安全层面，监听同一个端口的套接字只能位于同一个用户下（same effective UID）

有了 `SO_RESUEPORT` 后，每个进程可以 bind 相同的地址和端口，各自是独立平等的。

让多进程监听同一个端口，各个进程中 `accept socket fd` 不一样，有新连接建立时，内核只会调度一个进程来 `accept`，并且保证调度的均衡性。

其工作示意图如下

![图片](https://mmbiz.qpic.cn/mmbiz_png/2EiaKLQksVQI4nialLU0R9MuDGV4SoRfYrxrwGicicJMp3BtYGmvlSuhcwqibxcIMcsUe5jHQxCAiaviag99UJG9GRhgQ/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)

有了 `SO_REUSEADDR` 的支持，我们不仅可以创建多个具有相同 `IP:PORT` 的套接字能力，而且我们还得到了一种内核模式下的负载均衡能力。

## Go 如何设置 SO_REUSEPORT

Linux 经典的设计哲学：一切皆文件。当然，socket 也不例外，它也是一种文件。

如果我们想在 Go 程序中，利用上 linux 的 `SO_REUSEPORT` 选项，那就需要有修改内核 socket 连接选项的接口，而这可以依赖于 `golang.org/x/sys/unix` 库来实现，具体就在以下这个方法。

```
import “"golang.org/x/sys/unix"”
...
unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEPORT, 1)
```

因此，一个持有 `SO_REUSEPORT` 特性的完整 Go 服务代码如下

```go
package main

import (
	"context"
	"fmt"
	"net"
	"net/http"
	"os"
	"syscall"

	"golang.org/x/sys/unix"
)

var lc = net.ListenConfig{
	Control: func(network, address string, c syscall.RawConn) error {
		var opErr error
		if err := c.Control(func(fd uintptr) {
			opErr = unix.SetsockoptInt(int(fd), unix.SOL_SOCKET, unix.SO_REUSEPORT, 1)
		}); err != nil {
			return err
		}
		return opErr
	},
}

func main() {
	pid := os.Getpid()
	l, err := lc.Listen(context.Background(), "tcp", "127.0.0.1:8000")
	if err != nil {
		panic(err)
	}
	server := &http.Server{}
	http.HandleFunc("/", func(w http.ResponseWriter, r *http.Request) {
		w.WriteHeader(http.StatusOK)
		fmt.Fprintf(w, "Client [%s] Received msg from Server PID: [%d] \n", r.RemoteAddr, pid)
	})
	fmt.Printf("Server with PID: [%d] is running \n", pid)
	_ = server.Serve(l)
}

```

我们将其编译为 linux 可执行文件 `main`

```bash
$ CGO_ENABLED=0 GOOS=linux GOARCH=amd64 go build main.go
```

在 linux 主机上开启三个同时监听 8000 端口的进程，我们可以看到三个服务进程的 PID 分别是 32687 、32691 和 32697。

```
~ $ ./main &
Server with PID: [32687] is running
~ $ ./main &
Server with PID: [32691] is running
~ $ ./main &
Server with PID: [32697] is running
```

最后，通过 curl 命令，模拟多次 http 客户端请求

```
~ $ for i in {1..20}; do curl localhost:8000; done
Client [127.0.0.1:56876] Received msg from Server PID: [32697]
Client [127.0.0.1:56880] Received msg from Server PID: [32687]
Client [127.0.0.1:56884] Received msg from Server PID: [32687]
Client [127.0.0.1:56888] Received msg from Server PID: [32687]
Client [127.0.0.1:56892] Received msg from Server PID: [32691]
Client [127.0.0.1:56896] Received msg from Server PID: [32697]
Client [127.0.0.1:56900] Received msg from Server PID: [32691]
Client [127.0.0.1:56904] Received msg from Server PID: [32691]
Client [127.0.0.1:56908] Received msg from Server PID: [32697]
Client [127.0.0.1:56912] Received msg from Server PID: [32697]
Client [127.0.0.1:56916] Received msg from Server PID: [32687]
Client [127.0.0.1:56920] Received msg from Server PID: [32691]
Client [127.0.0.1:56924] Received msg from Server PID: [32697]
Client [127.0.0.1:56928] Received msg from Server PID: [32697]
Client [127.0.0.1:56932] Received msg from Server PID: [32691]
Client [127.0.0.1:56936] Received msg from Server PID: [32697]
Client [127.0.0.1:56940] Received msg from Server PID: [32687]
Client [127.0.0.1:56944] Received msg from Server PID: [32691]
Client [127.0.0.1:56948] Received msg from Server PID: [32687]
Client [127.0.0.1:56952] Received msg from Server PID: [32697]
```

可以看到，20 个客户端请求被均衡地打到了三个服务进程上。

## 总结

linux 内核自 3.9 提供的 `SO_REUSEPORT` 选项，可以让多进程监听同一个端口。

这种机制带来了什么：

- **提高服务器程序的吞吐性能**：我们可以运行多个应用程序实例，充分利用多核 CPU 资源，避免出现单核在处理数据包，其他核却闲着的问题。
- **内核级负载均衡**：我们不需要在多个实例前面添加一层服务代理，因为内核已经提供了简单的负载均衡。
- **不停服更新**：当我们需要更新服务时，可以启动新的服务实例来接受请求，再优雅地关闭掉旧服务实例。

如果你们的 Go 项目，一到高峰期就有请求堆积问题，这个时候就可以考虑采用 `SO_REUSEPORT` 选项。
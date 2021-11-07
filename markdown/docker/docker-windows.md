# Windows 上的容器平台工具

> https://docs.microsoft.com/zh-cn/virtualization/windowscontainers/deploy-containers/containerd#



Windows 容器平台正在不断扩展！ Docker 是容器旅程中的第一部分，现在我们正在构建其他容器平台工具。

- [containerd/cri](https://github.com/containerd/cri) - Windows Server 2019/Windows 10 1809 中的新增功能。

- [runhcs](https://github.com/Microsoft/hcsshim/tree/master/cmd/runhcs) - 与 runc 相对应的 Windows 容器主机。

- Hcs

  \- 主机计算服务 + 方便的填充码，使其更易于使用。

  - [hcsshim](https://github.com/microsoft/hcsshim)
  - [dotnet-computevirtualization](https://github.com/microsoft/dotnet-computevirtualization)

本文将讨论 Windows 和 Linux 容器平台以及每个容器平台工具。

## Windows 和 Linux 容器平台

在 Linux 环境中，像 Docker 这样的容器管理工具基于一组更细粒度的容器工具：[runc](https://github.com/opencontainers/runc) 和 [containerd](https://containerd.io/)。

![Linux 上的 Docker 体系结构](https://docs.microsoft.com/zh-cn/virtualization/windowscontainers/deploy-containers/media/docker-on-linux.png)

`runc` 是一个 Linux 命令行工具，用于根据 [OCI 容器运行时规范](https://github.com/opencontainers/runtime-spec)来创建和运行容器。

`containerd` 是一个守护程序，它管理容器生命周期：从下载容器映像到解压缩容器映像，直到容器执行和监督。

在 Windows 上，我们采用了另一种方法。 当我们着手采用 Docker 来支持 Windows 容器时，我们直接基于 HCS（主机计算服务）进行构建。 [此博客文章](https://techcommunity.microsoft.com/t5/Containers/Introducing-the-Host-Compute-Service-HCS/ba-p/382332)详细介绍了我们为什么构建 HCS，以及我们为什么最初采用这种针对容器的方法。

![Windows 上的初始 Docker 引擎体系结构](https://docs.microsoft.com/zh-cn/virtualization/windowscontainers/deploy-containers/media/hcs.png)

目前，Docker 仍直接调用 HCS。 但是，将来的容器管理工具会不断扩展，会包括 Windows 容器和 Windows 容器主机，可以像在 Linux 上调用 containerd 和 runc 那样调用 containerd 和 runhcs。

## runhcs

`runhcs` 是 `runc` 的分叉。 与 `runc` 一样，`runhcs` 是一个命令行客户端，用于运行根据 Open Container Initiative (OCI) 格式打包的应用程序，它是 Open Container Initiative 规范的一个兼容实现。

runc 与 runhcs 之间的功能差异包括：

- `runhcs` 在 Windows 上运行。 它与 [HCS](https://docs.microsoft.com/zh-cn/virtualization/windowscontainers/deploy-containers/containerd#hcs) 进行通信来创建和管理容器。
- `runhcs` 可以运行各种不同的容器类型。
  - Windows 和 Linux [Hyper-V 隔离](https://docs.microsoft.com/zh-cn/virtualization/windowscontainers/manage-containers/hyperv-container)
  - Windows 进程容器（容器映像必须与容器主机匹配）

**用法：**

cmd复制

```cmd
runhcs run [ -b bundle ] <container-id>
```

`<container-id>` 是要启动的容器实例的名称。 该名称在容器主机上必须独一无二。

捆绑包目录（使用 `-b bundle`）是可选的。 与 runc 的用法一样，容器是使用捆绑包配置的。 容器的捆绑包是容器的 OCI 规范文件“config.json”所在的目录。 “bundle”的默认值是当前目录。

OCI 规范文件“config.json”必须有以下两个字段才能正常运行：

- 容器的暂存空间的路径
- 容器的层目录的路径

runhcs 中可用的容器命令包括：

- 用于创建和运行容器的工具
  - **run** 创建并运行容器
  - **create** 创建容器
- 用于管理在容器中运行的进程的工具：
  - **start** 在创建的容器中执行用户定义的进程
  - **exec** 在容器内运行新进程
  - **pause** 挂起容器中的所有进程
  - **resume** 恢复之前已暂停的所有进程
  - **ps** 显示正在容器内运行的进程
- 用于管理容器状态的工具
  - **state** 输出容器的状态
  - **kill** 向容器的 init 进程发送指定的信号（默认值：SIGTERM）
  - **delete** 删除容器拥有的通常与分离的容器一起使用的任何资源

唯一可被视为多容器命令的命令是 **list**。 它列出 runhcs 在给定的根目录中启动的目前处于正在运行状态或已暂停状态的容器。

### HCS

GitHub 上提供了两个可与 HCS 进行交互的包装器。 HCS 是一个 C API，因此使用这些包装器可以轻松地从更高级的语言调用 HCS。

- [hcsshim](https://github.com/microsoft/hcsshim) - HCSShim 以 Go 编写，是 runhcs 的基础。 请从 AppVeyor 获取最新版本，你也可以自行生成它。
- [dotnet-computevirtualization](https://github.com/microsoft/dotnet-computevirtualization) - dotnet-computevirtualization 是 HCS 的 C# 包装器。

如果你希望使用 HCS（直接使用或通过包装器使用），或者希望创建用于 HCS 的 Rust/Haskell/InsertYourLanguage 包装器，请留下评论。

若要更深入地了解 HCS，请观看 [John Stark 的 DockerCon 演示](https://www.youtube.com/watch?v=85nCF5S8Qok)。

## containerd/cri

 重要

只有 Server 2019/Windows 10 版本 1809 及更高版本中提供了 CRI 支持。 我们还在积极开发适用于 Windows 的 containerd。 仅用于开发/测试。

虽然 OCI 规范定义了单个容器，但 [CRI](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/apis/cri/runtime/v1alpha2/api.proto)（容器运行时接口）将容器描述为共享沙盒环境（称为 pod）中的工作负荷。 Pod 可以包含一个或多个容器工作负荷。 Pod 允许容器业务流程协调程序（例如 Kubernetes 和 Service Fabric 网格）处理应当位于同一主机（其中包含一些共享资源，例如内存和 vNET）上的已分组工作负荷。

containerd/cri 为 Pod 启用了以下兼容性矩阵：

| 主机 OS                                 | 容器操作系统                                                 | 隔离                   | 支持 Pod？                                                   |
| :-------------------------------------- | :----------------------------------------------------------- | :--------------------- | :----------------------------------------------------------- |
| Windows Server 2019/1809Windows 10 1809 | Linux                                                        | `hyperv`               | 是—支持真正的多容器 Pod。                                    |
|                                         | Windows Server 2019/1809                                     | `process`* 或 `hyperv` | 是—如果每个工作负荷容器操作系统与实用程序 VM 操作系统匹配，则支持真正的多容器 Pod。 |
|                                         | Windows Server 2016、 Windows Server 1709、 Windows Server 1803 | `hyperv`               | 部分—如果容器操作系统与实用程序 VM 操作系统匹配，则支持 Pod 沙盒，而沙盒则可以支持每个实用程序 VM 采用单个进程隔离容器。 |

*Windows 10 主机仅支持 Hyper-V 隔离

CRI 规范的链接：

- [RunPodSandbox](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/apis/cri/runtime/v1alpha2/api.proto#L24) - Pod 规范
- [CreateContainer](https://github.com/kubernetes/kubernetes/blob/master/pkg/kubelet/apis/cri/runtime/v1alpha2/api.proto#L47) - 工作负荷规范

![基于 Containerd 的容器环境](https://docs.microsoft.com/zh-cn/virtualization/windowscontainers/deploy-containers/media/containerd-platform.png)

虽然 runHCS 和 containerd 都可以在任何 Windows 系统 Server 2016 或更高版本上进行管理，但若要支持 Pod（容器组），需要对 Windows 中的容器工具进行中断性变更。 Windows Server 2019/Windows 10 版本 1809 及更高版本上提供了 CRI 支持。
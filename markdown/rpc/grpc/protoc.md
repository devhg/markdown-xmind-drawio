https://segmentfault.com/a/1190000013339403

https://www.lixueduan.com/post/protobuf/01-import/



假定我们有一个项目需求，希望用`Rpc`作为内部`API`的通讯，同时也想对外提供`Restful Api`，写两套又太繁琐不符合

于是我们想到了`Grpc`以及`Grpc Gateway`，这就是我们所需要的

![grpc-gateway](https://segmentfault.com/img/bV38pY?w=749&h=369)

## 准备环节

在正式开始我们的`Grpc`+`Grpc Gateway`实践前，我们需要先配置好我们的开发环境

- Grpc
- Protoc Plugin
- Protocol Buffers
- Grpc-gateway

- 

### 安装

1、官方推荐（需科学上网）

```maxima
go get -u google.golang.org/grpc
```

2、通过`github.com`

而在`grpc`下有许多常用的包，例如：

- [metadata](https://link.segmentfault.com/?url=https%3A%2F%2Fgowalker.org%2Fgoogle.golang.org%2Fgrpc%2Fmetadata)：定义了`grpc`所支持的元数据结构，包中方法可以对`MD`进行获取和处理
- [credentials](https://link.segmentfault.com/?url=https%3A%2F%2Fgowalker.org%2Fgoogle.golang.org%2Fgrpc%2Fcredentials)：实现了`grpc`所支持的各种认证凭据，封装了客户端对服务端进行身份验证所需要的所有状态，并做出各种断言
- [codes](https://link.segmentfault.com/?url=https%3A%2F%2Fgowalker.org%2Fgoogle.golang.org%2Fgrpc%2Fcodes)：定义了`grpc`使用的标准错误码，可通用

## Protoc Plugin

编译器插件

### 安装

```vim
go get -u github.com/golang/protobuf/protoc-gen-go
```



## Protocol Buffers v3

### 是什么

> Protocol buffers are a flexible, efficient, automated mechanism for serializing structured data – think XML, but smaller, faster, and simpler. You define how you want your data to be structured once, then you can use special generated source code to easily write and read your structured data to and from a variety of data streams and using a variety of languages. You can even update your data structure without breaking deployed programs that are compiled against the "old" format.

```
Protocol Buffers`是`Google`推出的一种数据描述语言，支持多语言、多平台，它是一种二进制的格式，总得来说就是更小、更快、更简单、更灵活，目前分别有`v2`、`v3`的版本，我们推荐使用`v3
```

- [proto2 文档地址](https://link.segmentfault.com/?url=https%3A%2F%2Fdevelopers.google.com%2Fprotocol-buffers%2Fdocs%2Fproto)
- [proto3 文档地址](https://link.segmentfault.com/?url=https%3A%2F%2Fdevelopers.google.com%2Fprotocol-buffers%2Fdocs%2Fproto3)

建议可以阅读下官方文档的介绍，本系列会在使用时简单介绍所涉及的内容

### 安装

```awk
wget https://github.com/google/protobuf/releases/download/v3.5.1/protobuf-all-3.5.1.zip
unzip protobuf-all-3.5.1.zip
cd protobuf-3.5.1/
./configure
make
make install
```

检查是否安装成功

```ada
protoc --version
```

如果出现报错

```stata
protoc: error while loading shared libraries: libprotobuf.so.15: cannot open shared object file: No such file or directory
```

则执行`ldconfig`后，再次运行即可成功

#### 为什么要执行`ldconfig`

我们通过控制台输出的信息可以知道，`Protocol Buffers Libraries`的默认安装路径在`/usr/local/lib`

```stata
Libraries have been installed in:
   /usr/local/lib

If you ever happen to want to link against installed libraries
in a given directory, LIBDIR, you must either use libtool, and
specify the full pathname of the library, or use the `-LLIBDIR'
flag during linking and do at least one of the following:
   - add LIBDIR to the `LD_LIBRARY_PATH' environment variable
     during execution
   - add LIBDIR to the `LD_RUN_PATH' environment variable
     during linking
   - use the `-Wl,-rpath -Wl,LIBDIR' linker flag
   - have your system administrator add LIBDIR to `/etc/ld.so.conf'

See any operating system documentation about shared libraries for
more information, such as the ld(1) and ld.so(8) manual pages.
```

而我们安装了一个新的动态链接库，`ldconfig`一般在系统启动时运行，所以现在会找不到这个`lib`，因此我们要手动执行`ldconfig`，**让动态链接库为系统所共享，它是一个动态链接库管理命令**，这就是`ldconfig`命令的作用







### protoc使用

我们按照惯例执行`protoc --help`（查看帮助文档），我们抽出几个常用的命令进行讲解

1、`-IPATH, --proto_path=PATH`：指定`import`搜索的目录，可指定多个，如果不指定则默认当前工作目录

2、`--go_out`：生成`golang`源文件

#### 参数

若要将额外的参数传递给插件，可使用从输出目录中分离出来的逗号分隔的参数列表:

```ceylon
protoc --go_out=plugins=grpc,import_path=mypackage:. *.proto
```

- `import_prefix=xxx`：将指定前缀添加到所有`import`路径的开头
- `import_path=foo/bar`：如果文件没有声明`go_package`，则用作包。如果它包含斜杠，那么最右边的斜杠将被忽略。
- `plugins=plugin1+plugin2`：指定要加载的子插件列表（我们所下载的repo中唯一的插件是grpc）
- `Mfoo/bar.proto=quux/shme`： `M`参数，指定`.proto`文件编译后的包名（`foo/bar.proto`编译后为包名为`quux/shme`）

#### Grpc支持

如果`proto`文件指定了`RPC`服务，`protoc-gen-go`可以生成与`grpc`相兼容的代码，我们仅需要将`plugins=grpc`参数传递给`--go_out`，就可以达到这个目的

```jboss-cli
protoc --go_out=plugins=grpc:. *.proto
```





## Grpc-gateway

### 是什么

> grpc-gateway is a plugin of protoc. It reads gRPC service definition, and generates a reverse-proxy server which translates a RESTful JSON API into gRPC. This server is generated according to custom options in your gRPC definition.

[grpc-gateway](https://link.segmentfault.com/?url=https%3A%2F%2Fgithub.com%2Fgrpc-ecosystem%2Fgrpc-gateway)是protoc的一个插件。它读取gRPC服务定义，并生成一个反向代理服务器，将RESTful JSON API转换为gRPC。此服务器是根据gRPC定义中的自定义选项生成的。

### 安装

```vim
go get -u github.com/grpc-ecosystem/grpc-gateway/protoc-gen-grpc-gateway
```


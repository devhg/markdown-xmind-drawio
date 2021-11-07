我们总有这样的担忧：总有刁民想害朕，总有人偷偷在目录下删改文件，**高危操作想第一时间了解，怎么办**？ 而且通常我们还有这样的需求：

- 监听一个目录中所有文件，文件大小到一定阀值，则处理；
- 监控某个目录，当有文件新增，立马处理；
- 监控某个目录或文件，当有文件被修改或者删除，立马能感知，进行处理；

怎么做到这个事情呢？**最常见的通常有三个办法**：

1. 第一种：**当事人**主动通知你，这是侵入式的，需要当事人修改这部分代码来支持，依赖于当事人的自觉；
2. 第二种：**轮询观察**，这个是无侵入式的，你可以自己写个轮询程序，每隔一段时间唤醒一次，对文件和目录做各种判断，从而得到这个目录的变化；
3. 第三种：**操作系统支持**，以事件的方式通知到订阅这个事件的用户，达到及时处理的目的；

**很明显，第三种最好：**

1. 纯旁路的逻辑，对线上程序无侵入；
2. 操作系统直接支持，以事件的形式通知，性能也最好，100% 准确率（比较自己轮询判断要好）；

**怎么做到这个事情呢？**

既然是操作系统的支持，那么就涉及到系统调用。系统调用直接使用略微复杂了些，Go 里面有个库 fsnotify ，就是封装了系统调用，用来监控文件事件的。当指定目录或者文件，发生了创建，删除，修改，重命名的事件，里面就能得到通知。





## Go 的 fsnotify 的使用

使用方法非常简单：

1. 先用 fsnotify 创建一个监听器；
2. 然后放到一个单独的 Goroutine 监听事件即可，通过 channel 的方式传递；

```go
package main

import (
    "log"
    "github.com/fsnotify/fsnotify"
)

func main() {
    // 创建文件/目录监听器
    watcher, err := fsnotify.NewWatcher()
    if err != nil {
        log.Fatal(err)
    }
    defer watcher.Close()
    done := make(chan bool)
    go func() {
        for {
            select {
            case event, ok := <-watcher.Events:
                if !ok {
                    return
                }
                // 打印监听事件
                log.Println("event:", event)
            case _, ok := <-watcher.Errors:
                if !ok {
                    return
                }
            }
        }
    }()
    // 监听当前目录
    err = watcher.Add("./")
    if err != nil {
        log.Fatal(err)
    }
    <-done
}
```

先把上述程序编译，然后跑起来：

```
root@ubuntu:~/code/gopher/src/notify# ./notify 
```

再打开一个终端，准备进行你的操作：先 `touch` 一个新文件 hello.txt

```
touch hello.txt
```

使用 vim 打开这个文件，写入一行数据，然后关闭退出：

```bash
vim hello.txt
root@ubuntu:~/code/gopher/src/notify# ./notify 
# 触发事件：创建的时候
2021/08/20 17:02:52 event: "./hello.txt": CREATE
2021/08/20 17:02:52 event: "./hello.txt": CHMOD
# 触发事件：vim 打开初始化的时候（创建 swp 文件）
2021/08/20 17:17:08 event: "./.hello.txt.swp": CREATE
2021/08/20 17:17:08 event: "./.hello.txt.swx": REMOVE
2021/08/20 17:17:08 event: "./.hello.txt.swp": REMOVE
2021/08/20 17:17:08 event: "./.hello.txt.swp": CREATE
2021/08/20 17:17:08 event: "./.hello.txt.swp": WRITE
2021/08/20 17:17:08 event: "./.hello.txt.swp": CHMOD
# 触发事件：:w 写入保存的时候
2021/08/20 17:17:53 event: "./4913": REMOVE
2021/08/20 17:17:53 event: "./hello.txt": RENAME
2021/08/20 17:17:53 event: "./hello.txt~": CREATE
2021/08/20 17:17:53 event: "./hello.txt": CREATE
2021/08/20 17:17:53 event: "./hello.txt": WRITE
2021/08/20 17:17:53 event: "./hello.txt": CHMOD
2021/08/20 17:17:53 event: "./hello.txt": CHMOD
2021/08/20 17:17:53 event: "./hello.txt~": REMOVE
# 触发事件：:q 的退出时候
2021/08/20 17:17:57 event: "./.hello.txt.swp": WRITE
2021/08/20 17:18:11 event: "./.hello.txt.swp": REMOVE
```

惊喜就是，这里能和之前 [Linux 编辑器之神 vim 的 IO 存储原理](http://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247492777&idx=1&sn=1275fcad5ea72d36df24c5efe6f3f4c5&chksm=cf3df46cf84a7d7aab1509bbcf3c0b2bd4a9f3231e51ac8f4f863e8491f174b7aaec1c18cf8a&scene=21#wechat_redirect) 篇能结合上：

1. 看到了 ~ 镜像文件，还看到了 swp 文件，竟然还看到了 一个 4913 的文件（这个文件也是个临时文件，感兴趣的可以了解一下）；

**太神奇了，这样你就有一个新的手段监控你的文件发生的任何事情了。**这是什么原理呢？





## 深层原理



fsnotify 是跨平台的实现，奇伢这里只讲 Linux 平台的实现机制。fsnotify 本质上就是对系统能力的一个浅层封装，主要封装了操作系统提供的两个机制：

1. inotify 机制；
2. epoll 机制；

旁白：真的是何处都有 epoll 呀。如果还有对 epoll 不明白的赶紧复习下 Linux fd 系列，[深度 epoll 剖析](http://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247492165&idx=1&sn=b7556601db1d4118ea9188945cb891aa&chksm=cf3df280f84a7b96a6247a59218bc30ac2487d14905924a2e64568bfe21762157595316b909c&scene=21#wechat_redirect)。





### inotify 机制



什么是 inotify 机制？

这是一个内核用于通知用户空间程序**文件系统变化**的机制。

划重点：其实 inotify 机制的诞生源于一个通用的需求，由于IO/硬件管理都在内核，但用户态是有获悉内核事件的强烈需求，比如磁盘的热插拔，文件的增删改。这里就诞生了三个异曲同工的机制：hotplug 机制、udev 管理机制、inotify 机制。



#### inotify 的三个接口

操作系统提供了三个接口来支撑，非常简洁：

```
// fs/notify/inotify/inotify_user.c

// 创建 notify fd
inotify_init1

// 添加监控路径
inotify_add_watch

// 删除一个监控
inotify_rm_watch
```

用法非常简单，分别对应 inotify fd 的创建，监控的添加和删除。



#### inotify 怎么实现监控的

inotify 支持监听的事件非常多，除了增删改，还有访问，移动，打开，关闭，设备卸载等等事件。内核要上报这些文件 api 事件必然要采集这些事件。在哪一个内核层次采集的呢？

```
系统调用 -> vfs -> 具体文件系统（ ext4 ）-> 块层 -> scsi 层
```

**答案是：vfs 层。**其实这个很容易理解，这是必然的，因为这是所有“文件”操作的入口。

以 vfs 的 read/write 为例，我们看一下：

```c
ssize_t vfs_read(struct file *file, char __user *buf, size_t count, loff_t *pos)
{
    // ...
    ret = __vfs_read(file, buf, count, pos);
    if (ret > 0) {
        // 事件采集点：访问事件
        fsnotify_access(file);
    }

}

ssize_t vfs_write(struct file *file, const char __user *buf, size_t count, loff_t *pos)
{
    // ...
    ret = __vfs_write(file, buf, count, pos);
    if (ret > 0) {
        // 事件采集点：修改事件
        fsnotify_modify(file);
    }
}
```

`fsnotify_access` 和 `fsnotify_modify` 就是 inotify 机制的一员。有一系列 `fsnotify_xxx` 的函数，定义在 `include/linux/fsnotify.h` ，这函数里面全都调用到 `fsnotify` 这个函数。

```c
static inline void fsnotify_modify(struct file *file)
{
    // 获取到 inode

    if (!(file->f_mode & FMODE_NONOTIFY)) {
        fsnotify_parent(path, NULL, mask);
        // 采集事件，通知到指定结构
        fsnotify(inode, mask, path, FSNOTIFY_EVENT_PATH, NULL, 0);
    }
}
```

来看一下 `fsnotify` 的函数实现，我们简单的梳一下调用栈：

```
fsnotify
    -> send_to_group
        -> inotify_handle_event
            -> fsnotify_add_event
                -> wake_up （唤醒等待队列，也就是 epoll）
```

再看一眼具体的实现（其实非常简单，就是一个事件通知）：

```c
// 把事件通知到相应的 group 上；
int fsnotify(struct inode *to_tell, __u32 mask, const void *data, int data_is, const unsigned char *file_name, u32 cookie)
{
        // ...
        // 把事件通知给正在监听的 fsnotify_group
        while (fsnotify_iter_select_report_types(&iter_info)) {
                ret = send_to_group(to_tell, mask, data, data_is, cookie, file_name, &iter_info);
                if (ret && (mask & ALL_FSNOTIFY_PERM_EVENTS))
                        goto out;
                fsnotify_iter_next(&iter_info);
        }
out:
        return ret;
}

static int send_to_group(struct inode *to_tell, __u32 mask, const void *data, int data_is, u32 cookie, const unsigned char *file_name, struct fsnotify_iter_info *iter_info)
{
    // 通知相应的 group ，有事来了！
    return group->ops->handle_event(group, to_tell, mask, data, data_is, file_name, cookie, iter_info);
}

// group->ops->handle_event 被赋值为 inotify_handle_event

int inotify_handle_event(struct fsnotify_group *group, struct inode *inode, u32 mask, const void *data, int data_type, const unsigned char *file_name, u32 cookie, struct fsnotify_iter_info *iter_info)
{
    // 唤醒事件，通知相应的 group
    ret = fsnotify_add_event(group, fsn_event, inotify_merge);
}


// 添加事件到 group 
int fsnotify_add_event(struct fsnotify_group *group, struct fsnotify_event *event, int (*merge)(struct list_head *, struct fsnotify_event *))
{
    // 唤醒这个等待队列
    wake_up(&group->notification_waitq);
}
```

这里面的逻辑非常简单：**把这次的事件通知给关注的 `fsnotify_group` 结构体**，换句话说，就是把事件通知给 inotify fd。

这个就有意思了，inotify fd 句柄创建的时候，`file->private_data` 上就绑定了一个 `fsnotify_group` ，这就对上了。这样的话，针对文件的所有操作，都能有一份事件发送到 `fsnotify_group` 上，inotify fd 就有可读事件了。



### epoll 机制



在前面我们也提到了，Go 的 fsnotify 主要使用了两个系统机制 inotify 机制和 epoll 机制。fsnotify 把 inotify fd 放到 epoll 池里面管理。

换句话说，**inotify fd 支持 epoll 机制**。**划重点：有最明显的两个特征**：

1. inotify fd 的 `inotify_fops` 实现了 `.poll` 接口；
2. inotify fd 相关的某个结构体一定**有个 wait 队列的表头**；

这个结构体是啥？

其实跟 [timerfd](http://mp.weixin.qq.com/s?__biz=Mzg3NTU3OTgxOA==&mid=2247493549&idx=1&sn=8aef4d8825229316ab51e65e5218618a&chksm=cf3df768f84a7e7e56f4e040255569e18fafea470aef1c2768f0db58ec7414f313e2465afeb0&scene=21#wechat_redirect) 类似（读者有不熟悉的，可以去复习下哦），笔者直接揭秘啦，这个结构体就是 `fsnotify_group` 。被存放在 inotify fd 对应的 `file->private_data` 字段。这个 wait 队列表头就是 `group->notification_waitq` 。

来看一眼结构体的简要关系：

![图片](https://mmbiz.qpic.cn/mmbiz_png/4UtmXsuLoNcWBKvmaWDNr3aWaR4SgZU1L5NiaF4rAtXetkXNxW8X7wbzgFlwuZiaMWBT5gy89ug0223IialTNNclw/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)



###  

 **2**  **epoll 机制**



回到 Go 的 fsnotify 库的实现原理，fsnotify 利用的第二个系统机制就是 epoll 。inotify fd 通过 `inotify_init1` 创建出来之后，会把 inotify fd 注册进 epoll 管理，监听 inotify fd 的可读事件。

**inotify fd 的可读事件能是啥？**

就是它监听的文件或者路径发生的增删改的事件嘛，这些事件就是内核 inotify 报上来的。

报上来之后，epoll 监控到 inotify fd 可读，用户通过 read 调用，把 inotify fd 里面的“数据”读出来。这个读出来的所谓的“数据”就是一个个文件事件。

我们看一眼整体的模块层次：

![图片](https://mmbiz.qpic.cn/mmbiz_png/4UtmXsuLoNcWBKvmaWDNr3aWaR4SgZU1crHq6hu6W4W5HKq2nrV7qo2Z6kiaU9ia1vicfz77FvdTGRzUt1BoTCJ6Q/640?wx_fmt=png&tp=webp&wxfrom=5&wx_lazy=1&wx_co=1)







## 总结

1. Go 的 fsnotify 库很方便对文件、目录做监控，这里的充满了想象力，**因为一切皆文件，这代表着一切可监控**。童鞋们，这里的想象空间非常大哦；
2. 通过 fsnotify 我们映证了 vim 的秘密；
3. Go 的 fsnotify 其实操作系统能力的浅层封装，Linux 本质就是对 inotify 机制；
4. inotify 也是一个特殊句柄，属于匿名句柄之一，这个句柄用于**文件的事件监控**；
5. fsnotify 用 epoll 机制对 inotify fd 的**可读事件**进行监控，实现 IO 多路复用的事件通知机制；
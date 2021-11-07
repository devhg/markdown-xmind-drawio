# [pkg-config原理及用法](https://www.cnblogs.com/sddai/p/10266624.html)

原文 https://blog.csdn.net/luotuo44/article/details/24836901

我们在用第三方库的时候，经常会用到pkg-config这个东西来编译程序。那pkg-config究竟是什么呢？

 

# pkgconfig有什么用：

大家应该都知道用第三方库，就少不了要使用到第三方的头文件和库文件。我们在编译、链接的时候，必须要指定这些头文件和库文件的位置。

对于一个比较大第三方库，其头文件和库文件的数量是比较多的。如果我们一个个手动地写，那将是相当麻烦的。所以，pkg-config就应运而生了。pkg-config能够把这些头文件和库文件的位置指出来，给编译器使用。

如果你的系统装有gtk，可以尝试一下下面的命令`pkg-config --cflags gtk+-2.0`。可以看到其输出是gtk的头文件的路径。

我们平常都是这样用pkg-config的。$gcc main.c `pkg-config --cflags --libs gtk+-2.0` -o main

上面的编译命令中，`pkg-config --cflags --libs gtk+-2.0`的作用就如前面所说的，把gtk的头文件路径和库文件列出来，让编译去获取。--cflags和--libs分别指定头文件和库文件。

Ps:命令中的`不是引号，而是数字1左边那个键位的那个符号。

 

其实，pkg-config同其他命令一样，有很多选项，不过我们一般只会用到--libs和--cflags选项。更多的选项可以在[这里](http://linux.die.net/man/1/pkg-config)查看。

 

# 配置环境变量：

看到这里，大家可能想试一下将pkg-config用于自己的库。下面就说一下，怎么写。

首先要明确一点，因为pkg-config也只是一个命令，所以不是你安装了一个第三方的库，pkg-config就能知道第三方库的头文件和库文件所在的位置。pkg-config命令是通过查询XXX.pc文件而知道这些的。我们所需要做的是，写一个属于自己的库的.pc文件。

但pkg-config又是如何找到所需的.pc文件呢？这就需要用到一个环境变量PKG_CONFIG_PATH了。这环境变量写明.pc文件的路径，pkg-config命令会读取这个环境变量的内容，这样就知道pc文件了。

现在pkg-config能找到我们的.pc文件。但如果有多个.pc文件，那么pkg-config又怎么能正确找到我想要的那个呢？这就需要我们在使用pkg-config命令的时候去指定。比如$gcc main.c `pkg-config --cflags --libs gtk+-2.0` -o main就指定了要查找的.pc文件是gtk+-2.0.pc。又比如，有第三方库OpenCV，而且其对应的pc文件为opencv.pc，那么我们在使用的时候，就要这样写`pkg-config --cflags --libs opencv`。这样，pkg-config才会去找opencv.pc文件。

 

# pc文件书写规范：

​    好了，现在我们开始写自己的.pc文件。只需写5个内容即可：Name、Description、Version、Cflags、Libs。

​    比如简单的：

1. Name: opencv
2.  Description:OpenCV pc file
3.  Version: 2.4
4.  Cflags:-I/usr/local/include
5.  Libs:-L/usr/local/lib –lxxx –lxxx

 其中Name对应的内容要和这个pc文件的文件名一致。当然为了书写方便还会加入一些变量，比如前缀变量prefix。下面有一个更完整的pc文件的内容

​    ![img](https://img-blog.csdn.net/20140501110745640?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvbHVvdHVvNDQ=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)

​    其中，Cflags和Libs的写法，是使用了-I -L -l这些gcc的编译选项。原理可以参考我的一篇[博文](http://blog.csdn.net/luotuo44/article/details/16970841)。


### Seccomp机制简介

seccomp是一种内核中的安全机制,正常情况下,程序可以使用所有的syscall,这是不安全的,比如劫持程序流后通过execve的syscall来getshell.通过seccomp我们可以在程序中禁用掉某些syscall,这样就算劫持了程序流也只能调用部分的syscall了.

### 2、 使用seccomp

```c
#include <stdio.h>
#include <unisted.h>
#include <seccomp.h>

int main(void){

         scmp_filter_ctx ctx;

         ctx = seccomp_init(SCMP_ACT_ALLOW);

         seccomp_rule_add(ctx, SCMP_ACT_KILL, SCMP_SYS(execve), 0);

         seccomp_load(ctx);

         char * filename = "/bin/sh";

         char * argv[] = {"/bin/sh",NULL};

         char * envp[] = {NULL};

         write(1,"i will give you a shell\n",24);

         syscall(59,filename,argv,envp);//execve

         return 0;
}
1234567891011121314151617181920212223242526
```

### 今天做题遇到的沙盒机制

通过prctl(PR_SET_SECCOMP, 2, …)设置沙盒，禁用一些syscall，
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191008193819670.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxNzA2OTI0,size_16,color_FFFFFF,t_70)
PR_SET_NO_NEW_PRIVS=38
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191008194013711.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxNzA2OTI0,size_16,color_FFFFFF,t_70)

### 使用工具

seccomp（github上）可以识别沙盒的规则，还可以自己写规则
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191008194219653.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzQxNzA2OTI0,size_16,color_FFFFFF,t_70)
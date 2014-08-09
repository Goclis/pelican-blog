Title: 操作系统课设笔记  
Slug: os-experiment-note  
Date: 2014-07-30 17:22  
Tags: os  
Author: Goclis Yao  


因为实习，没法一次性把课设做完，就干脆些把笔记写成博文方便自己查阅吧，持续更新，直至完成。

这课设的设计基本上是依赖于一本书的，大概是怕我们遇到太多的坑吧。。而 OS 上的坑又不是那么容易解决的，总之以下操作不带有普遍性。

课设平台：

 - fedora 7
 - linux-2.6.21 source code
 - i386 ?

###实验1 编译内核
纯粹照着书搞。。

以下操作假设在 home 目录下，并且用户为 root
```
tar -zxf linux-2.6.21.tar.gz
cd linux-2.6.21
make mrproper  # 清楚掉之前编译产生的垃圾
make oldconfig  # 如果自己有配置文件的话，直接 copy 进来命名为 .config 即可
vim Makefile
```
修改
```
EXTARVERSION = 
```
为
```
EXTRAVERSION = .7-custom-version
```
这样的话你生成的内核版本号就为 2.6.21.7-custom-version （前面有几个参数是指定 2,6,2 的，这里没给出）

继续

```
make all
make modules_install
make install
```
这样的话新的内核就已经更新到了 /boot 下，并且 grub 的配置文件 (/etc/grub.conf) 也已经被更新了。

有兴趣研究的同学可以继续去查看 terminal 给出的那个 sh 命令之后的那个文件，进而会跟到另外一个文件里面，我只是粗略看了下没有深究，就不多说了。

最后一步 reboot，就可以看到在 grub 的选项中看到你的内核了（.7-custom-version 结尾的那个）。



###实验2 系统调用
这里只是添加一个最简单的系统调用，无参数，无实际操作！

加复杂的系统调用的时候可能会遇到各种坑，在做往后几个实验的时候再去解决吧。

假设要添加的系统调用名为 goclis，则内核对应的实现函数名一般为 sys_goclis，现在开始实现。

目录和用户假设同实验1

```
cd linux-2.6.21
vim arch/i386/kernel/syscall_table.S
```
往系统调用表的末尾添加要新增的系统调用
```
.long sys_tee       /* 315 */
.long sys_vmsplice
# 省略几个。。

.long sys_goclis    /* 320 */
```
320 指的是我要为它分配的系统调用号，继续

添加头文件 goclis.h
```
vim include/linux/goclis.h
```
正如我所说，这个系统调用非常简单，所以。。头文件除了宏保护就是空。。
```
// goclis.h

#ifndef _LINUX_GOCLIS_H
#define _LINUX_GOCLIS_H

#endif
```
添加实现文件 goclis.c
```
vim kernel/goclis.c
```
在里面要加入我们指定的系统调用的实现
```
// goclis.c

#include <linux/linkage.h>
#include <linux/goclis.h>

asmlinkage int sys_goclis()
{
    return 320;  // 这个是随意的
}
```
就是这么简单的系统调用，继续，为该系统调用添加支持
```
vim kernel/Makefile
```
在
```
obj-y = sched.o fork.o exec_domain.o \
        ...
```
这行加入
```
obj-y = goclis.o sched.o ...
```
这样才能保证 goclis.c 在内核编译的时候是可见的。

但其实这个调用的实现也不一定要写在 goclis.c 中，完全可以放在一个已经存在的 .c 中即可，只要保证这个 .c 对应的 .o 能被内核编译。

接着为这个系统调用定义系统调用，我们之前那个 320 只是个注释呀

```
vim include/asm-i386/unistd.h
```
是 i386 不是 ia64，因为我们是 32 位的系统。

修改
```
#define __NR_epoll_wait     319

#ifdef __KERNEL__

#define NR_syscalls         320
```
为
```
#define __NR_epoll_wait     319
#define __NR_goclis         320

#ifdef __KERNEL__

#define NR_syscalls         321
```
很显然吧，用 __NR_goclis 来命名我们的系统调用，然后把它定义为 320，这样就真的 320 了。。

最后，还得把你的这个系统调用加入到统一的系统调用头文件中。

```
vim include/linux/syscalls.h
```
加入头文件
```
#include <linux/goclis.h>
```
加入声明
```
asmlinkage int sys_goclis();
```

OK，重新编译内核然后 reboot，就 ok 了！

关于编译内核，因为之前已经编译过产生了很多的 .o 了，节省时间的话，直接 make 就可以接着编译了，否则重头编译超级浪费时间。。

重启后测试下我们的系统调用是否生效，注意得重启为我们创的这个内核！

```
// test.c
#include <sys/syscall.h>
#include <unistd.h>

int main()
{
    int ret = syscall(320);
    printf("%d\n", ret);
    return 0;
}
```
编译后运行
```
gcc test.c
./a.out

> 320
```
ok~，打印出 320 了。
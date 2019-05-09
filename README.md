# GDB 教程

本人做服务器开发，看日志和重审代码可以解决90%左右的问题，前者而且可以打印调用栈，剩下的很多问题要依赖工具，比如gdb，gdb查看崩溃时的core也是极好。
写这个目的就是给想要快速入门gdb的同学的，如果要深入理解gdb，还是要从源码入手才行。

## 内容

- [原理](#原理)
- [启动gdb](#启动gdb)
- [退出gdb](#退出gdb)
- [为gdb进行编译](#为gdb进行编译)
- [调试程序](#调试程序)
- [CoreDump简单概念](#CoreDump简单概念)
- [产生CoreDump文件](#产生CoreDump文件)
- [调试CoreDump文件](#调试CoreDump文件)
- [help命令](#help命令)
- [list命令](#list命令)
- [start命令](#start命令)
- [next命令](#next命令)
- [step命令](#step命令)
- [break命令](#break命令)
- [查看断点](#查看断点)
- [删除断点](#删除断点)
- [tbreak命令](#tbreak命令)
- [continue命令](#continue命令)
- [backtrace命令](#backtrace命令)
- [查看当前所处的函数堆栈帧](#查看当前所处的函数堆栈帧)
- [选择函数堆栈帧](#选择函数堆栈帧)
- [打印函数局部变量](#打印函数局部变量)
- [run命令](#run命令)
- [修改变量值](#修改变量值)
- [查看变量类型](#查看变量类型)
- [查看线程运行](#查看线程运行)


## 原理
断点功能一般是通过gdb捕获特定的内核信号来实现的，然后定位目标程序停止的地址来判断断点是否成功触发。大致的流程为，
首先gdb fork()出来一个子进程，该子进程启动目标程序(通过ptrace() 和 exec())，
父进程捕获该子进程的所有的信号(通过ptrace() 和 wait())，当子进程收到信号时，子进程就会被挂起，直到父进程通知其继续运行(通过ptrace())

## 启动gdb
1 常规启动，非常多的提示信息:

    $ gdb
    GNU gdb (GDB) Red Hat Enterprise Linux 7.6.1-114.el7
    Copyright (C) 2013 Free Software Foundation, Inc.
    License GPLv3+: GNU GPL version 3 or later <http://gnu.org/licenses/gpl.html>
    This is free software: you are free to change and redistribute it.
    There is NO WARRANTY, to the extent permitted by law.  Type "show copying"
    and "show warranty" for details.
    This GDB was configured as "x86_64-redhat-linux-gnu".
    For bug reporting instructions, please see:
    <http://www.gnu.org/software/gdb/bugs/>.
    (gdb)
    
2 简约启动，关闭提示信息:

    $ gdb -q
    (gdb)

## 退出gdb
1 输入quit:

    $ gdb -q
    (gdb) quit

2 输入Ctrl-d:

    $ gdb -q
    (gdb) quit
    
    
## 为gdb进行编译

    为了获得调试信息，需要添加 CFLAGS=-g -o0 选项
具体参考[gdb手册](https://sourceware.org/gdb/current/onlinedocs/gdb/Compilation.html#Compilation)

## 调试程序

```c 
//boom.c
#include <stdio.h>
#include <unistd.h>

void fun(void)
{
    printf("hello\n");
}
    
int main()
{
    fun();
    sleep(1000);
    return 0;
}
```

1 直接启动:

方法1

    $ gdb boom -q
    (gdb) 

方法2

    $ gdb -q
    (gdb) file boom 
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb)
   
2 调试正在运行的程序:
    
    $ ps ux | grep boom | grep -v 'grep' 
    dan       5647  0.0  0.0  11520   472 pts/0    S+   15:31   0:00 ./boom
    
    
    $ gdb boom 5647 -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    Attaching to program: /home/dan/work/learn_core/build/bin/boom, process 5647
    Reading symbols from /lib64/libpthread.so.0...(no debugging symbols found)...done.
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".
    Loaded symbols for /lib64/libpthread.so.0
    Reading symbols from /lib64/libdl.so.2...(no debugging symbols found)...done.
    Loaded symbols for /lib64/libdl.so.2
    Reading symbols from /lib64/libm.so.6...(no debugging symbols found)...done.
    Loaded symbols for /lib64/libm.so.6
    Reading symbols from /lib64/libc.so.6...(no debugging symbols found)...done.
    Loaded symbols for /lib64/libc.so.6
    Reading symbols from /lib64/ld-linux-x86-64.so.2...(no debugging symbols found)...done.
    Loaded symbols for /lib64/ld-linux-x86-64.so.2
    0x00007f1e2185be10 in __nanosleep_nocancel () from /lib64/libc.so.6
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) 

具体参考[gdb手册](https://sourceware.org/gdb/onlinedocs/gdb/Invoking-GDB.html#Invoking-GDB)

## CoreDump简单概念

CoreDump即核心转储，是程序运行异常崩溃时，系统内核为该程序产生的内存、寄存器、运行栈等快照，并保存为一个二进制文件，可以利用该文件进行GDB调试，发现运行错误。
    
## 产生CoreDump文件  

查看系统是否已经开启了该功能:
    
    $ ulimit -c
    0
上述输出结果为0说明当前系统已经关闭了该功能，所以需要打开该功能:

临时启用
    
    $ ulimit -c unlimited
    $ ulimit -c
    unlimited

永久启用

    在/etc/security/limits.conf添加一行:
    *       soft    core    unlimited
    
    修改文件格式，例如：
    echo "core.%e.%p.%t" >/proc/sys/kernel/core_pattern
    or
    echo "/home/dan/mycore/core.%e.%p.%t" >/proc/sys/kernel/core_pattern
    
## 调试CoreDump文件

```c 
#include <stdio.h>
#include <unistd.h>
 
int main()
{
    int* p = NULL;
    printf("here");
    *p = 7;
    sleep(1000);
    return 0;
}
```
运行上述程序就会产生core dump 文件，如果修了core dump文件的位置就需要加上绝对地址如:
    
方法1

    $ gdb /home/dan/work/learn_core/build/bin/boom /home/dan/mycore/core.boom.5859.1557305516 -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    [New LWP 5859]
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".
    Core was generated by `./boom'.
    Program terminated with signal 11, Segmentation fault.
    #0  0x0000000000400770 in main () at /home/dan/work/learn_core/boom.c:8
    8           *p = 7;
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) 

方法2

    gdb -q
    (gdb) file /home/dan/work/learn_core/build/bin/boom
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb) core /home/dan/mycore/core.boom.5859.1557305516
    [New LWP 5859]
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".
    Core was generated by `./boom'.
    Program terminated with signal 11, Segmentation fault.
    #0  0x0000000000400770 in main () at /home/dan/work/learn_core/boom.c:8
    8           *p = 7;
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) 
    
如果没有修改core dump文件的位置，该文件就会在程序的当前位置产生，就可以直接启动如:

方法1

    $ gdb boom core.boom.5941.1557306910 -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    [New LWP 5941]
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".
    Core was generated by `./boom'.
    Program terminated with signal 11, Segmentation fault.
    #0  0x0000000000400770 in main () at /home/dan/work/learn_core/boom.c:8
    8           *p = 7;
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) 
    
方法2

    $ gdb -q
    (gdb) file boom
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb) core  core.boom.5941.1557306910
    [New LWP 5941]
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".
    Core was generated by `./boom'.
    Program terminated with signal 11, Segmentation fault.
    #0  0x0000000000400770 in main () at /home/dan/work/learn_core/boom.c:8
    8           *p = 7;
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) 
    
    
## help命令

简写为h，查询命令帮助手册，例如:

    $ gdb -q
    (gdb) help start
    Run the debugged program until the beginning of the main procedure.
    You may specify arguments to give to your program, just as with the
    "run" command.
    (gdb) 


## list命令

简写为l，查看源代码，例如:
```c
#include <stdio.h>
 
void func()
{
    printf("here");
}
      
int main()
{
    int a = 0;
    func();
    a++;
    return 0;
}
```
list num 指定行号

    $ gdb boom -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb) l 7
    
    void func()
    {
        printf("here");
    }


    int main()
    {
        int a = 0;
 
 list function 指定函数名
 
    $ gdb boom -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb) list func
    #include <stdio.h>

    void func()
    {
        printf("here");
    }

    int main()
    {
        int a = 0;
    (gdb) 
    
 list start,end 指定范围
 
    $ gdb boom -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb) list 1,22
    #include <stdio.h>

    void func()
    {
        printf("here");
    }

    int main()
    {
        int a = 0;
        func();
        a++;
        return 0;
    }
    (gdb) 
 
 list + 向后打印
 list - 向前打印
 
## start命令

start命令会给main函数的第一个可执行语句打上临时断点，然后运行程序直到该断点，例如:

```c
#include <stdio.h>
 
void func()
{
    printf("here");
}
      
int main()
{
    int a = 0;
    func();
    a++;
    return 0;
}
```
    
    gdb boom -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb) start
    Temporary breakpoint 1 at 0x40074a: file /home/dan/work/learn_core/boom.c, line 10.
    Starting program: /home/dan/work/learn_core/build/bin/boom 
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".

    Temporary breakpoint 1, main () at /home/dan/work/learn_core/boom.c:10
    10          int a = 0;
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) 
    
## next命令

简写为n，继续运行到下一个代码行，遇到函数则直接运行函数，例如:

```c
#include <stdio.h>
 
void func()
{
    printf("here");
}
      
int main()
{
    int a = 0;
    func();
    a++;
    return 0;
}
```

    $ gdb boom -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb) start
    Temporary breakpoint 1 at 0x40074a: file /home/dan/work/learn_core/boom.c, line 10.
    Starting program: /home/dan/work/learn_core/build/bin/boom 
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".

    Temporary breakpoint 1, main () at /home/dan/work/learn_core/boom.c:10
    10          int a = 0;
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) next
    11          func();
    (gdb) n
    12          a++;
    (gdb) 
    13          return 0;
    (gdb) 
    14      }
    (gdb) 
    0x00007ffff730e3d5 in __libc_start_main () from /lib64/libc.so.6
    (gdb) 
    Single stepping until exit from function __libc_start_main,
    which has no line number information.
    here[Inferior 1 (process 6236) exited normally]
    (gdb) 
    
    
next num 可以指定连续运行的代码行数量，例如:  

    (gdb) start
    Temporary breakpoint 2 at 0x40074a: file /home/dan/work/learn_core/boom.c, line 10.
    Starting program: /home/dan/work/learn_core/build/bin/boom 
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".

    Temporary breakpoint 2, main () at /home/dan/work/learn_core/boom.c:10
    10          int a = 0;
    (gdb) n 3
    13          return 0;
    (gdb) 
    
## step命令

简写为s，继续运行到下一个代码行，遇到函数则进入函数，例如:

```c
#include <stdio.h>
 
void func()
{
    printf("here");
}
      
int main()
{
    int a = 0;
    func();
    a++;
    return 0;
}
```


    $ gdb boom -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb) start
    Temporary breakpoint 1 at 0x40074a: file /home/dan/work/learn_core/boom.c, line 10.
    Starting program: /home/dan/work/learn_core/build/bin/boom 
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".

    Temporary breakpoint 1, main () at /home/dan/work/learn_core/boom.c:10
    10          int a = 0;
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) step
    11          func();
    (gdb) 
    func () at /home/dan/work/learn_core/boom.c:5
    5           printf("here");
    (gdb) 
    6       }
    (gdb) 
    main () at /home/dan/work/learn_core/boom.c:12
    12          a++;
    (gdb) 
    13          return 0;
    (gdb) 
    14      }
    (gdb) 
    0x00007ffff730e3d5 in __libc_start_main () from /lib64/libc.so.6
    (gdb) 
    Single stepping until exit from function __libc_start_main,
    which has no line number information.
    here[Inferior 1 (process 6252) exited normally]
    (gdb) 

## break命令
简写为b，设置断点，程序运行到断点就会暂停挂起，例如:

```c
#include <stdio.h>
 
void func()
{
    printf("here");
}
      
int main()
{
    int a = 0;
    func();
    a++;
    return 0;
}
```

    gdb boom -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb) list 1,20
    1       #include <stdio.h>
    2
    3       void func()
    4       {
    5           printf("here");
    6       }
    7
    8       int main()
    9       {
    10          int a = 0;
    11          func();
    12          a++;
    13          return 0;
    14      }
    (gdb) b 5
    Breakpoint 1 at 0x400731: file /home/dan/work/learn_core/boom.c, line 5.
    (gdb) run
    Starting program: /home/dan/work/learn_core/build/bin/boom 
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".

    Breakpoint 1, func () at /home/dan/work/learn_core/boom.c:5
    5           printf("here");
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) 
    
## 查看断点
info breakpoints 可以查看全部设置的断点，命令缩写成 i b，例如:
    
```c
#include <stdio.h>
 
void func()
{
    printf("here");
}
      
int main()
{
    int a = 0;
    func();
    a++;
    return 0;
}
```

    $ gdb boom -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb) list 1,20
    1       #include <stdio.h>
    2
    3       void func()
    4       {
    5           printf("here");
    6       }
    7
    8       int main()
    9       {
    10          int a = 0;
    11          func();
    12          a++;
    13          return 0;
    14      }
    (gdb) b 5
    Breakpoint 1 at 0x400731: file /home/dan/work/learn_core/boom.c, line 5.
    (gdb) info break
    Num     Type           Disp Enb Address            What
    1       breakpoint     keep y   0x0000000000400731 in func at /home/dan/work/learn_core/boom.c:5
    (gdb) 

## 删除断点

```c
#include <stdio.h>
 
void func()
{
    printf("here");
}
      
int main()
{
    int a = 0;
    func();
    a++;
    return 0;
}
```

    $ gdb boom -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb) list 1,20
    1       #include <stdio.h>
    2
    3       void func()
    4       {
    5           printf("here");
    6       }
    7
    8       int main()
    9       {
    10          int a = 0;
    11          func();
    12          a++;
    13          return 0;
    14      }
    (gdb) b 5
    Breakpoint 1 at 0x400731: file /home/dan/work/learn_core/boom.c, line 5.
    (gdb) info break
    Num     Type           Disp Enb Address            What
    1       breakpoint     keep y   0x0000000000400731 in func at /home/dan/work/learn_core/boom.c:5
    (gdb) delete 1
    (gdb) info break
    No breakpoints or watchpoints.
    (gdb) 

## tbreak命令
设置临时断点，指令简写为ts，该断点一旦触发就会失效例如:

```c
#include <stdio.h>
 
void func()
{
    printf("here");
}
      
int main()
{
    int a = 0;
    func();
    a++;
    return 0;
}
```

    $ gdb boom -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb) tb 5
    Temporary breakpoint 1 at 0x400731: file /home/dan/work/learn_core/boom.c, line 5.
    (gdb) info b
    Num     Type           Disp Enb Address            What
    1       breakpoint     del  y   0x0000000000400731 in func at /home/dan/work/learn_core/boom.c:5
    (gdb) run
    Starting program: /home/dan/work/learn_core/build/bin/boom 
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".

    Temporary breakpoint 1, func () at /home/dan/work/learn_core/boom.c:5
    5           printf("here");
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) info b
    No breakpoints or watchpoints.
    (gdb)
    
## continue命令
遇到断点可以选择next，step等命令继续运行到下一个代码行，也可以使用continue，继续运行整个程序，例如:

```c
#include <stdio.h>
 
void func3()
{
    int d = 4;
    printf("d=%d\n", d);
}

void func2()
{
    int c = 3;
    printf("c=%d\n", c);
    func3();
}
 
void func1()
{
    int b = 2;
    printf("b=%d\n", b);
    func2();
}

int main()
{
    int a  = 1;
    printf("a=%d\n", a);
    func1();
    return 0;
}
```

    $ gdb boom -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb) b 5
    Breakpoint 1 at 0x400775: file /home/dan/work/learn_core/boom.c, line 5.
    (gdb) r
    Starting program: /home/dan/work/learn_core/build/bin/boom 
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".
    a=1
    b=2
    c=3

    Breakpoint 1, func3 () at /home/dan/work/learn_core/boom.c:5
    5           int d = 4;
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) c
    Continuing.
    d=4
    [Inferior 1 (process 4209) exited normally]
    (gdb) 
    
    
    
## backtrace命令
打印函数堆栈帧，回溯整个调用过程，之类简写为bt，例如:

```c
#include <stdio.h>

void func3()
{
    int d = 4;
    (void)(d);
    int* p = NULL;
    *p = 7;
}

void func2()
{
    int c = 3;
    (void)(c);
    func3();
}

void func1()
{
    int b = 2;
    (void)(b);
    func2();
}

int main()
{
    int a  = 1;
    (void)(a);
    func1();
    return 0;
}
               
```
    
    gdb boom core.boom.4042.1557369051 -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    [New LWP 4042]
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".
    Core was generated by `./boom'.
    Program terminated with signal 11, Segmentation fault.
    #0  0x0000000000400744 in func3 () at /home/dan/work/learn_core/boom.c:8
    8           *p = 7;
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) bt
    #0  0x0000000000400744 in func3 () at /home/dan/work/learn_core/boom.c:8
    #1  0x0000000000400765 in func2 () at /home/dan/work/learn_core/boom.c:15
    #2  0x0000000000400780 in func1 () at /home/dan/work/learn_core/boom.c:22
    #3  0x000000000040079b in main () at /home/dan/work/learn_core/boom.c:29
    
bt full 可以打印每个函数的局部变量

    (gdb) bt full
    #0  0x0000000000400744 in func3 () at /home/dan/work/learn_core/boom.c:8
            d = 4
            p = 0x0
    #1  0x0000000000400765 in func2 () at /home/dan/work/learn_core/boom.c:15
            c = 3
    #2  0x0000000000400780 in func1 () at /home/dan/work/learn_core/boom.c:22
            b = 2
    #3  0x000000000040079b in main () at /home/dan/work/learn_core/boom.c:29
            a = 1
    (gdb) 
    
## 查看当前所处的函数堆栈帧
通过 info frame 可以查看当前所处的函数堆栈帧信息，例如:

```c
#include <stdio.h>

void func3()
{
    int d = 4;
    (void)(d);
    int* p = NULL;
    *p = 7;
}

void func2()
{
    int c = 3;
    (void)(c);
    func3();
}

void func1()
{
    int b = 2;
    (void)(b);
    func2();
}

int main()
{
    int a  = 1;
    (void)(a);
    func1();
    return 0;
}
               
```

    $ gdb boom core.boom.4042.1557369051 -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    [New LWP 4042]
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".
    Core was generated by `./boom'.
    Program terminated with signal 11, Segmentation fault.
    #0  0x0000000000400744 in func3 () at /home/dan/work/learn_core/boom.c:8
    8           *p = 7;
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) i frame
    Stack level 0, frame at 0x7ffddb0d63d0:
    rip = 0x400744 in func3 (/home/dan/work/learn_core/boom.c:8); saved rip 0x400765
    called by frame at 0x7ffddb0d63f0
    source language c.
    Arglist at 0x7ffddb0d63c0, args: 
    Locals at 0x7ffddb0d63c0, Previous frame's sp is 0x7ffddb0d63d0
    Saved registers:
    rbp at 0x7ffddb0d63c0, rip at 0x7ffddb0d63c8
    (gdb) 
    
## 选择函数堆栈帧

通过frame 可以选择指定的函数堆栈帧，该指令缩写为f，例如:

```c
#include <stdio.h>

void func3()
{
    int d = 4;
    (void)(d);
    int* p = NULL;
    *p = 7;
}

void func2()
{
    int c = 3;
    (void)(c);
    func3();
}

void func1()
{
    int b = 2;
    (void)(b);
    func2();
}

int main()
{
    int a  = 1;
    (void)(a);
    func1();
    return 0;
}
               
```

    $ gdb boom core.boom.4042.1557369051 -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    [New LWP 4042]
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".
    Core was generated by `./boom'.
    Program terminated with signal 11, Segmentation fault.
    #0  0x0000000000400744 in func3 () at /home/dan/work/learn_core/boom.c:8
    8           *p = 7;
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) bt
    #0  0x0000000000400744 in func3 () at /home/dan/work/learn_core/boom.c:8
    #1  0x0000000000400765 in func2 () at /home/dan/work/learn_core/boom.c:15
    #2  0x0000000000400780 in func1 () at /home/dan/work/learn_core/boom.c:22
    #3  0x000000000040079b in main () at /home/dan/work/learn_core/boom.c:29
    (gdb) info frame
    Stack level 0, frame at 0x7ffddb0d63d0:
    rip = 0x400744 in func3 (/home/dan/work/learn_core/boom.c:8); saved rip 0x400765
    called by frame at 0x7ffddb0d63f0
    source language c.
    Arglist at 0x7ffddb0d63c0, args: 
    Locals at 0x7ffddb0d63c0, Previous frame's sp is 0x7ffddb0d63d0
    Saved registers:
    rbp at 0x7ffddb0d63c0, rip at 0x7ffddb0d63c8
    (gdb) frame 2
    #2  0x0000000000400780 in func1 () at /home/dan/work/learn_core/boom.c:22
    22          func2();
    (gdb) info frame
    Stack level 2, frame at 0x7ffddb0d6410:
    rip = 0x400780 in func1 (/home/dan/work/learn_core/boom.c:22); saved rip 0x40079b
    called by frame at 0x7ffddb0d6430, caller of frame at 0x7ffddb0d63f0
    source language c.
    Arglist at 0x7ffddb0d6400, args: 
    Locals at 0x7ffddb0d6400, Previous frame's sp is 0x7ffddb0d6410
    Saved registers:
     rbp at 0x7ffddb0d6400, rip at 0x7ffddb0d6408
    (gdb) 
    
## 打印函数局部变量
通过info locals 可以打印当前函数堆栈帧内的局部变量，例如:

```c
#include <stdio.h>

void func3()
{
    int d = 4;
    (void)(d);
    int* p = NULL;
    *p = 7;
}

void func2()
{
    int c = 3;
    (void)(c);
    func3();
}

void func1()
{
    int b = 2;
    (void)(b);
    func2();
}

int main()
{
    int a  = 1;
    (void)(a);
    func1();
    return 0;
}
               
```


    $ gdb boom core.boom.4042.1557369051 -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    [New LWP 4042]
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".
    Core was generated by `./boom'.
    Program terminated with signal 11, Segmentation fault.
    #0  0x0000000000400744 in func3 () at /home/dan/work/learn_core/boom.c:8
    8           *p = 7;
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) i frame
    Stack level 0, frame at 0x7ffddb0d63d0:
    rip = 0x400744 in func3 (/home/dan/work/learn_core/boom.c:8); saved rip 0x400765
    called by frame at 0x7ffddb0d63f0
    source language c.
    Arglist at 0x7ffddb0d63c0, args: 
    Locals at 0x7ffddb0d63c0, Previous frame's sp is 0x7ffddb0d63d0
    Saved registers:
      rbp at 0x7ffddb0d63c0, rip at 0x7ffddb0d63c8
    (gdb) info locals
    d = 4
    p = 0x0
    (gdb) frame 1
    #1  0x0000000000400765 in func2 () at /home/dan/work/learn_core/boom.c:15
    15          func3();
    (gdb) info locals
    c = 3
    (gdb) 


## run命令
简写为 r，直接运行程序直到发生错误或者遇到断点，和start不同的是，不会在第一个可执行点暂停，例如:
```c
#include <stdio.h>

void func3()
{
    int d = 4;
    printf("d=%d\n", d);
}

void func2()
{
    int c = 3;
    printf("c=%d\n", c);
    func3();
}

void func1()
{
    int b = 2;
    printf("b=%d\n", b);
    func2();
}

int main()
{
    int a  = 1;
    printf("a=%d\n", a);
    func1();
    return 0;
}
```

    $ gdb boom -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb) r
    Starting program: /home/dan/work/learn_core/build/bin/boom 
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".
    a=1
    b=2
    c=3
    d=4
    [Inferior 1 (process 4201) exited normally]
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) b 5
    Breakpoint 1 at 0x400775: file /home/dan/work/learn_core/boom.c, line 5.
    (gdb) r
    Starting program: /home/dan/work/learn_core/build/bin/boom 
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".
    a=1
    b=2
    c=3

    Breakpoint 1, func3 () at /home/dan/work/learn_core/boom.c:5
    5           int d = 4;
    (gdb) 

## 修改变量值
利用set 命令可以修改程序变量，利用print 可以打印变量的值，缩写为p，例如:

```c
#include <stdio.h>
#include <unistd.h>

void func1()
{
    int i = 0;

    while(i < 200)
    {
        sleep(1);
    }
}
 
int main()
{
    func1();
    return 0;
}
 
```
    $ gdb -q
    (gdb) file boom
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb) b 8
    Breakpoint 1 at 0x40073c: file /home/dan/work/learn_core/boom.c, line 8.
    (gdb) r
    Starting program: /home/dan/work/learn_core/build/bin/boom 
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".

    Breakpoint 1, func1 () at /home/dan/work/learn_core/boom.c:8
    8           while(i < 200)
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) p i
    $1 = 0
    (gdb) set var i = 200
    (gdb) p i
    $2 = 200
    (gdb) s
    12      }
    (gdb) 
    main () at /home/dan/work/learn_core/boom.c:17
    17          return 0;
    (gdb) 
    18      }
    (gdb) 
    0x00007ffff730e3d5 in __libc_start_main () from /lib64/libc.so.6
    (gdb) 
    Single stepping until exit from function __libc_start_main,
    which has no line number information.
    [Inferior 1 (process 4361) exited normally]
    (gdb) 

要用print打印数组，如果数组的元素数量大于200，是没办法显示完全的，但是可以设置最大元素数量，如:

    set print elements 0    //不进行限制
    
    
## 查看变量类型

```c
#include <stdio.h>
#include <unistd.h>
 

struct User {
    char openid[15];
    int  age;
};


void func1()
{
    struct User u = {"wx1234567", 7};
    printf("openid = %s\n", u.openid);
    int i = 0;

    while(i < 200)
    {
        sleep(1);
    }
}

int main()
{
    func1();
    return 0;
}
```


    $ gdb boom -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb) b 17
    Breakpoint 1 at 0x4007b8: file /home/dan/work/learn_core/boom.c, line 17.
    (gdb) r
    Starting program: /home/dan/work/learn_core/build/bin/boom 
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".
    openid = wx1234567

    Breakpoint 1, func1 () at /home/dan/work/learn_core/boom.c:17
    warning: Source file is more recent than executable.
    17          while(i < 200)
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) ptype u
    type = struct User {
        char openid[15];
        int age;
    }
    (gdb) 


## 查看线程运行情况

```c
#include <stdio.h>
#include <unistd.h>
#include <pthread.h>


void* func(void *p_arg)
{
    while(1)
    {
        sleep(3);
    }
}

int main()
{
    pthread_t t1;
    pthread_t t2;
    char t1n[] = "t1";
    char t2n[] = "t2";

    pthread_create(&t1, NULL, func, t1n);
    pthread_create(&t2, NULL, func, t2n);
    sleep(100);
    return 0;
}
```

info threads 可以查看全部的线程运行情况，包括线程的id和系统id以及当前栈， info thread ID 可以查看单独的线程的运行情况。

    $ gdb boom -q
    Reading symbols from /home/dan/work/learn_core/build/bin/boom...done.
    (gdb) b 23
    Breakpoint 1 at 0x40080b: file /home/dan/work/learn_core/boom.c, line 23.
    (gdb) r
    Starting program: /home/dan/work/learn_core/build/bin/boom 
    [Thread debugging using libthread_db enabled]
    Using host libthread_db library "/lib64/libthread_db.so.1".
    [New Thread 0x7ffff72eb700 (LWP 4675)]
    [New Thread 0x7ffff6aea700 (LWP 4676)]

    Breakpoint 1, main () at /home/dan/work/learn_core/boom.c:23
    23          sleep(100);
    Missing separate debuginfos, use: debuginfo-install glibc-2.17-260.el7.x86_64
    (gdb) i threads
    Id   Target Id         Frame 
    3    Thread 0x7ffff6aea700 (LWP 4676) "boom" 0x00007ffff73b0e2d in nanosleep () from /lib64/libc.so.6
    2    Thread 0x7ffff72eb700 (LWP 4675) "boom" 0x00007ffff73b0e2d in nanosleep () from /lib64/libc.so.6
    * 1    Thread 0x7ffff7fee740 (LWP 4671) "boom" main () at /home/dan/work/learn_core/boom.c:23
    (gdb) 
    
thread apply ID bt 命令可以打印 指定线程的调用栈

    (gdb) thread apply 3 bt

    Thread 3 (Thread 0x7ffff6aea700 (LWP 4676)):
    #0  0x00007ffff73b0e2d in nanosleep () from /lib64/libc.so.6
    #1  0x00007ffff73b0cc4 in sleep () from /lib64/libc.so.6
    #2  0x00000000004007b3 in func (p_arg=0x7fffffffe3b0) at /home/dan/work/learn_core/boom.c:10
    #3  0x00007ffff7bc6dd5 in start_thread () from /lib64/libpthread.so.0
    #4  0x00007ffff73e9ead in clone () from /lib64/libc.so.6

thread apply all bt 命令可以打印 全部线程的调用栈

    (gdb) thread apply all bt

    Thread 3 (Thread 0x7ffff6aea700 (LWP 4676)):
    #0  0x00007ffff73b0e2d in nanosleep () from /lib64/libc.so.6
    #1  0x00007ffff73b0cc4 in sleep () from /lib64/libc.so.6
    #2  0x00000000004007b3 in func (p_arg=0x7fffffffe3b0) at /home/dan/work/learn_core/boom.c:10
    #3  0x00007ffff7bc6dd5 in start_thread () from /lib64/libpthread.so.0
    #4  0x00007ffff73e9ead in clone () from /lib64/libc.so.6

    Thread 2 (Thread 0x7ffff72eb700 (LWP 4675)):
    #0  0x00007ffff73b0e2d in nanosleep () from /lib64/libc.so.6
    #1  0x00007ffff73b0cc4 in sleep () from /lib64/libc.so.6
    #2  0x00000000004007b3 in func (p_arg=0x7fffffffe3c0) at /home/dan/work/learn_core/boom.c:10
    #3  0x00007ffff7bc6dd5 in start_thread () from /lib64/libpthread.so.0
    #4  0x00007ffff73e9ead in clone () from /lib64/libc.so.6

    Thread 1 (Thread 0x7ffff7fee740 (LWP 4671)):
    #0  main () at /home/dan/work/learn_core/boom.c:23
    (gdb)  
 
    
具体参考[gdb手册](https://sourceware.org/gdb/onlinedocs/gdb/Threads.html)

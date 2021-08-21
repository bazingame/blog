---
title: Linux日志删除与恢复
date: 2018-01-17 16:00:00
tags: [Linux]
---
### Linux 日志删除
 > Linux的系统日志一般位于/var/log目录内
 > 以下为常见系统日志：

-- 
   <!--more-->
 > lastlog ：纪录最近几次成功登录的事件和最后一次不成功的登录  
utmp ：纪录当前登录的每个用户 
wtmp： 一个用户每次登录进入和退出时间的永久纪录
messages： 从syslog中记录信息（有的链接到syslog文件）  
sudolog：纪录使用sudo发出的命令 
sulog：纪录使用su命令的使用 
syslog：从syslog中记录信息（通常链接到messages文件）
acct 或 pacct：记录每个用户使用的命令记录
history日志：这个文件保存了用户最近输入命令的记录

--
>日志系统使用logrotate来进行自动清除以防止日志文件过大  在/etc/logrotate.conf logrotate.d 中
> rotate 为转存次数

#### 直接删除使用rm

删除一定天数前的日志文件。输入命令：
> find  /var/log  -mtime  +3  -name  "*.log"  -exec  rm  -rf  {}  ;
该命令将/var/log/目录下所有3天前带“.log”的文件删除。

**Tips:**
find 命令的使用:

-mtime -n +n
> 按照文件的更改时间来查找文件，-n表示n天以内，+n表示n天以前

" \*.log"
> 匹配希望查找的数据类型

-exec
> find命令对匹配的文件执行该参数所给出的shell命令。相应命令的形式为'command' { } ;

#### 脚本删除

cat /dev/null > /var/log/lastlog
> cat /dev/null 可以看作一个"黑洞".  所有写入它的内容都会永远丢失. 而尝试从它那儿读取内容则什么也读不到. 

因此可以使用此命令构建删除脚本: **clear_log.sh**

```powershell
#!/bin/sh
cat /dev/null > /var/log/lastlog
...
```

#### 清除history

history -c
> 简单清除history 很容易被发现，因此需要按需删除记录在**bash_history**中的内容

#### 应用日志同上

### Linux 日志恢复
> **原理：** 当进程打开了某个文件时，只要该进程保持打开该文件，即使将其删除，它依然存在于磁盘中。这意味着，进程并不知道文件已经被删除，它仍然可以向打开该文件时提供给它的文件描述符进行读取和写入。除了该进程之外，这个文件是不可见的，因为已经删除了其相应的目录索引节点。

-

> 要将日志恢复，首先要确保日志进程未停止运行，利用lsof命令找到该进程，利用进程标识符和文件描述符，使用cat命令将内容重写到日志文件，然后重启日志记录服务。

#### **lsof**的使用
>List Open Files 一个非常实用的系统级的监控、诊断工具
> 是有着最多开关的Linux/Unix命令之一

lsof直接输入
> 列出活跃进程的所有打开文件
```
COMMAND    PID  TID             USER   FD      TYPE             DEVICE  SIZE/OFF       NODE NAME
systemd      1                  root  cwd       DIR               8,20      4096          2 /
systemd      1                  root  rtd       DIR               8,20      4096          2 /
systemd      1                  root  txt       REG               8,20   1690360     925307 /lib/systemd/systemd
systemd      1                  root  mem       REG               8,20   1354616    1088074 /lib/x86_64-linux-gnu/libm-2.26.so
systemd      1                  root  mem       REG               8,20    121016    1050521 /lib/x86_64-linux-gnu/libudev.so.1.6.8
systemd      1                  root  mem       REG               8,20     84032    1050539 /lib/x86_64-linux-gnu/libgpg-error.so.0.22.0
systemd      1                  root  mem       REG               8,20     18832    1050740 /lib/x86_64-linux-gnu/libattr.so.1.1.0

```
> 信息如下
> COMMAND | 进程的名称
> PID | 进程标识符
> USER | 进程所有者
> FD | 文件描述符，应用程序通过文件描述符识别该文件。如cwd、txt等
> TYPE | 文件类型，如DIR、REG等
> DEVICE | 指定磁盘的名称
> SIZE | 文件的大小
> NODE | 索引节点（文件在磁盘上的标识）
> NAME | 打开文件的确切名称

-i
> 显示所有网络连接

filename
> 显示开启filename文件的进程

-c processname
> 显示该进程打开的文件

-c -p pid
> 显示该进程号对应的进程打开的文件

-d dir
> 显示该目录下被打开的文件

-D dir
> 同上 迭代显示目录下所有文件夹

#### **wc**命令的使用
> wc的功能为统计指定文件中的字节数、字数、行数，并将统计结果显示输出

-c
> 统计字节数

-l
>统计行数

-m
> 统计字符数，不能与 -c 标志一起使用

-w 
> 统计字数，一个字被定义为由空白、跳格或换行字符分隔的字符串

--version
> 显示版本信息

---
**Tips：**
查看文件大小的几种方式

stat filepath
> 时间、大小均显示，较详细

wc -c filename
> 只适用于文件，显示结果为字节数，无单位

du -b/-h -filepath
> b 表示字节数 无单位
> h 表示更友好的形式  有单位

ls -lh filepath
> h代表human 对人更友好 

---

Linux 内核中提供了一种通过 /proc 文件系统，在运行时访问内核内部数据结构、改变内核设置的机制。proc文件系统是一个伪文件系统，它只存在内存当中，而不占用外存空间。它以文件系统的方式为访问系统内核数据的操作提供接口。

/proc/N/fd 
> N为进程Pid ，该文件夹中包含进程相关的所有的文件描述符

/proc/N/status
>进程的状态


#### 查看日志情况
> - 使用lsof命令查看目前打开丢失日志的进程
	如 lsof | grep /var/log/syslog ，便可得到相应信息 COMMAND、PID、FD
> - 使用wc命令查看日志情况
	如 wc -l /proc/1/fd/1 可查看在内存中的日志记录

#### 重写日志
> cat /proc/1/fd/1 > /var/log/syslog

#### 重启服务

对于许多应用程序，尤其是日志文件和数据库，这种恢复删除文件的方法非常有用。


# unlink函数说明

## 原型：
#include <unistd.h>;
int unlink(const char \*pathname);

## 返回值：
0：成功   -1: 失败，并把错误码写到errno

## 常见errno：
ENOENT：路径不存在
ESIDIR：路径指向目录（旧内核不允许 unlink 目录；目录请用 rmdir）
EPERM / EACCES 权限不足或文件被标记为不可变（chattr +i）
EBUSY 试图卸载当前工作目录或挂载点

## 基本用法示例：
#include <unistd.h>
#include <stdio.h>
int main(void ){
    if (unlink("test.txt") == -1) {
    perror("unlink");
    return 1;
}

    puts("file unlinked");
    return 0;
}

编译运行后，test.txt 的目录项消失；若此时没有别的硬链接、也没有进程占用，则文件块真正回收。
unlink(path) 就是“从文件系统目录树里摘掉一个名字”，摘完以后，如果该名字是文件的最后一个硬链接，且没有被任何进程打开，文件才真正消失。


# socket函数说明
1 函数原型
#include <sys/types.h>
#include <sys/socket.h>
int socket(int domain, int type, int protocol);

## 2 参数总览
domain   协议族，决定地址格式与协议范围
type     套接字类型，决定数据传输语义
protocol 具体协议，一般填 0 让内核自动选

## 3 常用 domain 值
AF_INET      IPv4
AF_INET6     IPv6
AF_UNIX      本地 IPC（Unix 域）
AF_NETLINK   内核与用户态通信
AF_PACKET    裸链路层包

## 4 常用 type 值（可位或 SOCK_NONBLOCK、SOCK_CLOEXEC）
SOCK_STREAM    可靠面向连接字节流 → TCP
SOCK_DGRAM     不可靠无连接固定长度 → UDP
SOCK_RAW       直接访问 IP 层（需 CAP_NET_RAW）
SOCK_SEQPACKET 面向连接但保留消息边界（Unix 域常用）

## 5 protocol 取值
一般填 0：
AF_INET + SOCK_STREAM ⇒ IPPROTO_TCP
AF_INET + SOCK_DGRAM  ⇒ IPPROTO_UDP
若创建原始套接字需显式指定：IPPROTO_TCP、IPPROTO_UDP、IPPROTO_ICMP 等。

## 6 返回值
≥ 0  成功，返回套接字描述符
-1   失败，errno 被设置

## 7 常见 errno
EACCES  无权限（如 SOCK_RAW 且无 CAP_NET_RAW）
EMFILE  进程已打开描述符达上限
ENFILE  系统级文件表满
ENOBUFS/ENOMEM 内存不足
EPROTONOSUPPORT 内核不支持请求的协议组合

## 8 最简示例
TCP 套接字：
int fd = socket(AF_INET, SOCK_STREAM, 0);
if (fd == -1) { perror("socket"); exit(EXIT_FAILURE); }
UDP 套接字：
int fd = socket(AF_INET, SOCK_DGRAM, 0);
非阻塞 TCP + close-on-exec：
int fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, 0);

## 9 高级组合速查
IPv6 TCP           socket(AF_INET6, SOCK_STREAM, 0)
Unix 域字节流       socket(AF_UNIX, SOCK_STREAM, 0)
原始 ICMP            socket(AF_INET, SOCK_RAW, IPPROTO_ICMP)
Netlink 路由消息     socket(AF_NETLINK, SOCK_RAW, NETLINK_ROUTE)

## 10 关闭套接字
close(fd);                 立即释放 fd，触发 TCP 正常四次挥手
shutdown(fd, how);         半关闭（读、写或读写）

## SOCK_NONBLOCK
作用：
在新建套接字的同时把它设为“非阻塞”模式。

### 效果：
后续 read/recv/send/accept/connect 若暂时无法完成，立即返回 -1 并把 errno 设为 EAGAIN/EWOULDBLOCK，而不是把进程挂起。
不必再调用 fcntl(fd, F_SETFL, O_NONBLOCK)。

## SOCK_CLOEXEC
作用：
在新建套接字的同时给它设置 FD_CLOEXEC 标志。

### 效果：
进程后面调用 exec 家族（如 execl、execvp）时，该描述符会自动关闭，避免泄漏到新的可执行文件里。
不必再调用 fcntl(fd, F_SETFD, FD_CLOEXEC)。

### 用法：
int fd = socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, 0);

### 注意：
这两个都是 Linux 2.6.27 以后引入的 GNU 扩展，必须在代码开头定义 _GNU_SOURCE，并且只在 socket() 创建时一次性指定，不能事后“或”上去。


# fcntl

## 原型：
#include <unistd.h>
#include <fcntl.h>
int fcntl(int fd, int cmd, ... /* arg */);

## 功能
：对已打开的文件描述符进行各种“细粒度”控制，是通用“瑞士军刀”。
常用 cmd 与第三参数：

F_DUPFD / F_DUPFD_CLOEXEC
复制描述符，返回新 fd。
F_DUPFD_CLOEXEC 同时带 close-on-exec 标志。
例：int newfd = fcntl(oldfd, F_DUPFD, 0);

F_GETFD / F_SETFD
获取或设置“描述符标志”，目前只用得到 FD_CLOEXEC。
例：
int flags = fcntl(fd, F_GETFD);
fcntl(fd, F_SETFD, flags | FD_CLOEXEC);

F_GETFL / F_SETFL
获取或设置“文件状态标志”——O_RDONLY/O_RDWR/O_APPEND/O_NONBLOCK/O_SYNC...
例：设非阻塞
int fl = fcntl(fd, F_GETFL);
fcntl(fd, F_SETFL, fl | O_NONBLOCK);

F_GETLK / F_SETLK / F_SETLKW
咨询式文件锁（记录锁）。第三参数是 struct flock *。
F_SETLK  非阻塞加锁，失败立即返回 -1 并设 EAGAIN。
F_SETLKW 阻塞版本，等待到成功。

F_SETPIPE_SZ / F_GETPIPE_SZ (Linux 特有)
调整管道缓冲区大小。

## 返回值：
成功：依赖 cmd——
F_DUPFD 系列返回新 fd；
F_GETFD/F_GETFL/F_GETLK 返回标志或 0；
F_GETPIPE_SZ 返回当前大小。
失败：-1，errno 给出原因（EBADF/EINVAL/EPERM...）。

## 典型用途速记：
把套接字/文件设非阻塞：F_GETFL → 或 O_NONBLOCK → F_SETFL
加 close-on-exec：F_GETFD → 或 FD_CLOEXEC → F_SETFD
复制且自动 CLOEXEC：F_DUPFD_CLOEXEC
文件段加锁：F_SETLK/F_SETLKW
一句话：fcntl 就是“对已打开 fd 的各种小旋钮做调整”的唯一标准接口。
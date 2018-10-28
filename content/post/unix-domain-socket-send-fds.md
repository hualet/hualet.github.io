---
title: "通过 Unix Domain Socket 传递文件描述符"
date: 2018-10-28T21:08:45+08:00
---

Unix Domain Socket 是 Unix 系统下重要的本地进程间通信（IPC）机制之一，在 DDE、GNOME、KDE 等 Linux 桌面环境中常见的进程间通信方式 DBus 有一种实现方式就是基于 Unix Domain Socket 做的。虽然一直知道它的大名，也一直知道 Unix Domain Socket 可以用来传递文件描述符，但是碍于以前经验和眼界不足，加上没有深入去了解，完全不知道能传递文件描述符是多么强大的能力和必要性。

首先，我想着文件描述符不就是一个数字么，哪个 IPC 不能传递数字呢？完全没有思考到文件描述符是只在进程范围内有效，同样一个文件描述符放在不同的进程完全就不是一回事儿。这时候你肯定想，既然传递文件描述符这么麻烦，为啥非要传递文件描述符呢，使用文件名不也是一样的么？那么恭喜你，你也有更我一样在践行陶渊明老前辈看事物“不求甚解”的作风。传递文件描述符还是有它的必要性的，一方面，文件描述符代表的不只是一个文件，它还包含了文件打开的状态，比如 seek 的位置等，有点进程之于可执行程序的意思，拿到文件描述符也就拿到了这些动态的信息；另一方面，文件描述符不只对应于本地文件，它为了一众可读写对象提供了统一的读写接口，包含什么呢？本地文件、（匿名）管道、标准输入输出、甚至于 Socket 本身等……可以让你完全不关心文件描述符背后的实现是什么而方便实现自己的逻辑代码。

想通了以上道理，又有了以前似曾相识的感慨“古人诚不欺我”——前人留下的东西，必然有一定的合理性。这也是为什么我比较排斥看见一个软件不满意就立即重新造，尤其是能流传很广或者流程时间很长的软件，里面很多看起来不必要的东西，可能有其存在的合理性，最好的做法是尝试去改进，改进的过程了解其历史、学习其精髓，等自己胸有成竹的时候再下手重造不迟。额，跑题了……

回到正题，之所以前段时间突然研究了一下 Socket 传递文件描述符的东西，是遇到这样一个需求：一个进程将自己的标准输入、标准输出和标准错误输出映射到另外一个进程相应的位置。带着对 Unix Domain Socket 的朦胧认识，写了一个简单的实现原型：

```c
// outlet.c

#include <sys/socket.h>
#include <sys/un.h>
#include <string.h>
#include <stdio.h>
#include <errno.h>
#include <unistd.h>

#define SOCKET_NAME "/tmp/connection-uds-fd"

int make_connection()
{
    int fd;
    int connection_fd;
    int result;
    struct sockaddr_un sun;
    
    fd = socket(PF_UNIX, SOCK_STREAM, 0);

    sun.sun_family = AF_UNIX;
    strncpy(sun.sun_path, SOCKET_NAME, sizeof(sun.sun_path) - 1);

    result = bind(fd, (struct sockaddr*) &sun, sizeof(sun));
    listen(fd, 10);
    connection_fd = accept(fd, NULL, NULL);    

    return connection_fd;
}

void send_fds(int socket_fd)
{
    struct msghdr msg;
    struct cmsghdr *cmsg = NULL;
    int io[3] = { 0, 1, 2 };
    char buf[CMSG_SPACE(sizeof(io))];
    struct iovec iov;
    int dummy;

    memset(&msg, 0, sizeof(struct msghdr));

    iov.iov_base = &dummy;
    iov.iov_len = 1;

    msg.msg_iov = &iov;
    msg.msg_iovlen = 1;
    msg.msg_control = buf;
    msg.msg_controllen = sizeof(buf);

    cmsg = CMSG_FIRSTHDR(&msg);
    cmsg->cmsg_len = CMSG_LEN(sizeof(io));
    cmsg->cmsg_level = SOL_SOCKET;
    cmsg->cmsg_type = SCM_RIGHTS;

    memcpy(CMSG_DATA(cmsg), io, sizeof(io));

    msg.msg_controllen = cmsg->cmsg_len;

    int size;
    size = sendmsg(socket_fd, &msg, 0);
    // printf("%d %d\n", size, errno);
}

int main(int argc, char const *argv[])
{
    int connection_fd;

    printf("write from outlet\n");
    connection_fd = make_connection();
    send_fds(connection_fd);

    while(1) {
        sleep(1);
    }

    return 0;
}
```

```c
// inner.c

#include <sys/socket.h>
#include <sys/un.h>
#include <string.h>
#include <stdio.h>
#include <errno.h>
#include <unistd.h>

#define SOCKET_NAME "/tmp/connection-uds-fd"

int connect_socket()
{
    int fd;
    int result;
    struct sockaddr_un sun;

    fd = socket(PF_UNIX, SOCK_STREAM, 0);

    sun.sun_family = AF_UNIX;
    strncpy(sun.sun_path, SOCKET_NAME, sizeof(sun.sun_path) - 1);

    connect(fd, (struct sockaddr *)&sun, sizeof(sun));
    
    return fd;
}

void receive_fds(int socket_fd)
{
    int dummy = 0;
    int fds[3];

    struct iovec iov;
    iov.iov_base = &dummy;
    iov.iov_len = 1;

    char buf[CMSG_SPACE(sizeof(fds))];
    
    struct msghdr msg;
    memset(&msg, 0, sizeof(msg));
    msg.msg_iov        = &iov;
    msg.msg_iovlen     = 1;
    msg.msg_control    = buf;
    msg.msg_controllen = sizeof(buf);

    struct cmsghdr *cmsg;
    cmsg             = CMSG_FIRSTHDR(&msg);
    cmsg->cmsg_len   = CMSG_LEN(sizeof(fds));
    cmsg->cmsg_level = SOL_SOCKET;
    cmsg->cmsg_type  = SCM_RIGHTS;

    memcpy(CMSG_DATA(cmsg), fds, sizeof(fds));

    int size;
    size = recvmsg(socket_fd, &msg, 0);
    // printf("%d \n", size);

    memcpy(fds, CMSG_DATA(cmsg), sizeof(fds));
    printf("%d %d %d", fds[0], fds[1], fds[2]);

    char msg_buf[128];
    strcpy(msg_buf, "write from inner\n");
    write(fds[0], msg_buf, strlen(msg_buf));
}

int main(int argc, char const *argv[])
{
    int socket_fd;
    int fds[3];

    socket_fd = connect_socket();
    receive_fds(socket_fd);

    return 0;
}

```

其中，outlet 是“贡献”标准输入输出的进程，inner 是“抢占” outlet 标准输入输出的进程。整体上代码还比较好理解，只是如果需要发文文件描述符，需要注意 `send_fds` 函数中 `cmsg->cmsg_type = SCM_RIGHTS;` 一行，必须制定 `cmsg_type` 为 `SCM_RIGHTS`。

分别编译两个程序文件：

```bash
$ gcc -o outlet outlet.c
$ gcc -o inner inner.c
```

先执行 outlet，先输出 “write from outlet” 后阻塞，等待 inner 进程的连接：

```bash
$ ./outlet 
$ write from outlet
```

然后再启动 inner 进程，inner 进程输出“4 5 6”，并在 outlet 的标准输出输出 “write from inner“：

```bash
$ ./inner
$ 4 5 6
$ 
```

```bash
$ ./outlet
$ write from outlet
$ write from inner
```

这样就大功告成了。

inner 之所以输出 ”4 5 6“ 是比较好奇传递过来的文件描述符到底是长啥样的，这行打印看代码可以知道是打印了接收到的文件描述符的值，也就是创建了几个新的文件描述符么——就像侦探推理一样，没有揭开谜底前各种好奇，一旦揭开谜底了反倒趣味全无了。



>  P.S.  [unix(7)](http://man7.org/linux/man-pages/man7/unix.7.html) 中其实交代了传递文件描述符的效果跟 dup2 差不多的，还怪自己平常没有好好 RTFM 啊：
>
> ```
> SCM_RIGHTS
>               Send or receive a set of open file descriptors from another
>               process.  The data portion contains an integer array of the
>               file descriptors.  The passed file descriptors behave as
>               though they have been created with dup(2).
> ```
## 对端无 FIN 包发送
### 网络中断造成
在这种情况下，TCP 程序并不能及时感知到异常信息。除非网络中的其他设备，如路由器发出一条 ICMP 报文，说明目的网络或主机不可达，这个时 候通过 read 或 write 调用就会返回 Unreachable 的错误。

在没有 ICMP 报文的情况下，TCP 程序并不能理解感应到连接异常。如果程序是阻塞在 read 调用上，那么程序无法从异常中恢复。我们可以通过给 read 操作设置超时来解决

如果程序先调用了 write 操作发送了一段数据流，接下来阻塞在 read 调用上。Linux 系统的 TCP 协议栈会不断尝试将发送缓冲区的数据发送出去，大概在重传 12 次、合计时间约为 9 分钟之后，协议栈会标识该连接异常，这时，阻塞的 read 调用会返回一条 TIMEOUT 的错误信息。如果此时程序还执着地往这条连接写数据，写操作会立即失败，返回一个 SIGPIPE 信号给应用程序

### 系统崩溃造成
当系统突然崩溃，如断电时，网络连接上来不及发出任何东西。这种情况和网络中断造成的结果非常类似，在没有 ICMP 报文的情况下，TCP 程序只能通过 read 和 write 调用得到网络连接异常的信息，超时错误是一个常见的结果

系统在崩溃之后重启，当重传的 TCP 分组到达重启后的系统，由于系统中没有该 TCP 分组对应的连接数据，系统会返回一个 RST 重置分节，TCP 程序通过 read 或 write 调用可以分别对 RST 进行错误处理。如果是阻塞的 read 调用，会立即返回一个错误，错误信息为连接重置(Connection Resest)。如果是阻塞的 write 操作，也会立即失败，应用程序会被返回一个 SIGPIPE 信号


## 对端有 FIN 包发送
对端如果有 FIN 包发出，可能的场景是对端调用了 close 或 shutdown 显式地关闭了连接，也可能是对端应用程序崩溃，操作系统内核代为清理所发出的

阻塞的 read 操作在完成正常接收的数据读取之后，FIN 包会通过返回一个 EOF 来完成通知，此时，read 调用返回值为 0。收到 FIN 包相当于往接收缓冲区里放置了一个 EOF 符号，之前已经在接收缓冲区的有效数据不会受到影响

服务端
```c
#include "lib/common.h"

int main(int argc, char **argv)
{
    int connfd;
    char buf[1024];

    connfd = tcp_server(SERV_PORT);

    for (;;) {
        int n = read(connfd, buf, 1024);

        if (n < 0) {
            error(1, errno, "error read");
        } else if (n == 0) {
            error(1, 0, "client closed \n");
        }

        sleep(5);

        int write_nc = send(connfd, buf, n, 0);
        printf("send bytes: %zu \n", write_nc);

        if (write_nc < 0) {
            error(1, errno, "error write");
        }
    }

    exit(0);
}
```

客户端
```c
#include "lib/common.h"

int main(int argc, char **argv)
{
    if (argc != 2) {
        error(1, 0, "usage: reliable_client01 <IPaddress>");
    }

    int socket_fd = tcp_client(argv[1], SERV_PORT);
    char buf[128];
    int len;
    int rc;

    while (fgets(buf, sizeof(buf), stdin) != NULL) {
        len = strlen(buf);
        rc = send(socket_fd, buf, len, 0);

        if (rc < 0)
            error(1, errno, "write failed");

        sleep(3);

        rc = read(socket_fd, buf, sizeof(buf));

        if (rc < 0)
            error(1, errno, "read failed");
        else if (rc == 0)
            error(1, 0, "peer connection closed\n");
        else
            fputs(buf, stdout);
    }

    exit(0);
}
```

依次启动服务器端和客户端程序，在客户端输入字符之后，迅速结束掉服务器端程序。客户端 read 直接感知 FIN 包，于是正常退出

依次启动服务器端和客户端程序，在客户端输入字符之后，等待一段时间，直到客户端正确显示了服务端的回应字符之后，再杀死服务器程序。客户端再次输入，这时屏幕上打印出 peer connection closed

服务端
```c
#include "lib/common.h"

int main(int argc, char **argv)
{
    int connfd;
    char buf[1024];
    int time = 0;

    connfd = tcp_server(SERV_PORT);

    while (1) {
        int n = read(connfd, buf, 1024);

        if (n < 0) {
            error(1, errno, "error read");
        } else if (n == 0) {
            error(1, 0, "client closed \n");
        }

        time++;
        fprintf(stdout, "1K read for %d \n", time);
        usleep(1000);
    }

    exit(0);
}
```

客户端
```c
#include "lib/common.h"

int main(int argc, char **argv)
{
    if (argc != 2) {
        error(1, 0, "usage: reliable_client02 <IPaddress>");
    }

    int socket_fd = tcp_client(argv[1], SERV_PORT);
    signal(SIGPIPE, SIG_IGN);

    char *msg = "network programming";
    ssize_t n_written;

    int count = 10000000;

    while (count > 0) {
        n_written = send(socket_fd, msg, strlen(msg), 0);
        fprintf(stdout, "send into buffer %ld \n", n_written);

        if (n_written <= 0) {
            error(1, errno, "send error");
            return -1;
        }

        count--;
    }

    return 0;
}
```

在服务端读取数据并处理过程中，突然杀死服务器进程，可以看到客户端很快也会退出，并在屏幕上打印出 Connection reset by peer 的提示

这是因为服务端程序被杀死之后，操作系统内核会做一些清理的事情，为这个套接字发送一个 FIN 包。但是，客户端在收到 FIN 包之后，没有 read 操作，还是会继续往这个套接字写入数据。这是因为根据 TCP 协议，连接是双向的，收到对方的 FIN 包只意味着对方不会再发送任何消息。在一个双方正常关闭的流程中，收到 FIN 包的一端将剩余数据发送给对面，然后关闭套接字。当数据到达服务器端时，操作系统内核发现这是一个指向关闭的套接字，会再次向客户端发送一个 RST 包，对于发送端而言如果此时再执行 write 操作，立即会返回一个 RST 错误信息


## 对端异常防范
```c
struct timeval tv;
tv.tv_sec = 5;
tv.tv_usec = 0;

// 设置了套接字的读操作超时
setsockopt(connfd, SOL_SOCKET, SO_RCVTIMEO, (const char *) &tv, sizeof tv);

while (1) {
    int nBytes = recv(connfd, buffer, sizeof(buffer), 0);

    if (nBytes == -1) {
        if (errno == EAGAIN || errno == EWOULDBLOCK) {
            printf("read timeout\n");
            onClientTimeout(connfd);
        } else {
            error(1, errno, "error read message");
        }
    } else if (nBytes == 0) {
        error(1, 0, "client closed \n");
    }
}
```

使用 select 多路复用技术来对套接字进行 I/O 事件的轮询，如果连接不正常，从当前 read 阻塞中返回并处理
```c
struct timeval tv;
tv.tv_sec = 5;
tv.tv_usec = 0;

FD_ZERO(&allreads);
FD_SET(socket_fd, &allreads);

for (;;) {
    readmask = allreads;
    int rc = select(socket_fd + 1, &readmask, NULL, NULL, &tv);

    if (rc < 0) {
        error(1, errno, "select failed");
    }

    if (rc == 0) {
        printf("read timeout\n");
        onClientTimeout(socket_fd);
    }
}
```

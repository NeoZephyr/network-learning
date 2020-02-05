## 服务端
### 创建套接字
```c
int socket(int domain, int type, int protocol)
```

1. domain 表示套接字的类型，可选 PF_INET、PF_INET6 以及 PF_LOCAL 等值
2. type 可选 SOCK_STREAM、SOCK_DGRAM 和 SOCK_RAW。其中，SOCK_STREAM 表示字节流，对应 TCP; SOCK_DGRAM 表示数据报，对应 UDP; SOCK_RAW 表示原始套接字
3. protocol 原本用来指定通信协议，但基本废弃，一般设置为 0 即可

### 绑定
```c
bind(int fd, sockaddr * addr, socklen_t len)
```
bind 函数后面的第二个参数是通用地址格式 sockaddr * addr。但实际上传入的参数可能是 ipv4、
ipv6 或者本地套接字格式。bind 函数会根据 len 字段决定如何解析

```c
struct sockaddr_in name;
bind(sock, (struct sockaddr *) &name, sizeof (name));
```

地址绑定：
可以把地址设置成本机的 ip 地址，这表示仅仅对目标 ip 是本机 ip 地址的 ip 包进行处理。这样做有一个问题：我们并不清楚应用程序将会被部署到哪台机器上。此时，可以利用通配地址的能力解决这个问题。对于 ipv4 的地址来说，使用 INADDR_ANY 来完成通配地址的设置；对于 ipv6 的地址来说，使用 IN6ADDR_ANY 来完成通配地址的设置

```c
struct sockaddr_in name;
name.sin_addr.s_addr = htonl(INADDR_ANY);
```

端口绑定：
如果把端口设置成 0，表示把端口的选择权交给操作系统内核来处理，操作系统内核会根据一定的算法选择一个空闲的端口，完成套接字的绑定。这在服务器端不常使用，一般来说，服务器端的程序一定要绑定到一个众所周知的端口上

```c
#include <stdio.h>
#include <stdlib.h>
#include <sys/socket.h>
#include <netinet/in.h>

int make_socket(uint16_t port)
{
    int sock;
    struct sockaddr_in name;

    /* 创建字节流类型的IPV4 socket */
    sock = socket(PF_INET, SOCK_STREAM, 0);
    if (sock < 0) {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    /* 绑定到 port 和 ip */
    name.sin_family = AF_INET; /* IPV4 */
    name.sin_port = htons(port);  /* 指定端口 */
    name.sin_addr.s_addr = htonl(INADDR_ANY); /* 通配地址 */

    /* 把 IPV4 地址转换成通用地址格式，同时传递长度 */
    if (bind(sock, (struct sockaddr *) &name, sizeof(name)) < 0) {
        perror("bind");
        exit(EXIT_FAILURE);
    }

    return sock;
}

int main(int argc, char **argv) {
    int sockfd = make_socket(12345);
    exit(0);
}
```

### 监听
```c
int listen (int socketfd, int backlog)
```
1. socketfd 为套接字描述符
2. backlog 表示未完成连接队列的大小。这个参数越大，并发数目理论上也会越大，但是参数过大也会占用过多的系统资源。一些系统，比如 Linux 并不允许对这个参数进行改变

### 应答
```c
int accept(int listensockfd, struct sockaddr *cliaddr, socklen_t *addrlen);
```
函数的返回值有两个部分，第一个部分 cliadd 是通过指针方式获取的客户端的地址，addrlen 表示地址大小；另一个部分是函数的返回值，表示与客户端的连接


## 客户端
### 创建套接字

### 连接
```c
int connect(int sockfd, const struct sockaddr *servaddr, socklen_t addrlen);
```
1. sockfd 是连接套接字，即 socket 函数创建的
2. servaddr 和 addrlen 分别表示套接字地址指针和地址大小。套接字地址必须含有服务器的 ip 地址和端口号

客户在调用函数 connect 前不必非得调用 bind 函数，因为如果需要的话，内核会确定源 ip 地址，并按照一定的算法选择一个临时端口作为源端口

如果是 tcp 套接字，调用 connect 函数将激发 tcp 的三次握手过程，而且仅在连接建立成功或出错时才返回。其中出错返回可能有以下几种情况：
1. 三次握手无法建立，客户端发出的 SYN 包没有任何响应，于是返回 TIMEOUT 错误。比较常见的原因是对应的服务端 ip 写错
2. 客户端收到 RST 回答，这时候客户端会立即返回 CONNECTION REFUSED 错误。比较常见于客户端发送连接请求的端口写错，因为 RST 是 TCP 在发生错误时发送的一种 TCP 分节。产生 RST 的三个条件是：目的地为某端口的 SYN 到达，然而该端口上没有正在监听的服务器；TCP 想取消一个已有连接; TCP 接收到一个根本不存在的连接上的分节
3. 客户发出的 SYN 包在网络上引起了 "destination unreachable"，即目的不可达的错 误。比较常见的原因是客户端和服务器端路由不通


## 三次握手
1. 客户端的协议栈向服务器端发送了 SYN 包，并告诉服务器端当前发送序列号 j，客户端进入 SYNC_SENT 状态
2. 服务器端的协议栈收到这个包之后，和客户端进行 ACK 应答，应答的值为 j+1，表示对 SYN 包 j 的确认，同时服务器也发送一个 SYN 包，告诉客户端当前的发送序列号为 k，服务器端进入 SYNC_RCVD 状态
3. 客户端协议栈收到 ACK 之后，使得应用程序从 connect 调用返回，表示客户端到服务器端的单向连接建立成功，客户端的状态为 ESTABLISHED，同时客户端协议栈也会对服务器端的 SYN 包进行应答，应答数据为 k+1
4. 应答包到达服务器端后，服务器端协议栈使得 accept 阻塞调用返回，这个时候服务器端到客户端的单向连接也建立成功，服务器端也进入 ESTABLISHED 状态


## 数据传输
### 发送
1. 应用程序调用 send 或 write 方法，将数据送到发送缓冲区。如果缓存中没有空间，系统调用就会失败或者阻塞
2. TCP 协议栈创建 Packet 报文，并把报文发送到传输队列中。队列的最大值可以通过 ifocnfig 命令输出的 txqueuelen 来查看
3. TX ring 在网络驱动和网卡之间，也是一个传输请求的队列
4. 网卡工作在物理层，把要发送的报文保存到内部的缓存中，并发送出去

### 接收
1. 报文首先到达网卡，由网卡保存在自己的接收缓存中
2. 然后报文被发送至网络驱动和网卡之间的 RX ring
3. 网络驱动从 RX ring 获取报文，然后把报文发送到上层。网络驱动和上层之间没有缓存，因为网络驱动使用 NAPI 进行数据传输。因此，可以认为上层直接从 RX ring 中读取报文
4. 最后，报文的数据保存在套接字接收缓存中，应用程序从套接字接收缓存中读取数据


## 四次挥手
1. 客户端调用 close，发送 FIN 包，进入 FIN_WAIT_1 状态。服务器端收到后，进入 CLOSE_WAIT 状态，并发送 ACK 应答
2. 客户端收到服务器应答后，进入 FIN_WAIT_2 状态
3. 服务器端主动调用 close，发送 FIN 包，并进入 LAST_ACK 状态
4. 客户端收到 FIN 包后发送 ACK 应答，进入 TIME_WAIT 状态，并发送 ACK 应答
5. 服务端收到应答后进入 CLOSED 状态
6. 客户端在等待 2MSL 之后，进入 CLOSED 状态

Linux 系统里有一个 TCP_TIMEWAIT_LEN 硬编码的字段，值为 60 秒

当套接字被关闭时，TCP 为其所在端发送一个 FIN 包。在大多数情况下，这是由应用进程调用 close 而发生的，一个进程无论是正常退出（exit 或者 main 函数返回），还是非正常退出（收到 SIGKILL 信号关闭），所有该进程打开的描述符都会被系统关闭，这也导致 TCP 描述符对应的连接上发出一个 FIN 包。无论是客户端还是服务器，任何一端都可以发起主动关闭

### TIME_WAIT 的作用
1. 确保最后的 ACK 应答能让被动关闭方接收，从而帮助其正常关闭

如果客户端的 ACK 报文没有传输成功，那么服务端就会重新发送 FIN 报文。如果客户端没有维护 TIME_WAIT 状态，而直接进入 CLOSED 状态，它就失去了当前状态的上下文，只能回复一个 RST 操作，从而导致被动关闭方出现错误

反之，如果客户端知道自己处于 TIME_WAIT 的状态，就可以在接收到 FIN 报文之后，重新发出一个 ACK 报文，使得服务端可以进入正常的 CLOSED 状态

2. 让旧连接的重复分节在网络中自然消失

在网络中，经常会发生报文经过一段时间才能到达目的地的情况，产生的原因是多种多样的，如路由器重启、链路突然出现故障等。如果迷走报文到达时，发现 TCP 连接四元组所代表的连接不复存在，那么很简单，这个报文自然丢弃。如果在原连接中断后，又重新创建了一个原连接的化身，而迷失报文经过一段时间也到达，那么这个报文会被误认为是连接化身的一个 TCP 分节，这样就会对 TCP 通信产生影响

所以，TCP 就设计出了这么一个机制，经过 2MSL 这个时间，足以让两个方向上的分组都被丢弃，使得原来连接的分组在网络中都自然消失，再出现的分组一定都是新化身所产生的

如果在 TIME_WAIT 时间内，因为客户端的 ACK 没有传输到服务端，客户端又接收到了服务端重发的 FIN 报文，那么 2MSL 时间将重新计时。因为 2MSL 的时间，目的是为了让旧连接的所有报文都能自然消亡，现在客户端重新发送了 ACK 报文，自然需要重新计时，以便防止这个 ACK 报文对新可能的连接化身造成干扰

### TIME_WAIT 的危害
1. 内存资源占用
2. 端口资源的占用，可开启的端口可以通过 net.ipv4.ip_local_port_range 指定

### TIME_WAIT 的优化
通过 sysctl 命令，将 net.ipv4.tcp_max_tw_buckets 调小。这个值默认为 18000，当系统中处于 TIME_WAIT 的连接一旦超过这个值时，系统就会将所有的 TIME_WAIT 连接状态重置，并且只打印出警告信息。这个方法治标不治本，不推荐使用

调低 TCP_TIMEWAIT_LEN，重新编译系统

net.ipv4.tcp_tw_reuse
1. 只适用于连接发起方
2. 对应的 TIME_WAIT 状态的连接创建时间超过 1 秒才可以被复用

使用这个选项，还需要打开对 TCP 时间戳的支持，即 net.ipv4.tcp_time stamps=1（默认即为 1）


## 连接关闭
### close 函数
若成功则为 0，若出错则为 -1。这个函数会对套接字引用计数减一，一旦发现套接字引用计数到 0，就会对套接字进行彻底释放，并且会关闭 TCP 两个方向的数据流

在输入方向，系统内核会将该套接字设置为不可读，任何读操作都会返回异常
在输出方向，系统内核尝试将发送缓冲区的数据发送给对端，并最后向对端发送一个 FIN 报文，接下来如果再对该套接字进行写操作会返回异常

如果对端没有检测到套接字已关闭，还继续发送报文，就会收到一个 RST 报文

close 函数并不能帮助我们关闭连接的一个方向

### shutdown 函数
若成功则为 0，若出错则为 -1。howto 是这个函数的设置选项，有三个主要选项：
1. SHUT_RD(0)：关闭连接的读方向，对该套接字进行读操作直接返回 EOF。从数据角度来看，套接字上接收缓冲区已有的数据将被丢弃，如果再有新的数据流到达，会对数据进行 ACK，然后悄悄地丢弃。也就是说，对端还是会接收到 ACK，但不知道数据已经被丢弃了

2. SHUT_WR(1)：关闭连接的写方向，此时，不管套接字引用计数的值是多少，都会直接关闭连接的写方向。套接字上发送缓冲区已有的数据将被立即发送出去，并发送一个 FIN 报文给对端。应用程序如果对该套接字进行写操作会报错

3. SHUT_RDWR(2)：相当于 SHUT_RD 和 SHUT_WR 操作各一次，关闭套接字的读和写两个方向


### close vs shutdown
1. close 会关闭连接，并释放所有连接对应的资源，而 shutdown 并不会释放掉套接字和所有的资源
2. close 存在引用计数的概念，并不一定导致该套接字不可用，shutdown 则不管引用计数，直接使得该套接字不可用，如果有别的进程企图使用该套接字，将会受到影响
3. close 的引用计数导致不一定会发出 FIN 结束报文，而 shutdown 则总是会发出 FIN 结束报文

客户端
```c
#include "lib/common.h"
#define MAXLINE 4096

int main(int argc, char **argv)
{
    if (argc != 2) {
        error(1, 0, "usage: graceclient <IPaddress>");
    }

    int socket_fd;
    socket_fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in server_addr;
    bzero(&server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, argv[1], &server_addr.sin_addr);

    socklen_t server_len = sizeof(server_addr);
    int connect_rt = connect(socket_fd, (struct sockaddr *) &server_addr, server_len);
    if (connect_rt < 0) {
        error(1, errno, "connect failed ");
    }

    char send_line[MAXLINE], recv_line[MAXLINE + 1];
    int n;
    fd_set readmask;
    fd_set allreads;

    FD_ZERO(&allreads);
    FD_SET(0, &allreads);
    FD_SET(socket_fd, &allreads);

    for (;;) {
        readmask = allreads;
        int rc = select(socket_fd + 1, &readmask, NULL, NULL, NULL);
        if (rc < 0) {
            error(1, errno, "select failed");
        }

        if (FD_ISSET(socket_fd, &readmask)) {
            n = read(socket_fd, recv_line, MAXLINE);

            if (n < 0) {
                error(1, errno, "read error");
            } else if (n == 0) {
                error(1, 0, "server terminated \n");
            }
            recv_line[n] = 0;
            fputs(recv_line, stdout);
            fputs("\n", stdout);
        }

        if (FD_ISSET(0, &readmask)) {
            if (fgets(send_line, MAXLINE, stdin) != NULL) {
                if (strncmp(send_line, "shutdown", 8) == 0) {
                    FD_CLR(0, &allreads);
                    if (shutdown(socket_fd, 1)) {
                        error(1, errno, "shutdown failed");
                    }
                } else if (strncmp(send_line, "close", 5) == 0) {
                    FD_CLR(0, &allreads);
                    if (close(socket_fd)) {
                        error(1, errno, "close failed");
                    }

                    sleep(6);
                    exit(0);
                } else {
                    int i = strlen(send_line);

                    if (send_line[i - 1] == '\n') {
                        send_line[i - 1] = 0;
                    }

                    printf("now sending %s\n", send_line);
                    size_t rt = write(socket_fd, send_line, strlen(send_line));

                    if (rt < 0) {
                        error(1, errno, "write failed ");
                    }

                    printf("send bytes: %zu \n", rt);
                }
            }
        }
    }
}
```

服务端
```c
#include "lib/common.h"

static int count;

static void sig_int(int signo)
{
    printf("\nreceived %d datagrams\n", count);
    exit(0);
}

int main(int argc, char **argv)
{
    int listenfd;
    listenfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in server_addr;
    bzero(&server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(SERV_PORT);

    int rt1 = bind(listenfd, (struct sockaddr *) &server_addr, sizeof(server_addr));
    if (rt1 < 0) {
        error(1, errno, "bind failed ");
    }

    int rt2 = listen(listenfd, LISTENQ);
    if (rt2 < 0) {
        error(1, errno, "listen failed ");
    }

    signal(SIGINT, sig_int);
    signal(SIGPIPE, SIG_IGN);

    int connfd;
    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);

    if ((connfd = accept(listenfd, (struct sockaddr *) &client_addr, &client_len)) < 0) {
        error(1, errno, "bind failed ");
    }

    char message[MAXLINE];
    count = 0;

    for (;;) {
        int n = read(connfd, message, MAXLINE);
        if (n < 0) {
            error(1, errno, "error read");
        } else if (n == 0) {
            error(1, 0, "client closed \n");
        }

        message[n] = 0;
        printf("received %d bytes: %s\n", n, message);
        count++;

        char send_line[MAXLINE];
        sprintf(send_line, "Hi, %s", message);

        sleep(5);
        int write_nc = send(connfd, send_line, strlen(send_line), 0);
        printf("send bytes: %zu \n", write_nc);

        if (write_nc < 0) {
            error(1, errno, "error write");
        }
    }
}
```

1. 启动服务器，再启动客户端，依次在标准输入上输入 data1、data2 和 close
2. 启动服务器，再启动客户端，依次在标准输入上输入 data1、data2 和 shutdown


## 连接有效性
Keep-Alive 机制：在一个时间段内，如果没有任何连接相关的活动，TCP 保活机制会开始作用。每隔一个时间间隔，发送一个探测报文，该探测报文包含的数据非常少，如果连续几个探测报文都没有得到响应，则认为当前的 TCP 连接已经死亡，系统内核将错误信息通知给上层应用程序

上述的可定义变量，分别被称为保活时间、保活时间间隔和保活探测次数。在 Linux 系统中，这些变量分别对应 sysctl 变量 net.ipv4.tcp_keepalive_time、net.ipv4.tcp_keepalive_intvl、net.ipv4.tcp_keepalve_probes，默认设置是 7200 秒、75 秒和 9 次探测

如果开启了 TCP 保活，需要考虑以下几种情况:
1. 对端程序是正常工作的。当 TCP 保活的探测报文发送给对端, 对端会正常响应，这样 TCP 保活时间会被重置，等待下一个 TCP 保活时间的到来
2. 对端程序崩溃并重启。当 TCP 保活的探测报文发送给对端后，对端是可以响应的，但由于没有该连接的有效信息，会产生一个 RST 报文，这样很快就会发现 TCP 连接已经被重置
3. 是对端程序崩溃，或对端由于其他原因导致报文不可达。当 TCP 保活的探测报文发送给对端后，没有响应，达到保活探测次数后，TCP 会报告该 TCP 连接已经死亡

TCP 保活机制默认是关闭的，可以分别在连接的两个方向上开启，也可以单独在一个方向上开启。如果开启服务器端到客户端的检测，就可以在客户端非正常断连的情况下清除在服务器端保留的脏数据；而开启客户端到服务器端的检测，就可以在服务器无响应的情况下，重新发起连接

### 应用层探活
如果使用 TCP 自身的 keep-Alive 机制，在 Linux 系统中，最少需要经过 2 小时 11 分 15 秒才可以发现一个死亡连接。对很多对时延要求敏感的系统中，这个时间间隔是不可接受的所以，必须在应用程序这一层来寻找更好的解决方案。我们可以通过在应用程序中模拟 TCP Keep-Alive 机制，来完成在应用层的连接探活

需要保活的一方，比如客户端，在保活时间达到后，发起对连接的 PING 操作，如果服务器端对 PING 操作有回应，则重新设置保活时间，否则对探测次数进行计数，如果最终探测次数达到了保活探测次数预先设置的值之后，则认为连接已经无效

1. 需要使用定时器
2. 设计一个 PING-PONG 的协议

消息格式设计
```c
#ifndef MESSAGE_OBJECTE_H
#define MESSAGE_OBJECTE_H

typedef struct {
    u_int32_t type;
    char data[1024];
} messageObject;

#define MSG_PING 1
#define MSG_PONG 2
#define MSG_TYPE1 11
#define MSG_TYPE2 21

#endif
```

客户端
```c
#include "lib/common.h"
#include "message_objecte.h"

#define MAXLINE 4096
#define KEEP_ALIVE_TIME 10
#define KEEP_ALIVE_INTERVAL 3
#define KEEP_ALIVE_PROBETIMES 3

int main(int argc, char **argv)
{
    if (argc != 2) {
        error(1, 0, "usage: tcpclient <IPaddress>");
    }

    int socket_fd;
    socket_fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in server_addr;
    bzero(&server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, argv[1], &server_addr.sin_addr);

    socklen_t server_len = sizeof(server_addr);
    int connect_rt = connect(socket_fd, (struct sockaddr *) &server_addr, server_len);

    if (connect_rt < 0) {
        error(1, errno, "connect failed ");
    }

    char recv_line[MAXLINE + 1];
    int n;
    fd_set readmask;
    fd_set allreads;

    struct timeval tv;
    int heartbeats = 0;

    tv.tv_sec = KEEP_ALIVE_TIME;
    tv.tv_usec = 0;

    messageObject messageObject;

    FD_ZERO(&allreads);
    FD_SET(0, &allreads);
    FD_SET(socket_fd, &allreads);

    for (;;) {
        readmask = allreads;
        int rc = select(socket_fd + 1, &readmask, NULL, NULL, &tv);

        if (rc < 0) {
            error(1, errno, "select failed");
        }

        if (rc == 0) {
            if (++heartbeats > KEEP_ALIVE_PROBETIMES) {
                error(1, 0, "connection dead\n");
            }

            printf("sending heartbeat #%d\n", heartbeats);
            messageObject.type = htonl(MSG_PING);
            rc = send(socket_fd, (char *) &messageObject, sizeof(messageObject), 0);

            if (rc < 0) {
                error(1, errno, "send failure");
            }

            tv.tv_sec = KEEP_ALIVE_INTERVAL;
            continue;
        }

        if (FD_ISSET(socket_fd, &readmask)) {
            n = read(socket_fd, recv_line, MAXLINE);

            if (n < 0) {
                error(1, errno, "read error");
            } else if (n == 0) {
                error(1, 0, "server terminated \n");
            }

            printf("received heartbeat, make heartbeats to 0 \n");
            heartbeats = 0;
            tv.tv_sec = KEEP_ALIVE_TIME;
        }
    }
}
```

服务器端
```c
#include "lib/common.h"
#include "message_objecte.h"

static int count;

static void sig_int(int sig) {
    printf("\nreceived %d datagrams\n", count);
    exit(0);
}

int main(int argc, char **argv)
{
    if (argc != 2) {
        error(1, 0, "usage: tcpsever <sleepingtime>");
    }

    int sleepingTime = atoi(argv[1]);
    int listenfd;
    listenfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in server_addr;
    bzero(&server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(SERV_PORT);

    int rt1 = bind(listenfd, (struct sockaddr *) &server_addr, sizeof(server_addr));
    if (rt1 < 0) {
        error(1, errno, "bind failed ");
    }

    int rt2 = listen(listenfd, LISTENQ);
    if (rt2 < 0) {
        error(1, errno, "listen failed ");
    }

    signal(SIGINT, sig_int);
    signal(SIGPIPE, SIG_IGN);

    int connfd;
    struct sockaddr_in client_addr;
    socklen_t client_len = sizeof(client_addr);

    if ((connfd = accept(listenfd, (struct sockaddr *) &client_addr, &client_len)) < 0) {
        error(1, errno, "bind failed ");
    }

    messageObject message;
    count = 0;

    for (;;) {
        int n = read(connfd, (char *) &message, sizeof(messageObject));

        if (n < 0) {
            error(1, errno, "error read");
        } else if (n == 0) {
            error(1, 0, "client closed \n");
        }

        printf("received %d bytes\n", n);
        count++;

        switch (ntohl(message.type)) {
            case MSG_TYPE1:
                printf("process  MSG_TYPE1 \n");
                break;
            case MSG_TYPE2:
                printf("process  MSG_TYPE2 \n");
                break;
            case MSG_PING:
                messageObject pong_message;
                pong_message.type = MSG_PONG;
                sleep(sleepingTime);
                ssize_t rc = send(connfd, (char *) &pong_message, sizeof(pong_message), 0)
                if (rc < 0)
                    error(1, errno, "send failure");
                break;
            default:
                error(1, 0, "unknown message type (%d)\n", ntohl(message.type));
        }
    }
}
```


## 传输控制
1. 接收端不能在接收缓冲区空出一个很小的部分之后，就立马向发送端发送窗口更新通知，而是需要在自己的缓冲区大到一个合理的值之后，再向发送端发送窗口更新通知
2. Nagle 算法：限制大批量的小数据包同时发送，在任何一个时刻，未被确认的小数据包不能超过一个。这里的小数据包，指的是长度小于最大报文段长度 MSS 的 TCP 分组。发送端把连续的几个小数据包存储起来，等待接收到前一个小数据包的 ACK 分组之后，再将数据一次性发送出去
3. 延时 ACK：在收到数据后并不马上回复，而是累计需要发送的 ACK 报文，等到有数据需要发送给对端时，将累计的 ACK 捎带一并发送出去。延时 ACK 机制不能无限地延时下去，否则发送端误认为数据包没有发送成功，引起重传，会占用额外的网络带宽

在有些场景下，Nagle 算法和延时 ACK 的组合增大了处理时延，对时延敏感的应用并不适用

关闭 Nagle 算法
```c
int on = 1;
setsockopt(sock, IPPROTO_TCP, TCP_NODELAY, (void *)&on, sizeof(on));
```

写操作合并
```c
int main(int argc, char **argv)
{
    if (argc != 2) {
        error(1, 0, "usage: tcpclient <IPaddress>");
    }

    int socket_fd;
    socket_fd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in server_addr;
    bzero(&server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, argv[1], &server_addr.sin_addr);

    socklen_t server_len = sizeof(server_addr);
    int connect_rt = connect(socket_fd, (struct sockaddr *) &server_addr, server_len);
    if (connect_rt < 0) {
        error(1, errno, "connect failed ");
    }

    char buf[128];
    struct iovec iov[2];

    char *send_one = "hello,";
    iov[0].iov_base = send_one;
    iov[0].iov_len = strlen(send_one);
    iov[1].iov_base = buf;

    while (fgets(buf, sizeof(buf), stdin) != NULL) {
        iov[1].iov_len = strlen(buf);
        int n = htonl(iov[1].iov_len);
        if (writev(socket_fd, iov, 2) < 0)
            error(1, errno, "writev failure");
    }

    exit(0);
}
```


## 重用套接字
Linux 操作系统避免由于 TIME_WAIT 造成的新旧连接化身冲突的问题
1. 新连接 SYN 的初始序列号，一定比 TIME_WAIT 老连接的末序列号大，这样就可以区别出新老连接
2. 开启 tcp_timestamps，新连接的时间戳比老连接的时间戳大，通过时间戳也可以区别出新老连接

这样，一个 TIME_WAIT 的 TCP 连接可以忽略掉旧连接，重新被新的连接所使用

SO_REUSEADDR 套接字选项，允许启动绑定在一个端口，即使之前存在一个和该端口一样的连接，这样的 TCP 连接完全可以复用 TIME_WAIT 状态的连接
```c
// 在 bind 监听套接字之前，调用 setsockopt 方法，设置重用套接字选项
int on = 1;
setsockopt(listenfd, SOL_SOCKET, SO_REUSEADDR, &on, sizeof(on));
```

SO_REUSEADDR 套接字选项还有一个作用：如果本机服务器有多个地址，可以在不同地址上使用相同的端口提供服务

比如，一台服务器有 192.168.1.101 和 10.10.2.102 连个地址，我们可以在这台机器上启动三个不同的 HTTP 服务，第一个以本地通配地址 ANY 和端口 80 启动；第二个以 192.168.101 和端口 80 启动；第三个以 10.10.2.102 和端口 80 启动

服务器端程序，都应该设置 SO_REUSEADDR 套接字选项，以便服务端程序可以在极短时间内复用同一个端口启动

tcp_tw_reuse 是内核选项，主要用在连接的发起方。TIME_WAIT 状态的连接创建时间超过 1 秒后，新的连接才可以被复用

SO_REUSEADDR 选项用来告诉操作系统内核，如果端口已被占用，但是 TCP 连接状态位于 TIME_WAIT，可以重用端口。如果端口忙，而 TCP 处于其他状态，重用端口时依旧得到 Address already in use 的错误信息。这里一般都是连接的服务方

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

    sock = socket(PF_INET, SOCK_STREAM, 0);

    if (sock < 0)
    {
        perror("socket");
        exit(EXIT_FAILURE);
    }

    name.sin_family = AF_INET;
    name.sin_port = htons(port);
    name.sin_addr.s_addr = htonl(INADDR_ANY);

    if (bind(sock, (struct sockaddr *)&name, sizeof(name)) < 0)
    {
        perror ("bind");
        exit(EXIT_FAILURE);
    }

    return sock;
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


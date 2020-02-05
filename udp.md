## TCP vs UDP
1. TCP 是一个面向连接的协议，TCP 在 ip 报文的基础上，增加了诸如重传、确认、有序传 输、拥塞控制等能力
2. UDP 是一个不可靠的通信协议，没有重传和确认，没有有序控制，也没有拥塞控制。UDP 不保证报文的有效传递，不保证报文的有序，需要自己做好丢包、重传、报文组装等工作

常见的 DNS 服务，SNMP 服务都是基于 UDP 协议的，这些场景对时延、丢包都不是特别敏感。另外多人通信的场景，如聊天室、多人游戏等，也都会使用到 UDP 协议


## UDP 通信
服务器端创建 UDP 套接字之后，绑定到本地端口，调用 recvfrom 函数等待客户端的报文发送；客户端创建套接字之后，调用 sendto 函数往目标地址和端口发送 UDP 报文

```c
ssize_t recvfrom(int sockfd, void *buff, size_t nbytes, int flags,
                  struct sockaddr *from, socklen_t *addrlen);
```
1. sockfd 表示本地创建的套接字描述符
2. buff 指向本地的缓存
3. nbytes 表示最大接收数据字节
4. flags 是和 I/O 相关的参数，设置为 0
5. from 和 addrlen 是返回对端发送方的地址和端口信息

函数的返回值表示实际接收的字节数

```c
ssize_t sendto(int sockfd, const void *buff, size_t nbytes, int flags,
                const struct sockaddr *to, socklen_t *addrlen);
```
1. sockfd 表示本地创建的套接字描述符
2. buff 指向发送的缓存
3. nbytes 表示发送字节数
4. flags 设置为 0
5. to 和 addrlen 表示发送的对端地址和端口等信息

函数的返回值表示实际接收的字节数


## 示例
### server
```c
#include "lib/common.h"

static int count;

static void recvfrom_int(int signo)
{
    printf("\nreceived %d datagrams\n", count);
    exit(0);
}

int main(int argc, char **argv) {
    int socket_fd;
    socket_fd = socket(AF_INET, SOCK_DGRAM, 0);

    struct sockaddr_in server_addr;
    bzero(&server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = htonl(INADDR_ANY);
    server_addr.sin_port = htons(SERV_PORT);

    bind(socket_fd, (struct sockaddr *) &server_addr, sizeof(server_addr));
    socklen_t client_len;
    char message[MAXLINE];
    count = 0;

    signal(SIGINT, recvfrom_int);

    struct sockaddr_in client_addr;
    client_len = sizeof(client_addr);

    while (1) {
        int n = recvfrom(socket_fd, message, MAXLINE, 0,
            (struct sockaddr *) &client_addr, &client_len);
        message[n] = 0;
        printf("received %d bytes: %s\n", n, message);

        char send_line[MAXLINE];
        sprintf(send_line, "Hi, %s", message);
        sendto(socket_fd, send_line, strlen(send_line), 0,
            (struct sockaddr *) &client_addr, client_len);
        count++;
    }
}
```

### client
```c
#include "lib/common.h"

#define MAXLINE 4096

int main(int argc, char **argv)
{
    if (argc != 2)
    {
        error(1, 0, "usage: udpclient <IPaddress>");
    }

    int socket_fd;
    socket_fd = socket(AF_INET, SOCK_DGRAM, 0);

    struct sockaddr_in server_addr;
    bzero(&server_addr, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, argv[1], &server_addr.sin_addr);
    socklen_t server_len = sizeof(server_addr);
    struct sockaddr *reply_addr;
    reply_addr = malloc(server_len);

    char send_line[MAXLINE], recv_line[MAXLINE + 1];
    socklen_t len;
    int n;

    while (fgets(send_line, MAXLINE, stdin) != NULL) {
        int i = strlen(send_line);
        if (send_line[i - 1] == '\n') {
            send_line[i - 1] = 0;
        }

        printf("now sending %s\n", send_line);
        size_t rt = sendto(socket_fd, send_line, strlen(send_line), 0, (struct sockaddr *) &server_addr, &server_len);
        if (rt < 0) {
            error(1, errno, "send failed ");
        }
        printf("send bytes: %zu \n", rt);
        len = 0;
        n = recvfrom(socket_fd, recv_line, MAXLINE, 0, reply_addr, &len);
        if (n < 0) {
            error(1, errno, "recvfrom failed");
        }
        recv_line[n] = 0;
        fputs(recv_line, stdout);
        fputs("\n", stdout);
    }
    exit(0);
}
```

如果只运行客户端，UDP 程序会一直阻塞，TCP 客户端的 connect 函数会直接返回 Connection refused 的报错信息

服务器端重启后可以继续收到客户端的报文，这在 TCP 里是不可以的，因为 TCP 断联之后必须重新连接才可以发送报文信息，而 UDP 报文的无连接的，可以在 UDP 服务器重启之后，继续进行报文的发送


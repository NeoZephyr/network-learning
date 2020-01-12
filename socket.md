## 通用套接字
```c
typedef unsigned short int sa_family_t;

struct sockaddr {
    sa_family_t sa_family; // 地址族
    char sa_data[14]; // 具体的地址值
};
```

地址族
AF_LOCAL：表示本地地址，对应 Unix 套接字，一般用于本地 socket 通信，也可以写成 AF_UNIX、AF_FILE
AF_INET：ipv4 地址
AF_INET6：ipv6 地址


## ipv4 套接字
```c
typedef uint32_t in_addr_t;

struct in_addr {
    in_addr_t s_addr;
}

struct sockaddr_in {
    sa_family_t sin_family;
    in_port_t sin_port; // 两个字节
    struct in_addr sin_addr;
    unsigned char sin_zero[8];
}
```


## ipv6 套接字
```c
struct sockaddr_in6 {
    sa_family_t sin6_family;
    in_port_t sin6_port;
    uint32_t sin6_flowinfo;
    struct in6_addr sin6_addr;
    uint32_t sin6_scope_id;
};
```


## 本地套接字
```c
struct sockaddr_un {
    unsigned short sun_family;
    char sun_path[108];
}
```

TCP/UDP 即使在本地地址通信，也要走系统网络协议栈，而本地套接字，提供了一种单主机跨进程间调用的手段，减少了协议栈实现的复杂度，效率比 TCP/UDP 套接字都要高许多

### server
```c
#include "lib/common.h"

int main(int argc, char **argv)
{
    if (argc != 2) {
        error(1, 0, "usage: unixdataserver <local_path>");
    }

    int socket_fd;
    socket_fd = socket(AF_LOCAL, SOCK_DGRAM, 0);
    if (socket_fd < 0) {
        error(1, errno, "socket create failed");
    }

    struct sockaddr_un servaddr;
    char *local_path = argv[1];
    unlink(local_path);
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sun_family = AF_LOCAL;
    strcpy(servaddr.sun_path, local_path);

    if (bind(socket_fd, (struct sockaddr *) &servaddr, sizeof(servaddr)) < 0) {
        error(1, errno, "bind failed");
    }

    char buf[BUFFER_SIZE];
    struct sockaddr_un client_addr;
    socklen_t client_len = sizeof(client_addr);

    while (1) {
        bzero(buf, sizeof(buf));
        if (recvfrom(socket_fd, buf, BUFFER_SIZE, 0,
            (struct sockadd *) &client_addr, &client_len)) {
            printf("client quit");
            break;
        }

        printf("Receive: %s \n", buf);
        char send_line[MAXLINE];
        bzero(send_line, MAXLINE);
        sprintf(send_line, "Hi, %s", buf);
        size_t nbytes = strlen(send_line);
        printf("now sending: %s \n", send_line);
        if (sendto(socket_fd, send_line, nbytes, 0,
            (struct sockadd *) &client_addr, client_len)) {
            error(1, errno, "sendto error");
        }
    }

    close(socket_fd);
    exit(0);
}
```

### client
```c
#include "lib/common.h"

int main(int argc, char **argv)
{
    if (argc != 2) {
        error(1, 0, "usage: unixdataclient <local_path>");
    }

    int sockfd;
    struct sockaddr_un client_addr, server_addr;

    sockfd = socket(AF_LOCAL, SOCK_DGRAM, 0);

    if (sockfd < 0) {
        error(1, errno, "create socket failed");
    }

    bzero(&client_addr, sizeof(client_addr));
    client_addr.sun_family = AF_LOCAL;
    strcpy(client_addr.sun_path, tmpnam(NULL));

    if (bind(sockfd, (struct sockaddr *) &client_addr, sizeof(client_addr)) < 0) {
        error(1, errno, "bind failed");
    }

    bzero(&server_addr, sizeof(server_addr));
    server_addr.sun_family = AF_LOCAL;
    strcpy(server_addr.sun_path, argv[1]);

    char send_line[MAXLINE];
    bzero(send_line, MAXLINE);
    char recv_line[MAXLINE];

    while (fgets(send_line, MAXLINE, stdin) != NULL) {
        int i = strlen(send_line);
        if (send_line[i - 1] == '\n') {
            send_line[i - 1] = 0;
        }

        size_t nbytes = strlen(send_line);
        printf("now sending %s \n", send_line);

        if (sendto(sockfd, send_line, nbytes, 0, (struct sockaddr *) &server_addr, sizeof(server_addr))) {
            error(1, errno, "sendto error");
        }

        int n = recvfrom(sockfd, recv_line, MAXLINE, 0, NULL, NULL);
        recv_line[n] = 0;
        fputs(recv_line, stdout);
        fputs("\n", stdout);
    }

    exit(0);
}
```


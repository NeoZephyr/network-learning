## 发送数据
常见的文件写函数
```c
ssize_t write (int socketfd, const void *buffer, size_t size)
```

```c
ssize_t send (int socketfd, const void *buffer, size_t size, int flags)
```

```c
ssize_t sendmsg(int sockfd, const struct msghdr *msg, int flags)
```

对于普通文件描述符而言，一个文件描述符代表了打开的一个文件句柄，通过调用 write 函数，操作系统内核不断地往文件系统中写入字节流。写入的字节流大小通常和输入参数 size 的值是相同的，否则表示出错

对于套接字描述符而言，它代表了一个双向连接，调用 write 写入的字节数有可能比请求的数量少，产生这个现象的原因在于操作系统内核为读取和发送数据做了很多额外工作

### 发送缓冲区
当 TCP 三次握手成功，连接成功建立后，操作系统内核会为每一个连接创建配套的基础设施，比如发送缓冲区

发送缓冲区的大小可以通过套接字选项来改变，当调用 write 函数时，实际所做的事情是把数据从应用程序中拷贝到操作系统内核的发送缓冲区中，并不一定是把数据 通过套接字写出去

这里有几种情况：
1. 操作系统内核的发送缓冲区足够大，可以直接容纳这份数据，那么程序将从 write 调用中退出，返回写入的字节数就是应用程序的数据大小
2. 操作系统内核的发送缓冲区足够大，不过还有数据没有发送完，或者数据发送完了，但是操作系统内核的发送缓冲区不足以容纳应用程序数据，在这种情况下，操作系统内核并不会返回，也不会报错，而是应用程序被阻塞。大部分 UNIX 系统会一直等到可以把数据完全放到发送缓冲区中，再从系统调用中返回


## 读取数据
```c
ssize_t read (int socketfd, void *buffer, size_t size)
```
1. size 参数表示从套接字描述字 socketfd 读取最多多少个字节，并将结果存储到 buffer 中
2. 返回值表示实际读取的字节数目，如果返回值为 0，表示 EOF，这在网络中表示对端发送了 FIN 包，要处理断连的情况；如果返回值为 -1，表示出错。当然，这是阻塞 I/O 的情况

```c
ssize_t readn(int fd, void *vptr, size_t size)
{
    size_t nleft;
    ssize_t nread;
    char *ptr;

    prt = vptr;
    nleft = size;

    while (nleft > 0) {
        if ((nread = read(fd, ptr, nleft)) < 0) {
            if (errno == EINTR) {
                nread = 0;
            } else {
                return (-1);
            }
        } else if (nread == 0) {
            break;
        }

        nleft -= nread;
        prt += nread;
    }

    return (size - nleft);
}
```


## 示例
### server
```c
int main(int argc, char **argv)
{
    int listenfd, connfd;
    socklen_t clilen;
    struct sockaddr_in cliaddr, servaddr;

    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(12345);

    bind(listenfd, (SA *)&servaddr, sizeof(servaddr));
    listen(listenfd, 1024);

    while (1) {
        clilen = sizeof(cliaddr);
        connfd = accept(listenfd, (SA *)&cliaddr, &clilen);
        read_data(connfd);
        close(connfd);
    }
}
```
```c
void read_data(int sockfd)
{
    ssize_t  n;
    char buf[1024];

    int time = 0;

    while (1) {
        fprintf(stdout, "block in read\n");
        if ((n = Readn(sockfd, buf, 1024)) == 0)
            return;

        time ++;
        fprintf(stdout, "1K read for %d \n", time);
        usleep(1000);
    }
}
```

### client
```c
int main(int argc, char **argv)
{
    int sockfd;
    struct sockaddr_in servaddr;

    if (argc != 2) {
        err_quit("usage: tcpclient <IPaddress>");
    }

    sockfd = socket(AF_INET, SOCK_STREAM, 0);
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, argv[1], &servaddr.sin_addr);
    connect(sockfd, (SA *) &servaddr, sizeof(servaddr));
    send_data(stdin, sockfd);
    exit(0);
}
```
```c
# define MESSAGE_SIZE 10240000

void send_data(FILE *fp, int sockfd)
{
    char * query;
    query = malloc(MESSAGE_SIZE + 1);

    for (int i = 0; i < MESSAGE_SIZE; i++) {
        query[i] = 'a';
    }

    query[MESSAGE_SIZE] = '\0';
    const char *cp;
    cp = query;
    remaining = strlen(query);

    while (remaining) {
        n_written = send(sockfd, cp, remaining, 0);
        fprintf(stdout, "send into buffer %ld \n", n_written);
        if (n_written <= 0) {
            perror("send");
            return;
        }

        remaining -= n_written;
        cp += n_written;
    }

    return;
}
```


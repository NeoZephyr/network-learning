## select
select 函数就是一种常见的 I/O 多路复用技术。使用 select 函数时，通知内核挂起进程，当一个或多个 I/O 事件发生后，控制权返还给应用程序，由应用程序进行 I/O 事件的处理

### 函数声明
第一个参数，表示的是待测试的描述符基数，它的值是待测试的最大描述符加 1
接下来的三个参数，分别是读描述符集合 readset、写描述符集合 writeset 和异常描述符集合 exceptset，这三个分别通知内核，在哪些描述符上检测数据可以读，可以写和有异常发生
最后一个参数是 timeval 结构体时间，这个参数有以下取值：
1. 设置成空，表示如果没有 I/O 事件发生，则 select 一直等待下去
2. 设置一个非零的值，表示等待固定的一段时间后从 select 阻塞调用中返回
3. 将 tv_sec 和 tv_usec 设置成 0，表示不等待，检测完毕立即返回

```c
#include "lib/common.h"

#define MAXLINE 1024

int main(int argc, char **argv)
{
    if (argc != 2) {
        error(1, 0, "usage: select01 <IPaddress>");
    }
    int socket_fd = tcp_client(argv[1], SERV_PORT);

    char recv_line[MAXLINE], send_line[MAXLINE];
    int n;

    fd_set readmask;
    fd_set allreads;

    // 初始化描述符集合
    FD_ZERO(&allreads);

    // 将描述符 0，即标准输入，以及连接套接字描述符设置为待检测
    FD_SET(0, &allreads);
    FD_SET(socket_fd, &allreads);

    for (;;) {
        readmask = allreads;
        int rc = select(socket_fd + 1, &readmask, NULL, NULL, NULL);

        if (rc <= 0) {
            error(1, errno, "select failed");
        }

        // 判断描述符是否可读
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

        if (FD_ISSET(STDIN_FILENO, &readmask)) {
            if (fgets(send_line, MAXLINE, stdin) != NULL) {
                int i = strlen(send_line);
                if (send_line[i - 1] == '\n') {
                    send_line[i - 1] = 0;
                }

                printf("now sending %s\n", send_line);
                ssize_t rt = write(socket_fd, send_line, strlen(send_line));
                if (rt < 0) {
                    error(1, errno, "write failed ");
                }
                printf("send bytes: %zu \n", rt);
            }
        }
    }
}
```

### 就绪条件
1. 套接字接收缓冲区有数据可以读
2. 对方发送了 FIN，使用 read 函数执行读操作，不会被阻塞，直接返回 0
3. 有已经完成的连接建立，此时使用 accept 函数去执行不会阻塞，直接返回已经完成的连接
4. 套接字有错误待处理，使用 read 函数去执行读操作，不阻塞，且返回 -1




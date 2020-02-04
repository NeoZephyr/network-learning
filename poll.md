## poll
### 函数声明
```c
int poll(struct pollfd *fds, unsigned long nfds, int timeout);
```
第一个参数是一个 pollfd 的数组。其中 pollfd 的结构如下：
```c
struct pollfd
{
    int fd;
    short events;
    short revents;
};
```
fd 为描述符；events 为待检测的事件类型，可以表示多个不同的事件

和 select 不同的地方在于，poll 每次检测之后的结果不会修改原来的传入值，而是将结果保留在 revents 字段中，这样就不需要每次检测完都得重置待检测的描述字和感兴趣的事件

第二个参数，表示向 poll 申请的事件检测的个数

最后一个参数 timeout，有以下取值：
1. 小于 0 表示在有事件发生之前永远等待
2. 等于 0 表示不阻塞进程，立即返回
3. 大于 0 表示等待指定的毫秒数后返回

函数有以下返回值：
1. 返回 -1，表示有错误发生
2. 返回 0，表示时间到达之前没有任何事件发生
3. 返回大于 0，表示检测到的事件个数

### poll vs select
1. 如果不想对某个 pollfd 结构进行事件检测，可以把它对应的 pollfd 结构的 fd 成员设置成一个负值。这样，poll 函数将忽略这样的 events 事件，检测完成以后，所对应的 returned events 的成员值也将设置为 0
2. 在 select 里面，文件描述符的个数已经随着 fd_set 的实现而固定，没有办法对此进行配置；而在 poll 函数里，可以控制 pollfd 结构的数组大小。这意味着可以突破原来 select 函数最大描述符的限制

```c
#include "lib/common.h"

#define INIT_SIZE 128

int main(int argc, char **argv)
{
    int listen_fd, connected_fd;
    int ready_number;
    ssize_t n;
    char buf[MAXLINE];
    struct sockaddr_in client_addr;

    listen_fd = tcp_server_listen(SERV_PORT);

    // 初始化 pollfd 数组
    // 数组的第一个元素是 listen_fd，其余的用来记录将要连接的 connect_fd
    struct pollfd event_set[INIT_SIZE];
    event_set[0].fd = listen_fd;
    event_set[0].events = POLLRDNORM;

    // -1 表示这个数组位置还没有被占用
    int i;
    for (i = 1; i < INIT_SIZE; i++) {
        event_set[i].fd = -1;
    }

    for (;;) {
        if ((ready_number = poll(event_set, INIT_SIZE, -1)) < 0) {
            error(1, errno, "poll failed ");
        }

        // 连接建立事件
        if (event_set[0].revents & POLLRDNORM) {
            socklen_t client_len = sizeof(client_addr);
            connected_fd = accept(listen_fd, (struct sockaddr *) &client_addr, &client_len);

            // 找到一个可以记录该连接套接字的位置
            for (i = 1; i < INIT_SIZE; i++) {
                if (event_set[i].fd < 0) {
                    event_set[i].fd = connected_fd;
                    event_set[i].events = POLLRDNORM;
                    break;
                }
            }

            if (i == INIT_SIZE) {
                error(1, errno, "can not hold so many clients");
            }

            if (--ready_number <= 0)
                continue;
        }

        // 可读事件
        for (i = 1; i < INIT_SIZE; i++) {
            int socket_fd;
            if ((socket_fd = event_set[i].fd) < 0)
                continue;
            if (event_set[i].revents & (POLLRDNORM | POLLERR)) {
                if ((n = read(socket_fd, buf, MAXLINE)) > 0) {
                    if (write(socket_fd, buf, n) < 0) {
                        error(1, errno, "write error");
                    }
                } else if (n == 0 || errno == ECONNRESET) {
                    close(socket_fd);
                    event_set[i].fd = -1;
                } else {
                    error(1, errno, "read error");
                }

                if (--ready_number <= 0)
                    break;
            }
        }
    }
}
```
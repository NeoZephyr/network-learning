## epoll
epoll 通过监控注册的多个描述字，来进行 I/O 事件的分发处理。epoll 通过改进的接口设计，避免了用户态与内核态频繁的数据拷贝，大大提高了系统性能。不同于 poll 的是，epoll 不仅提供了默认的 level-triggered（条件触发）机制，还提供了性能更为强劲的 edge-triggered（边缘触发）机制

### epoll_create
该方法创建了一个 epoll 实例。如果这个 epoll 实例不再需要，比如服务器正常关机，需要调用 close 方法释放 epoll 实例，这样系统内核可以回收 epoll 实例所分配使用的内核资源

```c
int epoll_create(int size);
int epoll_create1(int flags);
```
关于这个参数 size，在一开始的 epoll_create 实现中，是用来告知内核期望监控的文件描述字大小，然后内核使用这部分的信息来初始化内核数据结构。在新的实现中，这个参数不再被需要，因为内核可以动态分配需要的内核数据结构。现在每次将 size 设置成一个大于 0 的整数就可以了

### epoll_ctl
通过调用 epoll_ctl 往这个 epoll 实例增加或删除监控的事件
```c
int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```
1. 参数 epfd 是调用 epoll_create 创建的 epoll 实例描述字
2. 参数 op 表示增加还是删除一个监控事件，有三个选项可供选择：EPOLL_CTL_ADD 表示向 epoll 实例注册文件描述符对应的事件；EPOLL_CTL_DEL 表示向 epoll 实例删除文件描述符对应的事件；EPOLL_CTL_MOD 表示修改文件描述符对应的事件
3. 参数 fd 是注册的事件的文件描述符
4. 参数 event 表示的是注册的事件类型，并且可以在这个结构体里设置用户需要的数据，其中最为常见的是使用联合结构里的 fd 字段，表示事件所对应的文件描述符。

```c
typedef union epoll_data {
    void        *ptr;
    int          fd;
    uint32_t     u32;
    uint64_t     u64;
} epoll_data_t;

struct epoll_event {
    uint32_t     events;
    epoll_data_t data;
}
```

事件类型：
1. EPOLLIN 表示对应的文件描述字可以读
2. EPOLLOUT 表示对应的文件描述字可以写
3. EPOLLRDHUP 表示套接字的一端已经关闭，或者半关闭
4. EPOLLHUP 表示对应的文件描述字被挂起
5. EPOLLET 设置为 edge-triggered，默认为 level-triggered

### epoll_wait
调用者进程被挂起，在等待内核 I/O 事件的分发
```c
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

1. 第一个参数是 epoll 实例描述字
2. 第二个参数返回给用户空间需要处理的 I/O 事件，是一个数组，数组的大小由 epoll_wait 的返回值决定，这个数组的每个元素都是一个需要待处理的 I/O 事件
3. 第三个参数是一个大于 0 的整数，表示 epoll_wait 可以返回的最大事件值
4. 第四个参数是 epoll_wait 阻塞调用的超时值，如果这个值设置为 -1，表示不超时；如果设置为 0 则立即返回，即使没有任何 I/O 事件发生

```c
#include  <sys/epoll.h>
#include "lib/common.h"

#define MAXEVENTS 128

char rot13_char(char c) {
    if ((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
        return c + 13;
    else if ((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
        return c - 13;
    else
        return c;
}

int main(int argc, char **argv)
{
    int listen_fd, socket_fd;
    int n, i;
    int efd;
    struct epoll_event event;
    struct epoll_event *events;

    listen_fd = tcp_nonblocking_server_listen(SERV_PORT);

    efd = epoll_create1(0);
    if (efd == -1) {
        error(1, errno, "epoll create failed");
    }

    event.data.fd = listen_fd;
    event.events = EPOLLIN | EPOLLET;
    if (epoll_ctl(efd, EPOLL_CTL_ADD, listen_fd, &event) == -1) {
        error(1, errno, "epoll_ctl add listen fd failed");
    }

    events = calloc(MAXEVENTS, sizeof(event));

    while (1) {
        n = epoll_wait(efd, events, MAXEVENTS, -1);
        printf("epoll_wait wakeup\n");
        for (i = 0; i < n; i++) {
            if ((events[i].events & EPOLLERR) ||
                (events[i].events & EPOLLHUP) ||
                (!(events[i].events & EPOLLIN))) {
                fprintf(stderr, "epoll error\n");
                close(events[i].data.fd);
                continue;
            } else if (listen_fd == events[i].data.fd) {
                struct sockaddr_storage ss;
                socklen_t slen = sizeof(ss);
                int fd = accept(listen_fd, (struct sockaddr *) &ss, &slen);
                if (fd < 0) {
                    error(1, errno, "accept failed");
                } else {
                    make_nonblocking(fd);
                    event.data.fd = fd;
                    event.events = EPOLLIN | EPOLLET; // edge-triggered
                    if (epoll_ctl(efd, EPOLL_CTL_ADD, fd, &event) == -1) {
                        error(1, errno, "epoll_ctl add connection fd failed");
                    }
                }
                continue;
            } else {
                socket_fd = events[i].data.fd;
                printf("get event on socket fd == %d \n", socket_fd);
                while (1) {
                    char buf[512];
                    if ((n = read(socket_fd, buf, sizeof(buf))) < 0) {
                        if (errno != EAGAIN) {
                            error(1, errno, "read error");
                            close(socket_fd);
                        }
                        break;
                    } else if (n == 0) {
                        close(socket_fd);
                        break;
                    } else {
                        for (i = 0; i < n; ++i) {
                            buf[i] = rot13_char(buf[i]);
                        }
                        if (write(socket_fd, buf, n) < 0) {
                            error(1, errno, "write error");
                        }
                    }
                }
            }
        }
    }

    free(events);
    close(listen_fd);
}
```


## epoll 性能
1. 每次使用 poll 或 select 之前，都需要准备一个感兴趣的事件集合，系统内核拿到事件集合，进行分析并在内核空间构建相应的数据结构来完成对事件集合的注册。而 epoll 维护了一个全局的事件集合，通过 epoll 句柄，可以增加、删除或修改这个事件集合里的某个元素。在绝大多数情况下，事件集合的变化没有那么的大，这样操纵系统内核就不需要每次重新扫描事件集合，构
建内核空间数据结构
2. 每次在使用 poll 或者 select 之后，应用程序都需要扫描整个感兴趣的事件集合，从中找出真正活动的事件。事实上，很多情况下扫描完一遍，可能发现只有几个真正活动的事件。 而 epoll 返回的直接就是活动的事件列表，应用程序减少了大量的扫描时间


## 边缘触发 VS 条件触发
条件触发的意思是只要满足事件的条件，比如有数据需要读，就一直不断地把这个事件传递给用户；而边缘触发的意思是只有第一次满足条件的时候才触发，之后就不会再传递同样的事件了。一般情况下，边缘触发的效率比条件触发的效率要高

边缘触发，服务器端只从 epoll_wait 中苏醒过一次，就是第一次有数据可读的时候
```c
#include  <sys/epoll.h>
#include "lib/common.h"

#define MAXEVENTS 128

int main(int argc, char **argv) {
    int listen_fd, socket_fd;
    int n, i;
    int efd;
    struct epoll_event event;
    struct epoll_event *events;

    listen_fd = tcp_nonblocking_server_listen(SERV_PORT);

    efd = epoll_create1(0);
    if (efd == -1) {
        error(1, errno, "epoll create failed");
    }

    event.data.fd = listen_fd;

    // 设置为 edge-triggered
    event.events = EPOLLIN | EPOLLET;
    if (epoll_ctl(efd, EPOLL_CTL_ADD, listen_fd, &event) == -1) {
        error(1, errno, "epoll_ctl add listen fd failed");
    }

    events = calloc(MAXEVENTS, sizeof(event));

    while (1) {
        n = epoll_wait(efd, events, MAXEVENTS, -1);
        printf("epoll_wait wakeup\n");
        for (i = 0; i < n; i++) {
            if ((events[i].events & EPOLLERR) ||
                (events[i].events & EPOLLHUP) ||
                (!(events[i].events & EPOLLIN))) {
                fprintf(stderr, "epoll error\n");
                close(events[i].data.fd);
                continue;
            } else if (listen_fd == events[i].data.fd) {
                struct sockaddr_storage ss;
                socklen_t slen = sizeof(ss);
                int fd = accept(listen_fd, (struct sockaddr *) &ss, &slen);
                if (fd < 0) {
                    error(1, errno, "accept failed");
                } else {
                    make_nonblocking(fd);
                    event.data.fd = fd;
                    event.events = EPOLLIN | EPOLLET; // edge-triggered
                    if (epoll_ctl(efd, EPOLL_CTL_ADD, fd, &event) == -1) {
                        error(1, errno, "epoll_ctl add connection fd failed");
                    }
                }
                continue;
            } else {
                socket_fd = events[i].data.fd;
                printf("get event on socket fd == %d \n", socket_fd);
            }
        }
    }

    free(events);
    close(listen_fd);
}
```

条件触发，服务器端不断地从 epoll_wait 中苏醒，告知有数据需要读取
```c
#include  <sys/epoll.h>
#include "lib/common.h"

#define MAXEVENTS 128

int main(int argc, char **argv)
{
    int listen_fd, socket_fd;
    int n, i;
    int efd;
    struct epoll_event event;
    struct epoll_event *events;

    listen_fd = tcp_nonblocking_server_listen(SERV_PORT);

    efd = epoll_create1(0);
    if (efd == -1) {
        error(1, errno, "epoll create failed");
    }

    event.data.fd = listen_fd;
    event.events = EPOLLIN | EPOLLET;
    if (epoll_ctl(efd, EPOLL_CTL_ADD, listen_fd, &event) == -1) {
        error(1, errno, "epoll_ctl add listen fd failed");
    }

    events = calloc(MAXEVENTS, sizeof(event));

    while (1) {
        n = epoll_wait(efd, events, MAXEVENTS, -1);
        printf("epoll_wait wakeup\n");
        for (i = 0; i < n; i++) {
            if ((events[i].events & EPOLLERR) ||
                (events[i].events & EPOLLHUP) ||
                (!(events[i].events & EPOLLIN))) {
                fprintf(stderr, "epoll error\n");
                close(events[i].data.fd);
                continue;
            } else if (listen_fd == events[i].data.fd) {
                struct sockaddr_storage ss;
                socklen_t slen = sizeof(ss);
                int fd = accept(listen_fd, (struct sockaddr *) &ss, &slen);
                if (fd < 0) {
                    error(1, errno, "accept failed");
                } else {
                    make_nonblocking(fd);
                    event.data.fd = fd;
                    event.events = EPOLLIN;  // level-triggered
                    if (epoll_ctl(efd, EPOLL_CTL_ADD, fd, &event) == -1) {
                        error(1, errno, "epoll_ctl add connection fd failed");
                    }
                }
                continue;
            } else {
                socket_fd = events[i].data.fd;
                printf("get event on socket fd == %d \n", socket_fd);
            }
        }
    }

    free(events);
    close(listen_fd);
}
```
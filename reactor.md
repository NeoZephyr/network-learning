## 单 reactor 反应堆模式
```c
#include <lib/acceptor.h>
#include "lib/common.h"
#include "lib/event_loop.h"
#include "lib/tcp_server.h"

char rot13_char(char c) {
    if ((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
        return c + 13;
    else if ((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
        return c - 13;
    else
        return c;
}

// 连接建立之后的 callback
int onConnectionCompleted(struct tcp_connection *tcpConnection) {
    printf("connection completed\n");
    return 0;
}

// 数据读到 buffer 之后的 callback
int onMessage(struct buffer *input, struct tcp_connection *tcpConnection) {
    printf("get message from tcp connection %s\n", tcpConnection->name);
    printf("%s", input->data);

    struct buffer *output = buffer_new();
    int size = buffer_readable_size(input);
    for (int i = 0; i < size; i++) {
        buffer_append_char(output, rot13_char(buffer_read_char(input)));
    }
    tcp_connection_send_buffer(tcpConnection, output);
    return 0;
}

// 数据通过 buffer 写完之后的 callback
int onWriteCompleted(struct tcp_connection *tcpConnection) {
    printf("write completed\n");
    return 0;
}

// 连接关闭之后的 callback
int onConnectionClosed(struct tcp_connection *tcpConnection) {
    printf("connection closed\n");
    return 0;
}

int main(int c, char **v) {
    // 主线程 event_loop，创建 reactor 对象
    struct event_loop *eventLoop = event_loop_init();

    // 初始化 acceptor
    struct acceptor *acceptor = acceptor_init(SERV_PORT);

    // 初始 tcp_server，可以指定线程数目
    // 如果线程是 0，就只有一个线程，既负责 acceptor，也负责 I/O
    struct TCPserver *tcpServer = tcp_server_init(
        eventLoop, acceptor,
        onConnectionCompleted, onMessage,
        onWriteCompleted, onConnectionClosed, 0);
    tcp_server_start(tcpServer);

    // 等待 acceptor 上有连接建立、新连接上有数据可读等
    event_loop_run(eventLoop);
}
```

## 主从 reactor 模式
主反应堆线程只负责分发 Acceptor 连接建立，已连接套接字上的 I/O 事件交给从反应堆负责分发。其中从反应堆的数量，可以根据 CPU 的核数来灵活设置。比如一个四核 CPU，我们可以设置从反应堆的数量为 4，这大大增强了 I/O 分发处理的效率。而且，同一个套接字事件分发只会出现在一个反应堆线程中，这会大大减少并发处理的锁开销

主反应堆线程一直在感知连接建立的事件，如果有连接成功建立，主反应堆线程通过 accept 方法获取已连接套接字，接下来会按照一定的算法选取一个从反应堆线程，并把已连接套接字加入到选择好的从反应堆线程中

```c
#include <lib/acceptor.h>
#include "lib/common.h"
#include "lib/event_loop.h"
#include "lib/tcp_server.h"

char rot13_char(char c) {
    if ((c >= 'a' && c <= 'm') || (c >= 'A' && c <= 'M'))
        return c + 13;
    else if ((c >= 'n' && c <= 'z') || (c >= 'N' && c <= 'Z'))
        return c - 13;
    else
        return c;
}

// 连接建立之后的 callback
int onConnectionCompleted(struct tcp_connection *tcpConnection) {
    printf("connection completed\n");
    return 0;
}

// 数据读到 buffer 之后的 callback
int onMessage(struct buffer *input, struct tcp_connection *tcpConnection) {
    printf("get message from tcp connection %s\n", tcpConnection->name);
    printf("%s", input->data);

    struct buffer *output = buffer_new();
    int size = buffer_readable_size(input);
    for (int i = 0; i < size; i++) {
        buffer_append_char(output, rot13_char(buffer_read_char(input)));
    }
    tcp_connection_send_buffer(tcpConnection, output);
    return 0;
}

// 数据通过 buffer 写完之后的 callback
int onWriteCompleted(struct tcp_connection *tcpConnection) {
    printf("write completed\n");
    return 0;
}

// 连接关闭之后的 callback
int onConnectionClosed(struct tcp_connection *tcpConnection) {
    printf("connection closed\n");
    return 0;
}

int main(int c, char **v) {
    // 主线程 event_loop
    struct event_loop *eventLoop = event_loop_init();

    // 初始化 acceptor
    struct acceptor *acceptor = acceptor_init(SERV_PORT);

    // 初始tcp_server，指定线程数目为 4
    // 一个 acceptor 线程，4 个 I/O 线程
    struct TCPserver *tcpServer = tcp_server_init(
        eventLoop, acceptor,
        onConnectionCompleted, onMessage,
        onWriteCompleted, onConnectionClosed, 4);
    tcp_server_start(tcpServer);

    event_loop_run(eventLoop);
}
```
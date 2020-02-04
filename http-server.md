## HTTP Server
### 设计
#### 反应堆
1. event_loop：无限循环着的事件分发器，一旦有事件发生，会回调预先定义好的回调函数，完成事件的处理
2. channel：表示各种注册到 event_loop 上的对象，例如注册到 event_loop 上的监听事件，注册到 event_loop 上的套接字读写事件等
3. acceptor：表示服务器端监听器，acceptor 对象最终会作为一个 channel 对象，注册到 event_loop 上，以便进行连接完成的事件分发和检测
4. event_dispatcher：对事件分发机制的一种抽象
5. channel_map：保存描述字到 channel 的映射，在事件发生时，根据事件类型对应的套接字快速找到 channel 对象里的事件处理函数

#### I/O 模型和多线程模型
thread_pool：维护线程列表，可以提供给主 reactor 线程使用，每次当有新的连接建立时，可以从 thread_pool 里获取一个线程，以便用它来完成对新连接套接字的 read/write 事件注册，将 I/O 线程和主 reactor 线程分离

#### Buffer 和数据读写
1. buffer：屏蔽对套接字进行的写和读的操作
2. tcp_connection：描述已建立的 TCP 连接。它的属性包括接收缓冲区、发送缓冲区、channel 对象等

### 实现
```c
#include <lib/acceptor.h>
#include <lib/http_server.h>
#include "lib/common.h"
#include "lib/event_loop.h"

// 数据读到 buffer 之后的 callback
int onRequest(struct http_request *httpRequest, struct http_response *httpResponse) {
    char *url = httpRequest->url;
    char *question = memmem(url, strlen(url), "?", 1);
    char *path = NULL;
    if (question != NULL) {
        path = malloc(question - url);
        strncpy(path, url, question - url);
    } else {
        path = malloc(strlen(url));
        strncpy(path, url, strlen(url));
    }

    if (strcmp(path, "/") == 0) {
        httpResponse->statusCode = OK;
        httpResponse->statusMessage = "OK";
        httpResponse->contentType = "text/html";
        httpResponse->body = "<html><head><title>This is network programming</title></head><body><h1>Hello, network programming</h1></body></html>";
    } else if (strcmp(path, "/network") == 0) {
        httpResponse->statusCode = OK;
        httpResponse->statusMessage = "OK";
        httpResponse->contentType = "text/plain";
        httpResponse->body = "hello, network programming";
    } else {
        httpResponse->statusCode = NotFound;
        httpResponse->statusMessage = "Not Found";
        httpResponse->keep_connected = 1;
    }

    return 0;
}

int main(int c, char **v) {
    // 主线程 event_loop
    struct event_loop *eventLoop = event_loop_init();
    struct http_server *httpServer = http_server_new(eventLoop, SERV_PORT, onRequest, 2);
    http_server_start(httpServer);
    event_loop_run(eventLoop);
}
```

event_dispatcher 让我们的线程挂起，等待事件的发生
event_dispatcher_data，它被定义为一个 void * 类型

event_loop 中如 owner_thread_id 是保留了每个 event loop 的线程 ID，mutex 和 con 是用来进行线程同步的

socketPair 是父线程用来通知子线程有新的事件需要处理。pending_head 和 pending_tail 是保留在子线程内的需要处理的新的事件


在 event_loop 不退出的情况下，一直在循环，循环体中调用了 dispatcher 对象的 dispatch 方法来等待事件的发生

event_dispacher 分析
为了实现不同的事件分发机制，这里把 poll、epoll 等抽象成了一个 event_dispatcher 结 构。event_dispatcher 的具体实现有 poll_dispatcher 和 epoll_dispatcher 两种，实现的 方法和性能篇21讲和22 讲类似，这里就不再赘述，你如果有兴趣的话，可以直接研读代 码。





channel 对象分析
channel 对象是用来和 event_dispather 进行交互的最主要的结构体，它抽象了事件分 发。一个 channel 对应一个描述字，描述字上可以有 READ 可读事件，也可以有 WRITE 可写事件。channel 对象绑定了事件处理函数 event_read_callback 和 event_write_callback。


channel_map 对象分析
event_dispatcher 在获得活动事件列表之后，需要通过文件描述字找到对应的 channel， 从而回调 channel 上的事件处理函数 event_read_callback 和 event_write_callback，为 此，设计了 channel_map 对象。


channel_map 对象是一个数组，数组的下标即为描述字，数组的元素为 channel 对象的地 址。
比如描述字 3 对应的 channel，就可以这样直接得到。

这样，当 event_dispatcher 需要回调 chanel 上的读、写函数时，调用 channel_event_activate 就可以，下面是 channel_event_activate 的实现，在找到了对应 的 channel 对象之后，根据事件类型，回调了读函数或者写函数。注意，这里使用了 EVENT_READ 和 EVENT_WRITE 来抽象了 poll 和 epoll 的所有读写事件类型。

增加、删除、修改 channel event
那么如何增加新的 channel event 事件呢?这几个函数是用来增加、删除和修改 channel
event 事件的。

前面三个函数提供了入口能力，而真正的实现则落在这三个函数上:

我们看一下其中的一个实现，event_loop_handle_pendign_add 在当前 event_loop 的 channel_map 里增加一个新的 key-value 对，key 是文件描述字，value 是 channel 对象 的地址。之后调用 event_dispatcher 对象的 add 方法增加 channel event 事件。注意这 个方法总在当前的 I/O 线程中执行。
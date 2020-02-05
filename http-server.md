## HTTP Server 设计
### 反应堆
#### event_loop
无限循环着的事件分发器，一旦有事件发生，会回调预先定义好的回调函数，完成事件的处理

#### channel
表示各种注册到 event_loop 上的对象，例如注册到 event_loop 上的监听事件，注册到 event_loop 上的套接字读写事件等。channel 对象绑定了事件处理函数 event_read_callback 和 event_write_callback

#### acceptor
表示服务器端监听器，acceptor 对象最终会作为一个 channel 对象，注册到 event_loop 上，以便进行连接完成的事件分发和检测

#### event_dispatcher
对事件分发机制的一种抽象，具体实现有 poll、epoll 等方式

#### channel_map
保存文件描述字到 channel 的映射，在事件发生时，根据事件类型对应的套接字快速找到 channel 对象里的事件处理函数，从而回调 channel 上的事件处理函数 event_read_callback 和 event_write_callback


### I/O 模型和多线程模型
#### 主 reactor
主 reactor 线程是一个 acceptor 线程，这个线程会以 event_loop 形式阻塞在 event_dispatcher 的 dispatch 方法上，一旦有连接完成，就会创建出连接对象 tcp_connection，以及 channel 对象等

#### 从 reactor
子线程是一个 event_loop 线程，它阻塞在 dispatch 上，一旦有事件发生，它就会查找 channel_map，找到对应的处理函数并执行它。之后它就会增加、删除或修改 pending 事件，再次进入下一轮的 dispatch


### Buffer 和数据读写
1. buffer：屏蔽对套接字进行的写和读的操作
2. tcp_connection：描述已建立的 TCP 连接。它的属性包括接收缓冲区、发送缓冲区、channel 对象等

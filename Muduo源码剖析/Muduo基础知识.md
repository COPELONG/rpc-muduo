# Muduo开发



## 网络编程基础知识

![image-20240514222753938](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240514222753938.png)

### 多reactor多线程

![image-20240515191131302](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240515191131302.png)

- 主线程中的 MainReactor 对象通过 select 监控连接建立事件，收到事件后通过 Acceptor 对象中的 accept  获取连接，将新的连接分配给某个子线程；
- 子线程中的 SubReactor 对象将 MainReactor 对象分配的连接加入 select 继续进行监听，并创建一个 Handler 用于处理连接的响应事件。
- 如果有新的事件发生时，SubReactor 对象会调用当前连接对应的 Handler 对象来进行响应。
- Handler 对象通过 read -> 业务处理 -> send 的流程来完成完整的业务流程。



**多 Reactor 多线程的方案虽然看起来复杂的，但是实际实现时比单 Reactor 多线程的方案要简单的多，原因如下：**

- 主线程和子线程分工明确，主线程只负责接收新连接，子线程负责完成后续的业务处理。
- 主线程和子线程的交互很简单，主线程只需要把新连接传给子线程，子线程无须返回数据，直接就可以在子线程将处理结果发送给客户端。

**在多Reactor模型中，多个子Reactor并行运行，每个子Reactor通过其`EventLoop`实例管理多个文件描述符及其事件。具体来说：**

1. **主Reactor**：
   - 主Reactor运行在主线程中，负责监听和接受新的连接。
   - 当有新连接到达时，主Reactor接受连接并将其分发给子Reactor。
2. **子Reactor**：
   - 每个子Reactor在一个独立的工作线程中运行，负责处理分配给它的连接上的所有I/O事件。
   - 每个子Reactor的`EventLoop`实例可以注册和管理多个文件描述符，并监听这些文件描述符上的事件（如读、写、异常等）。



### Epoll Demo



```c++

// 设置文件描述符为非阻塞模式
void setNonBlocking(int fd) {
    int flags = fcntl(fd, F_GETFL, 0);
    fcntl(fd, F_SETFL, flags | O_NONBLOCK);
}

int main() {
    int listen_fd, conn_fd, epoll_fd;
    struct sockaddr_in server_addr, client_addr;
    socklen_t client_len = sizeof(client_addr);
    struct epoll_event ev, events[MAX_EVENTS];

    // 创建监听套接字
    listen_fd = socket(AF_INET, SOCK_STREAM, 0);
    if (listen_fd < 0) {
        perror("socket failed");
        exit(EXIT_FAILURE);
    }

    // 设置监听套接字为非阻塞
    setNonBlocking(listen_fd);

    // 设置服务器地址结构
    memset(&server_addr, 0, sizeof(server_addr));
    server_addr.sin_family = AF_INET;
    server_addr.sin_addr.s_addr = INADDR_ANY;
    server_addr.sin_port = htons(PORT);

    // 绑定套接字
    if (bind(listen_fd, (struct sockaddr *)&server_addr, sizeof(server_addr)) < 0) {
        perror("bind failed");
        close(listen_fd);
        exit(EXIT_FAILURE);
    }

    // 监听套接字
    if (listen(listen_fd, SOMAXCONN) < 0) {
        perror("listen failed");
        close(listen_fd);
        exit(EXIT_FAILURE);
    }

    // 创建 epoll 实例
    epoll_fd = epoll_create1(0);
    if (epoll_fd < 0) {
        perror("epoll_create1 failed");
        close(listen_fd);
        exit(EXIT_FAILURE);
    }

    // 添加监听套接字到 epoll 实例中
    ev.events = EPOLLIN;
    ev.data.fd = listen_fd;
    if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, listen_fd, &ev) < 0) {
        perror("epoll_ctl failed");
        close(listen_fd);
        close(epoll_fd);
        exit(EXIT_FAILURE);
    }

    std::cout << "Server is listening on port " << PORT << std::endl;

    while (true) {
        int nfds = epoll_wait(epoll_fd, events, MAX_EVENTS, -1);
        if (nfds < 0) {
            perror("epoll_wait failed");
            close(listen_fd);
            close(epoll_fd);
            exit(EXIT_FAILURE);
        }

        for (int i = 0; i < nfds; ++i) {
            if (events[i].data.fd == listen_fd) {
                // 处理新连接
                conn_fd = accept(listen_fd, (struct sockaddr *)&client_addr, &client_len);
                if (conn_fd < 0) {
                    perror("accept failed");
                    continue;
                }

                // 设置新连接的套接字为非阻塞
                setNonBlocking(conn_fd);

                // 将新连接添加到 epoll 监视列表中
                ev.events = EPOLLIN | EPOLLET;
                ev.data.fd = conn_fd;
                if (epoll_ctl(epoll_fd, EPOLL_CTL_ADD, conn_fd, &ev) < 0) {
                    perror("epoll_ctl failed");
                    close(conn_fd);
                    continue;
                }

                std::cout << "Accepted new connection" << std::endl;
            } else {
                // 处理客户端数据
                int sock_fd = events[i].data.fd;
                char buffer[1024];
                int bytes_read = read(sock_fd, buffer, sizeof(buffer));

                if (bytes_read <= 0) {
                    // 关闭连接
                    close(sock_fd);
                    std::cout << "Closed connection" << std::endl;
                } else {
                    // 回显数据
                    write(sock_fd, buffer, bytes_read);
                }
            }
        }
    }

    // 清理资源
    close(listen_fd);
    close(epoll_fd);
    return 0;
}

```

![image-20240515213141605](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240515213141605.png)

#### epoll_wait()

epoll 使用**事件驱动**的机制，内核里**维护了一个链表来记录就绪事件**，当某个 socket 有事件发生时，通过**回调函数**内核会将其加入到这个就绪事件列表中，当用户调用 `epoll_wait()` 函数时，只会返回有事件发生的文件描述符的个数

```c++
int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);
```

- `epfd`：由 `epoll_create` 或 `epoll_create1` 创建的 epoll 实例的文件描述符。
- `events`：用于存储返回的事件的数组。
- `maxevents`：`events` 数组的大小。
- `timeout`：等待事件发生的超时时间，单位为毫秒。如果 `timeout` 为 -1，表示无限期地等待；如果为 0，表示立即返回（非阻塞）。

#### read() & write()

```c++
read(file, tmp_buf, len);
//read(socket, tmp_buf, len); 磁盘文件换成网卡
write(socket, tmp_buf, len);
```

![image-20240515214216215](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240515214216215.png)

首先，期间共**发生了 4 次用户态与内核态的上下文切换**，因为发生了两次系统调用，一次是  `read()` ，一次是 `write()`，每次系统调用都得先从用户态切换到内核态，等内核完成任务后，再从内核态切换回用户态。

其次，还**发生了 4 次数据拷贝**，其中两次是 DMA 的拷贝，另外两次则是通过 CPU 拷贝的，下面说一下这个过程：

- *第一次拷贝*，把磁盘上的数据拷贝到操作系统内核的缓冲区里，这个拷贝的过程是通过 DMA 搬运的。
- *第二次拷贝*，把内核缓冲区的数据拷贝到用户的缓冲区里，于是我们应用程序就可以使用这部分数据了，这个拷贝到过程是由 CPU 完成的。
- *第三次拷贝*，把刚才拷贝到用户的缓冲区里的数据，再拷贝到内核的 socket 的缓冲区里，这个过程依然还是由 CPU 搬运的。
- *第四次拷贝*，把内核的 socket 缓冲区里的数据，拷贝到网卡的缓冲区里，这个过程又是由 DMA 搬运的。

为了优化减少拷贝次数和上下文切换次数，可以采用零拷贝技术：

1.mmap() + write() :三次数据拷贝 + 4次上下文切换

2.linux中的sendfile(): 二次数据拷贝 + 2次上下文切换

具体解析参见小林coding（https://xiaolincoding.com/os/8_network_system/zero_copy.html#%E4%BD%BF%E7%94%A8%E9%9B%B6%E6%8B%B7%E8%B4%9D%E6%8A%80%E6%9C%AF%E7%9A%84%E9%A1%B9%E7%9B%AE）










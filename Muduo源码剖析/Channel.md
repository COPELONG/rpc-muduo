# Channel类



`Channel`表示一个文件描述符(fd)及其感兴趣的事件和回调函数。在Muduo中，`Channel`不直接持有文件描述符的资源，而是管理文件描述符的事件处理。`Channel`的主要功能包括：

- **事件类型**：记录文件描述符感兴趣的事件类型（如读事件、写事件）和poller返回的具体发生的事件。
- **事件回调**：保存事件发生时的回调函数（如读回调、写回调）。
- **事件注册**：向`EventLoop`注册需要监听的事件类型。
- **事件处理**：当事件发生时，调用相应的回调函数。



### 组件间的关系和工作流程



在多Reactor多线程模型中：

- **`EventLoop`**：每个线程运行一个`EventLoop`实例，主Reactor和子Reactor都使用`EventLoop`来管理事件循环，维护事件循环，持有`Poller`对象，负责事件的调度和分发。。
- **`Channel`**：`Channel`对象表示文件描述符及其事件和回调，每个`EventLoop`持有多个`Channel`。
- **`Poller`**：每个`EventLoop`实例持有一个`Poller`实例，`Poller`负责监视文件描述符事件并返回就绪事件。





### 组件之间的联系

- **`EventLoop`**：
  - 每个线程运行一个`EventLoop`实例，主Reactor和子Reactor都使用`EventLoop`。
  - 主Reactor的`EventLoop`负责处理新连接的接受和分发。
  - 子Reactor的`EventLoop`负责处理已接受连接的读写事件。
- **`Channel`**：
  - `Channel`对象表示一个文件描述符及其感兴趣的事件和回调。
  - 每个`EventLoop`持有多个`Channel`，这些`Channel`对应该`EventLoop`管理的文件描述符。
- **`Poller`**：
  - `Poller`是`EventLoop`用来监视文件描述符事件的组件。
  - 每个`EventLoop`实例持有一个`Poller`实例，`Poller`使用系统调用（如`epoll`）来等待事件发生。





1. **主Reactor接收新连接**：
   - 主Reactor通过`accept`系统调用接受新连接，并将新连接的fd创建相应的`Channel`对象。
   - 主Reactor将该`Channel`对象分配给某个子Reactor。
2. **子Reactor管理文件描述符**：
   - 子Reactor中的`EventLoop`负责管理分配给它的多个文件描述符。
   - 每个文件描述符由一个`Channel`对象管理，`Channel`对象记录该文件描述符感兴趣的事件类型（读、写、关闭等）以及相应的回调函数。
3. **事件循环**：
   - 每个子Reactor的`EventLoop`在事件循环中通过`Poller`（如`epoll`）等待事件的发生。
   - 当`Poller`返回就绪的文件描述符时，`EventLoop`遍历这些就绪的文件描述符，并调用相应的`Channel`的回调函数处理事件。





## 架构

```c++

/**
 * 理清楚  EventLoop、Channel、Poller之间的关系   《= Reactor模型上对应 Demultiplex
 * Channel 理解为通道，封装了sockfd和其感兴趣的event，如EPOLLIN、EPOLLOUT事件
 * 还绑定了poller返回的具体事件
 */ 
class Channel : noncopyable
{
public:
    using EventCallback = std::function<void()>;
    using ReadEventCallback = std::function<void(Timestamp)>;

    Channel(EventLoop *loop, int fd);
    ~Channel();

    // fd得到poller通知以后，处理事件的
    void handleEvent(Timestamp receiveTime);  

    // 设置回调函数对象
    void setReadCallback(ReadEventCallback cb) { readCallback_ = std::move(cb); }
    void setWriteCallback(EventCallback cb) { writeCallback_ = std::move(cb); }
    void setCloseCallback(EventCallback cb) { closeCallback_ = std::move(cb); }
    void setErrorCallback(EventCallback cb) { errorCallback_ = std::move(cb); }

    // 防止当channel被手动remove掉，channel还在执行回调操作
    void tie(const std::shared_ptr<void>&);

    int fd() const { return fd_; }
    int events() const { return events_; }
    int set_revents(int revt) { revents_ = revt; }

    // 设置fd相应的事件状态
    void enableReading() { events_ |= kReadEvent; update(); }
    void disableReading() { events_ &= ~kReadEvent; update(); }
    void enableWriting() { events_ |= kWriteEvent; update(); }
    void disableWriting() { events_ &= ~kWriteEvent; update(); }
    void disableAll() { events_ = kNoneEvent; update(); }

    // 返回fd当前的事件状态
    bool isNoneEvent() const { return events_ == kNoneEvent; }
    bool isWriting() const { return events_ & kWriteEvent; }
    bool isReading() const { return events_ & kReadEvent; }

    int index() { return index_; }
    void set_index(int idx) { index_ = idx; }

    // one loop per thread
    EventLoop* ownerLoop() { return loop_; }
    void remove();
private:

    void update();
    void handleEventWithGuard(Timestamp receiveTime);

    static const int kNoneEvent;
    static const int kReadEvent;
    static const int kWriteEvent;

    EventLoop *loop_; // 事件循环
    const int fd_;    // fd, Poller监听的对象
    int events_; // 注册fd感兴趣的事件
    int revents_; // poller返回的具体发生的事件
    int index_;

    std::weak_ptr<void> tie_;
    bool tied_;

    // 因为channel通道里面能够获知fd最终发生的具体的事件revents，所以它负责调用具体事件的回调操作
    ReadEventCallback readCallback_;
    EventCallback writeCallback_;
    EventCallback closeCallback_;
    EventCallback errorCallback_;
};

```





## 处理事件

```c++
// fd得到poller通知以后，处理事件的
void Channel::handleEvent(Timestamp receiveTime)
{
    if (tied_)
    {
        std::shared_ptr<void> guard = tie_.lock();
        if (guard)
        {
            handleEventWithGuard(receiveTime);
        }
    }
    else
    {
        handleEventWithGuard(receiveTime);
    }
}


// 根据poller通知的channel发生的具体事件， 由channel负责调用具体的回调操作
void Channel::handleEventWithGuard(Timestamp receiveTime)
{
    LOG_INFO("channel handleEvent revents:%d\n", revents_);

    if ((revents_ & EPOLLHUP) && !(revents_ & EPOLLIN))
    {
        if (closeCallback_)
        {
            closeCallback_();
        }
    }

    if (revents_ & EPOLLERR)
    {
        if (errorCallback_)
        {
            errorCallback_();
        }
    }

    if (revents_ & (EPOLLIN | EPOLLPRI))
    {
        if (readCallback_)
        {
            readCallback_(receiveTime);
        }
    }

    if (revents_ & EPOLLOUT)
    {
        if (writeCallback_)
        {
            writeCallback_();
        }
    }
}
```






































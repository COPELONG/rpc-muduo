# Poller



### 架构

```c++
// muduo库中多路事件分发器的核心IO复用模块
class Poller : noncopyable
{
public:
    using ChannelList = std::vector<Channel*>;

    Poller(EventLoop *loop);
    virtual ~Poller() = default;

    // 给所有IO复用保留统一的接口
    virtual Timestamp poll(int timeoutMs, ChannelList *activeChannels) = 0;
    virtual void updateChannel(Channel *channel) = 0;
    virtual void removeChannel(Channel *channel) = 0;
    
    // 判断参数channel是否在当前Poller当中
    bool hasChannel(Channel *channel) const;

    // EventLoop可以通过该接口获取默认的IO复用的具体实现
    
    //因为需要从poller父类中创建具体poll或者epoll，又因为这两者都是子类，为了符合规范，父类不能引用子类头文件。所以需要创建一个公共的.cc源文件去实现这个方法
    static Poller* newDefaultPoller(EventLoop *loop);
protected:
    // map的key：sockfd  value：sockfd所属的channel通道类型
    using ChannelMap = std::unordered_map<int, Channel*>;
    ChannelMap channels_;
private:
    EventLoop *ownerLoop_; // 定义Poller所属的事件循环EventLoop
}
```



## EPOLL

```c++
// channel未添加到poller中
const int kNew = -1;  // channel的成员index_ = -1
// channel已添加到poller中
const int kAdded = 1;
// channel从poller中删除
const int kDeleted = 2;
```

### 架构

```c++

/**
 * epoll的使用  
 * epoll_create
 * epoll_ctl   add/mod/del
 * epoll_wait
 */ 
class EPollPoller : public Poller
{
public:
    EPollPoller(EventLoop *loop);//epoll_create
    ~EPollPoller() override;

    // 重写基类Poller的抽象方法
    Timestamp poll(int timeoutMs, ChannelList *activeChannels) override;//epoll_wait
    
    void updateChannel(Channel *channel) override;//epoll_ctl
    void removeChannel(Channel *channel) override;//epoll_ctl
private:
    static const int kInitEventListSize = 16;

    // 填写活跃的连接
    void fillActiveChannels(int numEvents, ChannelList *activeChannels) const;
    
    // 更新channel通道 ==  epoll_ctl
    void update(int operation, Channel *channel);

    using EventList = std::vector<epoll_event>;

    int epollfd_;
    EventList events_;
};
```

### 删除channel

```c++
// 从poller中删除channel
void EPollPoller::removeChannel(Channel *channel)  //epoll树不监听这个fd了
{
    int fd = channel->fd();
    channels_.erase(fd);

    LOG_INFO("func=%s => fd=%d\n", __FUNCTION__, fd);
    
    int index = channel->index();
    if (index == kAdded)
    {
        update(EPOLL_CTL_DEL, channel);
    }
    channel->set_index(kNew);
}
```

### 更新/添加channel

Poller 管理 channel，channel管理fd

可以对poller 加入，修改，删除channel。

```c++
void EPollPoller::updateChannel(Channel *channel)
{
    const int index = channel->index();
    LOG_INFO("func=%s => fd=%d events=%d index=%d \n", __FUNCTION__, channel->fd(), channel->events(), index);

    if (index == kNew || index == kDeleted)
    {
        if (index == kNew)
        {
            int fd = channel->fd();
            channels_[fd] = channel;
        }

        channel->set_index(kAdded);
        update(EPOLL_CTL_ADD, channel);
    }
    else  // channel已经在poller上注册过了
    {
        int fd = channel->fd();
        if (channel->isNoneEvent())//只监听有事件的channel，若无事件，只在底层不监听channel，但是仍然管理channel。
        {
            update(EPOLL_CTL_DEL, channel);
            channel->set_index(kDeleted);
        }
        else
        {
            update(EPOLL_CTL_MOD, channel);
        }
    }
}
```

![image-20240519121249661](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240519121249661.png)

### 底层更新fd

```c++
// 更新channel通道 epoll_ctl add/mod/del
void EPollPoller::update(int operation, Channel *channel)
{
    epoll_event event;
    bzero(&event, sizeof event);
    
    int fd = channel->fd();

    event.events = channel->events();
    event.data.fd = fd; 
    event.data.ptr = channel; //指针指向channel，绑定了channel
    
    if (::epoll_ctl(epollfd_, operation, fd, &event) < 0)
    {
        if (operation == EPOLL_CTL_DEL)
        {
            LOG_ERROR("epoll_ctl del error:%d\n", errno);
        }
        else
        {
            LOG_FATAL("epoll_ctl add/mod error:%d\n", errno);
        }
    }
}
```



### 监听fd: epoll_wait

poller管理监听fd，所以当fd有事件时，需要返回给eventloop让其处理事件。

```c++
Timestamp EPollPoller::poll(int timeoutMs, ChannelList *activeChannels)
{
    // 实际上应该用LOG_DEBUG输出日志更为合理
    LOG_INFO("func=%s => fd total count:%lu \n", __FUNCTION__, channels_.size());

    int numEvents = ::epoll_wait(epollfd_, &*events_.begin(), static_cast<int>(events_.size()), timeoutMs);
    int saveErrno = errno;
    Timestamp now(Timestamp::now());

    if (numEvents > 0)
    {
        LOG_INFO("%d events happened \n", numEvents);
        fillActiveChannels(numEvents, activeChannels);//返回给loop
        if (numEvents == events_.size())
        {
            events_.resize(events_.size() * 2);
        }
    }
    else if (numEvents == 0)
    {
        LOG_DEBUG("%s timeout! \n", __FUNCTION__);
    }
    else
    {
        if (saveErrno != EINTR)
        {
            errno = saveErrno;
            LOG_ERROR("EPollPoller::poll() err!");
        }
    }
    return now;
}
```



```c++
// 填写活跃的连接
void EPollPoller::fillActiveChannels(int numEvents, ChannelList *activeChannels) const
{
    for (int i=0; i < numEvents; ++i)
    {
        Channel *channel = static_cast<Channel*>(events_[i].data.ptr);
        channel->set_revents(events_[i].events);
        activeChannels->push_back(channel); // EventLoop就拿到了它的poller给它返回的所有发生事件的channel列表了
    }
}
```

1. `__thread`: 这是一个线程局部存储（TLS）关键字，用于声明线程局部变量。线程局部变量是每个线程独有的，每个线程都有一份自己的变量副本，彼此之间不会相互影响。在多线程编程中，使用线程局部变量可以避免竞态条件和数据共享问题。








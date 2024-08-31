# noncopyable类

```c++
/**
 * noncopyable被继承以后，派生类对象可以正常的构造和析构，但是派生类对象
 * 无法进行拷贝构造和赋值操作
 */ 
class noncopyable
{
public:
    noncopyable(const noncopyable&) = delete;
    noncopyable& operator=(const noncopyable&) = delete;
protected:
    noncopyable() = default;
    ~noncopyable() = default;
};

class TcpServer : noncopyable
```

在C++中，当一个类的拷贝构造函数被删除或私有化时，派生类也不能使用**默认**的拷贝构造函数。其原因在于派生类的拷贝构造函数会隐式调用基类的拷贝构造函数。如果基类的拷贝构造函数被删除或私有化，派生类无法访问这个构造函数，从而导致编译错误。

### 解决方法

如果确实需要拷贝派生类对象，可以在派生类中显式定义拷贝构造函数，并手动实现需要的拷贝逻辑，而不是依赖于基类的拷贝构造函数。例如：

```c++
class Derived : public NonCopyable {
public:
    Derived() = default;

    // 显式定义拷贝构造函数
    Derived(const Derived& other) {
        // 实现派生类的拷贝逻辑
        // 注意：基类部分不能被拷贝，因为基类是NonCopyable
    }
};
```



# Logger日志类



```c++
// 定义日志的级别  INFO  ERROR  FATAL  DEBUG 
enum LogLevel
{
    INFO,  // 普通信息
    ERROR, // 错误信息
    FATAL, // core信息
    DEBUG, // 调试信息
};

// 输出一个日志单例类
class Logger : noncopyable
{
public:
    // 获取日志唯一的实例对象
    static Logger& instance();
    // 设置日志级别
    void setLogLevel(int level);
    // 写日志
    void log(std::string msg);
private:
    int logLevel_;
};
```



```c++
// 写日志  [级别信息] time : msg
void Logger::log(std::string msg)
{
    switch (logLevel_)
    {
    case INFO:
        std::cout << "[INFO]";
        break;
    case ERROR:
        std::cout << "[ERROR]";
        break;
    case FATAL:
        std::cout << "[FATAL]";
        break;
    case DEBUG:
        std::cout << "[DEBUG]";
        break;
    default:
        break;
    }

    // 打印时间和msg
    std::cout << Timestamp::now().toString() << " : " << msg << std::endl;
```

```c++
// LOG_INFO("%s %d", arg1, arg2)
#define LOG_INFO(logmsgFormat, ...) \
    do \
    { \
        Logger &logger = Logger::instance(); \
        logger.setLogLevel(INFO); \
        char buf[1024] = {0}; \
        snprintf(buf, 1024, logmsgFormat, ##__VA_ARGS__); \
        logger.log(buf); \
    } while(0) 
```



# TimeStamp类



```c++
// 时间类
class Timestamp
{
public:
    Timestamp();
    explicit Timestamp(int64_t microSecondsSinceEpoch);
    static Timestamp now();
    std::string toString() const;
private:
    int64_t microSecondsSinceEpoch_;
};


Timestamp Timestamp::now()
{
    return Timestamp(time(NULL));
}

std::string Timestamp::toString() const
{
    char buf[128] = {0};
    tm *tm_time = localtime(&microSecondsSinceEpoch_);
    snprintf(buf, 128, "%4d/%02d/%02d %02d:%02d:%02d", 
        tm_time->tm_year + 1900,
        tm_time->tm_mon + 1,
        tm_time->tm_mday,
        tm_time->tm_hour,
        tm_time->tm_min,
        tm_time->tm_sec);
    return buf;
}
```





# InetAddress 类



```c++
// 封装socket地址类型
class InetAddress
{
public:
    explicit InetAddress(uint16_t port = 0, std::string ip = "127.0.0.1");
    explicit InetAddress(const sockaddr_in &addr)
        : addr_(addr)
    {}

    std::string toIp() const;
    std::string toIpPort() const;
    uint16_t toPort() const;

    const sockaddr_in* getSockAddr() const {return &addr_;}
    void setSockAddr(const sockaddr_in &addr) { addr_ = addr; }
private:
    sockaddr_in addr_;
};
```

```c++
InetAddress::InetAddress(uint16_t port, std::string ip)
{
    bzero(&addr_, sizeof addr_);
    addr_.sin_family = AF_INET;
    addr_.sin_port = htons(port);
    addr_.sin_addr.s_addr = inet_addr(ip.c_str());
}

std::string InetAddress::toIpPort() const
{
    // ip:port
    char buf[64] = {0};
    ::inet_ntop(AF_INET, &addr_.sin_addr, buf, sizeof buf);
    size_t end = strlen(buf);
    uint16_t port = ntohs(addr_.sin_port);
    sprintf(buf+end, ":%u", port);
    return buf;
}
```


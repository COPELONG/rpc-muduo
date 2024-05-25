

### 分布式：

不同模块部署得到不同服务器上，组合成一个系统提供服务

![image-20240426150222600](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240426150222600.png)



### RPC：

![image-20240426150318296](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240426150318296.png)

黄色部分：设计rpc方法参数的打包和解析，也就是数据的序列化和反序列化，使用Protobuf。
绿色部分：网络部分，包括寻找rpc服务主机，发起rpc调用请求和响应rpc调用结果，使用muduo网络库和zookeeper服务配置中心（专门做服务发现）。
mprpc框架主要包含以上两个部分的内容。



```c++
syntax = "proto3";  //声明了protobuf的版本

package lddxfy; //声明了代码所在包

//定义下面的选项，表示生成service服务类和rpc方法的描述，默认不生成
option cc_generic_services = true;



message GetUserListRequest
{
    uint32 userid = 1;
}

message GetUserListResponse
{
    uint32 userid = 1;
    repeated User user = 2;
}

service UserServiceRPC
{
    rpc Login (LoginRequest) returns (LoginResponse);
    rpc GetUserList (GetUserListRequest) returns (GetUserListResponse);
}
```

![image-20240426180954800](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240426180954800.png)

![image-20240426182516079](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240426182516079.png)



### 读取配置项

```
mprpcserverip = 127.0.0.1
mprpcserverport = 8000
zookeeperip = 127.0.0.1
zookeeperport = 2181
```

```c++
#pragma once
#include <iostream>
#include <unordered_map>
#include <string>
class MprpcConfig
{
public:
    void LoadConfigFile(const char * config_file);

    std::string Load(const std::string &key);
private:
    std::unordered_map<std::string,std::string> map;
    void Trim(std::string &src_buf);
};
```



## 服务端

### muduo

#### 简单启动网络库

设置TCPserver

设置连接回调和消息读写回调

![image-20240429155857185](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240429155857185.png)



### 发布方法 (注册服务方法)

![image-20240429163603712](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240429163603712.png)

![image-20240429175929411](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240429175929411.png)

```c++
void RpcProvider::NotifyService(google::protobuf::Service *service)
{

    ServiceInfo serviceinfo;
    // 获取了服务对象的描述信息
    const google::protobuf::ServiceDescriptor *servicedesc = service->GetDescriptor();
    // 获取服务对象的名称
    std::string service_name = servicedesc->name();
    serviceinfo.m_service = service;
    // 获取服务对象service的方法的数量
    int methodCnt = servicedesc->method_count();
    //std::cout << "service name:" << service_name << std::endl;
    LOG_INFO("service name: %s",service_name.c_str());
    for (int i = 0; i < methodCnt; i++)
    {
        const google::protobuf::MethodDescriptor *methoddesc = servicedesc->method(i);
        std::string method_name = methoddesc->name();
        //std::cout << "method name:" << method_name << std::endl;
        LOG_INFO("method name: %s",method_name.c_str());
        serviceinfo.ServiceMethodMap.insert({method_name, methoddesc});
    }
    ServiceMap.insert({service_name, serviceinfo});
}
```



### 反序列化

区分具体服务的具体方法，并且为了防止TCP粘包，需要先读取4个字节：表示3个变量所占用的长度。

接受远端传输的数据类型是下面的proto类型，并且被序列化了，服务端使用API拿到数据后需要进行反序列化。

```c++
syntax = "proto3";

package mprpc;

message RpcHeader
{
    bytes service_name = 1;
    bytes method_name = 2;
    uint32 args_size = 3;
}
```

```c++
// 有消息传递设置回调  远程发起一个rpc请求，框架负责对其进行解析
// RpcProvider和RpcConsumer协商好之间通信用的protobuf数据类型
// service_name method_name args_size避免连包问题    定义proto的message类型，进行数据头的序列化和反序列化

// args_size： 表示参数字符串的长度

// service_name  +  method_name +  args_size相当于header_str

// header_size(4个字节) + header_str + args_str


void RpcProvider::onMessage(const muduo::net::TcpConnectionPtr &conn, muduo::net::Buffer *buf, muduo::Timestamp)
{
    // 网络上接收的远程rpc调用请求的字符流    Login args
    std::string recvbuf = buf->retrieveAllAsString();

    // 从字符流中读取前4个字节的内容
    uint32_t header_size = 0;
    //recvbuf.copy((char*)&header_size,4,0);
    
    // 使用std::memcpy从recvbuf中拷贝前4个字节的内容到header_size中
    std::memcpy(&header_size, recvbuf.data(), sizeof(uint32_t));

    // 根据header_size读取数据头的原始字符流，反序列化数据，得到rpc请求的详细信息
    std::string rpc_header_str = recvbuf.substr(4, header_size);
    mprpc::RpcHeader rpcHeader;
    std::string service_name;
    std::string method_name;
    uint32_t args_size;
    if (rpcHeader.ParseFromString(rpc_header_str))
    {
        service_name = rpcHeader.service_name();
        method_name = rpcHeader.method_name();
        args_size = rpcHeader.args_size();
    }
    else
    {
        // 数据头反序列化失败
        //std::cout << "rpc_header_str:" << rpc_header_str << " parse error!" << std::endl;
        LOG_ERROR("rpc_header_str: %s,parse error!",rpc_header_str.c_str());
        return;
    }
        // 获取rpc方法参数的字符流数据
    std::string args_str = recvbuf.substr(4 + header_size, args_size);
    
    
    
    
    // 获取service对象和method对象
    
    auto it = ServiceMap.find(service_name);
    if (it == ServiceMap.end())
    {
        std::cout << service_name << " is not exist!" << std::endl;
        return;
    }

    auto mit = it->second.ServiceMethodMap.find(method_name);
    if (mit == it->second.ServiceMethodMap.end())
    {
        //std::cout << service_name << ":" << method_name << " is not exist!" << std::endl;
        LOG_ERROR("%s : %s is not exist",service_name.c_str(),method_name.c_str());
        return;
    }
    // 获取service对象  new UserService
    google::protobuf::Service *service = it->second.m_service;
    // 获取method对象  Login
    const google::protobuf::MethodDescriptor *mehtond = mit->second;
    
    
    
    
      // 生成rpc方法调用的请求request和响应response参数
    google::protobuf::Message *request = service->GetRequestPrototype(mehtond).New();
    
        if (!request->ParseFromString(args_str))
    {
        std::cout << "request parse error, content:" << args_str << std::endl;
        LOG_ERROR("request parse error, content: %s",args_str.c_str());
        return;
    }
    
     google::protobuf::Message *response = service->GetResponsePrototype(mehtond).New();
    
      // 给下面的method方法的调用，绑定一个Closure的回调函数
    google::protobuf::Closure *done = google::protobuf::NewCallback<RpcProvider,
                                                                    const muduo::net::TcpConnectionPtr &,
                                                                    google::protobuf::Message *>(this, &RpcProvider::SendRpcResponse, conn, response);

    
    
        // 在框架上根据远端rpc请求，调用当前rpc节点上发布的方法
    // new UserService().Login(controller, request, response, done)
    service->CallMethod(mehtond, nullptr, request, response, done);
}
```



### 序列化



```c++
// Closure的回调操作，用于序列化rpc的响应和网络发送
void RpcProvider::SendRpcResponse(const muduo::net::TcpConnectionPtr &conn, google::protobuf::Message *response)
{
    std::string response_str;
    if(response->SerializeToString(&response_str))
    {
        // 序列化成功后，通过网络把rpc方法执行的结果发送会rpc的调用方
        conn->send(response_str);
    }
    else
    {
        //std::cout << "serialize response_str error!" << std::endl;
        LOG_ERROR("serialize response_str error!");
    }
    conn->shutdown(); // 模拟http的短链接服务，由rpcprovider主动断开连接
}
```



## 客户端

![image-20240430190946267](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240430190946267.png)

![image-20240430184650219](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240430184650219.png)



#### 序列化并发送数据

```c++
   void MprpcChannel::CallMethod(const google::protobuf::MethodDescriptor* method,
                          google::protobuf::RpcController* controller, const google::protobuf::Message* request,
                          google::protobuf::Message* response, google::protobuf::Closure* done)
{
const google::protobuf::ServiceDescriptor* sd = method->service();
    std::string service_name = sd->name();
    std::string method_name = method->name();

    // 获取参数的序列化字符串长度 args_size
    uint32_t args_size = 0;
    std::string args_str;
    if(request->SerializeToString(&args_str))
    {
        args_size = args_str.size();
    }
    else
    {
        controller->SetFailed("serialize request error!");
        return;
    }
    // 定义rpc的请求header
    mprpc::RpcHeader rpcHeader;
    rpcHeader.set_service_name(service_name);
    rpcHeader.set_method_name(method_name);
    rpcHeader.set_args_size(args_size);
    uint32_t header_size = 0;
    std::string rpc_header_str;
    if (rpcHeader.SerializeToString(&rpc_header_str))
    {
        header_size = rpc_header_str.size();
    }
    else
    {
        controller->SetFailed("serialize rpc header error!");
        return;
    }

    // 组织待发送的rpc请求的字符串
    std::string send_str;
    send_str.insert(0,std::string((char *)&header_size,4));
      //std::memcpy(&send_str[0], &header_size, sizeof(uint32_t));

    send_str += rpc_header_str;// rpcheader
    send_str += args_str;// args
```

#### TCP建立连接

```c++
    // 读取配置文件rpcserver的信息
    // std::string ip = MprpcApplication::GetInstance().GetConfig().Load("mprpcserverip");
    // uint16_t port = atoi(MprpcApplication::GetInstance().GetConfig().Load("mprpcserverport").c_str());
int clientfd = socket(AF_INET, SOCK_STREAM, 0);
    struct sockaddr_in server_addr;
    server_addr.sin_family = AF_INET;
    server_addr.sin_port = htons(port);
    server_addr.sin_addr.s_addr = inet_addr(ip.c_str());

    // 连接rpc服务节点
    if(-1 == connect(clientfd,(struct sockaddr*)&server_addr,sizeof(server_addr)))
    {
        close(clientfd);
        char errtest[1024];
        sprintf(errtest,"connect rpcserver error! errno: %d",errno);
        controller->SetFailed(errtest);
        return;
    }

     // 发送rpc请求
    if(-1 == send(clientfd,send_str.c_str(),send_str.size(),0))
    {
        close(clientfd);
        char errtest[1024];
        sprintf(errtest,"send message error! errno: %d",errno);
        controller->SetFailed(errtest);
        return;
    }

    // 接收rpc请求的响应值
    char recv_buf[1024] = {0};
    int recv_size = 0;
    if(-1 == (recv_size = recv(clientfd,recv_buf,1024,0)))
    {
        close(clientfd);
        char errtest[1024];
        sprintf(errtest,"recv message error! errno: %d",errno);
        controller->SetFailed(errtest);
        return;
    }

    // 反序列化rpc调用的响应数据
    if(!response->ParseFromArray(recv_buf,recv_size)){
        
    }
 close(clientfd);
```





## 控制 Controller

当序列化或者创建连接失败时，显示信息。

RPC调用的过程中成功与否。

![image-20240502180410673](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240502180410673.png)

```c++
#pragma once
#include <google/protobuf/message.h>
#include <google/protobuf/descriptor.h>
#include <google/protobuf/service.h>
class MprpcController : public google::protobuf::RpcController
{
public:
    MprpcController();
    void Reset();
    bool Failed() const;
    std::string ErrorText() const;
    void SetFailed(const std::string& reason);

    // 目前未实现具体的功能
    void StartCancel();
    bool IsCanceled() const;
    void NotifyOnCancel(google::protobuf::Closure* callback);
private:
    bool m_failed;         // RPC方法执行过程中的状态
    std::string m_errText; // RPC方法执行过程中的错误信息
};
```



## 日志模块



![image-20240506091956211](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240506091956211.png)

```c++
class Logger
{
public:
    //获取日志的单例
    static Logger& GetInstance();

    //设置日志级别
    void SetLogLevel(LogLevel level);

    //写日志
    void Log(std::string msg);


private:
    int m_loglevel;// 记录日志级别
    LockQueue<std::string> m_lckQue;// 日志缓冲队列

    Logger();
    Logger(const Logger&) = delete;
    Logger(Logger&&) = delete;
};

// 定义宏 LOG_INFO("xxx %d %s", 20, "xxxx")
#define LOG_INFO(logmsgformat,...)\
    do \
    { \
        Logger &logger = Logger::GetInstance();\
        logger.SetLogLevel(INFO); \
        char c[1024] = {0}; \
        snprintf(c, 1024, logmsgformat, ##__VA_ARGS__); \
        logger.Log(c); \
    }while(0)\
```

![image-20240506100416412](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240506100416412.png)



## Zookeeper

服务配置中心功能：用来记录分布式节点上所有发布RPC服务的主机IP和PORT。

- 假设有多台机器（B、C、D、E）作为服务提供者，它们会连接到运行 Zookeeper 服务端的一组机器（F端）。服务提供者会将自己的服务信息注册到 Zookeeper 中，包括服务名称、IP地址、端口等信息。

  然后，服务消费者（A端）会连接到同样运行 Zookeeper 服务端的一组机器（F端），通过 Zookeeper 客户端获取注册在 Zookeeper 中的服务信息。服务消费者可以根据获取的服务信息，选择合适的服务提供者进行服务调用。

  这种架构的好处是，Zookeeper 服务端被专门部署在一组机器上（F端），它们构成了一个 Zookeeper 集群，保证了 Zookeeper 的高可用性和可靠性。而服务提供者和服务消费者都作为 Zookeeper 的客户端连接到 Zookeeper 集群，通过 Zookeeper 来实现服务注册、发现和调用，使得整个系统更加稳定和可靠。

### Znode

![image-20240506171811091](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240506171811091.png)

![image-20240506173216275](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240506173216275.png)

划分Znode路径 ：/ UserServiceRpc / Login  存储的是IP+PORT

![image-20240506174208710](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240506174208710.png)



### watcher机制

客户端监听节点变化，ZK用来通知变化。

![image-20240506181409719](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240506181409719.png)



![image-20240506182304985](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240506182304985.png)

### 服务注册ZK

![image-20240506191856344](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240506191856344.png)

![image-20240506211628812](https://my-figures.oss-cn-beijing.aliyuncs.com/Figures/image-20240506211628812.png)




















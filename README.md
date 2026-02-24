# MPRPC
c++实现的基于muduo网络库与protobuf编写的分布式网络通信框架
一、技术栈
Linux 网络编程，基于Muduo 网络库的Reactor高并发模型开发，Protobuf 数据序列化与反序列化，RPC 原理实现（Stub、Service、Channel、Controller 机制），Zookeeper 服务注册与发现机制，基于 Zookeeper 的分布式节点会话管理（Session），多线程编程与线程安全设计，异步日志系统，CMake 构建系统，Git 版本管理
二、编译方式：
运行脚本autobuild.sh:
cd build
rm -rf *
cmake ..
make
三、项目相关细节
1.在proto文件中定义message类（google protobuf提供）的请求体与响应体还有rpc的service服务类，借助protoc工具直接生成对应的pb.cc,pb.h代码，也就是对应的c++请求消息类和响应消息类还有服务类（这些类都公共继承了对应的message类和service类，可使用其方法）。通过类对象的的serializetostring方法序列化对象为字符串，parsefromstring反序列化（序列化和反序列化都是在rpc框架中实现，业务代码无需关注）。
2.option cc_generic_services=true表示生成service服务类与rpc方法描述。默认情况下不生成。
3.proto生成对应的stub类是提供给调用方使用的。生成的userService_stub类的构造函数需要传入一个Rpc Channel* channel指针。该类方法的实现都直接依赖于底层的channel->method方法（我们要重写这个method方法构造我们的请求4字节header_size+servicename+methodname+args_size+args并且通过socket编程connect(),send()完成网络发送与接收recv()），负责rpc请求参数序列化以及网络发送还有zk服务查询以及等待响应反序列化response。构造stub(new mprpcchannel())对象以及request后，stub.方法(xx,&request,&response,xx)即可发送rpc请求。
4.发布具体服务到rpc节点：服务描述信息（服务类继承于服务rpc类继承于protobuf中的service基类，通过service基类指针调用getdescriptor）如服务名字dscriptor->name、服务类对象方法数量dscriptor->method_count以及第i个方法的描述pmethoddescriptor=dscriptor->method(i)。
5.定义rpc通信的message类型，包含header:service_name,method_nam。data:args_size（args即传入方法的参数，默认是一个请求包的数据部分，但出于防止粘包，传输一下size更保险）。
6.被调用方最终方法的实现依赖于google::protobuf::Service::CallMethod。service->CallMethod(method, controller, request, response, done);其中controller是RpcController类指针，RpcController类是抽象类，纯虚函数需要重写，作用是描述一次rpc调用的状态。
done即closure参数，作用是
7.日志缓冲队列的push操作是工作线程来实现的，而pop操作是写日志线程实现的。
8.zookeeper：分布式协调服务——服务注册中心。zookeeper 存储数据的方式是znode。zookeeper client常用命令：ls+目录(查看指定目录下的节点) delete+节点（删除指定目录下的节点）create+节点（创建指定目录下的节点）set（修改指定目录下的节点数据）get（获取指定目录下的节点描述信息，包括节点数据、节点数据长度、节点子节点个数、节点是临时节点还是永久节点）。zookeeper注册中心与注册的服务维持session会话，通过心跳机制确认节点活性，超出限制心跳次数的临时节点会被删除，而永久节点不会。watcher:zkserver监视znode（当前node的所有子node状态）改变并报告给zkclient。
9.zk的init函数只是创建了一个句柄m_handler，zkclient与zkserver的连接是异步的，需要等待global_watcher回调函数（等待连接状态）执行完毕sem_post()，这个过程通过信号量来控制。
10.为什么使用的是zookeeper_mt多线程版本，是因为zk的api客户端提供了三个线程，一个是API调用线程，一个是网络I/O线程（pthread_create poll），一个是watcher回调线程。
12.创建服务znode为永久节点，方法节点为临时节点。

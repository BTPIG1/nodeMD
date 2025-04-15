## 常见头文件
### unistd.h
定义： unistd.h（UNIX Standard Header）是POSIX标准定义的头文件，提供了对操作系统底层功能的访问接口，主要用于Unix/Linux系统编程。

常见功能：
- 文件操作：read(), write(), close(), lseek()
- 进程控制：fork(), exec(), getpid(), getppid()
- 系统信息：gethostname(), sysconf()
- 权限管理：access(), chmod()
- 管道和文件描述符：pipe(), dup(), dup2()
- 终端控制：isatty(), ttyname()

### sys/syscall.h
定义：sys/syscall.h（System Call Header）提供了直接调用操作系统底层系统调用的接口。它定义了系统调用号（如SYS_write、SYS_read）和syscall()函数，允许绕过C标准库（如glibc）直接发起系统调用。

特点：1. 直接与内核交互，可能更高效（但可能存在安全问题）2. 不同操作系统的系统调用不同，可移植性差。

### errno.h
定义：errno.h 是 C/C++ 标准库中用于错误处理的头文件，它定义了与系统错误相关的宏、变量和函数。

- 定义错误码宏：提供标准化的错误代码（如 EACCES、ENOENT、EINVAL 等），每个宏对应一个具体的错误类型。
- 声明全局变量 errno：errno 是一个线程局部的整型变量，用于存储最近一次函数调用产生的错误码。

## 下面是socket中常用头文件
### <sys/socket.h>：套接字核心API
````c++
int socket(int domain, int type, int protocol);  // 创建套接字
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen); // 绑定地址
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen); // 连接服务器
::setsockopt(sockfd_, IPPROTO_TCP, TCP_NODELAY, &optval, sizeof(optval));  // 设置套接字上的协议以及协议状态
struct sockaddr {                // 通用地址结构
    sa_family_t sa_family;       // 地址族（如AF_INET）
    char        sa_data[14];     // 地址数据
};
````

### <netinet/tcp.h>：TCP协议控制
````c++
#define TCP_NODELAY   1   // 禁用Nagle算法（立即发送小数据包）
#define TCP_KEEPIDLE  4   // 设置TCP保活探测起始时间（秒）
#define TCP_KEEPINTVL 5   // 保活探测间隔时间
#define TCP_KEEPCNT   6   // 保活探测次数
````

### <arpa/inet.h>：网络地址转换 p：ip字符串 n:二进制
````c++
int inet_pton(int af, const char *src, void *dst);  // 通用IP字符串转二进制（推荐）
const char* inet_ntop(int af, const void *src, char *dst, socklen_t size); // 二进制转字符串（推荐）
````

### <netinet/in.h>：IP协议族定义
````c++
struct sockaddr_in {             // IPv4地址结构
    sa_family_t    sin_family;   // 地址族（AF_INET）
    in_port_t      sin_port;     // 端口号（需用htons转换）
    struct in_addr sin_addr;     // IPv4地址
};

struct in6_addr { ... };         // IPv6地址结构
uint16_t htons(uint16_t hostshort);  // 主机字节序转网络字节序（端口）
uint32_t htonl(uint32_t hostlong);   // 主机字节序转网络字节序（IP地址）

struct sockaddr_in addr;
addr.sin_port = htons(8080);       // 设置端口为网络字节序
````

### <sys/types.h>：基础系统类型
````c++
typedef int pid_t;               // 进程ID类型
typedef unsigned int socklen_t;  // 套接字地址长度类型
typedef uint32_t in_addr_t;      // IPv4地址二进制类型
````

### 头文件协作使用场景
- 创建套接字
````c++
#include <sys/socket.h>   // socket()
#include <netinet/in.h>   // AF_INET、sockaddr_in
````

- 设置IP和端口
````c++
#include <netinet/in.h>   // htons(), INADDR_ANY
#include <arpa/inet.h>    // inet_pton()
````

- 调整TCP参数
````c++
#include <netinet/tcp.h>  // TCP_NODELAY
#include <sys/socket.h>   // setsockopt()
````



## evenfd

- select\poll\epoll都是io多路复用模型，都通过监听其注册文件描述符的状态来触发各种事件的回调，其中select通过最简单的循环遍历来查找状态发生变化的文件描述符，poll模型在select基础之上新增了一个无上限注册文件描述符的规则。
epoll基于红黑树加就绪队列的方法，能根据文件描述符快速找到对应的事件对象；并且不需要主动去遍历文件描述符集合，通过维护就绪队列，状态发生改变的文件描述符会被加入到队列中，只需要处理队列中响应的事件即可。

定义:eventfd 是 Linux 系统提供的一种轻量级进程间通信（IPC）机制，用于高效的事件通知。它通过一个文件描述符（file descriptor）实现，允许不同线程或进程之间通过读写操作来触发事件通知。

### 一、核心作用

- 事件通知:通过一个 eventfd 文件描述符，通知其他线程或进程“某个事件已发生”。
- 写操作：向 eventfd 写入一个整数值（通常为 1），触发事件。
- 读操作：读取 eventfd 的值，获取事件触发的次数（并自动重置计数器）。
- 计数器机制:eventfd 内部维护一个 64 位无符号整数计数器，支持累积事件次数。

写入时累加计数器，读取时返回当前值并重置为 0（或根据标志调整行为）。

### 二、典型使用场景

- 线程/进程间通知

多线程任务调度：主线程通过 eventfd 通知工作线程处理新任务。
多进程协作：父进程通知子进程执行特定操作。

- 与 I/O 多路复用结合将 eventfd 注册到 epoll、select 或 poll 中，实现异步事件驱动：

当 eventfd 可读时，表明有事件触发，程序可执行相应逻辑。
常用于高性能服务器框架（如 Nginx、Redis）的事件循环。
替代传统信号量

- eventfd 的计数器机制可以模拟轻量级信号量，用于资源计数或限制。

### 三、基本用法
````c++
1.创建
#include <sys/eventfd.h>

// 创建 eventfd，初始计数器值为 initval，flags 控制行为（如非阻塞）
int fd = eventfd(unsigned int initval, int flags); // flag可以指定非阻塞EFD_NONBLOCK和EFD_CLOEXEC(进程执行时关闭文件描述符)标志?

2.写入事件
uint64_t value = 1;
write(fd, &value, sizeof(value)); // 向 eventfd 写入一个值（触发事件）

3.读取事件
uint64_t count;
read(fd, &count, sizeof(count)); // 读取当前计数值（并重置计数器）

4.关闭
close(fd)
````

### 四、示例：结合 epoll 实现异步通知
````c++
#include <sys/eventfd.h>
#include <sys/epoll.h>
#include <unistd.h>
#include <iostream>

int main() {
    // 创建 eventfd 和 epoll 实例
    int efd = eventfd(0, EFD_NONBLOCK);
    int epoll_fd = epoll_create1(0);

    // 将 eventfd 注册到 epoll
    struct epoll_event ev;
    ev.events = EPOLLIN; // 监听可读事件
    ev.data.fd = efd;
    epoll_ctl(epoll_fd, EPOLL_CTL_ADD, efd, &ev); // EPOLL_CTL_ADD可以是增删改

    // 触发事件（例如在另一个线程中）
    uint64_t value = 1;
    write(efd, &value, sizeof(value));

    // 等待事件
    struct epoll_event events[10];
    int n = epoll_wait(epoll_fd, events, 10, -1); // 阻塞等待
    for (int i = 0; i < n; i++) {
        if (events[i].data.fd == efd) {
            uint64_t count;
            read(efd, &count, sizeof(count)); // 读取事件计数
            std::cout << "Event triggered! Count: " << count << std::endl;
        }
    }

    close(efd);
    close(epoll_fd);
    return 0;
}
````
### epoll_ctl
定义:epoll_ctl用于注册fd到epoll中,epoll维护并监听一个fd集合,通过epoll_ctl函数可以管理注册中的fd,操作包括增删改.

- 函数原型
```c++
#include <sys/epoll.h>

int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event);
```
参数：

epfd：由 epoll_create 创建的 epoll 实例的文件描述符。

op：操作类型，取值为以下常量之一：

1. EPOLL_CTL_ADD：添加 FD 到监视列表。
2. EPOLL_CTL_MOD：修改已注册 FD 的事件。
3. EPOLL_CTL_DEL：从监视列表中删除 FD。

fd：要操作的目标文件描述符（如 Socket、管道等）。

event：指向 epoll_event 结构体的指针，包括fd,fd感兴趣事件,ptr(与fd绑定channel对象的地址)


返回值：0：操作成功。-1：操作失败，需检查 errno（如 EBADF 表示无效的 FD）。
````c++
struct epoll_event {
    uint32_t     events;   // 监视的事件类型（如 EPOLLIN、EPOLLOUT）
    epoll_data_t data;     // 用户数据（可关联 FD、指针等）
};

typedef union epoll_data {
    void*     ptr;     // 自定义指针（用于高级场景）
    int       fd;      // 关联的 FD（常用）
    uint32_t  u32;
    uint64_t  u64;
} epoll_data_t;
````
- 常用的epoll_event中events类型:

| 事件          | 说明                                      |
|---------------|-------------------------------------------|
| EPOLLIN       | FD 可读（有数据到达）                    |
| EPOLLOUT      | FD 可写（发送缓冲区未满）                |
| EPOLLERR      | FD 发生错误（自动监听，无需显式设置）    |
| EPOLLHUP      | FD 被挂断（如对端关闭连接）              |
| EPOLLET       | 设置为边缘触发模式（默认是水平触发，内核缓存区数据必须一次性读完或写完才能再次触发事件）     |
| EPOLLONESHOT  | 事件触发后自动从监视列表移除（需重新注册）|


### epoll_wait
定义: epoll_wait是实现io多路复用epoll的核心函数之一,用于阻塞线程并等待与epoll_fd绑定的epoll_event数组上的事件发生.有三种情况:1.有注册的fd发生事件;2.出错;3.超时

触发读事件：1.监听套接字收到新连接 2.内核接受缓存区存在数据 3.非阻塞读轮询监听到可读事件 4.对端发送关闭连接FIN
触发写事件：1.内核发送缓存区有空间写 2.非阻塞写操作轮询检测到有空间（EPOLLOUT）

- 函数原型
````c++
#include <sys/epoll.h>

int epoll_wait(int epfd, struct epoll_event *events, 
               int maxevents, int timeout);
````
epfd：由 epoll_create 创建的 epoll 实例的文件描述符。

events：用户分配的数组，用于接收就绪事件。

maxevents：events 数组的最大容量（决定一次最多返回多少事件）。

timeout：超时时间（毫秒）：
-1：无限阻塞，直到有事件发生。
0：立即返回，不阻塞（仅检查当前就绪事件）。
\>0：最多阻塞指定毫秒数。

- 具体实现机制:epoll内核维护一个红黑树(保存所有注册的fd)以及一个就绪队列(保存已触发事件的fd)
1. 调用epoll_wait时,首先检查就绪队列是否为空?
- - 非空直接收集事件返回
- - 空则阻塞线程,等待事件发生
2. 某个fd发生事件时,内核会调用相关回调函数 ?????如何实现调用回调函数?由tcpconnection中定义的4个handle函数决定
3. 封装事件到epoll_event类型,并拷贝到提供的events数组中

## TCP服务器基本流程
````c++
#include <sys/socket.h>
#include <netinet/in.h>

int main() {
    // 1. 创建 TCP 套接字
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);
    
    // 2. 绑定地址和端口
    struct sockaddr_in addr = {};
    addr.sin_family = AF_INET;
    addr.sin_port = htons(8080);
    addr.sin_addr.s_addr = INADDR_ANY;
    bind(sockfd, (struct sockaddr*)&addr, sizeof(addr));
    
    // 3. 开始监听（关键步骤）
    if (listen(sockfd, 5) == -1) {  // 允许最多 5 个挂起连接
        perror("listen failed");
        return 1;
    }
    
    // 4. 接受客户端连接（循环处理）
    while (1) {
        struct sockaddr_in client_addr;
        socklen_t client_len = sizeof(client_addr);
        int client_fd = accept(sockfd, (struct sockaddr*)&client_addr, &client_len);
        // 处理 client_fd...
    }
    return 0;
}
````


## socket编程

### 总结
本质上所有对套接字的操作都可以看成是读写事件，采用不同io模型进行处理。

本地通过::socket创建的套接字通常用于监听外部新建连接，比如在我的项目中acceptor类，指定套接字地址族、传输方式以及协议类型创建了一个套接字，该套接字与主线程地址通过::bind绑定，随后根据该套接字创建channel对象并设置读回调函数，然后把channel对象注册到pollor监听器中，并监听读事件（也就是外部新建连接）。随后返回acceptor类中，开启套接字的监听状态。一旦外部有新连接请求，pollor就会监听到套接字对应的接受缓存区中存在数据触发读事件。在读事件中，调用::accept4系统指令获取新连接的套接字以及对端地址，选择一个子事件循环器，有了本地地址、对端地址、连接套接字、子事件循环器创建一个TCPconnection,后续该连接中发送的读写事件全由TCPconnection处理。

Tcpconnection是用来管理外部连接套接字上的读写事件，Tcpconnection管理一个socket与channel，并给channel设置处理不同事件的回调函数，利用socket来管理连接状态。

### ::socket 创建socket套接字，并返回一个sockfd
::socket 是用于 创建网络通信端点（套接字） 的系统调用函数，它是网络编程的核心基础，允许程序通过 TCP、UDP 等协议进行网络通信。
- 函数原型
````c++
#include <sys/socket.h>  // 头文件

int socket(int domain, int type, int protocol);
````
参数解析：
| 参数      | 作用                                   | 常见取值                                   |
|-----------|----------------------------------------|--------------------------------------------|
| domain    | 指定通信的 协议族（地址族）            | AF_INET（IPv4）、AF_INET6（IPv6）、AF_UNIX（本地域协议） |
| type      | 指定套接字的 类型（数据传输方式）      | SOCK_STREAM（流式，TCP）、SOCK_DGRAM（数据报，UDP）、SOCK_RAW（原始套接字） |
| protocol  | 指定具体的 协议（通常设为 0，表示根据前两个参数自动选择） | IPPROTO_TCP（TCP）、IPPROTO_UDP（UDP）、0（自动） |

- 作用

1. 通过::socket创建一个套接字，作为网络通信的端点，例如：服务端创建监听套接字；客户端创建连接套接字。
2. 通过参数指定通信协议（如 TCP/UDP）、地址格式（IPv4/IPv6）和数据传输方式（可靠传输或不可靠传输）。
3. ！！返回的文件描述符用于后续操作（如 bind、listen、connect、send、recv）。

- 用法

int sockfd = ::socket(AF_INET, SOCK_STREAM | SOCK_NONBLOCK | SOCK_CLOEXEC, IPPROTO_TCP);
创建了一个TCP连接IPv4

- Q1 ::socket 前的 :: 是什么作用？

:: 是 C++ 中的全局作用域解析符，确保调用的是 C 标准库的 socket 函数，而非可能存在的同名类或函数（尤其是在命名空间中使用时）。

### sockaddr_in 用于表示IPv4地址和端口的结构体
- 原型
````c++
struct sockaddr_in {
    sa_family_t    sin_family;  // 地址族：AF_INET（IPv4）
    in_port_t      sin_port;    // 端口号（16位，需用网络字节序）
    struct in_addr sin_addr;    // IPv4地址（32位，网络字节序）
    char           sin_zero[8]; // 填充字段（通常置0）
};

struct in_addr {
    in_addr_t s_addr;  // 32位 IPv4地址（网络字节序）
};
````
- 使用
````c++
struct sockaddr_in addr;

// 1. 初始化结构体
memset(&addr, 0, sizeof(addr));
addr.sin_family = AF_INET;  // IPv4

// 2. 设置端口（转换为网络字节序）
addr.sin_port = htons(8080);

// 3. 设置 IP 地址（字符串 → 二进制）
const char* ip_str = "192.168.1.1";
if (inet_pton(AF_INET, ip_str, &addr.sin_addr) != 1) {
    perror("Invalid IP address");
    exit(1);
}

// 4. 使用 addr（例如 bind() 或 connect()）
int sockfd = socket(AF_INET, SOCK_STREAM, 0);
bind(sockfd, (struct sockaddr*)&addr, sizeof(addr));
````
- 关键转换函数

 (1) 字符串 IP ↔ 二进制地址

| 函数         | 方向             | 说明                                                                                     |
|--------------|------------------|------------------------------------------------------------------------------------------|
| `inet_aton()` | 字符串 → 二进制  | 旧函数，支持点分十进制格式（如 "192.168.1.1"），但不可重入（非线程安全）。               |
| `inet_addr()` | 字符串 → 二进制  | 旧函数，返回 `in_addr_t`，但无法处理 "255.255.255.255"（返回 `INADDR_NONE`）。           |
| `inet_ntoa()` | 二进制 → 字符串  | 旧函数，不可重入（返回静态缓冲区，多线程不安全）。                                       |
| `inet_pton()` | 字符串 → 二进制  | 新函数（IPv4/IPv6 兼容），线程安全。                                                     |
| `inet_ntop()` | 二进制 → 字符串  | 新函数（IPv4/IPv6 兼容），线程安全。                                                     |

 (2) 字节序转换

| 函数    | 作用                              | 示例                                      |
|---------|-----------------------------------|-------------------------------------------|
| `htons()` | 主机字节序 → 网络字节序（端口）  | `addr.sin_port = htons(8080);`            |
| `htonl()` | 主机字节序 → 网络字节序（地址）  | `addr.sin_addr.s_addr = htonl(INADDR_ANY);` |
| `ntohs()` | 网络字节序 → 主机字节序（端口）  | `port = ntohs(addr.sin_port);`            |
| `ntohl()` | 网络字节序 → 主机字节序（地址）  | `ip = ntohl(addr.sin_addr.s_addr);`       | 


### ::bind() 与 std::bind()
定义：
::bind()：用于网络编程，将套接字绑定到地址和端口。`一般用于绑定监听套接字与本地localaddr`
std::bind()：用于函数式编程，生成可调用对象，适配参数或延迟调用。
- ::bind()原型
````c++
#include <sys/socket.h>
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
sockfd:由socket创建的套接字；addr：由sockaddr创建的结构体指针（包含地址和端口信息）；addrlen：结构体长度
````
- std::bind()原型
理解：可以绑定普通函数和具体的参数、也可以绑定成员函数与实例，最终返回一个函数对象。因此可以实现多元函数绑定后返回一个某元对象。
````c++
#include <functional>
template <class Fn, class... Args>
/* unspecified */ bind(Fn&& fn, Args&&... args);

#include <functional>
#include <iostream>

class Calculator {
public:
    int multiply(int a, int b,int c,int d) {
    }
};

int main() {
    Calculator calc;
    // 绑定对象实例和成员函数，固定第二个参数为 5
    auto bound_memfn = std::bind(&Calculator::multiply, &calc, 
                                std::placeholders::_2, 5,std::placeholders_3,std::placeholders_1);
    std::cout << bound_memfn(1,2,3); // 输出 15（等效于 calc.multiply(2,5,3,1)）
    return 0;
}
````

### ::listen()
定义： ::listen() 是用于 将 TCP 套接字设置为监听模式 的系统调用函数，使服务器能够接受客户端的连接请求。
- 原型
````c++
#include <sys/socket.h>  // 头文件

int listen(int sockfd, int backlog);

sockfd：已绑定（bind()）的 TCP 套接字 的文件描述符。
backlog：等待连接队列的最大长度（决定同时处理的挂起连接数）
````
- 作用

::listen() 在 TCP 服务器中的核心作用：

模式转换：将套接字设为被动监听模式。

队列管理：初始化连接队列，控制并发处理能力。

服务准备：为后续 accept() 接受客户端连接奠定基础。

### ::accept4()
定义：::accept4() 是 Linux 系统特有的扩展函数，用于 接受传入的 TCP 连接并创建新套接字，相较于传统的 accept()，它允许直接设置新套接字的非阻塞（SOCK_NONBLOCK）和关闭时执行（SOCK_CLOEXEC）标志。
- 原型
````c++
#include <sys/socket.h>  // 头文件

int accept4(int sockfd, struct sockaddr *addr, socklen_t *addrlen, int flags);
参数：

sockfd：已通过 listen() 监听的 TCP 套接字 文件描述符。
addr：指向 sockaddr 结构体的指针，用于存储客户端地址信息（可传 NULL）。
addrlen：输入时为 addr 的缓冲区大小，输出时为实际地址长度（可传 NULL）。
flags：标志位组合，控制新套接字的传输方式。(SOCK_NONBLOCK\SOCK_CLOEXEC)

返回值：
成功：返回新套接字的文件描述符（用于与客户端通信）。
失败：返回 -1，并设置 errno（如 EAGAIN、ECONNABORTED）
````


### ::setsockopt() 与 ::getsockopt()
定义：setsockopt() 和 getsockopt() 是用于设置和获取套接字选项的系统函数，允许开发者对套接字的行为进行精细控制。
- 原型
````c++
#include <sys/socket.h>

int setsockopt(int sockfd, int protocol, int optname, const void *optval, socklen_t optlen);
int getsockopt(int sockfd, int protocol, int optname, void *optval, socklen_t *optlen);
optname：选项名称，如 SO_REUSEADDR、TCP_NODELAY 等。

optval：指向选项值的指针（设置或获取的缓冲区）。

optlen：选项值的长度（setsockopt）或缓冲区长度指针（getsockopt）。

````


### ::write()、::read() 与 read()、write()
定义：这两种分别是网络编程中的系统调用 和 对文件进行读写流
- 原型
````c++
#include <unistd.h>

// 从文件描述符 fd 读取数据到 buf，最多读取 count 字节
ssize_t ::read(int fd, void* buf, size_t count);

// 将 buf 中的 count 字节写入文件描述符 fd
ssize_t ::write(int fd, const void* buf, size_t count);
````



### ::shutdown()
定义：用于关闭socket连接
- 原型
````c++
::shutdown(sockfd_, SHUT_WR)
````

### std::copy
定义：是 C++ 标准库 <algorithm> 头文件中提供的一个通用算法，用于将数据从一个范围（容器、数组等）复制到另一个范围。
- 原型
````c++
#include <algorithm>

template <class InputIt, class OutputIt>
OutputIt std::copy(InputIt first, InputIt last, OutputIt d_first);

first, last: 输入范围的起始和结束迭代器（左闭右开区间 [first, last)）。
d_first: 目标范围的起始迭代器（从此位置开始写入数据）。
返回值：目标范围中最后一个被写入元素的下一个位置的迭代器。
````



 
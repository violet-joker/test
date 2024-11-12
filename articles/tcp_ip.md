# 常用函数接口(linux环境)

```c++
#include <sys/socket.h>

// 创建套接字，成功返回文件描述符，失败返回-1
// 协议族，套接字数据传输类型，通信协议信息(区分具体是协议族里的哪个协议)
int socket(int domain, int type, int protocol);

// 成功返回0，失败返回-1
// 套接字，存有地址信息的结构体变量地址，结构体长度
int bind(int sockfd, struct sockaddr* myaddr, socklen_t addrlen);

// 服务端监听套接字，连接请求等待队列长度(最多使多少个连接请求进入队列)
int listen(int sockfd, int backlog);

// 成功返回创建的套接字文件描述符，用于IO操作；失败返回-1
// 服务器套接字(充当门卫)，保存客户端地址信息的变量，保存结构体长度的变量(调用函数后自动填充)
int accept(int sockfd, struct sockaddr* addr, socklen_t* addrlen);

// 自动给客户端sock分配ip和端口号
// 客户端套接字，保存服务端地址信息的变量，地址变量长度(以字节为单位)
int connect(int sockfd, struct sockaddr* serv_addr, socklen_t addrlen);
```

打开文件

```c++
#include <sys/types.h>
#include <sys/stat.h>
#include <fcntl.h>

// 文件路径，打开模式(用or运算连接)
int open(const char* path, int flag);
// O_CREAT | O_TRUNC | O_APPEND | O_RDONLY | O_WRONLY | O_RDWR
```

读写数据、关闭文件

```c++
#include <unistd.h>
// 返回写入字节数，失败返回-1
ssize_t write(int fd, const void* buf, size_t nbytes);
// 成功返回字节数，读到文件结尾符返回0，失败返回-1
ssize_t read(int fd, void* buf, size_t nbytes);
int close(int fd);
```

# 套接字类型与协议设置

PF_INET     IPv4
PF_INET6    IPv6
PF_LOCAL    本地通信UNIX协议
PF_PACKET   底层套接字协议
PF_IPX      IPX Nevell协议

套接字类型：

面向连接SOCK_STREAM: 可靠的、按序传递的、
基于字节的面向连接的数据传输方式的套接字

- 传输过程中数据不会消失

- 按序传输数据

- 传输的数据不存在数据边界

面向消息SOCK_DGRAM: 不可靠、不按序、以数据的高速传输为目的的套接字

- 强调快速传输而非传输顺序

- 传输的数据可能丢失，也可能损毁

- 传输的数据有数据边界

- 限制每次传输的数据大小

# 地址族与数据序列

```c++
struct sockaddr_in {
    sa_family_t     sin_family; // 地址族
    uint16_t        sin_port;   // 16位TCP/UDP端口号，以网络字节序保存
    struct in_addr  sin_addr;   // 32位IP地址，以网络字节序保存
    char            sin_zero[8];// 不使用，但需要填充0
};

struct in_addr {
    In_addr_t       s_addr;     // 32位IPv4地址
};

struct sockaddr {
    sa_family_t     sin_family; // 地址族
    char            sa_data[14];// 地址信息，包含IP地址和端口号，剩余填充0
};
```

网络字节序与地址变换

不同系统字节序可能不同，例如0x1234，大端序系统为0x12 0x34，
小端序为0x34 0x12，通过网络传输数据时统一约定大端序。

```c++
// 字节序转换，host to network
#include <arpa/inet.h>
unsigned short htons(unsigned short);
unsigned short ntohs(unsigned short);
unsigned long htons(unsigned long);
unsigned long ntohs(unsigned long);

// 成功返回32位大端序整数值，失败返回INADDR_NONE(还可以检测无效地址)
#include <arpa/inet.h>
in_addr_t inet_addr(const char* string);   // 点分十进制字符串
```

```c++
#include <stdio.h>
#include <arpa/inet.h>

int main(int argc, char* argv[]) {
    const char* addr1 = "1.2.3.4";
    unsigned long conv_addr = inet_addr(addr1);
    printf("addr: %#lx \n", conv_addr);

    const char* addr2 = "1.2.3.256";  // 最大255
    conv_addr = inet_addr(addr2);
    if (conv_addr == INADDR_NONE)
        printf("error\n");
}
```

```c++
#include <arpa/inet.h>
// 将转换的结果填入in_addr结构体，成功返回1，失败返回0
int inet_aton(const char* string, struct in_addr* addr);

// 将整数型IP转换成字符串，成功返回字符串地址，失败返回-1
// 再次调用前该地址有效，否则可能会被覆盖
char* inet_ntoa(struct in_addr adr);
```

```c++
#include <stdio.h>
#include <arpa/inet.h>

int main() {
    const char* addr = "127.232.124.79";
    struct sockaddr_in addr_inet;
    if (inet_aton(addr, &addr_inet.sin_addr))
        printf("addr: %x \n", addr_inet.sin_addr.s_addr);
    else
        printf("error \n");
}
```

```c++
{
    struct sockaddr_in addr;
    char* serv_port = "9190";
    memset(&addr, 0, sizeof(addr));
    addr.sin_family = AF_INET;
    addr.sin_addr.s_addr = htonl(INADDR_ANY); // 自动分配IP地址
    addr.sin_port = htons(atoi(serv_port));
}
```


# 基于TCP的服务端/客户端

## TCP/IP 四层协议栈

- 链路层: 定义LAN、WAN、MAN等网络标准(物理连接)

- IP层: 解决路径选择

- TCP/UDP层: 完成实际数据传输(传输层)

- 应用层: 利用封装好的套接字，根据程序特点决定服务器、客户端之间的数据传输规定

## TCP服务端

> socket() -> bind() -> listen() -> accept() -> read()/write() -> close()

echo_server

```c++
#include <iostream>
#include <unistd.h>
#include <cstring>
#include <sys/socket.h>
#include <arpa/inet.h>
#define BUF_SIZE 1024

const char* port = "9091";
char message[BUF_SIZE];

int main() {
    sockaddr_in serv_adr; 
    memset(&serv_adr, 0, sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_adr.sin_port = htons(atoi(port));
    
    int serv_sock = socket(PF_INET, SOCK_STREAM, 0);
    bind(serv_sock, (sockaddr*) &serv_adr, sizeof(serv_adr));
    listen(serv_sock, 1);

    sockaddr_in clnt_adr;
    socklen_t clnt_adr_sz = sizeof(clnt_adr); 
    int clnt_sock = accept(serv_sock, (sockaddr*) &clnt_adr, &clnt_adr_sz);
    
    int str_len;
    while (str_len = read(clnt_sock, message, BUF_SIZE), str_len) {
        message[str_len] = 0;
        std::cout << "received: " << message << "\n";
        write(clnt_sock, message, str_len);
    }
    close(clnt_sock);
    close(serv_sock);
}
```

read函数从文件读取数据到指定缓冲区，
需要手动末尾添'\0'，防止后续访问字符串时越界。

## TCP客户端

> socket() -> connect() -> read()/write() -> close()

```c++
#include <iostream>
#include <unistd.h>
#include <cstring>
#include <sys/socket.h>
#include <arpa/inet.h>
#define BUF_SIZE 1024

const char* ip = "127.0.0.1";
const char* port = "9091";
char message[BUF_SIZE];

int main() {
    sockaddr_in serv_adr;
    memset(&serv_adr, 0, sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = inet_addr(ip);
    serv_adr.sin_port = htons(atoi(port));
    
    int sock = socket(PF_INET, SOCK_STREAM, 0);
    connect(sock, (sockaddr*) &serv_adr, sizeof(serv_adr));

    std::cout << "send: ";
    // ctrl + D，文件终止符
    while (std::cin.getline(message)) {
        write(sock, message, strlen(message));
        str_len = read(sock, message, BUF_SIZE);
        message[str_len] = 0;
        std::cout << "echo: " << message << "\n";
        std::cout << "send: ";
    }
    close(sock);
    std::cout << "client over\n";
}
```

# TCP内部工作原理

## 与对方套接字建立连接(三次握手)
    
通过向数据包分配序号并确认，可以在数据丢失时马上查看并重传丢失的数据包

> A ------ SYN -----> B
>
> A <--- SYN, ACK --- B
>
> A ------ ACK -----> B
    
## 与对方主机的数据交互

ACK号 = SEQ号 + 传递字节数 + 1
(下次要传递的SEQ号)

> A --- SEQ ---> B
>
> A <-- ACK ---- B

TCP启动计时器等待ACK应答，若超时则重传


## 断开与对方套接字的连接(四次挥手)

双方各发送一次FIN消息后断开连接

> A ----   FIN: SEQ   ----> B
>
> A <--- ACK: SEQ, ACK ---- B
>
> A <--- FIN: SEQ, ACK ---- B
>
> A ---- ACK: SEQ, ACK ---> B


# 基于UDP的回声服务端/客户端

不存在请求连接和受理过程，某种意义上无法明确区分服务端和客户端

存在数据边界，在UDP通信过程中I/O函数调用次数需要保持一致

```c++
#include <sys/socket.h>

// 成功返回传输字节数，失败返回-1
// 第一次调用时，若sock未绑定ip信息，则自动分配IP和端口
// 套接字，缓冲区，待传输字节数，可选项参数，目标地址信息，目标地址结构体变量长度
ssize_t sendto(int sock, void* buff, size_t nbytes, int flags,
                sockaddr* to, socklen_t addrlen);

// ... from结构体变量地址
ssize_t recvfrom(int sock, void* buff, size_t, nbytes, int flags,
                sockaddr* from, socklen_t* addrlen);
```

## UDP服务端

> socket() -> bind() -> sendto()/recvfrom() -> close()

```c++
#include <iostream>
#include <unistd.h>
#include <cstring>
#include <sys/socket.h>
#include <arpa/inet.h>
#define BUF_SIZE 30

const char* port = "9091";
char message[BUF_SIZE];

int main() {
    int serv_sock = socket(PF_INET, SOCK_DGRAM, 0);
    sockaddr_in serv_adr;
    memset(&serv_adr, 0, sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_adr.sin_port = htons(atoi(port));

    bind(serv_sock, (sockaddr*) &serv_adr, sizeof(serv_adr));

    sockaddr_in clnt_adr;
    socklen_t clnt_adr_sz;
    std::cout << "listening...\n";
    while (1) {
        clnt_adr_sz = sizeof(clnt_adr);
        int str_len = recvfrom(serv_sock, message, BUF_SIZE, 0,
                            (sockaddr*) &clnt_adr, &clnt_adr_sz);
        message[str_len] = 0;
        std::cout << "received: " << message << "\n";
        sendto(serv_sock, message, str_len, 0,
                (sockaddr*) &clnt_adr, clnt_adr_sz);
        if (str_len == 1 && message[0] == 'q') break;
    }
    close(serv_sock);
    std::cout << "server closed\n";
}
```

## UDP客户端 

> socket() -> sendto()/recvfrom() -> close()

```c++
#include <iostream>
#include <unistd.h>
#include <cstring>
#include <sys/socket.h>
#include <arpa/inet.h>
#define BUF_SIZE 30

const char* ip = "127.0.0.1";
const char* port = "9091";
char message[BUF_SIZE];

int main() {
    int sock = socket(PF_INET, SOCK_DGRAM, 0);
    sockaddr_in serv_adr;
    memset(&serv_adr, 0, sizeof(serv_adr));
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = inet_addr(ip);
    serv_adr.sin_port = htons(atoi(port));

    sockaddr_in from; 
    socklen_t adr_sz;
    std::cout << "send: ";
    while (1) {
        std::cin.getline(message, BUF_SIZE);
        int str_len = strlen(message);
        sendto(sock, message, str_len, 0,
            (sockaddr*) &serv_adr, sizeof(serv_adr));

        if (str_len == 1 && message[0] == 'q') break;

        adr_sz = sizeof(from); 
        str_len = recvfrom(sock, message, BUF_SIZE, 0,
                    (sockaddr*) &from, &adr_sz);
        message[str_len] = 0;
        std::cout << "echo: " << message << "\n";
        std::cout << "send: "; 
    }
    close(sock);
    std::cout << "client closed\n";
}
```

# 优雅地断开套接字连接

## 基于TCP的半关闭

既发送了EOF，又保留了输入流，可以接收对方数据。

```c++
#include <sys/socket.h>

// 待断开套接字，断开方式
int shutdonw(int sock, int howto);

SHUT_RD     // 断开输入流
SHUT_WR     // 断开输出流
SHUT_RDWR   // 同时断开I/O流
```

```c++
#include <stdio.h>

// 参数: 文件名称(相对或绝对路径)，打开模式；失败返回NULL
// 模式: r, w, a, r+, w+, a+; b(二进制), x(独占)
FILE* fopen(const char* filename, const char* mode);

// 待存储数据的缓冲区，单个数据字节数，数据个数，被读取的文件指针;
// 成功返回读取元素个数，失败或仅读取到EOF返回0
size_t fread(void* ptr, size_t size, size_t nmemb, FILE* stream);

size_t fwrite(const void* ptr, size_t size, size_t nmemb, FILE* stream);

FILE* fp = fopen("filename", "r");
fclose(fp);
```

```c++
#include <fstream>
#include <iostream>

std::ofstream outFile(const char* filename);
outFile << "write data" << std::endl;
outFile.write(buf, str_len);
outFile.close();

std::ifstream inFile(const char* filename);
getline(inFile, line);
inFile.read(buf, BUF_SIZE);
inFile.gcount(); // 上一次读取到的字节数
inFile.close();

// 文件打开模式
ios::in     // 读
ios::out    // 写
ios::ate    // 初始位置：文件尾
ios::app    // 所有输出附加到文件末尾
ios::trunc  // 如果文件存在，将其长度截断为0
ios::binary // 以二进制打开
```

# 域名及网络地址

DNS (Domain Name System, 域名系统)

```c++
#include <netdb.h>

// 通过域名获取地址
struct hostent* gethostbyname(const char* hostname);
// 通过地址获取域名
struct hostent* gethostbyaddr(const char* addr, socklen_t len, int family);
```

```c++
struct hostent {
    char* h_name;       // 官方域名
    char** h_aliases;   // 其他域名(可能多个域名绑定同一IP)
    int h_addrtype;     // 地址族，如AF_INET
    int h_length;       // IP地址长度(字节数)
    char** h_addr_list; // 域名对应的IP地址(可能一个域名对应多个IP: 负载均衡)
};
```

# 套接字的多种可选项

...

## Time-wait状态

关闭连接时，先发送FIN包的一端最后需要进入Time-wait状态，
因此即使服务端程序结束了，而系统套接字实际未关闭，此时
端口处于被占用状态，需要等待一段时间结束Time-wait后即可
重新绑定端口号。由于客户端套接字使用connect动态随机分配
端口号，而服务端绑定固定端口号，往往受影响的是服务端。

## Nagle算法

数据包发送过度可能会引起网络过载；
nagle算法主要是避免发送小的数据包，
要求TCP连接上最多只能有一个未被确认的小分组，
在该分组的确认到达前不能发送其他小分组。

适用于发送大批量的小数据。

# 多进程服务端

## 并发服务器实现方法

- 多进程服务器：通过创建多个进程提供服务

- 多路复用服务器：通过捆绑并统一管理I/O对象提供服务

- 多线程服务器：通过生成与客户端等量的线程池提供服务


### fork创建进程

```c++
#include <unistd.h>

// 创建调用的进程副本
// 父进程fork函数返回子进程ID，子进程fork函数返回0
// 成功返回进程ID，失败返回-1
pid_t fork(void);
```
### 僵尸进程

终止方式：调用exit(); return语句。

子进程exit或return后返回值会传递给操作系统，而操作系统不会销毁
子进程，此时处于僵尸进程，需要父进程回收。

```c++
#include <sys/wait.h>

// 成功返回终止的子进程ID，失败返回-1
// 阻塞等待子进程终止
// statloc指针指向变量存储子进程终止状态
pid_t wait(int* statloc);

// 若pid传递-1，则与wait相同，等待任意子进程终止
// options传递sys/wait.h中的常量WNOHANG，没有终止的子进程时返回0
// WNOHANG(wait no hang),子进程未结束时不陷入阻塞。
pid_t waitpid(pid_t pid, int* statloc, int options);

WIFEXITED(status); // (wait if exited) 子进程正常终止，返回true
WEXITSTATUS(status); // (wait exit status) 返回子进程返回值
```
### 信号与signal函数

```c++
#include <signal.h>

// 函数名signal，参数int signo和void (*func)(int)
// 返回:参数为int的void型函数指针
// 第一个参数为特殊情况信息，第二个参数为该情况下要调用的函数指针
// 触发信号时，会将signo作为参数传递给函数
void (*signal(int signo, void(*func)(int)))(int);

SIGALRM     // 已到通过调用alarm函数注册的时间
SIGINT      // 输入CTRL + C
SIGCHLD     // 子进程终止
```

```c++
#include <unistd.h>

// 返回0或以秒为单位的距SIGALRM信号发生所剩时间
// 传入正数，相应时间后以秒为单位，产生SIGALRM信号
// 若传递0，则之前的SIGALRM信号的预约将取消
// 若预约信号后未指定该信号对应的处理函数，则终止进程
unsigned int alarm(unsigned int seconds);
```


进程处于睡眠状态时无法调用函数，产生信号时，为了调用信号处理器，
将唤醒由sleep函数进入阻塞状态的进程；而进程一旦被唤醒，不会再次
进入睡眠状态。


```c++
#include <signal.h>
// 成功返回0，失败返回-1
// act对应信号处理函数，oldact获取之间注册的信号函数处理函数指针(不需要则传0)
// 更通用
int sigaction(int signo, const struct sigaction* act, struct sigaction* oldact);

struct sigaction {
    void (*sa_handler)(int);    // 保存信号处理函数的地址
    sigset_t sa_mask;
    int sa_flags;
};
```

### 基于多任务的并发服务器

```c++
#include <iostream>
#include <unistd.h>
#include <signal.h>
#include <sys/wait.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#define BUF_SIZE 30

int port = 9091;
char buf[BUF_SIZE];

void read_childproc(int sig) {
    int pid, status;
    pid = waitpid(-1, &status, WNOHANG);
    std::cout << "removed child proc id: " << pid << "\n";
}

int main() {
    // 注册信号函数
    struct sigaction act {};
    act.sa_handler = read_childproc;
    sigaction(SIGCHLD, &act, 0);

    // 绑定服务端地址
    sockaddr_in serv_adr {}, clnt_adr;
    socklen_t adr_sz;
    serv_adr.sin_family = AF_INET;
    serv_adr.sin_addr.s_addr = htonl(INADDR_ANY);
    serv_adr.sin_port = port;
    
    int serv_sock = socket(PF_INET, SOCK_STREAM, 0);
    bind(serv_sock, (sockaddr*) &serv_adr, sizeof(serv_adr));
    listen(serv_sock, 5);
    while (true) {
        adr_sz = sizeof(clnt_adr);
        int clnt_sock = accept(serv_sock, (sockaddr*) &clnt_adr, &adr_sz);
        // 这里需要判断由资源竞争引起的accep连接失败
        if (clnt_sock == -1) continue;
        std::cout << "new client connected...\n";
        int pid = fork();
        if (pid == 0) {
            close(serv_sock);
            int str_len;
            while (str_len = read(clnt_sock, buf, BUF_SIZE), str_len)
                write(clnt_sock, buf, str_len);

            close(clnt_sock);
            std::cout << "client disconnected...\n";
            return 0;
        } else
            close(clnt_sock);
    }
    close(serv_sock);
}
```

子进程复制父进程socket的文件描述符，对通信的socket引用计数。

### 回声客户端的I/O程序分割

将read和write从循环里分开，主进程只进行写操作，子进程只进行读操作。


```c++
int pid = fork();
if (pid == 0)
    read_routine(sock, buf);
else
    write_routine(sock, buf);
close();
```

# 进程间通信

管道通信

```c++
#include <unistd.h>

// filedes[0]接收数据时的文件描述符，即管道出口
// filedes[1]传输数据时的文件描述符，即管道入口
int pipe(int filedes[2]);

write(fds[1], buf, sizeof(buf));
read(fds[0], buf, BUF_SIZE);
```

管道可以双向通信，但先调用read的进程将得到数据。
创建两个管道即可实现。

# I/O复用

## select函数

```c++
#include <sys/select.h>
#include <sys/time.h>

// 错误返回-1，超时返回0，成功返回发生事件的文件描述符数
int select(
    int maxfd,                      // 监视对象文件描述符数量
    fd_set* readset,                // 待读取的注册
    fd_set* writeset,               // 待写入的注册
    fd_set* exceptset,              // 发生异常的注册
    const struct timeval* timeout   // 限制超时，不设置超时传入NULL
);

struct timeval {
    long tv_sec;    // seconds
    long tv_usec;   // microseconds
};

FD_ZERO(fd_set* fdset);             // 所有位初始化成0
FD_SET(int fd, fd_set* fdset);      // 在参数指向的变量中注册fd信息
FD_CLR(int fd, fd_set* fdset);      // 清除fd的信息
FD_ISSET(int fd, fd_set* fdset);    // 若包含fd信息，则返回真

```

- 设置文件描述符、指定监视范围、设置超时

- 调用select函数

- 查看调用结果

```c++
#include <iostream>
#include <unistd.h>
#include <sys/select.h>
#include <sys/time.h>
#define BUF_SIZE 30

int main() {
    fd_set reads, tmps;
    FD_ZERO(&reads);
    FD_SET(0, &reads); // 0 is standard input(console)
    timeval timeout;
    timeout.tv_sec = 5;
    timeout.tv_usec = 0;
    while (true) {
        tmps = reads;
        int res = select(1, &tmps, 0, 0, &timeout); 
        if (res == -1)
            std::cout << "select error\n";
        else if (res == 0)
            std::cout << "timeout\n";
        else {
            int str_len = read(0, buf, BUF_SIZE); 
            buf[str_len] = '\0';
            std::cout << "echo: " << buf;
        }
    }
}
```


# 套接字描述符

## socket

```cpp
#include <sys/socket.h>
int socket (int domain, int type, int protocol);
```

返回值：成功返回文件（套接字）描述符，出错返回-1

参数：

+ domain：确定通信的特性，包括地址格式

|    域     |    描  述    |
| :-------: | :----------: |
|  AF_INET  | IPv4因特网域 |
| AF_INET6  | IPv6因特网域 |
|  AF_UNIX  |    UNIX域    |
| AF_UNSPEC |    未指定    |

+ type：确定套接字的类型，进一步确定通信特征

|     类  型     |                 描  述                 |
| :------------: | :------------------------------------: |
|   SOCK_DGRAM   |   长度固定的、无连接的不可靠报文传递   |
|    SOCK_RAW    |           IP协议的数据报接口           |
| SOCK_SEQPACKET | 长度固定、有序、可靠的面向连接报文传递 |
|  SOCK_STREAM   |    有序、可靠、双向的面向连接字节流    |

+ protocol：通常是0，表示按给定的域和套接字类型选择默认协议。

  > 当对同一域和套接字类型支持多个协议时，可以使用protocol参数选择一个特定协议：
  >
  > 在AF_INET通信域中套接字类型SOCK_STREAM的默认协议是TCP（传输控制协议）
  >
  > 在AF_INET通信域中套接字类型SOCK_DGRAM的默认协议时UDP（用户 数据包协议）

# 寻址

## 字节序

网络协议指定了字节序，因此异构计算机系统能够交换协议信息而不会混淆字节序

TCP/IP协议栈采用了大端字节序。

对于TCP/IP，地址用网络字节序来表示，所以应用程序有时需要在处理器的字节序（主机字节序）与网络字节序之间的转换

对于TCP/IP应用程序，提供了四个通用函数以实施在处理器字节序和网络字节序之间的转换

```cpp
#include <arpa/inet.h>
uint32_t htonl(uint32_t hostlong32);    // 返回值：以网络字节序表示的32位整型数
uint16_t htons(uint16_t hostshort16);    // 返回值：以网络字节序表示的16位整型数
uint32_t ntohl(uint32_t netlong32);    // 返回值：以主机字节序表示的32位整型数
uint16_t ntohs(uint16_t netshort16);    // 返回值：以主机字节序表示的16位整型数
```

h----host    n----network    l----long    s----short

## 地址格式

地址标识了特定通信域中的套接字端点，地址格式与特定的通信域相关。为使不同格式地址能够被传入套接字函数，地址被强制转换成通用的地址结构sockaddr表示：（Linux中的定义）

```cpp
struct sockaddr
{
    sa_family_t   sa_family;
    char          sa_data[14];
};
```

因特网地址定义在<netinet/in.h>中。在IPv4因特网域（AF_INET）中，套接字地址用如下结构sockaddr_in表示：(Linux中的定义)

```cpp
typedef uint32_t in_addr_t;
typedef uint16_t in_port_t;

struct sockaddr_in
{
    sa_family_t    sin_family;  // 地址族
    int_port_t     sin_port;    // 端口号
    struct in_addr sin_addr;    // IPv4地址
    unsigned char  sin_zero[8]; // 填充字段，必须被全置为0
};
struct in_addr
{
    in_addr_t       s_addr;
};
```



```cpp
#include <arpa/inet.h>
const char* inet_ntop(int domain, 
                      cosnt void* restrict addr, 
                      char* restrict str, 
                      socklen_t size);
// 将网络字节序的二进制地址转换成文本字符串格式
// 返回值：若成功则返回地址字符串空指针，若出错返回NULL

int inet_pton(int domain, const char* restrict str, 
             void* restrict addr);
// 将文本字符串格式转换成网络字节序的二进制地址
// 返回值：若成功返回1，若格式无效返回0，若出错返回1
```

参数 ：

+ domain：仅支持两个值：AF_INET 和 AF_INET6
+ str：保存转换后的缓冲区
+ size：指定了用以保存文本字符串的缓冲区（str）的大小



## bind

与客户端的套接字关联的没有太大意义，可以让系统选一个默认的地址。

然而，对于服务器，需要给一个接受客户端请求的套接字绑定一个总所周知的地址。

可以用bind函数将地址绑定到一个套接字

```cpp
#include <sys/socket.h>
int bind(int sockfd, const struct sockaddr* addr, 
         socklen_t len);
```

+ 返回值：若成功则返回0，若出错返回-1
+ 对于**所能使用的地址**的限制
  - 在进程所运行的机器上，指定的地址必须有效，不能指定一个其他机器的 
  - 地址必须和创建套接字时的地址族所支持的格式向匹配
  - 端口号必须不小于1024，除非该进程具有相应的特权（即为超级用户）
  - 一般只有套接字端点能够与地址绑定，尽管有些协议允许多重绑定

# 建立连接

## connect

如果处理的是面向处理连接的服务（SOCK_STREAM 或 SOCK_SEQPACKET），在开始叫唤数据以前，需要在请求服务的进程套接字（客户端）和提供服务的进程套接字（服务器）之间建立一个连接。

可以用connect建立一个连接：

```cpp
#include <sys/socket.h>
int connect(int sockfd, const struct sockaddr* addr, 
           socklen_t len);
```

+ 返回值：成功返回0，出错返回-1
+ 在connect中所指定的地址是想与之通信的服务器地址。如果sockfd没有绑定到一个地址，connect会给调用者绑定一个默认地址

## listen

服务器调用listen来宣告可以接受连接请求

```cpp
#include <sys/socket.h>
int listen(int sockfd, int backlog);
```

+ 返回值：成功返回0，失败返回-1
+ 参数backlog提供了一个提示，用于表示该进程所要入队的连接请求数量。其实际值由系统决定，但上限由<sys/socket.h>中SOMAXCONN指定
  - 一旦队列满，系统会拒绝多余连接请求，所以backlog的值应该基于服务器期望负载和接受连接请求与启动服务的处理能力来选择

## accept

一旦服务器调用了listen，套接字就能接受连接请求。使用函数accept获得连接请求并建立连接

```cpp
#include <sys/socket.h>
int accept(int sockfd, struct sockaddr* restrict addr, 
          socklen_t* restrict len);
```

+ 返回值：成功返回文件（套接字）描述符，出错返回-1
+ 函数accept返回的文件描述符是套接字描述符，该描述符连接到调用connect的客户端
+ 这个新的套接字描述符和原始套接字(sockfd)具有相同的套接字类型和地址族
+ 如果不关心客户端标识，可以将参数 addr 和 len 设为NULL；否则，在调用accept之前，应该将参数addr设为足够大的缓冲区来存放地址，并且将len设为指向代表这个缓冲区大小的整数的指针。返回时，accept会在缓冲区填充客户端的地址并且更新指针len所指向的整数为该地址的大小
+ 如果没有连接请求等待处理，accept或阻塞直到一个请求的到来。如果sockfd处于非阻塞模式，accept或返回-1并将 errno 设置为 RAGAIN 或 EWOULDBLOCK

# 数据传输

既然将套接字端点表示为文件描述符，那么只要建立连接，就可以使用 read 和 write 来通过套接字通信

尽管可以通过read和write交换数据，但这就是这两个函数所能做的一切。如果想指定选项、从多个客户端接收数据包或者发送带外数据，需要采用六个传递数据的套接字函数中的一个

## send

```cpp
#include <sys/socket.h>
ssize_t send(int sockfd, const void* buf, size_t nbytes,
             int flags);
```

+ 返回值：成功返回发送的字节数，出错返回-1
+ 类似write函数，使用send时套接字必须已经连接
+ send支持第四个参数flags：MSG_DONTROUTE、MSG_DONTWAIT、MSG_EOR、MSG_OOB
+ 如果send成功返回，并不必然表示连接另一端的进程接收数据。所保证的仅是当send成功返回时，数据已经无错误地发送到网络上
+ 对于支持为报文设限的协议，如果单个报文超过协议所支持的最大尺寸，send失败并将errno设为EMSGSIZE，对于字节流协议，send会阻塞直到整个数据被传输

## sendto

```cpp
#include <sys/socket.h>
ssize_t sendto(int sockfd, const void* buf, size_t nbytes,
              int flags, const struct sockaddr* destaddr,
              socklen_t destlen);
```

+ 返回值：成功返回发送的字节数，出错返回-1
+  面向连接的套接字，目标地址是忽略的，因为目标地址蕴涵在连接中。对于无连接的套接字，不能使用send。除非在调用connect时预先设定了目标地址，或者采用sendto来提供另一种发送报文方式

## sendmsg

```cpp
#include <sys/socket.h>
ssize_t sendmsg(int sockfd, const struct msghdr* msg, 
                int flags);
```

+ 可以调用带有msghdr结构的sendmsg来指定多重缓冲区传输数据

## recv

```cpp
#include <sys/socket.h>
ssize_t recv(int sockfd, void* buf, size_t nbytes, 
             int flags);
```

+ flags标志：

  |    标志     |                    描述                    |
  | :---------: | :----------------------------------------: |
  |   MSG_OOB   |         如果协议支持，接收带外数据         |
  |  MSG_PEEK   |        返回报文内容而不真正取走报文        |
  |  MSG_TRUNC  | 即使报文被截断，要求返回的是报文的实际长度 |
  | MSG_WAITALL |  等待直到所有的数据可用（仅SOCK_STREAM）   |

  1. 当指定MSG_PEEK标志时，可以查看下一个要读的数据但不会真正取走。当再次调用read或recv函数时会返回刚才查看的数据
  2. 对于SOCK_STREAM套接字，接收的数据可以比请求的少。标志MSG_WAITALL阻止这种行为，除非所需数据全部收到，recv函数才会返回。对于 SOCK_DGRAM 和 SOCK_SEQPACKET 套接字，MSG_WAITALL标志没有改变什么行为，因为这些基于报文的套接字类型一次读取就返回整个报文
  3. 如果发送者已经调用shutdown来结束传输，或者网络协议支持默认的顺序关闭并且发送端已经关闭，那么当所有的数据接收完毕后，recv返回0

如果有兴趣定位发送者，可以使用recvfrom来得到数据发送者的源地址

```cpp
#include <sys/socket.h>
ssize_t recvfrom(int sockfd, void* restrict buf, 
                 size_t len, int flags, 
                 struct sockaddr* restrict addr, 
                 socklen_t* restrict addrlen);
```

+ 返回值：以字节计数的消息长度，若无可用消息或对方已经按序结束则返回0，若出错则返回-1
+ 如果addr非空，它将包含数据发送者的套接字端点地址。当调用recvfrom时，需要设置addrlen参数指向一个包含addr所指向的套接字缓冲区字节大小的整数。返回时，该整数设为该地址的实际字节大小
+ 因为可以获得发送者的地址，recvfrom通常用于无连接套接字，否则，recvfrom等同于recv

为了将接受到的数据送入多个缓冲区，或者想接收辅助数据，可以使用recvmsg：

```cpp
#include <sys/socket.h>
ssize_t recvmsg(int sockfd, struct msghdr* msg, 
                int flags);
```

+ 以字节计数的消息长度，若无可用消息或对方已经按序结束则返回0，若出错则 -1
+ 结构msghdr被recvmsg用于指定接收数据的输入缓冲区。
+ 可以设置参数flags来改变recvmsg的默认行为


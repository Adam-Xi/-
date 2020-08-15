| 不带缓存直接进行IO操作                          | 基于缓存对文件进行操作                                       |
| ----------------------------------------------- | ------------------------------------------------------------ |
| int open(const char*path, int flags, int perms) | FILE* fopen(const char* path, const char* mode)              |
| int close(int fd)                               | int fclose(FILE *stream)                                     |
| size_t read(int fd, void *buf, size_t count)    | size_t fread(void* ptr,size_t size,size_t nmemb, FILE\* stream) |
| size_t write(int fd, void *buf, size_t count)   | size_t fwrite(const void \*ptr,size_t size,size_t nmemb,FILE*stream) |
| off_t lseek(int fd, off_t offset, int whence)   | int fseek(FILE *stream, long offset, int fromwhere);         |

1、fread带有缓冲，是read的衍生，即fread在底层调用的read实现

2、

+ fopen / fread / fwrite是C标准的库函数，操作的对象时 FILE STREAM（文件流）
+ open / read / write是和操作系统相关的系统调用，操作的对象时 FILE DESCRIPTOR（文件描述符）

3、返回的是一个FILE结构指针，read返回的是一个int的文件号（文件描述符）

> 一文件的大小是8k
>
> 如果用read/write，且只分配了2k的缓存，则要将此文件读出需要做4次系统调用 来实际从磁盘上读出
>
> 如果用fread/fwrite，则系统自动分配缓存，则读出此文件只要一次系统调用从磁 盘上读出
>
> 也就是用read/write要读4次磁盘，而用fread/fwrite则只要读1次磁盘。效率比read /write要高4倍。 如果程序对内存有限制，则用read/write比较好

4、

+ 一般情况下，用fread / fwrite来处理文件，它自动分配缓存，速度会很快
+ 用read / write 处理一些特殊的描述符，如套接字、管道等



## open / fopen

+ open返回一个文件描述符，fopen返回一个文件指针
+ 在用户态下open无缓冲区，fopen有缓冲区
  + fopen在用户态下有缓存，进行读写时减少了用户态和内核态的切换
  + open每次都需要进行内核态和用户态的切换
  + 表现为：如果顺序访问文件，fopen系列的函数比直接调用open系列快，如果随机访问文件open要比fopen快
+ open不可移植，fopen可移植
+ open常用来打开设备文件，fopen用来打开普通文件


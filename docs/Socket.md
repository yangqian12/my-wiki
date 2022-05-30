### Socket

---

**socket**

```cpp
int socket(int domain, int type, int protocol);
```

`socket` 函数对应于普通文件的打开操作。

* domain。 协议域，常用的协议族有 AF_INET、AF_INET6、AF_LOCAL（或称 AF_UNIX，Unix 域 socket ）、AF_ROUTE 等等。
* type。 指定 socket 类型。常用的 socket 类型有，SOCK_STREAM、SOCK_DGRAM、SOCK_RAW、SOCK_PACKET、SOCK_SEQPACKET 等等。
* protocol。 指定协议。常用的协议有 IPPROTO_TCP、IPPTOTO_UDP、IPPROTO_SCTP、IPPROTO_TIPC 等。

当我们调用 `socket` 创建一个 socket 时，返回的文件描述符存在于协议簇空间中，并没有一个具体的地址，如果想要给它赋值一个地址，需要使用 `bind()` 绑定一个地址，否则当调用 `connect()`, `listen()` 时系统会随机分配一个端口。



**bind**

```cpp
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

* sockfd。即 socket 描述符。
* addr。指向给 sockfd 绑定的地址。
* addrlen。对应地址的长度。



主机字节序

​    主机字节序与 CPU 类型有关，如 Intel 一般为 小端模式， IBM 和 Oracle 一般为大端模式。小端模式为低位字节存放在低地址端。

网络字节序

​    网络字节序为大端字节序。所以通过网络传输时需要将主机字节序转换为网络字节序。



**listen 和 connect**

```cpp
int listen(int sockfd, int backlog);
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);
```

`listen` 函数的第一个参数 sockfd 为监听的描述符， 第二个参数 backlog 为相应 socket 可以排队的最大连接数。

客户端通过调用 `connect` 函数获得与服务端的连接。



**accept**

Tcp 客户端依次调用 `socket`, `connect` 后向 Tcp 服务器发起一个连接请求，服务器监听到这个请求之后，就会调用 `accept` 接收请求。这样连接就建立好了。

```cpp
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);
```

如果 accept 成功，则返回由内核自动生成的一个全新的描述符，代表与返回客户的 tcp 连接。内核为由每个服务器进程接受的客户连接创建了一个已连接 socket 描述符，当服务器完成了对某个客户的服务，该描述字就会被关闭。2以下几组。

* read() / write()
* recv() / send()
* readv() / writev()
* recvmsg() / sendmsg()
* recvfrom() / sendto() 

`recvmsg` 和 `sendmsg` 是最通用的函数。



**close**

完成读写操作就要关闭相应的 socket 描述符

```cpp
int close(int sockfd);
```

close 操作只是使相应 socket 描述字的引用计数 -1，只有当引用计数为 0 的时候，才会触发 TCP 客户端向服务器发送终止连接请求。




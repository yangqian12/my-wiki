###  ceph 网络层代码

****

路径 src/msg

- simple
- async
- xio

这三个关系是并列关系， 是网络层的三种实现。

其中 SimpleMessenger 是网络层最早的实现。

数据格式

```cpp
 class Message : public RefCountedObject {
  protected:
     ceph_msg_header  header;      // headerelope
     ceph_msg_footer  footer;
     bufferlist       payload;  // "front" unaligned blob
     bufferlist       middle;   // "middle" unaligned blob
     bufferlist       data;     // data payload (page-alignment will be preserved where possible)
     ......
 }
```

消息内容可以分为三个部分

- header
- user data
- footer

user data 当中可以分为三个部分

+ payload
+ middle
+ data

payload 一般是 ceph 操作的元数据， middle 是预留字段目前没有使用， data 是读写数据。

先看 header

```cpp
struct ceph_msg_header {
    __le64 seq;       /* message seq# for this session */
	__le64 tid;       /* transaction id */
	__le16 type;      /* message type */
	__le16 priority;  /* priority.  higher value == higher priority */
	__le16 version;   /* version of message encoding */

	__le32 front_len; /* bytes in main payload */
	__le32 middle_len;/* bytes in middle payload */
	__le32 data_len;  /* bytes of data payload */
	__le16 data_off;  /* sender: include full offset;
			     receiver: mask against ~PAGE_MASK */

	struct ceph_entity_name src;

	/* oldest code we think can decode this.  unknown if zero. */
	__le16 compat_version;
	__le16 reserved;
	__le32 crc; 
}
```

front_len, middle_len, data_len 用来记录 payload， middle， data 的长度。

data_off    ??

footer 的数据结构

```cpp
struct ceph_msg_footer {
	__le32 front_crc, middle_crc, data_crc;
	// sig holds the 64 bits of the digital signature for the message PLR
	__le64  sig;
	__u8 flags;
} __attribute__ ((packed));
```

计算front, middle, data 的 crc， 得到 front_crc, middle_crc, data_crc。



#### 网络层的数据结构

`SimpleMessenger`

```cpp
class SimpleMessenger : public SimplePolicyMessenger {
 public:
    Accepter accepter;
    DispatchQueue dispatch_queue;
    
    friend class Accepter;
    Pipe *add_accept_pipe(int sd);
    // ...
    set<Pipe*> accepting_pipes;
    /// a set of all the Pipes we have which are somehow active
    set<Pipe*>      pipes;
    /// a list of Pipes we want to tear down
    list<Pipe*>     pipe_reap_queue;
    
    // ...
}
```

```cpp
class SimpleMessenger : public SimplePolicyMessenger {
｝

class SimplePolicyMessenger : public Messenger
{
}
```

`SimpleMessenger` 继承 `SimplePolicyMessenger` ， 而 `SimplePolicyMessenger` 继承自 `Messenger` 

```cpp
class Messenger {
private:
  std::deque<Dispatcher*> dispatchers;
  std::deque<Dispatcher*> fast_dispatchers;
  // ...
}
```

所以 `SimpleMessenger` 成员还包括 `dispatchers`, `fast_dispatchers` 

##### Accepter

和传统网络通信一样，用来接受网络连接。和传统socket通信一样，要想接受网络连接，必须要有电话机（socket），绑定到电话号码（bind），要接上线路，准备接受链接（listen），最后要等待其他地址拨过来的链接请求（accept）。

以 ceph_osd 为例， `SimpleMessenger` 会调用 bind， 做上面提到的事， 而 `SimpleMessenger` 的 bind， 将这些事情委托给了 ` Accepter accepter`  。

```cpp
int SimpleMessenger::bind(const entity_addr_t &bind_addr)
{
  lock.Lock();
  if (started) {
    ldout(cct,10) << "rank.bind already started" << dendl;
    lock.Unlock();
    return -1;
  }
  ldout(cct,10) << "rank.bind " << bind_addr << dendl;
  lock.Unlock();

  // bind to a socket
  set<int> avoid_ports;
  int r = accepter.bind(bind_addr, avoid_ports);
  if (r >= 0)
    did_bind = true;
  return r;
}
```

Accepter 类

```cpp
class Accepter : public Thread {
  SimpleMessenger *msgr;
  bool done;
  int listen_sd;
  uint64_t nonce;
  int shutdown_rd_fd;
  int shutdown_wr_fd;
  int create_selfpipe(int *pipe_rd, int *pipe_wr);

public:
  Accepter(SimpleMessenger *r, uint64_t n) 
    : msgr(r), done(false), listen_sd(-1), nonce(n),
      shutdown_rd_fd(-1), shutdown_wr_fd(-1)
    {}
    
  void *entry() override;
  void stop();
  int bind(const entity_addr_t &bind_addr, const set<int>& avoid_ports);
  int rebind(const set<int>& avoid_port);
  int start();
};

```

`Accepter::bind()` 的方法实现， 不外乎 socket， bind， listen

```cpp
int Accepter::bind(const entity_addr_t &bind_addr, const set<int>& avoid_ports) {
    
}
```

Accepter类继承自Thread，它本质是个线程

Accepter这个线程的主要任务，就是accept，接受到来的连接。前面也讲过，这个线程不能用来处理客户的通信请求，因为两者的通信可能很墨迹，不能因为双方通话，导致其他所有的连接请求无法响应。这个线程也是这么做的，基本上是接到请求，然后创建Pipe负责该请求，继续accept，等待新的请求。

线程的入口函数为 `entry()` 

```cpp
void *Accepter::entry() {
    while (!done) {
        // ...
        int sd = accept_cloexec(listen_sd, (sockaddr*)&ss, &slen);
    	if (sd >= 0) {
            errors = 0;
            ldout(msgr->cct,10) << __func__ << " incoming on sd " << sd << dendl;
            msgr->add_accept_pipe(sd);
        }
        // ...
    }
}
```

这个线程是个死循环，不停的等待客户端的连接请求，收到连接请求后，调用msgr->add_accept_pipe，可以理解为创建pipe，由专门的线程负责与客户端通信，而本线程继续accept。

SimpleMessenger的 add_accept_pipe函数：

```cpp
Pipe *SimpleMessenger::add_accept_pipe(int sd)
{
  lock.Lock();
  Pipe *p = new Pipe(this, Pipe::STATE_ACCEPTING, NULL);
  p->sd = sd;
  p->pipe_lock.Lock();
  p->start_reader();
  p->pipe_lock.Unlock();
  pipes.insert(p);
  accepting_pipes.insert(p);
  lock.Unlock();
  return p;
}
```

服务端， accept 之后， 创建了一个新的 pipe 数据结构，然后将新的pipe 放到 SimpleMessenger 的 pipes 和 accepting_pipes 中去。

client 端如何连接服务器监听端口

见（src/test/messenger/simple_client.cc)

- 创建 messenger 实例

  ```cpp
  messenger = Messenger::create(g_ceph_context, g_cong->ms_type,
                               entity_name_t::MON(-1),
                               "client",
                               getpid());
  // enable time print
  messenger->set_magic(MSG_MAGIC_TRACE_CTR);
  messenger->set_default_policy(Messenger::Policy::lossy_cli);
  ```

- 绑定一个地址

  ```cpp
  messenger->bind(bind_addr);
  ```

- 创建Dispatcher 类并添加到messenger，用于接收消息。

  ```cpp
  dispatcher = new SimpleDispatcher(messenger);
  messenger->add_dispatcher_head(dispatcher);
  dispatcher->set_active();  // this side is the pinger
  ```

- 启动 messenger

  ```cpp
  r = messenger->start();
  ```

- 获得服务端的连接

  ```cpp
  conn = messenger->connect_to_mon(dest_addrs);
  ```

- 通过 connection 发送消息

  ```cpp
  ConnectionRef SimpleMessenger::connect_to(int type,
  					  const entity_addrvec_t& addrs)
  {
    Mutex::Locker l(lock);
    if (my_addr == addrs.front()) {
      // local
      return local_connection;
    }
  
    // remote
    while (true) {
      Pipe *pipe = _lookup_pipe(addrs.legacy_addr());
      if (pipe) {
        ldout(cct, 10) << "get_connection " << addrs << " existing " << pipe << dendl;
      } else {
        pipe = connect_rank(addrs.legacy_addr(), type, NULL, NULL);
        ldout(cct, 10) << "get_connection " << addrs << " new " << pipe << dendl;
      }
      Mutex::Locker l(pipe->pipe_lock);
      if (pipe->connection_state)
        return pipe->connection_state;
      // we failed too quickly!  retry.  FIXME.
    }
  }
  ```

  首先尝试查找已经存在的 Pipe (_lookup_pipe)，如果可以复用， 就不再创建，否则通过 connect_rank 创建新的 Pipe。

  ```cpp
  Pipe *SimpleMessenger::connect_rank(const entity_addr_t& addr,
  				    int type,
  				    PipeConnection *con,
  				    Message *first)
  {
    ceph_assert(lock.is_locked());
    ceph_assert(addr != my_addr);
    
    ldout(cct,10) << "connect_rank to " << addr << ", creating pipe and registering" << dendl;
    
    // create pipe
    Pipe *pipe = new Pipe(this, Pipe::STATE_CONNECTING,
  			static_cast<PipeConnection*>(con));
    pipe->pipe_lock.Lock();
    pipe->set_peer_type(type);
    pipe->set_peer_addr(addr);
    pipe->policy = get_policy(type);
    pipe->start_writer();
    if (first)
      pipe->_send(first);
    pipe->pipe_lock.Unlock();
    pipe->register_pipe();
    pipes.insert(pipe);
  
    return pipe;
  }
  ```

  connect 是 client 的行为，类似于 socket 通信中的 connect() 系统调用，它真正通信前，创建了 Pipe 数据结构。到服务器短，accept 收到 connect 请求后，立刻创建了 Pipe，就返回了。accept 再等待新的连接请求。

  ##### Pipe 

  在 linux 编程中，也有管道的概念，用于进程间通信，有亲缘关系的进程之间，一个只有 read_fd, 一个只有 write_fd, 一个进程向 write_fd 写入内容，另一个进程就能从 read_fd 读入内容。从而达到进程间通信的目的。

  ceph 的 Pipe 类， 类似于 linux 下的 Pipe。数据流向是单向的，一个Pipe 只能做到单向通信， ceph 同时具有读线程和写线程。这样能做到双向通信。

  ```cpp
    class Pipe : public RefCountedObject {
      /**
       * The Reader thread handles all reads off the socket -- not just
       * Messages, but also acks and other protocol bits (excepting startup,
       * when the Writer does a couple of reads).
       * All the work is implemented in Pipe itself, of course.
       */
      class Reader : public Thread {
        Pipe *pipe;
      public:
        explicit Reader(Pipe *p) : pipe(p) {}
        void *entry() override { pipe->reader(); return 0; }
      } reader_thread;
  
      /**
       * The Writer thread handles all writes to the socket (after startup).
       * All the work is implemented in Pipe itself, of course.
       */
      class Writer : public Thread {
        Pipe *pipe;
      public:
        explicit Writer(Pipe *p) : pipe(p) {}
        void *entry() override { pipe->writer(); return 0; }
      } writer_thread;
  ```

  在 `add_accept_pipe()` 函数中有一句 `p->start_reader()` ；

  而在 client 端， `connect_rank()` 函数中有 `pipe->start_writer()` ；

  其实意思很明确，就是 client 端的 Pipe 主动的 start_writer, 而 server 端接受 client 端的请求，因此，它启动了它的 read 线程 start_reader。

  ```cpp
  void Pipe::start_reader()
  {
    ceph_assert(pipe_lock.is_locked());
    ceph_assert(!reader_running);
    if (reader_needs_join) {
      reader_thread.join();
      reader_needs_join = false;
    }
    reader_running = true;
    reader_thread.create("ms_pipe_read", msgr->cct->_conf->ms_rwthread_stack_bytes);
  }
  ```

  ```cpp
  void Pipe::start_writer()
  {
    ceph_assert(pipe_lock.is_locked());
    ceph_assert(!writer_running);
    writer_running = true;
    writer_thread.create("ms_pipe_write", msgr->cct->_conf->ms_rwthread_stack_bytes);
  }
  ```

  reader_thread 线程执行的是 `Pipe::reader`() , 主要是从 socket 中读出信息。刚刚创建的时候， Pipe 处于 `Pipe::STATE_ACCEPTING`  状态， 而在 `Pipe::reader` 函数中，如果状态处于 STATE_ACCEPTING 状态，会执行 `Pipe::accept()` ,  该函数中，会调用 `start_writer` , 启动 Pipe 的写线程。

  ```cpp
  int Pipe::accept() {
      // ...
      pipe_lock.Lock();
      discard_requeued_up_to(newly_acked_seq);
      if (state != STATE_CLOSED) {
          ldout(msgr->cct,10) << "accept starting writer, state " << get_state_name() << dendl;
          start_writer();
      }  
      // ...
  }
  ```

  再来看 client 端创建的 Pipe

  ```cpp
  Pipe *pipe = new Pipe(this, Pipe::STATE_CONNECTING,
                        static_cast<PipeConnection*>(con));
  ```

  新创建的 Pipe 处于 `STATE_CONNECTING`  状态。`start_writer()` 主要函数为 `Pipe::writer()` 。如果 Pipe 处于 `STATE_CONNECTING`  状态，会调用 `Pipe::connect()` 函数。

  ```cpp
  void Pipe::writer()
  {
    pipe_lock.Lock();
    while (state != STATE_CLOSED) {// && state != STATE_WAIT) {
      ldout(msgr->cct,10) << "writer: state = " << get_state_name()
  			<< " policy.server=" << policy.server << dendl;
  
      // standby?
      if (is_queued() && state == STATE_STANDBY && !policy.server)
        state = STATE_CONNECTING;
  
      // connect?
      if (state == STATE_CONNECTING) {
        ceph_assert(!policy.server);
        connect();
        continue;
      }
      // ...
    }  
    // ...
  }
  ```

  ```cpp
  int Pipe::connect() {
      // ...
    	set_socket_options();
  
    	{
      	entity_addr_t addr2bind = msgr->get_myaddr_legacy();
      	if (msgr->cct->_conf->ms_bind_before_connect && (!addr2bind.is_blank_ip())) {
        	addr2bind.set_port(0);
        	int r = ::bind(sd , addr2bind.get_sockaddr(), addr2bind.get_sockaddr_len());
        	if (r < 0) {
          	ldout(msgr->cct,2) << "client bind error " << ", " << cpp_strerror(errno) << dendl;
          	goto fail;
        	}
      	}
    	}
  
    	// 发起连接， 在 Server 端 Accepter 类对应的工作线程正阻塞在 accept 系统调用上，等待连接
    	ldout(msgr->cct,10) << "connecting to " << peer_addr << dendl;
      rc = ::connect(sd, peer_addr.get_sockaddr(), peer_addr.get_sockaddr_len());
      if (rc < 0) {
          int stored_errno = errno;
      	ldout(msgr->cct,2) << "connect error " << peer_addr
  	     	<< ", " << cpp_strerror(stored_errno) << dendl;
      	if (stored_errno == ECONNREFUSED) {
        	ldout(msgr->cct, 2) << "connection refused!" << dendl;
        	msgr->dispatch_queue.queue_refused(connection_state.get());
      	}
      	goto fail;
    	}
      // ...
  }
  ```

  client 端通过 `Pipe::connect`  函数, 会调用 connect 系统调用，尝试连接服务器端的监听地址。在通信的另一端，服务器端的 Accepter 线程正阻塞在 accept 系统调用上，等待 client 端的连接，一旦服务器端的 accept 函数返回，Accepter 中的线程就会调用 `add_accept_pipe`  函数来创建一个新的 Pipe，全权负责和 client 的通信。创建出来 Pipe 后， Accepter 继续循环等待连接。

  而 `SimpleMessenger::add_accept_pipe()` 中也创建了一个 Pipe ，启动 reader_thread 线程。

  ```cpp
  Pipe *p = new Pipe(this, Pipe::STATE_ACCEPTING, NULL);
  ```

  Pipe 的状态为 `STATE_ACCEPTING` 。在 `Pipe::reader()` 中，如果 Pipe 状态是 `STATE_ACCEPTING` ，会调用 `accept` 函数与 client 进行通信，创建会话。

  ```cpp
  void Pipe::reader()
  {
    pipe_lock.Lock();
  
    if (state == STATE_ACCEPTING) {
      accept();
      ceph_assert(pipe_lock.is_locked());
    }
  
    // ...
  }
  ```

  `Pipe::connect` 和 `Pipe::accept` 到底交换了哪些信息呢？

|         client          |          server          |
| :---------------------: | :----------------------: |
|         connect         |                          |
|                         |          accept          |
|                         |       send banner        |
|       recv banner       |                          |
|       send banner       |                          |
|                         |       recv banner        |
|                         |   send my address info   |
|   recv  address info    |                          |
|  send my address info   |                          |
|                         |    recv address info     |
| send connection message |                          |
|                         | reply connection message |
| connection successfully |                          |

关键代码

```cpp
int Pipe::connect() {
    // stop reader thread
    join_reader();
    // ...
    
    // create socket?
    sd = socket_cloexec(peer_addr.get_family(), SOCK_STREAM, 0); 
    set_socket_options();
    {
        // ...
        ::bind(sd, addr, len);
        // ...
    }
    // connect
    ::connect(sd, peeraddr, peeraddr_len);
    // ...
    // verify banner
    rc = tcp_read((char*)&banner, strlen(CEPH_BANNER));
    memset(&msg, 0, sizeof(msg));
    msgvec[0].iov_base = banner;
    msgvec[0].iov_len = strlen(CEPH_BANNER);
    msg.msg_iov = msgvec;
    msg.msg_iovlen = 1;
    msglen = msgvec[0].iov_len;
    // send banner
    rc = do_sendmsg(&msg, msglen);
    
    // identify peer
    // ...
    rc = tcp_read(addrbl.c_str(), addrbl.length());
    // ...
    memset(&msg, 0, sizeof(msg));
    msgvec[0].iov_base = myaddrbl.c_str();
    msgvec[0].iov_len = myaddrbl.length();
    msg.msg_iov = msgvec;
    msg.msg_iovlen = 1;
    msglen = msgvec[0].iov_len;
    // send addr
    rc = do_sendmsg(&msg, msglen);
    // ...
    
    // send connection message
    // ...
    memset(&msg, 0, sizeof(msg));
    msgvec[0].iov_base = (char*)&connect;
    msgvec[0].iov_len = sizeof(connect);
    msg.msg_iov = msgvec;
    msg.msg_iovlen = 1;
    msglen = msgvec[0].iov_len;
    if (authorizer) {
      msgvec[1].iov_base = authorizer->bl.c_str();
      msgvec[1].iov_len = authorizer->bl.length();
      msg.msg_iovlen++;
      msglen += msgvec[1].iov_len;
    }
    rc = do_sendmsg(&msg, msglen);
    // ...
    
    // read reply
    ceph_msg_connect_reply reply;
    rc = tcp_read((char*)&reply, sizeof(reply));
    // ...
    
    // verify reply
    // ...
}
```

```cpp
int Pipe::accept() {
    // ...
    // reset
    // ...
    set_socket_options();
    
    // send banner
    r = tcp_write(CEPH_BANNER, strlen(CEPH_BANNER));
    // ...
    // get peer name
    r = ::getpeername(sd, (sockaddr*)&ss, &len);
    socket_addr.set_sockaddr((sockaddr*)&ss);
    encode(socket_addr, addrs, 0);  // legacy
    // send addr
    r = tcp_write(addrs.c_str(), addrs.length());
    // ...
    
    // identify peer
    if (tcp_read(banner, strlen(CEPH_BANNER)) < 0) {
        ldout(msgr->cct,10) << "accept couldn't read banner" << dendl;
        goto fail_unlocked;
    }
    // ...
    
    // identify addr
    if (tcp_read(addrbl.c_str(), addrbl.length()) < 0) {
        ldout(msgr->cct,10) << "accept couldn't read peer_addr" << dendl;
        goto fail_unlocked;
    }
    // ...
    
    // read connect info
    while (1) {
        if (tcp_read((char*)&connect, sizeof(connect)) < 0) {
            ldout(msgr->cct,10) << "accept couldn't read connect" << dendl;
            goto fail_unlocked;
        }
        // ...
    }
    
    // send reply
    r = tcp_write((char*)&reply, sizeof(reply));
    // ...
}
```

banner 是一个常量

```cpp
#define CEPH_BANNER "ceph v027"
```

最重要的逻辑在 connect message 的交互。这段代码的主要用途在于，服务端会校验这些连接信息并确保面向这个地址的连接只有一条。

```cpp
struct ceph_msg_connect {
	__le64 features;     /* supported feature bits */
	__le32 host_type;    /* CEPH_ENTITY_TYPE_* */
	__le32 global_seq;   /* count connections initiated by this host */
	__le32 connect_seq;  /* count connections initiated in this session */
	__le32 protocol_version;
	__le32 authorizer_protocol;
	__le32 authorizer_len;
	__u8  flags;         /* CEPH_MSG_CONNECT_* */
} __attribute__ ((packed));

struct ceph_msg_connect_reply {
	__u8 tag;
	__le64 features;     /* feature bits for this session */
	__le32 global_seq;
	__le32 connect_seq;
	__le32 protocol_version;
	__le32 authorizer_len;
	__u8 flags;
} __attribute__ ((packed));
```

对于 `connect` 函数， 首先查找是否已经存在负责和 client 端通信的 Pipe。

```cpp
existing = msgr->_lookup_pipe(peer_addr);
```

如果不存在已有的 Pipe, 对 connect_seq 是否大于0 分为两种情况处理。

```cpp
if (existing) {
    // ...
} else if (connect.connect_seq > 0) {
    // we reset, and they are opening a new session
    ldout(msgr->cct,0) << "accept we reset (peer sent cseq " << connect.connect_seq << "), sending RESETSESSION" << dendl;
    msgr->lock.Unlock();
    reply.tag = CEPH_MSGR_TAG_RESETSESSION;
    goto reply;   
} else {
    // new session
    ldout(msgr->cct,10) << "accept new session" << dendl;
    existing = NULL;
    goto open;
}
```

一般对于新连接， connect_seq 都等于 0， 这时候， 需要将状态转换为STATE_OPEN，同时发送回应给 client 端。

```cpp
 open:
  // open
  ceph_assert(pipe_lock.is_locked());
  connect_seq = connect.connect_seq + 1;
  peer_global_seq = connect.global_seq;
  ceph_assert(state == STATE_ACCEPTING);
  state = STATE_OPEN;
  ldout(msgr->cct,10) << "accept success, connect_seq = " << connect_seq << ", sending READY" << dendl;

  // send READY reply
  reply.tag = (reply_tag ? reply_tag : CEPH_MSGR_TAG_READY);
  reply.features = policy.features_supported;
  reply.global_seq = msgr->get_global_seq();
  reply.connect_seq = connect_seq;
  reply.flags = 0;
  reply.authorizer_len = authorizer_reply.length();
  if (policy.lossy)
    reply.flags = reply.flags | CEPH_MSG_CONNECT_LOSSY;

  connection_state->set_features((uint64_t)reply.features & (uint64_t)connect.features);
  ldout(msgr->cct,10) << "accept features " << connection_state->get_features() << dendl;

  session_security.reset(
      get_auth_session_handler(msgr->cct,
			       connect.authorizer_protocol,
			       session_key,
			       connection_state->get_features()));

  // notify
  msgr->dispatch_queue.queue_accept(connection_state.get());
  msgr->ms_deliver_handle_fast_accept(connection_state.get());

  // ok!
  if (msgr->dispatch_queue.stop)
    goto shutting_down;
  removed = msgr->accepting_pipes.erase(this);
  ceph_assert(removed == 1);
  register_pipe();
  msgr->lock.Unlock();
  pipe_lock.Unlock();

  r = tcp_write((char*)&reply, sizeof(reply));
  if (r < 0) {
    goto fail_registered;
  }

  if (reply.authorizer_len) {
    r = tcp_write(authorizer_reply.c_str(), authorizer_reply.length());
    if (r < 0) {
      goto fail_registered;
    }
  }

  if (reply_tag == CEPH_MSGR_TAG_SEQ) {
    if (tcp_write((char*)&existing_seq, sizeof(existing_seq)) < 0) {
      ldout(msgr->cct,2) << "accept write error on in_seq" << dendl;
      goto fail_registered;
    }
    if (tcp_read((char*)&newly_acked_seq, sizeof(newly_acked_seq)) < 0) {
      ldout(msgr->cct,2) << "accept read error on newly_acked_seq" << dendl;
      goto fail_registered;
    }
  }

  pipe_lock.Lock();
  discard_requeued_up_to(newly_acked_seq);
  if (state != STATE_CLOSED) {
    ldout(msgr->cct,10) << "accept starting writer, state " << get_state_name() << dendl;
    start_writer();
  }
  ldout(msgr->cct,20) << "accept done" << dendl;

  maybe_start_delay_thread();

  return 0;   // success.
```

如果 client 端发过来的 connect_seq 不等于 0， 并且在服务器端又找不到与 client 通信的 Pipe，则说明服务端 reset 了，需要把这个情况通过 CEPH_MSGR_TAG_RESETSESSION tag 告知 client 端，client 端收到这个 tag 后，会执行 `was_session_reset`  函数, 然后将 connect_seq 设置为 0，然后重新发送 connect_message。

```cpp
if (reply.tag == CEPH_MSGR_TAG_RESETSESSION) {
    ldout(msgr->cct,0) << "connect got RESETSESSION" << dendl;
    was_session_reset();
    cseq = 0;
    pipe_lock.Unlock();
    continue;
}
```

```cpp
void Pipe::was_session_reset()
{
  ceph_assert(pipe_lock.is_locked());

  ldout(msgr->cct,10) << "was_session_reset" << dendl;
  in_q->discard_queue(conn_id);
  if (delay_thread)
    delay_thread->discard();
  discard_out_queue();

  msgr->dispatch_queue.queue_remote_reset(connection_state.get());

  randomize_out_seq();

  in_seq = 0;
  in_seq_acked = 0;
  connect_seq = 0;
}
```



##### 消息的发送

```cpp
SimpleMessenger::_send_message(Message *m, Connection *con) {
    // ...
    submit_message(m, static_cast<PipeConnection*>(con),
                   con->get_peer_addr(), con->get_peer_type(), false);
    // ...
}
```

```cpp
void SimpleMessenger::submit_message(Message *m, PipeConnection *con,
				     const entity_addr_t& dest_addr, int dest_type,
				     bool already_locked) {
    // ...
    if (con) {
        // ...
        pipe->_send(m);
        // ...
    }
    // ...
}
```

```cpp
void _send(Message *m) {
    ceph_assert(pipe_lock.is_locked());
    out_q[m->get_priority()].push_back(m);
    cond.Signal();
}
```

对于 Pipe 的 `_send` 函数而言，就是把消息放到 out_q 这个消息队列中。

在 Pipe 类中， out_q 被定义为一个优先队列。何时将队列中的消息发送出去？ Pipe 的写线程负责此事。

`Pipe::_send`  函数中的 `cond.Signal()` 会唤醒 Pipe 的写线程。

```cpp
void Pipe::writer() {
    pipe_lock.Lock();
    while (state != STATE_CLOSED) {
        // ...
        // grab outgoing message
        Message *m = _get_next_outgoing();
        // ...
        int rc = write_message(header, footer, blist);
        pipe_lock.Lock();
        // ...
        cond.Wait(pipe_lock);
    }
    // ...
}
```



##### 消息的接收 

```cpp
void Pipe::reader() {
    // ...
    while (state != STATE_CLOSED &&state != STATE_CONNECTING) {
        // ...
        // read tag
        if (tcp_read((char*)&tag, 1) < 0) {
            pipe_lock.Lock();
            ldout(msgr->cct,2) << "reader couldn't read tag, " << cpp_strerror(errno) << dendl;
            fault(true);
            continue;
        }
        
        if (tag == CEPH_MSGR_TAG_KEEPALIVE) {
            ldout(msgr->cct,2) << "reader got KEEPALIVE" << dendl;
            pipe_lock.Lock();
            connection_state->set_last_keepalive(ceph_clock_now());
            continue;
        }
        // ...
        
        else if (tag == CEPH_MSGR_TAG_MSG) {
            ldout(msgr->cct,20) << "reader got MSG" << dendl;
            Message *m = 0;
            // 将消息读取到 m 中。
            int r = read_message(&m, auth_handler.get());
            
            pipe_lock.Lock();
            // ...
            
            // get_seq 小于当前的 in_seq, 说明收到了老消息，直接丢弃
            if (m->get_seq() <= in_seq) {
                ldout(msgr->cct,0) << "reader got old message "
                    << m->get_seq() << " <= " << in_seq << " " << m << " " << *m
                    << ", discarding" << dendl;
                in_q->dispatch_throttle_release(m->get_dispatch_throttle_size());
                m->put();
                if (connection_state->has_feature(CEPH_FEATURE_RECONNECT_SEQ) &&
                    msgr->cct->_conf->ms_die_on_old_message)
                    ceph_abort_msg("old msgs despite reconnect_seq feature");
                continue;
            }
            // 如果 get_seq 大于当前的 in_seq, 说明有之前的消息没有收到
            if (m->get_seq() > in_seq + 1) {
                ldout(msgr->cct,0) << "reader missed message?  skipped from seq "
                    << in_seq << " to " << m->get_seq() << dendl;
                if (msgr->cct->_conf->ms_die_on_skipped_message)
                    ceph_abort_msg("skipped incoming seq");
            }
            m->set_connection(connection_state.get());
            
            // note last received message.
            in_seq = m->get_seq();
            
            cond.Signal();  // wake up writer, to ack this
            in_q->fast_preprocess(m);
            
            // 此处分为三种情况
            // 1. 正常处理，将消息放入 in_q 这个 DispatchQueue, DispatchQueue关联的线程会从该优先队列中取出消息处理。
            // 2. fast_dispatch, 直接在 reader 线程中处理，如以上的 CEPH_MSGR_TAG_ACK
            // 3. 延迟发送
            if (delay_thread) {
                utime_t release;
                if (rand() % 10000 < msgr->cct->_conf->ms_inject_delay_probability * 10000.0) {
                    release = m->get_recv_stamp();
                    release += msgr->cct->_conf->ms_inject_delay_max * (double)(rand() % 10000) / 10000.0;
                    lsubdout(msgr->cct, ms, 1) << "queue_received will delay until " << release << " on " << m << " " << *m << dendl;
                }
                delay_thread->queue(release, m);
            } else {
                if (in_q->can_fast_dispatch(m)) {
                    reader_dispatching = true;
                    pipe_lock.Unlock();
                    in_q->fast_dispatch(m);
                    pipe_lock.Lock();
                    reader_dispatching = false;
                    if (state == STATE_CLOSED ||notify_on_dispatch_done) { 
                        // there might be somebody waiting
                        notify_on_dispatch_done = false;
                        cond.Signal();
                    }
                } else {
                    in_q->enqueue(m, m->get_priority(), conn_id);
                }
            }
            // ...
        }
        // ...
}
```

##### 消息接受及处理

消息接收及处理部分，涉及一个重要数据结构，`DispatchQueue in_q` 

**DispatchQueue**

```cpp
class DispatchQueue {
    class QueueItem {
        // ...
    };

    // ...
    
    PrioritizedQueue<QueueItem, uint64_t> mqueue;

    // ...
    
    class DispatchThread : public Thread {
        DispatchQueue *dq;
     public:
        explicit DispatchThread(DispatchQueue *dq) : dq(dq) {}
        void *entry() override {
            dq->entry();
            return 0;
        }
    } dispatch_thread;
    
    // ...
};
```

DispatchQueue 中， mqueue 是一个优先队列 PrioritizedQueue，处理的时候，优先处理优先级高的消息。所以，`in_q->enqueue` 调用的时候，需指定优先级，如 `Pipe::reader` 中所示。

Dispatch 的 `enqueue` 函数。

```cpp
void DispatchQueue::enqueue(const Message::ref& m, int priority, uint64_t id)
{
  Mutex::Locker l(lock);
  if (stop) {
    return;
  }
  ldout(cct,20) << "queue " << m << " prio " << priority << dendl;
  add_arrival(m);
  if (priority >= CEPH_MSG_PRIO_LOW) {
    mqueue.enqueue_strict(id, priority, QueueItem(m));
  } else {
    mqueue.enqueue(id, priority, m->get_cost(), QueueItem(m));
  }
  cond.Signal();
}
```

消息入队，然后通过 `cond.Signal` 唤醒别的线程。

```cpp
// This function delivers incoming messages to the Messenger.
void DispatchQueue::entry() {
    lock.Lock();
    while (true) {
        while (!mqueue.empty()) {
            QueueItem qitem = mqueue.dequeue();
            if (!qitem.is_code())
                remove_arrival(qitem.get_message());
            lock.Unlock();
            
            if (qitem.is_code()) {
                if (cct->_conf->ms_inject_internal_delays &&
                    cct->_conf->ms_inject_delay_probability &&
                    (rand() % 10000)/10000.0 < cct->_conf->ms_inject_delay_probability) 
                {
                    utime_t t;
                    t.set_from_double(cct->_conf->ms_inject_internal_delays);
                    ldout(cct, 1) << "DispatchQueue::entry  inject delay of " << t
                        << dendl;
                    t.sleep();
                }
                switch (qitem.get_code()) {
                    case D_BAD_REMOTE_RESET:
                        msgr->ms_deliver_handle_remote_reset(qitem.get_connection());
                        break;
                    case D_CONNECT:
                        msgr->ms_deliver_handle_connect(qitem.get_connection());
                        break;
                    case D_ACCEPT:
                        msgr->ms_deliver_handle_accept(qitem.get_connection());
                        break;
                    case D_BAD_RESET:
                        msgr->ms_deliver_handle_reset(qitem.get_connection());
                        break;
                    case D_CONN_REFUSED:
                        msgr->ms_deliver_handle_refused(qitem.get_connection());
                        break;
                    default:
                        ceph_abort();
                }
            } else {
                const Message::ref& m = qitem.get_message();
                if (stop) {
                    ldout(cct,10) << " stop flag set, discarding " << m << " " << *m << dendl;
                } else {
                    // 处理消息
                    uint64_t msize = pre_dispatch(m);
                    msgr->ms_deliver_dispatch(m);
                    post_dispatch(m, msize);
                }
            }
            lock.Lock();
        }
        if (stop)
            break;
        
        // wait for something to be put on queue
        // 如果队列为空，就会再次沉睡，Dispatch::enqueue 中的 cond.Signal 会将其唤醒。
        cond.Wait(lock);
    }
    lock.Unlock();
}
```

消息处理部分

```cpp
uint64_t msize = pre_dispatch(m);
msgr->ms_deliver_dispatch(m);
post_dispatch(m, msize);
```

其中 `ms_deliver_dispatch` 是最主要的函数。

```cpp
void ms_deliver_dispatch(const Message::ref &m) {
    m->set_dispatch_stamp(ceph_clock_now());
    for (const auto &dispatcher : dispatchers) {
        if (dispatcher->ms_dispatch2(m))
            return;
    }
    lsubdout(cct, ms, 0) << "ms_deliver_dispatch: unhandled message " << m << " " << *m << " from "
        << m->get_source_inst() << dendl;
    ceph_assert(!cct->_conf->ms_die_on_unhandled_msg);
}
```

其中 dispatcher 是真正处理消息的模块，在 SimpleMessenger 中有两个该类型的链表，分别为 dispatcher 和 fast_dispatcher。

消息处理流程图

![img](https://bean-li.github.io/assets/ceph_internals/simple_process_message.jpg)




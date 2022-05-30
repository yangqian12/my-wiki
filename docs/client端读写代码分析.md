### Client 端读写代码分析

---



```cpp
int RGWRados::Object::Read::read(int64_t ofs, int64_t end, bufferlist& bl) {
    // ...
    op.read(read_ofs, read_len, pbl, NULL);
    // ...
    
    r = state.cur_ioctx->operate(read_obj.oid, &op, NULL);
    // ...
}
```

```cpp
    struct Read {
      RGWRados::Object *source;

      struct GetObjState {
        map<rgw_pool, librados::IoCtx> io_ctxs;
        rgw_pool cur_pool;
        librados::IoCtx *cur_ioctx{nullptr};
        rgw_obj obj;
        rgw_raw_obj head_obj;
      } state;
```

`operate ` 方法是所有同步接口都需要走的一个方法，也是很关键的一个方法。

```cpp
int librados::IoCtx::operate(const std::string& oid, librados::ObjectReadOperation *o, bufferlist *pbl)
{
  object_t obj(oid);
  return io_ctx_impl->operate_read(obj, &o->impl->o, pbl);
}
```

该方法的具体实现封装在 `librados::IoCtxImpl::operate_read` 函数中。

```cpp
int librados::IoCtxImpl::operate_read(const object_t& oid,
				      ::ObjectOperation *o,
				      bufferlist *pbl,
				      int flags)
{
  // ...
    
  Objecter::Op *objecter_op = objecter->prepare_read_op(oid, oloc,
	                                      *o, snap_seq, pbl, flags,
	                                      onack, &ver);
  objecter->op_submit(objecter_op);

  // ...

  return r;
}
```

其中 op_submit 将初始化好的 op 对象进行提交。

调用流程为

` op_submit  ->  _op_submit_with_budget  ->  _op_submit ` 

`Objecter::_op_submit`  做了 client 端的主要工作。这个方法的开始会根据 op 的信息计算出这个对象的目标 osd 节点，当计算出目标节点后，使用 `get_session` 方法获取一个链接来和对应的 osd 通信。

```cpp
void Objecter::_op_submit(Op *op, shunique_lock& sul, ceph_tid_t *ptid) {
    // rwlock is locked
    
    // pick target
    OSDSession *s = NULL;
    
    bool check_for_latest_map = _calc_target(&op->target, nullptr) // 计算目标 OSD
        == RECALC_OP_TARGET_POOL_DNE;
    
    // 获取目标 OSD 连接
    int r = _get_session(op->target.osd, &s, sul);
    
    // ...
    
    _send_op_account(op);
    
    // send ?
    // ...
    
    // 对osdmap的同步与校验，防止 osdmap因某些问题与实际情况不符合,osdmap中记录了集群中 osd 的全部信息
    if (osdmap->get_epoch() < epoch_barrier) {
        ldout(cct, 10) << " barrier, paused " << op << " tid " << op->tid
            << dendl;
        op->target.paused = true;
        _maybe_request_map();
    }
    // ...
    
    _session_op_assign(s, op);

    // send op
    if (need_send) {
        _send_op(op);
    }
    // ...
}
```

最后，通过 `_send_op` 函数将 op 发送出去。

```cpp
void Objecter::_send_op(Op *op) {
    // rwlock is locked
    // op->session->lock is locked
    
    // ...
    MOSDOp *m = _prepare_osd_op(op);
    // ...
    op->session->con->send_message(m);
}
```

通过相关的链接将消息发送出去。这里有一个问题，在 ceph 中消息都是由 Messenger 发送出去的，而 ceph 的 Messenger 有 SimpleMessenger 和 AsyncMessenger 等。较新的版本中都是使用的 AsyncMessenger，但是到底是如何调用的？

```cpp
extern "C" int _rados_create(rados_t *pcluster, const char * const id)
{
  CephInitParameters iparams(CEPH_ENTITY_TYPE_CLIENT);
  if (id) {
    iparams.name.set(CEPH_ENTITY_TYPE_CLIENT, id);
  }
  CephContext *cct = rados_create_cct("", &iparams);

  tracepoint(librados, rados_create_enter, id);
  *pcluster = reinterpret_cast<rados_t>(new librados::RadosClient(cct));
  tracepoint(librados, rados_create_exit, 0, *pcluster);

  cct->put();
  return 0;
}
LIBRADOS_C_API_BASE_DEFAULT(rados_create);
```

该函数中实例化了一个 RadosClient。rados 的 connect 实际上就是 Radosclient 的 `connect()` ，而 messenger 就是这个过程初始化的。

```cpp
int librados::RadosClient::connect()
{
  int err;

  // already connected?
  if (state == CONNECTING)
    return -EINPROGRESS;
  if (state == CONNECTED)
    return -EISCONN;
  state = CONNECTING;

  if (cct->_conf->log_early &&
      !cct->_log->is_started()) {
    cct->_log->start();
  }

  {
    MonClient mc_bootstrap(cct);
    err = mc_bootstrap.get_monmap_and_config();
    if (err < 0)
      return err;
  }

  common_init_finish(cct);

  // get monmap
  err = monclient.build_initial_monmap();
  if (err < 0)
    goto out;

  err = -ENOMEM;
  messenger = Messenger::create_client_messenger(cct, "radosclient");
  if (!messenger)
    goto out;
  // ...
}
```

`Messenger::create_client_messenger` 

```cpp
Messenger *Messenger::create_client_messenger(CephContext *cct, string lname) {
    std::string public_msgr_type = cct->_conf->ms_public_type.empty() ? 
        cct->_conf.get_val<std::string>("ms_type") : cct->_conf->ms_public_type;
    auto nonce = ceph::util::generate_random_number<uint64_t>();
    return Messenger::create(cct, public_msgr_type, entity_name_t::CLIENT(),
                             std::move(lname), nonce, 0);
}
```

```cpp
Messenger *Messenger::create(CephContext *cct, const string &type,
			     entity_name_t name, string lname,
			     uint64_t nonce, uint64_t cflags) {
    int r = -1;
    if (type == "random") {
        r = ceph::util::generate_random_number(0, 1);
    }
    if (r == 0 || type == "simple")
        return new SimpleMessenger(cct, name, std::move(lname), nonce);
    else if (r == 1 || type.find("async") != std::string::npos)
        return new AsyncMessenger(cct, name, type, std::move(lname), nonce);
#ifdef HAVE_XIO
    else if ((type == "xio") &&
             cct->check_experimental_feature_enabled("ms-type-xio"))
        return new XioMessenger(cct, name, std::move(lname), nonce, cflags);
#endif
    lderr(cct) << "unrecognized ms_type '" << type << "'" << dendl;
    return nullptr;
}
```



**Async**

select 和 poll 模型。select 会先阻塞，当内核中有数据准备好后，select 才会返回，这是后通知处理逻辑来对数据进行处理。而我们在源码目录下看到的 Epoll 与 Kqueue 其实就是更高级的 select，高级在哪呢？主要是这两种调用直接使用callback来代替了轮询。epoll可以理解为event poll，不同于 select 的忙轮询和无差别轮询，epoll 会把哪个流发生了怎样的 I/O 事件通知我们。

源码目录 `src/msg/async/` 目录

```cpp
// Event.h
class EventCenter;
class EventCallback {

 public:
  virtual void do_request(uint64_t fd_or_id) = 0;
  virtual ~EventCallback() {}       // we want a virtual destructor!!!
};

typedef EventCallback* EventCallbackRef;

struct FiredFileEvent {
  int fd;
  int mask;
};

class EventDriver {
 public:
  virtual ~EventDriver() {}       // we want a virtual destructor!!!
  virtual int init(EventCenter *center, int nevent) = 0;
  virtual int add_event(int fd, int cur_mask, int mask) = 0;
  virtual int del_event(int fd, int cur_mask, int del_mask) = 0;
  virtual int event_wait(vector<FiredFileEvent> &fired_events, struct timeval *tp) = 0;
  virtual int resize_events(int newsize) = 0;
  virtual bool need_wakeup() { return true; }
};
```

Event.h 头文件中包含了一个事件回调类。EventDriver 封装了不同操作系统之间的事件机制。EventEpoll.h，EventKqueue.h，EventSelect.h 等文件中包含了 EventDriver 的不同实现。

```cpp
class EventCenter;
```

该类的作用维护文件描述符然后操作注册的事件。



回到 `AsyncConnection::send_message` 

```cpp
int AsyncConnection::send_message(Message *m) {
    FUNCTRACE(async_msgr->cct);
    // logging ...
    
    // optimistic think it's ok to encode(actually may broken now)
    if (!m->get_priority())
        m->set_priority(async_msgr->get_default_send_priority());
    
    m->get_header().src = async_msgr->get_myname();
    m->set_connection(this);
    
    if (m->get_type() == CEPH_MSG_OSD_OP)
        OID_EVENT_TRACE_WITH_MSG(m, "SEND_MSG_OSD_OP_BEGIN", true);
    else if (m->get_type() == CEPH_MSG_OSD_OPREPLY)
        OID_EVENT_TRACE_WITH_MSG(m, "SEND_MSG_OSD_OPREPLY_BEGIN", true);
    
    if (async_msgr->get_myaddrs() == get_peer_addrs()) { //loopback connection
        ldout(async_msgr->cct, 20) << __func__ << " " << *m << " local" << dendl;
        std::lock_guard<std::mutex> l(write_lock);
        if (protocol->is_connected()) {
            dispatch_queue->local_delivery(m, m->get_priority());
        } else {
            ldout(async_msgr->cct, 10) << __func__ << " loopback connection closed."
                                 << " Drop message " << m << dendl;
            m->put();
        }
        return 0;
    }
    // ...
}
```

通过 `dispatch_queue->local_delivery()` 投递消息。

```cpp
void local_delivery(Message* m, int priority) {
    return local_delivery(Message::ref(m, false), priority); /* consume ref */
}
```

```cpp
void DispatchQueue::local_delivery(const Message::ref& m, int priority) {
    m->set_recv_stamp(ceph_clock_now());
    Mutex::Locker l(local_delivery_lock);
    if (local_messages.empty())
        local_delivery_cond.Signal();
    local_messages.emplace(m, priority);
    return;
}
```

`local_delivery` 中将消息放到了 local_messages 这个消息队列中。

```cpp
std::queue<pair<Message::ref, int>> local_messages;
```

`local_delivery` 中可以看到，当 local_messages 为空时，会唤醒别的线程。

```cpp
void DispatchQueue::run_local_delivery() {
  local_delivery_lock.Lock();
  while (true) {
    if (stop_local_delivery)
      break;
    if (local_messages.empty()) {
      local_delivery_cond.Wait(local_delivery_lock);
      continue;
    }
    auto p = std::move(local_messages.front());
    local_messages.pop();
    local_delivery_lock.Unlock();
    const Message::ref& m = p.first;
    int priority = p.second;
    fast_preprocess(m);
    if (can_fast_dispatch(m)) {
      fast_dispatch(m);
    } else {
      enqueue(m, priority, 0);
    }
    local_delivery_lock.Lock();
  }
  local_delivery_lock.Unlock();
}
```

在 `run_local_delivery` 中，主要处理消息的部分在 `fast_dispatch` 和 `enqueue` 操作上。`enqueue` 函数中最终会将消息放在 mqueue 这个优先队列中。Cond.Signal() 唤醒 `DispatchQueue::entry()` 相关线程处理消息。

```cpp
uint64_t msize = pre_dispatch(m);
msgr->ms_deliver_dispatch(m);
post_dispatch(m, msize);
```

```cpp
void ms_deliver_dispatch(const Message::ref &m) {
    m->set_dispatch_stamp(ceph_clock_now());    
    for (const auto &dispatcher : dispatchers) {        
        if (dispatcher->ms_dispatch2(m))           
            return;    
    }    
    lsubdout(cct, ms, 0) << "ms_deliver_dispatch: unhandled message " << m << " " 
        << *m << " from " << m->get_source_inst() << dendl;    
    ceph_assert(!cct->_conf->ms_die_on_unhandled_msg);
}
```

最后由 dispatcher 处理消息。



**回调机制**

```cpp
void AsyncConnection::connect(const entity_addrvec_t &addrs, int type,
                              entity_addr_t &target) {
    std::lock_guard<std::mutex> l(lock);
    set_peer_type(type);
    set_peer_addrs(addrs);
    policy = msgr->get_policy(type);
    target_addr = target;
    _connect();
}
```

查看 `_connect` 函数

```cpp
void AsyncConnection::_connect() {
    ldout(async_msgr->cct, 10) << __func__ << dendl;
    
    state = STATE_CONNECTING;
    protocol->connect();
    // rescheduler connection in order to avoid lock dep
    // may called by external thread(send_message)
    center->dispatch_event_external(read_handler);
}
```

EventCenter 类的作用是维护文件描述符然后操作注册的事件。看 `dispatch_event_external` 

```cpp
void EventCenter::dispatch_event_external(EventCallbackRef e)
{
  uint64_t num = 0;
  {
    std::lock_guard lock{external_lock};
    if (external_num_events > 0 && *external_events.rbegin() == e) {
      return;
    }
    external_events.push_back(e);
    num = ++external_num_events;
  }
  if (num == 1 && !in_thread())
    wakeup();

  ldout(cct, 30) << __func__ << " " << e << " pending " << num << dendl;
}
```

唤醒 epoll_wait() ?



查看 `AsyncMessenger` 的构造函数

```cpp
AsyncMessenger::AsyncMessenger(CephContext *cct, entity_name_t name,
                               const std::string &type, string mname, uint64_t _nonce)
  : SimplePolicyMessenger(cct, name,mname, _nonce),
    dispatch_queue(cct, this, mname),
    lock("AsyncMessenger::lock"),
    nonce(_nonce), need_addr(true), did_bind(false),
    global_seq(0), deleted_lock("AsyncMessenger::deleted_lock"),
    cluster_protocol(0), stopped(true)
{
  std::string transport_type = "posix";
  if (type.find("rdma") != std::string::npos)
    transport_type = "rdma";
  else if (type.find("dpdk") != std::string::npos)
    transport_type = "dpdk";

  auto single = &cct->lookup_or_create_singleton_object<StackSingleton>(
    "AsyncMessenger::NetworkStack::" + transport_type, true, cct);
  single->ready(transport_type);
  stack = single->stack.get();
  stack->start();
  local_worker = stack->get_worker();
  local_connection = new AsyncConnection(cct, this, &dispatch_queue,
					 local_worker, true, true);
  init_local_connection();
  reap_handler = new C_handle_reap(this);
  unsigned processor_num = 1;
  if (stack->support_local_listen_table())
    processor_num = stack->get_num_worker();
  for (unsigned i = 0; i < processor_num; ++i)
    processors.push_back(new Processor(this, stack->get_worker(i), cct));
}
```

```cpp
void NetworkStack::start()
{
  std::unique_lock<decltype(pool_spin)> lk(pool_spin);

  if (started) {
    return ;
  }

  for (unsigned i = 0; i < num_workers; ++i) {
    if (workers[i]->is_init())
      continue;
    std::function<void ()> thread = add_thread(i);
    spawn_worker(i, std::move(thread));
  }
  started = true;
  lk.unlock();

  for (unsigned i = 0; i < num_workers; ++i)
    workers[i]->wait_for_init();
}
```

```cpp
std::function<void ()> NetworkStack::add_thread(unsigned i)
{
  Worker *w = workers[i];
  return [this, w]() {
      char tp_name[16];
      sprintf(tp_name, "msgr-worker-%u", w->id);
      ceph_pthread_setname(pthread_self(), tp_name);
      const unsigned EventMaxWaitUs = 30000000;
      w->center.set_owner();
      ldout(cct, 10) << __func__ << " starting" << dendl;
      w->initialize();
      w->init_done();
      while (!w->done) {
        ldout(cct, 30) << __func__ << " calling event process" << dendl;

        ceph::timespan dur;
        int r = w->center.process_events(EventMaxWaitUs, &dur);
        if (r < 0) {
          ldout(cct, 20) << __func__ << " process events failed: "
                         << cpp_strerror(errno) << dendl;
          // TODO do something?
        }
        w->perf_logger->tinc(l_msgr_running_total_time, dur);
      }
      w->reset();
      w->destroy();
  };
}
```

```cpp
int EventCenter::process_events(unsigned timeout_microseconds,  ceph::timespan *working_dur) {
    // ...
    // if exists external events or poller, dont't block
    if (!blocking) {
        // ...
    } else {
        // ...
    }
    
    // ...
    vector<FiredFileEvent> fired_events;
    numevents = driver->event_wait(fired_events, &tv);
    // ...
    for (int j = 0; j < numevents; j++) {
        // ...
        cb->do_request(fired_events[j].fd);
        // ...
    }
    // ...
}
```

EventCallBack 的具体实现是什么？

回忆 AsyncConnection 的代码中注册了 read_handler 的回调（`_connect` 中)。这个回调在`AsyncConnection::AsyncConnection ` 中初始化。

```cpp
AsyncConnection::AsyncConnection(CephContext *cct, AsyncMessenger *m, DispatchQueue *q,
                                 Worker *w, bool m2, bool loca)
  : Connection(cct, m), delay_state(NULL), async_msgr(m), conn_id(q->get_id()),
    logger(w->get_perf_counter()),
    state(STATE_NONE), port(-1),
    dispatch_queue(q), recv_buf(NULL),
    recv_max_prefetch(std::max<int64_t>(msgr->cct->_conf->ms_tcp_prefetch_max_size, TCP_PREFETCH_MIN_SIZE)),
    recv_start(0), recv_end(0),
    last_active(ceph::coarse_mono_clock::now()),
    connect_timeout_us(cct->_conf->ms_connection_ready_timeout*1000*1000),
    inactive_timeout_us(cct->_conf->ms_connection_idle_timeout*1000*1000),
    msgr2(m2), state_offset(0),
    worker(w), center(&w->center),read_buffer(nullptr)
{
  // ...
  read_handler = new C_handle_read(this);
  write_handler = new C_handle_write(this);
  write_callback_handler = new C_handle_write_callback(this);
  wakeup_handler = new C_time_wakeup(this);
  tick_handler = new C_tick_wakeup(this);
  // ...
}
```

由此可知，do_request 是 C_handle_read 的实现。

```cpp
class C_handle_read : public EventCallback {
  AsyncConnectionRef conn;
 public:
  explicit C_handle_read(AsyncConnectionRef c): conn(c) {}
  void do_request(uint64_t fd_or_id) override {
    conn->process();
  }
};
```

```cpp
void AsyncConnection::process() {
    // ...
    protocol->read_event();
}
```

...

消息发送之后，client 端需要等待消息处理的结果，在之前的代码中，体现在 `librados::IoCtxImpl::operate` 中。

```cpp
int librados::IoCtxImpl::operate(const object_t& oid, ::ObjectOperation *o,
				 ceph::real_time *pmtime, int flags)
{
  // ...
  Context *oncommit = new C_SafeCond(&mylock, &cond, &done, &r);

  int op = o->ops[0].op.op;
  Objecter::Op *objecter_op = objecter->prepare_mutate_op(oid, oloc,
							  *o, snapc, ut, flags,
							  oncommit, &ver);
  objecter->op_submit(objecter_op);

  mylock.Lock();
  while (!done)
    cond.Wait(mylock);
  mylock.Unlock();
  ldout(client->cct, 10) << "Objecter returned from "
	<< ceph_osd_op_name(op) << " r=" << r << dendl;

  set_sync_op_version(ver);

  return r;
}
```

```cpp
class C_SafeCond : public Context {
  Mutex *lock;    ///< Mutex to take
  Cond *cond;     ///< Cond to signal
  bool *done;     ///< true after finish() has been called
  int *rval;      ///< return value (optional)
public:
  C_SafeCond(Mutex *l, Cond *c, bool *d, int *r=0) : lock(l), cond(c), done(d), rval(r) {
    *done = false;
  }
  void finish(int r) override {
    lock->lock();
    if (rval)
      *rval = r;
    *done = true;
    cond->Signal();
    lock->unlock();
  }
};
```

根据我们之前代码的分析，肯定需要有一个操作，来调用这个 finish 方法，从而触发条件变量让这个操作结束。那么到底在哪里调用这个方法？

在 `RadosClient::connect()` 中有一些初始化的操作

```cpp
 
objecter = new (std::nothrow) Objecter(cct, messenger, &monclient,
			  &finisher,
			  cct->_conf->rados_mon_op_timeout,
			  cct->_conf->rados_osd_op_timeout);

```

这里的 objecter 是一个 dispatcher，Objecter 继承 Dispatcher，用来处理网络消息。

```cpp
class Objecter : public md_config_obs_t, public Dispatcher {
public:
  // config observer bits
  const char** get_tracked_conf_keys() const override;
  void handle_conf_change(const ConfigProxy& conf,
                          const std::set <std::string> &changed) override;

public:
  Messenger *messenger;
  MonClient *monc;
  Finisher *finisher;
  // ...
};
```

查看它的 `ms_dispatch`  函数

```cpp
bool Objecter::ms_dispatch(Message *m)
{
  ldout(cct, 10) << __func__ << " " << cct << " " << *m << dendl;
  switch (m->get_type()) {
    // these we exlusively handle
  case CEPH_MSG_OSD_OPREPLY:
    handle_osd_op_reply(static_cast<MOSDOpReply*>(m));
    return true;
    // ...
  }
  return false;
}
```

`CEPH_MSD_OSD_OPREPLY`  用来处理 OSD 回应的消息。

```cpp
void Objecter::handle_osd_op_reply(MOSDOpReply *m)
{
      // ...
      // ...
      // do callbacks
    if (onfinish) {
        onfinish->complete(rc);
    }
    if (completion_lock.mutex()) {
        completion_lock.unlock();
    }

    m->put();
}
```

```cpp
  virtual void complete(int r) {
    finish(r);
    delete this;
  }
```

这个方法就会调用到前面回调类的finish方法，让条件变量触发从而结束整个写入的流程。
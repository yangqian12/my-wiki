### Ceph librados 与 OSDC

---



Librados 提供了Pool的创建、删除、对象的创建、删除等基本接口；Osdc则用于封装操作，计算对象的地址，发送请求和处理超时。

访问流程图

![这里写图片描述](https://img-blog.csdn.net/20171204102701858?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQvQ1NORF9QQU4=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/SouthEast)



根据配置文件调用Librados 创建一个 rados, 接下来为这个 rados 创建一个 radosclient， radosclient 包含3个主要模块(finisher、Messenger、Objector)。再根据 pool 创建对应的 ioctx，在 ioctx 中能够找到 radosclient。再调用 OSDC 生成对应的 OSD 请求，与 OSD 进行通信响应请求。

**Librados**

包含 Radosclient 和 IoctxImpl。Radosclient 处于最上层，是 librados 核心管理类，管理整个 rados 系统层面以及 pool 层面的管理。而 IoctxImpl 则对其中的一个 pool 进行管理，如对对象的读写的操作的控制等。

**radosclient**

```cpp
// radosclient.h
class librados::RadosClient : public Dispatcher {
    std::unique_ptr<CephContext, std::function<void(CephContext*)> > cct_deleter;
public:
    using Dispatcher::cct;
    const ConfigProxy& conf;  // 配置文件
private:
    enum {
        DISCONNECTED,
        CONNECTING,
        CONNECTED,
    } state;  // 网络连接状态
    
    MonClient monclient;
    MgrClient mgrclient;
    Messenger *messenger;  // 网络消息接口
    
    uint64_t instance_id;
    
    // ...
    
    Objecter *objecter;  // 用于发送封装好的 op 消息
    // ...
    
    Finisher finisher;  // 执行回调函数的类
    // ...
    // 创建一个 pool 相关的上下文信息
    int create_ioctx(const char *name, IoCtxImpl **io);
    int create_ioctx(int64_t, IoCtxImpl **io);
    // ...
};
```

其中一些函数

```cpp
int librados::RadosClient::connect() {
    // ...
    // get monmap
    err = monclient.build_initial_monmap();
    
    messenger = Messenger::create_client_messenger(cct, "radosclient"); // 创建通信模块
    // ...
    // 设置 policy 相关信息
    messenger>set_default_policy(Messenger::Policy::lossy_client(
        CEPH_FEATURE_OSDREPLYMUX));
    // ...
    // 创建 Objector 并初始化
    objecter = new (std::nothrow) Objecter(cct, messenger, &monclient,
                                           &finisher,
                                           cct->_conf->rados_mon_op_timeout,
                                           cct->_conf->rados_osd_op_timeout);
    // ...
    // 初始化 Monclient
    err = monclient.init();
    // ...
    // 定时器初始化
    timer.init();

    // finisher 对象初始化
    finisher.start();
    // ...
}
```

`create_ioctx` 用于创建一个pool相关的上下文信息  IoCtxImpl 对象。

```cpp
int librados::RadosClient::create_ioctx(const char *name, IoCtxImpl **io)
{
  int64_t poolid = lookup_pool(name);
  if (poolid < 0) {
    return (int)poolid;
  }

  *io = new librados::IoCtxImpl(this, objecter, poolid, CEPH_NOSNAP);
  return 0;
}
```

`mon_command` 用于处理 Monitor 相关命令

```cpp
int librados::RadosClient::mon_command(const vector<string>& cmd,
				       const bufferlist &inbl,
				       bufferlist *outbl, string *outs)
{
  C_SaferCond ctx;
  mon_command_async(cmd, inbl, outbl, outs, &ctx);
  return ctx.wait();
}
```

同样的 `osd_command` 用来处理 OSD 相关命令。

**IoctxImpl**

该类是 pool 的上下文信息，一个 pool 对应一个 IoctxImpl 对象。librados 中所有关于 IO 操作的 API 都设计在librados::IoCtx 中，接口的真正实现在 IoCtxImpl 中。处理过程如下：

（1） 把请求封装成 ObjectOperation 类。

（2） 把相关的 pool 信息添加到里面，封装成 Objector::Op 对象。

（3）调用相应的函数发送给 OSD。

（4） 操作完成后，调用相应的回调函数。

**AioCompletionImpl**

Aio 即 Async IO，AioCompletion 即 Async IO completion，也就是Async IO完成时的回调处理制作，librados 设计 AioCompletion 就是为了提供一种机制对 Aio 完成时结果码的处理。而处理函数则由使用者来实现。AioCompletion 是 librados 设计开放的库 API，真正的设计逻辑在 AioCompletionImpl中。

**read流程**

```cpp
int RGWRados::Object::Read::read(int64_t ofs, int64_t end, bufferlist& bl) {
    // ...
    ObjectReadOperation op;
    // ...
    op.read(read_ofs, read_len, pbl, NULL);  // 读函数
}
```

```cpp
void librados::ObjectReadOperation::read(size_t off, uint64_t len, bufferlist *pbl,
                                         int *prval) {
    ::ObjectOperation *o = &impl->o;
    o->read(off, len, pbl, prval, NULL);
}
```

```cpp
struct ObjectOperation {
    vector<OSDOp> ops;  // 封装的操作集合
    int flags;
    int priority;  // 优先级
    
    vector<bufferlist*> out_bl;  // 输出缓存队列
    vector<Context*> out_handler;  // 回调函数队列
    vector<int*> out_rval;  // 操作结果队列
    // ...
    void read(uint64_t off, uint64_t len, bufferlist *pbl, int *prval,
              Context* ctx) {
        bufferlist bl;
        add_data(CEPH_OSD_OP_READ, off, len, bl);
        // ...
    }
    // ...
}
```

```cpp
void add_data(int op, uint64_t off, uint64_t len, bufferlist& bl) {
    OSDOp& osd_op = add_op(op);
    // ...
}
```

**结构体 Osdop 封装对象的一个操作**

```cpp
struct OSDOp {
    ceph_osd_op op;
    sobject_t soid;
    
    bufferlist indata, outdata;
    errorcode32_t rval;
    // ...
}
```

**op_target 封装PG信息**

```cpp
//Object.h
class Objecter : public md_config_obs_t, public Dispatcher {
    // ...
    struct op_target_t {
        int flags = 0;
        
        epoch_t epoch = 0;  ///< latest epoch we calculated the mapping
        
        object_t base_oid;  // 读取的对象
        object_locator_t base_oloc;  // 对象的 pool 信息
        object_t target_oid;  // 最终读取的目标对象
        object_locator_t target_oloc;  // 最终该目标对象的 pool 信息
        
        ///< true if we are directed at base_pgid, not base_oid
        bool precalc_pgid = false;
        
        ///< true if we have ever mapped to a valid pool
        bool pool_ever_existed = false;
        
        ///< explcit pg target, if any
        pg_t base_pgid;
        
        pg_t pgid; ///< last (raw) pg we mapped to
        spg_t actual_pgid; ///< last (actual) spg_t we mapped to
        unsigned pg_num = 0; ///< last pg_num we mapped to
        unsigned pg_num_mask = 0; ///< last pg_num_mask we mapped to
        unsigned pg_num_pending = 0; ///< last pg_num we mapped to
        vector<int> up; ///< set of up osds for last pg we mapped to
        vector<int> acting; ///< set of acting osds for last pg we mapped to
        int up_primary = -1; ///< last up_primary we mapped to
        int acting_primary = -1;  ///< last acting_primary we mapped to
        int size = -1; ///< the size of the pool when were were last mapped
        int min_size = -1; ///< the min size of the pool when were were last mapped
        bool sort_bitwise = false; ///< whether the hobject_t sort order is bitwise
        bool recovery_deletes = false; ///< whether the deletes are performed during recovery instead of peering
        // ...
    }
}
```



op_target_t 封装了对象所在的 pg， 以及 pg 对应的 OSD 列表信息。

**op 封装操作信息**

```cpp
// Object.h
// Class Objecter
struct Op : public RefCountedObject {
    OSDSession *session;  // OSD 的相关 Session 信息，Session 是相关 connect 的信息
    int incarnation;

    op_target_t target;  // 地址信息

    ConnectionRef con;  // for rx buffer only
    uint64_t features;  // explicitly specified op features

    vector<OSDOp> ops;  // op操作

    snapid_t snapid;
    SnapContext snapc;
    ceph::real_time mtime;

    bufferlist *outbl;
    vector<bufferlist*> out_bl;
    vector<Context*> out_handler;
    vector<int*> out_rval;
    // ...
}
```

**分片 Striper**

文件到对象的映射，可能需要分片，使用 `Striper` 类来进行分片，并保存分片信息。

```cpp
class Striper {
    public:
    /*
     * map (ino, layout, offset, len) to a (list of) ObjectExtents (byte
     * ranges in objects on (primary) osds)
     */
    static void file_to_extents(CephContext *cct, const char *object_format,
                                const file_layout_t *layout,
                                uint64_t offset, uint64_t len,
                                uint64_t trunc_size,
                                map<object_t, vector<ObjectExtent> >& extents,
                                uint64_t buffer_offset=0);
     // ...
}
```

其中 ObjectExtent 保存对象内的分片信息

```cpp
// osd_types.h
class ObjectExtent {
 public:
  object_t    oid;       // object id
  uint64_t    objectno;
  uint64_t    offset;    // in object
  uint64_t    length;    // in object
  uint64_t    truncate_size;	// in object

  object_locator_t oloc;   // object locator (pool etc)

  vector<pair<uint64_t,uint64_t> >  buffer_extents;  // off -> len.  extents in buffer being mapped (may be fragmented bc of striping!)
  
  ObjectExtent() : objectno(0), offset(0), length(0), truncate_size(0) {}
  ObjectExtent(object_t o, uint64_t ono, uint64_t off, uint64_t l, uint64_t ts) :
    oid(o), objectno(ono), offset(off), length(l), truncate_size(ts) { }
};
```


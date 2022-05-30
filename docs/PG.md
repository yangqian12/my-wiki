### PG

---

Placement Groups 。即归置组。

对对象进行两级存储映射：*Objects -> PGs -> OSDs*

* 第一级映射是静态的，负责将任意前端类型的数据按照固定大小进行切割、编号后作为随机哈希函数输入，均匀映射至 PG，以实现负载均衡。

* 第二级映射通过 crush 算法，实现 PG 到 OSD 的映射

  

**PG 的意义**

​    PG 最引人关注的特性在于它可以在 OSD 之间自由的迁移，这是 ceph 赖以实现自动数据恢复、自动数据平衡等高级特性的基础。因为 PG 的数量远小于对象的数量，因此以 PG 为单位进行操作更具灵活性。

​    存储池中 PG 数目决定了其并发处理多个对象的能力。但是过多的 PG 会消耗大量 CPU，同时容易使得磁盘长期处于过载状态。



**术语**

* PGID。PG 全局唯一 id
* OS。 对象存储的种类，如 filestore ， bluestore
* Info。PG 内基本元数据信息。
* Authoritative History。 Peering 过程中进行数据同步的依据。
* PGBackend。 PG 后端。
* Acting Set。 指一个有序的 OSD 集合。
* Primary。 Acting Set 的第一个 OSD
* Peering
* Recovery
* Backfill。 回填
* PG Temp
* Stray

##### 

#### PG读写流程

​    OSD 绑定的 Public Messenger 收到客户端发送的读写请求后，通过 OSD 注册的回调函数 `ms_fast_dispatch` 快速进行派发。

* 基于消息创建一个 OP，用于对消息进行跟踪，并记录消息携带的 Epoch。
* 查找 OSD 关联的客户端上下文，将 op 加入其内部的 waiting_on_map 队列，获取 OSDMAP，并将其与 waiting_on_map 中的 op 一一进行比较，如果 OSD 当前的 OSDMap 的 Epoch 不小于 op 所携带的 Epoch，则进一步将其派发至 OSD 的 `op_shardedwq` 队列（OSD 内部的工作队列），否则直接终止派发。
* 此时将其加入 OSD 全局 `session_waiting_for_map` 集合，该集合汇集了当前所有需要等待 OSD 更新完OSDMap 之后才能继续处理的会话上下文。



**do_request**

​    完成对 PG 级别的检查。

**do_op**

​    进行对象级别的检查

**execute_ctx**

​    真正执行 op 的函数，为了保证数据的一致性，所以包含修改操作的 PG 会预先由 Primary 通过 `prepare_transaction` 封装为一个 PG 事务，然后由不同类型的 PGBackend 负责转化为 OS 能够识别的本地事务，最后在副本间分发与同步。



#### 事务准备

do_osd_ops


### ceph server 端读写源码

---

osd 的 server 端的启动都在 ceph_osd.cc 类中，这个类是 osd 进程的主类，osd 相关的初始化工作都是在这个类中进行的。而在 ceph_osd 这个主进程中处理 osd 相关请求的逻辑主要封装在 OSD 这个类中。OSD 同样继承 Dispatcher。

```cpp
class OSD : public Dispatcher, public md_config_obs_t {
    Mutex osd_lock;          // global lock
    SafeTimer tick_timer;    // safe timer (osd_lock)

    // Tick timer for those stuff that do not need osd_lock
    Mutex tick_timer_lock;
    SafeTimer tick_timer_without_osd_lock;
    std::string gss_ktfile_client{};
    // ...
}

```



**初始化**

```cpp
osd = new OSD(g_ceph_context,
              store,
              whoami,
              ms_cluster,
              ms_public,
              ms_hb_front_client,
              ms_hb_back_client,
              ms_hb_front_server,
              ms_hb_back_server,
              ms_objecter,
              &mc,
              data_path,
              journal_path);
int err = osd->pre_init();
```

查看 `ms_dispatch` 函数

```cpp
bool OSD::ms_dispatch(Message *m)
{
  dout(20) << "OSD::ms_dispatch: " << *m << dendl;
  if (m->get_type() == MSG_OSD_MARK_ME_DOWN) {
    service.got_stop_ack();
    m->put();
    return true;
  }

  // lock!

  osd_lock.Lock();
  if (is_stopping()) {
    osd_lock.Unlock();
    m->put();
    return true;
  }

  do_waiters();
  _dispatch(m);

  osd_lock.Unlock();

  return true;
}
```

`ms_dispatch` 之中主要处理的是一些 PG 或者 OSD 级的消息，没发现对于请求处理方面的逻辑。

再看 `ms_fast_dispatch` 

```cpp
void OSD::ms_fast_dispatch(Message *m)
{
  FUNCTRACE(cct);
  if (service.is_stopping()) {
    m->put();
    return;
  }

  // peering event?
  switch (m->get_type()) {
  case CEPH_MSG_PING:
    dout(10) << "ping from " << m->get_source() << dendl;
    m->put();
    return;
    // ...
  }

  OpRequestRef op = op_tracker.create_request<OpRequest, Message*>(m);
  {
  // ...

  if (m->trace)
    op->osd_trace.init("osd op", &trace_endpoint, &m->trace);

  // note sender epoch, min req's epoch
  op->sent_epoch = static_cast<MOSDFastDispatchOp*>(m)->get_map_epoch();
  op->min_epoch = static_cast<MOSDFastDispatchOp*>(m)->get_min_epoch();
  ceph_assert(op->min_epoch <= op->sent_epoch); // sanity check!

  service.maybe_inject_dispatch_delay();

  if (m->get_connection()->has_features(CEPH_FEATUREMASK_RESEND_ON_SPLIT) ||
      m->get_type() != CEPH_MSG_OSD_OP) {
    // queue it directly
    enqueue_op(
      static_cast<MOSDFastDispatchOp*>(m)->get_spg(),
      std::move(op),
      static_cast<MOSDFastDispatchOp*>(m)->get_map_epoch());
  } else {
    auto priv = m->get_connection()->get_priv();
    if (auto session = static_cast<Session*>(priv.get()); session) {
      std::lock_guard l{session->session_dispatch_lock};
      op->get();
      session->waiting_on_map.push_back(*op);
      OSDMapRef nextmap = service.get_nextmap_reserved();
        
      dispatch_session_waiting(session, nextmap);
        
      service.release_map(nextmap);
    }
  }
  OID_EVENT_TRACE_WITH_MSG(m, "MS_FAST_DISPATCH_END", false); 
}
```

```cpp
void OSD::dispatch_session_waiting(SessionRef session, OSDMapRef osdmap)
{
  ceph_assert(session->session_dispatch_lock.is_locked());

  auto i = session->waiting_on_map.begin();
  while (i != session->waiting_on_map.end()) {
    OpRequestRef op = &(*i);
    ceph_assert(ms_can_fast_dispatch(op->get_req()));
    const MOSDFastDispatchOp *m = static_cast<const MOSDFastDispatchOp*>(
      op->get_req());
    if (m->get_min_epoch() > osdmap->get_epoch()) {
      break;
    }
    session->waiting_on_map.erase(i++);
    op->put();

    spg_t pgid;
    if (m->get_type() == CEPH_MSG_OSD_OP) {
      pg_t actual_pgid = osdmap->raw_pg_to_pg(
	static_cast<const MOSDOp*>(m)->get_pg());
      if (!osdmap->get_primary_shard(actual_pgid, &pgid)) {
	continue;
      }
    } else {
      pgid = m->get_spg();
    }
    enqueue_op(pgid, std::move(op), m->get_map_epoch());
  }

  if (session->waiting_on_map.empty()) {
    clear_session_waiting_on_map(session);
  } else {
    register_session_waiting_on_map(session);
  }
}

```

这里找到了 op 入队的操作。osd端接到消息之后会将消息入队缓存，在初始化时应该有线程启动，会一直从这个队列中获取op请求来进行处理。这个线程时从哪个地方初始化并一步一步调用到出队这个方法的，这里我们暂时不考虑，我们直接从出队列这个方法来看。

```cpp
void OSD::dequeue_op(PGRef pg, OpRequestRef op, ThreadPool::TPHandle &handle) {
    // ...
    op->mark_reached_pg();
    op->osd_trace.event("dequeue_op");
    
    pg->do_request(op, handle);
    
    // finish
    dout(10) << "dequeue_op " << op << " finish" << dendl;OID_EVENT_TRACE_WITH_MSG(
        op->get_req(), "DEQUEUE_OP_END", false);
}
```

```cpp
void PrimaryLogPG::do_request(OpRequestRef& op, ThreadPool::TPHandle &handle) {
    // ...
    switch (msg_type) {
        case CEPH_MSG_OSD_OP:
        case CEPH_MSG_OSD_BACKOFF:
            // ...
            switch (msg_type) {
                case CEPH_MSG_OSD_OP:
                    // verify client features
                    if ((pool.info.has_tiers() || pool.info.is_tier()) &&
                        !op->has_feature(CEPH_FEATURE_OSD_CACHEPOOL)) {
                        osd->reply_op_error(op, -EOPNOTSUPP);
                        return;
                    }
                    do_op(op);
                    break;
                case CEPH_MSG_OSD_BACKOFF:
                    // object-level backoff acks handled in osdop context
                    handle_backoff(op);
                    break;
            }
            break;
        // ...
    }     
}
```

这个 switch 根据 op 的不同类型对数据进行了不同的处理。在这个流程中有两个关键的逻辑调用，一个是 reply_op_error 即如果出错了，向 client 端返回响应。

```cpp
void OSDService::reply_op_error(OpRequestRef op, int err, eversion_t v,
                                version_t uv)
{
  const MOSDOp *m = static_cast<const MOSDOp*>(op->get_req());
  ceph_assert(m->get_type() == CEPH_MSG_OSD_OP);
  int flags;
  flags = m->get_flags() & (CEPH_OSD_FLAG_ACK|CEPH_OSD_FLAG_ONDISK);

  MOSDOpReply *reply = new MOSDOpReply(m, err, osdmap->get_epoch(), flags, true);
  reply->set_reply_versions(v, uv);
  m->get_connection()->send_message(reply);
}
```

另一个关键流程是 `do_op` 。

```cpp
void PrimaryLogPG::do_op(OpRequestRef& op) {
    // 校验...
    // ...
    // op 被封装成一个新的对象
    OpContext *ctx = new OpContext(op, m->get_reqid(), &m->ops, obc, this);
    // ...
    execute_ctx(ctx);
    // ...
}
```

```cpp
  struct OpContext {
    OpRequestRef op;
    osd_reqid_t reqid;
    vector<OSDOp> *ops;

    const ObjectState *obs; // Old objectstate
    const SnapSet *snapset; // Old snapset

    ObjectState new_obs;  // resulting ObjectState
    SnapSet new_snapset;  // resulting SnapSet (in case of a write)
    //pg_stat_t new_stats;  // resulting Stats
    object_stat_sum_t delta_stats;

    bool modify;          // (force) modification (even if op_t is empty)
    bool user_modify;     // user-visible modification
    bool undirty;         // user explicitly un-dirtying this object
    bool cache_evict;     ///< true if this is a cache eviction
    bool ignore_cache;    ///< true if IGNORE_CACHE flag is set
    bool ignore_log_op_stats;  // don't log op stats
    bool update_log_only;
    // ...
  }
```

`execute_ctx` 

```cpp
void PrimaryLogPG::execute_ctx(OpContext *ctx) {
    // ...
    int result = prepare_transaction(ctx);
    // ...
    if (ctx->update_log_only) {
        // ...
        // 在处理流程中如果遇到异常，就构造MOSDOpReply对象，将结果记录到这个对象中去，在合适的时候将这个reply发送出去
        MOSDOpReply *reply = ctx->reply;
        // ...
        record_write_error(op, soid, reply, result);
        // ...
    }
    // ...
    osd->send_message_osd_client(reply, m->get_connection());
    // ...
    // 写副本
    ceph_tid_t rep_tid = osd->get_tid();
    
    RepGather *repop = new_repop(ctx, obc, rep_tid);
    
    issue_repop(repop, ctx); // 副本操作
    eval_repop(repop);  // 处理返回消息
    repop->put();
}
```

写相关逻辑

```cpp
void PrimaryLogPG::issue_repop(RepGather *repop, OpContext *ctx) {
    // ...
    Context *on_all_commit = new C_OSD_RepopCommit(this, repop); // 回调
    // ...
    pgbackend->submit_transaction(
    soid,
    ctx->delta_stats,
    ctx->at_version,
    std::move(ctx->op_t),
    pg_trim_to,
    min_last_complete_ondisk,
    ctx->log,
    ctx->updated_hset_history,
    on_all_commit,
    repop->rep_tid,
    ctx->reqid,
    ctx->op);
}
```

这个操作向各个副本发送同步请求。

跳回到 `prepare_transaction` 

```cpp
int PrimaryLogPG::prepare_transaction(OpContext *ctx) {
    // ...
    
    int result = do_osd_ops(ctx, *ctx->ops);
    // ...
}
```

`do_osd_ops` 是 OSD 的关键处理类。里面对不同类型的消息进行处理。如下

```cpp
int PrimaryLogPG::do_osd_ops(OpContext *ctx, vector<OSDOp>& ops) {
    // ...
    case CEPH_OSD_OP_READ:
    ++ctx->num_read;
    tracepoint(osd, do_osd_op_pre_read, soid.oid.name.c_str(),
               soid.snap.val, oi.size, oi.truncate_seq, op.extent.offset,
               op.extent.length, op.extent.truncate_size,
               op.extent.truncate_seq);
    if (op_finisher == nullptr) {
        if (!ctx->data_off) {
            ctx->data_off = op.extent.offset;
        }
        result = do_read(ctx, osd_op);
    } else {
        result = op_finisher->execute();
    }
    break;
    // ...
}
```

查看 `CEPH_OSD_OP_WRITE` 分支下的 `write` 方法。

```cpp
t->write(soid, op.extent.offset, op.extent.length, osd_op.indata, op.flags);
```

```cpp
void write(
    const hobject_t &hoid,         ///< [in] object to write
    uint64_t off,                  ///< [in] off at which to write
    uint64_t len,                  ///< [in] len to write from bl
    bufferlist &bl,                ///< [in] bl to write will be claimed to len
    uint32_t fadvise_flags = 0     ///< [in] fadvise hint
    ) {
    auto &op = get_object_op_for_modify(hoid);
    ceph_assert(!op.updated_snaps);
    ceph_assert(len > 0);
    ceph_assert(len == bl.length());
    op.buffer_updates.insert(
      off,
      len,
      ObjectOperation::BufferUpdate::Write{bl, fadvise_flags});
}
```

这个方法主要是向 transactions 中填充数据。prepare_transaction函数的最后，会调用finish_ctx函数，finish_ctx函数里就会调用ctx->log.push_back就会构造pg_log_entry_t插入到vector log里。

无论Primary还是Replica，他们在写入数据的时候，都分为两部，首先是写Journal日志，然后是数据落磁盘，而这个过程中Replica会给Primary发送两次消息，第一次消息在Journal日志写完之后发送，第二次消息在数据落盘后发送给Primary，在每个阶段Primary收到所有Replica的回应后，都会发相应的消息告知client。

关于这些状态，都在submit_transaction方法中记录

pgbackend 有两个具体实现类，一个是 `ECBackend`， 一个是 `ReplicatedBackend` 。从名字可以看出，`ECBackend` 是纠删码的实现， `ReplicatedBackend` 是副本的实现。一般情况下用 `ReplicatedBackend` 。

查看 `ReplicatedBackend::submit_transaction` 函数。中间有数据结构

```cpp
auto insert_res = in_progress_ops.insert(
    make_pair(tid,
              new InProgressOp(tid, on_all_commit,orig_op, at_version))
);
```

```cpp
struct InProgressOp : public RefCountedObject {
    ceph_tid_t tid;
    set<pg_shard_t> waiting_for_commit;
    Context *on_commit;
    OpRequestRef op;
    eversion_t v;
    InProgressOp(
      ceph_tid_t tid, Context *on_commit,
      OpRequestRef op, eversion_t v)
      : RefCountedObject(nullptr, 0),
	tid(tid), on_commit(on_commit),
	op(op), v(v) {}
    bool done() const {
      return waiting_for_commit.empty();
    }
};
```

这个结构用来记录之前提到的一些状态。

`waiting_for_commit` 维护了未完成写 journal 日志的 OSD 的集合。继续往后看 `submit_transaction` 的代码。

```cpp
issue_op(
    soid,
    at_version,
    tid,
    reqid,
    trim_to,
    at_version,
    added.size() ? *(added.begin()) : hobject_t(),
    removed.size() ? *(removed.begin()) : hobject_t(),
    log_entries,
    hset_history,
    &op,
    op_t);
```

`issue_op` 即向 Replica OSD 发送消息。

```cpp
void ReplicatedBackend::issue_op(
  const hobject_t &soid,
  const eversion_t &at_version,
  ceph_tid_t tid,
  osd_reqid_t reqid,
  eversion_t pg_trim_to,
  eversion_t pg_roll_forward_to,
  hobject_t new_temp_oid,
  hobject_t discard_temp_oid,
  const vector<pg_log_entry_t> &log_entries,
  boost::optional<pg_hit_set_history_t> &hset_hist,
  InProgressOp *op,
  ObjectStore::Transaction &op_t)
{
  if (parent->get_acting_recovery_backfill_shards().size() > 1) {
    if (op->op) {
      op->op->pg_trace.event("issue replication ops");
      ostringstream ss;
      set<pg_shard_t> replicas = parent->get_acting_recovery_backfill_shards();
      replicas.erase(parent->whoami_shard());
      ss << "waiting for subops from " << replicas;
      op->op->mark_sub_op_sent(ss.str());
    }

    // avoid doing the same work in generate_subop
    bufferlist logs;
    encode(log_entries, logs);

    for (const auto& shard : get_parent()->get_acting_recovery_backfill_shards()) {
      if (shard == parent->whoami_shard()) continue;
      const pg_info_t &pinfo = parent->get_shard_info().find(shard)->second;

      Message *wr;
      wr = generate_subop(
	  soid,
	  at_version,
	  tid,
	  reqid,
	  pg_trim_to,
	  pg_roll_forward_to,
	  new_temp_oid,
	  discard_temp_oid,
	  logs,
	  hset_hist,
	  op_t,
	  shard,
	  pinfo);
      if (op->op && op->op->pg_trace)
	wr->trace.init("replicated op", nullptr, &op->op->pg_trace);
      get_parent()->send_message_osd_cluster(
	  shard.osd, wr, get_osdmap_epoch());
    }
  }
}
```

其中， `send_message_osd_cluster` 为向 Replica OSD 发送消息的函数，消息通过 `generate_subop` 进行封装，然后通过 `send_message_osd_cluster` 发送出去。

```cpp
Message * ReplicatedBackend::generate_subop(...)
{
  int acks_wanted = CEPH_OSD_FLAG_ACK | CEPH_OSD_FLAG_ONDISK;
  // forward the write/update/whatever
  MOSDRepOp *wr = new MOSDRepOp(
    reqid, parent->whoami_shard(),
    spg_t(get_info().pgid.pgid, peer.shard),
    soid, acks_wanted,
    get_osdmap_epoch(),
    parent->get_last_peering_reset_epoch(),
    tid, at_version);

  // ship resulting transaction, log entries, and pg_stats
  if (!parent->should_send_op(peer, soid)) {
    ObjectStore::Transaction t;
    encode(t, wr->get_data());
  } else {
    encode(op_t, wr->get_data());
    wr->get_header().data_off = op_t.get_data_alignment();
  }

  wr->logbl = log_entries;

  if (pinfo.is_incomplete())
    wr->pg_stats = pinfo.stats;  // reflects backfill progress
  else
    wr->pg_stats = get_info().stats;

  wr->pg_trim_to = pg_trim_to;
  wr->pg_roll_forward_to = pg_roll_forward_to;

  wr->new_temp_oid = new_temp_oid;
  wr->discard_temp_oid = discard_temp_oid;
  wr->updated_hit_set_history = hset_hist;
  return wr;
}
```

消息为一个 `MOSDRepOp` 对象。

```cpp
MOSDRepOp(osd_reqid_t r, pg_shard_t from,
	    spg_t p, const hobject_t& po, int aw,
	    epoch_t mape, epoch_t min_epoch, ceph_tid_t rtid, eversion_t v)
    : MessageInstance(MSG_OSD_REPOP, HEAD_VERSION, COMPAT_VERSION),
	  map_epoch(mape),
      min_epoch(min_epoch),
      reqid(r),
      pgid(p),
      final_decode_needed(false),
      from(from),
      poid(po),
      acks_wanted(aw),
      version(v) {
          set_tid(rtid);
      }
```

我们可以看出这个消息的类型是 MSG_OSD_REPOP ，在 Primary 发送了这种消息之后，Replica OSD 会和Primary 前面的逻辑一样，进入到队列，然后从 osd.op_wq 中取出消息进行处理。当走到 do_request 函数之后，并没有机会执行 do_op，或者 do_sub_op 之类的函数，而是被 handle_message 函数拦截了，我们回过头来看一下 `do_request` 里的逻辑。

```cpp
  ceph_assert(is_peered() && flushes_in_progress == 0);
  if (pgbackend->handle_message(op))
    return;
```

`handle_message` 做了什么工作呢？

```cpp
bool PGBackend::handle_message(OpRequestRef op)
{
  switch (op->get_req()->get_type()) {
  case MSG_OSD_PG_RECOVERY_DELETE:
    handle_recovery_delete(op);
    return true;

  case MSG_OSD_PG_RECOVERY_DELETE_REPLY:
    handle_recovery_delete_reply(op);
    return true;

  default:
    break;
  }

  return _handle_message(op);
}
```

```cpp
bool ReplicatedBackend::_handle_message(OpRequestRef op)
{
    // ...
    case MSG_OSD_REPOP: {
        do_repop(op);
        return true;
    }
    // ...
}
```

`do_repop` 是Replica 节点处理消息的主要函数，逻辑和 Primary 节点相似。从回调函数可以看出该函数主要工作就是给主 OSD 发送消息。如 `ReplicatedBackend::repop_commit` 。

写入流程暂时到这。




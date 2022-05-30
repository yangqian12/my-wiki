### MapReduce

---

#### MapReduce 编程模型

以一组键值对作为输入，输出另一组键值对

用户通过编写 Map 函数和 Reduce 函数来指定所要进行的计算
$$
map (k1, v1)           -> list(k2, v2)
$$

$$
reduce(k2, list(v2))   ->list(v2)
$$

在实际的实现中， MapReduce 框架使用 Iterator 来代表作为输入的集合，主要是为了避免集合过大，无法被完整的放入内存中。

example

```cpp
map(String key, String value):
// key: document name
// value: document contents
for each word w in value:
EmitIntermediate(w, "1");
reduce(String key, Iterator values):
// key: a word
// values: a list of counts
int result = 0;
for each v in values:
result += ParseInt(v);
```



#### 计算执行过程

![](https://pic4.zhimg.com/80/v2-add6b28c0c1632fe764271b8ad7b14fb_720w.jpg)

计算流程

- 输入文件被分为 M 个 split，每个 split 的大小通常在 16~64 M 之间。
- M 个 Map 任务和 R 个 Reduce 任务，由 master 分配给空闲的 worker 。
- workers 读入对应的 split ，解析为输入键值并调用用户的 Map 函数。中间文件存在缓存区。
- Mapper 们周期性的将缓存区的中间文件存入磁盘， 根据 partition 函数（默认是 hash(key) mode R）将中间文件分割成 R 部分，然后向 master 报告位置和 size。
- master 将中间文件信息转发给 Reducer，Reducer 通过 rpc 读到这些文件后进行排序以便相同键值的文件连续分布。
- 调用 Reduce 函数，产生结果文件。

M 和 R 比集群中的 workers 数量多很多以便负载均衡。



#### MapReduce 容错机制

使用 GFS（google file system）提供的分布式原子文件读写操作

##### Worker failure

Master 向 worker 发送心跳，如果一段时间 worker 没有响应，则认为 worker 已经不可用。

失败的 worker 上的 Map 任务，无论是否完成，都需要 master 重新分配给其它的 worker 执行，因为 Map 任务只输出中间结果，已完成的 Reduce 任务可靠性由 GFS 提供，failure worker 上未完成的 Reduce 任务将被master 转发给其它 worker 完成。

##### Master failure

Master 会周期性的将当前集群的状态保存（checkpoint）写入到磁盘，重新启动后， master 进程可利用 checkpoint 恢复到上一次保存点的状态。

#####  Semantics in the Presence of Failures

一般情况下语义等同于顺序执行，在保证定义的 Map 和 Reduce 函数是确定的，但存在非确定性运算符情况下，会得到不同的执行结果。

#### Locality

##### 粒度

M 和 R 的数量应该远大于 workers 的数量。

##### 备份 tasks

如果集群中有某个 worker 花了特别长的时间来完成最后的几个 Map 或 Reduce 任务，整个 Map Reduce 计算任务的耗时就会被拖长， 这样的 worker 也就成了落后者（Straggler）。

MapReduce 在整个计算完成到一定程度时就会将剩余的任务进行备份，即同时将其分配给其他空闲 Worker 来执行，并在其中一个 Worker 完成后将该任务视作已完成。

#### 其它优化

##### Partitioning function

允许使用不同的 partition 函数

##### ordering guarantees

对 output data 进行排序

##### combiner function

在某些情形下，用户所定义的 Map 任务可能会产生大量重复的中间结果键。在这种情况下，Google MapReduce 系统允许用户声明在 Mapper 上执行的 Combiner 函数：Mapper 会使用由自己输出的 R 个中间结果 Partition 调用 Combiner 函数以对中间结果进行局部合并，减少 Mapper 和 Reducer 间需要传输的数据量。

##### 支持不同输入输出类型

##### 跳过 Bad records

##### 本地执行

MapReduce 采用 GFS 来保存输入和结果数据，因此 Master 在分配 Map 任务时会从 GFS 中读取各个 Block 的位置信息，并尽量将对应的 Map 任务分配到持有该 Block 的 Replica 的机器上；如果无法将任务分配至该机器，Master 也会利用 GFS 提供的机架拓扑信息将任务分配到较近的机器上。

##### 状态信息

##### 计数器


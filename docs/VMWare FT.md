### VMWare FT

---

#### 容错

主/备份方法

##### 保持同步的方法

- 状态转移。主服务器所有状态（包括内存、CPU 和 IO 设备）的变化发送给备份服务器。传输的数据量大，所需要的带宽非常大。
- 备份状态机。将备份状态机视为确定状态机。主/备服务器都以相同的初始状态启动，以相同的顺序执行输入的请求，然后得到相同的输出。方法较为复杂，但是传输的数据小的多。
- 采用备份状态机有一个问题。即对于真实物理机来讲，有些行为是**不确定的**（例如，中断，产生的随机数等）。VMware 的解决方法是将所有的操作虚拟化。并在 VM 和物理机之间加一个 Hypervisor（虚拟机管理程序）。通过 Logging channel 将所有操作从 primary 传递到 backup，然后 backup 进行 replay。从而实现两者的高度一致。

#### 架构

同一网络

共享磁盘

对于外界，它们只知道有一个 Primary VM 在工作，并且所有的输入输出都是由 Primary 处理。

##### replay 确定性重放

三点挑战：

- 正确捕获所有输入和不确定操作。保证在 backup 上的确定执行。
- 在 backup 上正确的执行所有的 input 和不确定操作。
- 在以上条件下，不牺牲性能。

保证不确定性操作的确定执行利用了 Log。primary 将操作写入log entry，log entry保存在内存中，通过 log channel 发送给 backup 并使其 replay。

VMWare 利用 Hypervisor 将所有操作虚拟化。

##### FT protocol

**Output Rule**

- primary 在所有关于本次 Output 的信息都发送给 backup 后（并且要**确保 backup 收到**，backup 会发送一个 ACK），才会把 output 发送给外界。
- primary 只是推迟将 output 发送给外界，不会暂停执行后面的任务（异步执行）

##### FT Logging buffers and channel

- primary 和 backup 它们各自会有一个 **log buffer** 缓存，primary 将产生的日志写入 log buffer，然后通过Logging channel 传到 backup 的 log buffer 中，backup 再从自己的缓存中读取日志并 replay。
- log buffer 满或 log buffer 空的情况。协调 primary 和 backup 的资源，使得 primary 产生 log 与 backup 消耗 log 保持同步。

##### 检测和故障响应

primary 与 backup 。heartbeat。

脑裂。test-and-set 原子操作。类似锁，获取 flag 后可对磁盘进行操作。

##### 虚拟机恢复

FT VMotion

它能够直接对一个 VM 进行克隆，并且在完成克隆后会建立好和 primary 的 Logging channel，被克隆一方是primary，另一个就是 backup。

此外，VMware vSphere 还实现了一个可以监控并管理集群资源的服务，可用于选取适合启动新虚拟机的宿主机。

#### design alternative

share storage | non-shared storage；

Disk reads on backup VM



#### VM-FT VS GFS

VM-FT 备份的是计算，GFS 备份的是存储

VM-FT 强一致，GFS 弱一致，GFS 更简单高效。
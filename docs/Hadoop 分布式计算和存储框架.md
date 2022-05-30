### Hadoop 分布式计算和存储框架

---

#### Hadoop 介绍

Hadoop 是一套开源的用于大规模数据集的分布式存储和处理的工具平台。用户可以在不了解分布式底层细节的情况下，开发分布式程序。充分利用集群的威力进行高速运算和存储。它最早由 Yahoo 的技术团队根据 Google 所发布的公开论文思想用 JAVA 语言开发，现在则隶属于 apache 基金会。

Hadoop 以分布式文件系统 hdfs 和 mapreduce 计算框架为核心，为用户提供了底层细节透明的分布式基础设施。hdfs 的高容错性、高伸缩性等优点，允许用户将 hadoop 部署在廉价的硬件上，构建分布式存储系统。mapreduce 分布式计算框架则允许用户在不了解分布式系统底层细节的情况下开发并行、分布式的应用程序，充分利用计算机资源，解决传统高性能单机无法解决的问题。

应用场景

- 大数据存储
- 日志处理
- 海量计算
- 数据分析
- 数据挖掘
- 机器学习等



#### 两大核心组件

##### mapreduce

​    MapReduce 是一种可用于数据处理的编程模型。MapReduce 是由 Google 设计，开发和使用的一个系统，相关的论文在 2004 年发表。MapReduce 的思想是，应用程序设计人员和分布式运算的使用者，只需要写简单的Map 函数和 Reduce 函数，而不需要知道任何有关分布式的事情，MapReduce 框架会处理剩下的事情。

**mapreduce 编程模型**

​    mapreduce 任务过程分为两个阶段：map 和 reduce，每阶段都以键值对作为输入输出。

​    map 

​	用户通过编写 Map 函数和 Reduce 函数来指定所要进行的计算
$$
map (k1, v1)           -> list(k2, v2)
$$

$$
reduce(k2, list(v2))   ->list(v2)
$$

​	在实际的实现中， MapReduce 框架使用 Iterator 来代表作为输入的集合，主要是为了避免集合过大，无法被完整的放入内存中。

**计算执行过程**

![](https://pic4.zhimg.com/80/v2-add6b28c0c1632fe764271b8ad7b14fb_720w.jpg)

计算流程

- 输入文件被分为 M 个 split，每个 split 的大小通常在 16~64 M 之间。
- M 个 Map 任务和 R 个 Reduce 任务，由 master 分配给空闲的 worker 。
- workers 读入对应的 split ，解析为输入键值并调用用户的 Map 函数。中间文件存在缓存区。
- Mapper 们周期性的将缓存区的中间文件存入磁盘， 根据 partition 函数（默认是 hash(key) mode R）将中间文件分割成 R 部分，然后向 master 报告位置和 size。
- master 将中间文件信息转发给 Reducer，Reducer 通过 rpc 读到这些文件后进行排序以便相同键值的文件连续分布。
- 调用 Reduce 函数，产生结果文件。

M 和 R 比集群中的 workers 数量多很多以便负载均衡。



##### hdfs

​    HDFS 是基于谷歌分布式文件系统 GFS 的论文用 java 语言实现的一个分布式文件系统，它采用分而治之的思想，将文件分布式的存放在大量独立的服务器上。hdfs 以流式数据访问模式来存储超大文件。

**一些基本概念**

**Block**

HDFS 中的存储单元是每个数据块 block，HDFS 默认的最基本的存储单位是 64M (在 Apache Hadoop 中默认是64M，Cloudera Hadoop 版本中默认是128M)的数据块。和普通的文件系统相同的是，HDFS 中的文件也是被分成 64M 一块的数据块存储的。不同的是，在 HDFS 中，如果一个文件大小小于一个数据块的大小，它是不需要占用整个数据块的存储空间的。（**最小化寻址开销**）

**NameNode**

元数据节点。该节点用来管理文件系统中的命名空间，是 master。该节点在硬盘上保存了命名空间镜像（namespace image）以及修改日志（edit log）。此外，NameNode 还保存了一个文件包括哪些数据块，分布在哪些数据节点上。然而，这些信息不存放在硬盘上，而是在系统启动的时候从数据节点(通过心跳）收集而成的。

**DataNode**

数据节点，hdfs 真正存储数据的地方。datanode 需要定期向 namenode 报告数据块信息。

**SecondaryNameNode**

从元数据节点。从元数据节点并不是 NameNode 出现问题时候的备用节点，它的主要功能是周期性的将NameNode 中的 namespace image 和 edit log 合并，以防 log 文件过大。此外，合并过后的namespace image文件也会在 Secondary NameNode 上保存一份，以防 NameNode 失败的时候，可以恢复。

##### HDFS 特点

1. 高容错和高可用。副本策略
2. 流式数据访问。
3. 弹性存储，支持大规模数据集
4. 简单一致性模型
5. 移动计算而非移动数据
6. 协议接口多样性
7. 多样的数据管理功能

#### 读写文件

**读文件**

![wxmp](https://www.yijiyong.com/assets/img/ds/hadoop/hdfsintro-2.png)

**写文件**

![wxmp](https://www.yijiyong.com/assets/img/ds/hadoop/hdfsintro-3.png)



#### Hadoop 支持 Tyds

**Tyds vs HDFS**

|                      | tyds                            | hdfs         |
| -------------------- | ------------------------------- | ------------ |
| metadata server      | 不存点单点故障                  | 存在单点故障 |
| 介面                 | POSIX                           | 不完全支持   |
| 副本抗灾性           | 是                              | 是           |
| 结构模式             | client/server                   | Master/Slave |
| 可扩展性             | 是                              | 是           |
| 高效性               | 高                              | 高           |
| 开发语言             | c++                             | java         |
| 是否基于本地文件系统 | filestore 基于，bluestore不基于 | 基于         |
| 文件系统划分         | object                          | block        |



**用 tyds 代替 hdfs**

1. 抽象文件系统

   hadoop 提供了抽象的文件系统接口，在实际读取或写入数据的时候，根据文件系统的具体实例来调用不同文件系统的接口，而 tydsfs 也同样提供了 java 接口，我们可以通过添加一个 TydsFileSystem 文件系统作为 hadoop 的后端存储系统。

   | 文件系统 | uri 方案 | java 实现                  | 描述                                           |
   | -------- | -------- | -------------------------- | ---------------------------------------------- |
   | Local    | file     | fs.LocalFileSystem         | 使用客户端校验和的本地磁盘文件系统。           |
   | HDFS     | hdfs     | hdfs.DistributedFileSystem | hadoop 的分布式文件系统，与 mapreduce 结合使用 |
   | ...      | ...      | ...                        | ...                                            |

   

2. FileSystem Java API

   **FileSystem**

   ```java
   public static FileSystem get(final URI uri, final Configuration conf,
           final String user) throws IOException, InterruptedException {...}
   ```

   Configuration 对象封装了客户端或服务器的配置，通过设置配置文件读取类路径来实现（如 etc/hadoop/core-site.xml)。 默认文件系统在 **core-site.xml** 中指定。

   ```java
   public class TydsFileSystem extends FileSystem {
       private TydsFsProto tyds = null;	// TydsFsproto 实例，用于调用 tydsfs 的接口
       
       public FSDataInputStream open(Path path, int bufferSize) throws IOException {...}
       // 重写 FileSystem 的 open 方法，打开对应路径的文件，返回 FSDataInputStream 类型的对象。
       
   	public FSDataOutputStream create(Path path, FsPermission permission, boolean overwrite, int bufferSize, short replication, long blockSize,Progressable progress) throws IOException {...}
       // 创建一个文件并且返回 FSDataOutputStream 类型的对象。
       ...
   }
   ```

   **read**

   有了文件系统实例后，就可以通过 `open ` 函数来获取输入流

   ```java
   public abstract FSDataInputStream open(Path f, int bufferSize)
       throws IOException;
   ```

   FSDataInputStream 对象

   FileSystem open 方法返回的对象，支持随机访问，可以从流的任意位置读取数据。

   **write**

   ```java
    public FSDataOutputStream create(Path f) throws IOException
   ```

   FSDataOutputStream 对象

   与 FSDataInputStream 类似，但是不支持在文件中定位，也就是说，只允许顺序写入，在文件末尾追加数据。相关对象还有 `write` 、`append` 方法。

3. tyds-hdfs adapter

   **TydsFileSystem** 

   该类继承于 Hadoop 的 FileSystem 接口，通过重写 FileSystem 抽象类的方法，实现 hdfs 中的一些功能。返回对应的 FSDataInputStream，FSDataOutputStream。

   **TYDSInputStream 和 TYDSOutputStream**

   hadoop 中数据的流动是通过 FSDataInputStream，FSDataOutputStream 进行的，提供任意位置读取，以及追加等功能，所以将 tyds 文件的输入输出流包装成 TydsInputStream，TydsOutputStream。

   **TydsTalker 和 TydsFsProto**

   TydsFileSystem 的功能依赖于我们 Tyds 文件系统的实现，TydsFsProto 是一个抽象基类，用于与 TydsFs 文件系统交互，调用 tydsfs 的 java 接口，而 TydsTalker 继承了 TydsFsProto 具体的实现了它的功能。包括初始化文件系统配置、获取文件系统状态、获取池 id、osd id 等功能。

   

**配置安装步骤**

1. 安装 tyds 集群

   注意要 enable java 接口。具体就是在编译的时候 CMakeLists.txt 中的 WITH_TYDSFS_JAVA 置为 ON。

2. 安装 hadoop

   - 安装 jdk
   - 安装 hadoop ，添加 java 运行路径。
   - 配置 ssh。节点之间可以免密登录

3. 安装 tyds-hdfs adapter

   tyds-hdfs 的代码编译打包成 jar 文件，在hadoop 的运行路径中添加该 jar 文件，将 tyds 的 libtydsfs-jni.so 动态库文件添加到 hadoop 运行路径中。

4. 编写 hadoop 的 core-site.xml。

   ```xml
   <configuration>
       <property>
           <name>hadoop.tmp.dir</name>
           <value>/home/hadoop/tmp</value>
       </property>
       <property>
           <name>fs.default.name</name>
           <value>tyds:///</value>
       </property>
       <property>
           <name>tyds.conf.file</name>
           <value>/workspace/tyds/build/tyds.conf</value>
       </property>
       <property>
           <name>tyds.data.pools</name>
           <value>tydsfs.a.data</value>
       </property>
       <property>
           <name>fs.tyds.impl</name>
           <value>org.apache.hadoop.fs.tyds.TydsFileSystem</value>
       </property>
   </configuration>
   ```

   

5. 启动 hadoop，sbin 目录下运行 `start-all.sh` 

5. 运行 hadoop 例子。

   运行 java 程序
   
   ```shell
   hadoop jar $jarfile $module $args
   ```
   
   
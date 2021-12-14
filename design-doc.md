# 要解决什么问题

问题1： 悬挂边

问题2： 出度、入度统计值

问题3： 优化图结构的访问速度

# 问题有什么价值？

## 问题1： 悬挂边

```
(src)-[edge]->           有起点，有边，没有终点
-[edge]->(dst)           没有起点，有边，有终点
-[edge]->               没有起点，没有终点，有边
```

> 在图论上，这是不合理的。但在nebula中物理上是存在的。

### 原因

因为图元素建模的时候，EdgeA_Out 的存在性和 srcVertex 是两个独立的key。有可能发生： 

1. graphd 只插入了边 EdgeA，没插入 srcVertx
2. 删除 Vertex 和边的操作不原子。导致垃圾 key 的存在。 边key没删成功，把点key删成功了。
3. Drop Tag 的时候，最后一个 Tag

![image](https://user-images.githubusercontent.com/50101159/145525464-05e899a2-3ca0-4bd4-8e54-f0bb78ed5bc4.png)


## 问题2： 出度、入度统计值

可以用于：

1. 判断一个点是否是超级节点？
2. 找到有哪些超级节点？
3. 最短路径等算法的时候，默认BFS，走到超级节点就先暂停BFS，改成DFS或者采样。
4. 执行计划CBO的时候，count是Cost的一个重要参数。
5. 删除一个超级节点（和周围的边），rpc超时参数值设置。

### 原因：
![image](https://user-images.githubusercontent.com/50101159/145525833-03bfa058-cbeb-482f-9fe8-7b5f57cac6c2.png)
```
EdgeA_Out 是单独的Key
EdgeA_Out1
EdgeA_Out2
EdgeA_Out3
…
EdgeA_Outn
```
对于每个点，不做 prefix scan 不知道 n 是多少。扫描，或者建索引。

## 问题3：优化图结构的访问速度
 
决大部分的图计算，只需要 edge key 的信息——图结构——以及一两个property（weight）。行式存储格式，SSD 扫描时带有基本无用的 value。

> BTW KV-seperation也是一种提升图结构访问的方式。


## 不解决的问题

- 分布式事务的能力

- TOSS 导致的逻辑边，正向和反向访问的最终一致性

- 并发的隔离性

- 点、边等数据模型: 点是否要有Tag/Lable.
 
- online DDL：DDL 不能实时感知，meta client时延问题。

- graphd, storaged 双进程 rpc 时延大的问题


## 其他设计目标

- 平滑升级，不依赖工具升级。

- 不更改通信协议

- 增加语法。

- Change: 悬挂边插入会报错。

# 业界方案参考

## JanusGraph (TitanDB)

![image](https://user-images.githubusercontent.com/50101159/145526122-f4896cce-dd05-4878-8b09-5b1a05522eda.png)

![image](https://user-images.githubusercontent.com/50101159/145958611-47138394-5ca8-4627-98be-7f507c5d5cf3.png)

JanusGraph 将一个点的ID作为KEY, “<所有属性>,<所有出边>，<所有入边>” 作为一个超级大的value

好处：

1.	不会有悬挂边
2.	Value里面可以增加一个count字段，统计出度和入度

坏处：

1.	The downside is that each edge has to be stored twice - once for each end vertex of the edge.
2.	一个边的修改，涉及到两个超大value的修改。—读取到内存，中间位置的（内存）插入，写回硬盘
3.	两个vertexID key的能否原子操作，由其存储引擎决定（HBASE，Cassandra）。Nebula 中的类似操作，叫做TOSS（见下）。

## Neo4j 的简略结构

![image](https://user-images.githubusercontent.com/50101159/145526558-500bb76d-ced0-4b8a-b167-afc91ad64cb5.png)

![image](https://user-images.githubusercontent.com/50101159/145526612-e14a110f-b0b0-41bd-abc7-2e26d9cd1d94.png)

点(Label)、边(图结构)、属性是各自独立的文件, Neo4j 自己实现这些硬盘文件的可靠性（以及Raft）

## B-tree

略

## KV-seperation (WiscKey)

略

## nebula-TOSS 

![image](https://user-images.githubusercontent.com/50101159/145527387-148e0eca-a68f-4b9a-808b-6eda8ebdb9a0.png)

保证EdgeA_Out 和 EdgeA_In的（两个机器上的）**最终一致性**

下文仍使用TOSS的逻辑，但有改动

# 解决方法详述

## 建模方式

### storage 原来的数据格式

![image](https://user-images.githubusercontent.com/50101159/145525464-05e899a2-3ca0-4bd4-8e54-f0bb78ed5bc4.png)


### vertex data （data_key）

| Key |   Value |
| -   |  - | 
| PartitionId  VertexId  TagId | Property values ,   value的开头含有当前的schema信息 |
| type(1byte NebulaKeyType::kVertex)_PartID(3bytes)_VertexID(8bytes)_TagID(4bytes)   需要vidlen来指定，不足补'\0'  | - |

index 

|  key |   Value |
| -  | - |
| type(NebulaKeyType::kIndex)_PartitionId  IndexId   Index binary  nullableBit   VertexId   | - |    
需要vidlen来指定，不足补'\0'

### edge data （data_key）

| key | value |
| -   | - |
| partId + srcId + edgeType + edgeRank + dstId + version   需要vidlen来指定，不足补'\0' |  Property values |
|    type(1byte)_PartID(3bytes)_VertexID(8bytes)_EdgeType(4bytes)_Rank(8bytes)_otherVertexId(8bytes)_version(1) | - |

index 

| key | value |
| -   | - |
| PartitionId  IndexId  Index binary  nullableBit   SrcVertexId  EdgeRank  DstVertexId | - |


## Ref_key 的设计

### 点的设计

![image](https://user-images.githubusercontent.com/50101159/145529888-a2b6e801-a69d-4598-92af-977113c78d73.png)

一个点的多个Tag是同一个 Key-value

### 边的设计

![image](https://user-images.githubusercontent.com/50101159/145528316-47b31ced-a276-4e03-ae97-8e09b7943920.png)

Partition Src

| Key  | CF 1 | CF 2|
| -    |  -   | - |
|type(1byte kedgeref_)_PartID(3bytes)_src(8bytes)_EdgeType(4bytes)+ \0 （第一个需要对齐） | countN (出度) = 10| dst_vid1 + dst_vid2  + ... dst_vid10|
|type(1byte kedgeref_)_PartID(3bytes)_src(8bytes)_EdgeType(4bytes)+ dst11               | countN (出度)       | dst_vid11 + dst_vid2 |
| ... | ... | ...|
| type(1byte kedgeref_)_PartID(3bytes)_src(8bytes)_EdgeType(入边) + ... | count (入度) = 10 | ... | ...
| ... | ... | ... |


Partition dst （略）

| Key  | CF 1 | CF 2|
| -    |  -   | - |
|type(1byte kedgeref_)_PartID()_dst(8bytes)_EdgeType(4bytes)  | ... | ... src_vid ...|

### 几个重要的说明

#### Ref 需要常驻内存

Ref_key 所在Rocksdb 与 Data_key 所在的Rocksdb 实例分开(rocksdb data,  rocksdb ref)

选用 Plaintable

#### countN = 10

这里假设一行存放 10 个dst，实际上这个参数取决于：内存占用，平均出入度，新插入的写效率。


内存空间估算（10亿条边为例: 3 拷贝 * 2 反向 * 1G /countN * 每行字节数 ）

| countN | 一行字节 | 集群内存  |
| - | - | - |
| 10  | 96 个字节 | 96GB |
| 50  | 416 | 49GB |
| 100 | 816 | 48GB |

TODO: 需要benchmark下

#### ranking 字段

目前不常用(缺乏对应操作语法)，也可以设计一个变长结构节省内存 (len(char)_dst1_str(0-8))。本次先考虑ranking只为0

#### 影响Raft wal的地方

1. leader 写入时，ref-key和data-key 用一条raftwal 发出去。follower 收到时，拆开； 持久化通过raft wal保证

#### 写入一个 Tag


![image](https://user-images.githubusercontent.com/50101159/145535047-caa67b60-81f0-4f8b-9f7b-4294f7892c8c.png)


> nebula的数据模型中，点必须有Tag。不允许没有Tag的点存在。

```
加内存写锁： 锁的vertexId/srcId  -- 读没有锁保护，并发隔离性不变，和当前nebula一致
{
   1）写vertex data key      写rocksdb data
   2）写tag_ref              写rocksdb ref
   3) 从内存读取tag_ref，     然后更新内存 -- 是否可以不需要了？
}
```

#### 读取一个 Tag

读取点或者tagId  ref：

1. 从rocksdb ref 中读取vertexid, tagId


读某个点的tag的属性：

1. 从rocksdb ref 中读取tagId， 如果没有，直接返回，如果有的话:
2. 从Rocksdb data读。

#### 写一条边的逻辑

![image](https://user-images.githubusercontent.com/50101159/145536908-1c68e7b6-8bc3-402d-a01e-2e27e9b5308e.png)


```
{
  加 src 内存写锁
  1）检查src点存在，读取tag_ref, 不存在报错
  2）本地edgeKey'   -- TOSS虚拟锁
   
  {
     加 dst机器 内存锁
     3) dst机器，检查dst点存在，读取tag_ref, 不存在报错
     4）反向边的data key，写rocksdb data
     5）data ref          rocksdb edge ref，并修改count
   }

  6) src机器: 
     正向边data key，写rocksdb edge data
  7）data ref，写rocksdb edge ref，并修改count
  同时TOSS解锁
}

  失败处理：TODO  必须保证ref key 在src 和dst两个partition的正确性，data key交给GC
```
#### 删除一条边

![image](https://user-images.githubusercontent.com/50101159/145541433-71994576-3e2b-43b9-9d65-aec20c3000b7.png)
![image](https://user-images.githubusercontent.com/50101159/145541446-d0a35341-ff81-4192-a16b-965fc5bd8c8d.png)

删除边时只需要操作 Ref_key，data_key交给GC

#### 关于CF Edge 过大分离的问题补充，& 批量插入边

![image](https://user-images.githubusercontent.com/50101159/145535785-a505fbbd-4b45-48a3-8cd2-723b368c2608.png)

1. 当同一类型的边是按顺序插入时, dst1, dst3, dst5；每插入countN个后，下一个 dst 作为ref的第二行key的一部分。`src+outEdge1+\0`和 `src+outEdge1+dst11`。这样方便dstxx的定位（见后）
2. 当同一类型的边是乱序插入时，`3*countN`满了之后（超级节点），这个Refkey要发生分裂。批量导入和增量导入可以考虑不同分裂的方式（见后讨论）
3. 不同类型Edge部分不同

#### 读取一条边或者多条边 （取边结构）

1. rocksdb ref  查看边是否存在，没有的话报错。有的话，得到edgekey

> countN 提升了取一批边结构的速度

#### 取边结构+边属性

1. rocksdb ref  查看边是否存在，没有的话报错。有的话，得到edgekey
2. rocksdb data 去获取edgekey对应的属性。 使用multiget，而不是prefix，性能更好。

多种边类型操作类似

#### 取点的出度、入度

1. 从rocksdb ref找到对应的key，直接累加。相同类型会更快

#### TODO 删除一个点，以及周围所有的边

Detach DELETE 是一个很重的操作，目前没有好的办法。

一些想法：判断是否是超级节点，然后批量删除边

#### GC

部分写成功，或者标记位删除时，可以用GC来回收

扫描每个data key,检查是否存在对应的ref key.

#### 大批量数据初始化——不实现

Sst ingest。逻辑放在spark里面 （TODO  ）

#### 大批量数据写入——不实现

TODO：也许可以采用类似 rebuild index方式，批量写(metad标记为不可读)，再重建Ref_Key

或者Ref_key不加锁

#### 索引——不实现

TODO：应该改动不大

#### 升级——不实现

TODO：类似rebuild index方案（快照+增量？）。V1代码仍可以读DataKey，V2代码读Ref_Key+DataKey。

标记位检查

# 展示的语法

1. 取出度、或者入度
2. GO FROM语句
3. insert 悬挂边失败

# 一些其他的可能性讨论

## VID 变长，内外的 id 解耦

一个图空间的VID定长，但MySQL是每个表PK定长。一个简单的例子，同一个库里面，设备mac和身份证号两张表的pk一定长度不一样，内存空间浪费严重。

通过ref-key 可以做一层语意隔离，把用户语意的 PK 和系统内部的 ID 分开

![image](https://user-images.githubusercontent.com/50101159/145542622-7521f35a-ecaa-4ad5-96ec-c30403125205.png)

如果记录Len1 Len2这样的offset，可以不要求VID必须是固定长度。

datakey用whole_key_bloom_filter定位。ref_key用内存定位。




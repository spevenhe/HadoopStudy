# flink 集群
### 1 flink 任务提交流程
对于session与per-job
首先在客户端本地执行main（），下载资源，然后将资源与jobGraph一起提交到集群中
![image](https://user-images.githubusercontent.com/42630862/110566809-8367fe80-818b-11eb-8eb5-897e08a763c1.png)

session模式：
优：资源使用
劣：一个job挂了，带走一个tm，所有使用那个tm的job都会被影响

per-job:
资源隔离，分散jm压力：生产使用

### 2 flink 聚合区别
![image](https://user-images.githubusercontent.com/42630862/110566516-1bb1b380-818b-11eb-9f98-c09ac03c00cd.png)

group aggregation的输出的结果是不断更新的，相同的字段会更新输出，比如先a,1  a,2  a,3..... **需要sink到能够更新结果的sink里，如mysql，es，hbase**。group 作业都要配置ttl，去清除状态。


### 2. flink sql
 1. flinksql 是什么：flink 是最高层的，批流统一
 2. flink sql 中必须包含时间字段
 3. flink sql 支持的数据类型：
 
 ![image](https://user-images.githubusercontent.com/42630862/110748462-b63cf000-827a-11eb-99b6-5e590cf2ff69.png)

3.1 注意点：timestamp（p） p代表秒的精度

  4. flink sql kafka
注意点 flink sql kafka 必填format或者value format，因为不是所有kafka message 都有key。一般用 ‘format’=‘json’，然后在 key.fields中能指定 key的对象，value也能指定value的对象，两者重叠使用key.fields_prefix具体例子见https://ci.apache.org/projects/flink/flink-docs-release-1.12/zh/dev/table/connectors/kafka.html

4.1 需要用刀flink-json包，sql-client 内置



#### 2.1 sql-client

1. 使用UDF 继承类scalarFunction，打包，扔到lib，查看show functions

2. flink-sql demo 

小技巧1： proctime as PROCTIME() as 可以作为生成一个虚拟列
小技巧2： over window 是
所有的聚合必须定义到同一个窗口中，即相同的分区、排序和区间。当前仅支持 PRECEDING (无界或有界) 到 CURRENT ROW 范围内的窗口、FOLLOWING 所描述的区间并未支持，ORDER BY 必须指定于单个的时间属性

kafka字段数限制，java数组字段数的限制

10个流的join：每次两个

使用blink

sql 接Kafka，解析ddl，生成consumer

嵌套的json ROW<>




flink sql 与kafka partrition 相同的并发度

激活catalog 持久化catalog

 ### 3 flink 调优
 #### 3.1 Flink 如何权衡吞吐与延迟的关系
 正在写入的 buffer 肯定不能被 Netty 消费到，只有写完的 buffer 才能被消费到。在 Flink 中有三种情况会认为 buffer 写完了，可以被 Netty  Server 消费：

buffer 写满了

buffer 超时了

遇到特殊的 Event，例如：Checkpoint barrier

 通过tm间 netty的buffer来设置：
 Flink 1.10 及以后的版本直接通过配置参数 execution.buffer-timeout: 100ms
 
 
 #### 3.2 Flink 并发度大会导致CPU资源消耗高
 
 生产环境很多任务的并发远大于 1000，所以造成很多 buffer 仅仅只缓存 1 条数据就被 timeout 策略触发发送给下游 Task。每条数据做为 1 个 buffer，每秒处理 1 万条数据，则后台线程每秒需要 flush 1 万个 buffer 到 NettyServer，从而大量消耗 CPU。

不仅是发送方效率降低，下游的 Subtask B 接受数据的效率也会降低。每秒接受 1 万的 buffer，每个 buffer 里 1 条数据。大量的小 buffer，大量的读取小数据，消耗大量的 CPU 资源。

 #### 3.3 Flink cpu调优

测试条件：业务逻辑简单，keyBy 后不做任何业务处理，并发 1200。

因为没有业务处理，所以从业务角度来讲，CPU 应该主要消耗在序列化和反序列化 protobuf。从引擎角度来讲，TM 之间数据 flush、传输需要消耗不少的 CPU。

基于上述条件，仅调节 bufferTimeout 参数，观察单个 TM 的平均 CPU 消耗。

bufferTimeout = 100ms 时，TM 平均 CPU 使用 0.59 core
bufferTimeout = 1s 时，TM 平均 CPU 使用 0.39 core，CPU 节省了 33%
bufferTimeout = 10s 时，TM 平均 CPU 使用 0.33 core，CPU 节省了 44%
 
 #### 3.4 Flink 并行度优化
 Flink 中算子链接（chain）起来的条件
Flink 在内部会将多个算子串在一起作为一个 operator chain（执行链）来执行，每个执行链会在 TaskManager 上的一个独立线程中执行，这样不仅可以减少线程的数量及线程切换带来的资源消耗，还能降低数据在算子之间传输序列化与反序列化带来的消耗。

举个例子，拿一个 Flink Job （算子的并行度都设置为 5）生成的 StreamGraph JSON 渲染出来的执行流程图如下图所示。

![image](https://user-images.githubusercontent.com/42630862/112783835-d1816b00-9082-11eb-9980-fa1f619995dc.png)


提交到 Flink UI 上的 JobGraph 如下图所示。



![image](https://user-images.githubusercontent.com/42630862/112783829-ccbcb700-9082-11eb-8063-26d2518a81f0.png)


可以看到 Flink 它内部将三个算子（source、filter、sink）都串成在一个执行链里。但是我们修改一下 filter 这个算子的并行度为 4，我们再次提交到 Flink UI 上运行，效果如下图所示。
![image](https://user-images.githubusercontent.com/42630862/112783816-c3334f00-9082-11eb-87b6-62c06678edbd.png)
你会发现它竟然拆分成三个了，我们继续将 sink 的并行度也修改成 4，继续打包运行后的效果如下图所示。
![image](https://user-images.githubusercontent.com/42630862/112783811-c0385e80-9082-11eb-99a9-138d96628053.png)

图片
神奇不，它变成了 2 个了，将 filter 和 sink 算子串在一起了执行了。经过简单的测试，我们可以发现其实如果想要把两个不一样的算子串在一起执行确实还不是那么简单的，的确，它背后的条件可是比较复杂的，这里笔者给出源码出来，感兴趣的可以独自阅读下源码。

public static boolean isChainable(StreamEdge edge, StreamGraph streamGraph) {
    //获取StreamEdge的源和目标StreamNode
 StreamNode upStreamVertex = edge.getSourceVertex();
 StreamNode downStreamVertex = edge.getTargetVertex();
 
    //获取源和目标StreamNode中的StreamOperator
 StreamOperator<?> headOperator = upStreamVertex.getOperator();
 StreamOperator<?> outOperator = downStreamVertex.getOperator();

 return downStreamVertex.getInEdges().size() == 1
   && outOperator != null
   && headOperator != null
   && upStreamVertex.isSameSlotSharingGroup(downStreamVertex)
   && outOperator.getChainingStrategy() == ChainingStrategy.ALWAYS
   && (headOperator.getChainingStrategy() == ChainingStrategy.HEAD ||
    headOperator.getChainingStrategy() == ChainingStrategy.ALWAYS)
   && (edge.getPartitioner() instanceof ForwardPartitioner)
   && upStreamVertex.getParallelism() == downStreamVertex.getParallelism()
   && streamGraph.isChainingEnabled();
}
从源码最后的 return 可以看出它有九个条件：

下游节点只有一个输入
下游节点的操作符不为 null
上游节点的操作符不为 null
上下游节点在一个槽位共享组（slotsharinggroup）内，默认是 default
下游节点的连接策略是 ALWAYS（可以与上下游节点连接）
上游节点的连接策略是 HEAD 或者 ALWAYS
edge 的分区函数是 ForwardPartitioner 的实例（没有 keyby 等操作）
上下游节点的并行度相等
允许进行节点连接操作（默认允许）
所以看到上面的这九个条件，你是不是在想如果我们代码能够合理的写好，那么就有可能会将不同的算子串在一个执行链中，这样也就可以提高代码的执行效率了。
 
 ### 4flink 问题汇总
 
**资源不足导致 container 被 kill**

`The assigned slot container_container编号 was removed.`
Flink App 抛出此类异常，通过查看日志，一般就是某一个 Flink App 内存占用大，导致 TaskManager（在 Yarn 上就是 Container ）被Kill 掉。

但是并不是所有的情况都是这个原因，还需要进一步看 yarn 的日志（ 查看 yarn 任务日志：yarn logs -applicationId  -appOwner），如果代码写的没问题，就确实是资源不够了，其实 1G Slot 跑多个Task（ Slot Group Share ）其实挺容易出现的。

因此有两种选择，可以根据具体情况，权衡选择一个。

将该 Flink App 调度在 Per Slot 内存更大的集群上。
通过 slotSharingGroup("xxx") ，减少 Slot 中共享 Task 的个数




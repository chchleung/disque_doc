# Disque 基础使用

## 版本

DISQUE_VERSION ：1.0-rc1

DOC_EDITION ： v1.0



## 简介

Disque 是一个内存储存的分布式任务队列实现， 它由 Redis 的作者 Salvatore Sanfilippo ([@antirez](https://twitter.com/antirez))开发， 目前正处于预览版（alpha）阶段。

与Redis有单结点和分布式模式不同，单一Disque结点也是只有一个结点的集群。它是一个AP系统，也就是具有Availability（可用性）和Partition tolerance（分区容错性）。另外它能在各种情形下保持高扩展性：无论是多生产者和多消费者处理多队列，还是所有生产者和消费者都在一个队列。

Disque只尽力提供而不保证消息的排序。

Disque的一些特性：

- 消息发送可以选择至少一次或者最多一次
- 消息需要消费者确认
- 如果没有确认，会一直重发，直至到期。确认信息会广播给拥有消息副本的所有结点，然后消息会被垃圾收集或者删除
- 队列是持久的
- Disque默认只运行在内存里，持久性是通过同步备份实现的
- 队列为了保证最大吞吐量，不是全局一致的，但会尽力提供排序
- 在压力大的时候，消息不会丢弃，但会拒绝新的消息
- 消费者和生产者可以通过命令查看队列中的消息
- 队列尽力提供FIFO
- 一组master作为中介，客户端可以与任一结点通信
- 中介有命名的队列，无需消费者和生产者干预
- 消息发送是事务性的，保证集群中会有所需数量的副本
- 消息接收不是事务性的
- 消费者默认是接收时是阻塞的，但也可以选择查看新消息
- 生产者在队列满时发新消息可以得到错误信息，也可以让集群异步地复制消息
- 支持延迟作业，粒度是秒，最久可以长达数年，但需要消耗内存
- 消费者和生产者可以连接不同的结点



## 安装与运行

1. 执行命令 `git clone https://github.com/antirez/disque.git` ，使用 Git 克隆 Disque 的项目文件夹。

2. 执行 `cd disque` 命令，切换至 Disque 的项目文件夹。

3. 执行 `make` 命令，编译 Disque 。

4. 可选：执行 `make test` ，测试刚刚编译好的 Disque 程序。

5. 执行 `cd src` 命令，进入 `src` 文件夹，里面有编译好的 Disque 服务器文件 `disque-server` 和 Disque 客户端文件 `disque` 。



## 搭建单节点集群

Disque 以集群模式运行， 每个服务器都是集群中的一个节点， 可以运行任意数量的节点， 只要确保每个节点的 IP:PORT 不同即可。

Disque根目录下运行

`./src/disque-server`

在默认情况下， 运行 Disque 服务器程序 `disque-server` 将启动一个端口号为 `7711` 的 Disque 节点

指定端口启动：

`./src/disque-server --port 10086`

或者指定配置文件：

`./src/disque-server ./disque.conf`

修改Disque根目录下disque.conf 可以自行设置守护进程，端口号， IP绑定， 持久化等参数



##搭建多节点集群

默认情况下，每个 Disque 节点都会将自己的节点配置信息和AOF文件存储到Disque根目录下。如果在单台机器上想要部署多个Disque服务，则需要在其配置文件修改对应node.conf文件/disque.aof文件的名称以示区别。

假定在本地端口7711, 7712， 7713中运行了三个Disque服务。为了加入集群，可以在Disque文件根目录下在命令行运行以下命令：

`./src/disque -h 127.0.0.1 -p 7711 cluster meet 127.0.0.1 7712`  
`./src/disque -h 127.0.0.1 -p 7711 cluster meet 127.0.0.1 7713`

或者先运行Disque命令行工具：

`./src/disque -h 127.0.0.1 -p 7711`

然后再添加其他节点：

`CLUSTER MEET 127.0.0.1 7712`

`CLUSTER MEET 127.0.0.1 7713`

Disqus是无中心节点的，连接节点的顺序和参数位置没有限制。可以在任何时间（服务刚启动或是服务运行中）向集群添加新节点，服务会自动处理。

此时可以在Disque命令行里输入命令： `CLUSTER  INFO` 将会显示集群的基本信息：

```
127.0.0.1:7711> CLUSTER INFO
cluster_state:ok
cluster_known_nodes:3                           -- 集群现在有三个节点
cluster_reachable_nodes:2
cluster_size:3
cluster_stats_messages_sent:23
cluster_stats_messages_received:23
```
或者输入`CLUSTER NODES`查看节点情况
```
127.0.0.1:7711> CLUSTER NODES
76a8d0e6dfcf931a50d457f5488f188adb673324 127.0.0.1:7711 myself 0 0 connected
a93e5cd36bdbc95dacb23e9674274c758cef223a 127.0.0.1:7712 noflags 0 1498554629402 connected
19762531abcad0d23caeb14f0ed8f81f3ee4b309 127.0.0.1:7713 noflags 0 1498554626399 connected
```

在Disque命令行里面命令不区分大小写



##启动Disque命令行工具

在Disque文件根目录下在命令行并运行：

`./src/disque -h 127.0.0.1 -p 7711`



## 常用命令

### ADDJOB

```
ADDJOB queue_name job <ms-timeout> [REPLICATE <count>] [DELAY <sec>] [RETRY <sec>] [TTL <sec>] [MAXLEN <count>] [ASYNC]

举例：
127.0.0.1:7711> ADDJOB greeting "hello world!" 0 REPLICATE 2 RETRY 300 TTL 86400 ASYNC

返回任务ID：
D-19762531-TCvWFp7vb7+goDMbG1Ou7pmf-05a1
```

向队列添加任务。

以下是各个参数的含义：

- `queue_name` ： 队列的名字， 可以是任意字符串。 如果指定的队列并不存在， 那么它将被自动创建， 用户无需手动创建队列。 当队列没有任务可传递时， 它也会自动被释放。
- `job` ： 使用字符串描述的任务。 Disque 不关心任务的具体含义， 对于它来说， 一项任务就是一条待传送的消息。最大大小为4GB。
- `ms-timeout` ： 毫秒精度的命令超时限制。 如果用户没有使用 `ASYNC` 选项， 并且在指定的毫秒数之内， 任务的复制级别（replication level）未能达到指定的要求， 那么命令将返回一个错误， 而节点会尽可能地清理未发出的消息， 并将遍布于整个集群的消息副本删除， 但是任务仍然有可能会在之后被传递。 在默认的服务器频率（hz）下， 实际的超时解析度为 1/10 秒钟。
- `REPLICATE <count>` ： 指定任务需要复制至多少个节点。默认情况下，如果节点数大于3，则REPLICATE为3，否则等于节点数量。
- `DELAY <sec>` ： 指定任务在放入各个节点的队列之前， 需要等待多少秒钟。默认情况下没有延迟。
- `RETRY <sec>` ： 在未接到 `ACK` 回复的情况下， 节点在多少秒钟之后才会重新将任务放入待传递的队列里面。 如果 `sec` 参数的值为 `0` ， 那么任务将以“最多一次”（at-most-once）的方式进行传递： 这个任务不会被重新放入到待传递队列里面， 并且它的复制因子（replication factor）也只会为 1 。默认的重试时间为5分钟。如果TTL太小，TTL的10％小于5分钟，默认的RETRY设置为TTL / 10（最小值为1秒）。
- `TTL <sec>` ： 秒级精度的任务生存时间。 在生存时间消耗完毕之后， 即使任务还未被成功传递， 它也会被删除。默认的TTL是一天。
- `MAXLEN <count>` ： 指定队列最多可以存放多少个待传递的任务。 在队列已满的情况下， 尝试添加新任务将被拒绝， 而客户端也会接收到一个错误。
- `ASYNC` ： 要求服务器让命令尽快返回， 并在后台执行将任务复制至其他节点的工作。 任务会尽快地被放入到队列里面。在普通情况下， 只有当任务已经被放入到队列里面的时候， 客户端才会接收到正面的回复。当指定了 `ASYNC` 选项， 或者任务已经被正确地复制到了指定数量的节点上面时， 命令返回已入队任务的 ID ； 否则命令返回错误。




###GETJOB

```
GETJOB [NOHANG] [TIMEOUT <ms-timeout>] [COUNT <count>] [WITHCOUNTERS] FROM queue1 queue2 ... queueN

举例：
127.0.0.1:7711> GETJOB FROM greeting    

返回：
1) 1) "greeting"                                           
   2) "DI216f7fa17693623ffb3bd8b0902e134f4ab6a5d305a0SQ"    
   3) "hello world!"  
```

- `NOHANG`：如果在所有指定的队列中没有作业，请求命令不堵塞。这样可以检查是否有可用的作业，而不会阻塞。
- `TIMEOUT` ： 超时停止阻塞
- `COUNT  <count>` ：一次取多少个任务。如果可取任务数量大于1, 即使没有足够任务，也返回。
- `WITHCOUNTERS`：返回此任务收到的NACK的次数以及此任务额外的发送次数。以此可以检查任务是否经多次尝试依旧失败。

从给定的队列里面取出可用的任务，默认将为阻塞， 或者在超时时间达到时， 返回 `nil` 。
在默认情况下， 命令每次最多只会返回一个任务， 但使用 `COUNT <count>` 选项可以指定每次最多可以获取的任务数量。

任务会以数组的形式被返回， 每个任务由三个元素组成：
1. 任务所在的队列
2. 任务的 ID 
3. 任务的内容

如果用户给定了多个队列， 那么命令将按照队列被给定的顺序， 从左到右地轮流每个队列取1个直到到达COUNT或者没有更多任务，然后返回。

如果给定的队列没有任务可以返回， 那么执行命令的客户端将被阻塞， 执行命令的节点会与其他节点进行消息交换， 以便将可能存在的任务移动到这个节点里面， 从而尽快地将任务返回给被阻塞的客户端。



### ACKJOB

```
ACKJOB jobid1 jobid2 ... jobidN

举例：
127.0.0.1:7711> ACKJOB D-76a8d0e6-W8219ZT6ek1Li0V3t+Wu7tKd-05a1

返回：
任务存在：
(integer) 1

重复ackjob或者任务不存在：
(integer) 0

```

通过给定任务 ID ， 向节点告知任务已经被执行。

接收到 `ACK` 消息的节点会将该消息复制至多个节点， 并尝试对任务和来自集群的 `ACK` 消息进行垃圾回收操作， 从而释放被占用的内存。

接收到关于其不知道的任务ID的ACKJOB命令的节点将创建一个特殊的空任务，状态设置为“acknowledge”，称为“dummy ACK”。dummy ACK用于保留ACK信息，以便拥有该任务的失效结点在重新上线时可以及时收到ACKJO确认。

任务ID包含了”至多一次“或者是”至少一次“的retry信息，“dummy ACK"只会为”至少一次发送“的任务创建。

如果任务在retry到达并且requeue之后才收到ACKJOB消息，队列中的任务会被删去，避免重复发送。



### FASTACK

```
FASTACK jobid1 jobid2 ... jobidN
```

尽最大努力在集群范围内对给定的任务进行删除。

在网络连接良好并且所有节点都在线时， 这个命令的效果和 `ACKJOB` 命令的效果一样， 但是因为这个命令引发的消息交换比 `ACKJOB` 要少， 所以它的速度比 `ACKJOB` 要快不少。

ACKJOB处理流程：

1. 客户端向一个节点发送ACKJOB。
2. 节点向其认为拥有副本的每个节点发送一个SETACK消息。
3. SETACK的接收节点用GOTACK回复确认。
4. 节点最终将DELJOB发送到所有节点。

如果消息被复制到3个节点，则确认需要1 + 2 + 2 + 2个消息。

FASTACK处理流程：

1. 客户端发送`FASTACK`到一个节点。
2. 该节点删除该任务，并向尽可能具有副本的所有节点发送DELJOB。

当集群中包含了失效节点的时候， `FASTACK` 命令比 `ACKJOB` 命令更容易出现多次发送同一消息的情况。如果重复发送并不影响业务，FASTACK会更高效。



### WORKING 

```
WORKING jobid

举例：
127.0.0.1:7711> working D-fcc6cde5-ZtDlZbywcKjFxdGMTASrqy+E-05a1

返回任务的retry：
(integer) 300  

通常用法（伪代码）：
retry = WORKING(jobid)
RESET timer
WHILE ... work with the job still not finished ...
    IF timer reached 80% of the retry time
        WORKING(jobid)
        RESET timer
    END
END
```

用于知会结点该任务仍旧处理中，重置retry的计算时间。如果该节点没有这个任务或者该任务已经ack，将会返回错误：`(error) NOJOB Job not known in the context of this node.`

如果已经过了任务的TTL的50%， `WORKING`命令将被忽略，避免任务被单个程序长时间占用。



### NACK 

```
NACK <job-id> ... <job-id>

举例：
127.0.0.1:7711> nack D-fcc6cde5-HcdNDSYjI+yODn6NDYBzcw/Q-05a1

返回：
(integer) 1
```

`NACK`命令告诉Disque尽快将作业放回队列。和`ENQUEUE`命令相似，但它增加了任务的`nacks`计数器而不是`additional-deliveries`计数器。当任务无法处理并希望将任务放回队列中以便再次处理时，可主动使用`NACK`



### SHOW

```
SHOW <job-id>

举例：
127.0.0.1:7711> show D-fcc6cde5-HcdNDSYjI+yODn6NDYBzcw/Q-05a1

返回：
 1) "id"                                                            -- 任务 ID
 2) "D-fcc6cde5-HcdNDSYjI+yODn6NDYBzcw/Q-05a1"
 3) "queue"															-- 所属队列
 4) "greeting"
 5) "state"															-- 任务状态
 6) "queued"
 7) "repl"															-- 副本数量
 8) (integer) 3														
 9) "ttl"															-- 生存时间
10) (integer) 85757	
11) "ctime"															-- 创建时间
12) (integer) 1498569138301000000
13) "delay"															-- 延迟时间
14) (integer) 0
15) "retry"															-- 重试时间
16) (integer) 300
17) "nacks"															-- NACK计数
18) (integer) 1
19) "additional-deliveries"											-- 重新enqueue计数
20) (integer) 0
21) "nodes-delivered"												-- 已向以下节点发送该任务
22) 1) "76a8d0e6dfcf931a50d457f5488f188adb673324"
    2) "fcc6cde5f3f52c4be6f589f76b513aafe9d547a8"
    3) "19762531abcad0d23caeb14f0ed8f81f3ee4b309"
23) "nodes-confirmed"												-- 以下节点已确认收到该任务 ？（不确定）
24) (empty list or set)
25) "next-requeue-within"											-- retry 倒计时
26) (integer) 83236
27) "next-awake-within"												-- 未知作用 ？ 
28) (integer) 82736
29) "body"															-- 任务的内容
30) "hello world"

```

描述任务的具体情况。



### QLEN 

```
QLEN <queue-name>

举例：
127.0.0.1:7711> QLEN greeting  

返回：
(integer) 1
```

返回队列目前存放的任务数量。



### QPEEK

```
QPEEK <queue-name> <count>

举例：
127.0.0.1:7711> qpeek greeting 4
1) 1) "greeting"
   2) "D-fcc6cde5-+AOLC66GxlQJidFY2EoYmBsD-05a1"
   3) "hello world"

```

从该节点的该队列返回COUNT数量的任务信息，如果不够COUNT，则尽可能返回最多数量的任务ID。如果该节点的该队列没有任务，则返回`nil`。但并不代表集群的其他节点没有该队列的任务。

使用QPEEK只是用于查看任务信息，并不会消费队列中的任务。



### QSTAT 

```
QSTAT <queue-name>

举例：
127.0.0.1:7711> QSTAT greeting  

返回：
 1) "name"
 2) "greeting"
 3) "len"
 4) (integer) 1
 5) "age"
 6) (integer) 7
 7) "idle"
 8) (integer) 7
 9) "blocked"
10) (integer) 0
11) "import-from"
12) (empty list or set)
13) "import-rate"
14) (integer) 0
15) "jobs-in"
16) (integer) 1
17) "jobs-out"
18) (integer) 0
19) "pause"
20) "none"

```

将队列的信息显示为键-值对数组, 但不一定保证字段的输出顺序。

如果队列不存在，则返回`nil`。

因为队列信息在各个节点是独立的，节点不存在队列并不代表集群中没有关于此队列的任务。当有GETJOB请求时，队列将立即创建并尝试从集群中寻找该队列的任务信息。

- ` name` ：队列名称
- `len`  ：队列任务数量


- `age` ：队列存在时长
- `idle ` ：暂不清楚用处
- `blocked` ：显示目前有多少个client在该节点阻塞等待从该队列获取任务
- `import-from ` ：该队列从哪些节点获取任务以满足任务请求
- `import-rate` ：瞬时的从其他节点获取任务的速率 jobs/sec
- `jobs-in / jobs-out`  ：队列任务的 enqueued /  dequeued 数量。 注意 len + jobs-out 并不一定等于  jobs-in， 因为如果任务在requeue后才收到ACKJOB，会执行DELJOB使得jobs-out不会相应增加
- `pause` ：队列暂停enqueue或者dequeue的状态， 可能出现为 in / out / all / none




### QSCAN

```
QSCAN [COUNT <count>][BUSYLOOP] [MINLEN <len>][MAXLEN ] [IMPORTRATE <rate>]

举例： 
127.0.0.1:7711> QSCAN

返回：
1) "0"
2) 1) "greeting"
   2) "greeting2"
```

迭代并返回本地节点中所有的现有队列名称。

- `COUNT <count>` ： ”A hint about how much work to do per iteration.“
- `BUSYLOOP` ：”Block and return all the elements in a busy loop.“
- `MINLEN <count>` ：不要返回少于`count`任务数量的队列
- `MAXLEN <count>` ： 不要返回超过`count`任务数量的队列
- `IMPORTRATE <rate>` ：只返回从其他节点导入任务的速率大于等于 `rate`的队列

实际上，此命令提供一个cursor接口， 详见[DISQUE_DOC](https://github.com/antirez/disque)



### JSCAN

```
JSCAN [<cursor>] [COUNT <count>] [BUSYLOOP] [QUEUE <queue>] [STATE <state1> STATE <state2> ... STATE <stateN>] [REPLY all|id]

举例：
127.0.0.1:7711> JSCAN

返回：
1) "0"
2) 1) D-fcc6cde5-9jGOzjj4OL2xcddRkQhs7CfZ-05a0
   2) D-fcc6cde5-9dO4uNRMzBDuLj34+N4ubk8p-05a0
   3) D-fcc6cde5-EKJlJXUPDwNPz3qAwEWFBVW9-05a0

```

迭代并返回本地节点记录的所有任务ID，包含任务副本。

- `COUNT <count>` ： ”A hint about how much work to do per iteration.“
- `BUSYLOOP` ：”Block and return all the elements in a busy loop.“
- `QUEUE <queue>`： 仅返回指定队列中的任务
- `STATE <state>`： 返回指定状态的任务。可以多个OR来组成条件。
- `REPLY <type>`： 回复类型 类型可以是`all`或`id`。默认是仅返回任务ID。如果使用`all`则会返回完整的任务状态，就像`SHOW`命令一样。

实际上，此命令提供一个cursor接口， 详见[DISQUE_DOC](https://github.com/antirez/disque)



### PAUSE

```
PAUSE <queue-name> option1 [option2 ... optionN]

举例：
PAUSE greeting in

返回：
in
```

控制当前节点队列的暂停状态，并不会影响其他节点该队列的暂停状态。返回执行后的队列暂停状态。

输入或输出中暂停的队列永远不会被清除和回收内存，即使它们是空且长时间不活动，否则暂停状态将被遗忘。

可选OPTION：

- **in**：暂停添加任务，此时向该节点ADDJOB将返回错误：`(error) PAUSED Queue paused in input, try later`。到达retry的任务不会重新enqueue，而是会重置retry计时器，在之后重试enqueue; 同时也不接受来自其他节点的副本复制请求
- **out**：暂停输出任务。其他节点也无法通过此节点获取该队列任务
- **all**：同时暂停  in 和 out
- **none**：清除暂停状态
- **state**：只报告当前的队列暂停状态
- **bcast**：向群集的所有可达节点发送一个PAUSE命令，将其他节点中的同一队列设置为相同的状态




### ENQUEUE

```
ENQUEUE <job-id> ... <job-id>

举例：
127.0.0.1:7711> enqueue D-437658fb-dA0ISEENjQc3N8CcnDSenjKI-05a1

返回：
(integer) 1
```

如果任务未曾进入队列，或任务未曾ack，则可通过此命令将任务添加到队列中，否则将返回0



### DEQUEUE

```
DEQUEUE <job-id> ... <job-id>

举例：
127.0.0.1:7711> dequeue D-437658fb-CSL394qNDT7FxdvmZr2d3RIa-05a1

返回：
(integer) 1
```

从该节点的队列中删除该任务。如果此节点的队列没有该任务，将返回0， 但并不代表集群其他节点没有该任务。

retry时间到达后，任务会重新enqueue



### DELJOB

```
DELJOB <job-id> ... <job-id>

举例：
127.0.0.1:7711> deljob D-437658fb-CSL394qNDT7FxdvmZr2d3RIa-05a1

返回：
(integer) 1
```

从当前节点中完全删除任务，并不影响其他节点的任务情况。如果该节点并没有该任务，将返回0。



### INFO

```
INFO [OPTIONS]

举例：
127.0.0.1:7712> INFO JOBS

返回：
# Jobs
registered_jobs:1
```

显示服务器及统计信息。默认显示Server、Clients、Memory、Jobs、Queues、Persistence、 Stats、CPU 的信息；或指定单独项目显示信息。



### HELLO

```
HELLO

举例：
127.0.0.1:7711> HELLO

返回：
1) (integer) 1
2) "437658fb3dd2aa52d8c187e18a89ce265d6ef2b5"
3) 1) "4dc92cb4680969f48036e2fab8e51143e6f6ce98"
   2) "127.0.0.1"
   3) "7711"
   4) "1"
```

握手命令，返回此节点ID，所有节点ID，IP地址，端口和优先级（越低越好）。



### DEBUG FLUSHALL

```
DEBUG FLUSHALL

举例：
127.0.0.1:7711> DEBUG FLUSHALL

返回：
OK
```

此命令用于将整个节点的全部信息清空。



## 持久化

### 设置开启AOF

默认情况下，Disque并不开启AOF，Disque的可靠性通过在多个节点备份任务副本来保证。可以通过修改配置文档来启用硬盘持久化：

- `--appendfsync everysec`: 每秒写入1次AOF文件，还可以设置为`always`，每命令存储AOF文件，更安全，但是性能会差一些
- `--appendfilename`: AOF文件路径
- `--aof-enqueue-jobs-once`: 启动／重启Disque的时候，读取AOF文件，加载数据。默认为no。即使修改为yes，重启后会自动变成no。
- `--appendonly`: 启用AOF特性，默认为no。

一般情况下，设置 `--appendonly yes`后持久化开启。

通过命令`./src/disque-server ./disque.conf`重启Disque服务器将会自动加载AOF文件读取任务信息，此时节点队列为empty。Disque只重新加载内存中的任务数据，而不会填充队列，因为未确认的作业最终被重新入列。

通过命令`./src/disque-server ./disque.conf  --aof-enqueue-jobs-once yes`重启Disque服务器将会同时把旧的队列信息重新加载。但对于重试值设置为0的任务，重新加载队列数据是不安全的。

如果在受控情况下希望从AOF重新加载完整的队列状态，可以通过以下命令进行服务器的重启：

```
CONFIG SET aof-enqueue-jobs-once yes
CONFIG REWRITE
SHUTDOWN REWRITE-AOF
```

执行以上命令，无论是否开启了AOF，都会产生AOF文件，并在下一次启动的时候执行加载队列信息。

每次重启Disque服务器之后`--aof-enqueue-jobs-once`都会重置为no。

###优化AOF加载数据

如果Disque长期运行，则会积累大量的命令，再重启时，会加载过多不必要的数据。想在运行中重新生成AOF文件的优化版本，可在客户端执行：

```
127.0.0.1:7711 > BGREWRITEAOF
```

这样可在Disque不中断的情况下生成AOF文件的优化版本。



## DEAD LETTER QUEUE

Disque不像其他消息队列一样拥有DEAD LETTER QUEUE的设置。需要通过使用`GETJOB WITHCOUNTERS`或者`SHOW`命令检查任务的`NACK COUNTER` 以及 `ADDITIONAL COUNTER`来统计任务的失效次数，从而考虑额外的处理。



## 删除节点

在目标节点执行命令：

```
CLUSTER LEAVING yes
```

当前节点标记自身即将离开，并广播集群。对于集群其他节点而言reachable nodes 减少，REPLICATE的可用节点数目减少。在HELLO命令中该目标节点的优先级会降低（数值增大），并不会创建 `dummy ack`



然后在其他节点执行命令：

```
CLUSTER FORGET <old-node-id>
```

知会当前节点在集群信息中删去目标节点，但不影响其他节点的集群情况，所以需要在各个节点执行此命令来确保目标节点的离开。



- 标记为`CLUSTER LEAVING yes`的节点只能接受REPLICATE = 1的任务。此前作为副本的任务将变得不可用并且不会迁移。之后对于其他节点的REPLICATE的请求将执行 “external replication” ，即尝试在其他节点执行副本操作，自身将不再作为可用副本。阻塞等待任务的程序将会解除阻塞并接收ERROR消息：`(error) LEAVING This node is leaving the cluster, please connect to a different one`
- 可以使用`CLUSTER LEAVING no `来重新标记为正常集群节点


- 对于离开集群的目标节点，需要多次执行`CLUSTER FORGET <old-node-id>`来删除联结的旧集群节点。更有效的方法是，直接删除对应的nodes.conf文件，将清空节点所有的集群信息




## 任务ID

Disque 中的任务由类似这样的 ID 进行唯一标识： `D-dcb833cf-8YL1NT17e9+wsA/09NqxscQI-05a1` 。

每个任务 ID 总是以 `"D"` 开头，由正好 40 个字符组成。

任务 ID 可以被分为几个不同的部分：

```
D- | dcb833cf | 8YL1NT17e9+wsA/09NqxscQI | 05a1
```

以下是各个部分的含义：

1. `D-` 是 ID 的前缀。
2. `dcb833cf` 是一个节点 ID 的前 8 个字节，正是这个 ID 所记录的节点创建了这个任务。
3. `8YL1NT17e9+wsA/09NqxscQI` 是在base64中编码的144位ID伪随机部分 。
4. `05a1` 是以分钟计算的任务生存时间。这个值使得任务 ID 可以安全地过期，即使节点并不知道任务的具体表示。如果RETRY = 0的任务，编码将会是偶数，否则总会是奇数。

任务 ID 是 `ADDJOB` 命令在成功创建任务时的返回值， 也是 `GETJOB` 输出的一部分， 并且在任务已经交由工作进程（worker）处理完毕时， 可以使用任务 ID 向 Disque 报告该任务已经完成。

任务 ID 包含了一部分节点 ID ， 这使得为给定队列处理任务的工作进程可以很容易地知道自己正在处理的任务是由哪个节点创建的， 然后直接向该节点发送请求， 这样可以避免将请求发送至其他无关的节点， 从而提高效率。

因为任务 ID 只是用了 32 个二进制位来储存节点 ID ， 所以在包含 100 个节点的 Disque 集群里面， 这个 32 位 ID 出现碰撞的几率可以通过生日悖论（birthday paradox）计算得出：

```
P(100,2^32) = .000001164
```

在发生碰撞的情况下， 工作进程可能会做出不高效的选择。

任务 ID 中的 144位 ID 几乎不可能出现碰撞， 它由以下方法计算得出的：

```
144 bit ID = HIGH_144_BITS_OF_SHA1(seed || counter)
```

其中 `seed` 是服务器在启动时通过 `/dev/urandom` 生成的， 而 `counter` 则是一个在每次生成 ID 时都会进行自增的计数器。



## 一些基本理念

###乱序

Disque更准确的说法是消息代理而不是消息队列。因为Disque不保证消息的顺序，只能尽可能的确保FIFO。对于Retry的任务，会按相对位置重新排列到队首，而不是队尾。

###至少消费一次

Disque通过RETRY来确保消息一定发送成功。

### 最多消费一次

RETRY = 0  的任务，只会被最多消费一次， 并且在  ADDJOB 的时候需要显式的声明  REPLICATE = 1 



## 一些测试获得的结论

1. 任务的REPLICATE 并不会动态的检查确保满足设置。REPLICATION = 2的任务，如果存有任务副本的有效节点先后失效（即便间隔一定时长），在未开启AOF的情况下，任务会丢失。
2. 任务一般情况下会在唯一的有效节点保持queued状态，其他副本节点保持active状态。如果原节点失效，会在随机一个有效副本节点requeue, 状态从active转变为queued。
3. 单节点开启AOF情况和多节点集群情况一样，能够保证任务数据恢复。但需要等待RETRY时间到达，任务状态从active变成queued，才能`GETJOB`取出任务。 




## PYTHON客户端

- [disq](https://github.com/ryansb/disq)（[PyPi](https://pypi.python.org/pypi/disq)）
- [pydisque](https://github.com/ybrs/pydisque)（[PyPi](https://pypi.python.org/pypi/pydisque)）

推荐使用pydisque，实现的命令相对disq更完整。



## 参考

https://github.com/antirez/disque

https://my.oschina.net/llzx373/blog/409937

http://www.phperz.com/article/15/0913/155957.html

http://marshal.ohtly.com/2016/01/11/Disque-and-disk-persistence/



## EDITOR

[YIMIAN.INC](https://www.yimian.com.cn/)

Liangzhichong@yimian.com.cn

2017-06-28
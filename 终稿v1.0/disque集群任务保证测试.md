## 测试环境

DISQUE_VERSION : 1.0-rc1

操作系统：Ubuntu 17.04



##测试一：

###测试目的：
检查集群中节点失效后信息的丢失情况： 逐步关闭各个节点，查看剩余节点的任务状况，然后重启节点，查看各节点的任务状况


####测试集群：
disque -h 127.0.0.1 -p 7711
disque -h 127.0.0.1 -p 7712
disque -h 127.0.0.1 -p 7713

####各节点配置情况
appendonly = no， aof-enqueue-job-once = no

####测试流程：
1.  在节点7711 向队列testqueue添加3个replicate不等的任务a,b,c：  
  分别是：  
  任务a: `addjob testqueue a 0 replicate 1 retry 300`  
  任务b: `addjob testqueue b 0 replicate 2 retry 300`  
  任务c: `addjob testqueue c 0 replicate 3 retry 300`  

  ​

2. 检查此时各节点的任务情况：

  7711节点：  
  `qscan  :  testqueue`  
  `jscan  :  a, b, c`  
  `qstat testqueue: len = 3, jobsin = 3, jobsout = 0`  

  ​

  7712节点：  
  `qscan  : empty`  
  `jscan  : c`  
  `qstat testqueue: nil`  

  ​

  7713节点：  
  `qscan : empty`  
  `jscan : b, c`  
  `qstat testqueue: nil`

  ​

3. 检查关闭节点后各节点任务情况  

  1. 单独关闭7711节点 : 7711 shutdown  

  检查各节点情况：  

  7712节点：  
  `qscan : testqueue`  
  `jscan : c`  
  `qstat testqueue: len = 1, jobsin = 1, jobsout =  0`  

  ​

  7713节点：  
  `qscan : testqueue`  
  `jscan : b , c`  
  `qstat testqueue: len = 1, jobsin = 1, jobsout = 0`  

  结论： replicate = 1 的任务a在节点失效的情况下会丢失  

  ​

  2. 同时关闭7711和7713节点  

  查看7712节点情况：  

  `qscan: testqueue`  
  `jscan: c`  
  `qstat testqueue: len = 1, jobsin = 1, jobsout = 0`  
  `getjob from testqueue: 只能获取任务c`  

  ​

  结论： replicate = 2 的任务b，在第一个节点关闭的时候，没有自动备份到新节点来保证 replicate数量， 导致第二个节点关闭的时候，任务会丢失。此时任务a,b均丢失

  ​

  3. 重启7711及7713节点，查看各节点状况：  

  7711节点：  
  `qscan: empty`  
  `jscan: empty`  
  `qstat testqueue: nil`  

  ​

  7713节点：  
  `qscan: empty`  
  `jscan: empty`  
  `qstat testqueue: nil`  

  ​

  结论： 节点重启后，集群并不会将原节点信息写入节点，现有任务c (replicate = 3)也没有写到重启后的节点中

  ​


###测试一结论：
1. 任务a（replicate=1)在关闭节点7711的时候丢失

2. replicate = 2 的任务，在第一个节点关闭的时候，没有自动备份到新节点来保证 replicate数量， 导致第二个节点关闭的时候，任务会丢失

3. 节点重启后，默认的队列信息为空，执行`getjob from testqueue`后会从集群的其他节点获取目标队列的信息。对于replicate = 1 的任务，如果在其他节点进行过`getjob from testqueue`操作， 则会把该任务复制到新节点, 此时如果原节点失效，集群中依旧有该任务存在于新节点中。说明各节点的队列独立，qstat信息独立

   ​



##测试二：
###测试目的： 
检查开启aof情况下，集群的任务数据完整性

####测试集群：
disque -h 127.0.0.1 -p 7711
disque -h 127.0.0.1 -p 7712
disque -h 127.0.0.1 -p 7713

####各节点配置情况
appendonly = yes， aof-enqueue-job-once = no, appendfsync everysec

####测试流程：
1.  在节点7711 向队列testqueue添加1个replicate = 2 的任务a：   
  `addjob testqueue a 0 replicate 2 retry 300`

  检查各节点情况：  
    7711节点：  
    `qscan: testqueue`  
    `jscan: a`  
    `qstat testqueue: len = 1, jobsin = 1, jobsout = 0`  
    `SHOW a<job-id> : state: queued`   

  ​

    7712节点:  
    `qscan: empty`  
    `jscan: a`  
    `qstat testqueue: nil`  
    `SHOW a<job-id> : stat: active`  

  ​

    7713节点:  
    `qscan: empty`  
    `jscan: empty`  
    `qstat testqueue: nil`  
    `SHOW a<job-id> : nil`  

  即任务有效节点为7711, 7712  

  ​

  ######考虑单个有效节点失效情况

2. 单独关闭节点7711：

  检查各节点情况：

    7712节点:  
    `qscan: testqueue`  
    `jscan: a`  
    `qstat testqueue: len = 1, jobsin = 1, jobsout = 0`  
    `SHOW a<job-id> : stat: queued`  

  ​

    7713节点:  
    `qscan: empty`  
    `jscan: empty`  
    `qstat testqueue: nil`  
    `SHOW a<job-id> : nil`  

  ​

    结论： 拥有任务的副本的节点， 会在其中一个有效节点enqueue该任务，其他节点的任务副本状态为active。 若原节点失效，任务会在retry时间到达后在其他拥有该任务的有效节点requeue， 该节点的任务状态由active变为queued， 并且产生qscan信息。 


3. 重启7711节点：

  情况1： 不加其他参数直接重启 `./src/disque-server ./disque.conf`  

    7711节点：  
    `qscan: empty`  
    `jscan: a`  
    `qstat testqueue: nil`  
    `SHOW a<job-id> : stat: active`  

  ​

    7712节点:  
    `qscan: testqueue`  
    `jscan: a`  
    `qstat testqueue: len = 1, jobsin = 1, jobsout = 0`  
    `SHOW a<job-id> : stat: queued`  

  ​

    7713节点:  
    `qscan: empty`  
    `jscan: empty`  
    `qstat testqueue: nil`  
    `SHOW a<job-id> : nil`  

  ​

  情况2： 添加aof-enqueue-jobs-once参数重启   `./src/disque-server ./disque.conf --aof-enqueue-jobs-once yes`  

    7711节点:  
    `qscan: testqueue`  
    `jscan: a`  
    `qstat testqueue: len = 1, jobsin = 1, jobsout = 0`  
    `SHOW a<job-id> : stat: queued`  

  ​

    7712节点:  
    `qscan: testqueue`  
    `jscan: a`  
    `qstat testqueue: len = 1, jobsin = 1, jobsout = 0`  
    `SHOW a<job-id> : stat: queued`  

  ​

    7713节点:  
    `qscan: empty`  
    `jscan: empty`  
    `qstat testqueue: nil`  
    `SHOW a<job-id> : nil`  

  ​

    结论： 

  1. 任务默认情况下会主动寻找唯一有效节点enqueue。 
  2. 使用`--aof-enqueue-jobs-once yes`会恢复节点的任务队列信息。 
  3. 事实上， 创建 retry = 0的任务， replicate 只能设置为 1 ， 否则会导致addjob error， 所以retry = 0的任务不会造成情况2的情形。但考虑到aof的滞后性，重新加载队列数据是不安全的（“reloading queue data is not safe in the case of at-most-once jobs having the retry value set to 0”）所以官方默认设置`--aof-enqueue-jobs-once no `。 

  ​

  ######考虑全部任务有效节点失效情况  

4.  同时关闭节点7711及7712：  
  `SHUTDOWN`  

    此时7713节点：  
    `qscan = empty`  
    `jscan = empty`  
    `qstat testqueue: nil`  
    `SHOW a<job-id> : nil`  

  ​

5. 重启7711及7712节点：  

  情况1：不加其他参数直接重启 `./src/disque-server ./disque.conf`  
  启动顺序，先7711节点，然后7712节点  

  ​

  此时各节点情况：  

  7711节点：  
  `qscan: empty  -->  testqueue`   在retry时间后从启动时候的empty变为拥有信息  
  `jscan: a`  
  `qstat testqueue: nil  -->  len = 1, jobsin = 1, jobsout = 0`  在retry时间后从启动时候的nil变为拥有信息  
  `SHOW a<job-id>: stat: active  -->  queued`    在retry时间后从启动时候的active变为queued  

  ​

  7712节点:  
  `qscan: empty`  
  `jscan: a`  
  `qstat testqueue: nil`  
  `SHOW a<job-id>: stat: active`  

  ​

  7713节点:  
  `qscan: empty`  
  `jscan: empty`  
  `qstat testqueue: nil`  
  `SHOW a<job-id> : nil`  

  ​

  考虑调转节点的重启顺序，先7712, 再7711： 节点7711和7712情况刚好相反  

  结论： 有效节点重启后，最先启动的有效节点会优先enqueue任务， 并产生队列信息

  ​

  情况2： 添加aof-enqueue-jobs-once参数重启 `./src/disque-server ./disque.conf --aof-enqueue-jobs-once yes`  
  启动顺序，先7711节点，然后7712节点

  ​

  此时各节点情况：  

  7711节点：  
  `qscan: testqueue`  
  `jscan: a`  
  `qstat testqueue: len = 1, jobsin = 1, jobsout = 0`  
  `SHOW a<job-id>: stat: queue`  

  ​

  7712节点:  
  `qscan: empty`  
  `jscan: a`  
  `qstat testqueue: nil`  
  `SHOW a<job-id>: stat: active`  

  ​

  7713节点:  
  `qscan: empty`  
  `jscan: empty`  
  `qstat testqueue: nil`  
  `SHOW a<job-id> : nil`  

  ​

  考虑调转节点的重启顺序，先7712, 再7711： 情况不变  

  结论：` --aof-enqueue-jobs-once yes `参数恢复了节点的队列信息， 在retry时间内重启的节点的任务保持了queued状态，其他重启节点任务保持了active状态

  ​

###测试二结论：  
1. 通过开启AOF设置`appendonly = yes`， 会确保节点任务消息在节点重启后重新获得，不会丢失。

2. 重启的Disque默认加载内存中的任务数据，而不会填充队列，因为未确认的任务最终被重新排列。

3. 如果在启动节点服务时添加`--aof-enqueue-jobs-once yes`参数，将恢复节点最新的一次的队列情况。

4. 任务一般情况下会在唯一的有效节点保持queued状态，其他副本节点保持active状态。如果原节点失效，会在其他一个有效副本节点requeue, 状态从active转变为queued

   ​


##测试三：

###测试目的
检查开启aof情况下，单节点任务数据完整性

####测试节点：
disque -h 127.0.0.1 -p 7711

####节点配置情况
appendonly = yes， aof-enqueue-job-once = no, appendfsync everysec

####测试流程：  
1.  在节点7711 向队列testqueue添加1个replicate = 1 的任务a：  
   `addjob testqueue a 0 replicate 1 retry 300  `

   此时节点情况：  
   `qscan: testqueue`  
   `jscan: a`  
   `qstat testqueue: len = 1, jobsin = 1, jobsout = 0`  
   `SHOW a<job-id>: stat: queue`  

   ​

2. 关闭节点
  `shutdown`  

  ​

3. 重启节点

    情况1：不加其他参数直接重启 `./src/disque-server ./disque.conf`

    此时节点情况：  
    `qscan: empty  -->  testqueue`   在retry时间后从启动时候的empty变为拥有信息  
    `jscan: a`  
    `qstat testqueue: nil  -->  len = 1, jobsin = 1, jobsout = 0`  在retry时间后从启动时候的nil变为拥有信息  
    `SHOW a<job-id>: stat: active  -->  queued`    在retry时间后从启动时候的active变为queued  
    `GETJOB FROM testqueue : a`    成功获取到任务  

    ​

    情况2： 添加aof-enqueue-jobs-once参数重启   `./src/disque-server ./disque.conf --aof-enqueue-jobs-once yes`  
    `qscan: testqueue`  
    `jscan: a`  
    `qstat testqueue: len = 1, jobsin = 1, jobsout = 0`  
    `SHOW a<job-id>: stat: queue`  
    `GETJOB FROM testqueue : a`     成功获取到任务  

    ​

###测试三结论：
单节点开启aof情况和多节点集群情况一样，能够保证任务数据恢复。




*注*:
关闭节点使用了shutdown命令。在使用kill命令的情况下测试结果相同。考虑到appendfsync everysec，单步测试下结果相同是合理的。 未测试在高频率数据吞吐情况下节点失效时的数据持久化保证和丢失情况。


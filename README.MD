



# MySQL实战高手



## 1. SQL执行流程

![](./images/mysql-01.png)

### 1.1 客户端

+ Tomcat 通过mysql驱动连接MySQL服务器

+ 为了节约开启和关闭mysql连接的开销，Tomcat采用连接池来保存连接



### 1.2 服务端

```mysql
select id, name, age from user where id = 1
```

+ mysql服务端也通过连接池来处理客户端请求
+ mysql工作线程接受到mysql语句以后，就会交给**SQL接口**去执行
+ SQL 语句通过**解析器**被解析
  + 从users里面查询数据
  + 查询id是1的数据
  + 从需要的数据中提取id，name，age三个字段
+ **查询优化器**来优化查询路径
  + 直接定位到user表中id等于1的数据，提取字段
  + 查询user表，提取字段，从中过滤到id等于1的数据
+ **执行器**去调用存储引擎按照一定顺序和步骤执行SQL

+ **存储引擎**负责查询**缓存数据**和查询**磁盘数据**



## 2. InnoDB引擎

### 2.1 缓冲池和undo日志

![](./images/mysql-02.png)

**undo日志**：记录事务提交之前的值，以便回滚

**缓冲池**：把需要更新或者读取的数据放入缓冲池中，在从磁盘文件加载数据进入缓冲池的时候，需要加锁。



### 2.2 Redo日志

InnoDB 特有的日志文件。redo日志是为了防止机器宕机，缓存数据数据没有刷入磁盘。redo日志用于恢复数据。

![](./images/mysql-04.png)

**Innodb_flush_log_at_trx_commit=0**，**提交事务的时候不会redo log 刷入磁盘**。如果提交事务成功而**mysql宕机**，内存中的数据和redo日志都会丢失



![](./images/mysql-05.png)

**Innodb_flush_log_at_trx_commit=1**，**提交事务的时候必须把redo log 刷入磁盘**。如果提交事务成功而**mysql或者机器宕机**，内存中的数据丢失。mysql重启以后可以通过redo日志来恢复数据

![](./images/mysql-06.png)

**Innodb_flush_log_at_trx_commit=2**，**提交事务的时候必须把redo log 刷os cache**。如果提交事务成功而**机器宕机**，内存中的数据丢失。如果os cache没有把数据刷入磁盘，os cache 和 redo日志都会丢失。



### 2.3 Binlog日志

binlog日志属于mysql server的日志文件

![](./images/mysql-07.png)

+ 加载数据进入缓冲池
+ 把旧数据写入undo日志
+ 更新内存中的数据
+ 写入redo log buffer
+ 准备提交事务，把redo日志刷入文件
+ 准备提交事务，把binlog日志写入磁盘

![](./images/mysql-08.png)

**sync_binding 设置为0**，binlog日志不直接进入磁盘，而是写入os cache。如果机器宕机，os cache 和 binlog日志会丢失

![](./images/mysql-09.png)

**sync_binding 设置为1**，binlog日志直接进入磁盘，而是写入os cache。如果机器宕机 binlog日志不会丢失



### 2.4 基于redo日志和binlog完成事务提交

![](./images/mysql-10.png)

事务最终提交。把这次更新对应的binlog文件名和更新的binlog日志文件里的位置写入redo log日志里面。同时在redo log 日志里面写入一个commit

**假如5步骤以后宕机**，没有最终事务的标记在redo日志里面，判定这次事务不成功。

**假如6步骤以后宕机**，redo日志没有commit标记，判定事务提交失败

必须redo日志中有事务commit标记，redo日志和binlog完全一致，才判断事务成功。

![](./images/mysql-11.png)

即使在步骤8，刷入磁盘文件之前宕机了，那么redo日志也会恢复之前事务的提交，IO线程会把已经修改的值放入磁盘数据文件。



## 3. 案例实战：数据库的配置

**数据库部署的时候常选用的机器配置最低在8核16G以上**，正常在16核32G

一般Java应用系统部署在4核8G的机器上，每秒钟抗下500左右的并发访问量，差不多是比较合适的，当然这个也不一定。因为你得考虑一下，假设你每个请求花费1s可以处理完，那么你一台机器每秒也许只可以处理100个请求，但是如果你每个请求只要花费100ms就可以处理完，那么你一台机器每秒也许就可以处理几百个请求。



高并发的情况下，对性能要求极高。对于你Java系统接收到的每个请求，**耗时最多的还是发送网络请求到数据库上去，等待数据库执行一些SQL语句，返回结果给你**。所以其实我们常说你有一个Java系统压力很大，负载很高，但是其实你要明白一点，你这个Java系统其实主要的**压力和复杂都是集中在你依赖的那个MySQL数据库上的！**



因为你执行大量的增删改查的SQL语句的时候，**MySQL数据库需要对内存和磁盘文件进行大量的IO操作**，所以数据库往往是负载最高的！所以往往对一个数据库而言，都是选用8核16G的机器作为起步，最好是选用16核32G的机器更加合适一些，因为数据库需要执行大量的磁盘IO操作，他的每个请求都比较耗时一些，所以机器的配置自然需要高一些了。



## 4. 数据库压测

### 4.1 压测指标

#### Query Per Second

其实就是英文字面意思已经很明确了，QPS就是说，你的这个数据库每秒可以处理多少个请求，你大致可以理解为，一次请求就是一条SQL语句，也就是说这个数据库每秒可以处理多少个SQL语句。



#### Transaction Per Second

每秒可处理的事务量，这个TPS往往是用在数据库中较多一些，其实从字面意思就能看的出来，他就是说数据库每秒会处理多少次事务提交或者回滚。



#### IOPS

这个指的是机器的**随机IO并发处理的能力**，比如机器可以达到200 IOPS，意思就是说每秒可以执行200个随机IO读写请求。在内存中更新的脏数据库，最后都会由**后台IO线程在不确定的时间，刷回到磁盘里去，这就是随机IO的过程**。如果说IOPS指标太低了，那么会导致你内存里的脏数据刷回磁盘的效率就会不高。



#### 吞吐量

这个指的是机器的磁盘存储**每秒可以读写多少字节的数据量**。执行各种SQL语句的时候，提交事务的时候，其实都是大量的会写redo log之类的日志的，这些日志都会直接写磁盘文件。所以一台机器他的存储每秒可以读写多少字节的数据量，**就决定了他每秒可以把多少redo log之类的日志写入到磁盘里去**。一般来说我们写redo log之类的日志，都是对磁盘文件进行顺序写入的，也就是一行接着一行的写，不会说进行随机的读写，那么一般普通磁盘的顺序写入的吞吐量每秒都可以达到200MB左右。



#### latency

这个指标说的是往磁盘里**写入一条数据的延迟**。因为我们执行SQL语句和提交事务的时候，都需要顺序写redo log磁盘文件，所以此时你写一条日志到磁盘文件里去，到底是延迟1ms，还是延迟100us，这就对你的数据库的SQL语句执行性能是有影响的。



### 4.2 Sysbench 进行压测

基于sysbench构造测试表和测试数据

```bash
sysbench --db-driver=mysql --time=300 --threads=10 --report-interval=1 --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=root --mysql-db=test --tables=20 --table_size=1000000 oltp_read_write --db-ps-mode=disable prepare
```

+ --db-driver=mysql：这个很简单，就是说他基于mysql的驱动去连接mysql数据库，你要是oracle，或者sqlserver，那自然就是其他的数据库的驱动了

+ --time=300：这个就是说连续访问300秒

+ --threads=10：这个就是说用10个线程模拟并发访问

+ --report-interval=1：这个就是说每隔1秒输出一下压测情况

+ --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=test_user --mysql-password=test_user：就是说连接到哪台机器的哪个端口上的MySQL库，他的用户名和密码是什么

+ --mysql-db=test_db --tables=20 --table_size=1000000：就是说在test_db这个库里，构造20个测试表，每个测试表里构造100万条测试数据，测试表的名字会是类似于sbtest1，sbtest2这个样子的

+ oltp_read_write：这个就是说，执行oltp数据库的读写测试

+ --db-ps-mode=disable：这个就是禁止ps模式

  

**测试数据库的综合读写TPS，使用的是oltp_read_write**模式

```bash
sysbench --db-driver=mysql --time=300 --threads=10 --report-interval=1 --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=root --mysql-db=test --tables=20 --table_size=1000000 oltp_read_write --db-ps-mode=disable run
```



**测试数据库的只读性能，使用的是oltp_read_only**模式

```bash
sysbench --db-driver=mysql --time=300 --threads=10 --report-interval=1 --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=root --mysql-db=test --tables=20 --table_size=1000000 oltp_read_only --db-ps-mode=disable run
```



**测试数据库的删除性能，使用的是oltp_delete**模式：

```bash
sysbench --db-driver=mysql --time=300 --threads=10 --report-interval=1 --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=root --mysql-db=test --tables=20 --table_size=1000000 oltp_delete --db-ps-mode=disable run
```



**测试数据库的删除性能，使用的是oltp_update_index**模式：

```bash
sysbench --db-driver=mysql --time=300 --threads=10 --report-interval=1 --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=root --mysql-db=test --tables=20 --table_size=1000000 oltp_update_index --db-ps-mode=disable run
```



**测试数据库的删除性能，使用的是oltp_update_non_index**模式：

```bash
sysbench --db-driver=mysql --time=300 --threads=10 --report-interval=1 --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=root --mysql-db=test --tables=20 --table_size=1000000 oltp_update_non_index --db-ps-mode=disable run
```



**测试数据库的删除性能，使用的是oltp_insert**模式：

```bash
sysbench --db-driver=mysql --time=300 --threads=10 --report-interval=1 --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=root --mysql-db=test --tables=20 --table_size=1000000 oltp_insert --db-ps-mode=disable run
```



**测试数据库的删除性能，使用的是oltp_write_only**模式：

```bash
sysbench --db-driver=mysql --time=300 --threads=10 --report-interval=1 --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=root --mysql-db=test --tables=20 --table_size=1000000 oltp_write_only --db-ps-mode=disable run
```



最后完成压测之后，**可以执行下面的cleanup命令，清理数据。**

```bash
sysbench --db-driver=mysql --time=300 --threads=10 --report-interval=1 --mysql-host=127.0.0.1 --mysql-port=3306 --mysql-user=root --mysql-password=root --mysql-db=test --tables=20 --table_size=1000000 oltp_write_only --db-ps-mode=disable run
```

#### 压测结果

```bash
[ 297s ] thds: 10 tps: 1116.72 qps: 17876.46 (r/w/o: 15644.03/0.00/2232.43) lat (ms,95%): 16.41 err/s: 0.00 reconn/s: 0.00
[ 298s ] thds: 10 tps: 1055.59 qps: 16889.45 (r/w/o: 14777.27/0.00/2112.18) lat (ms,95%): 17.63 err/s: 0.00 reconn/s: 0.00
[ 299s ] thds: 10 tps: 1148.81 qps: 18373.91 (r/w/o: 16076.29/0.00/2297.62) lat (ms,95%): 16.41 err/s: 0.00 reconn/s: 0.00
[ 300s ] thds: 10 tps: 1110.97 qps: 17765.56 (r/w/o: 15545.61/0.00/2219.95) lat (ms,95%): 14.46 err/s: 0.00 reconn/s: 0.00
```

+ thds: 10，这个意思就是有10个线程在压测
+ tps: 1116.72，这个意思就是每秒执行了380.99个事务
+ qps: 17876.46，这个意思就是每秒可以执行7610.20个请求
+ (r/w/o: 15644.03/0.00/2232.43)，这个意思就是说，在每秒17876.46个请求中，有15644.03个请求是读请求，0.00个请求是写请求，2232.43个请求是其他的请求，就是对QPS进行了拆解
+ lat (ms, 95%): 21.33，这个意思就是说，95%的请求的延迟都在14.46毫秒以下
+ err/s: 0.00 reconn/s: 0.00，这两个的意思就是说，每秒有0个请求是失败的，发生了0次网络重连

#### 压测报告

```
SQL statistics:
    queries performed:
        read:                            4738538	// 这就是说在300s的压测期间执行了470万多次的读请求
        write:                           0				// 这是说在压测期间执行了0次的写请求
        other:                           676934		// 这是说在压测期间执行了67次的其他请求
        total:                           5415472	// 这是说在压测期间一共执行了540万次的写请求
    transactions:                        338467 (1128.19 per sec.)
    queries:                             5415472 (18051.05 per sec.)
    ignored errors:                      0      (0.00 per sec.)
    reconnects:                          0      (0.00 per sec.)

// 下面就是说，一共执行了300s的压测，执行了10万+的事务
General statistics:
    total time:                          300.0073s
    total number of events:              338467

Latency (ms):
         min:                                    2.35
         avg:                                    8.86
         max:                                  290.91
         95th percentile:                       20.00
         sum:                              2999396.83

Threads fairness:
    events (avg/stddev):           33846.7000/77.93
    execution time (avg/stddev):   299.9397/0.00

```



#### 压测中的机器性能

**CPU负载**

```
top - 15:52:00 up 42:35, 1 user, load average: 0.15, 0.05, 0.01
```

load average: 0.15, 0.05, 0.01这行信息，他说的是CPU在1分钟、5分钟、15分钟内的负载情况。

这里要给大家着重解释一下这个CPU负载是什么意思，**假设我们是一个4核的CPU**，此时如果你的CPU负载是0.15，这就说明，4核CPU中连一个核都没用满，4核CPU基本都很空闲，没啥人在用。

**如果你的CPU负载是1，那说明4核CPU中有一个核已经被使用的比较繁忙了，另外3个核还是比较空闲一些**。要是CPU负载是1.5，说明有一个核被使用繁忙，另外一个核也在使用，但是没那么繁忙，还有2个核可能还是空闲的。

如果你的CPU负载是4，那说明4核CPU都被跑满了，**如果你的CPU负载是6，那说明4核CPU被繁忙的使用还不够处理当前的任务，很多进程可能一直在等待CPU去执行自己的任务**。



**内存占用**

```
Mem: 33554432k total, 20971520k used, 12268339 free, 307200k buffers
```

这个其实很简单，**明显可以看出来就是总内存大概有32GB，已经使用了20GB左右的内存，还有10多G的内存是空闲的**，然后有大概300MB左右的内存用作OS内核的缓冲区了。

对于内存而言，同样是要在压测的过程中紧密的观察，**一般来说，如果内存的使用率在80%以内，基本都还能接受**，在正常范围内，但是如果你的机器的内存使用率到了70%~80%了，就说明有点危险了，此时就不要继续增加压测的线程数量和QPS了，差不多就可以了。



**磁盘读写**

使用dstat -d命令，会看存储的IO吞吐量是每秒钟读取103kb的数据，每秒写入211kb的数据，像这个存储IO吞吐量基本上都不算多的，因为**普通的机械硬盘都可以做到每秒钟上百MB**的读写数据量。

```bash
-dsk/total -
read writ
103k 211k
  0   11k
```



使用 dstat -r命令，会看到IOPS和写IOPS分别是多少，也就是说随机磁盘读取每秒钟多少次，随机磁盘写入每秒钟执行多少次，大概就是这个意思，一般来说，随机磁盘读写每秒在两三百次都是可以承受的

```bash
--io/total-
read writ
0.25 31.9
  0   253
  0   39.0
```



**网卡占用**

使用 dstat -n命令, 每秒钟网卡接收到流量有多少kb，每秒钟通过网卡发送出去的流量有多少kb，通常来说，如果你的机器使用的是千兆网卡，那么每秒钟网卡的总流量也就在100MB左右，甚至更低一些。

```bash
-net/total-
recv send
16k  17k
```



### 4.3 Prometheus + Grafana 可视化监控

下载prometheus-2.20.0.darwin-amd64, 配置prometheus.yml

```yaml
# my global config
global:
  scrape_interval:     15s # Set the scrape interval to every 15 seconds. Default is every 1 minute.
  evaluation_interval: 15s # Evaluate rules every 15 seconds. The default is every 1 minute.
  # scrape_timeout is set to the global default (10s).

  # Attach these labels to any time series or alerts when communicating with
  # external systems (federation, remote storage, Alertmanager).
  external_labels:
      monitor: 'codelab-monitor'

# Load rules once and periodically evaluate them according to the global 'evaluation_interval'.
rule_files:
  # - "first.rules"
  # - "second.rules"

# A scrape configuration containing exactly one endpoint to scrape:
# Here it's Prometheus itself.
scrape_configs:
  # The job name is added as a label `job=<job_name>` to any timeseries scraped from this config.
  - job_name: 'prometheus'

    # metrics_path defaults to '/metrics'
    # scheme defaults to 'http'.

    static_configs:
      - targets: ['localhost:9090']

  - job_name: mysql
    static_configs:
      - targets: ['127.0.0.1:9104']
        labels:
          instance: db1
```



安装node exporter 并启动**./node_exporter&**

```bash
➜  node_exporter-1.0.1.darwin-amd64 ./node_exporter&
[1] 9625
➜  node_exporter-1.0.1.darwin-amd64 level=info ts=2020-07-25T09:02:10.036Z caller=node_exporter.go:177 msg="Starting node_exporter" version="(version=1.0.1, branch=HEAD, revision=3715be6ae899f2a9b9dbfd9c39f3e09a7bd4559f)"
level=info ts=2020-07-25T09:02:10.036Z caller=node_exporter.go:178 msg="Build context" build_context="(go=go1.14.4, user=root@4c8e5c628328, date=20200616-12:52:07)"
level=info ts=2020-07-25T09:02:10.036Z caller=node_exporter.go:105 msg="Enabled collectors"
level=info ts=2020-07-25T09:02:10.036Z caller=node_exporter.go:112 collector=boottime
level=info ts=2020-07-25T09:02:10.036Z caller=node_exporter.go:112 collector=cpu
level=info ts=2020-07-25T09:02:10.036Z caller=node_exporter.go:112 collector=diskstats
level=info ts=2020-07-25T09:02:10.036Z caller=node_exporter.go:112 collector=filesystem
level=info ts=2020-07-25T09:02:10.036Z caller=node_exporter.go:112 collector=loadavg
level=info ts=2020-07-25T09:02:10.036Z caller=node_exporter.go:112 collector=meminfo
level=info ts=2020-07-25T09:02:10.036Z caller=node_exporter.go:112 collector=netdev
level=info ts=2020-07-25T09:02:10.036Z caller=node_exporter.go:112 collector=textfile
level=info ts=2020-07-25T09:02:10.036Z caller=node_exporter.go:112 collector=time
level=info ts=2020-07-25T09:02:10.036Z caller=node_exporter.go:112 collector=uname
level=info ts=2020-07-25T09:02:10.037Z caller=node_exporter.go:191 msg="Listening on" address=:9100
level=info ts=2020-07-25T09:02:10.037Z caller=tls_config.go:170 msg="TLS is disabled and it cannot be enabled on the fly." http2=false
```

安装mysql exporter 配置**.my.cnf** 并启动 **./mysqld_exporter --config.my-cnf=".my.cnf" &**

```cnf
[client]
host=127.0.0.1
user=root
password=root
```

启动prometheus,  **./prometheus --storage.tsdb.retention=30d &**

```
prometheus-2.20.0.darwin-amd64 level=warn ts=2020-07-25T09:16:23.542Z caller=main.go:297 deprecation_notice="'storage.tsdb.retention' flag is deprecated use 'storage.tsdb.retention.time' instead."
level=info ts=2020-07-25T09:16:23.542Z caller=main.go:343 msg="Starting Prometheus" version="(version=2.20.0, branch=HEAD, revision=e5a06b483527d4fe0704b8fa3a2b475b661c526f)"
level=info ts=2020-07-25T09:16:23.542Z caller=main.go:344 build_context="(go=go1.14.6, user=root@ac954b6d5c6e, date=20200722-19:00:45)"
level=info ts=2020-07-25T09:16:23.542Z caller=main.go:345 host_details=(darwin)
level=info ts=2020-07-25T09:16:23.542Z caller=main.go:346 fd_limits="(soft=65536, hard=65536)"
level=info ts=2020-07-25T09:16:23.542Z caller=main.go:347 vm_limits="(soft=unlimited, hard=unlimited)"
level=info ts=2020-07-25T09:16:23.543Z caller=main.go:684 msg="Starting TSDB ..."
level=info ts=2020-07-25T09:16:23.543Z caller=web.go:524 component=web msg="Start listening for connections" address=0.0.0.0:9090
level=info ts=2020-07-25T09:16:23.548Z caller=head.go:641 component=tsdb msg="Replaying on-disk memory mappable chunks if any"
level=info ts=2020-07-25T09:16:23.548Z caller=head.go:655 component=tsdb msg="On-disk memory mappable chunks replay completed" duration=6.048µs
level=info ts=2020-07-25T09:16:23.548Z caller=head.go:661 component=tsdb msg="Replaying WAL, this may take a while"
level=info ts=2020-07-25T09:16:23.553Z caller=head.go:713 component=tsdb msg="WAL segment loaded" segment=0 maxSegment=4
level=info ts=2020-07-25T09:16:23.578Z caller=head.go:713 component=tsdb msg="WAL segment loaded" segment=1 maxSegment=4
level=info ts=2020-07-25T09:16:23.579Z caller=head.go:713 component=tsdb msg="WAL segment loaded" segment=2 maxSegment=4
level=info ts=2020-07-25T09:16:23.583Z caller=head.go:713 component=tsdb msg="WAL segment loaded" segment=3 maxSegment=4
level=info ts=2020-07-25T09:16:23.583Z caller=head.go:713 component=tsdb msg="WAL segment loaded" segment=4 maxSegment=4
level=info ts=2020-07-25T09:16:23.583Z caller=head.go:716 component=tsdb msg="WAL replay completed" checkpoint_replay_duration=45.611µs wal_replay_duration=35.393816ms total_replay_duration=35.45933ms

```

http://localhost:9090/targets

![](./images/mysql-12.png)

启动grafanam, **brew services start grafana**

**配置 prometheus数据源**

![](./images/mysql-14.png)

 从 https://github.com/percona/grafana-dashboards 导入 mysql_overview.json

![](./images/mysql-13.png)



## 5. 缓存池

![](./images/mysql-15.png)

Buffer Pool是数据库中的核心组件，因为增删改操作首先就是针对这个内存中的Buffer Pool里的数据执行的，同时配合了后续的redo log、刷磁盘等机制和操作。

所以Buffer Pool就是数据库的一个内存组件，里面缓存了磁盘上的真实数据，然后我们的Java系统对数据库执行的增删改操作，其实主要就是对这个内存数据结构中的缓存数据执行的。



### 5.1 缓冲池数据结构

![](./images/mysql-17.png)

```properties
[server]
Innodb_buffer_pool_size = 1247483648
```

**数据页**: 磁盘文件中有很多数据页，一个数据页包含多行数据，默认大小16kb

**缓存页**：buffer pool的有缓存页可以和磁盘数据页一一对应，都是16kb

**缓存页描述数据**：**数据页所属表空间，数据页标号**。这个描述信心本身也是一块数据。在buffer pool中，**每个缓存页的描述数据在最前面。然后是每个缓存页。**每个描述数据大概800字节，**描述数据占buffer pool 5%左右**



### 5.2 Free链表

![](./images/mysql-16.png)

数据库的Buffer Pool包含了很多个缓存页，同时每个缓存页还有一个**描述数据**，也可以叫做是控制数据。

数据库只要一启动，就会按照你设置的Buffer Pool大小，去找操作系统申请一块内存区域，作为Buffer Pool的内存区域。然后当内存区域申请完毕之后，数据库就会按照默认的**缓存页的16KB的大小以及对应的800个字节左右的描述数据的大小**，在Buffer Pool中划分出来一个一个的缓存页和一个一个的他们对应的描述数据。



#### 数据读取

数据库运行起来之后，不停的执行增删改查的操作，此时就需要不停的从磁盘上读取一个一个的数据页放入Buffer Pool中的对应的缓存页里去，把数据缓存起来，那么以后就可以对这个数据在内存里执行增删改查了。因为默认情况下磁盘上的**数据页和缓存页是一 一对应起来的，都是16KB，一个数据页对应一个缓存页**。



刚开始数据库启动的时候，可能所有的缓存页都是空闲的，因为此时可能是一个空的数据库，一条数据都没有，所以此时所有缓存页的描述数据块，都会被放入这个**free链表中**。这个free链表里面就是各个缓存页的描述数据块，只要缓存页是空闲的，那么他们对应的描述数据块就会加入到这个free链表中，每个节点都会双向链接自己的前后节点，**组成一个双向链表**。

这个free链表，他本身其实就是由Buffer Pool里的描述数据块组成的，你可以认为是每个描述数据块里都有两个指针，**一个是free_pre，一个是free_next**，分别指向自己的上一个free链表的节点，以及下一个free链表的节点。



磁盘上的数据页读取到对应的缓存页里去，同时把相关的一些描述数据写入缓存页的描述数据块里去，比如这个数据页所属的表空间之类的信息，**最后把那个描述数据块从free链表里去除**。



#### 数据页缓存哈希表

**数据库还会有一个哈希表数据结构，他会用表空间号+数据页号，作为一个key，然后缓存页的地址作为value。**当你要使用一个数据页的时候，通过**“表空间号+数据页号”作为key**去这个哈希表里查一下，如果没有就读取数据页，如果已经有了，就说明数据页已经被缓存了。



### 5.3 Flush链表

![](./images/mysql-19.png)

**脏页**

更新的数据页都会在Buffer Pool的缓存页里，供在内存中直接执行增删改的操作。**接着你肯定会去更新Buffer Pool的缓存页中的数据**，此时一旦**更新了缓存页中的数据，那么缓存页里的数据和磁盘上的数据页里的数据，是不是就不一致了**。这个时候，我们就说缓存页是脏数据，脏页

数据库在这里引入了另外一个跟free链表类似的**flush链表**，这个flush链表本质也是通过缓存页的描述数据块中的两个指针，让被修改过的缓存页的描述数据块，**组成一个双向链表**。

凡是被修改过的缓存页，都会把他的描述数据块加入到flush链表中去，flush的意思就是这些都是脏页，后续都是要flush刷新到磁盘上去的



### 5.4 LRU策略淘汰缓存

![](./images/mysql-20.png)

因为只要你把一个数据页加载到一个空闲缓存页里去，free链表中就会减少一个空闲缓存页。所以，当你不停的把磁盘上的数据页加载到空闲缓存页里去，**free链表中不停的移除空闲缓存页，迟早有那么一瞬间，你会发现free链表中已经没有空闲缓存页了**

此时无法从磁盘上加载新的数据页到缓存页里去了，那么此时你只有一个办法，就是**淘汰掉一些缓存页**。你必**须把一个缓存页里被修改过的数据，给他刷到磁盘上的数据页里去，然后这个缓存页就可以清空了，让他重新变成一个空闲的缓存页**。



引入一个新的LRU链表了，**这个所谓的LRU就是Least Recently Used**，最近最少使用的意思。假设我们从磁盘加载一个数据页到缓存页的时候，就把这个缓存页的描述数据块放到LRU链表头部去，那么只要有数据的缓存页，他都会在LRU里了，而且最近被加载数据的缓存页，都会放到LRU链表的头部去。后续你只要查询或者修改了这个缓存页的数据，也要把这个缓存页挪动到LRU链表的头部去，也就是说最近被访问过的缓存页，一定在LRU链表的头部。



当缓存页没有一个空闲的时候，要找出来那个最近最少被访问的缓存页去刷入磁盘？此时你就直接**在LRU链表的尾部找到一个缓存页，他一定是最近最少被访问的那个缓存页！**



#### 简单LRU算法问题

![](./images/mysql-21.png)

一个LRU机制在实际运行过程中，是会存在巨大的隐患的。

假设现在有两个空闲缓存页，然后在加载一个数据页的时候，**连带着把他的一个相邻的数据页也加载到缓存里去了，正好每个数据页放入一个空闲缓存页**。但是接下来呢，实际上只有一个缓存页是被访问了，另外一个**通过预读机制加载的缓存页，其实并没有人访问**，此时这两个缓存页可都在LRU链表的前面。

假如没有空闲缓存页了，那么此时要加载新的数据页了，是不是就要从LRU链表的尾部把所谓的“最近最少使用的一个缓存页”给拿出来，刷入磁盘，然后腾出来一个空闲缓存页了。

这个时候，**如果你把上图中LRU尾部的那个缓存页刷入磁盘然后清空其实并不合理。他可是之前一直频繁被人访问的啊！**只不过在这一个瞬间，被新加载进来的两个缓存页给占据了LRU链表前面的位置，尤其是第二个缓存页，居然还是通过预读机制加载进来的，根本就不会有人访问！

**预加载读**

+ 有一个参数是innodb_read_ahead_threshold，他的默认值是56，意思就是**如果顺序的访问了一个区里的多个数据页，访问的数据页的数量超过了这个阈值，此时就会触发预读机制**，把下一个相邻区中的所有数据页都加载到缓存里去
+ **如果Buffer Pool里缓存了一个区里的13个连续的数据页**，而且这些数据页都是比较频繁会被访问的，此时就会直接触发预读机制，把这个区里的其他的数据页都加载到缓存里去
+ innodb_read_ahead_threshold默认是关闭的
+ 全表扫描



#### 冷热数据的LRU链表

所以真正的LRU链表，会被拆分为两个部分，一部分是热数据，一部分是冷数据，这个冷热数据的比例是由**innodb_old_blocks_pct**参数控制的，他默认是37，也就是说冷数据占比37%。

这个时候，LRU链表实际上看起来是下面这样子的。

![](./images/mysql-22.png)

首先数据页第一次被加载到缓存的时候，**缓存页会被放在冷数据区域的链表头部**。

如果你刚加载了一个数据页到那个缓存页，他是在冷数据区域的链表头部，然后立马（在1ms以内）就访问了一下这个缓存页，之后就再也不访问他了呢，这样也是不合理的。

MySQL设定了一个规则，他设计了一个**innodb_old_blocks_time**参数，默认值1000，也就是1000毫秒

+ 必须是一个数据页被加载到缓存页之后，在1s之后，你访问这个缓存页，他才会被挪动到热数据区域的链表头部去。因为假设你加载了一个数据页到缓存去，然后过了1s之后你还访问了这个缓存页，说明你后续很可能会经常要访问它，这个时间限制就是1s.

+ **在1s内你就访问缓存页，此时他是不会把这个缓存页放入热数据区域的头部的。**



#### LRU链表冷热数据区域优化

接着我们来看看LRU链表的热数据区域的一个性能优化的点，就是说，在热数据区域中，如**果你访问了一个缓存页，是不是应该要把他立马移动到热数据区域的链表头部去，但是其实没有必要**。热数据区域里的缓存页可能是经常被访问的，所以这么频繁的进行移动是不是性能也并不是太好？也没这个必要。

LRU链表的热数据区域的访问规则被优化了一下，**即你只有在热数据区域的后3/4部分的缓存页被访问了**，才会给你移动到链表头部去。



#### LRU链表中尾部的缓存页淘汰机制

![](./images/mysql-24.png)

**冷数据区域**

并不是在缓存页满的时候，才会挑选LRU冷数据区域尾部的几个缓存页刷入磁盘**，而是有一个后台线程，他会运行一个定时任务，这个定时任务每隔一段时间就会把LRU链表的冷数据区域的尾部的一些缓存页**，刷入磁盘里去，清空这几个缓存页，把他们加入回free链表去！



**热数据区域**

如果仅仅是把LRU链表中的冷数据区域的缓存页刷入磁盘，明显不够啊，因为在lru链表的**热数据区域里的很多缓存页可能也会被频繁的修改**，难道他们永远都不刷入磁盘中了吗？所以这个后台线程同时也会在MySQL不怎么繁忙的时候，**找个时间把flush链表中的缓存页都刷入磁盘中**，这样被你修改过的数据，迟早都会刷入磁盘的！**只要flush链表中的一波缓存页被刷入了磁盘，那么这些缓存页也会从flush链表和lru链表中移除，然后加入到free链表中去！**



**没有空闲缓存页**

可能所有的free链表都被使用了，然后flush链表中有一大堆被修改过的缓存页，lru链表中有一大堆的缓存页，根据冷热数据进行了分离，大致是如此的效果。这个时候如果要从磁盘加载数据页到一个空闲缓存页中**，此时就会从LRU链表的冷数据区域的尾部找到一个缓存页，他一定是最不经常使用的缓存页！然后把他刷入磁盘和清空，然后把数据页加载到这个腾出来的空闲缓存页里去！**



### 5.5 生产经验

#### 多个buffer pool 优化并发

![](./images/mysql-25.png)

现在多个线程来并发的访问这个Buffer Pool了，此时他们都是在访问内存里的一些共享的数据结构，比如说缓存页、各种链表之类的。多线程并发访问一个Buffer Pool，必然是要加锁的，然后让一个线程先完成一系列的操作，比如说加载数据页到缓存页，更新free链表，更新lru链表，然后释放锁，接着下一个线程再执行一系列的操作。



即使就一个Buffer Pool，即使多个线程会加锁串行着排队执行，其实性能也差不到哪儿去。因为大部分情况下，**每个线程都是查询或者更新缓存页里的数据，这个操作是发生在内存里的，基本都是微秒级的**，很快很快，包括更新free、flush、lru这些链表，他因为都是基于链表进行一些指针操作，性能也是极高的。

```properties
[server]
innodb_buffer_pool_size = 8589934592
innodb_buffer_pool_instances = 4
```

我们给buffer pool设置了8GB的总内存，然后设置了他应该有4个Buffer Pool，此时就是说，每个buffer pool的大小就是2GB。这个时候，MySQL在运行的时候就会有4个Buffer Pool了！每个Buffer Pool负责管理一部分的缓存页和描述数据块，有自己独立的free、flush、lru等链表。



#### 基于chunk动态调整buffer pool

因为动态调整buffer pool大小，比如buffer pool本来是8G，运行期间你给调整为16G了，此时是怎么实现的呢。 就是需要这个时候向操作系统申请一块新的16GB的连续内存，**然后把现在的buffer pool中的所有缓存页、描述数据块、各种链表，都拷贝到新的16GB的内存中去。这个过程是极为耗时的，性能很低下，是不可以接受的！**

![](./images/mysql-26.png)





MySQL设计了一个chunk机制，也就是说buffer pool是由很多chunk组成的，他的大小是**innodb_buffer_pool_chunk_size**参数控制的，默认值就是**128MB**。假设我们给buffer pool设置一个总大小是8GB，然后有4个buffer pool，那么每个buffer pool就是2GB，此时每个buffer pool是由一系列的128MB的chunk组成的，也就是说每个**buffer pool会有16个chunk。**然后每个buffer pool里的每个chunk里就是一系列的描述数据块和缓存页，每个buffer pool里的多个chunk共享一套free、flush、lru这些链表，此时的话，看起来可能大致如下图所示。

**只要申请一系列的128MB大小的chunk就可以了，只要每个chunk是连续的128MB内存就行了。然后把这些申请到的chunk内存分配给buffer pool就行了。**



#### 生产环境buffer pool参数

建议一个比较合理的、健康的比例，是给buffer pool设置你的机器内存的50%~60%左右。比如你有32GB的机器，那么给buffer设置个20GB的内存，剩下的留给OS和其他人来用，这样比较合理一些。



接着确定了buffer pool的总大小之后，就得考虑一下设置多少个buffer pool，以及chunk的大小了。此时要记住，有一个很关键的公式就是：**buffer pool总大小=(chunk大小 * buffer pool数量)的倍数**

假设你的buffer pool的数量是16个，这是没问题的，那么此时**chunk大小 * buffer pool的数量 = 16 * 128MB = 2048MB**，然后buffer pool总大小如果是20GB，此时buffer pool总大小就是2048MB的10倍，这就符合规则了。



**SHOW ENGINE INNODB STATUS**

```
----------------------
BUFFER POOL AND MEMORY
----------------------
Total large memory allocated 274857984
Dictionary memory allocated 244224
Buffer pool size   16382
Free buffers       110
Database pages     16063
Old database pages 5948
Modified db pages  8885
Pending reads      0
Pending writes: LRU 0, flush list 0, single page 0
Pages made young 2987, not young 35398
149.34 youngs/s, 1769.81 non-youngs/s
Pages read 24749, created 542, written 4488
1146.04 reads/s, 25.40 creates/s, 222.59 writes/s
Buffer pool hit rate 962 / 1000, young-making rate 5 / 1000 not 59 / 1000
Pages read ahead 0.00/s, evicted without access 0.00/s, Random read ahead 0.00/s
LRU len: 16063, unzip_LRU len: 0
I/O sum[12500]:cur[990], unzip sum[0]:cur[0]
--------------
ROW OPERATIONS
--------------
0 queries inside InnoDB, 0 queries in queue
10 read views open inside InnoDB
Process ID=12665, Main thread ID=123145416486912, state: sleeping
Number of rows inserted 2448, updated 4896, deleted 2448, read 1020824
122.39 inserts/s, 244.79 updates/s, 122.39 deletes/s, 51038.25 reads/s
----------------------------
END OF INNODB MONITOR OUTPUT
============================


```



+ **Total memory allocated**，这就是说buffer pool最终的总大小是多少
+ **Buffer pool size**，这就是说buffer pool一共能容纳多少个缓存页
+ **Free buffers**，这就是说free链表中一共有多少个空闲的缓存页是可用的
+ **Database pages和Old database pages**，就是说lru链表中一共有多少个缓存页，以及冷数据区域里的缓存页数量
+ **Modified db pages**，这就是flush链表中的缓存页数量
+ **Pending reads和Pending writes**，等待从磁盘上加载进缓存页的数量，还有就是即将从lru链表中刷入磁盘的数量、即将从flush链表中刷入磁盘的数量
+ **Pages made young和not young**，这就是说已经lru冷数据区域里访问之后转移到热数据区域的缓存页的数量，以及在lru冷数据区域里1s内被访问了没进入热数据区域的缓存页的数量
+ **youngs/s和not youngs/s**，这就是说每秒从冷数据区域进入热数据区域的缓存页的数量，以及每秒在冷数据区域里被访问了但是不能进入热数据区域的缓存页的数量
+ Pages read xxxx, created xxx, written xxx，xx reads/s, xx creates/s, 1xx writes/s，这里就是说已经读取、创建和写入了多少个缓存页，以及每秒钟读取、创建和写入的缓存页数量
+ **Buffer pool hit rate xxx / 1000**，这就是说每1000次访问，有多少次是直接命中了buffer pool里的缓存的

+ **young-making rate xxx / 1000 not xx / 1000**，每1000次访问，有多少次访问让缓存页从冷数据区域移动到了热数据区域，以及没移动的缓存页数量
+ LRU len：这就是lru链表里的缓存页的数量
+ I/O sum：最近50s读取磁盘页的总数
+ I/O cur：现在正在读取磁盘页的数量



## 6. 磁盘存储模型

一个请求就直接对磁盘文件进行随机读写，然后更新磁盘文件里的数据，但是那必然导致执行请求的性能极差。**因为磁盘随机读写的性能是最差的**，所以直接更新磁盘文件，必然导致我们的数据库完全无法抗下任何一点点稍微高并发一点的场景。所以MySQL才设计了如此复杂的一套机制，通过内存里更新数据，然后写redo log以及事务提交，后台线程不定时刷新内存里的数据到磁盘文件里。通过这种方式保证，你每个更新请求，**尽量就是更新内存，然后顺序写日志文件**。



### 6.1 COMPACT存储格式

```sql
CREATE TABLE table_name (columns) ROW_FORMAT=COMPACT
ALTER TABLE table_name ROW_FORMAT=COMPACT
```

变长字段的长度列表，null值列表，数据头，column01的值，column02的值，column0n的值......



### 6.2 变长字段长度列表

现在有一行数据，他的几个字段的类型为VRACHAR(10)，CHAR(1)，CHAR(1)，那么他第一个字段是VARCHAR(10)，这个长度是可能变化的，所以这一行数据可能就是类似于：**hello a a**。另外一行数据，同样也是这几个字段，他的第一个字段的值可能是“hi”，后面两个字段也是“a”，所以这一行数据可能是类似于：**hi a a**。一共三个字段，第一个字段的长度是是不固定的，后面两个字段的长度都是固定的1个字符。



hello”的长度是5，十六进制就是0x05，“hello a a” 格式是0x05 null值列表 数据头 hello a a。0x02 null值列表 数据头 hi a a

```
0x05 null值列表 数据头 hello a a 0x02 null值列表 数据头 hi a a
```



比如一行数据有VARCHAR(10) VARCHAR(5) VARCHAR(20) CHAR(1) CHAR(1)，一共5个字段，其中三个是变长字段，此时假设一行数据是这样的：hello hi hao a a。**现在hello hi hao三个字段的长度分别是0x05 0x02 0x03**，但是实际存放在变长字段长度列表的时候，是**逆序放的**，所以一行数据实际存储可能是下面这样的：

```
0x03 0x02 0x05 null值列表 头字段 hello hi hao a a
```



### 6.3 NULL存储

```SQL
CREATE TABLE customer (
name VARCHAR(10) NOT NULL,
address VARCHAR(20),
gender CHAR(1),
job VARCHAR(30),
school VARCHAR(50)
) ROW_FORMAT=COMPACT;
```

“jack NULL m NULL xx_school”，他的5个字段里有两个字段都是NULL。有name和school两个变长字段是有值的，把他们的长度按照逆序放在变长字段长度列表中就可以了，如下所示：

```
0x09 0x04 NULL值列表 头信息 column1=value1 column2=value2 ... columnN=valueN
```

NULL值列表，这个NULL值列表是这样存放的，你所有允许值为NULL的字段，注意，是允许值为NULL，不是说一定值就是NULL了，只要是允许你为NULL的字段，在这里每个字段都有一个二进制bit位的值，**如果bit值是1说明是NULL，如果bit值是0说明不是NULL。**这一行数据的值是“jack NULL m NULL xx_school”，然后其中2个字段是null，2个字段不是null，所以**4个bit位应该是：1010**。**实际放在NULL值列表的时候，他是按逆序放的**，所以在NULL值列表里，放的是：**0101**

```
0x09 0x04 0101 头信息 column1=value1 column2=value2 ... columnN=valueN
```



### 6.4 数据头

每一行数据存储的时候，还得有40个bit位的数据头，这个数据头是用来描述这行数据的。40个bit位里，

+ 第一个bit位和第二个bit位，都是预留位，是没任何含义的。
+ 1个bit位是**delete_mask**，他标识的是这行数据是否被删除了，其实看到这个bit位，很多人可能已经反映过来了，这么说在MySQL里删除一行数据的时候，未必是立马把他从磁盘上清理掉，而是给他在数据头里搞1个bit标记
+ 1个bit位是**min_rec_mask**，就是说在B+树里每一层的非叶子节点里的最小值都有这个标记。
+ 4个bit位是**n_owned**
+ 13个bit位是**heap_no**，他代表的是当前这行数据在记录堆里的位置
+ 3个bit位的**record_type**，这就是说这行数据的类型, 0代表的是普通类型，1代表的是B+树非叶子节点，2代表的是最小值数据，3代表的是最大值数据
+ 16个bit的**next_record**，这个是指向他下一条数据的指针。



### 6.5 其他字段

数据是“jack NULL m NULL xx_school”, 实际上字符串这些东西都是根据我们数据库指定的字符集编码，进行编码之后再存储的

```
0x09 0x04 00000101 0000000000000000000010000000000000011001 616161 636320 6262626262
```

在实际存储一行数据的时候，会在他的真实数据部分，加入一些隐藏字段

+ **DB_ROW_ID字段**，这就是一个行的唯一标识，是他数据库内部给你搞的一个标识，不是你的主键ID字段。如果我们没有指定主键和unique key唯一索引的时候，他就内部自动加一个ROW_ID作为主键。
+ **DB_TRX_ID字段**，这是跟事务相关的，他是说这是哪个事务更新的数据，这是事务ID，这个后续我们讲解到事务的时候会跟大家说
+ **DB_ROLL_PTR字段**，这是回滚指针，是用来进行事务回滚的，也是我们后续在讲解事务的时候再详细说。

![](./images/mysql-27.png)

每一行数据是不是就是类似

```
0x09 0x04 00000101 0000000000000000000010000000000000011001 00000000094C（DB_ROW_ID）00000000032D（DB_TRX_ID） EA000010078E（DB_ROL_PTR）  616161 636320 6262626262
```



### 6.6 行溢出

比如有一个表的字段类型是VARCHAR(65532)，意思就是最大可以包含65532个字符，那也就是65532个字节，这就远大于16kb的大小了，也就是说这一行数据的这个字段都远超一个数据页的大小了！

这个时候实际上会在那一页里存储你这行数据，然后在那个字段中，仅仅包含他一部分数据，**同时包含一个20个字节的指针，指向了其他的一些数据页**，那些数据页用链表串联起来，存放这个VARCHAR(65532)超大字段里的数据。

![](./images/mysql-28.png)

### 6.7 数据页

16kb的数据页包含了文件头、数据页头、最小记录和最大记录、多个数据行、空闲空间、数据页目录、文件尾部。

![](./images/mysql-29.png)

+ 文件头占据了38个字节
+ 数据页头占据了56个字节
+ 最大记录和最小记录占据了26个字节
+ 数据行区域的大小是不固定的，空闲区域的大小也是不固定的，数据页目录的大小也是不固定的
+ 文件尾部占据8个字节。

![](./images/mysql-30.png)

假设我们现在要插入一行数据，此时数据库里可是一行数据都没有的，那么此时应该先是从磁盘上加载一个空的数据页到缓存页里。缓存页跟数据页是一 一对应的，他在磁盘上的时候就是数据页，数据页加载到缓存页里了，我们就叫他缓存页了。

此时在缓存页里插入一条数据，实际上就是在数据行那个区域里插入一行数据，然后空闲区域的空间会减少一些，此时当缓存页里插入了一行数据之后。

![](./images/mysql-31.png)

### 6.8 表空间

自己创建的表对应的表空间可能就是对应了一个“表名.ibd”数据文件。一个表空间的磁盘文件里，其实是有很多的数据页的。一个表空间里包含的数据页实在是太多了，不便于管理，所以在表空间里又引入了一个**数据区**的概念，英文就是**extent**

**一个数据区对应着连续的64个数据页**，每个数据页是16kb，所以一个数据区是1mb，然后256个数据区被划分为了一组。

对于表空间而言，他的第一组数据区的第一个数据区的前3个数据页，都是固定的，里面存放了一些描述性的数据。

+ **FSP_HDR**这个数据页，他里面就存放了表空间和这一组数据区的一些属性。
+ **IBUF_BITMAP**数据页，里面存放的是这一组数据页的所有insert buffer的一些信息。
+ **INODE**数据页，这里也是存放了一些特殊的信息

当我们需要执行crud操作的时候，说白了，就是从磁盘上的表空间的数据文件里，去加载一些数据页出来到Buffer Pool的缓存页里去使

**表空间存储结构如下**

![](./images/mysql-32.png)

### 6.9 顺序读写与随机读写

MySQL在实际工作时候的两种数据读写机制，一种是对redo log、binlog这种日志进行的磁盘顺序读写，一种是对表空间的磁盘文件里的数据页进行的磁盘随机读写。

![](./images/mysql-34.png)

#### 随机读写

MySQL在工作的时候，尤其是执行增删改操作的时候，肯定会先从表空间的磁盘文件里读取数据页出来，这个过程其实就是典型的磁盘随机读操作，随机的位置读取一个数据页到缓存，这就是**磁盘随机读**。磁盘随机读的性能是比较差的，所以不可能每次更新数据都进行磁盘随机读，必须是读取一个数据页之后放到Buffer Pool的缓存里去，下次要更新的时候直接更新Buffer Pool里的缓存页。

对于磁盘随机读来说，主要关注的性能指标是**IOPS和响应延迟**

**IOPS**就是说底层的存储系统每秒可以执行多少次磁盘读写操作，**底层存储的IOPS越高，你的数据库的并发能力就越高**。

**响应延迟**，推荐用SSD固态硬盘的，而不是机械硬盘，因为SSD固态硬盘的随机读写并发能力和响应延迟要比机械硬盘好的多



#### 顺序读写

磁盘顺序写的性能其实是很高的，某种程度上来说，几乎可以跟内存随机读写的性能差不多，尤其是在数据库里其实也用了os cache机制，就是redo log顺序写入磁盘之前，先是进入os cache，就是操作系统管理的内存缓存里。最核心关注的是磁盘每秒读写多少**数据量的吞吐量指标**，就是说每秒可以写入磁盘100MB数据和每秒可以写入磁盘200MB数据，对数据库的并发能力影响也是极大的。



## 7. Linux存储和IO调优

### 7.1 存储结构

Linux的存储系统分为VFS层、文件系统层、Page Cache缓存层、通用Block层、IO调度层、Block设备驱动层、Block设备层

![](./images/mysql-33.png)

当MySQL发起一次数据页的随机读写，或者是一次redo log日志文件的顺序读写的时候，实际上会把磁盘IO请求交给Linux操作系统的VFS层

**VPS层的作用，就是根据你是对哪个目录中的文件执行的磁盘IO操作，把IO请求交给具体的文件系统。**

举个例子，在linux中，有的目录比如/xx1/xx2里的文件其实是由NFS文件系统管理的，有的目录比如/xx3/xx4里的文件其实是由Ext3文件系统管理的，那么这个时候VFS层需要根据你是对哪个目录下的文件发起的读写IO请求



**文件系统会先在Page Cache这个基于内存的缓存里找你要的数据在不在里面**，如果有就基于内存缓存来执行读写，如果没有就继续往下一层走



请求会交给通用Block层，在这一层会把你对**文件的IO请求转换为Block IO请求**



IO请求转换为Block IO请求之后，会把这个Block IO请求交给**IO调度层**，在这一层里**默认是用CFQ公平调度算法**的。有一个SQL语句可能非常简单，比如update xxx set xx1=xx2 where id=1，他其实可能就只要更新磁盘上的一个block里的数据就可以了。但是有的SQL语句，比如说select * from xx where xx1 like "%xx%"可能需要IO读取磁盘上的大量数据。

那么此时如果基于公**平调度算法，就会导致他先执行第二个SQL语句的读取大量数据的IO操作，耗时很久，然后第一个仅仅更新少量数据的SQL语句的IO操作，就一直在等待他**，得不到执行的机会。一般建议**MySQL的生产环境，需要调整为deadline IO调度算法**，他的核心思想就是，任何一个IO操作都不能一直不停的等待，在指定时间范围内，都必须让他去执行



最后IO完成调度之后，就会决定哪个IO请求先执行，哪个IO请求后执行，此时可以执行的**IO请求就会交给Block设备驱动层**，然后最后经过驱动把IO请求发送给真正的存储硬件，也就是Block设备层



### 7.2 RAID存储结构

![](./images/mysql-35.png)

其实有的RAID磁盘冗余阵列技术里，是可以把你写入的同样一份数据，在两块磁盘上都写入的，这样可以让两块磁盘上的数据一样，作为冗余备份，然后当你一块磁盘坏掉的时候，可以从另外一块磁盘读取冗余数据出来，这一切都是RAID技术自动帮你管理的，不需要你操心，



#### RAID 0

RAID 0的意思，就是我们之前一幅图里画的，你有很多磁盘组成了一个阵列，然后你所有的数据是分散写入不同磁盘的，因为有多块磁盘，所以你的磁盘阵列的整体容量就很大，而且同时写入多块磁盘，让你的磁盘读写并发能力很强

![](./images/mysql-37.png)

这种模式下，最大的问题就是万一你磁盘坏了一块，那么就会丢失一部分数据了！所以一般如果你要严格保证磁盘数据不丢失的话，就得用RAID 1。



#### RAID 1

![](./images/mysql-37.png)

RAID 1的意思，就是两块磁盘为镜像关系，你写的所有数据，在两块磁盘上都有，形成了数据冗余，一块磁盘坏了，另外一块磁盘上还有数据。



#### RAID 10

![](./images/mysql-39.png)

**RAID 10，就是RAID 0 + RAID 1组合起来**，就是说当时生产环境的服务器部署，我们有6块磁盘组成了一个RAID 10的阵列，那么其实就是每2块磁盘组成一个RAID 1互为镜像的架构，存放的数据是冗余一样的，一共有3组RAID 1，然后对于每一组RAID 1写入数据的时候，是用RAID 0的思路，就是不同组的磁盘的数据是不一样的，但是同一组内的两块磁盘的数据是冗余一致的





## 8. 案例分析：RAID 锂电池放电引发的数据库抖动

![](./images/mysql-36.png)

RAID卡一般都配置有自己独立的锂电池或者是电容，如果服务器突然掉电了，无法接通电源了，RAID卡自己是基于锂电池来供电运行的，然后他会赶紧把缓存里的数据写入到阵列中的磁盘上去。 

但是**锂电池是存在性能衰减问题的，所以一般来说锂电池都是要配置定时充放电的**，也就是说**每隔30天~90天**（不同的锂电池厂商是不一样的），就会自动对锂电池充放电一次，这可以延长锂电池的寿命和校准电池容量。如果你要是不这么做的话，那么可能锂电池用着用着就会发现容量不够了，可能容纳的电量在你服务器掉电之后，都没法一次性把缓存里的数据写回磁盘上去，那就会导致数据丢失了！

锂电池充放电的过程中，**RAID的缓存级别会从write back变成write through**，我们通过RAID写数据的时候，IO就直接写磁盘了，如果写内存的话，性能也就是0.1ms这个级别，但是直接写磁盘，就性能退化10倍到毫秒级了！



+ **给RAID卡把锂电池换成电容**，电容是不用频繁充放电的，不会导致充放电的性能抖动，还有就是电容可以支持透明充放电，就是自动检查电量，自动进行充电，不会说在充放电的时候让写IO直接走磁盘，但是更换电容很麻烦，而且**电容比较容易老化，其实一般不常用**
+ **手动充放电**，这个比较常用，包括一些大家知道的顶尖互联网大厂的数据库服务器的RAID就是**用了这个方案避免性能抖动，就是关闭RAID自动充放电，然后写一个脚本，脚本每隔一段时间自动在晚上凌晨的业务低峰时期**，脚本手动触发充放电，这样可以避免业务高峰期的时候RAID自动充放电引起性能抖动
+ 充放电的时候不要关闭write back，就是设置一下，**锂电池充放电的时候不要把缓存级别从write back修改为write through**，这个也是可以做到的，可以和第二个策略配合起来使用



## 9. 案例分析：数据库无法连接Too many connections

据库部署在64GB的大内存物理机上，机器配置各方面都很高，然后连接这台物理机的Java系统部署在2台机器上，Java系统设置的连接池的最大大小是200，也就是说每台机器上部署的Java系统，最多跟MySQL数据库建立200个连接，一共最多建立400个连接

![](./images/mysql-40.png)

MySQL报异常说Too many Connections，就说明目前MySQL甚至都无法建立400个网络连接。于是我们检查了一下MySQL的配置文件，my.cnf，里面有一个关键的参数是**max_connections**，就是MySQL能建立的最大连接数，设置的是**800**。

> Could not increase number of max_open_files to more than mysqld (request: 65535)
> Changed limits: max_connections: 214 (requested 2000)
> Changed limits: table_open_cache: 400 (requested 4096)



MySQL发现自己无法设置max_connections为我们期望的800，只能强行限制为214了！这是为什么呢？简单来说，就是因为底层的linux操作系统把进程可以打开的**文件句柄数限制为了1024了，导致MySQL最大连接数是214**！

```bash
ulimit -HSn 65535
cat /etc/security/limits.conf
cat /etc/rc.local
```

因为如果linux限制你一个进程的文件句柄太少的话，那么就会导致我们没办法创建大量的网络连接，此时我们的系统进程就没法正常工作了

举个例子，比如MySQL运行的时候，其实就是linux上的一个进程，那么他其实是需要跟很多业务系统建立大量的连接的，结果你限制了他的最大文件句柄数量，那么他就不能建立太多连接了！

所以说，往往你在生产环境部署了一个系统，比如**数据库系统、消息中间件系统、存储系统、缓存系统之后，都需要调整一下linux的一些内核参数**，这个文件句柄的数量是一定要调整的，**通常都得设置为65535**

所以我们平时可以用ulimit命令来设置每个进程被限制使用的资源量，用**ulimit -a**就可以看到进程被限制使用的各种资源的量



## 10. Redo 日志

### 10.1 Redo日志机制

更新完Buffer Pool中的缓存页之后，必须要写一条redo log，这样才能记录下来我们对数据库做的修改。

redo log可以保证我们事务提交之后，如果事务中的增删改SQL语句更新的缓存页还没刷到磁盘上去，此时MySQL宕机了，那么MySQL重启过后，就可以把redo log重做一遍，恢复出来事务当时更新的缓存页，然后再把缓存页刷到磁盘就可以了。redo log本质是保证事务提交之后，修改的数据绝对不会丢失的。

![](./images/mysql-41.png)

redo log机制，这个机制就是说，你提交事务的时候，绝对是保证把你对缓存页做的修改以日志的形式，写入到redo log日志文件里去的。这种日志大致的格式如下：对表空间XX中的数据页XX中的偏移量为XXXX的地方更新了数据XXX。只要你事务提交的时候保证你做的修改以日志形式写入redo log日志，那么哪怕你此时突然宕机了，也没关系！

因为你MySQL重启之后，把你之前事务更新过做的修改根据redo log在Buffer Pool里重做一遍就可以了，就可以恢复出来当时你事务对缓存页做的修改，然后找时机再把缓存页刷入磁盘文件里去。

实际上，**如果你把修改过的缓存页都刷入磁盘，这首先缓存页一个就是16kb，数据比较大**，刷入磁盘比较耗时，而且你可能就修改了缓存页里的几个字节的数据，难道也把完整的缓存页刷入磁盘吗。而且**你缓存页刷入磁盘是随机写磁盘**，性能是很差的



### 10.2 Redo日志格式

redo log就划分为了不同的类型

+ MLOG_1BYTE类型的日志指的就是修改了1个字节的值
+ MLOG_2BYTE类型的日志指的就是修改了2个字节的值，以此类推，还有修改了4个字节的值的日志类型，修改了8个字节的值的日志类型。
+ 修改了一大串的值，类型就是**MLOG_WRITE_STRING**，就是代表你一下子在那个数据页的某个偏移量的位置插入或者修改了一大串的值。

```
日志类型（就是类似MLOG_1BYTE之类的），表空间ID，数据页号，数据页中的偏移量，具体修改的数据
```



### 10.3 Redo Block

MySQL内有另外一个数据结构，叫做**redo log block**，大概你可以理解为，平时我们的数据不是存放在数据页了的么，用一页一页的数据页来存放数据。那么对于redo log也不是单行单行的写入日志文件的，他是用一个redo log block来存放多个单行日志的。

12字节的header头又分为了4个部分。

+ 4个字节的**block no**，就是块唯一编号；
+ 2个字节的**data length**，就是block里写入了多少字节数据；
+ 2个字节的**first record group**。这个是说每个事务都会有多个redo log，是一个redo log group，即一组redo log。那么在这个block里的**第一组redo log的偏移量**，就是这2个字节存储的；
+ 4个字节的checkpoint on

![](./images/mysql-43.png)

每一个redo log都是写入到文件里的一个redo log block里去的，**一个block最多放496自己的redo log日志**。

![](./images/mysql-42.png)

一个redo log block就是512字节，那么真正写入的时候，把这个redo log block的512字节的数据，就写入到redo log文件里去就可以了。那么redo log文件里就多了一个block。**日志文件里是存放一个又一个的redo log block的**



### 10.4 Redo Log Buffer

![](./images/mysql-44.png)



这个redo log到底是如何通过内存缓冲之后，再进入磁盘文件里去的，这就涉及到了一个新的组件，**redo log buffer，他就是MySQL专门设计了用来缓冲redo log写入的。**

这个redo log buffer其实就是MySQL在启动的时候，就跟操作系统申请的一块**连续内存空间**.

通过设置mysql的**innodb_log_buffer_size**可以指定这个redo log buffer的大小，**默认的值就是16MB**，其实已经够大了，毕竟一个redo log block才512自己而已，每一条redo log其实也就几个字节到几十个字节罢了。

写满了一个redo log block，就会继续写下一个redo log block，以此类推，直到所有的redo log block都写满。**万一要是redo log buffer里所有的redo log block都写满了呢。那此时必然会强制把redo log block刷入到磁盘中去的**！



其实在我们平时执行一个事务的过程中，每个事务会有多个增删改操作，那么就会有多个redo log，这多个redo log就是一组redo log，其实每次一组redo log都是先在别的地方暂存，然后都执行完了，再把一组redo log给写入到redo log buffer的block里去的

**如果一组redo log实在是太多了，那么就可能会存放在两个redo log block中**

![](./images/mysql-45.png)

**如果说一个redo log group比较小，那么也可能多个redo log group是在一个redo log block里的**

![](./images/mysql-46.png)



![](./images/mysql-47.png)

**Redo log block是哪些时候会刷入到磁盘文件里去：**

+ 如果写入**redo log buffer的日志已经占据了redo log buffer总容量的一半了**，也就是超过了8MB的redo log在缓冲里了，此时就会把他们刷入到磁盘文件里去
+ **一个事务提交的时候**，必须把他的那些redo log所在的redo log block都刷入到磁盘文件里去，只有这样，当事务提交之后，他修改的数据绝对不会丢失，因为redo log里有重做日志，随时可以恢复事务做的修改
+ **后台线程定时刷新**，有一个后台线程每隔1秒就会把redo log buffer里的redo log block刷到磁盘文件里去
+ **MySQL关闭的时候**，redo log block都会刷入到磁盘里去



当然，绝对保证数据不丢，还得配置一个参数，提交事务把redo log刷入磁盘文件的os cache之后，还得强行从os cache刷入物理磁盘。redo log都会写入一个目录中的文件里，这个目录可以通过**show variables like 'datadir'**来查看，可以通过**innodb_log_group_home_dir**参数来设置这个目录的。

redo log是有多个的，写满了一个就会写下一个redo log，而且可以限制redo log文件的数量，通过**innodb_log_file_size**可以指定每个redo log文件的大小，默认是**48MB**，通过**innodb_log_files_in_group**可以指定**日志文件的数量**，默认就2个。分别为ib_logfile0和ib_logfile1，每个48MB。如果第二个也写满了呢，继续写第一个，覆盖第一个日志文件里原来的redo log就可以了。


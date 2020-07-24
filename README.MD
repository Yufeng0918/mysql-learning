

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

redo日志是为了防止机器宕机，缓存数据数据没有刷入磁盘。redo日志用于恢复数据。

![](./images/mysql-04.png)

**Innodb_flush_log_at_trx_commit=0**，**提交事务的时候不会redo log 刷入磁盘**。如果提交事务成功而mysql宕机，内存中的数据和redo日志都会丢失



![](./images/mysql-05.png)

**Innodb_flush_log_at_trx_commit=1**，**提交事务的时候必须把redo log 刷入磁盘**。如果提交事务成功而mysql宕机，内存中的数据丢失。mysql重启以后可以通过redo日志来恢复数据



![](./images/mysql-06.png)

**Innodb_flush_log_at_trx_commit=2**，**提交事务的时候必须把redo log 刷os cache**。如果提交事务成功而mysql宕机，内存中的数据丢失。如果os cache没有把数据刷入磁盘，os cache 和 redo日志都会丢失。
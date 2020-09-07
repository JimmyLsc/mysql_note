# 应用优化

通过对前台应用进行一些优化降低数据库的访问压力：

- （连接）建立数据库连接池：访问数据库中，开启关闭连接十分消耗资源。我们需要建立数据库连接池来维持某些暂时不使用的连接。
- （数据）减少对MYSQL的访问：
  1. 避免对数据进行重复检索
  2. 增加cache层（Mybatis，Hibernate）
- （服务器）负载均衡：
  1. 利用mysql复制分流查询（读写分离）
  2. 利用分布式数据库



# Mysql中查询缓存优化

## 缓存查询流程

开启Mysql的查询缓存，当执行完全相同SQL语句时，服务器会直接从缓存中读取数据，当数据被修改，之前的缓存会失效，修改比较频繁的表不适合做查询缓存

![img](https://user-gold-cdn.xitu.io/2019/2/22/16910d7a35f569f9?imageslim)

1. 客户端发送一条查询给服务器
2. 服务器会检查查询缓存，如果命中缓存，则立即返回存储在缓存中的结果，否则进入下一阶段；
3. 服务器进行SQL解析，预处理，再有优化器生成对应的执行计划



## 查询缓存参数配置

查看是否支持查询缓存

```mysql
SHOW VARIABLES LIKE 'have_query_cache'
```

查看是否开启了查询缓存

```mysql
SHOW VARIABLES LIKE 'query_cache_type'
```

查看查询缓存占用大小

```mysql
SHOW VARIABLES like 'query_cache_size' -- 单位byte
```

查看查询缓存状态

```mysql
SHOW STATUS LIKE 'Qcache%'
```

| 参数                    | 含义                                                         |
| :---------------------- | ------------------------------------------------------------ |
| Qcache_free_blocks      | 查询缓存中的可用内存块数                                     |
| Qcache_free_memory      | 查询缓存中的可用内存量                                       |
| Qcache_hits             | 查询缓存命中数                                               |
| Qcache_inserts          | 添加到查询缓存中的查询数                                     |
| Qcache_lowmen_prunes    | 由于内存不足而从查询缓存中删除的查询数                       |
| Qcache_not_cached       | 非缓存查询的数量（由于query_cache_type设置而无法缓存或未缓存） |
| Qcache_queries_in_cache | 查询缓存中注册的查询数                                       |
| Qcache_total_blocks     | 查询缓存中的块总数                                           |

开启查询缓存

在/usr/my.conf下修改

```mysql
query_cache_type=1 -- 关闭：OFF或0， 打开：ON或1：指定SQL_NO_CACHE则不缓存， 手动：DEMAND或2:指定SQL_CACHE才会缓存
```

查询缓存的SELECT选项：通过指定选项来显式决定是否缓存或不缓存

```mysql
SELECT SQL_CACHE <select_columns> FORM <table>
SELECT SQL_NO_CACHE <select_columns> FROM <table>
```

查询缓存失效情况

1. SQL语句不一致（包括大小写，因为解析器在缓存区之后）
2. 使用不确定函数时不缓存（例如：now(), current_date(), curdate(), curtime(), rand(), uuid(), database() ）

3. 不使用任何表查询语句（SELECT常量）
4. 查询系统数据库时不走查询缓存
5. 在存储的函数，触发器或时间主体内执行的查询
6. 查询表发生变化



# Mysql内存管理与优化

## 优化原则

1. 将尽量多的内存分配给Mysql做缓存，但要给操作系统和其他程序预留足够多的空间
2. MyISAM存储引擎的数据文件读取依赖于操作系统自身的IO缓存，因此，如果有MyISAM表，就要预留更多的内存给操作系统做IO缓存
3. 排序区，连接区的缓存时分配给每个数据库会话使用的，其默认值的设置根据最大连接数合理分撇，如果设置过大，不但浪费资源，且在并发连接时导致物理内存消耗

## MyISAM内存优化

MyISAM存储引擎使用key_buffer缓存索引块，加速MyISAM索引的读写速度，对于MyISAM表的数据块，Mysql没有特别的缓存机制，完全依赖IO缓存



key_buffer_size

索引块缓存区大小：一般至少分配总内存的1/4

在usr/my.conf 中配置



read_buffer_size

全表扫描时的缓存大小，由每个会话独占



read_rnd_buffer_size

排序操作中的缓存大小，由每个会话独占



## InnoDB内存优化

innodb用一块内存区做IO缓存池，该缓存池不仅缓存Innodb的索引块，而且也用来缓存innodb的数据块



innodb_buffer_pool_size

该变量决定InnoDB缓存的索引与数据总大小

该变量越高，缓存命中率越高，访问InnoDB表需要的磁盘I/O越少，性能也就越高



*innodb_log_buffer_size

*决定innodb重做日志缓存大小



# Mysql 并发参数调整

从实现来说。Mysql server是多线程的，包括后台线程和客户服务线程，多线程可以有效利用服务器资源，提高数据库的并发性能。再Mysql中，控制并发连接和线程的主要参数包括max_connections, back_log, thread_cache_size,table_open_cache

## max_connections

采用max_connections 控制允许链接到Mysql数据库的最大数量，默认值为151， 如果状态变量connection_errors_max_connections 不为0， 并且一直增长，说明不断有新连接导致失败，所以可以考虑增大max_connections的值

Mysql最大可支持的连接数，取决于很多因素，包括给定操作平台的线程库的质量，内存大小，每个连接负荷，CPU的处理速度，期望的响应时间等。

## back_log

back_log参数控制Mysql监听TCP端口时挤压请求栈大小。如果Mysql的连接数达到max_connections时，新来的请求会存在堆栈中。

back_log 的默认值为 50+(max_connections/5)



## table_open_cache

该参数用来控制所有SQL语句可打开的表缓存数量

表缓存数量：所有SQL操作的表数量

一般设置为max_connections * M



## thread_cache_size

为了加快连接数据库的速度，Mysql会缓存一定数量的服务线程作为备用，通过参数thread_cache_size可控制Mysql缓存客户服务线程的数量。（线程池大小）



## innodb_lock_wait_timeout

该参数是用来设置InnoDB事务等待行锁的时间，默认值为50ms，可以根据需要进行动态设置。对于需要快速反馈的业务系统来说，可以将行锁的等待时间减小，以避免事务长时间挂起；对于后台运行的批量处理程序来说，可以将行锁的等待时间调大，以避免发生大的回滚操作。



# Mysql锁问题

锁是计算机协调多个进程或线程并发访问某一资源的一种机制

在数据库中，除传统的计算资源（如CPU，RAM，I/O等）的争用以外，数据也是一种供许多用户共享的资源。如何保证数据并发访问的一致性，有效性是所有数据库必须解决的一个问题，锁冲突也是影响数据库并发访问性能的一个重要因素。从这个角度来说，锁对数据库而言尤其重要，也更加复杂。

## 锁分类

操作粒度来分：

1. 表锁：操作时锁定全表
2. 行锁：操作时锁定单行

操作数据类型来分：

1. 读锁：针对同一份数据，多个读操作可以同时进行而不会互相影响
2. 写锁：当前操作没有完成之前，他会阻断其他写锁和读锁



## 支持引擎

| 存储引擎 | 表级锁       | 行级锁       | 页面锁 |
| -------- | ------------ | ------------ | ------ |
| MyISAM   | 支持（默认） | 不支持       | 不支持 |
| InnoDB   | 支持         | 支持（默认） | 不支持 |
| MEMORY   | 支持（默认） | 不支持       | 不支持 |
| BDB      | 支持         | 不支持       | 支持   |



| 锁类型 | 特点                                                         |
| ------ | ------------------------------------------------------------ |
| 表级锁 | 偏向MyISAM引擎，开销小，加锁快；不会出现死锁；锁定粒度大，发生锁冲突的概率最高，并发度最低 |
| 行级锁 | 偏向InnoDB引擎，开销大，加锁慢；会出现死锁；锁粒度最小，发生锁冲突的概率最低，并发度也最高 |
| 页面锁 | 开销和加锁时间于表锁与行锁之间；会出现死锁；锁粒度介于表锁和行锁之间，并发度一般 |

表级锁更加适合一查询为主，只有少量按索引条件更新数据的应用，如WEB应用

行级锁更适合大量按索引条件并发更新少量不同数据，同时又有并发查询的应用，如一些在线事务处理



## MyISAM 锁

MyISAM存储引擎只支持表锁

MyISAM会自动再增删改查中给表加表锁

用户一般不需要直接用LOCK TABLE给MyISAM表显示加锁

```mysql
读锁：LOCK TABLE <table_name> READ
写锁：LOCK TABLE <table_name> WRITE
```

释放锁

```mysql
UNLOCK TABLES
```

 

| **当前锁模式\请求锁模式** | **None** | **读锁** | **写锁** |
| :------------------------ | -------- | -------- | -------- |
| **读锁**                  | 是       | 是       | 否       |
| **写锁**                  | 是       | 否       | 否       |



1. 对于MyISAM表的读操作，不会阻塞其他用户对同一表的读请求但会阻塞对同一表的写请求
2. 对MyISAM表的写操作，则会阻塞其他用户对同一表的读和写操作

MyISAM读写锁调度机制是写优先，所以MyISAM不适合作为以写为主的表存储引擎



查看锁的争用情况

```mysql
SHOW OPEN TABLES
```

返回值中 in_use为0则无锁，1则有锁

```mysql
SHOW STATUS LIKE 'Table_locks%'
```

返回值中Table_locks_immediate：指的是能够立即获得表级锁的次数，每次获得表级锁，值加一

Table_locks_waited：指的是不能理解获得表级锁而需要等待的次数，每等待一次，值加一，值高说明较为严重的表级锁争用情况



## InnoDB行锁

偏向InnoDB存储引擎，开销大，加锁慢；锁定粒度最小，发生锁冲突的概率比较低，并发度最高，InnoDB与MyISAM的最大不同有两点：一是支持事务；二是采用的行级锁



### 事务

事务是由一组SQL语句组成的逻辑处理单元

事务具有一下ACID四个特性

| ACID属性              | 含义                                                         |
| --------------------- | ------------------------------------------------------------ |
| 原子性（Atomicity）   | 事务是一个原子操作单元，其对数据的修改，要么全部成功，要么全部失败 |
| 一致性（Consistency） | 在事务开始和完成时，数据都必须保持一致状态                   |
| 隔离性（Isolation）   | 数据库系统提供一定的隔离机制，保证事务再不受外部并发操作影响的独立环境下运行 |
| 持久性（duration）    | 事务完成之后，对于数据的修改也是永久的                       |



### 并发事务处理带来的问题

| 问题                                | 含义                                                         |
| ----------------------------------- | ------------------------------------------------------------ |
| 丢失更新（Lost Update）             | 当两个或多个事务选择同一行，最初的事务修改的值，会被后面的事务修改值覆盖 |
| 脏读（Dirty Reads）                 | 当一个事务读取到了另一个事务还未提交的数据                   |
| 不可重复读（Non-Repeateable Reads） | 一个事务再读取某些数据后的某个时间，再次读取以前读过的数据，却发现和以前读到的不一致（同一个事务内部） |
| 幻读（Phantom Reads）               | 一个事务按照相同的查询条件重新读取以前查询过的数据，却发现其他事务插入满足其查询条件的新数据（同一个事务内部） |

### 事务隔离级别

为了解决上述提到的事务并发问题，数据库采用一定事务隔离机制来解决这个问题。数据据库的事务隔离越严格，并发副作用越小，但付出的代价也就越大，因为事务隔离实质上就是使用事务在一定程度上“串行化”进行，这显然与“并发”是矛盾的。

数据库的隔离级别有4个，由低到高依次Read uncommitted， Read committed， Repeatable Read， Serializable，这四个可以解决脏写，脏读，不可重复读，幻读这几类问题

| 隔离级别(F为不会出现，T为会出现） | 丢失更新 | 脏读 | 不可重复读 | 幻读 |
| --------------------------------- | -------- | ---- | ---------- | ---- |
| Read uncommitted                  | F        | T    | T          | T    |
| Read committed                    | F        | F    | T          | T    |
| Repeatable read（默认）           | F        | F    | F          | T    |
| Serializable                      | F        | F    | F          | F    |

Mysql默认使用Repeatable read查看方式为：

```mysql
SHOW VARIABLES LIKE 'tx_isolation'
```



### InnoDB的行锁模式

InnoDB实现了两种类型行锁：

- 共享锁（S）：又称为读锁，简称S锁，共享锁就是多个事务对于同一个数据可以共享一把锁，都能访问到数据，但是只能读不能更改
- 排他锁（X）：又称为写锁，简称X锁，排他锁就是不能与其他所并存，如果一个事务获取了一个数据行的排他锁，其他事务就不能在获得该行的其他所，包括共享锁和排他锁，但是获取排他锁的事务是可以对数据就行读取和修改。

对于UPDATE, DELETE, INSERT 语句，InnoDB会自动给涉及数据集添加排他锁（X）

对于普通SELECT语句，InnoDB不会添加任何锁



可以通过以下语句显示给记录集添加共享锁或者排他锁

```mysql
S：SELECT * FROM <table_name> WHERE <line_identification> LOCK IN SHARE MODE

X: SELECT * FROM <table_name> WHERE <line_identification> FOR UPDATE
```



### 锁自动升级机制

如果不通过索引检索数据，那么InnoDB将表中所有记录加锁，实际效果与表锁一致



## 间隙锁

当我们使用范围条件查询时，当使用共享锁或者排他锁是，InnoDB会自动给间值在范围内但是不存在的索引值添加间隙锁



### 行锁争用情况

查看行锁争用情况的指令

```mysql
SHOW STATUS LIKE 'innodb_row_lock%'
```

 

### 优化建议

- 尽可能让所有数据检索能通过索引来完成，避免无索引行锁升级为表锁
- 合理设计索引，尽量缩小锁的范围
- 尽可能减少查询条件，查询范围，避免间隙锁
- 尽量控制事务的大小，减少锁定资源量和时间长度
- 尽可能降低事务隔离级别（满足业务需求前提下）



## 执行顺序

SQL编写顺序

```mysql
SELECT DISTINCT
	<select list>
FROM 
	<left table><join type>
JOIN 
	<right table> ON <join_condiion>
WHERE 
	<where condition>
GROUP BY
	<group by list>
HAVING 
	<having condition>
ORDER BY
	<order by confition>
LIMIT 
	<limit params>
```

执行顺序

```mysql
FROM <left table>

ON <join condition>

<join type> JOIN <right table>

WHERE <where_condition>

GROUP BY <group by list>

HAVING <having condition>

SELECT DISTINCT <select list>

ORDER BY <order by condition>

LIMIT <limit params>
```


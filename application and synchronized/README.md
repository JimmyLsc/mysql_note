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




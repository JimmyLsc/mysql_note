# 体系结构

![img](https://img-blog.csdn.net/20150730152113099?watermark/2/text/aHR0cDovL2Jsb2cuY3Nkbi5uZXQv/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70/gravity/Center)

连接层：Connection Pool接收客户端请求开启线程提供服务

服务层：Management Services and Utilities, SQL Interface, Parser, Optimizer, Cashes and Buffers

存储引擎层：Pluggable Storage Engines(插件式存储引擎)

存储层：File System, File and Logs(日志，错误信息，二进制信息，索引信息)



整个MYSQL SERVER 由以下组成

- Connection Pool：连接池组件
- Mangement Services & Utilities：管理服务和工具组件
- SQL Interface：SQL接口组件
- Parser：查询分析器组件
- Optimizer：优化器组件
- Caches & Buffers：缓冲池组件
- Pluggable Storage Engines：存储引擎
- File System：文件系统



## 连阶层

最上层是一些客户端和连接服务，包含本地sock通信和大多数给予客户端/服务端工具实现的类似TCP/IP的通信，主要完成一些类似于连接处理，授权认证，及相关的安全方案，在该层上引入了线程池的概念，为通过认证安全接入的客户端提供线程。同样在该层上可以实现基于SSL的安全链接。

## 服务层

第二层架构主要完成大多数核心服务功能，如SQL接口，并完成缓存的查询，SQL分析与优化，部分内置函数的执行。所有跨存储引擎功能也在这一层实现，如过程，函数等。在该层，服务器会解析查询并创建相应的内部解析树，并对其完成相应的优化如确定表的查询的顺序，是否利用索引等，最后生成相应的执行操作。如果是SELECT语句，服务器还会查询内部的缓存，如果缓存空间足够大，这样的在解决大量读操作的环境中能够很好的提升系统的性能。

## 引擎层*

存储引擎层，存储引擎真正的负责了MYSQL中数据的存储与提取，服务器通过API和存储引擎进行通信。不同的存储引擎具有不同的功能，这样我们可以根据自己的需要，来选去合适的存储引擎

## 存储层

数据存储层，主要是讲数据存储在文件系统之上，并完成存储引擎的交互



## * 

MYSQL与其他SQL最大的区别就是引擎层的区别，就是它的架构可以在多种不同场景中应用并发挥良好的作用。主要体现在存储引擎上，插件式的存储引擎架构，<u>将查询处理和其他的</u><u>系统任务以及数据的存储提取分离</u>。这种架构可以根据业务的需求和实际需要选择合适的存储引擎。

# 存储引擎

Mysql可以根据不同的需求选择最优的存储引擎

存储引擎是存储数据，建立索引，更新查询数据等等技术的实现方式。存储引擎是基于表的，而不是基于库的。所以存储引擎也可被称为表类型。

Oracle，SqlServer等数据库只有一种存储引擎，Mysql提供了插件式的存储引擎架构。所以Mysql存在多种存储引擎，可以根据需要使用相应引擎，或者编写存储引擎。



查看当前数据库支持引擎

```mysql
SHOW ENGINES
```

## InnoDB

支持事务

```mysql
START TRANSACTION
<sql statement>
COMMIT
```

支持外键

```mysql
CREATE TABLE people(
	id INT NOT NULL AUTO_INCREMENT,
	name VARCHAR(50) NOT NULL,
	PRIMARY KEY (id)
	)ENGINE=innodb AUTO_INCREMENT=1 DEFAULT CHARSET=utf8;
CREATE TABLE grade(
	id INT NOT NULL AUTO_INCREMENT,
	student_id INT NOT NULL,
	PRIMARY KEY (id),
	FOREIGN KEY (student_id) REFERENCES people(id) ON DELETE RESTRICT ON UPDATE CASCADE
  -- 自动生成外键
  [CONSTRAINT 'fk_grade_people' FOREIGN KEY (student_id) REFERENCES people(id) ON DELETE RESTRICT ON UPDATE CASCADE] -- 命名式定义外键
	)ENGINE=innodb AUTO_INCREMENT=1 DEFAULT CHARSET=utf8
```

ON DELETE RESTRICT： 删除主表数据时，如果子表有关联记录，则无法删除

ON UPDATE CASCADE：更新主表数据时，如果子表有关联记录，则更新子表记录



存储方式

InnoDB存储表和索引有以下两种方式：

1. 使用共享空间存储，这种方式创建的表的表结构保存在.frm文件中，数据和索引保存在innodb_data_home_dir和innodb_data_file_path定义的表空间中，可以是多个文件
2. 使用多表空间存储，这种方式创建的表结构依然存在.frm文件中，但是每个表的数据和索引单独保存在.ibd中



## MyISAM

不支持事务，不支持外键

访问速度快，适用于对事务完整性不作要求，或者对容忍小部分文件丢失的情况



存储方式：

.frm：定义表结构

.MYD：存储表数据

.MYI：存储表索引



## MEMORY 

Memory 存储引擎将表的数据存放在内存中。每个MEMORY表实际对应一个磁盘文件，格式为.frm，该文件只存储表的结构，而数据存储在内存中，这样有利于数据的快速处理，提高表的效率。MEMORY类型的表访问非常快，因为它的数据是存放在内存中的，并且默认使用HASH索引，一旦服务关闭，表中数据丢失。



## MERGE

Merge存储引擎是一组MyISAM表的组合，<u>这些MyISAM表结构必须完全相同</u>，<u>Merge表本身没有存储数据</u>，对Merge类型的表可以进行查询，更新，删除操作，这些操作实际上是对内部的MyISAM进行的。

对Merge类型的表的插入操作，Merge是通过INSERT_METHOD字句定义插入的表，可以有3个不同的值，使用FIRST或LAST值来使得插入操作被相应的作用在第一或者最后一个表上，不定义这个子句或者定义为NO，表示不能对这个MERGE执行插入操作

可以对MERGE表进行DROP操作，但是这个操作知识删除MERGE表的定义，对内部表没有任何影响

```mysql
CREATE TABLE <merge_table_name>(
	<structure>
)ENGINE=merge UNION=([<table1_name>,<table2_name>...]) INSERT_METHOD=FIRST|LAST DEFAULT CHARSET=utf8
```



## 存储引擎选择

在绝大部分场景中，都应选择InnoDB

- InnoDB：是Mysql的默认存储引擎，用于事务处理应用程序，支持外键。如果应用对事务的<u>完整性</u>有比较高的要求，在并发条件下要求<u>数据的一致性</u>，数据除了在插入和查询外，还包含很多更新，删除操作，那么InnoDB是比较合适的选择。InnoDB存储引擎除了有效的降低由于删除和更新导致的锁定，还可以<u>保证事务的完整提交与回滚</u>，对于类似计费系统或者财务系统等对数据准确性要求比较高的系统，InnoDB是最合适的选择。
- MyISAM：如果应用是以读和插入操作为主，只有很少的更新和删除操作，并且对事务的完整性，并发性要求不是很高，那么这个存储引擎是非常适合的。
- MEMORY：<u>将所有数据保存在RAM中</u>，在需要快速定位记录和其他类似的数据环境下，可以提供极快的访问。MEMORY的缺陷就是对表的大小有限制，太大的表无法缓存在内存中，其次是要确保表的数据可以恢复，数据库异常终止后表中的数据是可以恢复的。MEMORY表通常用于更新不太频繁的小表，用以快速得到访问结果。
- MERGE：用于将一系列等同的MyISAM表以逻辑方式组合在一起，并作为一个对象引用他们。MERGE表的有点在于可以突破对单个MyISAM表的大小限制，并且通过将不同的表分布在多个磁盘上，可以有效的改善MERGE表的访问效率。这对于存储诸如数据仓库等VLDB环境十分合适。



# SQL问题检索

## 查询

查询数据库操作频次，判断数据库以什么操作为主

```mysql
SHOW [GLOBAL|SESSION] STATUS LIKE 'COM______' -- 数据库操作
```

```mysql
SHOW [GLOBAL|SESSION] STATUS LIKE 'Innodb_rows_%' -- 针对引擎查询
```



## 定位

主要通过两种方式定位低效率SQL语句

- 慢查询日志：通过慢查询日志定位那些执行效率较低的SQL语句，查询较长执行长度的SQL语句。查询的是以完成被记录的操作日志。
- SHOW PROCESSLIST：processlist查看sql语句实时执行情况。



## EXPLAIN查询分析（最常用的分析工具）

EXPLAIN 指令是描述查询计划的指令

```mysql
EXPLAIN <sql_select_statement>
```

得到的列：

- id：select查询的序列号（以表为单位），是一组数字，表示的是查询中执行select子句或者是操作表的顺序（<u>id值越大表示优先级越高</u>，id相同则从上到下）

- select_type：表示SELECT的类型，以下类型效率从高到低：

  - SIMPLE：简单的SELECT不包含子查询与UNION
  - PIRIMARY：查询中包含复杂的子查询，标记最外层为PRIMARY
  - SUBQUERY：在SELECT与<u>WHERE中的子查询</u>
  - DERIVED：在<u>FROM列表中包含的子查询</u>，MYSQL会递归的执行这些子查询，并加载临时表
  - UNION：若第二个SELECT在UNION之后，则标记为UNION；若UNION包含在FROM子句的子查询中，最外层SELECT会被标记为DERIVED
  - UNION RESULT：从UNION表获取结果的SELECT

- table：输出结果集的表

- type：表示表的连接类型，性能由好到差的连接类型为

  - NULL：不访问任何表，索引
  - system：只查询一行数据，一般出现在系统表查询中
  - const：通过索引一次就查到了，const用于比较primary key或者unique索引。因为只匹配一行数据，所以很快。如将primary key放入where的列表中，mysql就能将该查询转换为一个常量。const于将唯一索引的部分与常量进行比较
  - eq_ref：类似ref，区别在于使用的是唯一索引，使用主键的关联查询（包括UNION，JOIN，CROSS JOIN），关联查询出的记录只有一条，常见于主键或唯一索引扫描
  - ref：非唯一性索引扫描，返回匹配某个单独值的所有行，本质也是一种索引访问，返回所有匹配某个单独值的所有行（多个）
  - range：只检索给定返回的行，使用过一个索引来选择行。Where之后出现between，<，>, in 等操作
  - index：index和ALL的区别为index类型只遍历了索引树，通常比ALL快，ALL是遍历数据文件（index本质就是select了一个索引列）
  - ALL：遍历全表找到匹配的行

  一般来说达到ref或者range级别就可以满足条件需求

- possible_keys：表示查询时，可能使用的索引

- key：表示实际使用的索引

- key_len：索引字段的长度

- rows：扫描行的数量

- extra：执行情况的说明和描述



## Show profiles

查看是否支持profile

```mysql
SELECT @@have_profiling
```

开启show profiles功能

```mysql
SET profiling=1
```

查询所有操作耗时

```mysql
SHOW PROFILES
```

查询系统具体操作耗时

```mysql
SHOW PROFILE FOR QUERY <query_id>
```

## TRACE

TRACE 可以用来追踪优化器工作

打开TRACE，设置格式为JSON，并设置TRACE最大能够使用的内存大小，避免解析过程中因为默认内存过小而不能完整展示

```mysql
SET optimizer_trace="enabled=on", end_markers_in_json=on
SET optimizer_trace_max_size=1000000
```

通过查询系统库的优化日志表来获取trace信息

```mysql
SELECT * FROM information_schema.optimizer_trace\G
```

# 索引的使用注意事项

索引是数据库优化最常用且最高效的手段

1. 全值匹配，对索引中所有列表指定具体值。该情况下，索引生效，执行效率高
2. 最左前缀索引，如果索引了多列，要遵守最左前缀法则，指的是查询从索引的最左前列开始，且不跳过索引的列
3. 范围查询右边的列，不能使用索引（复合索引中范围查询后面字段的索引失效）
4. 不要在索引列上进行运算操作，否则索引失效
5. 字符串不加单引号，索引失效
6. 覆盖索引，查询值被索引覆盖时能提高查询效率（避免获得索引后的回表查询）
7. or值中如果包含非非索引内容，之前的索引也会失效
8. like模糊搜索时，查询全值时如果%在开头则索引会失效，只查询覆盖索引内容是%放在开头或结尾索引都能起效
9. 如果Mysql评估索引搜索比全表扫描慢，则mysql不使用索引（多发生在某种条件占几大部分情况）
10. IS NULL 与 IS NOT NULL有时索引失败（本质是因为第9条原因）
11. IN会使用索引，NOT IN不会使用索引
12. 实际开发当中尽量选择复合索引，原因：
    - 创建复合索引时只创建一个索引表可以有多个索引列
    - Mysql会使用最优的索引方式，用单列索引有时会产生索引冗余（与索引‘辨识度’有关）



查看索引使用情况(仅作为参考)

```mysql
SHOW STATUS [GLOBLE|SESSION] LIKE 'Handler_read%';
```

# SQL优化

##  大批量插入数据

使用load导入数据时，适当的设置可以提高导入的效率

load基本语法

```mysql
LOAD DATA LOCAL INFILE '<file_path>' INTO TABLE '<table_name>' FIELDS TERMINATED BY '<file_terminated>' LINES TERMINATED BY '<line_terminated>'
```

对于InnoDB类型的表，有以下方式提高效率

1. 主键顺序插入
2. 关闭唯一性校验（SET UNIQUE_CHECKS=1）

3. 手动提交事务（SET AUTOCOMMIT=0）



## Insert优化

1. Insert多行数据时，尽量使用多个值表的insert语句，可以减少客户端与数据库的连接，关闭次数

   ```mysql
   INSERT INTO <table_name> VALUES(<values1>),(<values2>),(<values3>),...
   ```

2. 在事务中进行数据插入

   ```mysql
   START TRANSACTION
   <sql_statement>
   COMMIT
   ```

3. 数据有序插入



## 排序ORDER BY优化

排序方式分为两种：

1. 第一种方式通过对返回数据进行排序（FILESORT），所有不是通过索引直接返回排序结果的排序都叫FILESORT
2. 第二种通过有序索引顺序扫描直接返回有序数据，这种为using index，不需要额外排序，操作效率高

### using index

使用覆盖索引能使用using index

注意：多字段排序 ：

- 要么全部升序，要么全部降序，否则会使用filesort
- order顺序需要与索引顺序保持一致

### filesort

mysql有两种排序算法：

1. 两次扫描算法：根据排序字段与行指针在sort buffer排序，之后再根据行指针在表中返回排序结果。对内存要求不高，但是大量随机I/O造成大量开销
2. 一次扫描算法：一次性取出所有满足条件的字段，之后直接在sort buffer输出结果集。内存开销大，但效率高



Mysql通过比较系统变量max_length_for_sort_data的大小和Query语句去除的字段总大小，来判定哪种排序算法，如果max_length_for_sort_data更大，那么使用第二种优化之后的算法，否则使用第一种

优化时可以适当提高max_length_for_sort_data与sort_buffer_size的大小来提高排序效率（促使系统使用一次扫描算法）



## GROUP BY优化

- Group by会在group的时候默认排序，可以通过阻止排序提高分组效率

```mysql
SELECT count(<count_column>) FROM <table_name> GROUP BY <column> -- 优化前

SELECT count(<count_column>) FROM <table_name> GROUP BY <column> ORDER BY null -- 优化后
```

- 创建对group by的索引也能提高效率



## 嵌套查询

使用JOIN多表连接查询代替子查询可以提高mysql效率（详情可参考SQL问题检索）



## OR优化

对于包含or的索引，必须保证or各个字段都必须要有索引且不能使用复合索引，否则后续索引失效。（原因是复合索引中不包含后面列的单列索引）

建议通过union替换or来达到优化目的



## 分页（LIMIT）操作优化

LIMIT语法

```mysql
SELECT * FROM <table_name> LIMIT 10 -- 前10条
SELECT * FROM <table_name> LIMIT 10,10 -- 11条到20条
SELECT * FROM <table_name> LIMIT 20,10 -- 21条到30条
...
```

在LIMIT语法中，对于LIMIT 20000,10这个语句，查询的是20010条目进行排序，然后丢弃20000个条目，效率极低



### 优化思路

- 使用索引（主键）先进行排序取出id，然后关联得到的id表与原表进行查询

  ```mysql
  SELECT * FROM <table_name> t, (SELECT id FROM <table_name> LIMIT 20000,10) a WHERE t.id = a.id
  ```

  

- 把LIMIT查询转换为某个位置的查询，仅适用于主键自增的表

  ```mysql
  SELECT * FROM <table_name> WHERE id > 20000 LIMIT 10
  ```

  （原因：id走索引查询）

  

## 索引提示

sql 提示是给数据参考索引：

- 使用USE INDEX <index_name>提示数据库使用该索引
- 使用IGNORE INDEX <index_name> 避免数据库使用某些索引
- 使用FORCE INDEX <index_name> 强制数据库使用某索引
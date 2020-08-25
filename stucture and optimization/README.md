# 体系结构

![20150730152113099](/Users/jimmylsc/Desktop/20150730152113099.jpeg)

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



## EXPlAIN分析


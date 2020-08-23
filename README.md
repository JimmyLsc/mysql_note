# 索引

## **优势**

1 ）提高数据检索的效率，降低数据库的IO成本

2 ）通过对数据进行排序，降低排序成本，降低cpu消耗

## **劣势**

1 ）索引也是一张表，<u>保存了key 与索引字段</u>，并指向实体类

2 ）虽然索引提高查询效率，但是降低了更新效率



## **索引结构**

索引结构由存储引擎层提供，每种存储引擎提供与支持的索引类型不一样

mysql 提供了BTREE， HASH，R-TREE，Full-text

其中：

- InnoDB支持BTREE， FULL-text
- MyISAM支持BTREE索引，R-TREE索引，Full-Text索引
- Memory支持BTREE索引，HASH索引

其中：

- BTREE索引所有引擎都支持
- 只有Memory引擎支持HASH索引



## **BTREE**

BTREE多路查找树

一个m叉树的特性如下：

- 树中每个节点最多包含m个孩子
- 除根节点与叶节点外， 每个节点至少有[cell(m/2)]个孩子
- 若根节点不是叶子节点，则至少有两个孩子（一个key，两个分支）
- 所有的叶节点都在同一层
- 每个非叶子节点有n个key(值)与n+1个指针构成，其中[cell(m/2)-1] <= n <= m-1 

## B+TREE

B+TREE是BTREE的变种，B+TREE与BTREE的区别：

1. n叉B+TREE最多含有n个key， 而BTREE最多含有n-1个key
2. B+TREE的叶子节点保存所有的key信息，依key大小顺序排列
3. 所有的非叶子节点都可以看作key的索引部分（只起到查找的作用）

B+TREE的稳定性高于BTREE的稳定性

## MYSQL中的B+TREE

MYSQL索引数据结构对B+TREE进行了优化，在B+TREE的基础上，增加了一个指向相邻叶节点的链表指针，就形成了带有顺序指针的B+TREE，提高区间访问的性能



## 索引分类

1 ）单值（列）索引：只针对一个表的一个列

2 ）唯一索引：（针对某一列）索引列的值必须唯一，但允许有空值 在NON-UNIQUE索引字段中体现

3 ）复合索引：一个索引包含多个列

除此之外还有

1. 主键索引：表生成的时候会自动创建主键索引



## 索引代码

创建表的时候会自动生成主键索引

CREATE方法创建单列索引

```mysql
CREATE INDEX <index_name> ON <table_name>(<column_name>)
```

CREATE方法创建复合索引

```mysql
CREATE INDEX <index_name> ON <table_name>(<column_name>)
```

显示索引

```mysql
SHOW INDEX from <table_name>
SHOW INDEX from <table_name>/G
```

加上/G会显示更好观察的形式

删除索引

```mysql
DROP INDEX <index_name> ON <table_name>
```

ALTER方法添加单列索引

```mysql
ALTER TABLE <table_name> ADD INDEX <index_name> (<column_name>)
```

ALTER方法添加唯一索引

```mysql
ALTER TABLE <table_name> ADD UNIQUE <index_name>(<column_name>)
```

ALTER方法添加全文索引

```mysql
ALTER TABLE <table_name> ADD FULLTEXT <index_name>(<column_name>)
```



## 索引设计原则

- 对<u>查询频次较高</u>，且<u>数据量大</u>的表建立索引

- 索引字段的选择，最佳候选应当从where的字句的条件中选取（查询条件中的字段），如果where子句的组合比较多，那么应当选<u>最常用，过滤效果</u>最好的列的组合

- 尽量使用唯一索引，区分度更高，索引效率更高

- 索引量保持适当

- 尽可能使用短索引（短字段），减少磁盘使用空间，提高I/O效率

- （复合索引）利用最左索引：创建复合索引时，所有包含最左前缀的连续索引都会自动生成，利用最左索引可以减少索引量。

  例如：id，name，school创建复合索引

  生成id索引

  id，name索引

  id，name，school索引

  共三个索引

# 视图

视图是一条SELECT语句执行后的结果集，本质上是创建一个虚拟的表

## 视图的作用

简单：用户不用关心表的结构，关联，筛选条件

安全：实现数据库某列的可见性

数据独立：接触表结构变化对用户访问界面的影响

## 视图语句

创建视图

```mysql
CREATE VIEW <view_name> AS <SELECT SENTENCE>
```

查询视图

```mysql
SELECT * FROM <view_name>
```

修改视图

```mysql
UPDATE <view_name> SET <operation>
```

```mysql
WITH [CASCADED|LOCAL] CHECK OPTION
```

视图选项：

- LOCAL：满足本视图条件即可更新
- CASCADED：必须满足针对该视图所有视图条件才能更新，默认

更新视图是，原表值会发生变化。

视图主要用于查询操作，视图可以更新，但不建议更新。



查询视图创建语句

```mysql
SHOW CREATE VIEW <view_name>
```

删除视图

```mysql
DROP VIEW <view_name>
DROP VIEW IF EXISTS <view_name>
```

# 存储过程存储函数

存储过程和函数是事先经过编译并存储在数据库的一段SQL的集合，调用存储过程和函数可以简化应用开发人员很多工作，减少数据在数据库和应用服务器之间的传输（减少应用程序与数据库的交互），对于提高数据处理的效率是有好处的

存储过程：有返回值

存储函数：无返回值



## 基本操作

<u>！！！！使用存储过程前要求修改分隔符为例如$等符号，避免mysql识别；符号后直接执行</u>

```mysql
DELIMITER <delimiter>
```

创建存储过程

```mysql
CREATE PROCEDURE <procedure_name> (<[procedure_params]>)
BEGIN
	<SQL SENTENCE>
END 
```

调用存储过程

```mysql
CALL <procedure_name>
```

查看存储过程状态信息

```mysql
SHOW PROCEDURE STATUS
SHOW PROCEDURE STATUS/G
```

查询存储过程定义

```mysql
SHOW CREATE PROCEDURE <procedure_name>
SHOW CREATE PROCEDURE <procedure_name>/G
```

删除存储过程

```mysql
DROP PROCEDURE <procedure_name>
```

## 语法结构

### 变量

DECLARE 声明变量

```
DECLARE <var_name>[, <var_name1> ...]  <var_type> [<default_value>]
```

SET 设置变量

```
SET <set_sentence>
```

SELECT 语句设置变量

```mysql
SELECT <select_sentence> INTO <var_name>
```

### IF条件判断

```mysql
IF <search_condition> then 
	<statement>
[elseif <search_condition2> then 
 	<statement2>]
[else 
 	<statement3>]
END IF
```

### 参数

IN: 输入参数

OUT: 输出参数

INOUT：输入输出参数（既可以作为输入的值，返回值也用这个变量）

在参数中声明的参数已被声明，不需要重新声明

```mysql
CREATE PROCEDURE <procedure_name>([IN|OUT|INOUT] <params_name> <params_type>)
BEGIN 
	<SQL_statement>
END
```

例：

```mysql
CREATE PROCEDURE <procedure_name>(OUT <output_name> <output_type>)
BEGIN
	<SQL_statement>
END
```

调用(将@<output_name>作为接收返回值的参数，output_name 可与上文不一致)

```
CALL <procedure_name>(@<output_name>)
```

```mysql
SELECT @<output_name>
```

其中 <name> 为临时变量,只有当前语句有效

@<name> 为用户会话变量，在整个回话过程中起作用

@@<name> 为系统变量

### Case语法结构

条件语法有两种结构

```mysql
CASE <case_value>
	WHEN <value1> THEN 
		<statement1>
	[WHEN <value2> THEN 
  	<statement2>]
  ELSE 
  	<statement3>
END CASE
```

```mysql
CASE
	WHEN <condition> THEN
		<statement1>
	[WHEN <condition> THEN
		<statement2>]
	ELSE 
		<statement3>
END CASE
```

### 循环

while循环

```mysql
WHILE <condition> DO
	<statement>
END WHILE
```

Repeat 循环

```mysql
REPEAT
	<statement>
	UNTIL <condition>
END REPEAT
```

loop循环(loop实现循环时，需要利用其他语句（例如LEAVE）定义推出条件)

```mysql
[<label>:] LOOP
	<statement>
END LOOP [<label>]
```

```mysql
[<label>:] LOOP
	<statement>
	IF <condition> THEN
		LEAVE <label>
	END IF
END LOOP [<label>]
```

### 游标

游标是用来访问查询结果集的数据类型

本质是用于将select返回的数据集保存在变量中

声明游标

```mysql
DECLARE <cursor_name> CURSOR FOR <select_statement>
```

打开游标

```mysql
OPEN <cursor_name>
```

捕获内容

游标最开始会出现在列表的第一行，每次调用FETCH函数就会往下移动一行（var的数量需要与列表行数一致）

```mysql
FETCH <cursor_name> INTO [<var_name1>,<var_name2>...]
```

退出游标

```mysql
CLOSE <cursor_name>
```


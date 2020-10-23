[TOC]

# 查询优化

## 优化特定类型的查询


### 优化limit分页 -> 大偏移量使用覆盖索引

在很多应用场景中我们需要将数据进行分页，一般会使用limit加上偏移量的方法实现，同时加上合适的orderby 的子句，如果这种方式有索引的帮助，效率通常不错，否则的化需要进行大量的文件排序操作，还有一种情况，**当偏移量非常大的时候，前面的大部分数据都会被抛弃，这样的代价太高。**

**要优化这种查询的话，要么是在页面中限制分页的数量，要么优化大偏移量的性能**

==优化此类查询的最简单的办法就是尽可能地使用覆盖索引，而不是查询所有的==

    select film_id,description from film order by title limit 50,5
    
    select film.film_id,film.description from film inner join (select film_id from film order by title limit 50,5) as lim using(film_id);

    查看执行计划查看扫描的行数
    //执行计划：第一条的rows=1000,第二条的rows=111

### 优化union查询

集合运算：union、union all; 

union:重复行只显示一次；union all：重复行重复显示

**mysql不支持intersect（交集）、except（差集）**

union的应用：行转列

什么是行转列？

    name  subject  score
    zs     shuxue   89
    zs     yuwen   89
    zs     yingyu   89
    ls     shuxue   90
    ls     yuwen   90
    ls     yingyu   90
    
    行转列
    name yuwen  shuxue   yingyu 
    
**面试：行转列（常问）**

oracle的有四种实现方式：join  /   union   /   decode / case when

**mysql有三种实现方式： join /  union  /  case when**

**基础课程中有相关的实现方式《04行专列.md》**

**也有相应的面试题《经典面试题.md》**

---


mysql总是通过**创建并填充临时表的方式来执行union查询**，因此**很多优化策略在union查询中都没法很好的使用**。**经常需要手工的将where、limit、order by等子句下推到各个子查询中，以便优化器可以充分利用这些条件进行优化**


==除非确实需要服务器消除重复的行，否则一定要使用union all，因此没有all关键字，mysql会在查询的时候给临时表加上distinct的关键字，这个操作(挨个进行比较)的代价很高==

**这也是为什么计算DV（distinct value，使用hyperloglog算法）是估算值，就是因为计算精确值的代价太高了**


### 推荐使用用户自定义变量

用户自定义变量是一个容易被遗忘的mysql特性，但是如果能够用好，在某些场景下可以写出非常高效的查询语句，在查询中混合使用过程化和关系话逻辑的时候，自定义变量会非常有用。

用户自定义变量是一个用来存储内容的临时容器，在连接mysql的整个过程中都存在。

用户自定义变量 类似oracle的row_num

**问题：mysql的开窗函数 了解吗？**

==mysql8.0+ 可以直接使用窗口函数row_number() over(order by )==

https://www.cnblogs.com/gered/p/10430829.html

大数据课-spark会讲到

select @@autocommit; //==@@代表系统变量；   @代表用户自定义变量==

《mysql练习题.md》中用到了大量的 用户自定义变量

**无法自定义系统变量**

#### 自定义变量的使用
    
    set @i:=1
    set @i:=@i+1
    set @one:=1
	set @min_actor:=(select min(actor_id) from actor)
	set @last_week:=current_date-interval 1 week;

#### 自定义变量的限制

1、无法使用查询缓存

2、**不能在使用常量或者标识符的地方使用自定义变量，例如表名、列名或者limit子句**

3、用户自定义变量的生命周期是在一个连接中有效，所以不能用它们来做连接之间的通信【==自定义变量只在当前会话有效==】

4、不能显式地声明自定义变量地类型

5、==mysql优化器在某些场景下可能会将这些变量优化掉，这可能导致代码不按预想地方式运行【一般情况下没有问题的，案例没有找到】==

6、**赋值符号：=的优先级非常低，所以在使用赋值表达式的时候应该明确的使用括号**

7、==使用未定义变量不会产生任何语法错误==


#### 自定义变量的使用案例

**优化排名语句**

1、在给一个变量赋值的同时使用这个变量

    select actor_id,@rownum:=@rownum+1 as rownum from actor limit 10; //不会报错
    
    set @rownum=100;

2、查询获取演过最多电影的前10名演员，然后根据出演电影次数做一个排名
	
	set @actor_number:=0;
	
	select t1.*,@actor_number:=@actor_number+1 as number from (select actor_id,count(*) as ant from film_actor group by actor_id order by ant desc limit 10) t1;


**避免重新查询刚刚更新的数据**

3.**当需要高效的更新一条记录的时间戳，同时希望查询当前记录中存放的时间戳是什么**
    
    create table t1(id int,t_date datetime);
    insert into t1 values(1,now());
    
    //方式一
    update t1 set  t_date=now() where id =1;
    select t_date from t1 where id =1;
	
	//方式二
	update t1 set t_date = now() where id = 1 and @now:=now();
    select @now; //直接获取时间，不需要再重新查询
    
4.**确定取值的顺序**

**在赋值和读取变量的时候可能是在查询的不同阶段**，==所以：自定义变量不要乱用==

    set @rownum:=0;
    select actor_id,@rownum:=@rownum+1 as cnt from actor where @rownum<=1;
    因为where和select在查询的不同阶段执行，所以看到查询到两条记录，这不符合预期

    set @rownum:=0;
    select actor_id,@rownum:=@rownum+1 as cnt from actor where @rownum<=1 order by first_name
    正常顺序为：where先执行，orderby后执行
    当引入了order by之后，发现打印出了全部结果，这是因为order by引入了文件排序，而where条件是在文件排序操作之前取值的
    
    引入自定义变量以后，先执行orderby ,后执行where条件，执行顺序不一样

    解决这个问题的关键在于让变量的赋值和取值发生在执行查询的同一阶段：
    set @rownum:=0;
    select actor_id,@rownum as cnt from actor where (@rownum:=@rownum+1)<=1;
    结果为：一条数据

==注意：如果面试时问索引的话，一般都会问到底，直到不会==

# 分区表

官网 -》 partitioning 有一整个章节来介绍

==mysql的数据文件带#说明是分区表==

**对于用户而言，分区表是一个独立的逻辑表，但是底层是由多个物理子表组成**。分区表对于用户而言是一个完全封装底层实现的黑盒子，对用户而言是透明的，从文件系统中可以看到多个使用#分隔命名的表文件。

mysql在创建表时使用partition by子句**定义每个分区存放的数据**，在执行查询的时候，**优化器会根据分区定义过滤那些没有我们需要数据的分区**，这样查询就无须扫描所有分区。

分区的主要目的是将数据按照一个较粗的力度分在不同的表中，这样可以将相关的数据存放在一起。

## 分区表的应用场景

1.**表非常大以至于无法全部都放在内存中**，或者**只在表的最后部分有热点数据**，其他均是历史数据

2.分区表的数据更容易维护

    批量删除大量数据可以使用清除整个分区的方式
    对一个独立分区进行优化、检查、修复等操作

3.**分区表的数据可以分布在不同的物理设备上，从而高效地利用多个硬件设备**

    一般mysql是集群模式

4.可以使用分区表来避免某些特殊的瓶颈

    innodb的单个索引的互斥访问【innodb的锁是对索引加锁】
    ext3文件系统的inode锁竞争【ls -li 可以查看inode,inode不能重复】
    //linux的软连接、硬连接？？？

5.**可以备份和恢复独立的分区**




## 分区表的限制

1.==一个表最多只能有1024个分区，在5.7版本的时候可以支持8196个分区==

问题：为什么要对分区数量做限制？

    跟linux中FD 文件描述符有关
    如果打开的文件描述符过多，会影响IO效率；
    文件打开个数是跟内存相关的，1G内存可以打开10万个文件
    65535？？？
    
    cd /proc/$$/fd
    ulimit -a //open files=1024 最大为1024

2.在早期的mysql中，分区表达式必须是整数或者是返回整数的表达式，**在mysql5.5中，某些场景可以直接使用列来进行分区**

3.**如果分区字段中有主键或者唯一索引的列，那么所有主键列和唯一索引列都必须包含进来**

   
```
create table t2(id int primary key,name varchar(10) unique)
PARTITION BY RANGE (id) (
    PARTITION p0 VALUES LESS THAN (6),
    PARTITION p1 VALUES LESS THAN (11),
    PARTITION p2 VALUES LESS THAN (16),
    PARTITION p3 VALUES LESS THAN (21)
);
# A unique index must include all columns in the tables's partition function


```

**4.分区表无法使用外键约束【外键约束会失效】**

## 分区表的原理

《分区表的底层原理.md》

## 分区表的类型：6类

问题：**一般用时间做分区列 用的比较多，为什么？**

因为**会形成一张时间拉链表，查询数据的时候，只需要查询某一天或某一个时间范围的数据就可以**，这样可以大大减少整个数据检索的IO量，

官网学习 -》 

### **范围分区**

根据列值在给定范围内将行分配给分区

<范围分区.md>
		
### **列表分区**

**类似于按range分区，区别在于list分区是基于列值匹配一个离散值集合中的某个值来进行选择**
	
LIST(列名)
	
```
CREATE TABLE employees (

    id INT NOT NULL,

    fname VARCHAR(30),

    lname VARCHAR(30),

    hired DATE NOT NULL DEFAULT '1970-01-01',

    separated DATE NOT NULL DEFAULT '9999-12-31',

    job_code INT,

    store_id INT

)

PARTITION BY LIST(store_id) (

    PARTITION pNorth VALUES IN (3,5,6,9,17),

    PARTITION pEast VALUES IN (1,2,10,11,19,20),

    PARTITION pWest VALUES IN (4,12,13,14,18),

    PARTITION pCentral VALUES IN (7,8,15,16)

);
```

	
### **列分区** -用处更少

**mysql从5.5开始支持column分区，可以认为i是range和list的升级版，在5.5之后，可以使用column分区替代range和list，但是column分区只接受普通列不接受表达式**

RANGE COLUMNS(c1)

RANGE COLUMNS(c1,c3)

LIST COLUMNS(c3)


```
 CREATE TABLE `list_c` (

 `c1` int(11) DEFAULT NULL,

 `c2` int(11) DEFAULT NULL

) ENGINE=InnoDB DEFAULT CHARSET=latin1

/* PARTITION BY RANGE COLUMNS(c1)

(PARTITION p0 VALUES LESS THAN (5) ENGINE = InnoDB,

 PARTITION p1 VALUES LESS THAN (10) ENGINE = InnoDB) */



 CREATE TABLE `list_c` (

 `c1` int(11) DEFAULT NULL,

 `c2` int(11) DEFAULT NULL,

 `c3` char(20) DEFAULT NULL

) ENGINE=InnoDB DEFAULT CHARSET=latin1

/* PARTITION BY RANGE COLUMNS(c1,c3)

(PARTITION p0 VALUES LESS THAN (5,'aaa') ENGINE = InnoDB,

 PARTITION p1 VALUES LESS THAN (10,'bbb') ENGINE = InnoDB) */



 CREATE TABLE `list_c` (

 `c1` int(11) DEFAULT NULL,

 `c2` int(11) DEFAULT NULL,

 `c3` char(20) DEFAULT NULL

) ENGINE=InnoDB DEFAULT CHARSET=latin1

/* PARTITION BY LIST COLUMNS(c3)

(PARTITION p0 VALUES IN ('aaa') ENGINE = InnoDB,

 PARTITION p1 VALUES IN ('bbb') ENGINE = InnoDB) */


```


### **hash分区**

基于用户定义的表达式的返回值来进行选择的分区，该表达式使用将要插入到表中的这些行的列值进行计算。**这个函数可以包含myql中有效的、产生非负整数值的任何表达式**

就是**简单的取模运算**，如果hash算法复杂，那么在分区数据操作时时间复杂度也会变高

 HASH(store_id)
 
 LINEAR HASH(YEAR(hired))

```
CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT,
    store_id INT
)
PARTITION BY HASH(store_id)
PARTITIONS 4;
# 使用列store_id,只创建4个分区，取模运算


CREATE TABLE employees (
    id INT NOT NULL,
    fname VARCHAR(30),
    lname VARCHAR(30),
    hired DATE NOT NULL DEFAULT '1970-01-01',
    separated DATE NOT NULL DEFAULT '9999-12-31',
    job_code INT,
    store_id INT
)
PARTITION BY LINEAR HASH(YEAR(hired))
PARTITIONS 4;

```


### **key分区**

类似于hash分区，区别在于key分区只支持一列或多列，且mysql服务器提供其自身的哈希函数，必须有一列或多列包含整数值

LINEAR KEY (col1)

==区别：hash分区是普通列，key分区是主键和唯一键==

问题：如果一张表既有主键又有唯一键，创建分区时需要都指定吗？是的，都要包含

```
CREATE TABLE tk (
    col1 INT NOT NULL,
    col2 CHAR(5),
    col3 DATE
)
PARTITION BY LINEAR KEY (col1)
PARTITIONS 3;

```



### **子分区**

在分区的基础之上，再进行分区后存储

PARTITION BY RANGE(id)

SUBPARTITION BY HASH(sGrade) SUBPARTITIONS 2

ts#P#p0#SP#p0sp0.ibd

ts#P#p0#SP#p0sp1.ibd



```
CREATE TABLE `t_partition_by_subpart`
(
  `id` INT AUTO_INCREMENT,
  `sName` VARCHAR(10) NOT NULL,
  `sAge` INT(2) UNSIGNED ZEROFILL NOT NULL,
  `sAddr` VARCHAR(20) DEFAULT NULL,
  `sGrade` INT(2) NOT NULL,
  `sStuId` INT(8) DEFAULT NULL,
  `sSex` INT(1) UNSIGNED DEFAULT NULL,
  PRIMARY KEY (`id`, `sGrade`)
)  ENGINE = INNODB
PARTITION BY RANGE(id)
SUBPARTITION BY HASH(sGrade) SUBPARTITIONS 2
(
PARTITION p0 VALUES LESS THAN(5),
PARTITION p1 VALUES LESS THAN(10),
PARTITION p2 VALUES LESS THAN(15)
);

分区文件为：ts#P#p0#SP#p0sp0.ibd
ts#P#p0#SP#p0sp1.ibd
ts#P#p1#SP#p1sp0.ibd
ts#P#p1#SP#p1sp1.ibd
ts#P#p2#SP#p2sp0.ibd
ts#P#p2#SP#p2sp1.ibd

```

==面试的时候如何说分区，建议带上应用场景：哪个业务有哪张表数据量比较大或查询比较慢，因此做了分区，把热数据和历史数据做了区分==

==面试聊业务时，别慌，你是最熟悉业务的，除非有特别大的漏洞；如果被指出bug，就说我回去跟我的leader说一下==

**以前的分区是怎么设计的？**

业务数据量大的话，一般使用分库分表，用mycat实现

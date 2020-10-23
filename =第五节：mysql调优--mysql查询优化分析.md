[TOC]

# 查询优化

在编写快速的查询之前，需要清楚一点，**真正重要的是响应时间**，而且要知道在整个SQL语句的执行过程中**每个步骤都花费了多长时间，要知道哪些步骤是拖垮执行效率的关键步骤**，想要做到这点，**必须要知道查询的生命周期，然后进行优化**，不同的应用场景有不同的优化方式，不要一概而论，具体情况具体分析

问题：Mysql是开源的吗？是的

问题：可以在Mysql上面做二次开发？有机会，参与一下mysql的二次开发


## 查询慢的原因

网络、CPU、IO、上下文切换、系统调用、生成统计信息(profiles/schema)、锁等待时间

### 面试中问到Mysql中的锁，应该如何回答？

    Mysql中的锁跟存储引擎相关，
    
    InnoDB：共享锁、排它锁；可以锁表，也可以锁行；锁的对象：索引【锁的列是索引列的话，是行锁；如果不是索引会从行锁退化为表锁】
    
    MyISAM：共享读锁、独占写锁；只能锁表；

    自增锁：
    
    间隙锁：


## 优化数据访问

### 1.查询性能低下的主要原因是访问的数据太多，某些查询不可避免的需要筛选大量的数据，我们可以通过**减少访问数据量的方式进行优化**

#### **确认应用程序是否在检索大量超过需要的数据** -》 查看执行计划-rows

    --下面的查询不会利用索引
    explain select rental_id,staff_id from rental where rental_date>'2005-05-25' order by rental_date,inventory_id\G
    *************************** 1. row ***************************
               id: 1
      select_type: SIMPLE
            table: rental
       partitions: NULL
             type: ALL
    possible_keys: rental_date
              key: NULL
          key_len: NULL
              ref: NULL
             rows: 16005
         filtered: 50.00
            Extra: Using where; Using filesort
    ============================================
    explain select rental_id,staff_id from rental where rental_date>'2006-05-25' order by rental_date,inventory_id\G
    *************************** 1. row ***************************
    Extra:Using index condition
    上面的type=all,它的type=range
    为什么呢？因为rows，上面的rows=16005,下面的rows=1;数据量大的时候，不会使用索引；
    
    数据量大到什么程序不会索引呢？《高效能Mysql》说阈值是30%

==数据量大到什么程序不会索引呢？《高效能Mysql》说阈值是30%==

---

**面试题：explain select * from rental limit 10000,5; 这条语句会使用索引吗？ 不会**

    explain select * from rental limit 10000,5;
    *************************** 1. row ***************************
    type=all
    rows=16005
    
**原因：limit前面的值如果过大，会触发全表扫描，而不会使用索引**
    
**那么如何优化？ 子查询**

**select * from rental a join (select rental_id from rental limit 10000,5) b on a.rental_id = b.rental_id;**

==数据量越大，效果越明显；数据量在100万可以明显看出==

#### **确认mysql服务器层是否在分析大量超过需要的数据行**


### 2.是否向数据库请求了不需要的数据


#### 查询不需要的记录

我们常常会误以为mysql会只返回需要的数据，实际上mysql却是先返回全部结果再进行计算，在日常的开发习惯中，经常是先用select语句查询大量的结果，然后获取前面的N行后关闭结果集。

**优化方式是在查询后面添加limit**

#### 多表关联时返回全部列

    select * from actor inner join film_actor using(actor_id) inner join film using(film_id) where film.title='Academy Dinosaur';

    select actor.* from actor inner join film_actor using(actor_id) inner join film using(film_id) where film.title='Academy Dinosaur';
    
    // 执行时间少
    
==建议：不要select * ==

==另外，sql语句对 对应的表名后 使用 别名；==

**【原因：在进行语法解析、词法解析时，会用这个别名代表这张表，匹配的时候用别名就可以匹配了，不需要到表里面进行匹配了】**


#### 总是取出全部列

在公司的企业需求中，禁止使用select *,虽然这种方式能够简化开发，但是会影响查询的性能，所以尽量不要使用


#### 重复查询相同的数据 -> 缓存；mysql8已经去掉了缓存

如果需要不断的重复执行相同的查询，且每次返回完全相同的数据，因此，基于这样的应用场景，我们可以将这部分数据缓存起来，这样的话能够提高查询效率

Mysql8已经将缓存去掉了

缓存满了以后，会进行淘汰策略，LRU

## 执行过程的优化

### 查询缓存

问题：你们公司的Mysql版本？ 5.7

在解析一个查询语句之前，如果查询缓存是打开的，那么mysql会优先检查这个查询是否命中查询缓存中的数据，**如果查询恰好命中了查询缓存**，那么会在返回结果之前会**检查用户权限**，如果权限没有问题，那么mysql会跳过所有的阶段，就直接从缓存中拿到结果并返回给客户端

常量表放入缓存，**经常变化的表不要放入缓存**

### 查询优化处理

mysql查询完缓存之后会经过以下几个步骤：**解析SQL、预处理、优化SQL执行计划**，这几个步骤出现任何的错误，都可能会终止查询

#### 语法解析器和预处理

mysql通过**关键字将SQL语句进行解析**，并**生成一颗解析树**，**mysql解析器将使用mysql语法规则验证和解析查询**，例如验证使用使用了错误的关键字或者顺序是否正确等等，**预处理器会进一步检查解析树是否合法**，例如表名和列名是否存在，是否有歧义，还会验证权限等等

AST-tree(大数据周老师讲) 不需要自己做解析，有一些开源框架：**calcite**

www.apache.org -> projects -> calcite  ->

AST 抽象语法树

#### 查询优化器（重点）

**当语法树没有问题之后**，相应的要**由优化器将其转成执行计划**，一条查询语句可以使用非常多的执行方式，最后都可以得到对应的结果，但是不同的执行方式带来的效率是不同的，**优化器的最主要目的就是要选择最有效的执行计划**

**优化器有：CBU 基于成本的优化；RBU 基于规则的优化**

**约束大于规范，所以选择CBU**

**mysql使用的是基于成本的优化器**，在优化的时候会尝试预测一个查询使用某种查询计划时候的成本，并选择其中成本最小的一个

##### show status like 'last_query_cost';

    select count(*) from film_actor;
    show status like 'last_query_cost';

可以看到这条查询语句**大概需要做1104个数据页才能找到对应的数据**，这是经过一系列的统计信息计算来的

	每个表或者索引的页面个数
	索引的基数
	索引和数据行的长度
	索引的分布情况
	
##### 在很多情况下mysql会选择错误的执行计划，原因如下：

1.统计信息不准确 【原因：**InnoDB因为其mvcc的架构，并不能维护一个数据表的行数的精确统计信息**】


2.**执行计划的成本估算不等同于实际执行的成本**

有时候某个执行计划虽然需要读取更多的页面，但是他的成本却更小，因为如果**这些页面都是顺序读或者这些页面都已经在内存中的话**，那么它的访问成本将很小，mysql层面并不知道哪些页面在内存中，哪些在磁盘，所以**查询之际执行过程中到底需要多少次IO是无法得知**的

3.mysql的最优可能跟你想的不一样

**mysql的优化是基于成本模型的优化，但是有可能不是最快的优化**

4.**mysql不考虑其他并发执行的查询**

**mysql是基于单条的预估**

5.mysql不会考虑不受其控制的操作成本【执行存储过程或者用户自定义函数的成本】

**虽然现在存储过程很少用，但是建议了解一下**

##### 优化器的优化策略：动态优化、静态优化

==面试时，可能问不到，但是可以提一下：优化器可能会出现会出现问题==

**静态优化**：直接对解析树进行分析，并完成优化

**动态优化**：**动态优化与查询的上下文有关，也可能跟取值、索引对应的行数有关**

就是上面提到的sql语句：rental_date>'2006-05-25'    rental_date>'2005-05-25'

**查询的上下文：例如：A表在缓存，B表不在缓存**

上下文：没有人可能说清楚上下文到底是什么；我们可以大概知道有哪些信息，可能会造成什么影响

**mysql对查询的静态优化只需要一次，但对动态优化在每次执行时都需要重新评估**


##### 优化器的优化类型

**1.重新定义关联表的顺序**

**2.将外连接转化成内连接，内连接的效率要高于外连接**

**问题：为什么内连接的效率要高于外连接的效率**？

    因为内连接获取的数据量要少于外连接获取的数据量
    左外连接、右外连接都是使用了循环嵌套的方式，需要将左边的数据都展示，然后再逐条匹配右表的数据
    可以查看join原理
    

**3.使用等价变换规则**，mysql可以使用一些等价变化来简化并规划表达式

例如： a>4 and a<4 变换为  a!=4

**书写sql的时候稍微注意，但是对cpu的影响并不大**

**4.优化count(),min(),max()**

**聚合函数，最好带上分组条件；因为分组会使用索引，效率会高很多**

**索引和列是否可以为空通常可以帮助mysql优化这类表达式：例如，要找到某一列的最小值，只需要查询索引的最左端的记录即可，不需要全文扫描比较**

5.预估并转化为常数表达式，当mysql检测到一个表达式可以转化为常数的时候，就会一直把该表达式作为常数进行处理

    explain select film.film_id,film_actor.actor_id from film inner join film_actor using(film_id) where film.film_id = 1
    // type=const

6.**索引覆盖扫描**，当索引中的列包含所有查询中需要使用的列的时候，可以使用覆盖索引

7.**子查询优化**

mysql在某些情况下**可以将子查询转换一种效率更高的形式**，从而减少多个查询多次对数据进行访问，例如 **将经常查询的数据放入到缓存中**

8.**等值传播**(非常有意思)

**平时一直在用，只是不知道名词：等值传播**；==面试时问到“等值传播”，还以为很高端的东西，实际很low==

**如果两个列的值通过等式关联，那么mysql能够把其中一个列的where条件传递到另一个上：**

    explain select film.film_id from film inner join film_actor using(film_id) where film.film_id > 500;
    
    这里使用film_id字段进行等值关联，film_id这个列不仅适用于film表而且适用于film_actor表
    
    explain select film.film_id from film inner join film_actor using(film_id) where film.film_id > 500 and film_actor.film_id > 500;
    

##### 关联查询

###### join的实现方式原理

Simple Nested-Loop Join

Index Nested-Loop Join

Block Nested-Loop Join

**join_buffer_size的默认值是256K，join_buffer_size的最大值在MySQL 5.1.22版本前是4G-1，而之后的版本才能在64位操作系统下申请大于4G的Join Buffer空间。**

**使用Block Nested-Loop Join算法需要开启优化器管理配置的==optimizer_switch(优化器开关)==的设置block_nested_loop为on，默认为开启。**

    Block Nested-Loop Join
    （1）Join Buffer会缓存所有参与查询的列而不是只有Join的列。
    （2）可以通过调整 join_buffer_size 缓存大小 (show variables liek 'join_buffer')
    （3）join_buffer_size的默认值是256K，join_buffer_size的最大值在MySQL 5.1.22版本前是4G-1，而之后的版本才能在64位操作系统下申请大于4G的Join Buffer空间。
    （4）使用Block Nested-Loop Join算法需要开启优化器管理配置的optimizer_switch的设置block_nested_loop为on，默认为开启。
    
    show variables like '%optimizer_switch%'

###### 案例演示

    查看不同的顺序执行方式对查询性能的影响：

    explain select film.film_id,film.title,film.release_year,actor.actor_id,actor.first_name,actor.last_name from film inner join film_actor using(film_id) inner join actor using(actor_id);
    
    执行计划：读取表的顺序依次为：actor、film_actor、film，并且rows=200+27+1=228
    
    查看执行的成本:show status like 'last_query_cost';  //7892
    
    ========================
    
    按照自己预想的规定顺序执行：
    
    explain select straight_join film.film_id,film.title,film.release_year,actor.actor_id,actor.first_name,actor.last_name from film inner join film_actor using(film_id) inner join actor using(actor_id);
    
    执行计划：读取表的顺序依次为：film、film_actor、actor,并且rows=1000+5+1=10006
    
    查看执行的成本：show status like 'last_query_cost'; //8885
    
    
结论：

**mysql的优化器会对join进行优化；读取表的顺序不一定是sql中表的顺序**

**如果想要指定join时读取表的数据使用关键字：straight**
    


##### 排序优化

**无论如何排序都是一个成本很高的操作**，所以**从性能的角度出发，应该尽可能避免排序或者尽可能避免对大量数据进行排序。**

**推荐利用索引进行排序**，但是**当不能使用索引的时候，mysql就需要自己进行排序，如果数据量小则再内存中进行，如果数据量大就需要使用磁盘，mysql中称之为filesort**

如果**需要排序的数据量小于排序缓冲区**(show variables like '%sort_buffer_size%';),**mysql使用内存进行快速排序操作**，==如果内存不够排序，那么mysql就会先将树分块，对每个独立的块使用快速排序进行排序，并将各个块的排序结果存放再磁盘上，然后将各个排好序的块进行合并，最后返回排序结果==

###### 排序的算法

**两次传输排序**

第一次数据读取是将需要排序的字段读取出来，然后进行排序，第二次是将排好序的结果按照需要去读取数据行。
    
**这种方式效率比较低，原因是第二次读取数据的时候因为已经排好序，需要去读取所有记录而此时更多的是随机IO，读取数据成本会比较高**
        
==两次传输的优势：在排序的时候存储尽可能少的数据，让排序缓冲区可以尽可能多的容纳行数来进行排序操作==

**单次传输排序**

**先读取查询所需要的所有列，然后再根据给定列进行排序，最后直接返回排序结果，此方式只需要一次顺序IO读取所有的数据，而无须任何的随机IO，**
        
==问题在于查询的列特别多的时候，会占用大量的存储空间，无法存储大量的数据==

==当需要排序的列的总大小超过**max_length_for_sort_data**定义的字节，mysql会选择双次排序，反之使用单次排序，当然，用户可以设置此参数的值来选择排序的方式==

    show variables like 'max_length_for_sort_data' //1024
    
    
## 优化特定类型的查询

### 优化count()查询

count()是特殊的函数，有两种不同的作用，一种是某个列值的数量，也可以统计行数

==问题：count(*) count(1) count(id)，哪个效率更高？都一样==

通过**查看explain执行计划**，**show profiles执行时间**，**show status like 'last_query_cost'执行成本**，结果都是一样的

==总有人认为myisam的count函数比较快，这是有前提条件的，只有没有任何where条件的count(*)才是比较快的==

使用近似值

    在某些应用场景中，不需要完全精确的值，可以参考使用近似值来代替，比如可以使用explain来获取近似的值

    其实在很多OLAP的应用中，需要计算某一个列值的基数，有一个计算近似值的算法叫hyperloglog。


更复杂的优化

    一般情况下，count()需要扫描大量的行才能获取精确的数据，其实很难优化，在实际操作的时候可以考虑使用索引覆盖扫描，或者增加汇总表，或者增加外部缓存系统。


### 优化关联查询

**确保on或者using子句中的列上有索引**，在创建索引的时候就要考虑到关联的顺序

**确保任何的groupby和order by中的表达式只涉及到一个表中的列，这样mysql才有可能使用索引来优化这个过程**


### 优化子查询

**子查询的优化最重要的优化建议是尽可能使用关联查询代替**

**问题：为什么尽量不要用子查询？因为临时表，临时表也是IO**

==join中的临时表是用来存放最终的结果的；子查询的临时表是用来存放子查询的经过，外层的查询还要跟这个临时表进行关联==

### 优化group by和distinct（作废）

很多场景下，mysql使用相同的方法来优化groupby和distinct的查询，使用索引是最有效的方式，当时有很多的情况下无法使用索引，可以使用临时表或文件排序来分组

如果对关联查询做分组，并且是按照查找表中某个列进行分组，那么可以采用查找表的标识列分组的效率比其他列更高

    select actor.first_name,actor.last_name,count(*) from film_actor inner join actor using(actor_id) group by actor.first_name,actor.last_name;
    
    select actor.first_name,actor.last_name,count(*) from film_actor inner join actor using(actor_id) group by actor.actor_id;
    
==1.从语法层次，group by中没有select的列，是不报错的【新的认知】==

2.如果first_name,last_name中没有重复值，group by first_name,last_name 和“group by actor_id”的结果是一样的

3.一般使用groug by都是因为last_name,first_name是有重复值的，所以意义不大，但是第1点是很重要的




	
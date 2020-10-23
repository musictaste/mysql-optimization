[TOC]

# 调优时，mysql语句、执行计划都没有问题，但是mysql就是慢，怎么解决？

使用performance schema监控mysql

在数据库运行时实时检查server的内部执行情况

简单点：mysql服务中执行了哪些线程？这些线程分别做什么事情？这些事情分别用了多长时间？占用了多少资源？

详细：performance_schema通过监视server的**事件**来实现监视server内部运行情况， “事件”就是server内部活动中所做的任何事情以及对应的时间消耗，利用这些信息来判断server中的相关资源消耗在了哪里？一般来说，**事件可以是函数调用、操作系统的等待、SQL语句执行的阶段（如sql语句执行过程中的parsing 或 sorting阶段）或者整个SQL语句与SQL语句集合**。事件的采集可以方便的提供server中的相关存储引擎对**磁盘文件、表I/O、表锁等资源的同步调用信息**

# performance_schema与information_schema的区别

performance_schema 数据库主要关注数据库运行过程中的**性能相关的数据**，

与information_schema不同，information_schema主要关注server运行过程中的**元数据信息**

# Mysql中有哪些存储引擎？

存储引擎：Innodb   MyISAM   memory(不能持久化，使用比较少)

其他的存储引擎：performance_schema

.frm 表结构  
.ibd  存储的数据文件，代表存储引擎为InnoDB  
.MYD【数据文件】或.MYI【索引文件】代表存储引擎为MyASM  

# 市场上主流的数据库线程池有哪些？有什么区别【加分项】

现在市面上的数据库连接池：DBCP(很少使用)   C3PO（老项目）  druid（德鲁伊 性能很强）【性能监控信息-github中文文档，面试时可以帮到】

==日本的BoneCP，连接比druid快，但是一些复杂的功能不支撑，例如：PScache、ExceptionSorter==

https://github.com/alibaba/druid/wiki/%E5%90%84%E7%A7%8D%E6%95%B0%E6%8D%AE%E5%BA%93%E8%BF%9E%E6%8E%A5%E6%B1%A0%E5%AF%B9%E6%AF%94

![103](299CB128AF1B432E9F506BB82BAFD87E)

**LRU** 

LRU是一个**性能关键指标**，特别Oracle，每个Connection对应数据库端的一个进程，如果数据库连接池遵从LRU，有助于数据库服务器优化，这是重要的指标。**在测试中，Druid、DBCP、Proxool是遵守LRU的。BoneCP、C3P0则不是。BoneCP在mock环境下性能可能好，但在真实环境中则就不好了。**

**PSCache**

PSCache是数据库连接池的关键指标。在Oracle中，类似SELECT NAME FROM USER WHERE ID = ?这样的SQL，**启用PSCache和不启用PSCache的性能可能是相差一个数量级的**。Proxool是不支持PSCache的数据库连接池，**如果你使用Oracle、SQL Server、DB2、Sybase这样支持游标的数据库，那你就==完全不用考虑Proxool==**。

**PSCache-Oracle-Optimized**

**Oracle 10系列的Driver**，如果开启PSCache，**会占用大量的内存**，必须做特别的处理，**启用内部的EnterImplicitCache等方法优化才能够减少内存的占用**。==这个功能只有DruidDataSource有==。如果你使用的是Oracle Jdbc，你应该毫不犹豫采用DruidDataSource。

**ExceptionSorter**

ExceptionSorter是一个很重要的**容错特性**，如果一个连接产生了一个不可恢复的错误，必须立刻从连接池中去掉，否则会连续产生大量错误。这个特性，**目前只有JBossDataSource和Druid实现**。Druid的实现参考自JBossDataSource，经过长期生产反馈补充。



# 问题：文件实际大小与占用空间有什么区别？

磁盘有page,4Kb, 使用4Kb是为了提高数据读取效率， 4K对齐

一个文件大小为4.23Kb,会占用两个Page,所以占用空间为8Kb

#问题：page页知道吗？有什么作用？

1.提供数据读取效率

2.**磁盘预读**：虽然程序只需要一个字节的内容，但是操作系统会将这1字节所在的page的数据都读取，认为接下来程序可能会用到

3.InnoDB每次读取16Kb的数据，提供磁盘和内存的IO效率

4.对索引也有作用：3层B+树，3个Page，12Kb，支持百万数据量的快速检索

# 面试题：int长度改短会有什么影响？
# 面试题：mysql中字段定义为int(1)，可以存储100000吗？

可以

没有影响；

跟底层存储字节数有关，int的默认4个字节，32位 

定义的长度只是规范，

```sql
mysql> create table test_int(id int(1));
Query OK, 0 rows affected (0.01 sec)

mysql> insert into test_int values(100000);
Query OK, 1 row affected (0.00 sec)
```

# 面试题：varchar长度改小会有什么影响？

```sql
# 如果已有数据的长度大于要修改的位宽，会报错，不能修改成功
# 另外如果插入的数据的长度大于规定的位宽，也无法插入

mysql> create database test;
mysql> use test;
mysql> create table test_varchar(name varchar(5));
mysql> insert into test_varchar values('12345');

mysql> insert into test_varchar values('12345678');
ERROR 1406 (22001): Data too long for column 'name' at row 1

mysql> alter table test_varchar modify name varchar(1);
ERROR 1265 (01000): Data truncated for column 'name' at row 1

```


# 问题：mysql中datetime、timestamp、date的区别

比较项 | datetime | timestamp | data
--- | ---|--- | ---
字节大小 |8 | 4 | 3
是否与时区有关 | 无关 | 有关 | 无关
精确范围 | 毫秒 | 秒 | 天
时间范围 | 很大 | 1970-01-01到2038-01-19 | 1000-01-01到9999-12-31 

# 问题：知道timestamp的时间范围吗？为什么是这个范围？

时间范围：1970-01-01到2038-01-19

原因：4个字节，32位，能表示的值为4294967296，42亿多，从格林尼治时间1970-01-01开始，加上42亿毫秒，就是32038-01-19

# 问题：mysql中char、varchar、text、blob数据类型的区别？

char： 定长；==最大长度255个字符==；会自动删除末尾的空格；检索、写效率会比varchar高，以空间换时间

应用场景  
1. 存储长度波动不大的数据，如：md5摘要  
2. 存储短字符串、经常更新的字符串

=====

varchar：变长；==最大长度65535个字节；不同的编码方式字符数不同【utf-8:65535/3; GBK:65535/2】==  
varchar(n) 如果n小于等于255，需要额外的一个字节保存长度；大于255需要额外的两个字节保存长度  
varchar(5)和varchar(255)占用的空间是一样的，但是实际文件大小是不一样的  
varchar在mysql5.6之前变更长度，或从255一下变更到255以上时，会导致锁表

应用场景  
1. 存储长度波动较大的数据，如：文章，有的会很短有的会很长  
2. 字符串很少更新的场景，每次更新后都会重算并使用额外存储空间保存长度  
3. ==适合保存多字节字符，如：汉字，特殊字符等== 

====

text:不设置长度  
Blob和Text都是==存储很大的数据而设计的字符串类型==，分别采用==二进制和字符方式存储==  
==Text和Blob，一般不用，如果过长可以存放在文件中，将文件路径保存在数据表中==

# 面试题：varchar变长会有什么影响？

**varchar(5)与varchar(255)保存同样的内容，硬盘存储空间相同，但内存空间占用不同，是指定的大小；也就是==实际文件大小不一样，占用空间一样==**

==varchar(n) n小于等于255使用额外一个字节保存长度，n>255使用额外两个字节保存长度==。

==varchar在mysql5.6之前变更长度，或者从255一下变更到255以上时时，都会导致锁表==。
    
#  ======================================================

# 什么是范式、反范式？

**范式：减少数据冗余，查询时需要关联**

**反范式：数据冗余，减少表关联**

==可以适当的数据冗余，但是需要注意：需要更新冗余的数据==，类似物化视图

**物化视图：oracle有，oracle提供两种机制：uncommit、undemand**



# 三范式

1NF：列不可分；【符合1NF的关系中的每个属性都不可再分】  
2NF：不能存在传递依赖；【保证一张表只描述一件事情】  
3NF：必须唯一的依赖于主键；【保证每列都和主键直接相关】  
**三范式的作用：减少数据冗余**  

# 阿里规范：超过三站表，禁止使用join，为什么？

join过程慢；

也得看数据量的大小，数据量少也很快

# 全排序为什么慢？如何优化？

全排序，意味着需要将数据全部加载到内存中，这个过程非常慢；

解决：  
1.把数据放到一张表中；  
2.或使用索引，索引底层数据结构是B+树，本身就是有序树，可以按照当前列进行排序，不需要放到内存进行排序

# 什么是自然主键、代理主键？

代理主键：例如id；与业务无关的，无意义的数字序列

自然主键：人的身份证号；事物属性中的自然唯一标识

==区分自然、代理主键：是否跟业务相关==

**推荐使用代理主键：**

1. 它们不与业务耦合，因此更容易维护
2. 一个大多数表，最好是全部表，通用的键策略能够减少需要编写的源码数量，减少系统的总体拥有成本


# 你们中文存储采用哪种字符集？  -》 utfmb4

**通过使用合适的字符集，可以帮助我们尽可能减少数据量，进而减少IO操作次数。**

1. **纯拉丁字符**能表示的内容，没必要选择**latin1**之外的其他字符编码，因为这会节省大量的存储空间。
2. 如果我们可以**确定不需要存放多种语言，就没必要非得使用UTF8或者其他UNICODE字符类型**，这回造成大量的存储空间浪费。
3. ==中文存储，不建议utf-8；中文有可能是2个或3个字符来存储；建议utf8mb4== 

# MyISAM 和InnoDB的对比

![203](192EA054C677408E905916308E36C637)

非聚簇索引：索引文件和数据文件不放在一起  
聚簇索引：索引文件和数据文件放在一起

# 如何区分InnoDB是表锁还是行锁？
InnoDB存储引擎默认是给索引加锁；

所以只需要判断一件事：==增删改查的那个列是否是建索引的那个列==；如果是索引列，就是行锁；否则为表锁

# 为什么尽量不要使用like？
like中有%，那么索引不生效

# 存储引擎可以转换吗？

可以

但是需要注意：**转换时需要对索引、列也进行转换，这个转换有很多需要注意的点，一般没人这么干**

# mysql优化，首先要知道哪块执行慢，然后去优化；不能只根据执行计划就去优化


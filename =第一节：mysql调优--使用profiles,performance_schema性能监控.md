[TOC]

# mysql5.7安装

《MYSQL5.7详细安装步骤.md》

# mysql基础架构

![101](738FC906D6004E02AF17FE10DBCF2ECF)

![102](9926C4CCCEE84228986DBB31858932EE)

1. **客户端**：向数据库发送请求

2. **Mysql Server**

-  1.连接器：管理连接，权限认证(用户名密码验证)

    sql请求

- 2.分析器：词法分析，语法分析

    from where  
    语法分析 -》**抽象语法树AST**

- 3.优化器：执行计划，索引选择

    优化sql语句，规定执行流程
    
    RBO:基于规则的优化  
    ==CBO:基于成本的优化  -使用较多==
    
    可以查看sql语句的执行计划，可以采用对应的优化点，来加快查询

- 4.执行器：操作引擎，返回结果

    sql语句的实际执行组件  
    跟存储引擎挂钩

- 存储缓存，在mysql8.0以后被废除了

    **提升缓存命中率：将不变的数据放到缓存中**

    ==大部分操作发生在：分析器、优化器==

3. 存储引擎

    不同的文件位置、不同的文件格式  
    InnoDB：磁盘  
    MyISAM：磁盘  
    Memory：内存  

# 性能监控

## 1.使用show profile查询剖析工具，可以指定具体的type

==mysql 官网   -》搜profile -> sql statements -> show statements ->show profilelist statement==

**此工具默认是禁用的**，可以通过服务器变量在绘画级别动态的修改


```sql

--#低版本可以用，官网有明确的提示:未来可能会被弃用
--The SHOW PROFILE and SHOW PROFILES statements are deprecated and will be removed in a future MySQL release. Use the Performance Schema instead; see Section 25.19.1, “Query Profiling Using Performance Schema”.
--当设置完成之后，在服务器上执行的所有语句，都会测量其耗费的时间和其他一些查询执行状态变更相关的数据。
set profling=1

select * from mylock;

--在mysql的命令行模式下只能显示两位小数的时间，可以使用如下命令查看具体的执行时间
show profiles; 

--默认为最近一条查询
show profile;
   
--执行如下命令可以查看详细的每个步骤的时间： 
--查看queryID=1的执行时间，默认为最近一条查询
show profile for query 1;  
#starting  开始运行，准备mysql服务  浪费时间
#sengding data 发送数据  浪费实际
#这两个浪费时间，是正常的

+----------------------+----------+
| Status               | Duration |
+----------------------+----------+
| starting             | 0.000058 |
| checking permissions | 0.000006 |
| Opening tables       | 0.000012 |
| init                 | 0.000065 |
| System lock          | 0.000007 |
| optimizing           | 0.000003 |
| statistics           | 0.000008 |
| preparing            | 0.000007 |
| executing            | 0.000001 |
| Sending data         | 0.000048 |
| end                  | 0.000002 |
| query end            | 0.000003 |
| closing tables       | 0.000005 |
| freeing items        | 0.000010 |
| cleaning up          | 0.000008 |
+----------------------+----------+
15 rows in set, 1 warning (0.00 sec)

#type
# \G更容易查看，由列表改为信息展示
	--all：显示所有性能信息
	show profile all for query n\G;
	--block io：显示块io操作的次数
	show  profile block io for query n
	--context switches：显示上下文切换次数，被动和主动
	show profile context switches for query n
	--cpu：显示用户cpu时间、系统cpu时间
	show profile cpu for query n
	--IPC：显示发送和接受的消息数量
	show profile ipc for query n
	--Memory：暂未实现
	--page faults：显示页错误数量
	show profile page faults for query n
	--source：显示源码中的函数名称与位置
	show profile source for query n
	--swaps：显示swap的次数
	show profile swaps for query n


```

        

## 2.使用performance schema来更加容易的监控mysql

==mysql 官网   -》 MySQL Performance Schema==

The MySQL Performance Schema is a feature for monitoring MySQL Server execution at a low level. 

==注意一个悖论：因为Schema是监控程序，长时间运行，会占用系统资源，但是不能因为占用系统资源，所以不开启Schema==

另外，Mysql 5.7默认开启了Schema；

```sql
show databases；#会多出：performance schema数据库
use performance_schema;
show tables;  #包含了87多张表，详细学习查看官网

SHOW VARIABLES LIKE 'performance_schema';  #通过配置文件修改；/etc/my.cnf
```

==详细文档:《MYSQL performance schema详解》==


面试、工作中用到的时候，知道怎么查

==问题：调优时，mysql语句、执行计划都没有问题，但是mysql就是慢，怎么解决？==-》就是这部分知识，mysql服务中执行了哪些线程？这些线程分别做什么事情？这些事情分别用了多长时间？占用了多少资源？

schema 监控这台mysql服务器的使用情况

==可以开发一套schema监控系统，有兴趣自己做，没兴趣算了==

利用schema可以做mysql监控

schema监控一般如果是公司使用，都是内部使用，不会开源

运维监控也不用schema，更多关注的是整个服务器的性能



## 3.使用show processlist查看连接的线程个数，来观察是否有大量线程处于不正常的状态或者其他不正常的特征、

官网  -》 SQL Statements  -》 Database Administration Statements   -》SHOW Statements  -》SHOW PROCESSLIST Statement


```
id表示session id
user表示操作的用户
host表示操作的主机
db表示操作的数据库
command表示当前状态
	sleep：线程正在等待客户端发送新的请求**
	query：线程正在执行查询或正在将结果发送给客户端
	locked：在mysql的服务层，该线程正在等待表锁**
	analyzing and statistics：线程正在收集存储引擎的统计信息，并生成查询的执行计划
	Copying to tmp table：线程正在执行查询，并且将其结果集都复制到一个临时表中
	sorting result：线程正在对结果集进行排序
	sending data：线程可能在多个状态之间传送数据，或者在生成结果集或者向客户端返回数据
info表示详细的sql语句
time表示相应命令执行时间
state表示命令执行状态
```



**这个在优化时，优化比较少**


---


==面试题：市场上主流的数据库线程池有哪些？有什么区别【加分项】==

现在市面上的数据库连接池：DBCP(很少使用)   C3P0（老项目）  druid（德鲁伊 性能很强）【性能监控信息-github中文文档，面试时可以帮到】

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


---

# Schema与数据类型优化（面试不会问到，DBA会问到）

==书籍《高效能mysql》==

## 数据类型的优化

### 更小的通常更好
  
应该尽量使用可以正确存储数据的最小数据类型，==更小的数据类型通常更快，因为它们占用更少的磁盘、内存和CPU缓存，并且处理时需要的CPU周期更少==，但是要确保没有低估需要存储的值的范围，如果无法确认哪个数据类型，就选择你认为不会超过范围的最小类型

案例：

设计两张表，设计不同的数据类型，查看表的容量

```sql
create table psn(id int(10),name varchar(10));
create table psn2(id bigint(10),name varchar(10));
# mysql安装目录：data/db1中两张表的大小一样
```

```shell
mysql> create table test_myisam(id int(10),name varchar(10) ) engine=myisam;
mysql> create table test_innodb(id int(10),name varchar(10) ) engine=innodb;

find / -name  mysql 
# /var/lib/mysql #mysql数据文件地址

cd /var/lib/mysql/test/

db.opt           test_innodb.ibd  test_int.ibd     test_myisam.MYD  test_varchar.frm
test_innodb.frm  test_int.frm     test_myisam.frm  test_myisam.MYI  test_varchar.ibd

```

.frm 表结构  
.ibd  存储的数据文件，代表存储引擎为InnoDB  
.MYD【数据文件】或.MYI【索引文件】代表存储引擎为MyASM  

**运行结果：不同的数据类型，文件大小是一样的，但是空间占用是不一样的**


```
import java.sql.Connection;
import java.sql.DriverManager;
import java.sql.PreparedStatement;

public class Test {
    public static void main(String[] args) throws Exception{
        Class.forName("com.mysql.jdbc.Driver");
        Connection conn = DriverManager.getConnection("jdbc:mysql://localhost:3306/db1","root","123456");
        PreparedStatement pstmt = conn.prepareStatement("insert into psn2 values(?,?)");
        for (int i = 0; i < 20000; i++) {
            pstmt.setInt(1,i);
            pstmt.setString(2,i+"");
            pstmt.addBatch();
        }
        pstmt.executeBatch();
        conn.close();
    }
}

```

### 简单更好

==简单数据类型的操作通常需要更少的CPU周期==，例如，

1、**整型比字符操作代价更低**，因为==字符集和校对规则==是字符比较比整型比较更复杂，

2、 **使用mysql自建类型而不是字符串来存储日期和时间**

3、**用整型存储IP地址**

案例：创建两张相同的表，改变日期的数据类型，查看SQL语句执行的速度


### 尽量避免null

如果查询中包含可为NULL的列，对mysql来说很难优化，因为可为null的列使得索引、索引统计和值比较都更加复杂，**坦白来说，通常情况下null的列改为not null带来的性能提升比较小**，所有没有必要将所有的表的schema进行修改，**但是应该尽量避免设计成可为null的列**

### 实际细则

#### 整数类型

可以使用的几种整数类型**TINYINT，SMALLINT，MEDIUMINT，INT，BIGINT分别使用8，16，24，32，64位存储空间**

尽量使用满足需求的最小数据类型

#### 字符和字符串类型

1、char长度固定，即每条数据占用等长字节空间；**最大长度是255个字符**，**适合用在身份证号、手机号等定长字符串**

2、varchar可变程度，可以设置最大长度；**最大空间是65535个字节**，适合用在长度可变的属性

3、text不设置长度，当不知道属性的最大长度时，适合用text

==按照查询速度：char>varchar>text==


---

##### 	varchar根据实际内容长度保存数据

1. 使用最小的符合需求的长度。
2. ==varchar(n) n小于等于255使用额外一个字节保存长度，n>255使用额外两个字节保存长度==。
3. **varchar(5)与varchar(255)保存同样的内容，硬盘存储空间相同，但内存空间占用不同，是指定的大小**
4. ==varchar在mysql5.6之前变更长度，或者从255一下变更到255以上时时，都会导致锁表==。

**应用场景**

1. 存储长度波动较大的数据，如：文章，有的会很短有的会很长
2. 字符串很少更新的场景，每次更新后都会重算并使用额外存储空间保存长度
3. 适合保存多字节字符，如：汉字，特殊字符等

---

==问题：文件实际大小与占用空间有什么区别？==

    磁盘有page,4Kb, 使用4Kb是为了提高数据读取效率， 4K对齐；
    一个文件大小为4.23Kb,会占用两个Page,所以占用空间为8Kb

操作系统中有：局部性原理，磁盘预读

**磁盘预读**：虽然程序只需要一个字节的内容，但是操作系统会将这1字节所在的page的数据都读取，认为接下来程序可能会用到

==InnoDB的存储引擎，每次读取16Kb的数据，提高磁盘和内存的IO效率==

4K对索引也有作用：3层B+树，意味着读3个磁盘块，12Kb的数据可以支撑百万级别的数据量，可以快速检索数据

---

##### 	char固定长度的字符串

1. 最大长度：255
2. ==会自动删除末尾的空格 【这是规定，不是bug】==
3. ==检索效率、写效率 会比varchar高，以空间换时间==

**应用场景**

1. 存储长度波动不大的数据，如：md5摘要
2. **存储短字符串、经常更新的字符串**
 
---

#### BLOB和TEXT类型

**Text和Blob，一般不用，如果过长可以存放在文件中，将文件路径保存在数据表中**

MySQL 把每个 BLOB 和 TEXT 值当作一个独立的对象处理。

两者都是为了存储很大数据而设计的字符串类型，**分别采用二进制和字符方式存储**。


---

==面试题：int长度改短会有什么影响？==

==面试题：面试题：mysql中字段定义为int(1)，可以存储100000吗？==

    没有影响；跟底层存储字节数有关，int的默认为32位 定义的长度只是规范，
    
```sql
mysql> create table test_int(id int(1));
Query OK, 0 rows affected (0.01 sec)

mysql> insert into test_int values(100000);
Query OK, 1 row affected (0.00 sec)
```

==面试题：varchar 长度改小会有什么影响？==

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

---

#### datetime和timestamp
##### 	datetime

**占用8个字节**

==与时区无关，数据库底层时区配置，对datetime无效==

**可保存到毫秒**

可保存时间范围大

不要使用字符串存储日期类型，占用空间大，损失日期类型函数的便捷性

##### 	timestamp

**占用4个字节**

==时间范围：1970-01-01到2038-01-19==

**精确到秒**

**采用整形存储**

==依赖数据库设置的时区==

自动更新timestamp列的值

==问题：知道timestamp的时间范围吗？为什么是这个范围？==

    时间范围：1970-01-01到2038-01-19
    
    原因：4个字节，32位，能表示的值为4294967296，42亿多，从格林尼治时间1970-01-01开始，加上42亿毫秒，就是32038-01-19


##### 	date

占用的字节数比使用字符串、datetime、int存储要少，使用date类型**只需要3个字节**

使用date类型还可以利用**日期时间函数**进行日期之间的计算

date类型用于保存==1000-01-01到9999-12-31==之间的日期


---


#### 使用枚举代替字符串类型

 ==枚举== -》数据字典

有时可以使用枚举类代替常用的字符串类型，mysql存储枚举类型会非常紧凑，会根据列表值的数据压缩到一个或两个字节中，mysql在内部会将每个值在列表中的位置保存为整数，并且在表的.frm文件中保存“数字-字符串”映射关系的查找表

```sql

create table enum_test(e enum('fish','apple','dog') not null);
insert into enum_test(e) values('fish'),('dog'),('apple');
select e+0 from enum_test;
select * from enum_test;

select * from enum_test order by e; # 排序跟插入的顺序无关，跟表字段的定义顺序有关
# 结果为：fish apple dog

select * from enum_test where e=1; //下标从0开始
select * from enum_test where e='fish';

alter table enum_test modify column (e enum('fish','apple','dog','cat'));

```

#### 特殊类型数据

**人们经常使用varchar(15)来存储ip地址**，然而，它的本质是32位无符号整数不是字符串，可以使用**INET_ATON()和INET_NTOA**函数在这两种表示方法之间转换


```sql
-- 1.IP地址转整型  【整型：可读性不好，但是存储空间更小】
select INET_ATON('192.168.85.111')
select INET_NTOA('3232257391')
select INET_ATON('255.255.255.257')  #无法转换

```

#### ==存储过程现在不建议使用==
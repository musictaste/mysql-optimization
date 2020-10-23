[TOC]


# 服务器参数设置

## cache

### 问题：mysql中哪些地方用到缓存？ -》 查询缓存

查询缓存【8.0版本已经剔除了】、join、索引、排序


---

**key_buffer_size**  索引缓存区的大小（只对myisam表起作用）
    
    机器中配置为：8M
    
query cache查询缓存 【公司中使用比较少】

经常变化的数据不放查询缓存，因为命中率低

	query_cache_size ： 查询缓存的大小，未来版本被删除
		show status like '%Qcache%';查看缓存的相关属性
		
		Qcache_free_blocks：缓存中相邻内存块的个数，如果值比较大，那么查询缓存中碎片比较多
		Qcache_free_memory：查询缓存中剩余的内存大小
		Qcache_hits：表示有多少此命中缓存
		Qcache_inserts：表示多少次未命中而插入
		Qcache_lowmen_prunes：多少条query因为内存不足而被移除cache
		Qcache_queries_in_cache：当前cache中缓存的query数量
		Qcache_total_blocks：当前cache中block的数量
	
	query_cache_limit  超出此大小的查询将不被缓存
	
	query_cache_min_res_unit  缓存块最小大小
	
	==query_cache_type  缓存类型，决定缓存什么样的查询
		0表示禁用
		1表示将缓存所有结果，除非sql语句中使用sql_no_cache禁用查询缓存
		2表示只缓存select语句中通过sql_cache指定需要缓存的查询

**sort_buffer_size** 每个需要排序的线程分派该大小的缓冲区

    show variables like '%sort_buffer_size%';
    可以看到有 innodb_sort_buffer_size 、  myisam_sort_buffer_size 、 sort_buffer_size
    问题：如果设置了innodb_sort_buffer_size、sort_buffer_size，innodb取哪个的配置
    如果设置了缓冲区大小，取精度更小的设置，也就是innodb_sort_buffer_size

**max_allowed_packet=32M** 限制server接受的数据包大小

**join_buffer_size=2M** 表示关联缓存的大小【join的第三种方式：block- ？？？】

**thread_cache_size** 【mysql自带的线程池】

服务器线程缓存，这个值表示可以重新利用 保存 在缓存中的线程数量，当断开连接时，那么客户端的线程将被放到缓存中以响应下一个客户而不是销毁，如果线程重新被请求，那么请求将从缓存中读取，如果缓存中是空的或者是新的请求，这个线程将被重新请求，那么这个线程将被重新创建，如果有很多新的线程，增加这个值即可

    Threads_cached：代表当前此时此刻线程缓存中有多少空闲线程
    Threads_connected：代表当前已建立连接的数量
    Threads_created：代表最近一次服务启动，已创建现成的数量，如果该值比较大，那么服务器会一直再创建线程
    Threads_running：代表当前激活的线程数
    
主从复制的时候，主机生成binlog,IO thread将log拉过来，拉过来变成终极日志；还有一个sql thread用终级日志把数据同步

## INNODB

工作中更多用到的存储引擎为INNODB

innodb_buffer_pool_size= 该参数指定大小的内存来**缓冲数据和索引**，==最大可以设置为物理内存的80%==

    hbase中的读写缓存，最大也是物理内存的80%
		
**innodb_flush_log_at_trx_commit** 主要控制innodb将log buffer中的数据写入日志文件并flush磁盘的时间点，值分别为0，1，2

    undo log 和redo log 将数据从log buffer经过os buffer写到磁盘
    1：数据比较安全；2：效率最高，但是可能会丢失1秒的数据

innodb_thread_concurrency **设置innodb线程的并发数，默认为0表示不受限制**,==如果要设置建议跟服务器的cpu核心数一致或者是cpu核心数的两倍==


innodb_log_buffer_size 此参数确定日志文件所用的内存大小，以M为单位


innodb_log_file_size 此参数确定数据日志文件的大小，以M为单位


innodb_log_files_in_group=2 以循环方式将日志文件写到多个文件中【redo log循环写，默认写到2个文件】

顺序读、随机读【索引】
    
    INnodb 使用B+树，非叶子节点不存放数据；叶子节点存放数据；叶子结点之间有前后指针，随机索引，顺序索引

read_buffer_size mysql读入缓冲区大小，对表进行顺序扫描的请求将分配到一个读入缓冲区

read_rnd_buffer_size mysql随机读的缓冲区大小

innodb_file_per_table 此参数确定为每张表分配一个新的文件【很重要】


    mysql中有 表空间
    什么是表空间？mysql的innodb存储引擎，创建一张表的话，应该能看到.idb的文件，但是神奇的发现：如果没有对mysql做修改的话是看不到.db文件的；只有设置了属性，才能看到.db文件【设置为on，每张表有一个单独的数据文件；如果设置为off，那么只有.ifm文件】
    
    如果设置为off，数据文件在哪？ -》/var/lib/mysql中的ibdata1文件就是表空间
    
    set global innodb_file_per_table=off;  //注意：global
    create table test(id int);  //看不到.ibd文件

### 问题：为什么要跟cpu的核心数一致？ -》 并发执行

### 问题：为什么web容器选nginx ，不选httpd？ -》IO模型不一样；NIO，BIO -》并发执行

    
# 服务器参数学习的目标：知道有哪些属性值，这些属性值能启动什么样的控制

## 问题：如果不小心把表删除了，是否可以恢复? -> binlog

如果日志文件没有删除，那么可以恢复；可以根据binlog日志进行恢复,所以一定要记得开启binlog（log_bin）

undo log是循环写，所以无法恢复
    

---


log_bin：**指定二进制日志文件名称**，用于记录对数据造成更改的所有查询语句【**在做主从复制时很重要，默认不开启，需要手动开启;并且强烈建议开启**；==开启会影响性能，但是保证数据的安全性==】

    bin_log的等级=row  
    log-bin=master-bin
    biglog-format=ROW

**binlog_do_db** ：指定将更新记录到二进制日志的数据库，**其他所有没有显式指定的数据库更新将忽略，不记录在日志中**

问题：binlog-do-db指定的数据库越多越好吗？

> 不是，binlog-do-db指定的数据库越多，说明要生成的binlog日志文件越大，恢复越麻烦；虽然设置上是好的，但是会增加吞吐量；所以要做合理选择：哪些数据库的数据重要就存biglog，如果不重要、或能及时恢复，就没必要做biglog日志了
    

**binlog_ignore_db** ： 指定不将更新记录到二进制日志的数据库

==**注意：新项目上线，会设置binlog-do-db或binlog-ignore-db，并且这个设置很重要；一般有经验的DBA或运维会做这件事**==


    
# Mysql锁

《mysql的锁机制.md》

Mysql中的锁跟存储引擎相关，

InnoDB：共享锁、排它锁；可以锁表，也可以锁行；锁的对象：索引【锁的列是索引列的话，是行锁；如果不是索引会从行锁退化为表锁】

MyISAM：共享读锁、独占写锁；只能锁表；

InnoDB的意向共享锁、意向排它锁 -》自己查？ 

![708](48383A8A667F4340B8095AD2906B68C2)

自增锁：针对自增列自增长【auto_increment】的**一个特殊的表级别锁**

    如果一个插入语句id=20，事务回滚了，下一个插入的id=21，,20永久丢失了
    

间隙锁：

**间隙锁锁定的区域**

根据检索条件向左寻找最靠近检索条件的记录值A，作为左区间，向右寻找最靠近检索条件的记录值B作为右区间，即锁定的间隙为（A，B）。
图一中，where number=5的话，那么间隙锁的区间范围为（4,11）；

**间隙锁的目的是为了防止幻读，其主要通过两个方面实现这个目的：**

（1）防止间隙内有新数据被插入

（2）防止已存在的数据，更新成间隙内的数据（例如防止numer=3的记录通过update变成number=5）

**innodb自动使用间隙锁的条件**：

（1）必须在RR(repeatable read)级别下

（2）检索条件必须有索引（没有索引的话，mysql会全表扫描，那样会锁定整张表所有的记录，包括不存在的记录，此时其他事务不能修改不能删除不能添加）
    
    
# 演示死锁  -> mysql发生死锁，会自动释放锁  -》 死锁时后给谁加锁，就释放哪个

![709](C5C0A305CE2543729762C97064C603EC)


# 主从复制

==MySQL 默认采用异步复制方式==

mysql的主从复制，**通过binlog进行数据同步**，**binlog传输存在的问题；延迟**

**延迟问题如何解决？ -》 mysql5.7版本，引入了并行复制技术MTS**

MTS 搞明白

## mysql复制原理

​		（1）**master服务器将数据的改变记录二进制binlog日志**，当master上的数据发生改变时，则将其改变写入二进制日志中；		

​		（2）**slave服务器会在一定时间间隔内对master二进制日志进行探测其是否发生改变**，==如果发生改变，则开始一个I/OThread请求master二进制事件==

​		（3）**同时主节点为每个I/O线程启动一个dump线程，用于向其发送二进制事件，并保存至从节点本地的中继日志中**，==从节点将启动SQL线程从中继日志中读取二进制日志，在本地重放，使得其数据和主节点的保持一致，最后I/OThread和SQLThread将进入睡眠状态，等待下一次被唤醒。==

## 也就是说：

- 从库会生成两个线程,一个I/O线程,一个SQL线程;
- I/O线程会去请求主库的binlog,并将得到的binlog写到本地的relay-log(中继日志)文件中;
- 主库会生成一个log dump线程,用来给从库I/O线程传binlog;
- SQL线程,会读取relay log文件中的日志,并解析成sql语句逐一执行;



# 读写分离

mysql proxy  -》 《mysql读写分离.md》  -> 阿里巴巴  变形虫Amoeba  -》 mycat

官网学习 -》 

## 为什么要用Amoeba

目前要实现mysql的主从读写分离，主要有以下几种方案：

1、  通过程序实现，网上很多现成的代码，比较复杂，如果添加从服务器要更改多台服务器的代码。

2、  ==通过mysql-proxy来实现==，**由于mysql-proxy的主从读写分离是通过lua脚本来实现，目前lua的脚本的开发跟不上节奏，而写没有完美的现成的脚本，因此导致用于生产环境的话风险比较大**，据网上很多人说mysql-proxy的性能不高。

3、  自己开发接口实现，这种方案门槛高，开发成本高，不是一般的小公司能承担得起。

4、  利用阿里巴巴的开源项目Amoeba来实现，具有负载均衡、高可用性、sql过滤、读写分离、可路由相关的query到目标数据库，并且安装配置非常简单。国产的开源软件，应该支持，目前正在使用，不发表太多结论，一切等测试完再发表结论吧，哈哈！
[TOC]

# 通过索引进行优化

## 优化小细节

### union all,in,or都能够使用索引，但是推荐使用in

union 和union all 推荐使用union all；因为union有distinct的操作，而去重是很麻烦的，浪费性能

通过执行计划看不出来，通过show profile可以看出实际执行时间的差距

```
explain select * from actor where actor_id = 1 union all select * from actor where actor_id = 2;
# 有两条sql语句
# union 和union all 推荐使用union all；因为union有distinct的操作，而去重是很麻烦的，浪费性能


explain select * from actor where actor_id in (1,2);
# oracle in限制1000个值？？？自己测一下，不确定
# 应该in 里面是子查询，子查询跟直接设置值是不一样的

explain select * from actor where actor_id = 1 or actor_id =2;
 
运行结果：图片401.png 
in 和 or 执行计划信息是一样的
```
![401](E0A29660A52A4371A47CD58C90562D1F)

```
set profiling =1
select * from actor where actor_id = 1 union all select * from actor where actor_id = 2;
select * from actor where actor_id in (1,2);
select * from actor where actor_id = 1 or actor_id =2;
show profiles;

运行结果：图片402.png
in的执行时间相比or 更短
```
![402](D5BB17D716B04D4FB197FFAEBE67840A)
    
### 补充exists:
    
exists 必须是子查询

    select * from emp where deptno=20 or deptno = 30;
    // 期望的结果
    
    select * from emp where exists(select deptno from dept where deptno=20 or deptno=30)
    // 结果是整张表
    // 这个语句相当于两个for循环；
    // 外层循环：select * from emp where
    // 内层循环：select deptno from dept where deptno=20 or deptno=30
    // exists 只要里面的子查询有结果，就相当于true，不需要关心结果是多少，那么外层的for循环就一定会执行，
    
    // 那么应该如何修改？
    // 加别名，不可行
    select * from emp e where exists(select deptno from dept d where deptno=20 or deptno=30 and d.deptno=e.deptno)
    // 执行结果还是整张表
    // 原因：因为and 的优先级 大于or的优先级； d.deptno=e.deptno会先执行，or deptno=30后执行
    // 不是很理解？？
    
    
    // exsits 正确结果
    select * from emp e where exists(select deptno from dept d where (deptno=20 or deptno=30) and d.deptno=e.deptno)
    // 期望结果
    
    exists总结：外层循环控制内层循环，还有很有用处的，但是sql语句会比较复杂
    
    2.外面的子查询，不能查询exists子查询中的字段????
    select *,d.deptno from emp e where exists(select deptno from dept d where (deptno=20 or deptno=30) and d.deptno=e.deptno)  ??能执行成功吗
    
### 范围列可以用到索引

范围条件是：<、<=、>、>=、between

范围列可以用到索引，但是范围列后面的列无法用到索引，索引最多用于一个范围列

### 强制类型转换会全表扫描

    create table user(id int,name varchar(10),phone varchar(11));
    alter table user add index idx_1(phone);
    
    explain select * from user where phone=13800001234;
    # 没有使用索引
    
    explain select * from user where phone='13800001234';
    // 使用了索引
    
    执行结果：图片403.png
![403](2873B3B4A69A4D1C99748931B66C9705)
### 更新十分频繁，数据区分度不高的字段上不宜建立索引

更新会变更B+树，更新频繁的字段建议索引会大大降低数据库性能

类似于性别这类区分不大的属性，建立索引是没有意义的，不能有效的过滤数据，

==一般区分度在80%以上的时候就可以建立索引，区分度可以使用count(distinct(列名))/count(*)来计算==

### 创建索引的列，不允许为null，可能会得到不符合预期的结果

值匹配时，null != null ,应该是null is (not) null

### 当需要进行表连接的时候，最好不要超过三张表，因为需要join的字段，数据类型必须一致

#### 补充join的多种方式 

官网-》Nested-Loop join algorithms

 Nested-Loop algorithms 嵌套循环算法
 
 自己补《Join的多种方式》
 
![404](61522FEF681F4CD4B275455E7E9A788B)
![405](6D22A02534904A9983C2E3B8E1922A26)
![406](BC14A99895364C4FA70D56E817D9D635)

---


问题：table A join table B ，一定是A表joinB表吗？
 
 不一定；也有可能是B join A；优化器优化时决定
 
 可以强制指定 A join B：A constact(???) join B 
 
 所以一般为：A.外建 join B.主键
 
 最好是：小表 join  大表 ，主要是执行次数的数量； 小表放内存，大表放磁盘，这样可以是小表快速加载；  小表 join 大表 类似大数据中的 mapjoin (Hive)
 
 
 join buffer默认为256K；如果数据过大，buffer放不下，那么无法采用Block Nested-loop join,所以可以调大 (join_buffer_size=262144), 调整的大小要考虑内存的大小
 
    show variables like '%join_buffer%'
 
 问题：为什么要引入 join-on的形式？-》 join指定的连接条件， on指定了筛选条件
 
 

#### 问题：MYSQL 中 LEFT JOIN ON 后的AND 和WHERE
 
 问题：select * from t1 join t2 on t1.id=t2.id and t1.id="XXX" 和 select * from t1 join t2 on t1.id=t2.id where t1.id="XXX" 这两条sql语句有什么区别？
 
先是针对左右连接,这里与inner join区分

在使用left join时,on and 和on where会有区别

1. on and的条件是在连接生成临时表时使用的条件,以左表为基准 ,不管on中的条件真否,都会返回左表中的记录
2. on where条件是在临时表生成好后,再对临时表过滤。此时 和left join有区别(返回左表全部记录),条件不为真就全部过滤掉,on后的条件来生成左右表关联的临时表,where后的条件是生成临时表后对临时表过滤

on and是进行韦恩运算时,连接时就做的动作,where是全部连接完后，再根据条件过滤



---

**LEFT JOIN ON WHERE：在临时表生成后，再对临时表的数据进行过滤，再返回左表。**

**LEFT JOIN ON AND：在临时表生成的过程时，ON中的条件不管是否为真，都将返回左表。**

**用and这种情况也比较多，比方说就是想统计满足条件的左边表的记录，就算为空，也要列出来，那这种方法就很好使用，否则还要在连接一次用户表。**

**对于inner join,on and和on where结果一致
在使用inner join时，不管是对左表还是右表进行筛选，on and和on where都会对生成的临时表进行过滤**


#### 再扩展：下面语句的执行结果

select * from emp e inner join dept d on 1!=1  ->空

select * from emp e left join dept d on 1!=1  ->左表数据

select * from emp e right join dept d on 1!=1  ->右表数据


---

#### 官网-》Nested join Otimization  join案例  自己阅读并整理


#### 扩展：大表 join 大表

大表 join 大表，**本质上没有太好的匹配方式**，**唯一的方式：采用分区运算，也（就是分区表）**，**手动限定范围 或者 把语句拆分为N条结果来执行**，分而治之的思想，类似hive

**大表 join 大表，还要排序**；排序是order by; **hive中有sort by、distbut by（？？？）、classer by,这些方式mysql不一定支持**；但是可以采用n个小表 join，然后排序；然后再合并到一起，**这样的操作复杂度比较高，需要很多临时表，IO量会增加**

### 数据类型必须一致

Hive中sql语句，A join B join C,如果join的连接键相同的话，会转化成一个mapreduce来执行；如果连接键不同，会转化成多个mapreduce来执行

mysql中也一样，如果数据类型一样的，会直接进行关联，统一进行处理；如果数据类型不一样的，是按多个连接条件进行join的,性能低


### 能使用limit的时候尽量使用limit

==limit的作用：是限制输出，不是分页；说专业一点==

limit到限制的数量后，不会再往下进行搜索



### 单表索引建议控制在5个以内

**表中列为varchar类型，长度为10个字节，如果列值为空字符串，会不会占用空间？会**

**如果值为1个字节，那么存储空间是几？ 10个字节**

### 单索引字段数不允许超过5个（组合索引）

字段过多浪费存储空间

### 创建索引的时候应该避免以下错误概念

==索引越多越好==

==过早优化，在不了解系统的情况下进行优化==

## 面试MySql调优应该怎么说？

不要上来就把思维导图中的内容说一遍，应该是：在公司的时候，查询比较慢，通过执行计划查询发现type是all 、index或range，我调整以后type变成const，怎么调的具体再说一下


思维导图中的内容是在写sql时要注意的地方，只有出问题的时候，才会做实质性的优化

## 索引监控（不是很重要的点）

官网-》 搜索Handler_read_first  -> 

    show status like 'Handler_read%';
    
    Handler_read_first：读取索引第一个条目的次数
        // 访问根节点的次数
    
    Handler_read_key：通过index获取数据的次数
    Handler_read_last：读取索引最后一个条目的次数
    Handler_read_next：通过索引读取下一条数据的次数
    Handler_read_prev：通过索引读取上一条数据的次数
    Handler_read_rnd：从固定位置读取数据的次数
    Handler_read_rnd_next：从数据节点读取下一条数据的次数

**这些值在索引优化中非常有用：知道索引的有没有作用、效率**

==重点关注：Handler_read_key、Handler_read_rnd_next，如果这两个值比较大，说明索引生效了；如果值很小，说明没有用到索引，或索引建的有问题==

**注意：这是全部的索引信息，不是单个的索引**

索引优化：通过一个命令（？？？），可以将分裂也进行合并【这个命令很少用，所以在迁移数据时要将索引关闭】

==索引整理的时候会将现有的数据锁住，所以不建议这样的操作；新迁移的数据可以做索引整理==

## 简单案例

《索引优化分析案例》

自己看一下


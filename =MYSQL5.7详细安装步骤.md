

# MYSQL5.7详细安装步骤：

### 0、更换yum源

1、打开 mirrors.aliyun.com，选择centos的系统，点击帮助

2、执行命令：yum install wget -y

3、改变某些文件的名称

```
mv /etc/yum.repos.d/CentOS-Base.repo /etc/yum.repos.d/CentOS-Base.repo.backup
```

4、执行更换yum源的命令

```
wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.aliyun.com/repo/Centos-6.repo
```

5、更新本地缓存

yum clean all

yum makecache

### 1、查看系统中是否自带安装mysql

```shell
yum list installed | grep mysql
```

![1570541665646](https://raw.githubusercontent.com/musictaste/mysql-tuning/master/typora-user-images/1570541665646.png)
![1570541665646](3522A801DF03402D9BA4B196460EA9EF)

### 2、删除系统自带的mysql及其依赖（防止冲突）

```shell
yum -y remove mysql-libs.x86_64
```

![1570541838485](https://raw.githubusercontent.com/musictaste/mysql-tuning/master/typora-user-images/1570541838485.png)
![1570541838485](3704BAF5465945238E1D1F27301BCEBB)

### 3、安装wget命令

```
yum install wget -y 
```

![1570541946471](https://raw.githubusercontent.com/musictaste/mysql-tuning/master/typora-user-images/1570541946471.png)
![1570541946471](A2C573BF20D64BD0B212E9267D53DE9B)

### 4、给CentOS添加rpm源，并且选择较新的源

```
wget dev.mysql.com/get/mysql-community-release-el6-5.noarch.rpm
```

![1570542045332](https://raw.githubusercontent.com/musictaste/mysql-tuning/master/typora-user-images/1570542045332.png)
![1570542045332](BC34F445C7814D67A601469817B5F98A)

### 5、安装下载好的rpm文件

```
 yum install mysql-community-release-el6-5.noarch.rpm -y
```

![1570542254949](https://raw.githubusercontent.com/musictaste/mysql-tuning/master/typora-user-images/1570542254949.png)
![1570542254949](150102014B9A4B4783F422FE06FFFD7C)


### 6、安装成功之后，会在/etc/yum.repos.d/文件夹下增加两个文件

![1570542341604](https://raw.githubusercontent.com/musictaste/mysql-tuning/master/typora-user-images/1570542341604.png)
![1570542341604](BB8940A5412048E690024524FE115ABA)

### 7、修改mysql-community.repo文件

原文件：

![1570542415955](https://raw.githubusercontent.com/musictaste/mysql-tuning/master/typora-user-images/1570542415955.png)
![1570542415955](EC99C88A7F16437289BD606B9B7080B7)

修改之后：

![1570542471948](https://raw.githubusercontent.com/musictaste/mysql-tuning/master/typora-user-images/1570542471948.png)
![1570542471948](143055F04F3247D1B70BFEF6EC72946B)

### 8、使用yum安装mysql

```
yum install mysql-community-server -y
```

![1570542688796](https://raw.githubusercontent.com/musictaste/mysql-tuning/master/typora-user-images/1570542688796.png)
![1570542688796](A9195BFAF889424FB3DF266F0098C91B)

### 9、启动mysql服务并设置开机启动

```shell
#启动之前需要生成临时密码，需要用到证书，可能证书过期，需要进行更新操作
yum update -y
#启动mysql服务
service mysqld start
#设置mysql开机启动
chkconfig mysqld on
```

### 10、获取mysql的临时密码

```shell
grep "password" /var/log/mysqld.log
```

![1570604493708](https://raw.githubusercontent.com/musictaste/mysql-tuning/master/typora-user-images/1570604493708.png)
![1570604493708](2CC8CD10B7AA4E7DB356EEF783AEBB14)

### 11、使用临时密码登录

```shell
mysql -uroot -p
#输入密码

0rO&t-D<:4H_
```

### 12、修改密码

```sql
set global validate_password_policy=0;
set global validate_password_length=1;
ALTER USER 'root'@'localhost' IDENTIFIED BY '123456';

=================说明==========================
--这些参数是要安装好validate_password 插件后才能通过show variables like 'validate_password%';看到。
--001、validate_password_policy 这个参数用于控制validate_password的验证策略 0-->low  1-->MEDIUM  2-->strong。
--002、validate_password_length密码长度的最小值(这个值最小要是4)。
--003、validate_password_number_count 密码中数字的最小个数。
--004、validate_password_mixed_case_count大小写的最小个数。
--005、validate_password_special_char_count 特殊字符的最小个数。
--006、validate_password_dictionary_file 字典文件

mysql> show variables like 'validate_password%';
+--------------------------------------+--------+
| Variable_name                        | Value  |
+--------------------------------------+--------+
| validate_password_check_user_name    | OFF    |
| validate_password_dictionary_file    |        |
| validate_password_length             | 8      |
| validate_password_mixed_case_count   | 1      |
| validate_password_number_count       | 1      |
| validate_password_policy             | MEDIUM |
| validate_password_special_char_count | 1      |
+--------------------------------------+--------+
7 rows in set (0.00 sec)


```

### 13、修改远程访问权限

```sql
grant all privileges on *.* to 'root'@'%' identified by '123456' with grant option; 
-- GRANT：赋权命令
-- ALL PRIVILEGES：当前用户的所有权限
-- ON：介词
-- *.*：当前用户对所有数据库和表的相应操作权限
-- TO：介词
-- ‘root’@’%’：权限赋给root用户，所有ip都能连接
-- IDENTIFIED BY ‘123456’：连接时输入密码，密码为123456
-- WITH GRANT OPTION：允许级联赋权

flush privileges;
```

### 14、设置字符集为utf-8

```shell
/etc/my.cnf文件

#在[mysqld]部分添加：
character-set-server=utf8
#在文件末尾新增[client]段，并在[client]段添加：
default-character-set=utf8


#重启服务器
service mysqld restart

```


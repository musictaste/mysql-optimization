# Linux下mysql的彻底卸载

### 1、查看mysql的安装情况

```
rpm -qa | grep -i mysql
```

![1570605325400](https://raw.githubusercontent.com/musictaste/mysql-tuning/master/typora-user-images/1570605325400.png)
![1570605325400](0DF34242585A42B8BCDE4D275893F15C)



### 2、删除上图安装的软件

```
rpm -ev mysql-community-libs-5.7.27-1.el6.x86_64 --nodeps
rpm -ev mysql-community-common-5.7.27-1.el6.x86_64
....

```

### 3、都删除成功之后，查找相关的mysql的文件

```
find / -name mysql
```

![1570605553095](https://raw.githubusercontent.com/musictaste/mysql-tuning/master/typora-user-images/1570605553095.png)
![1570605553095](C37B99201D2848D0971A2FF945392F16)

### 4、删除全部文件

```
rm -rf /var/lib/mysql
rm -rf /var/lib/mysql/mysql
rm -rf /etc/logrotate.d/mysql
rm -rf /usr/share/mysql
rm -rf /usr/bin/mysql
rm -rf /usr/lib64/mysql
```

### 5、再次执行命令

```shell
rpm -qa | grep -i mysql
#如果没有显式则表示卸载完成
```


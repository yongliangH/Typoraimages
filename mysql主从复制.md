##### MySQL主从复制

###### 工作原理



1、主服务器执行完命令后，写入事务日志

2、从服务器使用SQL I/O线程，复制主服务器日志中的信息

3、从服务器将日志信息写入relay log 文件中，进行命令整理

4、从服务器使用SQL线程进行调用命令，重新执行

5、从服务器执行命令后，更新日志。



###### 主从复制的作用

实现mysql数据库的数据同步:

1.全表复制：将主服务器的表同步复制到从服务器，当表中的数据发生变化时，同步整张表。能够保证数据的同步性，最大可能保持两个表的数据相同。缺点：效率很低。

2.行记录复制：验证主服务器表中的数据记录和从服务器表中数据记录的区别，只复制不同的行记录。传输效率高，但是本地验证时间太长。

3.事务日志同步复制：事务？对数据库提交的，能够对数据库或数据修改的命令。从服务器复制主服务器的事务日志条目，重新执行命令，实现数据同步。



##### MySQL主从复制部署

###### 1、拓扑图



![](C:\Users\26098\AppData\Roaming\Typora\typora-user-images\image-20210827101448303.png)

###### 2、基本配置

| 服务器名      | 安装软件                    | IP地址       |
| ------------- | --------------------------- | ------------ |
| mariadb主     | mariadb<br />mariadb-server | 192.168.10.1 |
| mariadb从     | mariadb<br />mariadb-server | 192.168.10.2 |
| mysql从       | 开源MySQL（5.5）<br />cmake | 192.168.10.3 |
| windows-mysql | mysql5.5（大概为5.2）       | 192.168.10.4 |

###### 3、mariadb主安装配置

​	1、yum安装mariadb

```shell
yum -y install mariadb  mariadb-server
```

​	2、启动服务，设置密码。

```shell
systemctl restart mariadb
mysqladmin -u root password 123.com
```

​	3、编辑my.cnf配置文件

```shell
vim /etc/my.cnf
在mysqld下面添加：
server-id=1					#服务器id号，所有mysql服务器不能相同
log-bin=master-bin			#启用日志文件
log-slave-updates=true		#允许从服务器的日志文件更新
```

![image-20210827155625006](C:\Users\26098\AppData\Roaming\Typora\typora-user-images\image-20210827155625006.png)

​	4、启动服务

```shell
systemctl restart mariadb
然后到/var/lib/mysql/ 会出现主服务器的日志文件。master-bin.000001
```

​	5、授权mysql从服务器

```mariadb
grant replication slave on *.* to 'slave'@'192.168.10.%' identified by '123.com';
```

​	6、查看主服务器信息

```mysql
show master status;
#会显示日志的节点。
#记住初始节点，帮助添加新mysql数据
```

###### 4、mariadb从配置

​	1、第一二步一模一样（略）

​	2、编辑my.cnf 配置文件

```shell
vim /etc/my.cnf
添加：
server-id=2								#服务器id号
relay-log=relay-log-bin					#定义relay-log的位置和名称
relay-log-index=slave-relay-bin.index	#同relay-log，定义relay-log的位置和名称，一般和relay-log在同一目录
```

<img src="C:\Users\26098\AppData\Roaming\Typora\typora-user-images\image-20210827161000509.png" alt="image-20210827161000509"  />

​	3、连接主服务器，进行日志复制

```mariadb
change master to master_host='192.168.10.1',master_user='slave',master_password='123.com',master_log_file='master-bin.000001',master_log_pos=538;

#最后节点数，新数据库查看节点at
```

​	4、启动从服务器进程

```mariadb
MariaDB [(none)]> slave start;
#slave包括IO进程和sql进程
```

​	5、停止从服务器。进行节点复制时，一定要停止从服务器的复制进程

```mariadb
MariaDB [(none)]> slave stop;
```



###### 4、开源mysql从配置

​	1、安装开源mysql数据库

```
yum -y install perl perl-devel perl-DBD* ncurses-devel			#安装依赖关系
tar -zxvf cmake-2.8.6.tar.gz -C /usr/src/						#解压cmake，用户编辑安装mysql
cd /usr/src/cmake-2.8.6/
./configure && gmake && gmake install							#编译安装
cd
tar -zxvf mysql-5.6.36.tar.gz -C /usr/src/						#解压mysql
cd /usr/src/mysql-5.6.36/
cmake -DCMAKE_INSTALL_PREFIX=/usr/local/mysql -DSYSCONFDIR=/etc -	DDEFAULT_CHARSET=utf8 -DDEFAULT_COLLATION=utf8_general_ci -DWITH_EXTRA_CHARSETS=all
#编译
make && make install											#安装
ln  -s  /usr/local/mysql/bin/*  /usr/local/bin					#命令优化路径
useradd   -M   -s   /sbin/nologin   mysql						#创建mysql服务用户
chown   -R   mysql:mysql  /usr/local/mysql/						#修改mysql目录的属主属组
cp  /usr/src/mysql-5.6.36/support-files/my-default.cnf /etc/my.cnf 
#拷贝mysql配置文件到/etc/my.cnf
cp   /usr/src/mysql-5.6.36/support-files/mysql.server /etc/rc.d/init.d/mysqld
#拷贝mysql脚本文件到/etc/下
chmod   a+x   /etc/rc.d/init.d/mysqld    	#给脚本文件设置可执行权限
chkconfig  --add  mysqld					#将mysqld服务添加到服务管理器
chkconfig  mysqld  on						#设置开机自启动
/usr/local/mysql/scripts/mysql_install_db  --user=mysql  --group=mysql --basedir=/usr/local/mysql  --datadir=/usr/local/mysql/data
#初始化数据库（方便以后下载过来的数据库也是这个用户与这个目录）
systemctl   restart   mysqld
```

​	2、编辑my.cnf配置文件

```shell
vim /etc/my.cnf   （vim /usr/local/mysql/my.cnf ,我是在这个配置文件下添加的，一样）
添加：
server-id=3
relay-log=relay-log-bin
relay-log-index=slave-relay-bin.index
```

![image-20210827170119905](C:\Users\26098\AppData\Roaming\Typora\typora-user-images\image-20210827170119905.png)

​	3、连接主数据库，并同步数据

```mysql
change master to master_host='192.168.10.1',master_user='slave',master_password='123.com',master_log_file='master-bin.000001',master_log_pos=538;
```

​	4、启动slave进程（包括IO进程与sql进行，可单独启动）

```mysql
mysql> start slave;
#如重新读入日志，如停止slave进程
mysql> stop slave;
#然后在同步数据
```

​	5、查看从服务器信息

```mysql
mysql> show slave status\G      #以视图形式查看
```

![image-20210827170957471](C:\Users\26098\AppData\Roaming\Typora\typora-user-images\image-20210827170957471.png)

​	两个线程显示yes，表示同步数据成功

###### 5、查看从数据库是否成功同步主数据库

```mysql
show databases;
use 表名；
select * from 表名；
```

###### 6、查看日志命令

```shell
mysqlbinlog 日志文件
```



##### MySQL主从复制运维

在生产环境中，我们常常会给mysql主从结构添加新的从服务器或者修复从数据库。下面如何添加。

###### windows版的mysql举例，其他系统相同

​	1、添加新的mysql服务器

​	windows安装步骤，选择第三项（完全安装complete），一直下一步即可

​	2、编辑mysql的配置文件

```
默认路径为：C:\Program Files (x86)\MySQL\MySQL Server 5.5\my.ini
修改这个文件：记事本打开
添加：mysqld下
server-id=4
relay-log=relay-log-bin
relay-log-index=slave-relay-bin.index

#这一步相当于linux编辑my.cnf文件那一步
保存保存保存保存保存再退出
```

![image-20210827172842313](C:\Users\26098\AppData\Roaming\Typora\typora-user-images\image-20210827172842313.png)

​	3、在服务管理器中重启服务

```
运行框输入services.msc
找到mysql服务，重新启动
```

<img src="C:\Users\26098\AppData\Roaming\Typora\typora-user-images\image-20210827173007339.png" alt="image-20210827173007339" style="zoom: 80%;" />

<img src="C:\Users\26098\AppData\Roaming\Typora\typora-user-images\image-20210827173056699.png" alt="image-20210827173056699" style="zoom:50%;" />

​	4、查看主服务器上的日志节点

```shell
cd /var/lib/mysql/
mysqlbinlog 日志文件名
#找到创建数据库的节点at
```

​	5、从服务器连接数据库进行同步数据命令

```mysql
change master to master_host='192.168.10.1',master_user='slave',master_password='123.com',master_log_file='master-bin.000001',master_log_pos=495;
```

![image-20210827173245402](C:\Users\26098\AppData\Roaming\Typora\typora-user-images\image-20210827173245402.png)

​	6、启动salve进程

```mysql
start slave；
```




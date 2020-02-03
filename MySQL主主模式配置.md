# MySQL主主模式配置

## 安装MySQL5.6.38

操作系统

```bash
cat /etc/redhat-release

CentOS Linux release 7.6.1810 (Core)
```

### 创建系统帐号

```bash
groupadd -r mysql
useradd -r -g mysql -s /sbin/nologin mysql
```

### 关闭SELinux

```bash
setenforce 0

sed -i 's#SELINUX=enforcing#SELINUX=disabled#g' /etc/selinux/config
```

### 安装依赖包

```bash
yum install ncurses-devel libaio-devel -y
yum install cmake gcc gcc-c++ make autoconf -y
```

### 下载MySQL5.6.38安装包

```bash
wget https://cdn.mysql.com/archives/mysql-5.6/mysql-5.6.38.tar.gz
```

### 解压缩

```bash
tar xf mysql-5.6.38.tar.gz 
cd mysql-5.6.38/
```

### 进行编译安装

```bash
 cmake . -DCMAKE_INSTALL_PREFIX=/usr/local/mysql-5.6.38 \         # 指定安装目录
 -DMYSQL_DATADIR=/usr/local/mysql-5.6.38/data \
-DMYSQL_UNIX_ADDR=/usr/local/mysql-5.6.38/mysql.sock \
-DDEFAULT_CHARSET=utf8 \
-DDEFAULT_COLLATION=utf8_general_ci \
-DWITH_EXTRA_CHARSETS=all \
-DWITH_INNOBASE_STORAGE_ENGINE=1 \
-DWITH_FEDERATED_STORAGE_ENGINE=1 \
-DWITH_BLACKHOLE_STORAGE_ENGINE=1 \
-DWITHOUT_EXAMPLE_STORAGE_ENGINE=1 \
-DWITH_SSL=bundled \
-DWITH_ZLIB=bundled \
-DENABLED_LOCAL_INFILE=1 \
-DWITH_EMBEDDED_SERVER=1 \
-DENABLE_DOWNLOADS=1 \
-DWITH_DEBUG=0 -DSYSCONFDIR=/etc
```

```bash
make && make install 
```

### 做软链接并给MySQL目录授权

```bash
ln -s /usr/local/mysql-5.6.38/ /usr/local/mysql
chown -R mysql.mysql /usr/local/mysql
```

### 初始化数据目录

```bash
/usr/local/mysql/scripts/mysql_install_db --basedir=/usr/local/mysql --datadir=/usr/local/mysql/data -user=mysql
```

### 拷贝启动服务的脚本

```bash
cp /usr/local/mysql/support-files/mysql.server /etc/init.d/mysqld
chmod 700 /etc/init.d/mysqld
```

### 修改环境变量

```bash
echo 'PATH=/usr/local/mysql/bin:$PATH' >>/etc/profile
source /etc/profile
```

### 修改配置文件

```bash
# vim /etc/my.cnf
[mysqld]
server_id=1
port=3306
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data
socket=/usr/local/mysql/mysql.sock 
log_bin=/usr/local/mysql/mysql-bin
log_error=/var/log/mysql.log
character-set-server=utf8

[client]socket=/usr/local/mysql/mysql.sock
```

### 启动数据库

```bash
/etc/init.d/mysqld start
```

### 设置数据库密码并清空MySQL不安全帐号

```bash
mysqladmin -u root password 123456
mysql -u root -p 123456

# 清除不安全的用户，先查询用户名为空和没有密码的
> select user,password,host from mysql.user;
> drop user root@'127.0.0.1';
> drop user ''@'localhost';
```

## 克隆虚拟机

A机ip：192.168.127.161

B机ip：192.168.127.162

## 配置mysql主主模式

### A机数据库

#### 开启binlog

```bash
[mysqld]
server_id=1
port=3306
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data
socket=/usr/local/mysql/mysql.sock
log_bin=/usr/local/mysql/mysql-bin
log_error=/var/log/mysql.log
character-set-server=utf8

[client]
socket=/usr/local/mysql/mysql.sock
```

#### 重启mysql,创建同于同步的用户账号

```bash
systemctl restart mysql
```

**登陆数据库**

```
mysql -u root -p
```

 **创建用户并授权：用户：test，密码：123456，ip：B主机的ip**

```bash
create user 'test'@'192.168.127.162' identified by '123456';
```

 分配权限

```bash
grant replication slave on *.* to 'test'@'192.168.127.161';
flush privileges;
```

 锁表，禁止写入，当前窗口不能退出，这时候开启另一个终端继续操作

```
flush table with read lock;
```

####  新终端数据库操作，查看master状态，记录二进制文件名和位置

```mysql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |      120 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

#### 将当前数据库导出，如果两个数据库不一致，手动调整

```bash
mysqldump -u root -p --all-databases > /home/alldb.sql
```

#### 解锁查看binlog日志位置，如果没变化证明锁定成功。从库将从这个binlog日志开始恢复

```mysql
unlock tables;

show master status;

show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000003 |      120 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

### B机数据库操作

#### 导入数据

```bash
scp -r 192.168.127.161:/home/alldb.sql /home/
```

```bash
mysql -u root -p < /home/alldb.sql
```

#### 修改配置文件

```mysql
[mysqld]
server_id=2
port=3306
basedir=/usr/local/mysql
datadir=/usr/local/mysql/data
socket=/usr/local/mysql/mysql.sock
log_bin=/usr/local/mysql/mysql-bin
log_error=/var/log/mysql.log
character-set-server=utf8

[client]
socket=/usr/local/mysql/mysql.sock
```

#### 连接数据库，需要A机服务器主机名，登陆凭证，二进制文件名称和位置

```bash
mysql -u root -p
```

```mysql
change master to master_host='192.168.127.161',
    -> master_user='test',
    -> master_password='novell',
    -> master_log_file=' mysql-bin.000003',
    -> master_log_pos=120;
```

#### 开启slave，查看slave状态

```mysql
start slave;

show slave status\G
```

#### 配置作为A数据库的主库

 **创建用户并授权：用户：test，密码：123456，ip：A主机的ip**

```
create user 'test'@'192.168.127.161' identified by '123456';
```

 **分配权限**

```bash
grant replication slave on *.* to 'test'@'192.168.127.161';
flush privileges;
```

这次不用锁表了，因为B库在同步A库数据的时候，已经一致了。

**新窗口操作，查看master状态，记录二进制文件名和位置**

```mysql
mysql> show master status;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000006 |      120 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
```

### A机数据库操作

#### 需要B服务器主机名，登陆凭证，二进制文件名和位置

change master to master_host='192.168.127.162',
    -> master_user='test',
    -> master_password='novell',
    -> master_log_file='  mysql-bin.000006',
    -> master_log_pos=120;

#### 开启查看slave状态

```mysql
start slave；
show slave status\G
```



# 问题记录

**问题1：/etc/init.d/mysqld start启动mysql失败**

**解决步骤**

查看mysql日志

```bash
more /var/log/mysql.log
```

```bash
/usr/local/mysql/bin/mysqld: File '/usr/local/mysql/mysql-bin.index' not found (Errcode: 13 - Permission denied)
2020-02-02 19:47:38 24299 [ERROR] Aborting

2020-02-02 19:47:38 24299 [Note] Binlog end
2020-02-02 19:47:38 24299 [Note] /usr/local/mysql/bin/mysqld: Shutdown complete
```

推测是mysql的文件夹权限错误导致mysql用户可不读取

查看/usr/local/mysql的文件夹权限

```bash
ll /usr/local
```

发现mysql-5.6.38文件夹的权限是root导致mysql用户不可读取

```bash
lrwxrwxrwx.  1 mysql mysql  24 Feb  2 19:32 mysql -> /usr/local/mysql-5.6.38/
drwxr-xr-x. 13 root  root  205 Feb  2 19:32 mysql-5.6.38
```

修改mysql-5.6.38文件夹权限

```bash
chmod  755 mysql-5.6.38/ 
chown -R mysql.mysql mysql-5.6.38/
```

启动mysql

```bash
[root@localhost local]# /etc/init.d/mysqld start
Starting MySQL. SUCCESS!
```

**问题2：由于B机由A机复制而来所以报错The slave I/O thread stops because master and slave have equal MySQL server UUIDs**

**解决步骤**

检查发现他们的auto.cnf中的server-uuid是一样的

```bash
cat /usr/local/mysql-5.6.38/data/auto.cnf
[auto]
server-uuid=04b6e4bc-45b2-11ea-aea9-000c29771c3d
```

停止从库的mysqld服务，删除他的auto.cnf文件，再启动数据库服务即可

```bash
mv  /usr/local/mysql-5.6.38/data/auto.cnf  /usr/local/mysql-5.6.38/data/auto.cnf.bak
systemctl start mysqld.service
```

此时再去查看从库auto.cnf，已自动生成新的server-uuid

```bash
cat /usr/local/mysql-5.6.38/data/auto.cnf
[auto]
server-uuid=7d4463e4-45bc-11ea-aeed-000c29e4e6ed
```

再查看从库状态，已正常

```mysql
mysql> show slave status\G
```

**问题3：Got fatal error 1236 from master when reading data from binary log**

在source那边，执行：

```bash
flush logs;
show master status;
```

记下File, Position。

在target端，执行：

```bash
CHANGE MASTER TO MASTER_LOG_FILE='testdbbinlog.000008',MASTER_LOG_POS=107;
slave start;
show slave status \G
```

一切正常。
**问题4：Can't connect to local MySQL server through socket '/usr/local/mysql/mysql.sock' (XXX)**
```bash
/etc/init.d/mysqld start 
```
**问题5： Operation CREATE USER failed for 'test'@'xx.xx.xx.xx'**
```bash
查看是不是存在这个用户
select user from user;
发现没有这个用户。
记得上次有删除过这个用户。可能没有刷新权限
flush privileges;
之后还是不行报错ERROR 1396 (HY000): Operation CREATE USER failed for ‘test’@’%’
没办法再删除一次：
drop user ‘test’@’%’;
flush privileges;
之后create user ‘test’@’%’ identified by ‘test’;
```



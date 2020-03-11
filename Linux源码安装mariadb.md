# Linux源码安装mariadb

## 下载mariadb源码包

```bash
wget http://mirror.biznetgio.com/mariadb//mariadb-10.4.12/source/mariadb-10.4.12.tar.gz
```

## 安装cmake

```bash
yum -y install readline-devel
yum -y install zlib-devel
yum -y install openssl-devel
yum -y install libaio-devel
yum -y install cmake
```

## 安装MariaDB

### 提前预定安装目录和数据目录

**提前预定安装目录/usr/local/mysql和数据目录/data/mysql并且赋予mysql用户权限**

```bash
groupadd -r mysql
useradd -g mysql -s /sbin/nologin mysql
mkdir /usr/local/mysql
mkdir -p /data/mysql
chown -R mysql:mysql /data/mysql/
```

### 解压tar.gz文件

```bash
tar -xvzf mariadb-10.4.12.tar.gz
cd mariadb-10.4.12
```

### 执行编译安装

```bash
cmake . -DCMAKE_INSTALL_PREFIX=/usr/local/mysql -DMYSQL_DATADIR=/data/mysql -DSYSCONFDIR=/etc -DWITHOUT_TOKUDB=1 -DWITH_INNOBASE_STORAGE_ENGINE=1 -DWITH_ARCHIVE_STPRAGE_ENGINE=1 -DWITH_BLACKHOLE_STORAGE_ENGINE=1 -DWIYH_READLINE=1 -DWIYH_SSL=system -DVITH_ZLIB=system -DWITH_LOBWRAP=0 -DMYSQL_UNIX_ADDR=/tmp/mysql.sock -DDEFAULT_CHARSET=utf8 -DDEFAULT_COLLATION=utf8_general_ci
```

```bash
make && make install
```

## 配置

```bash
cd /usr/local/mysql/
```

```bash
chown -R mysql:mysql .
scripts/mysql_install_db --datadir=/data/mysql --user=mysql
chown -R root .
cp support-files/mysql.server /etc/init.d/mysqld
```

### 将mysqld添加至系统服务

```bash
chkconfig --add mysqld   # 添加至系统服务
chkconfig mysqld on    # 设置开机自启动
```

### 建立日志目录

```bash
mkdir /var/log/mariadb 
```

### 启动mariadb

```bash
systemctl start mysqld
```

### 软链接本地socket

```bash
ln -s /var/lib/mysql/mysql.sock /tmp/mysql.sock
```

### 进入MariaDB交互式界面

```bash
./bin/mysql
```

　**为了方便可以将mysql目录添加到环境变量**
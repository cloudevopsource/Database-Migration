# OL7.7安装Oracle19c记录

## 环境

1. 系统：Oracle Linux 7.7 
2. cpu:    4 core   
3. 硬盘：100g
4. swap：8G
5. 内存： 8G

## 安装前准备

- 选择服务器角色

  **a server with GUI勾选Develop tools和传统库**

- 下载Oracle19c安装包

```bash
mkdir -p /stage/oracle

cd  stage

scp -r  172.16.0.8:/stage/* .
```

- 修改hosts文件

```bash
vim /etc/hosts

172.16.102.131 oracle19c.localdomain oracle19c
```

- 重启

```bash
reboot
```

- oracle安装先决条件

```bash
yum install -y oracle-database-preinstall-19c
```

- 创建用户组

```bash
groupadd -g 54321 oinstall  
groupadd -g 54322 dba  
groupadd -g 54323 oper  
groupadd -g 54324 backupdba  
groupadd -g 54325 dgdba  
groupadd -g 54326 kmdba  
groupadd -g 54327 asmdba  
groupadd -g 54328 asmoper  
groupadd -g 54329 asmadmin  
groupadd -g 54330 racdba 
useradd -u 54321 -g oinstall -G dba,oper,asmdba,backupdba,dgdba,kmdba,racdba oracle
useradd -u 54322 -g oinstall -G asmadmin,asmdba,asmoper,dba,racdba grid

groupmod -g 54321 oinstall  
groupmod -g 54322 dba  
groupmod -g 54323 oper  
groupmod -g 54324 backupdba  
groupmod -g 54325 dgdba  
groupmod -g 54326 kmdba  
groupmod -g 54327 asmdba  
groupmod -g 54328 asmoper  
groupmod -g 54329 asmadmin  
groupmod -g 54330 racdba
usermod -u 54321 -g oinstall -G dba,oper,asmdba,backupdba,dgdba,kmdba,racdba oracle
usermod -u 54322 -g oinstall -G asmadmin,asmdb,asmoper,dba,racdba grid
```

- 更新系统

```bash
yum -y update
```

- 安装必要软件包

```bash
yum install -y bc    
yum install -y binutils
yum install -y compat-libcap1
yum install -y compat-libstdc++-33
#yum install -y dtrace-modules
#yum install -y dtrace-modules-headers
#yum install -y dtrace-modules-provider-headers
yum install -y dtrace-utils
yum install -y elfutils-libelf
yum install -y elfutils-libelf-devel
yum install -y fontconfig-devel
yum install -y glibc
yum install -y glibc-devel
yum install -y ksh
yum install -y libaio
yum install -y libaio-devel
yum install -y libdtrace-ctf-devel
yum install -y libXrender
yum install -y libXrender-devel
yum install -y libX11
yum install -y libXau
yum install -y libXi
yum install -y libXtst
yum install -y libgcc
yum install -y librdmacm-devel
yum install -y libstdc++
yum install -y libstdc++-devel
yum install -y libxcb
yum install -y make
yum install -y net-tools # Clusterware
yum install -y nfs-utils # ACFS
yum install -y python # ACFS
yum install -y python-configshell # ACFS
yum install -y python-rtslib # ACFS
yum install -y python-six # ACFS
yum install -y targetcli # ACFS
yum install -y smartmontools
yum install -y sysstat

# Added by me.
yum install -y unixODBC
```

- 附加设置

```
passwd oracle

vim /etc/selinux/config

SELINUX=disable
```

- 配置生效

```
setenforce 0
```

- 关闭防火墙

```bash
 systemctl stop firewalld
 systemctl disable firewalld
```

- 创建Oracle安装目录

```bash
#grid用户
mkdir -p  /u01/app/19.3.0/grid
mkdir -p /u01/app/grid
chown -R grid:oinstall /u01
chmod -R 755 /u01
#oracle用户
mkdir -p  /u02/app/oracle/product/19.3.0/db_1
chown -R oracle:oinstall /u02/
chmod -R 775 /u02/
```

- 接受连接机器

```bash
export DISPLAY=:0.0
xhost +
```

- 创建脚本目录

```
mkdir /home/oracle/scripts
```

- grid用户环境变量

```bash
export ORACLE_SID=+ASM
export ORACLE_BASE=/u01/app/grid
export ORACLE_HOME=/u01/app/19.3.0/grid
export PATH=/usr/sbin:/usr/local/bin:$PATH
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
```

- oracle用户环境变量

```bash
export ORACLE_BASE=/u02/app/oracle
export ORACLE_HOME=$ORACLE_BASE/product/19.3.0/db_1
export ORACLE_UNQNAME=cdb1
export ORACLE_SID=cdb1
export PDB_NAME=pdb1
export PATH=/usr/sbin:/usr/local/bin:$PATH
export PATH=$ORACLE_HOME/bin:$PATH
export LD_LIBRARY_PATH=$ORACLE_HOME/lib:/lib:/usr/lib
export CLASSPATH=$ORACLE_HOME/jlib:$ORACLE_HOME/rdbms/jlib
```
## 设置UDEV（方法一：挂载新硬盘）

- UDEV 绑定硬盘

```bash
# /usr/lib/udev/scsi_id -g -u /dev/sdb 

1ATA_VBOX_HARDDISK_VBc15c17be-e6e41f2d
```

- 创建 asmasmdevices 规则文件

```bash
# vi /etc/udev/rules.d/99-oracle-asmdevices.rules
 
KERNEL=="sd*", SUBSYSTEM=="block", PROGRAM=="/usr/lib/udev/scsi_id --whitelisted --replace-whitespace --device=/dev/$name", RESULT=="1ATA_VBOX_HARDDISK_VBc15c17be-e6e41f2d",SYMLINK+="asm-diskb", OWNER="grid", GROUP="asmadmin", MODE="0660"
```

- 编写udev规则文件 

```bash
KERNEL=="sda",SUBSYSTEM=="block", PROGRAM=="/lib/udev/scsi_id -g -u -d /dev/sda" ,RESULT=="0QEMU_QEMU_HARDDISK_252469da-64a5-4445-8", SYMLINK+="asmdisk1", OWNER="grid", GROUP="asmadmin", MODE="0660"
KERNEL=="sdb",SUBSYSTEM=="block", PROGRAM=="/lib/udev/scsi_id -g -u -d /dev/sdb" ,RESULT=="0QEMU_QEMU_HARDDISK_fee8bf61-7061-40bd-9", SYMLINK+="asmdisk2", OWNER="grid", GROUP="asmadmin", MODE="0660"
KERNEL=="sdc",SUBSYSTEM=="block", PROGRAM=="/lib/udev/scsi_id -g -u -d /dev/sdc" ,RESULT=="0QEMU_QEMU_HARDDISK_b6390142-36db-4d46-9", SYMLINK+="asmdisk3", OWNER="grid", GROUP="asmadmin", MODE="0660"

```

- 重新加载分区

```bash
# /usr/sbin/partprobe /dev/sdb 

/sbin/udevadm trigger

#partprobe：
#将磁盘分区表变化信息通知内核，请求操作系统重新加载分区表。
 -d 不更新内核
 -s 显示磁盘分区汇总信息
 -h 显示帮助信息
 -v 显示版本信息
```

- 加载udev 配置文件

```bash
/sbin/udevadm trigger --type=devices --action=change
udevadm control --reload-rules
ls -l /dev/as*
udevadm test /sys/block/sd* #测试
systemctl status systemd-udevd.service #状态
systemctl enable systemd-udevd.service #开机
```

## 设置UDEV（方法二：挂载新硬盘并分区）

- 进行分区

```bash
fdisk /dev/sdd
```

- 查看硬盘信息

```
udevadm info -a -p /sys/block/sdd/sdd1
udevadm info -a -p /sys/block/sdd/sdd2
```

我们使用ATTR{start}=="2048"  ATTR{size}=="19997953"来唯一标识这个sdd1设备 

- 创建 asmasmdevices 规则文件

```bash
#vi /etc/udev/rules.d/99-oracle-asmdevices.rules

KERNEL=="sdd1",SUBSYSTEM=="block", ATTR{start}=="2048", ATTR{size}=="19997953", SYMLINK+="asmdisk4", OWNER="grid", GROUP="asmadmin", MODE="0660"
KERNEL=="sdd2", SUBSYSTEM=="block", ATTR{start}=="20000768", ATTR{size}=="21942272", SYMLINK+="asmdisk5", OWNER="grid", GROUP="asmadmin", MODE="0660"
```

- 重新加载分区

```bash
#/usr/sbin/partprobe /dev/sdd1 
#/usr/sbin/partprobe /dev/sdd2 
/sbin/udevadm trigger

#partprobe：
#将磁盘分区表变化信息通知内核，请求操作系统重新加载分区表。
 -d 不更新内核
 -s 显示磁盘分区汇总信息
 -h 显示帮助信息
 -v 显示版本信息
```

- 加载udev 配置文件

```bash
/sbin/udevadm trigger --type=devices --action=change
udevadm control --reload-rules
ls -l /dev/as*
udevadm test /sys/block/sd* #测试
systemctl status systemd-udevd.service #状态
systemctl enable systemd-udevd.service #开机
```

##  grid安装

```bash
export DISPLAY=:0.0
cd /u01/app/19.3.0/grid
- 解压
unzip -oq V982068-01.zip
./gridSetup.sh
```

![](https://gitee.com//yileng146/Pic/raw/master/img/20200506222838.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200506222902.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200506222928.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200506222952.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200506223026.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200506223329.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200506224103.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200506224214.png)

- ASM安装

在grid下运行./gridSetup.sh再次运行安装

![](https://gitee.com//yileng146/Pic/raw/master/img/20200506224555.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507111155.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507111249.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507111437.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507111628.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507111711.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507112106.png)

- asmca创建闪回区

运行asmca创建FRA组

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507125312.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507125721.png)



## OracleDB安装

- 切换oracle用户并添加环境变量

```bash
DISPLAY=:0.0; export DISPLAY
```

- 运行安装程序

```bash
cd /u01/app/oracle/product/19.3.0/db_1/
- 解压
unzip -oq V982063-01.zip
./runInstaller
```

![](https://gitee.com//yileng146/Pic/raw/master/img/20200506220643.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507112617.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507112656.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507112722.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507112755.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507112836.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507112904.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507113237.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507113256.png)

## DBCA

- 运行dbca

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507113458.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507113540.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507113659.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507113744.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507113837.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507130421.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507130526.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507130604.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507130744.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507130955.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507131154.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507131249.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507131344.png)

![](https://gitee.com//yileng146/Pic/raw/master/img/20200507135601.png)
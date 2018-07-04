# 离线安装Cloudera Manager 5和CDH5(5.1.4) 指南

## 系统环境

* 操作系统：CentOS Linux release 7.2.1511
* Cloudera Manager：5.1.4
* CDH: 5.1.4

## 安装包

### Cloudera Manager 5

* cm5.14.0-centos7.tar.gz

### CDH5

* CDH-5.14.0-1.cdh5.14.0.p0.24-el7.parcel
* CDH-5.14.0-1.cdh5.14.0.p0.24-el7.parcel.sha1
* manifest.json

## 系统环境搭建（以下操作均用root用户操作）

### 网络配置(所有节点)

修改 hostname：

    hostnamectl set-hostname mater

或者使用：

    vi /etc/hostname

接着修改 ip 与主机名的对应关系：

    vi /etc/hosts

如下：

    172.16.101.69   master
    172.16.101.70   slave01
    172.16.101.71   slave02
    172.16.101.72   slave03

### 打通SSH，设置ssh无密码登陆（所有节点）

在主节点上执行：

    ssh-keygen -t rsa

一路回车，生成无密码的密钥对。

~~将公钥添加到认证文件中：~~

~~cat ~/.ssh/id_rsa.pub >> ~/.ssh/authorized_keys~~

~~并设置authorized_keys的访问权限:~~

~~chmod 600 ~/.ssh/authorized_keys~~

~~scp文件到所有datenode节点：~~

~~scp ~/.ssh/authorized_keys root@slave:~/.ssh/~~

公钥授权到其他机器：

    ssh-copy-id root@slave01~03

### 关闭防火墙和SELinux（所有节点）

**需要在所有的节点上执行，因为涉及到的端口太多了，临时关闭防火墙是为了安装起来更方便，**
**安装完毕后可以根据需要设置防火墙策略，保证集群安全。**

#### 关闭防火墙：

关闭防火墙：

    systemctl stop firewalld.service

禁止开机启动:

    systemctl disable firewalld.service

查看防火墙状态:

    firewall-cmd --state

#### 关闭SELinux:

查看SELinux状态:

    /usr/sbin/sestatus  -v

返回

    SELinux status:                 enabled

说明SELinux开启

修改

    vim /etc/selinux/config

将

    SELINUX=enforcing

改为

    SELINUX=disabled （重启后永久生效）

### 配置NTP服务（所有节点）

**集群中所有主机必须保持时间同步，如果时间相差较大会引起各种问题。 具体思路如下：**
**master 节点作为 ntp 服务器与外界对时中心同步时间，随后对所有 datanode 节点提供时间同步服务。**
**所有 datanode 节点以 master 节点为基础同步时间。**

查看是否安装ntp

    rpm -aq | grep ntpd

安装：

    yum install ntp

有了就不用安装了，命令如下：

启动NTP服务：

    systemctl start ntpd.service

设置开机启动：

    systemctl enable ntpd.service

重启服务:

    systemctl restart ntpd.service

查看状态：

    systemctl status ntpd.service

#### 配置 ntp 服务器端（主节点）

打开 /etc/ntp.conf 配置文件

    server 101.201.72.121  # 中国国家授时中心
    server 133.100.11.8  #日本[福冈大学]
    server 3.cn.pool.ntp.org
    server 1.asia.pool.ntp.org
    server 3.asia.pool.ntp.org

    restrict 101.201.72.121 nomodify notrap noquery
    restrict 133.100.11.8 nomodify notrap noquery
    restrict 3.cn.pool.ntp.org nomodify notrap noquery
    restrict 1.asia.pool.ntp.org nomodify notrap noquery
    restrict 3.asia.pool.ntp.org nomodify notrap noquery

    server 127.127.1.0  # local clock
    fudge 127.127.1.0  stratum 10

修改完成后重启 ntpd 服务

检查是否成功，用 ntpstat 命令查看同步状态，出现以下状态代表启动成功：

    synchronised to NTP server (85.199.214.100) at stratum 2
       time correct to within 133 ms
       polling server every 64 s

查看 ntp 服务是否能够被同步：

    remote           refid      st t when poll reach   delay   offset  jitter
    ==============================================================================
    LOCAL(0)        .LOCL.          10 l   6h   64    0    0.000    0.000   0.000

显示为 “LOCAL”，表示成功。如果没有任何显示，客户端将无法同步。

#### 配置ntp客户端（所有datanode节点）

打开 /etc/ntp.conf 配置文件

    server 172.16.101.69
    restrict 172.16.101.69  nomodify notrap noquery

    #server 101.201.72.121  # 中国国家授时中心
    #server 133.100.11.8  #日本[福冈大学]
    #server 3.cn.pool.ntp.org
    #server 1.asia.pool.ntp.org
    #server 3.asia.pool.ntp.org

    #restrict 101.201.72.121 nomodify notrap noquery
    #restrict 133.100.11.8 nomodify notrap noquery
    #restrict 3.cn.pool.ntp.org nomodify notrap noquery
    #restrict 1.asia.pool.ntp.org nomodify notrap noquery
    #restrict 3.asia.pool.ntp.org nomodify notrap noquery

    server 127.127.1.0  # local clock
    fudge 127.127.1.0  stratum 10

修改完成后重启 ntpd 服务

使用 ntpdate 手动同步一下时间：

    ntpdate -u master

检查是否成功，用 ntpstat 命令查看同步状态，出现以下状态代表启动成功：

    synchronised to NTP server (172.16.101.69) at stratum 3
       time correct to within 170 ms
       polling server every 1024 s

### 安装Oracle的Java（所有节点）

**目前版本的 Linux 系统都自带有 OpenJDK，需要卸载，使用自己的jdk。**

使用 rpm -qa | grep java 查看位置

使用 rpm -e --nodeps 删除

或者使用

    rpm -qa | grep openjdk | xargs rpm -e --nodeps

删除系统自带 openjdk

将1.8的JDK传到 /usr/local/ 目录下,解压tar文件：

    tar -zxvf jdk-8u11-linux-x64.tar.gz

解压之后配置环境变量

    vi /etc/profile

按 Shift+g 到最后一行添加路径

    export JAVA_HOME=/usr/lib/jvm/jdk1.8.0_131
    export JRE_HOME=${JAVA_HOME}/jre
    export CLASSPATH=.:${JAVA_HOME}/lib:${JRE_HOME}/lib
    export PATH=${JAVA_HOME}/bin:$PATH

然后

    source /etc/profile

进行刷新，使立即生效

#### 拷贝 jdk 和环境变量到其他机器

    scp -r /usr/local/jdk1.8.0_151 root@slave01:/usr/local
    scp -r /etc/profile root@slave01:/etc/profile

每个机器都要再刷新一下环境变量：

    source /etc/profile

### 安装配置MySql（主节点）

先查看是否有 MySQL：

    rpm -qa | grep mysql
    rpm -qa | grep postfix
    rpm -qa | grep mariadb

如果有，执行 rpm -e --nodeps 或者 rpm -ev 进行删除

mysql 5.7 包的安装顺序为(RPM方式)：
mysql-community-common-5.7.9-1.el7.x86_64.rpm
mysql-community-libs-5.7.9-1.el7.x86_64.rpm
mysql-community-libs-compat-5.7.9-1.el7.x86_64.rpm
mysql-community-client-5.7.9-1.el7.x86_64.rpm
mysql-community-server-5.7.9-1.el7.x86_64.rpm

使用 rpm -ivh *.rpm 和 tar -xvf *.tar 命令进行安装

安装mysql之后，启动

    service mysqld start

取得root默认密码

    grep "password" /var/log/mysqld.log
    mysql -u root -p
    
安装之后需要先修改密码

修改密码规则为长度

    mysql> set global validate_password_policy=0;

修改密码长度为最小4位

    mysql> set global validate_password_length=1;

修改初始密码为 root

    mysql> ALTER USER USER() IDENTIFIED BY "root";

设置表名大小写不敏感

    vi etc/my.cnf
    
设置

    [mysqld]
    lower_case_table_names=1 //0为敏感

之后输入

    mysql -u root -pxxxx（xxxx为密码）

进入 MySQL 命令行

设置 MySQL 密码强度规则，允许密码为三位字母

    mysql> set global validate_password_policy=0;  
    Query OK, 0 rows affected (0.05 sec)  
       
    mysql> set global validate_password_mixed_case_count=0;  
    Query OK, 0 rows affected (0.00 sec)  
      
    mysql> set global validate_password_number_count=3;  
    Query OK, 0 rows affected (0.00 sec)  
      
    mysql> set global validate_password_special_char_count=0;  
    Query OK, 0 rows affected (0.00 sec)  
      
    mysql> set global validate_password_length=3;  
    Query OK, 0 rows affected (0.00 sec) 

## 安装 Cloudera Manager 5 和 CDH5

在 MySQL 中新建 scm 用户和数据库

    mysql> CREATE DATABASE SCM CHARACTER SET UTF8; 
    mysql> CREATE USER 'scm'@'%'IDENTIFIED BY 'scm';
    mysql> GRANT ALL PRIVILEGES ON *.* TO 'scm'@'%';
    mysql> FLUSH PRIVILEGES;
	
新建 hive 数据库

    mysql> create database hive DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
	
新建 activity monitor 数据库

    mysql> create database amon DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
	
设置 root 用户授权访问以上所有的数据库：

    mysql> grant all privileges on *.* to 'root'@'%' identified by 'root' with grant option;
    mysql> flush privileges;

### 安装 Cloudera Manager Server 和 Agent

主节点解压安装 cm5.14.0-centos7.tar.gz 到 /opt 文件夹

    tar xzvf cm5.14.0-centos7.tar.gz
    
将解压后的 cm-5.14.0 和 cloudera 目录放到 /opt 目录下

下载 MySQL 驱动 tar 包解压到任意目录，将 MySQL 驱动 mysql-connector-java-5.1.46-bin.jar 复制到 /opt/cm-5.14.0/share/cmf/lib 目录下

修改 /opt/cm-5.14.0/etc/cloudera-scm-agent/config.ini 中的 server_host 为主节点的主机名

    server_host=master

在主节点初始化CM5的数据库：

    /opt/cm-5.14.0/share/cmf/schema/scm_prepare_database.sh mysql cm -hlocalhost -uroot -proot --scm-host localhost scm scm scm

同步配置到其他节点

    scp -r /opt/cm-5.14.0 root@slave01:/opt/
    
在所有节点创建 cloudera-scm 用户

    useradd --system --home=/opt/cm-5.14.0/run/cloudera-scm-server/ --no-create-home --shell=/bin/false --comment "Cloudera SCM User" cloudera-scm

准备 Parcels，用以安装CDH5

将 CHD5 相关的 Parcel 包放到主节点的 /opt/cloudera/parcel-repo/ 目录中（parcel-repo 需要手动创建）。

相关的文件如下：

    CDH-5.14.0-1.cdh5.14.0.p0.24-el7.parcel
    CDH-5.14.0-1.cdh5.14.0.p0.24-el7.parcel.sha1
    manifest.json

最后将 CDH-5.14.0-1.cdh5.14.0.p0.24-el7.parcel.sha1，
重命名为 CDH-5.14.0-1.cdh5.14.0.p0.24-el7.parcel.sha，
这点必须注意，否则系统会重新下载 CDH-5.14.0-1.cdh5.14.0.p0.24-el7.parcel 文件

相关启动脚本

通过：

    /opt/cm-5.14.0/etc/init.d/cloudera-scm-server start 
    
启动服务端。

通过：

    tail -f /opt/cm-5.14.0/log/cloudera-scm-agent/cloudera-scm-server.log 
    
查看server日志。

通过：

    /opt/cm-5.14.0/etc/init.d/cloudera-scm-agent start 
    
启动Agent服务。

通过：

    tail -f /opt/cm-5.14.0/log/cloudera-scm-agent/cloudera-scm-agent.log 
    
查看agent日志。

我们启动的其实是个 service 脚本，需要停止服务将以上的start参数改为 stop 就可以了，重启是 restart。

Cloudera Manager Server 和 Agent 都启动以后，就可以进行CDH5的安装配置了。
这时可以通过浏览器访问主节点的 7180 端口测试一下了
（由于 CM Server 的启动需要花点时间，这里可能要等待一会才能访问），默认的用户名和密码均为 admin 。

### 安装配置 CDH5



### 测试

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
** master 节点作为 ntp 服务器与外界对时中心同步时间，随后对所有 datanode 节点提供时间同步服务。**
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

如果有，执行 rpm -e --nodeps 删除

## 安装 Cloudera Manager 5 和 CDH5

### 安装 Cloudera Manager Server 和 Agent

### 安装配置 CDH5

### 测试

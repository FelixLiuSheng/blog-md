---
title: heartbeat+drbd+mysql Replication实现服务稳定以及性能保证高可用方案
categories:
  - 分布式
  - 宕机处理
tags:
  - heartbeat
  - drbd
  - mysql Replication
  - 热备份
  - mysql
  - 宕机
date: 2016-05-07 22:56:37
---

## 对于heartbeat+drbd+mysql Replication的认识
### heartbeat
 对于heartbeat来说，也许很多人都较为陌生，因为一般的公司都不会去用，因为 这些公司对他们的程序要求不高，所以我们会经常看到，访问某一些网页的时候发现打不开。但对于大公司或者公司对自己的程序需要有一定的稳定性要求，并且对于服务器故障会有一定的急救措施， 当然，对于检测程序是否处于运行状态以及自动切换服务器的处理程序中，heartbeat将是一个很好地第三方程序。heartbeat主要是用在当一台服务器宕掉的时候，heartbeat很好地去自动切换到另一台服务器上继续完成相应的任务。可以在百度中搜索到heartbeat，介绍如下：heartbeat （Linux-HA）的工作原理：heartbeat最核心的包括两个部分，心跳监测部分和资源接管部分，心跳监测可以通过网络链路和串口进行，而且支持冗 余链路，它们之间相互发送报文来告诉对方自己当前的状态，如果在指定的时间内未收到对方发送的报文，那么就认为对方失效，这时需启动资源接管模块来接管运 行在对方主机上的资源或者服务。
###  drbd
  相对heartbeat而言，drbd并不会去启动什么程序，drbd的主要任务便是完成两台或者多台服务器间文件的实时同步功能，例如用户上传的文件，如果当前的服务器宕掉了后，难道该用户就不能看到对应的文件了么？这时可以通过drbd做文件实时同步，完成两台服务器之间的文件复制功能，这样，当一台服务器宕掉了，用heartbeat自动切换到另外一台服务器，然后切换为为宕掉的文件服务器，这时，用户同样可以访问他所上传的文件。
###  mysql replication
  很多人可能都对mysql并不陌生，但大部分人都只会简单的使用mysql的存储功能，并未对mysql做相应的处理，例如当我们存储数据库Mysql的服务器宕掉了，那么我们的额数据将无法被访问，对应的程序也将会无法进行正确运行，很多人也会做mysql集群，但如果我们对程序锁提供的接口所在的服务器宕掉了，那么还是无法正确的运行，所以我们将会采用mysdql reolication做数据库的双热备份，及主主备份机制，这样，当我们的数据库所在的一台服务器宕掉了，我们会采用heartbeat自动切换到备用的服务器上，由于数据库是采用的双热备份，所以备份数据库中的数据是实时更新宕掉的数据库的，而当宕掉的数据库完成修复后，heartbeat将会切换到原有的服务器中，同时mysql数据库也将会从备份数据库中生产的数据更新，这样就完美的解决的当服务器宕掉后的程序自动处理效果，而不会给用户体验带来任何的问题。
## 简单的双服务器程序自动切换搭建
  本例子中只是简单地完成两台服务器中的自动切换搭建，可以根据需要搭建一个集群并做相关的配置编号，只需要理解相应的原理。
  目前我有两台服务器，分别为：
  服务器1 eth0 192.168.8.20 eth1 10.10.10.20   wshuttle-test1
  服务器2 eth0 192.168.8.21 eth1 10.10.10.21   wshuttle-test2
  {% img  /images/blog/heartbeat-drbd-mysql-Replication/net-xuli-heartbeat.png 300 200  %}
  如上图，其中eth1为两个服务器之间的通讯局域网络，而eth0为两个服务器向外通讯的ip,但在最上面他们都被一个虚拟ip192.168.8.100所代替，及对外所开放的ip地址。
  其中wshuttle-test1和wshuttle-test2分别为两台服务器的hostname.
### 搭建heartbeat
#### 如上所示，首先设置两台服务器中的hostname，并关闭eth1网卡防火墙
#### 分别在两台服务器中，在/etc/hosts配置文件中添加如下 
          192.168.8.20 wshuttle-test1
          192.168.8.21 wshuttle-test2
#### 安装epel源，如果你也安装，跳过该步骤。
           命令：yum -y install epel-release
#### 安装heartbeat和nginx（用nginx做测试）
           命令：yum -y install heartbeat nginx
#### 在主机服务器1（wshuttle-test1）配置：
``` bash
cd /usr/share/doc/heartbeat-3.0.4/
cp  authkeys  ha.cf    haresources   /etc/ha.d/
cd /etc/ha.d
vi authkeys 
```
#打开下面两项：一共有三种认证方式供选择，第一种是CRC循环冗余校验，第二种是SHA1哈希算法，第三种是MD3哈希算法，其中他们的密码可以任意设置，但是两边密码必须保持一致。
``` bash
auth 1
1 crc
chmod 600 authkeys    #给认证文件授权为600
vi  haresources  #加入如下语句
wshuttle-test1 192.168.8.100/24/eth0:0 nginx   #设定虚拟IP和对应的接口，并且指定启动虚拟IP时启动NGINX服务，
#这里的NGINX服务必须是能够直接在/etc/init.d/目录下启动的服务。
vi  ha.cf   #改为如下内容：
debugfile /var/log/ha-debug   #设定debug文件目录
logfile /var/log/ha-log     #设定日志文件目录
logfacility     local0
keepalive 2            #设定检查时间间隔为2s
deadtime 30            #设定死亡时间为30s
warntime 10            #设定告警时间为10s（10s以上没有收到对方的回应就报警）
initdead 60            #设定初始化时间为60s
udpport 694            #启动udp694监听端口（该端口可以修改）
ucast eth1 10.10.10.21    #设定侦听的心跳线的接口和对应的对端接口的IP地址
auto_failback on         #启动抢占模式（主在挂了以后重新起来后备会自动切换成备）
node    wshuttle-test1         #指定两个节点
node    wshuttle-test2
#ping 10.6.100.1         #指定一个第三方的仲裁节点(可以不需要)
respawn hacluster /usr/lib/heartbeat/ipfail  #使用这个脚本去侦听对方是否还活着（使用的是ICMP报文检测）
```
#### 1.6 把当前服务器中的上个配置文件拷贝到备用服务器（wshuttle-test2）中
   ``` bash
   cd /etc/ha.d/
   scp  authkeys  ha.cf   haresources   wshuttle-test2:/etc/ha.d/
   ```
#### 1.7 修改备份服务器中的ha.cf配置文件
   ``` bash
   vim /etc/ha.d/ha.cf
   ```
   只需要将备份服务器中的ha.cf 将 ucast eth1 10.10.10.21 改为 ucast eh1 10.10.10.20
#### 1.8 启动heartbeat
``` bash
 service heartbeat start
```
记得先启动主服务器（wshuttle-test1）后启动备份服务器中的heartbeat
启动完成后，查看是否正常运行，即：
``` bash
service heartbeat status
```
显示如下：
{% img  /images/blog/heartbeat-drbd-mysql-Replication/heartbeat-status.png  %}
表示已经运行成功，这是我们用浏览器访问192.168.8.100（虚拟ip）:
{% img  /images/blog/heartbeat-drbd-mysql-Replication/nginx-start-20.png %}
发现nginx已经自动启动，并且是来自主服务器中的nginx的数据。
#### 1.9 宕掉主服务器（wshuttle-test1）
    service heartbeat stop 
    service nginx stop
再打开浏览器访问192.168.8.100(虚拟ip):
{% img  /images/blog/heartbeat-drbd-mysql-Replication/nginx-start-20.png %}
结果发现程序还在运行，并且程序是来至备份服务器wshuttle-test2中的Nginx
#### 1.10 再次启动主服务器的hearbeat
        service heartbeat restart
      等待主服务其中的heartbeat启动完成以后，再次打开浏览器访问192.168.8.100（虚拟ip）
{% img  /images/blog/heartbeat-drbd-mysql-Replication/nginx-start-20.png %}
发现服务程序已自动切换到了主服务器中。到此我们需要的结果已基本完成，如果你需要启动多个程序的时候只需要配置上诉文件 haresources即可，具体的配置可以访问官网完成相应的脚本或者普通程序的启动。
### 2.搭建drbd
#### 2.1 准备工作
搭建drbd时，需要对两台服务器的块进行一个分区操作，可以fdisk -l 列出所有的磁盘和分区的情况时，在对两台服务器中进行分区的时候，需要做到，两台服务器的同步区块大小一致，分区操作可以在网上查看对应的操作即可。
#### 2.2 安装drbd
首先根据系统安装当前的drbd包，可以通过命令：
``` bash
yum list *drbd*
```
查看当前的包，并选择同一版本中的kmod-drbd和drbd进行安装，例如：
``` bash
yum -y install kmod-drbd83 drbd83
```
查看是否安装成功：
``` bash
modprobe -l | grep -i drbd
```
安装完成后再/sbin 目录下有drbd的命令文件， 在/etc/init.d/目录下有drbd启动脚本
``` bash
 ls /sbin/drbd*
 ///sbin/drbdadm  /sbin/drbdmeta  /sbin/drbdsetup
```
#### 2.3 配置drbd
##### 2.3.1 配置hostname
由于在安装配置heartneat中已经配置好两台服务器中的hostname，即在此就不用配置，如果没有配置的需要配置两台服务器中的hostname.
##### 2.3.2 配置/etc/drbd.cnf文件
将该文件的配置文件保存为：
``` bash
include “drbd.d/global_common.conf”;
include “drbd.d/*.res”;
```
将一般情况下，该配置文件都默认为以上的配置文件，不需要做任何改动，但如果你需要定制更多的配置文件，按照原理做相应的改动即可。
##### 2.3.3 修改global_common.cnf配置文件（/etc/drbd.d/global_common.cnf）
``` bash
global {  
    usage-count no;  
}  
common {  
    protocol C;  
    startup {  
        wfc-timeout 15;    
        degr-wfc-timeout 15;    
        outdated-wfc-timeout 15;    
    }  
    disk {  
        on-io-error detach;    
        fencing resource-only;    
    }  
    net {  
        cram-hmac-alg sha1;    
        shared-secret “123456”;     
    }  
    syncer {  
        rate 100M;    
    }  
}  
```
##### 2.3.4 创建一个同步块的配置文件，例如:img_server.res,内容如下：
``` bash
resource img_server{     
    meta-disk internal;     
    device /dev/drbd0; #device指定的参数最后必须有一个数字，用于global的minor-count，  
    #否则会报错。device指定drbd应用层设备。   
    on wshuttle-test1{    #注意：drbd配置文件中，机器名大小写敏感！  
        address 192.168.8.20:7789;     
        disk /dev/sdb5;      
    }     
    on wshuttle-test2 {     
        address 192.168.8.21:7789;     
        disk /dev/sdb5;    
    }     
}  
```
#####  2.3.5 将以上的配置文件，相应的拷贝到备份文件中
#####  2.3.6 在两台服务器上同时创建drbd元数据信息
   
``` bash
  命令：
       drbdadm create-md all
       Writing meta data…
       initializing activity log
       NOT initialized bitmap
       New drbd meta data block successfully created.
```
发现由上面的信息输出表示已创建成功！
##### 2.3.7 启动drbd服务
``` bash
service drbd start 
```
在两台机器中同时启动服务， 在两台机器上用下面的命令drbd-overview或者cat /proc/drbd查看，发现都是Secondary 备份状态
##### 2.3.8 在主节点192.168.8.20上执行下面的命令让其成为主节点

``` bash
drbdadm — –overwrite-data-of-peer primary all
```
然后再看状态:
``` bash
drbd-overview
```
##### 2.3.9将/dev/drbd0格式化并挂载
在主节点192.168.8.20上执行下面的命令
``` bash
     mkfs.ext3 /dev/drbd0
     mkdir /felix
     mount /dev/drbd0 /felix
```
##### 2.3.10 测试同步
     在主节点192.168.8.20上执行下面的命令
``` bash
 cd /felix
 echo “a file created in server5” > felix.data1
```
在备份节点192.168.8.21上执行下面的命令
``` bash
   mkdir /felix
   mount /dev/drbd0 /felix
```
   #mount 会出错，因为mount只能在Primary一端使用在主节点192.168.8.20上执行下面的命令变成备份节点
``` bash
   umount /felix
   drbdadm secondary all
```
  在备份节点192.168.8.21上执行下面的命令变为主节点，可以看到192.168.8.20同步过来的内容
``` bash
  drbdadm primary all
  mount /dev/drbd0 /felix
  less /felix/testfile 
```
 可以查看到文件的内容为”a file created in server5″
 在192.168.8.21新建一个文件
``` bash
 echo “a file created in server6” > felix.data2
```
 再将192.168.8.20变为主节点后mount可以看到 felix.data2的内容也同步了
### 3 搭建mysql replication 
#### 3.1 安装mysql
需要在两台服务器中安装mysql，安装mysql的资料很多，安装上能运行起来，并设置好相关的密码即可。
#### 3.2 就是以上1中搭建的hearbeat，wshuttle-test1和wshuttle-test2分别为主服务器和备份服务器。
在主服务器中创建同步账号：
 sql命令：
``` sql
   grant replication slave on *.* to ‘felix’@’10.10.10.21’ identified by ‘123456’;
 ```
注意事项：其中10.10.10.21是备份服务的ip(eth1) ，而felix为共同的同步账号。123456为账号密码
再次执行sql命令:
``` sql
flush privileges; 
```
在备份服务器中创建一个同步账号：
sql命令：
``` sql
grant replication slave on *.* to ‘felix’@’10.10.10.20’ identified by ‘123456’;
```
注意事项：其中10.10.10.20是主服务的ip(eth1) ，而felix为共同的同步账号。123456为账号密码
再次执行sql命令:
``` sql
flush privileges; 
```
#### 3.3 修改mysql的my.cnf配置文件（/etc/my.cnf）
在主服务器中添加如下配置文件：
``` javascript
    server-id=1
    log-bin=mysql-bin
    binlog-do-db=felix #需要记录二进制日志的数据库.如果有多个数据库可用逗号分隔,或者使用多个binlog-do-db选项
    binlog-ignore-db=mysql #不需要记录进制日志的数据库.如果有多个数据库可用逗号分隔,或者使用多个binlog-ignore-db项
    #需要同步的数据库
    replicate-do-db=felix #需要进行同步的数据库.如果有多个数据库可用逗号分隔,或者使用多个binlog-do-db选项
    replicate-ignore-db=mysql,information_schema #不需要同步的数据库.如果有多个数据库可用逗号分隔,或者使用多binlog-do-db选项
    #同步参数:
    #保证slave挂在任何一台master上都会接收到另一个master的写入信息
    log-slave-updates
    sync_binlog=1
    auto_increment_offset=1
    auto_increment_increment=2
    slave-skip-errors=all #过滤掉一些没啥大问题的错误
 ```
在备份服务器中的my.cnf配置文件中添加如下：
 ``` bash
 server-id=2 
 log-bin=mysql-bin
 binlog-do-db=felix #需要记录二进制日志的数据库.如果有多个数据库可用逗号分隔,或者使用多个binlog-do-db选项 
 binlog-ignore-db=mysql #不需要记录进制日志的数据库.如果有多个数据库可用逗号分隔,或者使用多个binlog-ignore-db选项 
 #需要同步的数据库 
 replicate-do-db=felix #需要进行同步的数据库.如果有多个数据库可用逗号分隔,或者使用多个binlog-do-db选项 
 replicate-ignore-db=mysql,information_schema #不需要同步的数据库.如果有多个数据库可用逗号分隔,或者使用多个binlog-do-db选项 
 #同步参数: 
 ##保证slave挂在任何一台master上都会接收到另一个master的写入信息 
 log-slave-updates 
 sync_binlog=1 
 auto_increment_offset=2 
 auto_increment_increment=2 
 slave-skip-errors=all #过滤掉一些没啥大问题的错误 
 ```
#### 3.4 分别重启两个服务器中的mysql服务
 命令：
``` bash
     service mysqld restart
```
#### 3.5 查看mysql状态
   在主服务器中：
{% img  /images/blog/heartbeat-drbd-mysql-Replication/mysql-master-info.png  %}
在备份服务器中：
{% img  /images/blog/heartbeat-drbd-mysql-Replication/mysql-slave-info.png  %}
#### 3.6 分别在主服务器和备份服务器中用change master语句指定同步位置 : 
主服务器：
sql命令：
``` sql
change master to master_host=’10.10.10.21′,master_user=’felix’,master_password=’123456′,master_log_file=’mysql-bin.000006′,master_log_pos=106;
```
注意事项：其中10.10.10.21为备份服务器的ip,mysql-bin.000006为备份数据库的日志文件
备份服务器：
sql命令：
``` sql
change master to master_host=’10.10.10.20′,master_user=’felix’,master_password=’123456′,master_log_file=’mysql-bin.000005′,master_log_pos=106;
```
注：master_log_file，master_log_pos由上面主服务器查出的状态值中确定 
master_log_file对应File，master_log_pos对应Position 
在备份mysql中执行：
 sql命令：
``` sql
 unlock tables; 
```
#### 3.7 分别在服务器主服务器和备份服务器上启动从服务器线程 
执行sql命令：
``` sql
start slave;
```
分别在两个服务器中查看从服务器的状态：
执行sql命令: 
``` sql
show slave status\G;
```
主：
{% img  /images/blog/heartbeat-drbd-mysql-Replication/mysql-master-status.png  %}
备：
{% img  /images/blog/heartbeat-drbd-mysql-Replication/mysql-slave-status.png  %}
到此，你的主主备份就OK了。
#### 3.8 测试
你可以分别在两个服务器的数据库中增加相应的表及数据，在另外一台数据库中便可以看到，数据已经被同步了。
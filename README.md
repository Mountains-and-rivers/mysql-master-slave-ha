# mysql主从高可用安装
## 环境说明

| 角色   | ip             |
| ------ | -------------- |
| master | 192.168.31.214 |
| slave  | 192.168.31.209 |

```
cat /etc/redhat-release 
CentOS Linux release 8.0.1905 (Core) 
```



## 安装[分别在主备执行]

1,配置yum源

```
curl -LO http://dev.mysql.com/get/mysql57-community-release-el7-11.noarch.rpm
```

2,安装 mysql 源

```
sudo yum localinstall mysql57-community-release-el7-11.noarch.rpm -y
```

3,检查 yum 源是否安装成功

```
[root@master ~]# sudo yum repolist enabled | grep "mysql.*-community.*"
mysql-connectors-community              MySQL Connectors Community
mysql-tools-community                   MySQL Tools Community
mysql57-community                       MySQL 5.7 Community Server
```

4,清除冲突

```
yum remove  mariadb-connector-c-config-3.0.7-1.el8.noarch -y
```

5,安装

```
sudo yum module disable mysql -y  
sudo yum install mysql-community-server -y
```

6,卸载

```

# 卸载安装包
yum remove `rpm -qa |grep -i mysql` -y
# 清除相关目录
rm -rf `find / -name mysql`
# 删除配置文件
rm -rf /etc/my.cnf
# 删除日志
rm -rf /var/log/mysqld.log

```

7,启动

```
 sudo systemctl enable mysqld
 sudo systemctl start mysqld
 sudo systemctl status mysqld
 
 # 创建mysql home目录
 mkdir  /home/mysql
 # 变更属组
 chgrp  mysql mysql
 # 切换mysql 用户
 sudo -u mysql # 未授权脚本运行需添加参数 /bin/bash
 # 创建脚本配置文件
 touch .bashrc 
 # 设置mysql 登录名
 export PS1="[\u@\h]\[$(tput sgr0)\]"
 
```
参考：http://bashrcgenerator.com 创建登录样式
![image](https://github.com/Mountains-and-rivers/mysql-master-slave-ha/blob/main/image/mysql-ps1.png)
8,修改root默认密码

```
MySQL 5.7 启动后，在 /var/log/mysqld.log 文件中给 root 生成了一个默认密码。通过下面的方式找到 root 默认密码，然后登录 mysql 进行修改：

$ grep 'temporary password' /var/log/mysqld.log
[Note] A temporary password is generated for root@localhost: **********
登录 MySQL 并修改密码

$ mysql -u root -p
Enter password: 
mysql> ALTER USER 'root'@'localhost' IDENTIFIED BY 'MyNewPass4!';
注意：MySQL 5.7 默认安装了密码安全检查插件（validate_password），默认密码检查策略要求密码必须包含：大小写字母、数字和特殊符号，并且长度不能少于 8 位。

通过 MySQL 环境变量可以查看密码策略的相关信息：

mysql> SHOW VARIABLES LIKE 'validate_password%';
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
```

9,密码校验策略

指定密码校验策略

```
sudo vi /etc/my.cnf

[mysqld]
# 添加如下键值对, 0=LOW, 1=MEDIUM, 2=STRONG
validate_password_policy=0
```

禁用密码策略

```
sudo vi /etc/my.cnf
	
[mysqld]
# 禁用密码校验策略
validate_password = off
```

```
#重启 MySQL 服务，使配置生效
sudo systemctl restart mysqld
```

10,添加远程登录用户

```
GRANT ALL PRIVILEGES ON *.* TO 'admin'@'%' IDENTIFIED BY 'MyNewPass4!' WITH GRANT OPTION; 
flush privileges;
```

11,配置默认编码为 utf8

MySQL 默认为 latin1, 一般修改为 UTF-8

```
vi /etc/my.cnf
[mysqld]
# 在mysqld下添加如下键值对
character_set_server=utf8
init_connect='SET NAMES utf8'
```

重启 MySQL 服务，使配置生效

```
sudo systemctl restart mysqld
```

查看字符集

```
mysql> SHOW VARIABLES LIKE 'character%';
+--------------------------+----------------------------+
| Variable_name            | Value                      |
+--------------------------+----------------------------+
| character_set_client     | utf8                       |
| character_set_connection | utf8                       |
| character_set_database   | utf8                       |
| character_set_filesystem | binary                     |
| character_set_results    | utf8                       |
| character_set_server     | utf8                       |
| character_set_system     | utf8                       |
| character_sets_dir       | /usr/share/mysql/charsets/ |
+--------------------------+----------------------------+
8 rows in set (0.00 sec
```

12,开启端口[若防火墙未关闭]

```
sudo firewall-cmd --zone=public --add-port=3306/tcp --permanent
sudo firewall-cmd --reload
```

## 主从配置

1，创建复制账号

```
mysql -u root -p
Enter password: ****
mysql> GRANT REPLICATION SLAVE, REPLICATION CLIENT ON *.* TO repl@'192.168.31.%' IDENTIFIED BY 'MyNewPass4!';
```

2,配置主库

```
主库 mysql101 上开启设置

$ sudo vi /etc/my.cnf
[mysqld] 这个部分添加如下内容

server_id=101
log_bin=/var/log/mysql/mysql-bin
log_bin 参数必须唯一, 本例主库设置为 101 ，从库设置为 102；

其他参数：binlog_do_db 参数是复制指定的数据库。如果需要，可以这样设置：

binlog_do_db=db1
binlog_do_db=db2
binlog_do_db=db3
:wq 保存

本例设置的二进制日志文件的目录不是默认的，需要新建一下

$ sudo mkdir /var/log/mysql
分配权限

$ sudo chown mysql:mysql /var/log/mysql
重启主库的 MySQL 服务

$ sudo systemctl restart mysqld
确认一下配置是否成功，登录 MySQL:

$ mysql -u root -p
使用 SHOW MASTER STATUS 命令查看

mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      154 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
从结果看到， File 字段有值，并且前面与配置文件一致，说明配置正确。后面的 000001 说明是第一次，如果 MySQL 从启服务，这个值会递增为 mysql-bin.000002
```

3,配置从库

```
从库 mysql102 上配置

$ sudo vi /etc/my.cnf
添加如下内容

server_id=102
log_bin=/var/log/mysql/mysql-bin.log
relay_log=/var/log/mysql/mysql-relay-bin.log
read_only=1
从库使用 read_only=1，这样会将从库设为只读的，如果有其他需求（如在从库上需要建表），请去掉这个配置选项。

log_slave_updates=1，这个配置一般在 MySQL 5.6 以前使用， MySQL 5.7 以后可以不使用，本例就没有写这个参数。

跟主库一样，从库设置的二进制日志文件的目录不是默认的，需要新建一下

$ sudo mkdir /var/log/mysql
分配权限

$ sudo chown mysql:mysql /var/log/mysql
重启从库的 MySQL 服务

$ sudo systemctl restart mysqld
设置从库的复制参数，登录 MySQL

$ sudo mysql -u root -p
Enter password: ****
设置

mysql> CHANGE MASTER TO MASTER_HOST='192.168.31.214',
    -> MASTER_USER='repl',
    -> MASTER_PASSWORD='password',
    -> MASTER_LOG_FILE='mysql-bin.000001',
    -> MASTER_LOG_POS=0;
MASTER_LOG_POS 设置为 0，因为要从日志的开头读起。 可以通过 SHOW SLAVE STATUS\G 查看复制是否执行正确

MASTER_LOG_FILE 设置为 mysql-bin.000001 ，此选项初始化设置时需要跟主库中的一致。设置好后，如果主库发生重启等，不需再次设置，从库会跟着更新。

查看从库状态

mysql> SHOW SLAVE STATUS \G
*************************** 1. row ***************************
               Slave_IO_State: 
                  Master_Host: 192.168.31.214
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 4
               Relay_Log_File: mysql-relay-bin.000001
                Relay_Log_Pos: 4
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: No
            Slave_SQL_Running: No
                               ...
                               ...
                               ...
1 row in set (0.00 sec)
从 Slave_IO_State, Slave_IO_Running: No, Slave_SQL_Running: No 表明当前从库的复制服务还没有启动。

运行如下命令，开始复制

mysql> START SLAVE;
再次查看状态

mysql> SHOW SLAVE STATUS\G
*************************** 1. row ***************************
               Slave_IO_State: Waiting for master to send event
                  Master_Host: 192.168.31.214
                  Master_User: repl
                  Master_Port: 3306
                Connect_Retry: 60
              Master_Log_File: mysql-bin.000001
          Read_Master_Log_Pos: 154
               Relay_Log_File: mysql-relay-bin.000003
                Relay_Log_Pos: 367
        Relay_Master_Log_File: mysql-bin.000001
             Slave_IO_Running: Yes
            Slave_SQL_Running: Yes
                               ...
                               ...
                               ...
        Seconds_Behind_Master: 0
Master_SSL_Verify_Server_Cert: No
                Last_IO_Errno: 0
                Last_IO_Error: 
               Last_SQL_Errno: 0
               Last_SQL_Error: 
  Replicate_Ignore_Server_Ids: 
             Master_Server_Id: 101
                  Master_UUID: dfa5774e-555e-11e7-a079-080027783c42
             Master_Info_File: /var/lib/mysql/master.info
                    SQL_Delay: 0
          SQL_Remaining_Delay: NULL
      Slave_SQL_Running_State: Slave has read all relay log; waiting for more updates
           Master_Retry_Count: 86400
                               ...
                               ...
                               ...
1 row in set (0.00 sec)
从 Slave_IO_State, Slave_IO_Running, Slave_SQL_Running 的值，可以看出复制已经运行。
```

4,主备切换

```
切换
如果遇到突发情况（如主库计算机硬件异常等），需要将从库改为主库。

登录到从库服务器 mysql102 进行配置

关闭复制

mysql> STOP SLAVE;
重置，清除复制信息，这样再启动时就不会进行复制了。

mysql> RESET SLAVE ALL;
更改从库的 MySQL 配置

$ sudo vi /etc/my.cnf
注释掉 read_only 选项，这样使从库变为可读也可写。

# read_only=1
最后，重启 MySQL 服务

$ sudo systemctl restart mysqld
注意，本例使用 IP 进行 MySQL 主从服务的配置，使用域名访问时，如果主库发生异常，应用程序需要更改访问从库的连接，更好的办法是使用 DNS 服务，只要更新 DNS ，并删除 DNS 的缓存即可，不需要更改程序的配置。 DNS 设置可以参考 CentOS 8 使用 bind 配置私有网络的 DNS
```

5,从另一个服务器开始复制

```
本例之前的内容，演示的是 MySQL 实例初始化后的复制配置。一般的情况是：已经有数据库了，新建复制数据库，这就需要先使两个 MySQL 实例的内容一致，之后在进行复制配置

主库操作
设置锁

mysql> flush tables with read lock;
查看 binlog 的偏移量（注意 Position 字段的值，后面复制需要用到）

mysql> SHOW MASTER STATUS;
+------------------+----------+--------------+------------------+-------------------+
| File             | Position | Binlog_Do_DB | Binlog_Ignore_DB | Executed_Gtid_Set |
+------------------+----------+--------------+------------------+-------------------+
| mysql-bin.000001 |      184 |              |                  |                   |
+------------------+----------+--------------+------------------+-------------------+
1 row in set (0.00 sec)
备份数据库

$ mysqldump db1 > db1.sql
解锁

mysql> unlock tables;
从库设置
将备份上传至从库，并在从库还原数据库

mysql> source db1.sql;
如果主库的表有使用 blob 等存储二进制数据类型的字段，需要设置从库的 max_allowed_packet 参数，之后再导入数据库。

$ sudo vi /etc/my.cnf
添加如下参数

max_allowed_packet=100M
net_buffer_length=8K
bulk_insert_buffer_size=128M
注意

以上字段的值，请根据实际情况修改。

其他设置都一样，此处不再重复说明，主要是设置从库的复制参数

mysql> CHANGE MASTER TO MASTER_HOST='192.168.31.214',
    -> MASTER_USER='repl',
    -> MASTER_PASSWORD='MyNewPass4!',
    -> MASTER_LOG_FILE='mysql-bin.000001',
    -> MASTER_LOG_POS=184;
注意这个 184 即，日志的偏移量要保持一致，这样从库就开始从 184 开始进行，保证与主库执行同样的操作。

这样，复制基本完成。
```

## 高可用配置keepalived

1，安装keepalived

下载地址: https://www.keepalived.org/download.html

```
mkdir /usr/local/src
# keepalived-1.2.13.tar.gz copy到该目录
cd /usr/local/src
yum -y install openssl-devel make gcc 
tar -zxvf keepalived-1.2.13.tar.gz
cd keepalived-1.2.13
./configure && make && make install
cp /usr/local/etc/rc.d/init.d/keepalived /etc/rc.d/init.d/
cp /usr/local/etc/sysconfig/keepalived /etc/sysconfig/
mkdir /etc/keepalived
cp /usr/local/etc/keepalived/keepalived.conf /etc/keepalived/
cp /usr/local/sbin/keepalived /usr/sbin/
chkconfig --add keepalived
chkconfig --level 345 keepalived on
```

2，主从配置文件

master：

```
global_defs {
   router_id MySQL-HA
} 

vrrp_script check_run {
script "/home/mysql/mysql_check.sh"
interval 60
}

vrrp_sync_group VG1 {
group {
VI_1
}
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens160
    virtual_router_id 51
    priority 100  
    advert_int 1
    nopreempt
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    track_script {
    check_run
    }
    notify_master /home/mysql/master.sh
    notify_stop /home/mysql/stop.sh

    virtual_ipaddress {
        192.168.31.230
    }
}
```

slave:

master与slave的keepalived配置文件中只有priority设置不同，master为100，slave为90，其它全一样。配置文件是以块形式组织的，每个块都在{}包围的范围内，#和！开头的行都是注释。

```
global_defs {
   router_id MySQL-HA
} 

vrrp_script check_run {
script "/home/mysql/mysql_check.sh"
interval 60
}

vrrp_sync_group VG1 {
group {
VI_1
}
}

vrrp_instance VI_1 {
    state BACKUP
    interface ens160
    virtual_router_id 51
    priority 90  
    advert_int 1
    nopreempt
    authentication {
        auth_type PASS
        auth_pass 1234
    }
    track_script {
    check_run
    }
    notify_master /home/mysql/master.sh
    notify_stop /home/mysql/stop.sh

    virtual_ipaddress {
        192.168.31.230
    }
}
```

3,参数介绍

```
参数名	参数描述
router-id	主机标识符,主从可以相同也可以不相同
state	表示keepalived角色，都是设成BACKUP则以优先级为主要参考
interface	指定HA监听的网络接口 ifconfig看到的网络接口名，此处是eth2
virtual_router_id	虚拟路由标识，取值0-255，master-1和master-2保持一致
priority	优先级，用来选举master，取值范围1-255
advert_int	发VRRP包时间间隔，即多久进行一次master选举
nopreempt	不抢占，即允许一个priority值比较低的一个节点作为master
delay_loop	设置运行情况检查时间，单位为秒
lb_algorr	设置后端调度器算法，rr为轮询算法
lb_kindDR	设置LVS实现负载均衡的机制，有DR、NAT、TUN三种模式可选
persistence_timeout	会话保持时间，单位为秒
protocol	指定转发协议，有 TCP和UDP可选
notify_down	检查mysql服务down掉后执行的脚本
```

4,检查以及启停脚本

/home/mysql/mysql_check.sh文件用以检测MySQL服务是否正常，当发现连接不上mysql，自动把keepalived进程杀掉，让VIP进行漂移。

/home/mysql/master.sh的作用是状态改为master以后执行的脚本。首先判断复制是否有延迟，如果有延迟，等1分钟后，不论是否有延迟，都并停止复制，并且记录binlog和pos点。

/home/mysql/stop.sh表示Keepalived停止以后需要执行的脚本。检查是否还有复制写入操作，最后无论是否执行完毕都退出。

```
[mysql@master ~]$ more /home/mysql/mysql_check.sh 
#!/bin/bash
. /home/mysql/.bashrc
count=1

while true
do

/usr/bin/mysql -uroot -pMyNewPass4! -S /var/lib/mysql/mysql.sock -e "show status;" > /dev/null 2>&1
i=$?
ps aux | grep mysqld | grep -v grep > /dev/null 2>&1
j=$?
if [ $i = 0 ] && [ $j = 0 ]
then
   exit 0
else
   if [ $i = 1 ] && [ $j = 0 ]
   then
       exit 0
   else
        if [ $count -gt 5 ]
        then
              break
        fi
   let count++
   continue
   fi
fi

done

/etc/init.d/keepalived stop

[mysql@master ~]$ more /home/mysql/master.sh 
#!/bin/bash

. /home/mysql/.bashrc

Master_Log_File=$(/usr/bin/mysql -uroot -pMyNewPass4! -S /var/lib/mysql/mysql.sock -e "show slave status\G" | grep -w Master_Log_File | awk -F": " '{print $2}')
Relay_Master_Log_File=$(/usr/bin/mysql -uroot -pMyNewPass4! -S /var/lib/mysql/mysql.sock -e "show slave status\G" | grep -w Relay_Master_Log_File | awk -F": " '{print $2}')
Read_Master_Log_Pos=$(/usr/bin/mysql -uroot -pMyNewPass4! -S /var/lib/mysql/mysql.sock -e "show slave status\G" | grep -w Read_Master_Log_Pos | awk -F": " '{print $2}')
Exec_Master_Log_Pos=$(/usr/bin/mysql -uroot -pMyNewPass4! -S /var/lib/mysql/mysql.sock -e "show slave status\G" | grep -w Exec_Master_Log_Pos | awk -F": " '{print $2}')

i=1

while true
do

if [ $Master_Log_File = $Relay_Master_Log_File ] && [ $Read_Master_Log_Pos -eq $Exec_Master_Log_Pos ]
then
   echo "ok"
   break
else
   sleep 1

   if [ $i -gt 60 ]
   then
      break
   fi
   continue
   let i++
fi
done

/usr/bin/mysql -uroot -pMyNewPass4! -S /var/lib/mysql/mysql.sock -e "stop slave;"
/usr/bin/mysql -uroot -pMyNewPass4! -S /var/lib/mysql/mysql.sock -e "reset slave all;"
/usr/bin/mysql -uroot -pMyNewPass4! -S /var/lib/mysql/mysql.sock -e "show master status;" > /tmp/master_status_$(date "+%y%m%d-%H%M").txt
[mysql@master ~]$ 
[mysql@master ~]$ 
[mysql@master ~]$ more /home/mysql/stop.sh 
#!/bin/bash

. /home/mysql/.bashrc

M_File1=$(/usr/bin/mysql -uroot -pMyNewPass4! -S /var/lib/mysql/mysql.sock -e "show master status\G" | awk -F': ' '/File/{print $2}')
M_Position1=$(/usr/bin/mysql -uroot -pMyNewPass4! -S /var/lib/mysql/mysql.sock -e "show master status\G" | awk -F': ' '/Position/{print $2}')
sleep 1
M_File2=$(/usr/bin/mysql -uroot -pMyNewPass4! -S /var/lib/mysql/mysql.sock -e "show master status\G" | awk -F': ' '/File/{print $2}')
M_Position2=$(/usr/bin/mysql -uroot -pMyNewPass4! -S /var/lib/mysql/mysql.sock -e "show master status\G" | awk -F': ' '/Position/{print $2}')

i=1

while true
do

if [ $M_File1 = $M_File1 ] && [ $M_Position1 -eq $M_Position2 ]
then
   echo "ok"
   break
else
   sleep 1

   if [ $i -gt 60 ]
   then
      break
   fi
   continue
   let i++
fi
done
```

5,启动keepalived

```
/etc/init.d/keepalived start

如果未成功，查看日志，并调整配置文件
[root@mater ~]# tail -100f /var/log/messages
...
...
...
```

## 测试

1，vip远程登录

```
mysql -uroot MyNewPass4! -h192.168.31.230

```

2，建表名录入数据

```
mysql> create table t2(id int,name varchar(100));
Query OK, 0 rows affected (0.07 sec)

mysql>
mysql> insert into t2 values (1,'abc');
Query OK, 1 row affected (0.02 sec)

```

3，模拟mysql主库crash

```
主库执行

pkill -9 mysqld
[root@mater mysql]# pkill -9 mysqld
[root@mater mysql]# 
[root@mater mysql]# 
[root@mater mysql]# ip addr
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:d7:f7:b7 brd ff:ff:ff:ff:ff:ff
    inet 192.168.31.214/24 brd 192.168.31.255 scope global dynamic noprefixroute ens160
       valid_lft 34123sec preferred_lft 34123sec
    inet6 fe80::c6de:67c0:837b:1c77/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever

查看vip是否飘到备库
1: lo: <LOOPBACK,UP,LOWER_UP> mtu 65536 qdisc noqueue state UNKNOWN group default qlen 1000
    link/loopback 00:00:00:00:00:00 brd 00:00:00:00:00:00
    inet 127.0.0.1/8 scope host lo
       valid_lft forever preferred_lft forever
    inet6 ::1/128 scope host 
       valid_lft forever preferred_lft forever
2: ens160: <BROADCAST,MULTICAST,UP,LOWER_UP> mtu 1500 qdisc mq state UP group default qlen 1000
    link/ether 00:0c:29:d7:f7:b7 brd ff:ff:ff:ff:ff:ff
    inet 192.168.31.214/24 brd 192.168.31.255 scope global dynamic noprefixroute ens160
       valid_lft 34123sec preferred_lft 34123sec
    inet 192.168.148.230/24 scope global ens160
    inet6 fe80::c6de:67c0:837b:1c77/64 scope link noprefixroute 
       valid_lft forever preferred_lft forever

此时客户端连接正常
查看切换的时候的binlog位置
[root@candicate tmp]# more /tmp/master_status_200822-2249.txt
File Position Binlog_Do_DB Binlog_Ignore_DB Executed_Gtid_Set
binlog.000001 120
```

4，切换后处理

```
切换后，排查主库问题，启动主库，然后将主库变成新主库(原从库)的从库
测试过，当新主库异常关闭后，会自动切换到新的从库
```

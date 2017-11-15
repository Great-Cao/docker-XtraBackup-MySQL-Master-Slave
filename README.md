# docker-XtraBackup-MySQL-Master-Slave
利用docker实现XtraBackup配置MySQL Master/Slave 主从同步
## 1. docker pull Ubuntu 镜像
* `docker pull ubuntu`    默认是Ubuntu最新版本镜像
* `docker pull ubuntu:14.04` 选择Ubuntu的14.04版本镜像

## 2. 用Ubuntu镜像起两个容器充当主从服务器
* `docker run --name mysql-master1 -it ubuntu bash`  mysql-master充当主服务器
* `docker run --name mysql-slave1 -it ubuntu bash`   mysql-slave充当从服务器

## 3. 主从服务器安装准备环境
* `apt-get update` 更新源
* `apt-get install vim` 安装vim
* `apt-get install net-tools` 安装net-tools网络分发包可以执行ifconfig命令查看主机ip
* `apt-get install wget` 安装文件下载工具wget
* `apt-get install lsb-release` 安装lsb-release查询linux系统版本
* `apt-get install openssh-client` 安装openssh-client可以执行ssh

## 4. 主从服务器安装XtraBackup
* `wget https://repo.percona.com/apt/percona-release_0.1-4.$(lsb_release -sc)_all.deb`
* `sudo dpkg -i percona-release_0.1-4.$(lsb_release -sc)_all.deb`
* `sudo apt-get update`
* `sudo apt-get install percona-xtrabackup-24`

## 5. 主从服务器安装mysql
* `apt-get install mysql-client mysql-server` 安装Mysql
* `/etc/init.d/mysql start` 启动mysql


## 6. 配置master服务器
**6.1开启 MySQL Binlog**

编辑`/etc/mysql/mysql.conf.d/mysqld.cnf`，文件找到`log-bin`以及`server-id` 参数去掉注释.

* `log-bin     = /var/log/msyql/mysql-bin.log`
* `server-id    = 1`

编辑`/etc/mysql/mysql.conf.d/mysqld.cnf`注释掉`bind-address=172.0.0.1`如果开启mysql默认只会连接本地端口。

**6.2创建mysql同步账号**

* `GRANT REPLICATION SLAVE ON *.* to SYNC_USER@'SLAVE_IP' IDENTIFIED BY 'SYNC_PASSWORD';`
* `FLUSH PRIVILEGES;`


注意: SYNC_USER, SLAVE_IP, SYNC_PASSWORD分别是待同步的账号，从服务器IP, 同步账号密码

**6.3检查 Binlog 是否成功开启**

进入 MySQL 后执行：

* `show variables like "log_bin";`


看到以下输出，表示配置成功：
`+---------------+-------+
| Variable_name | Value |
+---------------+-------+
| log_bin       | ON    |
+---------------+-------+
1 row in set (0.00 sec)`

**6.4备份master数据**

* `mkdir -p /data/backup/`
* `innobackupex --user=root --password='ROOT_PASSWORD' --no-timestamp /data/backup/mysql-master` 创建备份
* `innobackupex --apply-log /data/backup/mysql-master` 准备备份
* `rsync -avpP /data/backup/mysql-master root@SLAVE_IP:/data/backup/mysql-master` 复制数据到 Slave 服务器：

**6.5主从服务器通过密钥免密登录**

主服务器生成私钥和公钥id_rsa，id_rsa.pub把公钥内容复制到从服务器的`authorized_keys`文件内就可以实现主服务器免密登录从服务器。

* `ssh-keygen`生成id_rsa，id_rsa.pub

## 7. 配置slave服务器
**7.1配置server-id**

编辑`/etc/mysql/mysql.conf.d/mysqld.cnf`

* `server-id = 2`
* `read-only=1`

**7.2重置数据**

* `mv /var/lib/mysql  /data/backup/mysql-slave-bak` 或者使用`cp -r /var/lib/mysql  /data/backup/mysql-slave-bak && rm -rf /var/lib/mysql`
* `mv /data/backup/mysql-master /var/lib/mysql` 或者使用 `cp -r /data/backup/mysql-master /var/lib/mysql && rm -rf /data/backup/mysql-master`
* `chown -Rf mysql:mysql /var/lib/mysql`

重启mysql `/etc/init.d/mysql restart`

* 编辑`/etc/mysql/mysql.conf.d/mysqld.cnf`注释掉`bind-address=172.0.0.1`,如果开启mysql默认只会连接本地端口


**7.3测试 Slave 连接 Master**

* `mysql --host=MASTER_IP --user=SYNC_USER --password=SYNC_PASSWORD`其中MASTER_IP， SYNC_USER， SYNC_PASSWORD分别是上面主服务器mysql生成的。

**7.4查看 Binlog 位置信息**

* `cat /var/lib/mysql/xtrabackup_binlog_info`
会显示如下信息：

* `mysql-bin.000002	154` 
 
即下个操作步骤的{MASTER_LOG_FILE}、{MASTER_LOG_POS}参数的值。

**7.5Slave 连接 Maser**
进入从服务器Mysql，执行
`CHANGE MASTER TO MASTER_HOST='{MASTER_IP}',
    MASTER_USER='{REPL_USER}',
    MASTER_PASSWORD='{REPL_PASSWORD}',
    MASTER_LOG_FILE='{MASTER_LOG_FILE}',
    MASTER_LOG_POS={MASTER_LOG_POS};`

**7.6在mysql中启动 Slave：**

* `start slave;`

**7.7查看同步状态**

* `show slave status\G`

显示

`Slave_IO_Running: Yes`
`Slave_SQL_Running: Yes`

 则启动成功，主服务器Mysql数据库变化，从服务器Mysql也会同步变化。

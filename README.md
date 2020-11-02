# **CentOS 7离线安装CDH：**
---
- ## **系统环境准备** ##
- 关闭防火墙(所有节点)
   - 停服务
	`sudo systemctl stop firewalld.service`
   - 屏蔽服务
	`sudo systemctl mask firewalld.service`
- 关闭SELinux（所有节点）
	```
	sudo vi /etc/selinux/config
	SELINUX=disabled
	```
- 配置NTP服务，同步时间（所有节点）
   - 安装
	`sudo yum -y install ntp`
   - 设置开机启动
	`sudo chkconfig ntpd on`
   - 服务启动
	`sudo systemctl start ntpd` 
   - 手动同步时钟的方法
	`sudo ntpdate -u 0.cn.pool.ntp.org`
- 修改每台Server的hostname，文件目录/etc/hostname
- 配置每台Server的Hosts，将每台主机host都配置到/etc/hosts文件中，包含本机，并注释原localhost等的配置
---
- ## **文件下载** ##
   - [Cloudera Manager安装包](https://archive.cloudera.com/cm6/6.3.1/repo-as-tarball/cm6.3.1-redhat7.tar.gz)
   - [CDH相关](https://archive.cloudera.com/cdh6/6.3.2/parcels/)
   - [Mysql下载](https://dev.mysql.com/downloads/mysql/5.7.html)，Mysql档案管如图选择
	![mysql下载](./mysql_download_choose.png 'mysql下载时选择')
   - [mysql connector下载](https://downloads.mysql.com/archives/c-j/)
	![mysql下载](./mysql_connector_download.png 'mysql下载时选择')
   - [Oracle JDK8下载地址](https://www.oracle.com/java/technologies/javase/javase8-archive-downloads.html)
- 下载完成后的文件列表应该是这样的
   - cm部分
	```
	cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm
	cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm
	cloudera-manager-server-6.3.1-1466458.el7.x86_64.rpm
	```
   - CDH部分
	```
	CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel
	CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha1
	manifest.json
	```
   - Mysql部分
	```
	mysql-community-common-5.7.31-1.el7.x86_64.rpm
	mysql-community-libs-5.7.31-1.el7.x86_64.rpm
	mysql-community-client-5.7.31-1.el7.x86_64.rpm
	mysql-community-server-5.7.31-1.el7.x86_64.rpm
	mysql-community-libs-compat-5.7.31-1.el7.x86_64.rpm
	```
   - mysql connector部分
   `mysql-connector-java-5.1.44.tar.gz`
   - oracle jdk 1.8部分
   `jdk-8u181-linux-x64.tar.gz`

#### *建议所有操作在root用户下完成，因一些原因不能在root操作的，可以新建用户，但新建用户一定要获得免密sudo权限* ####
- ## **JDK安装** ##(如需要新增，则每台Server都需要新增同一个用户名)
   - 新增用户
	`useradd yourusername`
   - 修改密码
	`passwd yourpasswd`  
   - 配置sudo免密
	```
	sudo vi /etc/sudoers
	yourusername   ALL=(ALL)NOPASSWD:      ALL
	```
---
## 以下所有操作均在自建用户操作 ##
- ## **JDK安装** ##
- 安装jdk、配置环境变量
这里注意开始坑了，CDH对JDK的版本要求是**绝对的苛刻**，苛刻到小版本都最好不要去尝试官方未推荐的
[这里](https://docs.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_java_requirements.html)有官方要求的JDK版本要求，其中列出了一些已知的小版本BUG，这里笔者用了JDK1.8.0_181
~~受之前工作影响喜欢将自己安装的一些东西装在自己建的 一个目录（/data）下，事实证明这里就正式跳入第一个坑了，因为cm安装的时候要看/usr/bin/下面，所以后面又删除了JAVA_HOME的配置，在/usr/bin/下面建立软链接~~
就算添加了软链接到/usr/bin等系统PATH目录也没有什么卵用，笔者受这个问题困惑了很久，最后在[这里](https://docs.cloudera.com/documentation/enterprise/6/6.3/topics/cdh_ig_jdk_installation.html)找到了答案，**只能安装在/usr/java/jdk-version**下面，如果这里有其它版本，建议删除之，为不影响原来其它应用需要重新将系统使用的链接修复。

- ### 用户目录生成SSH key，如有请跳过 ###
`sshkey-gen -t rsa` 
一路回车就算完成，操作完成后用户目录下应有.ssh目录，里面应有公私钥和一个known_hosts，如下三个文件
```
id_rsa  id_rsa.pub  known_hosts
```
- ### 配置免密登录 ###
可使用ssh-copy-id -i ~/.ssh/id_rsa.pub [romte_ip]
这里笔者使用比较直接的，直接在.ssh目录下添加客户机公钥，如下
```
touch authorized_keys
cat id_rsa.pub >> authorized_keys (先将自己添加进免密登录列表)
```
复制客户机的公钥依次添加进该文件即可，有多少台机器都需要操作，且相互之间要实现免密登录
  这里须要注意一点authorized_keys文件的权限必须是600
`chmod 600 authorized_keys`
- 更新yum源
`sudo yum upgrade`
---
- ## **Mysql 5.7安装** ##
   - 首先卸载centos7自带的mariadb
	```
	rpm -qa | grep mariadb (查询当前机器上安装的mariadb版本)
	sudo rpm -e mariadb-libs-5.5.65-1.el7.x86_64 --nodeps (版本根据上面查询出来的视情况修改)
	```
   - 安装依赖
	```
	yum -y install perl.x86_64
	yum install -y libaio.x86_64
	yum -y install net-tools.x86_64
	```
   - 安装Mysql**（注意安装顺序，有依赖）**
	```
	sudo rpm -ivh mysql-community-common-5.7.31-1.el7.x86_64.rpm
	sudo rpm -ivh mysql-community-libs-5.7.31-1.el7.x86_64.rpm
	sudo rpm -ivh mysql-community-client-5.7.31-1.el7.x86_64.rpm
	sudo rpm -ivh mysql-community-server-5.7.31-1.el7.x86_64.rpm
	sudo rpm -ivh mysql-community-libs-compat-5.7.31-1.el7.x86_64.rpm
	```
   - 修改密码安全级别、字符集、时区，在/etc/my.conf添加
	```
	validate_password_policy=0
	validate_password=off
	character_set_server=utf8
	init_connect='SET NAMES utf8'
	default-time_zone='+8:00'
	```
   - 启动
	`sudo systemctl start mysqld.service`
   - 查找初始密码
	`sudo cat /var/log/mysqld.log|grep password`
   - 使用上面的密码登录Mysql，修改初始密码
	```
	mysql -uroot -pxxx
	set password = password('yourpassword')
	```
   - 授权用户root使用密码passwd从任意主机连接到mysql服务器
	```
	GRANT ALL PRIVILEGES ON *.* TO 'root'@'%' IDENTIFIED BY 'youpassword' WITH GRANT OPTION;
	flush privileges;
	```
   - 设置开机启动
	`sudo systemctl enable mysqld`
   - 添加数据库
	```
	create database hive DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
	create database oozie DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
	create database hue DEFAULT CHARSET utf8 COLLATE utf8_general_ci;
	create user 'hive'@'%' identified by '<yourpassword>';
	create user 'oozie'@'%' identified by '<yourpassword>';
	create user 'hue'@'%' identified by '<yourpassword>';
	grant all privileges on hive.* to 'hive'@'%' with grant option;
	grant all privileges on oozie.* to 'oozie'@'%' with grant option;
	grant all privileges on hue.* to 'hue'@'%' with grant option;
	```
- 安装Mysql驱动
	```
	tar xvf mysql-connector-java-5.1.44.tar.gz
	sudo mkdir /usr/share/java
	sudo mv mysql-connector-java-5.1.44-bin.jar /usr/share/java/mysql-connector-java.jar
	```
---
- ## **Cloudera Manager安装** ##
   - master节点安装CM
	```
	sudo yum -y localinstall cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm
	sudo yum -y localinstall cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm
	sudo yum -y localinstall cloudera-manager-server-6.3.1-1466458.el7.x86_64.rpm
	```
   - copy安装包到slave节点：
	`scp -r ./cdh spark@datanode:/data`
   - slave安装
	```
	sudo yum -y localinstall cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm
	sudo yum -y localinstall cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm
	```
	
   - 建立数据库(**注意这里只是建立了一个空库，表是CM启动后自动创建的**)
	`/opt/cloudera/cm/schema/scm_prepare_database.sh mysql -hlocalhost -uroot -p scm scm`
   - 配置Agent(每个节点)
	```
	sudo vi /etc/cloudera-scm-agent/config.ini
	server_host=youmasterhostname
	```
- ## **CDH文件导入** ##
- 准备parcels
   - 将CDH相关文件拷贝到主节点/opt/cloudera/parcel-repo/
   - 修改文件名**很重要，否则会导致重新下载**
	`sudo mv CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha1 CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha`
- 启动Cloudera Manager
   - 主切点
   	```
	sudo systemctl start cloudera-scm-server
	systemctl start cloudera-scm-agent
	```
   - 其它节点
   	```
	systemctl start cloudera-scm-agent
	```

## 启动后，可能遇到的一些问题，需要根据情况作如下调整 ##
- 添加HDFS、HIVE中内存
- CDH中JobHistory启动失败
   - 1. 因权限受拒
	`sudo -u hdfs hdfs dfs -chmod -R 777 /`
   - 2. 内存太小，参考[这里](https://community.cloudera.com/t5/Support-Questions/History-Server-can-not-start/td-p/166247)
   
- nodemanager启动不了的问题：检查/web/yarn目录所属用户及权限
	`sudo chown -R yarn:yarn /web/yarn/nm`

---
[参考](https://www.jianshu.com/p/94f32f90a47a)

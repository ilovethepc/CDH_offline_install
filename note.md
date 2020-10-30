#**CentOS 7离线安装CDH：**
首选准备下载以下文件，可以在官方找到下载链接
[Cloudera Manager安装包](https://archive.cloudera.com/cm6/6.3.1/repo-as-tarball/cm6.3.1-redhat7.tar.gz)
[CDH相关](https://archive.cloudera.com/cdh6/6.3.2/parcels/)
[Mysql下载](https://dev.mysql.com/downloads/mysql/5.7.html)，如图选择![mysql下载](https://github.com/ilovethepc/CDH_offline_install/blob/master/mysql%20download%20choose.png ''mysql下载时选择'')
下载完成后的文件列表应该是这样的
```
cloudera-manager-agent-6.3.1-1466458.el7.x86_64.rpm
cloudera-manager-daemons-6.3.1-1466458.el7.x86_64.rpm
cloudera-manager-server-6.3.1-1466458.el7.x86_64.rpm
CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel
CDH-6.3.2-1.cdh6.3.2.p0.1605554-el7.parcel.sha1
manifest.json
```

#### *建议所有操作在root用户下完成，因一些原因不能在root操作的，可以新建用户，但新建用户一定要获得免密sudo权限* ####
## 新增用户 ##
```
新增用户：useradd spark
修改密码：passwd spark  
配置sudo免密：修改/etc/sudoers文件，末尾添加spark   ALL=(ALL)NOPASSWD:      ALL
```
---
***
***
## 以下所有操作均在spark用户 ##
- 安装jdk、配置环境变量
[OracleJDK8下载地址](https://www.oracle.com/java/technologies/javase/javase8-archive-downloads.html)
这里注意开始坑了，CDH对JDK的版本要求是**绝对的苛刻**，苛刻到小版本都最好不要去尝试官方未推荐的
[这里](https://docs.cloudera.com/documentation/enterprise/6/release-notes/topics/rg_java_requirements.html)有官方要求的JDK版本要求，其中列出了一些已知的小版本BUG，这里笔者用了JDK1.8.0_181
~~受之前工作影响喜欢将自己安装的一些东西装在自己建的 一个目录（/data）下，事实证明并没有什么卵用，因为cm安装的时候要看/usr/bin/下面，所以后面又删除了JAVA_HOME的配置，在/usr/bin/下面建立软链接~~
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
接下来直接用VI复制客户机的公钥依次添加进该文件即可，有多少台机器都需要操作，且相互之间要实现免密登录
这里须要注意一点authorized_keys文件的权限必须是600
`chmod 600 authorized_keys`
- 更新yum源
`sudo yum upgrade`
- 挑选一台性能相对较好的机器作为master，安装mysql


```flow
st=>start: Start|past:>http://www.google.com[blank]
e=>end: End:>http://www.google.com
op1=>operation: My Operation|past
op2=>operation: Stuff|current
sub1=>subroutine: My Subroutine|invalid
cond=>condition: Yes or No?|approved:>http://www.google.com
c2=>condition: Good idea|rejected
io=>inputoutput: catch something…|request

st->op1(right)->cond
cond(yes, right)->c2
cond(no)->sub1(left)->op1
c2(yes)->io->e
c2(no)->op2->e
```
~~测试删除线~~
> 引用
>> 引用1
>>> 引用3
>>>> 引用4

---
[参考](https://www.jianshu.com/p/94f32f90a47a)
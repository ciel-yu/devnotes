Cloudera 备忘录
==============
Date: 2017-05-16

CDH Version: 5.11

主要参考文档 [Cloudera Installation](https://www.cloudera.com/documentation/enterprise/latest/topics/installation.html)

安装方法选用 [Installation Path B](https://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_install_path_b.html)

安装环境
-------
Ubuntu 14.04 (trusty)

### 设置无密码的SUDO账户

假设安裝时创建用户为`service`, 由于安装 CDH 时需要用到无密码的 SUDO 账号, 通过下列方式可以把`service`对应的密码提示关闭:

在 `/etc/sudoers.d/` 增加文件 `service` 内容为：
```
service ALL=(ALL) NOPASSWD: ALL
```

### 安装 Oracle JDK

用 root 身份运行下列指令：
```
add-apt-repository ppa:webupd8team/java
apt-get update
apt-get install oracle-java8-installer 
```


### 配置 Cloudera repository

```
curl "http://archive.cloudera.com/cm5/ubuntu/xenial/amd64/cm/archive.key" | apt-key add -
curl "http://archive.cloudera.com/cm5/ubuntu/xenial/amd64/cm/cloudera.list" > /etc/apt/sources.list.d/cloudera.list
apt-get update
```

### 安装 Cloudera Manager

```
apt-get install -y cloudera-manager-daemons cloudera-manager-server libmysql-java
```
按照[官方文档](https://www.cloudera.com/documentation/enterprise/latest/topics/cm_ig_mysql.html)配置数据库，摘录如下：
```SQL
create database <database> DEFAULT CHARACTER SET utf8;
grant all on <database>.* TO 'cmf'@'%' IDENTIFIED BY 'cmf';
```
`<database>` 需要替换为具体的数据库名称。这里用户名和密码都设置为 `cmf` 。
数据库配置完毕，编辑 cloudera 的数据配置文件 `/etc/cloudera-scm-server/db.properties` :

```
com.cloudera.cmf.db.type=mysql
com.cloudera.cmf.db.host=<数据库服务器的ip>
com.cloudera.cmf.db.name=<database>
com.cloudera.cmf.db.user=cmf
com.cloudera.cmf.db.password=cmf
com.cloudera.cmf.db.setupType=EXTERNAL
```

最后，启动 Cloudera Manager
```
start cloudera-scm-server
```
启动完毕后就可以通过7180端口来访问 Cloudera Manager ，后续的 Hadoop 和其它组件的安装都通过 Cloudera Manager 的安装向导来完成。
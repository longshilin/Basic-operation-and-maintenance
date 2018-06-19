# 非容器部署

## 1.软件环境部署
### 查看linux系统版本号：
```
$ cat /etc/redhat-release
CentOS Linux release 7.4.1708 (Core)
```
### 安装Java
```
sudo yum install -y java-1.8.0-openjdk.x86_64

$ java -version
java version "1.8.0_162"
Java(TM) SE Runtime Environment (build 1.8.0_162-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.162-b12, mixed mode)
```
### 增加软件资源库
```
获取软件包：wget http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
安装源：sudo rpm -ivh epel-release-6-8.noarch.rpm
```
### 安装memcached
```
sudo yum install -y memcached.x86_64
```
### 安装Redis
```
sudo yum install -y redis.x86_64
```
### 安装Tomcat-7
```
获取资源包：wget http://apache.mirrors.ionfish.org/tomcat/tomcat-7/v7.0.88/bin/apache-tomcat-7.0.88.tar.gz
安装：tar -zxvf apache-tomcat-7.0.88.tar.gz
mv apache-tomcat-7.0.88 tomcat7 && mv tomcat7 /home/centos/longsl/app/
```
### 安装MySQL
```
sudo yum -y install http://www.percona.com/downloads/percona-release/redhat/0.1-3/percona-release-0.1-3.noarch.rpm
sudo yum install -y Percona-Server-server-56.x86_64
启动mysql服务：sudo mysqld start
```
### 安装Python脚本运行所需环境
```
sudo yum -y install python-redis.noarch
sudo yum -y install mysql-connector-python.noarch
```
## 2.部署proxy和gameproxy
### 部署proxy
① 上传proxy.tar包到`/home/centos/longsl/app`
② 将proxy.tar解压：mkdir proxy && mv proxy.tar proxy && tar xvf proxy.tar
③ chmod u+x /home/centos/longsl/app/proxy/startup && chmod u+x /home/centos/longsl/app/proxy/startproxy.sh
④ ./startup
### 部署Tomcat相关server（仅包含：KP）
① 上传kp.tar包到`/home/centos/longsl/app`
② 将kp.tar解压：mkdir kp && mv kp.tar kp && tar xvf kp.tar
③ 将kp资源包放到Tomcat-7的webapps文件夹下：mv kp/ tomcat7/webapps
### 部署Redis和Memcached
#### 启动Redis
① 修改redis配置文件，允许其在后台执行：sudo vi /etc/redis.conf 修改：daemonize yes
② 启动redis-server：/usr/bin/redis-server /etc/redis.conf
③ 验证redis是否启动成功：sudo netstat -lntp | grep 6379
#### 启动Memcached
① 启动Memcached：/usr/bin/memcached -d -u root -m 512 -p 11211 -c 5000
② 验证Memcached是否启动成功：sudo netstat -lntp | grep 11211

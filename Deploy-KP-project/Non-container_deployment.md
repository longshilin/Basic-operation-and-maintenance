# 非容器部署

## 环境部署
### 查看linux系统版本号：
```
$ cat /etc/redhat-release
CentOS Linux release 7.4.1708 (Core)
```

### 安装Java
```
yum install  -y java-1.8.0-openjdk.x86_64

$ java -version
java version "1.8.0_162"
Java(TM) SE Runtime Environment (build 1.8.0_162-b12)
Java HotSpot(TM) 64-Bit Server VM (build 25.162-b12, mixed mode)
```
### 增加软件资源库
```
wget http://dl.fedoraproject.org/pub/epel/6/x86_64/epel-release-6-8.noarch.rpm
安装源： rpm -ivh epel-release-6-8.noarch.rpm
```
### 安装memcached
```
yum install -y memcached.x86_64
```

### 安装Redis
```
yum install -y redis.x86_64
```
### 安装Tomcat-7

http://apache.mirrors.ionfish.org/tomcat/tomcat-7/v7.0.88/bin/apache-tomcat-7.0.88.tar.gz

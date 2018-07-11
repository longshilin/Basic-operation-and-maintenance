# 安装Docker
CentOS中更新源后安装docker，官网https://docs.docker.com/engine/installation/linux/centos/
安装完成Docker环境之后不要去关闭CentOS的防火墙和Selinux，因为Docker的安全机制是基于iptables的，关闭selinux会使的Docker的安装出错。

在我的机器上，我是通过`sudo yum -y install docker`下载最新版docker，因此我的docker版本环境：
```
$ sudo docker version
Client:
 Version:         1.13.1
 API version:     1.26
 Package version: docker-1.13.1-63.git94f4240.el7.centos.x86_64
 Go version:      go1.9.4
 Git commit:      94f4240/1.13.1
 Built:           Fri May 18 15:44:33 2018
 OS/Arch:         linux/amd64

Server:
 Version:         1.13.1
 API version:     1.26 (minimum version 1.12)
 Package version: docker-1.13.1-63.git94f4240.el7.centos.x86_64
 Go version:      go1.9.4
 Git commit:      94f4240/1.13.1
 Built:           Fri May 18 15:44:33 2018
 OS/Arch:         linux/amd64
 Experimental:    false
```
修改Docker的系统配置文件：
```
# /etc/sysconfig/docker

# Modify these options if you want to change the way the docker daemon runs
OPTIONS='--selinux-enabled --insecure-registry localhost -H tcp://0.0.0.0:2375  -H unix:///var/run/docker.sock'
```

修改centos的系统配置文件，将/etc/selinux/config文件中SELINUX=enforcing改为SELINUX=disabled，永久关闭SElinux状态，避免出现Docker挂载主机目录Docker访问出现Permission denied。

参考文档有：

[1] [Docker挂载主机目录Docker访问出现Permission denied的解决办法](https://blog.csdn.net/rznice/article/details/52170085)

[2] [关闭SELinux](http://blog.51cto.com/zhaodongwei/1745837)

# 安装Docker Registry
目前Docker Registry已经升级到了v2，最新版的Docker已不再支持v1。Registry v2使用Go语言编写，在性能和安全性上做了很多优化，重新设计了镜像的存储格式。 先切换为root用户，再执行下面命令：
```
 # curl -L https://github.com/docker/compose/releases/download/1.5.2/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose
 # chmod +x /usr/local/bin/docker-compose
 # ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose
```
安装htpasswd
```
# sudo yum install httpd-tools
```

# 运行Registry Container并使用Nginx做代mv理
## 运行nginx和registry容器

　创建一个工作目录，例如/data/programs/docker，并在该目录下创建docker-compose.yml文件，将以下docker-compose.yml内容复制粘贴到的docker-compose.yml文件中。
```
# sudo mkdir -p /data/programs/docker
# cd /data/programs/docker/
# sudo mkdir data && sudo mkdir nginx
```
　　docker-compose.yml文件内容大致意思为: 基于“nginx:1.9” image运行nginx容器，暴露容器443端口到host 443端口。并挂载当前目录下的nginx/目录为容器的/etc/nginx/config.d目录。
　　nginx link到registry容器。基于registry:2 image创建registry容器，将容器5000端口暴露到host 5000端口，使用环境变量指明使用/data为根目录，并将当前目录下data/文件夹挂载到容器的/data目录。
```
nginx:  
  image: "nginx:1.9"  
  ports:  
    - 443:443  
  links:  
    - registry:registry  
  volumes:  
    - ./nginx/:/etc/nginx/conf.d  
registry:  
  image: registry:2  
  ports:  
    - 127.0.0.1:5000:5000  
  environment:  
    REGISTRY_STORAGE_FILESYSTEM_ROOTDIRECTORY: /data  
  volumes:  

    - ./data:/data  
```

## 配置nginx

在nginx目录中创建registry.conf文件配置nginx。配置nginx与registry的关系，转发端口，以及其他nginx的配置选项。复制，粘贴如下内容到你的registry.conf文件中：

```
# cat /data/programs/docker/nginx/registry.conf

upstream docker-registry {
  server registry:5000;
}

server {
  listen 443;
  server_name myregistrydomain.com;

  # SSL
  # ssl on;
  # ssl_certificate /etc/nginx/conf.d/domain.crt;
  # ssl_certificate_key /etc/nginx/conf.d/domain.key;
  # disable any limits to avoid HTTP 413 for large image uploads

  client_max_body_size 0;

  # required to avoid HTTP 411: see Issue #1486 (https://github.com/docker/docker/issues/1486)

  chunked_transfer_encoding on;
  location /v2/ {

    # Do not allow connections from docker 1.5 and earlier
    # docker pre-1.6.0 did not properly set the user agent on ping, catch "Go *" user agents

    if ($http_user_agent ~ "^(docker\/1\.(3|4|5(?!\.[0-9]-dev))|Go ).*$" ) {
      return 404;
    }

    # To add basic authentication to v2 use auth_basic setting plus add_header
    # auth_basic "registry.localhost";
    # auth_basic_user_file /etc/nginx/conf.d/registry.password;
    # add_header 'Docker-Distribution-Api-Version' 'registry/2.0' always;

    proxy_pass                          http://docker-registry;
    proxy_set_header  Host              $http_host;   # required for docker client's sake
    proxy_set_header  X-Real-IP         $remote_addr; # pass on real client's IP
    proxy_set_header  X-Forwarded-For   $proxy_add_x_forwarded_for;
    proxy_set_header  X-Forwarded-Proto $scheme;
    proxy_read_timeout                  900;

  }
}
```
添加用户名和密码
在/data/programs/docker/nginx目录下执行下面命令创建用户名和密码对，如果要创建多个用户名和密码对，则不是使用“-c“选项。
```
# htpasswd -c registry.password docker

New password:
Re-type new password:
Adding password for user docker
```
然后修改Registry.conf文件，取消下面三行的注释。
```
auth_basic "registry.localhost";
auth_basic_user_file /etc/nginx/conf.d/registry.password;
add_header 'Docker-Distribution-Api-Version' 'registry/2.0' always;
```

再次执行docker-compose up运行registry，这时使用localhost:5000端口访问得到的结果为”{}”,但是使用localhost:443访问将得到”401 Authorisation Required“的提示。加入用户名和密码验证才能得到与直接访问registry 5000端口相同的结果。
```
# curl http://localhost:5000/v2/

{}
```
```
# curl http://localhost:443/v2/

<html>
<head><title>401 Authorization Required</title></head>
<body bgcolor="white">
<center><h1>401 Authorization Required</h1></center>
<hr><center>nginx/1.9.15</center>
</body>
</html>
```
四、加入SSL验证
如果你有经过认证机构认证的证书，则直接使用将证书放入nginx目录下即可。如果没有，则使用openssl创建自己的证书。
进入`/data/programs/docker/nginx`目录。
```
生成一个新的root key
$ openssl genrsa -out devdockerCA.key 2048
生成根证书（一路回车即可）
$ openssl req -x509 -new -nodes -key devdockerCA.key -days 10000 -out devdockerCA.crt
为server创建一个key。（这个key将被nginx配置文件registry.con中ssl_certificate_key域引用）
$ openssl genrsa -out domain.key 2048
```
制作证书签名请求。注意在执行下面命令时，命令会提示输入一些信息，”Common Name”一项一定要输入你的域名（官方说IP也行，但是也有IP不能加密的说法），其他项随便输入什么都可以。不要输入任何challenge密码，直接回车即可。
```
$ openssl req -new -key domain.key -out dev-docker-registry.com.csr
You are about to be asked to enter information that will be incorporated
into your certificate request.
What you are about to enter is what is called a Distinguished Name or a DN.
There are quite a few fields but you can leave some blank
For some fields there will be a default value,
If you enter '.', the field will be left blank.
-----
Country Name (2 letter code) [XX]:
State or Province Name (full name) []:
Locality Name (eg, city) [Default City]:
Organization Name (eg, company) [Default Company Ltd]:
Organizational Unit Name (eg, section) []:
Common Name (eg, your name or your server's hostname) []:docker-registry.com
Email Address []:
Please enter the following 'extra' attributes
to be sent with your certificate request
A challenge password []:
An optional company name []:
```
签署认证请求
```
$ openssl x509 -req -in dev-docker-registry.com.csr -CA devdockerCA.crt -CAkey devdockerCA.key -CAcreateserial -out domain.crt -days 10000
```
配置nginx使用证书
修改registry.conf配置文件，取消如下三行的注释
```
ssl on;  
ssl_certificate /etc/nginx/conf.d/domain.crt;  
ssl_certificate_key /etc/nginx/conf.d/domain.key;
```
运行Registry
执行`docker-compose up -d`在后台运行Registry，并使用curl验证结果。这时使用localhost:5000端口仍然可以直接访问Registry，但是如果使用443端口通过nginx代理访问，因为已经加了SSL认证，所以使用http将返回“400 bad request”
```
$ curl http://localhost:5000/v2/
{}

$ curl http://localhost:443/v2/
<html>
<head><title>400 The plain HTTP request was sent to HTTPS port</title></head>
<body bgcolor="white">
<center><h1>400 Bad Request</h1></center>
<center>The plain HTTP request was sent to HTTPS port</center>
<hr><center>nginx/1.9.9</center>
</body>
</html>
```

应该使用https协议
```
$ curl https://localhost:443/v2/
curl: (60) Peer certificate cannot be authenticated with known CA certificates
More details here: http://curl.haxx.se/docs/sslcerts.html
curl performs SSL certificate verification by default, using a "bundle"
of Certificate Authority (CA) public keys (CA certs). If the default
bundle file isn't adequate, you can specify an alternate file
using the --cacert option.
If this HTTPS server uses a certificate signed by a CA represented in
the bundle, the certificate verification probably failed due to a
problem with the certificate (it might be expired, or the name might
not match the domain name in the URL).
If you'd like to turn off curl's verification of the certificate, use
the -k (or --insecure) option.
```

由于是使用的未经任何认证机构认证的证书，并且还没有在本地应用自己生成的证书。所以此时会提示使用的是未经认证的证书，可以使用“-k”选项不进行验证。
```
$ curl -k https://localhost:443/v2/
<html>
<head><title>401 Authorization Required</title></head>
<body bgcolor="white">
<center><h1>401 Authorization Required</h1></center>
<hr><center>nginx/1.9.9</center>
</body>
</html>
```

# Docker客户端使用Registry
 添加证书  Centos 6/7 添加证书具体步骤如下
 ```
 安装ca-certificates包
$ yum install ca-certificates
使能动态CA配置功能
$ update-ca-trust force-enable
将key拷贝到/etc/pki/ca-trust/source/anchors/
$ cp devdockerCA.crt /etc/pki/ca-trust/source/anchors/
使新拷贝的证书生效
$ update-ca-trust extract
证书拷贝后，需要重启docker以保证docker能使用新的证书
$ service docker restart
Docker pull/push image测试
制作要push到registry的镜像
 ```

# 测试
```
#查看本地已有镜像
$ docker images
REPOSITORY          TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
registry            2                   cd57aad0bd45        3 days ago          224.5 MB
nginx               1.9                 813e3731b203        3 weeks ago         133.9 MB
#为本地镜像打标签
$ docker tag registry:2 docker-registry.com/registry:2
$ docker tag nginx:1.9 docker-registry.com/nginx:1.9
$ docker images
REPOSITORY                     TAG                 IMAGE ID            CREATED             VIRTUAL SIZE
registry                       2                   cd57aad0bd45        3 days ago          224.5 MB
docker-registry.com/registry   2                   cd57aad0bd45        3 days ago          224.5 MB
nginx                          1.9                 813e3731b203        3 weeks ago         133.9 MB
docker-registry.com/nginx      1.9                 813e3731b203        3 weeks ago         133.9 MB
push测试

#不登陆直接push镜像到registry，会提示失败
[root@PRO-REGISTRY-220 ~]# docker push docker-registry.com/registry
The push refers to a repository [docker-registry.com/registry] (len: 1)
cd57aad0bd45: Image push failed
cd57aad0bd45: Buffering to Disk
Please login prior to push:
Username:
Error response from daemon: no successful auth challenge for https://docker-registry.com/v2/ - errors: [basic auth attempt to https://docker-registry.com/v2/ realm "registry.localhost" failed with status: 401 Unauthorized]
#登陆后，再试
$docker login https://docker-registry.com
Username: docker
Password:
Email:
WARNING: login credentials saved in /root/.docker/config.json
Login Succeeded
#可以push 镜像到registry
$ docker push docker-registry.com/registry
The push refers to a repository [docker-registry.com/registry] (len: 1)
cd57aad0bd45: Image already exists
b3c39a7768ea: Image successfully pushed
4725a48b84d4: Image successfully pushed
7b4078296418: Image successfully pushed
7bd663e30ad0: Image successfully pushed
28864e830e4d: Image successfully pushed
7bd2d56d8449: Image successfully pushed
af88597ec24b: Image successfully pushed
b2ae0a712b39: Image successfully pushed
02e5bca4149b: Image successfully pushed
895b070402bd: Image successfully pushed
Digest: sha256:92835b3e54c05b90e416a309d37ca02669eb5e78e14a0f5ccf44b90d4c21ed4c
搜索镜像

curl https://docker:123456@docker-registry.com/v2/_catalog
{"repositories":["registry"]}
curl https://docker:123456@docker-registry.com/v2/nginx/tags/list
{"name":"registry","tags":["2"]}
pull测试

$ docker logout https://docker-registry.com
Remove login credentials for https://docker-registry.com
#不登陆registry直接pull镜像也会失败
$ docker pull docker-registry.com/registry:2
Pulling repository docker-registry.com/registry
Error: image registry:2 not found
#登陆后再测试
$ docker login https://docker-registry.com
Username: docker
Password:
Email:
WARNING: login credentials saved in /root/.docker/config.json
Login Succeeded
#登陆后可以pull
$ docker pull docker-registry.com/registry:2
1.9: Pulling from dev-docker-registry.com/registry
6d1ae97ee388: Already exists
8b9a99209d5c: Already exists
3244b9987276: Already exists
50e5c9c52d5d: Already exists
146400830f31: Already exists
b412cc1cde63: Already exists
7fe375038652: Already exists
c43f11a030f9: Already exists
152297b50994: Already exists
01e808fa2993: Already exists
813e3731b203: Already exists
Digest: sha256:af688d675460d336259d60824cd3992e3d820a90b4f31015ef49dc234a00adc3
Status: Downloaded newer image for docker-registry.com/registry:2
```

# CentOS 7安装Docker及常用命令
```
yum install docker    #安装docker
sudo systemctl daemon-reload    #启动docker-daemon
sudo systemctl restart docker     #重启启动docker
systemctl start docker.service     #启动docker
systemctl enable docker.service    #docker开机启动
docker -v     #查看docker版本
docker info     #查看docker具体信息
docker pull centos     #下载centos image
docker images     #显示已有image
docker rmi  imageid     #删除image
sudo usermod -a -G docker wisely     #非root用户使用
docker run -i -t centos /bin/bash     #启动系统
docker stop $(docker ps -a -q)     #停止所有容器
docker rm $(docker ps -a -q)      #删除所有container
docker rmi $(docker images -q)     #删除所有image
docker inspect container_name     #查看容器信息
docker inspect container_name | grep IPAddress     #查看当前容器ip地地址
docker attach --sig-proxy=false 304f5db405ec      (按control +c 退出不停止容器)
```
---
参考文档：

[使用SSL验证和Nginx做代理搭建生产环境的Docker仓库](https://blog.csdn.net/Tomstrong_369/article/details/51145467)

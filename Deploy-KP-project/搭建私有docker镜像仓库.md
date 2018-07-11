一、安装Docker
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


二、安装Docker Registry
目前Docker Registry已经升级到了v2，最新版的Docker已不再支持v1。Registry v2使用Go语言编写，在性能和安全性上做了很多优化，重新设计了镜像的存储格式。 先切换为root用户，再执行下面命令：
 `# curl -L https://github.com/docker/compose/releases/download/1.5.2/docker-compose-`uname -s`-`uname -m` > /usr/local/bin/docker-compose`
 `# chmod +x /usr/local/bin/docker-compose`
 `# ln -s /usr/local/bin/docker-compose /usr/bin/docker-compose`

安装htpasswd
`# sudo yum install httpd-tools`

三、运行Registry Container并使用Nginx做代mv理
# 运行nginx和registry容器
　　创建一个工作目录，例如/data/programs/docker，并在该目录下创建docker-compose.yml文件，将以下docker-compose.yml内容复制粘贴到的docker-compose.yml文件中。
`# sudo mkdir -p /data/programs/docker`
`# cd /data/programs/docker/`
`# sudo mkdir data && sudo mkdir nginx`

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

# 配置nginx
在nginx目录中创建registry.conf文件配置nginx。配置nginx与registry的关系，转发端口，以及其他nginx的配置选项。复制，粘贴如下内容到你的registry.conf文件中：
`# cat /data/programs/docker/nginx/registry.conf`
```
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
`# htpasswd -c registry.password docker`
New password:
Re-type new password:
Adding password for user docker

然后修改Registry.conf文件，取消下面三行的注释。
```
auth_basic "registry.localhost";
auth_basic_user_file /etc/nginx/conf.d/registry.password;
add_header 'Docker-Distribution-Api-Version' 'registry/2.0' always;
```

再次执行docker-compose up运行registry，这时使用localhost:5000端口访问得到的结果为”{}”,但是使用localhost:443访问将得到”401 Authorisation Required“的提示。加入用户名和密码验证才能得到与直接访问registry 5000端口相同的结果。
`curl http://localhost:5000/v2/`
{}

`curl http://localhost:443/v2/`
<html>
<head><title>401 Authorization Required</title></head>
<body bgcolor="white">
<center><h1>401 Authorization Required</h1></center>
<hr><center>nginx/1.9.15</center>
</body>
</html>

四、加入SSL验证







注意事项：
Docker挂载主机目录Docker访问出现Permission denied的解决办法 https://blog.csdn.net/rznice/article/details/52170085

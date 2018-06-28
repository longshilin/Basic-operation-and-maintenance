# 基于deploy项目的部署流程
1. 从aws-git上pull项目源码，aws-git地址：https://git-codecommit.us-east-1.amazonaws.com/v1/repos/yunying
2. 通过eclipsIDE来导入项目源码
3. 对于maven依赖的报错，需要删除本地仓库中的依赖包，然后更新pom.xml文件重新下载资源包
4. 项目默认是通过/deploy/src/main/resources/config/dev/application.properties配置文件进行启动的，当配置项确实配置时，项目会启动失败，这时新增配置项即可（这里可以参考na区域的配置文件）。
5. 项目可以通过Tomcat8+Jdk1.8来启动。

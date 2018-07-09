# 基于deploy项目的部署流程（完整）及注意事项
1. 从aws-git上pull项目源码，aws-git地址：https://git-codecommit.us-east-1.amazonaws.com/v1/repos/yunying
2. 通过eclipsIDE来导入项目源码，这边不用IDEA，因为项目存在兼容性问题。
3. 对于maven依赖的报错，需要删除本地仓库中的依赖包，然后更新pom.xml文件重新下载资源包
4. 项目默认是通过/deploy/src/main/resources/config/dev/application.properties配置文件进行启动的，当配置项确实配置时，项目会启动失败，这时新增配置项即可（这里可以参考na区域的配置文件）。
5. 项目可以通过Tomcat8+Jdk1.8来启动。

# KP项目架构在deploy中的部署流程
## 前期准备
1. deploy的web环境搭建好，包括Tomcat环境，
2. deploy的数据库创建好，需要支持deploy整个项目。配置machine表，包括所有的node机信息
3. deploy相关的脚本远程创建环境以及机器人Python脚本的导入环境。
4. 远程Docker镜像仓库的访问 registry-kp.90km.com
5. swarm集群环境的搭建
6. node机上的游戏数据库中，需要将kpchatdb和kpserverlistdb数据库都搭建好（表结构的创建）

## 部署步骤
1. 在本地进行调试部署：
首先需要解决deploy中，有两个地方需要连接线上服务的地方，因此需要将deploy所在主机的ip地址加入到线上请求机器的安全组中，才能获取到请求的信息，否则代码运行会报错 URL连接失败等。

2. 通过开服向导进行整体部署（准备工作）:
  1. 首先在node机器上安装redis和mysql数据库。安装文档查看之前的整理。
  2. 在deploy数据库表中对这两个服务进行相应的依赖配置等。包括service_config和service_machine依赖关系进行配置
  3. 对deploy数据库表中的memcached服务进行相应配置，其IMAGE_TAG标签设为：1.4.15-9，并在Image表中设置其镜像的相应信息，手动增加memcached版本。这样在deploy前台真正启动之后，会自动根据image表中设置的资源地址进行下载镜像并进行安装
  4. 注意deploy数据库中的channel表是有增长序号的，设置一个序号 在deploy前台开渠道的时候会用上。

3. 开始进入正式配置
  1. 点击 开服向导->新增/维护全局服务 选择各个服务组件的版本，这个是通过线上的镜像仓库来获取的。最后点击保存，开始全局服务的创建。在创建全局服务时，后台会进行的有deploydb中，表关系的建立，其中涉及服务依赖的建立以及各服务端口号的建立。
  + 修改的数据库表信息：deploydb中service_config service_db service_memecached service_redis service_TD表进行修改，涉及到这些服务的依赖配置

  2. 点击 开服向导->渠道维护 这个界面配置的是 kpserverlistdb 中的channel_list表和server_list表，其中channel_list表是通过前台输入的，server_list表是手动配置进去的。除了配置serverlist相关属性外，还需要设置GameProxy、GameServer以及Third版本，这里设置成功后，在node机上就会自动建立所有的容器并启动。
  + 修改的数据库表信息：deploydb中service_config service_db service_memecached service_redis service_TD表进行修改，涉及到这些服务的依赖配置。kpserverlistdb数据库中channel_list表的修改以及server_list的手动新增。

  3. 点击 开服向导->开新服 进行服务器的搭建，其中包括proxy和KP两个服务组件，这就是一个游戏服的标准配置，在；开新服界面上，区服ID是根据在server_config表中进行插入的，其中会关联此服务器属于哪一个渠道号中等信息。最后设置proxy版本和kp版本号，点击保存即可开始proxy和kp服务组件容器的创建。其中在创建kp容器时，会先进行机器人插入，要保住玩家数据库中的USER机器人数量是10000时，才表示插入机器人成功，也才可以进行下一步操作。`SELECT COUNT(*) FROM `USER` WHERE USER.IS_ROBOT=1;`
  + 修改的数据库表信息：deploydb中server_config service_config service_db service_memecached service_redis service_TD表进行修改，涉及到这些服务的依赖配置。

4. 补充信息：
** 当其中某一个操作出现错误时，需要将这个步骤中所涉及的表信息都回滚到之前的状态上。保证数据的健全性 **

5. 客户端测试：
现在在deploy中新开了两个游戏服:ZondId分别是8001和8002
因此在客户端的登录界面，用户名随机，需要输入的服务器IP是Node机的共有IP：18.217.50.73，其端口是gateway服务的对外端口号：9150 ZoneId填写：8001 / 8002

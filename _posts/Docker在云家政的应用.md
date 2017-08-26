title: Docker在云家政的应用
date: 2017/8/26  11:36
categories: 技术分享
---

本文是前几天在同学群里的分享，经过整理后分享出来。

---
大家晚上好，我是Fighter，目前在云家政担任高级运维开发，工作7年多了，对电商架构、运维自动化开发有一些经验。今晚由我给大家分享《Docker在云家政的应用》
首先我介绍一下公司的背景，公司属于中小型创业公司，服务器数量不多，但是为了解决一些问题，我们引入了现在比较火的Docker技术。目前公司大规模使用了Docker，目前除了数据库应用，其他所有应用都在Docker容器内运行，下面我就Docker在公司的应用做一些分享。 

{% img [图片] /images/docker/docker-500.jpg %}

上面这个报错大家应该也都见过。程序错误了，我们公司以前也会出现这个错误。
**那么我们在没用Docker之前都遇到了哪些问题呢：**
1、线上环境和测试环境不完全一致，导致测试好的功能上线后会出现一些BUG。
2、部署新项目步骤繁琐，批量部署运行环境后，需要根据每个项目不同的情况，手动修改配置参数。
3、新项目环境部署耗费时间长。有些项目部署需要几十分钟甚至更长时间。
4、操作系统版本的差异，导致批量部署遇到麻烦。
5、不能跨平台部署环境。

这就是我们的现状，正是有了这些问题，我们就要解决这些问题。这里我再简单对Docker做一下介绍：

{% img [图片] /images/docker/docker-jieshao.jpg %}
{% blockquote %}
Docker是一个新的容器化开源项目，诞生于 2013 年初，最初是 dotCloud 公司内部的一个业余项目，项目后来加入了 Linux |基金会，遵从了 Apache 2.0 协议，基于 Google 公司推出的 Go 语言实现。
 
Docker 提供了一个可以运行你的应用程序的容器,它可以将应用以及依赖包到一个可移植的容器中,然后发布到任何 Linux机器上；

Docker 扩展了 Linux 容器（LinuxContainers）通过一个高层次的 API 为进程单独提供了一个轻量级的虚拟环境，有点类似虚拟机的概念。
{% endblockquote %}

了解了Docker后，接下来看我们是怎么把Docker用起来的，这里容我再介绍一下公司的背景，公司属于中小型创业公司，服务器数量不多，没有用高大上的Kubernetes、Swarm等Docker集群管理工具，利用Python开发运维平台实现Docker的自动化管理。

----
**我们都知道为了方便Docker的部署，一般都需要一个Docker私有仓库来存放镜像，我们也有自己的私有仓库。**
看一下我们公司的私有镜像仓库是什么样子的，里面都存放了哪些镜像。
{% img [图片] /images/docker/docker-registry.jpg %}
我们的镜像仓库里面存放了应用服务镜像，如Apache，Nginx等。
API服务镜像。
NoSQL镜像：如Redis服务，MongoDB服务，ES服务等；
这些镜像都是根据我们自己的实际需要打包好的环境镜像，新项目需要什么服务，直接拉取私有仓库的镜像，快速的部署。

**有了镜像仓库，看一下我们是怎么制作镜像的？**
我们使用了Dockerfile制作镜像，每个环境都有对应的Dockerfille文件，可以根据实际需要随时调整镜像。

以我们其中一个应用服务环境镜像为例（Nginx+php），看一下我们的镜像制作过程：
{% img [图片] /images/docker/docker-registry-build.jpg %}
1、从Docker官方镜像仓库拉取PHP5.6作为基础镜像；
2、基于基础镜像安装Nginx以及PHP需要的扩展；
3、修改Nginx和PHP的配置；
4、生成指定服务的专用镜像；
5、将生成好的镜像提交至私有仓库；

**看一下公司的Dockerfile文件及构建镜像的命令：**
```
# cat Dockerfile
FROMphp:5.6.31-fpm
|RUN apt-get update&& apt-get install -y \
    nginx \
    libfreetype6-dev \
    libjpeg62-turbo-dev \
    libmcrypt-dev \
    libpng12-dev \
    libxml2-dev \
    libssl-dev \
    git \
    vim \
    && pecl install redis mongodb mongo\
    && docker-php-ext-enable redismongodb mongo \
COPY./nginx_vhost_conf/* /etc/nginx/sites-enabled/
```

构建镜像命令：
```
# docker build –t  hub.yunjiazheng.com/front_web:v1.0 .   
```

提交镜像到私有仓库：
```
# docker pushhub.yunjiazheng.com/front_web:v1.0   
```

**接下来看一下我们如何利用镜像快速部署环境的。**
{% img [图片] /images/docker/docker-registry-env.jpg %}

- 首先，我们服务器在安装完操作系统，初始化系统的时候就会把Docker客户端安装好。
- 然后，服务器上只需要执行docker pull 拉取一个镜像。
- 最后执行docker run 启动镜像，就可以快速部署好一个需要的环境的。


执行docker部署的命令：
```
# docker pullhub.yunjiazheng.com/front_web:v1.0
# docker run –d –p80:80 hub.yunjiazheng.com/front_web:v1.0
```

我来解释一下上述两条命令
```
docker pullhub.yunjiazheng.com/front_web:v1.0 
从hub.yunjiazheng.com这个私有镜像仓库拉取front_web镜像，镜像版本是v1.0；

docker run –d –p80:80 hub.yunjiazheng.com/front_web:v1.0 
这条命令-d是在后端运行容器，-p是映射容器的80端口。然后启动容器；
```
{% blockquote %}
这样就部署好了一个需要的环境，大家看，是不是很easy。:-D
{% endblockquote %}

上面看了Docker部署环境的流程后，有一个问题，同一个镜像运行起来的容器如何区分测试环境和线上环境呢？
为了区分容器运行的环境，接下来要用到云家政的运维平台了。

{% img [图片] /images/docker/docker-ops.jpg %}
{% blockquote %}
云家政运维平台运维是自主开发的平台，平台集成了环境管理、配置管理、发布管理、任务管理、监控报警管理、等功能。
{% endblockquote %}
在环境管理会先创建好需要的多套环境，例如beta、线上，创建完环境后，会为每个环境添加不同的配置参数，然后发布的时候选择主机和镜像及要发布的环境就可以自动化部署一套环境。

**举个栗子指定服务器A部署A1项目的测试环境：**
运维平台自动登录A服务器，拉取A1项目需要的环境镜像，拉取A1项目代码，再拉取平台上为A1项目配置好的测试环境参数，然后启动容器就可以自动部署一套可运行的环境。

--------------------
来看一下我们环境管理的界面：
{% img [图片] /images/docker/docker-ops-manage.jpg %}

环境参数的管理界面：
{% img [图片] /images/docker/docker-ops-manage-env.jpg %}

对不同的环境 配置不同的参数。
{% img [图片] /images/docker/docker-ops-manage-env-key.jpg %}

运维平台里面的配置管理，可以在线管理线上、测试环境等配置信息，配置管理可以添加、删除、修改代码连接的数据库信息、redis信息等配置信息。
**实现逻辑大致如图所示。**
{% img [图片] /images/docker/docker-ops-manage-build.jpg %}

接下来看一下我们通过运维平台部署好的应用的界面：
{% img [图片] /images/docker/docker-ops-host-ok.jpg %}

{% blockquote %}
主机就是发布好的主机，版本是容器运行镜像的版本，状态是容器的运行状态，在这里可以对容器进行远程管理。
{% endblockquote %}
**公司目前使用情况：**

- 目前云家政所有服务除了数据库是直接运行在操作系统上，其他所有应用服务都实现了容器化，每个项目服务都有对应的镜像，可以在最快几秒内实现服务的快速部署。
- 运维平台通过调用服务器上Docker API接口实现对容器的启动、关闭、执行命令、更新镜像等自动化管理。

**那么引入Docker给我们又带来了什么好处呢？**

- 保证了运行环境的一致性，线上环境和测试环境使用同一个镜像，测试环境测试通过后，上线后不会出现因为环境差异而导致Bug；
- 部署新项目方便快捷，不用考虑操作系统的差异而导致自动部署失败；
- 新项目部署速度快，可在秒级部署好一个项目环境；
- 服务镜像制作完成后，可以多次快速部署，方便快速横向扩展服务；
- 支持跨平台部署；

这些解决了之前前面所说的问题。

目前我们公司运维平台因为一些功能还不完善，等完善后，后续会将运维平台开源。

------
以上是公司对Docker使用的一点分享，后续如果有机会可以分享一下我们的运维平台。

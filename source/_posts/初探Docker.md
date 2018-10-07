title: 初探Docker
date: 2016-07-07 16:57:28
tags: [Docker]
---
![docker](http://nnblog-storage.b0.upaiyun.com/img/docker.jpg)
Docker让开发人员易于快速构建可随时运行的容器化应用程序，它大大简化了管理和部署应用程序的任务。下面主要介绍一些基本指令和用法，由于主要是在Mac上使用，所以有些东西在Mac上才会涉及到。
<!-- more -->
## 安装（只适用于macOS）
1. 下载官网的DockerToolbox.pkg安装包安装，这里推荐在阿里云上的镜像http://mirrors.aliyun.com/docker-toolbox/mac/?spm=0.0.0.0.8EK1Sh 。
2. 使用brew安装
```
brew cask update
brew cask install docker
```

## 快速开始
```
docker-machine create --engine-registry-mirror=https://f9fd872q.mirror.aliyuncs.com -d virtualbox default
# 创建一台安装有Docker环境的Linux虚拟机，指定机器名称为default，同时配置Docker加速器地址.
```
记得一定要用加速器，这很重要！
```
docker-machine env default
eval "$(docker-machine env default)"
```
通过运行docker info指令，若能看到Containers、Running和Images等相关的信息，这就表明Docker环境搭建好了。

## 一些指令
docker ps `列出所有运行中的容器` <br />
docker ps -a `列出所有容器` <br />
docker images `列出镜像` <br />
docker rmi [Image] `删除一个镜像` <br />
docker rm [Container] `删除一个容器` <br />
docker stop [Container] `停止一个正在运行的容器`

**docker run [OPTIONS] IMAGE[:TAG] [COMMAND] [ARG...] 依附一个镜像运行一个容器**

OPTIONS：
- -d：分离模式，在后台运行容器，并且打印出容器ID
- -i：保持输入流开放即使没有附加输入流
- -t：分配一个伪造的终端输入
- -p：匹配镜像内的网络端口号
- -e：设置环境变量
- -link：连接到另一个容器
- -name：分配容器的名称，如果没有指定就会随机生成一个

Examples：
```
docker run -it ubuntu 启动一个依赖Ubuntu镜像并带有终端输入的容器

docker run --name db -d -e MYSQL_ROOT_PASSWORD=123 -p 3306:3306 mysql:latest 启动一个依赖mysql镜像的容器，-name db将容器命名为db；-d在后台运行；
-e MYSQL_ROOT_PASSWORD=123设置环境变量，参数告诉docker所提供的环境变量，这之后跟着的变量正是MySQL镜像检查且用来设置的默认root用户的密码；
-p 3306:3306告诉引擎用户想要将容器内的3306端口映射到外部的3306端口上。

docker exec -it db 告诉docker用户想要在名为db的容器里执行一个命令。

docker run -it -p 8123:8123 --link db:db -e DATABASE_HOST=db node4 使用link参数告诉docker我们想要连接到另外的容器上，link db:db连接
到名为db的容器上，并且用db指代该容器。
```
## 在工程中使用Docker
在项目工程中常常在根目录一个Dockerfile文件，用来构建项目的运行环境。下面是一个Node.js工程中Docerfile文件的示例。
```
FROM node:4  #指定基础镜像

ADD . /app  #将当前目录下的所有东西拷贝到名为app/的容器目录里

RUN cd /app;
	npm install --production  在镜像里运行命令

EXPOSE  3000 #将服务的端口暴露出来

CMD ["node"， "/app/index.js"]  #运行服务
```
有了这个Dockerfile文件后，只需在项目根目录执行docker build -t node4 .就可以在docker引擎中创建所需要的容器，并将代码放到容器中，-t node4表示使用标签node4标记该镜像，之后就可以使用这个标签来代指该镜像，.最后那个点表示在当前目录查找Dockerfile文件。<br />
通过执行docker run -it -p 8123:8123 node4就可以将该容器运行起来了。
## Docker Compose
通常一个Node.js工程都需要数据库，我们将数据库也放在一个容器里面，那个这个Node.js工程所在的容器需要连到数据库所在的容器。我们可通过另一个Dockerfile创建一个数据库容器，并初始化一些数据。<br />
这是创建数据库容器的Dockerfile
```
FROM mysql:5

ENV MYSQL_ROOT_PASSWORD 123  
ENV MYSQL_DATABASE users  
ENV MYSQL_USER users_service  
ENV MYSQL_PASSWORD 123

ADD setup.sql /docker-entrypoint-initdb.d #任何添加到镜像的 /docker-entrypoint-initdb.d目录的.sql或者.sh文件会在搭建DB的时候执行。
```
docker build -t test-database .  <br />
docker run --name db test-database <br />
这样需要通过docker run -it -p 8123:8123 --link db:db -e DATABASE_HOST=DB node4将node4容器连接到数据库容器，才能在Node.js工程中，访问数据库。<br />
这样看似解决了问题，但如果Node.js工程还依赖Redis、RabbitMQ和MongoDb等，如果是手动来处理这些容器的连接，简直是费时费力，这时候就可以用docker-compose来构建一次构建多个容器间的连接。<br />
Docker Compose需要一个docker-compose.yml文件，下面是一个示例：
```
version: '2'  
services:  
  users-service:
    build: ./node4
    ports:
     - "8123:8123"
    depends_on:
     - db
    environment:
     - DATABASE_HOST=db
  db:
    build: ./test-database
```
通过执行`docker-compose build`指令，就可以构建docker-compose.yml文件里列出的每个镜像，通过`docker-compose up`将构建的镜像运行起来，`docker-compose down`停止运行的镜像。<br />
在docker-compose.yml中，每个servies下的每个服务的build值告诉到哪里找到Dockerfile。<br />

*参考：http://blog.jobbole.com/103069/*

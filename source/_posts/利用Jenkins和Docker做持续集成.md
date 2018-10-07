title: 利用Jenkins和Docker做持续集成
date: 2016-07-13 15:12:16
tags: [Docker, Jenkins]
---
![docker_jenkins](http://nnblog-storage.b0.upaiyun.com/img/docker_jenkins.jpg!watermark1.0)
持续集成（Continuous integration，简称CI）是一套标准的互联网软件开发和发布流程，主要指频繁的将代码集成到主干，让产品可以快速迭代，同时能保证高质量。
<!-- more -->
## 什么是Jenkins
Jenkins是一款由Java开发的开源软件项目，旨在提供一个开放易用的软件平台，使持续集成变成可能。<br />
利用Jenkins持续集成Node.js项目之后，就不用每次都登录到服务器，执行pm2 restart xxx或者更原始一点的kill xx，然后node xxx。通过Jenkins，只需单击『立即构建』按钮，就可以自动从Git仓库拉取代码，然后部署到远程服务器，执行一些安装依赖包和测试的命令，最后启动应用。如果需要部署到多台服务器上，只需在Jenkins上多配置相应的服务器数量，就可以通过Jenkins部署到多台服务器上。

## 通过Docker安装和启动Jenkins
Jenkins需要Java环境，有了Docker这个利器，我们就省去了安装Java环境的麻烦，只需执行如下命令即可。
```
docker pull jenkins:latest
```
有一个通常的做法是要将Jenkins文件存储地址挂载到主机上，万一Jenkins的服务器重装或迁移，可以很方便的把之前的项目配置保留。所用可通过如下命令来启动Jenkins容器。
```
docker run -d --name myjenkins -p 49001:8080 -v ${pwd}/data:/var/jenkins_home jenkins
```
上面安装和启动Jenkins容器的做法，常常会出现错误，错误日志如下。
```
touch: cannot touch ‘/var/jenkins_home/copy_reference_file.log’: Permission denied
Can not write to /var/jenkins_home/copy_reference_file.log. Wrong volume permissions?
```
这里推荐直接从 https://github.com/denverdino/docker-jenkins 获得相关代码，并构建自己的Jenkins镜像。执行命令如下：
```
git clone https://github.com/AliyunContainerService/docker-jenkins
cd docker-jenkins/jenkins
docker build -t myjenkins .
```
然后基于新镜像启动Jenkins容器。
```
docker rm -f jenkins
docker run -d -p 8080:8080 -p 50000:50000 -v $(pwd)/data:/var/jenkins_home --name jenkins myjenkins
```
用`docker-machine ip`获取到docker的IP后，用浏览器访问这个IP的8080端口，就会有如下图所示的Jenkins的主界面。
![docker_homepage](http://nnblog-storage.b0.upaiyun.com/img/jenkins_homepage.jpg)

## 通过Jenkins自动化部署Node.js项目
要实现自动化部署，需要在Jenkins中安装Git Plugin和Publish Over SSH这连个插件。<br />
插件安装完成后重启Jenkins，进入系统管理→系统设置来对插件进行简单的配置，增加远程的服务器配置。如下图所示，填入我们待部署的生产服务器的IP地址、SSH端口及用户名、密码等信息。如果远程服务是通过key来登录的，那么还需要把key的存放路径写上。
![docker_jenkins_ssh](http://nnblog-storage.b0.upaiyun.com/img/jenkins_remotehost.jpg)
回到Jenkins主页，单击左上角的『新建』按钮就可以开启一个新项目，给项目起名node_test，选择创建一个自由风格的软件项目，单击「OK」按钮，就进入了此项目的创建页面。<br />
在配置页，我们找到『源码管理』选项，配置好Github上的源码地址。
![docker_github_code](http://nnblog-storage.b0.upaiyun.com/img/jenkins_git.jpg)
然后单击「Add」按钮，配置Github账号，Jenkins就是通过这个账号来拉取源代码的。
![docker_github_account](http://nnblog-storage.b0.upaiyun.com/img/jenkins_git_account.png)
在『构建』一栏，单击下拉菜单，选择Execute shell，Jenkins从Github获取代码代码是自动执行的，这一步主要是将最新的代码打包，如下图。
![execShell](http://nnblog-storage.b0.upaiyun.com/img/execShell_1.png)
将代码打包后需要把代码发送的远程的生产服务器上，这时需要选择「Send files...」选项，需要填入的指令如下图。
![sendFile](http://nnblog-storage.b0.upaiyun.com/img/sendFile_1.png)
在Source files一栏中将刚刚打包的文件名node_test.tar.gz填入<br />
在Remote directory一栏中，填写发送代码包的远程保存地址，我们在这里写入var/，在远程主机上代码包的路径就是/var/node_test.tar.gz<br />
Exec Command一栏中的命令的主要作用是：
1. 删除之前的容器。
2. 删除原来项目的代码。
3. 解压新代码。
4. 在容器中安装新代码的依赖包，然后将真个应用启动起来。
5. 删除发送过来的代码包。
<br />

至此项目配置完成，保存之后回到首页，单击左侧的『立即构建』按钮，如果小球变为蓝色的话就是构建成功，我们的项目也就部署成功了。

*参考：https://yq.aliyun.com/articles/53990*、*《Node.js实战（第二季）》*

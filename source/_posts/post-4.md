---
title: docker和git的使用
date: 2018-09-02 14:58:26
tags:
---
#### docker是什么
docker是一种虚拟技术，和虚拟机差不多，但是没有虚拟机这么臃肿，因为虚拟机需要运行一个操作系统，需要宿主机更多的资源，而docker提供一个隔离的环境，它有一个守护进程，根据需要向宿主机申请资源。如果把宿主机比作一片大海，那么docker就是航行在海上的巨轮，而docker容器中的镜像则是巨轮上的货物，而docker则为这些货物提供了一个隔离外部环境的空间
<!-- more -->
#### docker优点
- **便捷**docker容器里面可以运行不同的镜像，根据不同需要将不同镜像进行组合，定制一个开发需要的环境，也方便开发完成后一键部署到线上
-  **灵活**，镜像在刚拉下来的时候是初始的基础镜像，docker每运行一次RUN命令就会生成一个新的层级去覆盖原有的内容，所以可以根据不同情况进行对基础镜像进行修改，而这个也是docker可以方便组合出不同环境的原因
- **轻量**，docker不像虚拟机需要运行整个操作系统，只要在运行时根据需要向宿主机申请资源即可

#### 使用docker
- 首先确定系统打开了HypeV虚拟化，如果没打开需要进入bios中将虚拟化打开
- 对于window系统，首先去官网上将docker for window下载下来，然后安装之后右下角出现一个小鲸鱼就是成功启动了
- 默认用的是国外的镜像拉取，可以在Daemon面板的Registry mirrors中设置阿里云的镜像
- 我们可以用```docker pull```命令拉取镜像，```docker rmi```命令删除镜像
- 我们可以把需要的命令和操作提前写在dockerfile中，然后再运行，这样能把需要的镜像和资源一键拉取，定制docker images
```dockerfile```
FROM images:tag #必须以FROM这个作为第一行开头，images表示的是启动的镜像，tag表示的是版本，如果本地没有该镜像那么会从网上拉取，如果未写明tag,则会拉取latest版本

EXPOSE 8888 #EXPOSE关键字暴露Docker服务端容器对外映射的本地端口，容器和宿主机通信的通道

RUN apt-get update && apt-get install -y vim cron ssh
#执行每条RUN指令将在当前镜像基础上执行指定命令，并提交为新的镜像，后续的RUN都在之前RUN提交后的镜像为基础，这个就是docker的镜像分层

RUN mkdir -p /work #这里是在docker中创建一个work路径
WORKDIR /work #等同于cd的功能，切换到work路径下

RUN mkdir -p /root/.ssh	#在root路径下创建.ssh文件
COPY ./conf/.ssh /root/.ssh  #复制.ssh文件到docker容器中
RUN ssh-keyscan -t rsa XXX >> /root/.ssh/known_hosts #生成用于ssh连接的秘钥

COPY ./server/requirements.txt /work #从宿主机复制文件到docker容器的work路径中
RUN pip install -r requirements.txt #根据文件中列举的py库进行加载
```
- dockerflie可以帮助我们记录定制镜像的过程，定制完成后就是启动镜像了，那么如果我需要启动很多镜像的时候是不是得一个一个的敲命令呢？肯定不可能，我们可以用docker-compose进行配置，然后一键把所有镜像启动，这就构建出了一个服务，便于我们构建开发环境，线上部署等等。
- 例如开发的流程是，从浏览器发请求，后台接收到请求之后，向数据库查询，然后返回结果，而前端开发完后还需要进行build打包，而从浏览器发请求到后台之间，还会有nginx分发请求等等，那么我们就可以利用docker轻松的搭建出这个流程
```dockerfile```
version: '3.3'

services: #服务列表
  nginx: #nginx服务
    image: nginx #镜像名称
    ports: #端口映射，AA:BB，AA是宿主机端口，BB是docker服务端口
      - "8888:80"
    restart: on-failure #自动重启
    volumes:  #路径映射
      - ./nginx:/etc/conf.d
      - ./gitbook:/book
    links:
      - web
    depends_on:  #依赖，web开启后nginx才会开启
      - web
    container_name: nginx #容器名字
  web:
    image: 
    volumes:
      - ./:/workspace
    command: python /work/server/app.py #命令行输入
    restart: on-failure
    depends_on: #依赖于frontend和db服务
      - frontend
      - db
    container_name: server
  frontend: #前端
      image: node:6
      working_dir: /home/work #工作路径
      volumes:
        - ./frontend:/home/work
        - command: "npm run build" #node执行build命令打包前端资源
      container_name: frontend
      healthcheck:
        test: "exit 0"
  db:
    image: mysql:5.7
    container_name: db
    ports: #端口映射
      - "3307:3306"
```
从上面可以看出，docker-compose便利的地方，而且docker-compose和宿主机间的通信都是通过端口进行，整个流程就是，浏览器向8888端口发出请求，映射到容器的80端口中，然后转发到后台，查询数据库之后再返回，然后我们可以用navicat工具将sql数据从3307端口导入到容器的sql实例中

-----

#### git是什么
git可以用来做分布式版本控制，不同于基于中心化的版本控制，git记录数据的时候记录的是增量数据，即是改变的数据才会进行上传更新。

了解git首先从commit对象，branch，HEAD指针对象开始，每一次的commit都会产生一个commit对象，commit对象会有两个指针，一个指针指向的是file tree的根节点，然后每个file是tree的一个结点，每个文件夹则是另一个tree的根结点，commit的另一个指针则会指向最新的commit，根据commit不断向前推进形成一条链，commitA -> commitB -> commitC 以此类推，而每个commit有一个唯一的hash值标记，这条链就是一条branch分支，每个git文件必定会有一个master分支，而分支则是由分支指针确定，分支指针会指向各自分支的最新提交，而HEAD则是指向分支指针，代表着目前在哪条分支上进行操作。
```javascript
commitA -> commitB -> commitC -> commitD -> commitE -> commitF -> commitG -> commitH 
						^                                            \          ^
						|                                             \         |
		              master                    			         commitG1  BranchA			
																		^
																		| 
																	  BranchB <- HEAD


```

git的状态分为***untracked***未跟踪, 此文件没有加入到git库，***unmodify***文件未修改，即版本库中的文件快照内容与文件夹中一致***modified***文件已修改，仅仅是修改，可以通过git add可进入暂存staged状态, 使用git checkout则丢弃修改返回到***unmodify***状态，***staged***暂存状态，执行git commit则将修改同步到库中，这时库中的文件和本地文件又变为一致, 文件为***unmodify***状态. 执行git reset HEAD可以取消暂存

#### git的优点
- 用户只要把工程clone下来，那么就会有一份完整的记录保存在用户本地中，这就避免了中心化储存中如果中心服务器坏了或者其他情况导致文件损坏难以修复
- 即使是没有网络环境，用户也可以commit到本地中，等有网络环境再push到远程仓库中
- 每个开发者都可以维护自己的分支进行开发


#### git如何使用
- git add ，可以将untracked文件和modified的文件加入到暂存区中
- git commit ，可以将staged状态的文件提交到库中，文件状态变为unmodify
- git pull ，可以从远程仓库拉取资源并合并到本地仓库中，根据commit的hash进行合并
- git push，可以将本地commit的代码推到远程仓库中
- git diff，可以观测文件之间的差异
- git status，可以查看文件的状态
- git checkout，可以切换分支，也可以检出modified状态的文件，取消修改状态变为unmodify
- git clone，从远程仓库拉取代码
- git branch，可以查看分支情况，分支会储存在.git文件夹中的refs中
- git reset，可以恢复本地的代码，默认--fixed意思是将commit的文件去掉，--hard意思是将add和commit的文件去掉，--soft则是将add的文件去掉
- git log，可以查看远程仓库的commit情况，然后在里面获取hash值用于reset
- git reflog，可以查看本地的git操作情况
- git rebase，可以将不同的分支合并到一条分支上，结果只剩一条操作分支，被合过来的分支会根据commit把正在操作的分支commit向前推进，被合过来的分支还存在，只是没有被引用了
- git merge，也是将不同分支进行合并，但是结果还是会剩两条分支，然后两条分支根据最新的commit进行比较合并，在操作分支上形成新的commit
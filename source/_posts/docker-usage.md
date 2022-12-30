---
title: docker-usage
tags:
  - docker
description: >-
  在上一篇文章中，我们实现了一个简单的Container，在资源可见性与可用性上与宿主机隔离。在这篇文章中，我们将学习最流行的容器技术：Docker的基本使用。
cover: 'https://pic.imgdb.cn/item/63aec3ee08b683016379e600.jpg'
abbrlink: 394c83b4
date: 2022-12-28 14:20:12
categories:
---

# Docker简介

> Docker的基本结构（docker engine），container、image的关系，

在上一篇文章中，我们实现了一个简单的Container，在资源可见性与可用性上与宿主机隔离。不过我们的Container还非常简单，在这篇文章中，我们将学习最流行的容器技术：Docker的基本使用。

Docker的基本原理和我们的mini-docker没有什么区别，但是为了其稳定性、可用性等方面的考虑，应该也是做了很多设计。



首先我们先来了解一下docker的组成部分

![](https://pic.imgdb.cn/item/63aec2fe08b683016378e2ce.jpg)

Docker主要分为了三个部分：

- Client：用户与Docker交互的入口，接受用户命令，使用内部的API将命令转发给`docker daemon (dockerd)`
- Docker Host：作为Container最核心的功能实现
  - Docker daemon：负责接受命令，管理docker组件（Docker objects），以及与其他Docker daemon交互以便管理多个Docker服务。
  - Docker objects：Docker使用这些抽象来实现容器虚拟化和隔离功能。其中最主要的是images, containers, networks, volumes。这里先简单介绍他们的概念，后面在使用的过程中会详细介绍。
    - Images：构造Container的指令集，比如redis的images包含了安装、配置、启动redis的步骤指令。
    - Container：Images的运行实例，即一个运行中的程序，比如使用docker运行的redis服务器。
    - Networks：在同一个Network下的Containers可以直接交流，否则Container之间不可以互相直接通信。
    - Volumes：Container数据持久化的地方。
- Registry：存放Docker image 的仓库，[Docker Hub](https://hub.docker.com/) 是官方公开的仓库，里面有很多常用组件的image。我们也可以使用自己的私有仓库，比如AWS的[ECR](https://aws.amazon.com/cn/ecr/)。





# Docker Startup

在这一小节，我们将学习docker的基本命令，帮助我们快速上手使用docker。

并且会附带着介绍命令背后的逻辑，也是对docker中的组件的介绍，即在使用的过程中学习docker的构造。



假设为我们需要一个应用程序来操作MongoDB，即我们自己的app程序和一个MongoDB服务器。

[安装docker](https://docs.docker.com/get-docker/)之后，首先我们需要[启动dockerd](https://docs.docker.com/config/daemon/start/)，这是程序的入口。

由于官方仓库中已经有mongo的image了，我们直接`pull`就好了。

```bash
-> % docker pull mongo

Using default tag: latest
latest: Pulling from library/mongo
846c0b181fff: Downloading [=========>                                         ]  5.573MB/28.58MB
ef773e84b43a: Download complete
2bfad1efb664: Downloading [=====================>                             ]  3.578MB/8.348MB
84e59a6d63c9: Downloading [=>                                                 ]  40.98kB/1.235MB
d2f00ac700e0: Waiting
96d33bf42f45: Waiting
ebaa69d77b61: Waiting
aa77b709a7d6: Waiting
245bd0c9ace2: Waiting

# 查看所有本地下载的image
-> % docker images
REPOSITORY                                  TAG           IMAGE ID       CREATED        SIZE
mongo                                       latest        0850fead9327   2 weeks ago    700MB
```

我们看到docker正在从官方仓库下载mongo的image，并且有很多个任务，这是因为image是一层一层叠上去的。

通常，一个Image是基于另一个Image的，在此基础上带有一些额外的定制。 比如mongo的Image就是是以ubuntu的image为基底，并安装了mongodb构建的，从其[dockerfile](https://github.com/dockerfile/mongodb)可以看出（后面会介绍）。

前面介绍了，Image只是一份启动配置，相当于我们的程序代码，此时我们的mongodb还没有真正的运行，我们可以使用`docker run` 来运行Image，即创建一个Container。我们的程序启动时会传递一些参数，同样的，[docker run](https://docs.docker.com/engine/reference/run/)命令也可以传递一些通用的参数，这里介绍几个常用的:

```bash
docker run [OPTIONS] IMAGE[:TAG|@DIGEST] [COMMAND] [ARG...]

[OPTIONS] can be:
-d : Run container in background and print container ID
-p : Publish a container's port(s) to the host
     format: ip:hostPort:containerPort | ip::containerPort | hostPort:containerPort | containerPort
     e.g. `-p 8081:8080` means map the container's port 8080 to the host's 8081
-v : Bind mount a volume
-e : Set environment variables
--name : Assign a name to the container
--network : Connect a container to a network

# 启动mongo
-> % docker run --name demo-mongo -d mongo
fce3a38530b02cd4ac59be78bd5b95f7a6d0c88f2d48e9747935c0371f8e011f  #打印了container的hash—id

# 查看所有运行中的containers, 加上-a可以查看停止运行的container
-> % docker ps
CONTAINER ID   IMAGE     COMMAND                  CREATED          STATUS          PORTS       NAMES
fce3a38530b0   mongo     "docker-entrypoint.s…"   59 seconds ago   Up 59 seconds   27017/tcp   demo-mongo

```

Container运行的过程我们可能需要排查某些问题，`docker logs`和`docker exec`非常好用

```bash
# docker logs 可以查看container输出的运行日志
-> % docker logs demo-mongo
{"t":{"$date":"2022-12-29T13:32:31.702+00:00"},"s":"I",  "c":"NETWORK",  "id":4915701, "ctx":"-","msg":"Initialized wire specification","attr":{"spec":{"incomingExternalClient":{"minWireVersion":0,"maxWireVersion":17},"incomingInternalClient":{"minWireVersion":0,"maxWireVersion":17},"outgoing":{"minWireVersion":6,"maxWireVersion":17},"isInternalClient":true}}}
{"t":{"$date":"2022-12-29T13:32:31.707+00:00"},"s":"I",  "c":"CONTROL",  "id":23285,   "ctx":"main","msg":"Automatically disabling TLS 1.0, to force-enable TLS 1.0 specify --sslDisabledProtocols 'none'"}
{"t":{"$date":"2022-12-29T13:32:31.708+00:00"},"s":"I",  "c":"NETWORK",  "id":4648601, "ctx":"main","msg":"Implicit TCP FastOpen unavailable. If TCP FastOpen is required, set tcpFastOpenServer, tcpFastOpenClient, and tcpFastOpenQueueSize."}
{"t":{"$date":"2022-12-29T13:32:31.710+00:00"},"s":"I",  "c":"REPL",     "id":5123008, "ctx":"main","msg":"Successfully registered PrimaryOnlyService","attr":{"service":"TenantMigrationDonorService","namespace":"config.tenantMigrationDonors"}}
{"t":{"$date":"2022-12-29T13:32:31.897+00:00"},"s":"I",  "c":"NETWORK",  "id":23016,   "ctx":"listener","msg":"Waiting for connections","attr":{"port":27017,"ssl":"off"}}


# docker exec 可以在容器内运行一个命令，比如使用bash登录到container内部
-> % docker exec -it demo-mongo /bin/bash
root@fce3a38530b0:/# ls
bin  boot  data  dev  docker-entrypoint-initdb.d  etc  home  js-yaml.js  lib  lib32  lib64  libx32  media  mnt	opt  proc  root  run  sbin  srv  sys  tmp  usr	var
root@fce3a38530b0:/#
```



# Docker Networks: Container communications

前面我们已经运行了一个mongo的server，让我们加点料，给mongo加一个数据管理的后台，更方便数据的展示和操作。[mongo-express](https://hub.docker.com/_/mongo-express)是一个mongo的管理后台，也在官方仓库中有提供。

默认情况下，container之间是不能直接访问对方的服务的，不然也就失去了容器的隔离性。为了能让mongo-express能访问到mongo的server，我们需要把他们加入同一个Network。

```bash
# 首先创建一个network
$ docker network create mongo-net
9a7c90242755e05e0d82a9ec15a53b134d219141508493bb6c29fc9d6a099e51

# 查看所有的network
$ docker network ls
NETWORK ID     NAME        DRIVER    SCOPE
14d5dcb6cf3a   bridge      bridge    local
906aafb9e796   host        host      local
9a7c90242755   mongo-net   bridge    local
b9b96cc9ff14   none        null      local

# 启动mongo
$ docker run -d \
	--network mongo-net \
	--name demo-mongo \
	-e MONGO_INITDB_ROOT_USERNAME=root \
	-e MONGO_INITDB_ROOT_PASSWORD=password \
	mongo

# 启动mongo-express
$ docker run -d \
    --network mongo-net \
    --name mongo-express \
    -p 8081:8081 \
    -e ME_CONFIG_OPTIONS_EDITORTHEME="ambiance" \
    -e ME_CONFIG_MONGODB_SERVER="demo-mongo" \
    -e ME_CONFIG_MONGODB_ADMINUSERNAME="root" \
    -e ME_CONFIG_MONGODB_ADMINPASSWORD="password" \
    mongo-express
    
# 本地找不到会去官方docker hub找对应的image下载并运行
Unable to find image 'mongo-express:latest' locally
latest: Pulling from library/mongo-express

```

访问http://localhost:8081/ 便可以看到我们mongo的可视化界面了，即我们的mongo-express可以正常访问mongo服务了。

# Docker Compose: All-in-one

通过前面的方法我们已经能够通过命令行启动一个服务的container了，但是现实中我们一个Application可能会依赖很多个其他服务，MongoDB、Redis、MySQL等等，这样一个个的使用命令行参数启动未免有些麻烦了。

[Docker Compose](https://docs.docker.com/compose/)是官方提供的一个管理多个docker服务的工具，可以一建启动多个containers，所有服务的启动参数都配置在一个`yaml`文件中，可以使用`docker compose`命令进行多个containers的管理。

我们创建一个`mongo.yaml`文件，包含上面的两个container。

```yaml
services:
  demo-mongo:
    image: mongo
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: password
    ports: 
      - 27017:27017

  mongo-express:
    image: mongo-express
    restart: always
    ports:
      - 8081:8081
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: root
      ME_CONFIG_MONGODB_ADMINPASSWORD: password
      ME_CONFIG_MONGODB_SERVER: demo-mongo 
```

注意这里并没有指定network，这是因为docker默认为一个compose文件创建一个单独的network。

我们在`mongo.yaml`同级目录下使用`docker compose up`启动服务。

```bash
$ docker compose -f mongo.yaml up -d
[+] Running 3/3
 ⠿ Network docker_demo_default            Created
 ⠿ Container docker_demo-demo-mongo-1     Started
 ⠿ Container docker_demo-mongo-express-1  Started
 
$ docker ps
CONTAINER ID   IMAGE           COMMAND                  CREATED         STATUS         PORTS                      NAMES
8b55ae6c6661   mongo           "docker-entrypoint.s…"   4 minutes ago   Up 4 minutes   0.0.0.0:27017->27017/tcp   docker_demo-demo-mongo-1
e43ea72401e0   mongo-express   "tini -- /docker-ent…"   4 minutes ago   Up 3 minutes   0.0.0.0:8081->8081/tcp     docker_demo-mongo-express-1
```

再次访问http://localhost:8081/ 也可以看到我们mongo-express的界面了。

但是这里有个问题，我们之前创建的mongo记录丢失了，即每次重启container，其内部的数据都会丢失。

如果我们想要保留这些数据，这就需要使用到docker volumes了。

# Docker Volumes: Persist Data

我们的mongo image是基于ubuntu的基础image的，所有的数据库操作写入的文件都会储存在这个ubuntu的文件系统中，和我们mini-docker的mount file system类似，但当我们结束container之后，container相关的临时文件也会被清除。

[Volumes](https://docs.docker.com/storage/volumes/) 提供了链接host和container的文件path映射，我们将一个container内的目录bind到host后，所有对这个container目录的文件修改都会**直接对host的文件系统进行操作**，而不会写入container的内部文件系统中。

我们可以使用`-v`参数或者compose `yaml`配置volumes，docker提供了两种volumes，`named volumes` 和` bind mount`

|                  | Named Volumes                                      | Bind Mounts                    |
| :--------------- | :------------------------------------------------- | :----------------------------- |
| 使用格式示例     | ${volume_name}:${container_path}                   | ${host_path}:${container_path} |
| 文件在Host的位置 | Docker 管理  (`/var/lib/docker/volumes/` on Linux) | 用户参数指定${host_path}       |

`named volumes`的好处在于host文件的位置完全由docker自动管理，我们不需要自己维护各种索引映射，是比较推荐的一种使用方式。另外如果没有指定`${volume_name}`则被视为匿名卷，名字由docker生成的hash_value代替。

我们使用docker-compose配置mongo的volumes，然后重新启动mongo服务。

```yaml
services:
  demo-mongo:
    image: mongo
    environment:
      MONGO_INITDB_ROOT_USERNAME: root
      MONGO_INITDB_ROOT_PASSWORD: password
    ports: 
      - 27017:27017
    volumes: 
      - mongo-data:/data/db

  mongo-express:
    image: mongo-express
    restart: always
    ports:
      - 8081:8081
    environment:
      ME_CONFIG_MONGODB_ADMINUSERNAME: root
      ME_CONFIG_MONGODB_ADMINPASSWORD: password
      ME_CONFIG_MONGODB_SERVER: demo-mongo 
      
volumes:
  mongo-data:
```

```bash
# 先停止services
$ % docker compose -f mongo.yaml  down
[+] Running 3/3
 ⠿ Container docker_demo-mongo-express-1  Removed 
 ⠿ Container docker_demo-demo-mongo-1     Removed
 ⠿ Network docker_demo_default            Removed

# 修改mongo.yaml后重新启动，发现创建了一个volume
$ % docker compose -f mongo.yaml  up -d
[+] Running 4/4
 ⠿ Network docker_demo_default            Created 
 ⠿ Volume "docker_demo_mongo-data"        Created
 ⠿ Container docker_demo-mongo-express-1  Started
 ⠿ Container docker_demo-demo-mongo-1     Started
 
# 查看所有的volumes
$ % docker volume ls
DRIVER    VOLUME NAME
local     docker_demo_mongo-data

# 查看某个volume的详细信息
$ docker volume inspect docker_demo_mongo-data
[
    {
        "CreatedAt": "2022-12-30T08:44:03Z",
        "Driver": "local",
        "Labels": {
            "com.docker.compose.project": "docker_demo",
            "com.docker.compose.version": "2.6.0",
            "com.docker.compose.volume": "mongo-data"
        },
        "Mountpoint": "/var/lib/docker/volumes/docker_demo_mongo-data/_data",
        "Name": "docker_demo_mongo-data",
        "Options": null,
        "Scope": "local"
    }
]
```

如果这时你重新创建数据库记录，然后重启services，你会发现数据仍然保留着。



# Dockerfile: Build our own image

上面都是直接用的开源的images，如果我们想发布自己代码的一个image，并使用docker运行该程序应该怎么做呢？

答案就是`Dockerfile`！Dockerfile没有文件扩展名， 包含了Docker 用于创建image的脚本指令，其内包含了一条条的 **指令(Instruction)**，**每一条指令构建一层**（这个概念非常重要），因此每一条指令的内容，就是描述该层应当如何构建。

下面是一个简单的golang应用程序的Dockerfile示例

```dockerfile
# FROM指定基础镜像
FROM golang:alpine

# RUN命令：安装git
RUN apk --no-cache add git ca-certificates

# 指定工作目录
WORKDIR /go/src/github.com/go/helloworld/

# 拷贝app.go文件（from host）到当前目录（. in container）
COPY app.go .

# RUN命令：go build 编译文件，拷贝编译产物到 /root
RUN CGO_ENABLED=0 GOOS=linux go build -a -installsuffix cgo -o app . \
  && cp /go/src/github.com/go/helloworld/app /root

# 指定工作目录
WORKDIR /root/

# CMD执行默认启动命令，即启动编译好的文件
CMD ["./app"]
```

有这样一个Dockerfile之后，我们可以使用 `docker build -t go/helloworld:1 -f Dockerfile .`来编译镜像，之后就可以像启动其他image一样`docker run go/helloworld:1`使用docker启动程序了。

一个Dockerfile可能需要用到的指令有：

- **FROM：指定基础镜像**。所有的image都是是以另一个image为基础，并在其上进行定制。 `FROM`命令就是指定 **基础镜像**，一个 `Dockerfile` 中 `FROM` 是必备的指令，并且必须是第一条指令。Docker Hub中提供了很多基础镜像，比如`golang`，`ubuntu`等。除了选择现有镜像为基础镜像外，Docker 还存在一个特殊的镜像，名为 `scratch`。这个镜像是虚拟的概念，并不实际存在，它表示一个空白的镜像。如果你以 `scratch` 为基础镜像的话，意味着你不以任何镜像为基础，接下来所写的指令将作为镜像第一层开始存在。不以任何系统为基础，直接将可执行文件复制进镜像的做法并不罕见，比如对于 Linux 下静态编译的程序来说，其程序本身已经包含了所有需要的Linux lib。
- **RUN：执行命令**。如果你的基础镜像中有bash，那么你就可以执行bash相关的命令了。`RUN`有两种格式：
  - `shell`格式：`RUN`后面直接跟shell命令， `RUN echo 'Hello, Docker!' > ./hello.txt`
  - `exec`格式： `RUN ["可执行文件", "参数1", "参数2"]` ，`RUN ["echo", "Hello, Docker!"]`

- **COPY：复制Host文件到container目录**。同样是复制文件，`COPY`和`RUN copy <源路径>... <目标路径>`的区别在于，COPY是复制host文件到container，而`RUN copy xxx xxx`是在container内的文件操作。该命令通常用于复制源文件到container内，然后进行编译。

- **CMD：指定默认的容器主进程的启动命令**。如果`docker run image` 后面跟了参数的话，就会**替换**掉CMD的命令。对于容器而言，其启动程序就是容器应用进程，容器就是为了主进程而存在的，主进程退出，容器就失去了存在的意义，从而退出。

- **ENTRYPOINT：入口点命令。**`ENTRYPOINT` 的目的和 `CMD` 一样，都是在指定容器启动程序及参数。当指定了 `ENTRYPOINT` 后，`CMD` 的含义就发生了改变，不再是直接的运行其命令，而是将 `CMD` 的内容**作为参数**传给 `ENTRYPOINT` 指令。这在一些需要定制启动参数的情形下很好用。

- **ENV：设置环境变量**。这个指令很简单，就是设置环境变量而已，无论是后面的其它指令，如 `RUN`，还是运行时的应用，都可以直接使用这里定义的环境变量。

- **VOLUME：定义匿名卷**。为了防止运行时用户忘记将动态文件所保存目录挂载为卷，在 `Dockerfile` 中，我们可以事先指定某些目录挂载为匿名卷，这样在运行时如果用户不指定挂载，其应用也可以正常运行，不会向容器存储层写入大量数据。比如mongo就指定了默认的匿名卷`VOLUME ["/data/db"]`

- **EXPOSE：暴露端口**。`EXPOSE` 指令是声明容器运行时提供服务的端口，这只是一个声明，在容器运行时并不会因为这个声明应用就会开启这个端口的服务。在 Dockerfile 中写入这样的声明有两个好处，一个是帮助镜像使用者理解这个镜像服务的守护端口，以方便配置映射；另一个用处则是在运行时使用随机端口映射时，也就是 `docker run -P` 时，会自动随机映射 `EXPOSE` 的端口。

- **WORKDIR：指定工作目录**。使用 `WORKDIR` 指令可以来指定工作目录（或者称为当前目录），以后各层的当前目录就被改为指定的目录，如该目录不存在，`WORKDIR` 会帮你建立目录。由于Dockerfile的每一条命令都会创建一个新的image，因此下面这种写法，在执行第二条`RUN`命令时`pwd`并不在`/app`下。

  ```dockerfile
  RUN cd /app
  RUN echo "hello" > world.txt
  ```

# DockerHub: Share your images

目前 Docker 官方维护了一个公共仓库 [Docker Hub](https://hub.docker.com/)，大部分需求都可以通过在 Docker Hub 中直接下载镜像来实现。

可以通过执行 `docker login` 命令交互式的输入用户名及密码来完成在命令行界面登录 Docker Hub。

用户也可以在登录后通过 `docker push` 命令来将自己的镜像推送到 Docker Hub。

```bash
$ docker tag ubuntu:18.04 username/ubuntu:18.04

$ docker push username/ubuntu:18.04
```

如果你使用AWS的[ECR](https://aws.amazon.com/cn/ecr/)云服务来创建自己的私有仓库，操作也是类似的。首先需要登录到AWS registry云服务，然后打上tag，最后push。

`docker push`推送的地址是由tag确定的，tag格式为 `registryDomain/imageName:tag`，比如对于官方docker hub，`mongo:4.2` image的完整格式为`docker.io/library/mongo:4.2`

# Conclusion

本文只是对Docker的简单介绍，主要是docker的基本使用，对于原理部分没有深入。如果想了解更多信息，可以参考[官方文档](https://docs.docker.com/)和[Docker — 从入门到实践](https://yeasy.gitbook.io/docker_practice/)。

对于Docker的内部实现和源码将会在后续的文章中介绍，不过在此之前，可能会先学习一些`Kubernetes`的相关知识。


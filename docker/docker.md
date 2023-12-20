## Docker

### Docker是什么

#### 演化历史

硬件，操作系统和应用程序之间的关系：
```
+--------------------------+
|       Applications       |
+--------------------------+
|+------------------------+|
||    Runtime Library     ||
|+------------------------+|
||         Kernel         ||
|+------------------------+|
|     Operating System     |
+-----+--------+-----------+
| CPU | Memory | IO Device |
+-----+--------+-----------+
```

1. 随着硬件的发展，很多时候电脑的硬件性能其实是过剩的，即使无法完全满足峰值性能需求，但是大多数的时间是会有很大一部分资源是闲置的；
2. 不同的软件在同一个操作系统下面会有冲突；

为了解决不同的软件在同一个操作系统下面冲突的问题，只能配置多台计算机，或者在同一个计算机上面装多个操作系统，前者会导致性能的浪费，而后者切换起来非常麻烦。

由于硬件性能的提升，未来解决上述的问题，充分利用硬件的资源，所以出现了硬件虚拟化的概念。也就是通过软件仿真出计算机的硬件，这样就可以在这些虚拟出的硬件上面安装操作系统，即来宾操作系统。然后再在来宾操作系统上安装不同的软件。虚拟化的软件负责将软件对硬件的需求转发到真实的硬件中，如VMWare。

这类软件有一个专有名词Hypervisor，也就是Virtual Machine Monitor(VMM)。

根据访问硬件资源方式的不同，可以分为两类：
```
+-----+-----+-----+-----+
                             |App A|App B|App C|App D|
+-----+-----+-----+-----+    +-----+-----+-----+-----+
|App A|App B|App C|App D|    |Guest|Guest|Guest|Guest|
+-----+-----+-----+-----+    | OS0 | OS1 | OS2 | OS3 |
|Guest|Guest|Guest|Guest|    +-----+-----+-----+-----+
| OS0 | OS1 | OS2 | OS3 |    |        Hypervisor     |
+-----+-----+-----+-----+    +-----------------------+
|        Hypervisor     |    |         Host OS       |
+-----------------------+    +-----------------------+
|        Hardware       |    |        Hardware       |
+-----------------------+    +-----------------------+
          Type I                       Type II
```
一类是直接访问硬件资源，这种情况下，所有的虚拟机实际上是和当前操作系统所在的物理机是平级的，如Hyper-V，Xen；
另一类和普通的应用一样，运行在操作系统上，通过Host OS来访问硬件资源；
理论上，前者少了一层，性能会更好一点；

由于硬件虚拟化需要消耗大量的资源，因为不同的虚拟机之间资源是不共享的；而且大多数时候Guest Host类型实际上都是一样的，完全可以复用，只是运行库和应用不一样；所以为了复用操作系统，隔离运行库和应用，出现了操作系统虚拟化，也就是容器的概念。

物理机，容器和第二类虚拟机之间的区别：
```
+-----+-----+-----+-----+                                   +-----+-----+-----+-----+
|App A|App B|App C|App D|     +-----+-----+-----+-----+     |App A|App B|App C|App D|
+-----+-----+-----+-----+     |App A|App B|App C|App D|     +-----+-----+-----+-----+
|+---------------------+|     +-----+-----+-----+-----+     |Guest|Guest|Guest|Guest|
||   Runtime Library   ||     |Lib A|Lib B|Lib C|Lib D|     | OS0 | OS1 | OS2 | OS3 |
|+---------------------+|     +-----+-----+-----+-----+     +-----+-----+-----+-----+
||       Kernel        ||     |    Container Engine   |     |        Hypervisor     |
|+---------------------+|     +-----------------------+     +-----------------------+
|   Operating System    |     |         Host OS       |     |         Host OS       |
+-----------------------+     +-----------------------+     +-----------------------+
|       Hardware        |     |        Hardware       |     |        Hardware       |
+-----------------------+     +-----------------------+     +-----------------------+
    Physical Machine                  Container                 Type II Hypervisor
```

每个App和Lib结合，就是一个容器，也就是Docker里面的一个集装箱；
和虚拟机相比：
1. 迅速启动，因为没有虚拟机硬件的初始化，也不需要启动操作系统，也就是所谓的开箱即用；
2. 占用资源少，没有Guest OS占用更多的内存资源，也不需要提前预分配运行内存，也不需要提前安装运行App不需要的Lib；

但是由于共用内核，只靠cgroups进行隔离，所以不同应用之间的隔离是没有虚拟机那么彻底的，如果某个应用运行时导致内核崩溃，那么所有容器都会崩溃；而如果某个虚拟机里面的应用崩溃了，实际上是不会影响到其他虚拟机里面的应用的，除非虚拟机的Hypervisor有问题，或者是影响到了硬件；

Docker把App和Lib文件打包成一个镜像，采用类似多次快照的存储技术，如aufs/device mapper/btrs/zfs等，实现：
1. 多个App可以共用相同的底层镜像（初始的操作系统镜像）；
2. App运行时的IO操作和镜像文件隔离；
3. 通过挂载包含不同配置/数据文件的目录或者卷（Volume），单个App镜像可以同时用来运行无数个不同业务的容器；
```
+---------+  +---------+  +---------+    +-----+ +-----+ +-----+
| abc.com |  | def.com |  | xyz.com |    | DB1 | | DB2 | | DB3 |    
+----+----+  +----+----+  +----+----+    +--+--+ +--+--+ +--+--+    
     |            |            |            |       |       |
+----+----+  +----+----+  +----+----+    +--+--+ +--+--+ +--+--+    
|   abc   |  | def.com |  | xyz.com |    | DB1 | | DB2 | | DB3 |
| config  |  | config  |  | config  |    | conf| | conf| | conf|
|  data   |  |  data   |  |  data   |    | data| | data| | data|
+----+----+  +----+----+  +----+----+    +--+--+ +--+--+ +--+--+
     |            |            |            |       |       |
     +------------+------------+            +-------+-------+
                  |                                 |
           +------+------+                   +------+------+          
           | Nginx Image |                   | MySQL Image |
           +------+------+                   +------+------+
                  |                                 |
                  +----------------+----------------+
                                   |
                            +------+-------+ 
                            | Alpine Image |
                            +------+-------+
```

而且Docker提供公共的镜像仓库，Github Connect自动构建镜像，大大简化了应用分发、部署、升级的流程，而且Docker可以非常方便的建立各种自定义镜像文件；

所以绝大部分应用，开发者都可以通过docker build创建镜像，通过docker push上传镜像，用户通过docekr pull下载镜像，用docker run运行镜像。用户不需要关系如何搭建环境，如何安装，如何解决不同发行版的库冲突，而且通常不会消耗更多的硬件资源，不会明显降低性能，也就是所谓的标准化、集装箱；

除了docker以外，还有很多其他的容器；
windows由于使用微内核，而且内核和各种运行库耦合紧密，虽然从windows10开始也支持容器，但实际上还是通过Hyper-V运行不同的虚拟机进行内核级隔离，虽然也有线程级隔离，但是只有Windows Server支持，而且只能运行相同版本的镜像；即使是Hyper-V也支持运行更低版本的镜像而不能运行更高版本的镜像，而且Windows容器的镜像体积通常还是很大；

### Install Docker

[Docker desktop](https://docs.docker.com/desktop/)
Or
Colima(For mac)

Docker Engine
Docker Desktop(Daemon & CLI)


### First Docker project

init node project
```
npm init
```
install packages
```
npm install <package name>
```

run
```
node <main js file>
```

build image
```
docker build .
```

run container
```
docker run -p <localPort>:<dockerPort> <imageId>
```

list all running container
```
docker ps
```

stop container
```
docker stop <container-id>
```

error getting credentials - err: exec: "docker-credential-desktop": executable file not found in $PATH, out
```
rm ~/.docker/config.json
colima delete
colima start
https://stackoverflow.com/questions/65896681/exec-docker-credential-desktop-exe-executable-file-not-found-in-path
```

### Foundation

#### Images & Containers

```
+-----Container----+  +-----Container----+  +-----Container-----+  
|  Running NodeJS  |  |  Running NodeJS  |  |  Running NodeJS   |
|   App            |  |     App          |  |  App              |  
+--------+---------+  +---------+--------+  +----------+--------+ 
         |                      |                      |
         +----------------------+----------------------+          
                                |  
                                |
                    +---------Image----------+ 
                    |    NodeJS App Code     |
                    |    NodeJS Environment  |
                    +-----------+------------+
```

Image是运行程序的模板，包含运行环境，代码等；
Container是具体运行程序等实例，基于一个Image可以运行多个Containers；

show all containers include stopping status (ps: process 进程)
```
docker ps -a
```

run container and enter it(interactive 交互的)
```
docker run -it <imageId>
```

Dockerfiel
```
FROM node # 基于已经构建好的node镜像开始构建

WORKDIR /app # 指定工作路径，后续所有的命令都从这个路径开始执行

COPY . /app # 把容器之外的路径下的文件拷贝到容器内的对应app路径下，因为前面指定了工作路径(也可以写成./ 这样就是相对路径)

RUN npm install # RUN是构建镜像时执行

EXPOSE 80 # 暴露端口，只是声明暴露这个端口，如果运行时没有通过-p指定是无法访问的

CMD ["node", "server.js"] # CMD是运行容器时执行，只有最后一个会生效，可以通过命令行覆盖，因为这里执行的这个命令不会停止，所以容器会一直运行

```

##### layers

docker是基于layer的，所以如果每层没有什么变化，docker会缓存每层，这样构建docker image时，就会非常快；
Image是只读的；
Container相当于在原有的image上又加了一层Read-Write；
所以实际上存在于Image中的code和Environment即使创建了多个容器，实际上也只有一份，也就是保存在image中的那一份；

所以上面的代码进行一些优化
```
FROM node 

WORKDIR /app 

COPY package.json /app

RUN npm install 

COPY . /app

EXPOSE 80 

CMD ["node", "server.js"] 
```

因为依赖变动会更小一些，所以如果只是代码改变了，依赖没有改变，那么`npm install`层就可以使用缓存，从而加快构建速度；

##### Attached & Detached
默认的`run`命令是attached的，所以如果container一直运行，就会block当前命令行，将container里面命令行的输出打印到外面；
`-d`指令以detached模式运行，不会block当前命令行；
`attach`指令可以重新attach；
`logs`可以看到所有打印过的日志


##### deleting Images & containers

```
docker rm <containerId> <containerId2>
docker rmi <imageId>
```
##### copying files from container

```
docker cp <host path> <containerId>:<containerFile> # From to
docker cp <containerId>:<containerFile> <host path> # From to
```

##### sharing images

sharing `Dockerfile`，可能需要一些其他的文件
share a built Image，直接拉取Image就可以了
1. Docker hub 或者 Private Registry

share
```
docker push <imageName>:<version>
```
use
```
docker pull <imageName>:<version>
```

```
docker tag <oldName>:<version> <newName>:<version> # 不会删除旧的，会重新复制一个
docker login
docker logout
```

#### Data & Volumes

Application 只读，包含在Image中
Temporary App Data Read/Write，包含在容器层
Permanent App Data Read/Write，使用volumes

##### Volumes(Managed by docker)

```
docker volume ls
```
anonymous volumes 自动删除当容器停止运行时，如果指定了`--rm`参数
如果没有指定`--rm`参数，即使通过`docker rm <containerID>`删除了container，匿名volume也不会自动删除，可以通过`docker volume rm <volumeName>`or`docker volume prune`命令进行删除，为了保证容器的无状态性
named volumes 不会自动删除
必须要在运行时指定volume的name
```
docker run -p 3000:80 -d --rm --name fn1 -v feedback:/app/feedback feedback-node:1
```
下一次重新启动container的时候，就可以通过feedback这个volume的名字，将对应的volume挂载在对应的容器上
对于volume，不知道在电脑的哪个位置

##### Bind Mounts(managed by you)

可以绑定到host的某个具体文件夹下面，可以编辑
获取当前目录，直接以当前目录作为要复制的目录
```
macOS / Linux: -v $(pwd):/app

Windows: -v "%cd%":/app
```

路径最长的最先被使用，对于volume和Mounts

```
docker run -p 3000:80 -d --rm --name fn1 -v "/Users/xingkun.zhang/Documents/Study/study_codes_notes/docker/data-volumes-01-starting-setup:/app" -v /app/node_modules feedback-node:1
```
这里挂载了anonymous volumes和Bind mounted，虽然bind mounted可能会把node moudel覆盖掉，但是由于anonymous volume路径比较长，所以docker会优先使用这个；

anonymous volumes & named volumes & bind Mounts
```
docker run -v /app/data
docker run -v data:/app/data
docker run -v /path/to/code:/app/data
```

##### Read Only volumes

volume后面加`:ro`，表示只读
```
docker run -p 3000:80 -d --rm --name fn1 -v feedback:/app/feedback -v "/Users/xingkun.zhang/Documents/Study/study_codes_notes/docker/data-volumes-01-starting-setup:/app:ro" -v /app/temp -v /app/node_modules feedback-node:1
```

##### manage docker volumes

```
docker volume create <volumeName>
docker volume inspect <volumeName>
```

##### .dockerignore


##### Environment Variables & ".env"

`--env`指定环境变量
`--env-file` 指定含有环境变量的文件，例如`./.env`
```
docker run -p 3000:8000 --env PORT=8000 -d --rm --name fn1 -v feedback:/app/feedback -v "/Users/xingkun.zhang/Documents/Study/study_codes_notes/docker/data-volumes-01-starting-setup:/app:ro" -v /app/temp -v /app/node_modules feedback-node:1
```
`docker history <imageName>` 可以看到硬编码到docker中的环境变量，所以不要把敏感信息放到Dockerfile中

##### build arg

```
ARG

docker build -t feedback-node:1 --build-arg DEFAULT_PORT=8000 .
```

#### Containers & Networking

##### Container to www communication & Local machine & Container

与外网进行通信，比如连接数据库，访问网页等
`App -> Get www.some-api.com -> www`

`host.docker.internal` 在docker那边可以识别为host machine地址


不同容器之间通信，可以通过`host.docker.internal` 以及主机端口映射的方式，也可不开放端口，docker内部网络进行通信
```
docker run --rm -d --name mgDB -p 27017:27017 mongo
docker run --name fn1 -d --rm -p 3000:3000 favorities-node
```

```
docker network create f-net
docker run --rm -d --network f-net --name mgDB mong
docker run --name fn1 -d --rm --network f-net -p 3000:3000 favorities-node
```
容器里请求的host就是f-net

### Real Life

#### Multi-Container Projects

data must persist for db;
access should be limited for db;

data must persist for backend;
live source code update for backend;

live source code update for frontend;

```
docker network create goals-net
docker run --name mgdb --rm -d --network goals-net -v data:/data/db  mongo
docker build -t goals-node .
docker run --rm -d --network goals-net -p 80:80 --name gnb goals-node
docker build -t goals-react .
docker run --name grf --rm -p 3000:3000 -d goals-react
```


```
docker run --name mgdb --rm -d --network goals-net -e MONGO_INITDB_ROOT_USERNAME=admin -e MONGO_INITDB_ROOT_PASSWORD=secret mongo

docker run --rm -d -v /Users/xingkun.zhang/Documents/Study/study_codes_notes/docker/multi-01-starting-setup/backend:/app -v logs:/app/logs -v /app/node_modules --network goals-net -p 80:80 --name gnb goals-node

docker run --name grf --rm -p 3000:3000 -d -v /Users/xingkun.zhang/Documents/Study/study_codes_notes/docker/multi-01-starting-setup/frontend/src:/src goals-react
```

#### Using Docker-Compose

1. 不会改变Image
2. 不会替代Image或Container
3. 无法跨机器管理

```
docker-compose up -d --build
docker-compose down -v 
```

#### Utility Containers

Application Containers (Environmnet app) run app use cmd
Utility Containers (Environment) run custom cmd

```
docker run -it -d node
docker exec -it a26c2a npm init
docker run -it node npm init
docker build -t node-util .
docker run -it -v /Users/xingkun.zhang/Documents/Study/study_codes_notes/docker/utility-container:/app node-util npm init

docker build -t test .
docker run -it -v /Users/xingkun.zhang/Documents/Study/study_codes_notes/docker/utility-container:/app test install express --save

docker-compose run --rm npm install
```

#### More complex setup

```
docker-compose run --rm composer create-project --prefer-dist laravel/laravel .
docker-compose run --rm composer create-project --prefer-dist laravel/laravel:^8.0 .

docker-compose up -d --build server php mysq

docker-compose run --rm artisan migrate

```
 
#### Deploying Docker Containers

### Kubernetes

#### Kubernetes Introduction & Basics

#### Kubernetes: Data & Volumes

#### Kubernetes: Networking

#### Deploying a Kubernetes Cluster

### Docker Hub

### Docker compose

### Kubernetes


### COMMAND

```
docker ps
docker ps -a
docker images
docker build .
docker build -t test:latest . 
docker build -t feedback-node:1 --build-arg DEFAULT_PORT=8000 .
docker images -f dangling=true
docker run <imageId>
docker run -it <imageId>
docker run -p <hostPort>:<containerPort> --name <containerName> <imageId>
docker run -d <imageId>
docker run --rm <imageId> # 退出容器时自动删除
docker run -p 3000:80 -d --rm --name fn1 -v feedback:/app/feedback feedback-node:1
docker run -p 3000:8000 --env PORT=8000 -d --rm --name fn1 -v feedback:/app/feedback -v "/Users/xingkun.zhang/Documents/Study/study_codes_notes/docker/data-volumes-01-starting-setup:/app:ro" -v /app/temp -v /app/node_modules feedback-node:1
docker rm <containerId>
docker stop <containerId>
docker attach <containerId>
docker logs -f <containerId>
docker start -a <containerId>
docker start -a -i <containerId>
docker rm <containerId> <containerId2> ...
docker rmi <imageId> <imageId2> ...
docker image prune # 删除所有没有使用的images
docker image inspect <imageId> # 查看image相关的信息
docker cp <host path> <containerId>:<containerFile> # From to
docker cp <containerId>:<containerFile> <host path> # From to
docker push <imageName>:<version>
docker pull <imageName>:<version>
docker tag <oldName>:<version> <newName>:<version> # 不会删除旧的，会重新复制一个
docker login
docker logout
docker volume ls
docker-compose up -d --build
docker-compose down -v
--help

```

### Script
```
FROM
COPY . ./app
COPY . /app
WORKDIR /app
RUN 
CMD
ARG
ENV
```
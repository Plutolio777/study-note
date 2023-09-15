# Docker

## Docker命令

### Docker容器命令

docker容器运行后必须要有一个前台进程，容器运行的命令如果不是一个一直挂起如 `top` `tail`就是会自动推出的。

- 查看容器日志 `docker logs <CT>`
- 查看容器内运行的进程 `docker top <CT>`
- 查看容器运行细节 `docker inspect <CT>`
- 进入运行中的容器 `docker exec -it <CT> <CMD>`
- 另一种进入容器的方式 `docker attach <CT>`
  
  tips: `attach` 和 `exec` 进入容器的区别
  
  ~~~text
  attach 直接进入容器启动命令的终端，不会启动新的进程用exit退出，会导致容器停止

  exec 是在容器中打开新的终端，并且可以启动新的进程用exit退出，不会导致容器停止
  ~~~

- 从容器内拷贝文件都外部主机 `docker cp <CID>:<容器内目录> <目的主机路径>`
- 将容器内的文件系统到处到外部主机tar中 `docker export <CID> > <xxx.tar>`
- 将tar包中的内容创建一个新的文件系统导入到镜像中 `cat <xxx.tar> | docker import - <镜像用户>/<镜像名>:<镜像版本号>`

## 镜像

镜像是一种轻量级，可执行的独立软件包，包含运行某个软件的所有内容我们把应用程序和配置依赖打包好形成一个可交付的运行环境，这个打包好的运行环境就是image镜像文件

### 镜像分层

`docker` 依赖联合文件系统 `UnionFS` 来对镜像进行分层，联合文件系统是一种分层、轻量级且高性能的文件系统，它支持对文件系统修改作为一次提交操作来一层一层叠加，同时可以将不同目录挂载到同一个虚拟文件系统下，它是docker镜像的基础。镜像可以通过分层来继承 

#### Docker镜像加载原理




#### commit命令

docker commit 可以提交一个容器副本使之成为一个新的镜像

`docker commit <-m="description"> <-a="creator"> <CID>  repositories/name:tag` 

在ubuntu镜像中安装vim，并提交成新的镜像

~~~bash
docker pull ubuntu // 下载ubuntu镜像
docker run -it ubuntu /bin/bash //运行镜像
apt-get update //更新apt-get
apt-get -y install vim //安装vim

~~~

解决docker容器中因为源问题执行apt-get update以及安装失败问题 直接pull ubuntu:20.04无问题，最新版本的有问题 死循环apt解决不了


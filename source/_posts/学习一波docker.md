---
title: 学习一波docker
date: 2019-08-19 17:18:22
tags:
- 容器
- docker
---
# Introduction
> 
> 我们用的传统虚拟机如 VMware ， VisualBox 之类的需要模拟整台机器包括硬件，每台虚拟机都需要有自己的操作系统，虚拟机一旦被开启，预分配给它的资源将全部被占用。每一台虚拟机包括应用，必要的二进制和库，以及一个完整的用户操作系统。
> 
> 而容器技术是和我们的宿主机共享硬件资源及操作系统，可以实现资源的动态分配。容器包含应用和其所有的依赖包，但是与其他容器共享内核。容器在宿主机操作系统中，在用户空间以分离的进程运行。
> 
> 容器技术是实现操作系统虚拟化的一种途径，可以让您在资源受到隔离的进程中运行应用程序及其依赖关系。通过使用容器，我们可以轻松打包应用程序的代码、配置和依赖关系，将其变成容易使用的构建块，从而实现环境一致性、运营效率、开发人员生产力和版本控制等诸多目标。容器可以帮助保证应用程序快速、可靠、一致地部署，其间不受部署环境的影响。容器还赋予我们对资源更多的精细化控制能力，让我们的基础设施效率更高。通过下面这幅图我们可以很直观的反映出这两者的区别所在。

![][image-1]

可能目前了解的不够透彻，我理解docker的主要优势就是以下两点：
1. 资源隔离。拿VM做对比，如果我们都是部署一个服务，这个服务里某次上线出现了内存泄漏或是死循环这样的BUG打满了CPU和内存。使用VM部署的服务就会吃掉整台虚拟机的资源，而我们在VM上可能还部署了其他服务，这些服务都无法正常使用了。而如果是容器的话，只会影响到本服务分配的资源，其他服务都可以正常运行。
2.  镜像（创建容器的模板）封装了运行对应服务所需的环境和依赖，有了镜像，你可以像集装箱一样快速、可靠、一致地打包和部署你的应用程序，不需要再每次考虑由环境不一致带来的部署问题。

# Example

以macOS为例，我们在[官网][1]下载对应的dmg安装包，安装后首先换个镜像仓库地址，我这里用的是中科大的镜像：[https://docker.mirrors.ustc.edu.cn/][2]
这样在远程拉取镜像时速度会快一些。
常用的docker命令，可以参考这个[链接][3]

## 部署一个tomcat服务
简单实践一下在docker里部署一个tomcat的web服务。
首先拉取一个tomcat的镜像：
```js
docker pull tomcat
```

默认没有版本号会拉取latest版本。
```js
docker ps -a
```
会获取所有本地存在的镜像。

![][image-2]
获取镜像后，为该镜像启动一个容器：

```js
docker run -p 8080:8080 --name webdemo tomcat
```

用tomcat镜像启动一个名为webdemo的容器，将容器的8080端口映射为主机的8080端口。

```js
docker exec -it webdemo /bin/bash //进入容器内部开启一个交互式终端
```

```js
docker cp [LOCAL PATH] [CONTAINER]:/usr/local/tomcat/webapps/ //本地文件拷贝到容器内部
```
![][image-3]

```js
docker restart webdemo //重启容器
```

启动后，我们需要进入容器内部，把war包拷贝到webapps下面的目录，重启容器，容器会帮助解压war包并部署服务。启动成功后，就可以根据映射的端口地址正常访问部署的服务了。

[1]:	https://www.docker.com/products/docker-desktop
[2]:	https://docker.mirrors.ustc.edu.cn/
[3]:	https://www.runoob.com/docker/docker-command-manual.html

[image-1]:	https://myblog-1259548259.cos.ap-beijing.myqcloud.com/docker_structure.png
[image-2]:	https://myblog-1259548259.cos.ap-beijing.myqcloud.com/docker_psa.png
[image-3]:	https://myblog-1259548259.cos.ap-beijing.myqcloud.com/docker_exec.png
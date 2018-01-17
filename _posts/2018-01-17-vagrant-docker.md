---
layout: post
title:  vagrant与docker
date:   2018-01-17 23:05:00
category: "虚拟化"
keywords: docker vagrant 开发环境部署 运行环境部署
---

# vagrant与docker
## 区别
Vagrant和Docker都是虚拟化技术。Vagrant是基于Virtualbox的虚拟机来构建你的开发环境，而Docker则是基于LXC(LXC)轻量级容器虚拟技术。虚拟机重，容器虚拟技术轻。前者的Image一般以GB计算，Docker则以100MB为单位计算。简单点讲，Vagrant就是你的开发环境的部署工具；而docker是你的运行环境部署工具。

*Vagrant* 并不提供虚拟化技术，本质上是一个虚拟机外挂，通过虚拟机的管理接口来管理虚拟机，让用户更轻松的进行一些常用配置，比如：CPU/Memory/IP/DISK等分配。并且提供了一些其它的管理操作：比如开机运行指定命令，镜像二次打包，插件编写等等。  

vagrant官方有介绍
>To achieve its magic, Vagrant stands on the shoulders of giants. Machines are provisioned on top of VirtualBox, VMware, AWS, or any other provider. Then, industry-standard provisioning tools such as shell scripts, Chef, or Puppet, can be used to automatically install and configure software on the machine.

*docker*是一个容器引擎，每一个实例是一个相对隔离的空间，与宿主机共享操作系统内核，并且共享宿主机资源。相对于披着虚拟机皮的vagrant，docker更加轻量，消耗更少的资源。
贴一张docker官方介绍图

![docker_arc](/images/posts/vagrant_docker/docker_arc.jpg)

关于虚拟机和docker的区别这边文章有更形象的解释：[一篇不一样的docker原理解析 - uncle creepy的文章 - 知乎专栏](https://zhuanlan.zhihu.com/p/22382728)

## 应用场景
**关于应用场景没有绝对，把两个东西都用熟，自己觉得用哪个方便用哪个好管理就用哪个。**  

**vagrant**  
既然vagrant本质是虚拟机外挂，那么它的应用场景就是，节省你用原生虚拟机管理软件的时间。原来我们新增一台虚拟机需要配置好内存、硬盘、CPU等，然后添加iso，安装。创建用户，等等。一套下来好几十分钟是吧？聪明点你可能会想到复制一个创建好的镜像然后粘贴。但这一切vagrant都帮你想好了安装vagrant后你只需要6步就能创建一台新的虚拟机，其中两步是创建文件夹和切换文件夹

```
$ mkdir vagrant_getting_started
$ cd vagrant_getting_started
$ vagrant box add hashicorp/precise32
$ vi Vagrantfile
```
![vagrant](/images/posts/vagrant_docker/vagrant_conf.jpg)

```
$ vagrant init
```

从安装到创建一台新的虚拟机就成功了。如果你想要再添加一台虚拟机，你只需要执行最后两步，添加一个不同名字的配置就能再新建一台虚拟机。还支持镜像、开机自动运行脚本、插件编写等。

**docker**  
docker主要应用于解决环境依赖以及为应用程序提供一个相对隔离的空间，一个实例像操作系统里运行的一个程序。原来部署一套环境是不是得自己编写自动化部署依赖环境以及程序的脚本？如果有两个依赖同一程序或库的不同版本怎么办？绝对路径？软连接？docker能很好的解决你的烦恼。把需要的依赖环境打包成一个镜像，再把程序放镜像里面运行。

**总的来说**  
vagrant更适合给开发大爷们创造一个统一的开发、测试、接近于完全隔离的环境，以及提高对高配机的闲置利用。
docker更方便地解决了同一机器上的环境隔离，以及提高运维锅们解决部署时环境依赖的效率。

# 参考
[知乎用户](https://www.zhihu.com/question/32324376/answer/91562849)


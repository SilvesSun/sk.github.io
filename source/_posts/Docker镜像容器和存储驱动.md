---
title: Docker镜像容器和存储驱动
date: 2019-03-13 15:30:53
tags:
- 读书笔记
- 虚拟化技术
categories:
- Docker
---

为了更有效率的使用存储驱动, 必须立即 Docker 是如何构建和存储镜像的. 了解这些镜像是如何被容器使用, 以及一些有关镜像和容器操作的技术

## 镜像和数据层
一个镜像由一系列的数据层构成, 这些数据层除开最后一个外都是只读的, 每个数据层代表着 Dockerfile 中的指令. 以下面这个 Dockerfile 为例

```shell
FROM ubuntu:15.04
COPY . /app
RUN make /app
CMD python /app/app.py

```

上面这个 Dockerfile 包含4条命令, 每一条命令都会生成一个数据层. 如图所示

![](http://supcoder.net/dockerImages.png)

每一层相较于之前的一层, 仅仅只有一些不同. 这些数据层相互堆叠在一起. 当创建一个新的容器时, 将会在底层堆栈的顶部创建一个新的可写的数据层. 即容器数据层.
所有对正在运行的容器的修改, 如新增文件, 对已有文件修改, 删除文件等都会写到这个数据层

## 容器和数据层

容器和镜像间的主要区别是顶部的可写数据层. 所有容器数据新增, 修改都会保存到这个数据层. 容器删除时, 这个可写数据层也会被删除, 但是底层镜像保持不变.

由于不同容器各自拥有自己的可写数据层, 所以不同的容器可以在共用基础镜像的同时拥有自己的数据状态. 如图显示多个容器共享一个相同的Ubuntu 15.04镜像:

![](http://supcoder.net/sharing-layers.jpg)

**如果想要多个容器共享同一份数据, 将数据保存到 Docker 数据卷并将它挂载到容器上**

Docker 使用存储驱动来管理镜像数据层和可写容器数据层. 不同的存储驱动处理这两个数据层的方式有所不同, 但是都使用可堆叠镜像数据层和写时拷贝(copy-on-write)

## 容器大小



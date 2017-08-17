---
title: 为aws中国配置docker镜像加速
date: 2017-08-17 10:46:34
tags: [docker, AWS]
---

在AWS中国，docker镜像基本无法拉取，更换国内镜像是必须的。

# 更换镜像

## 修改docker配置文件

```
sudo vi /etc/sysconfig/docker
```

找到OPTIONS参数，在后面加上--registry-mirror=<国内镜像地址>并保存。

<!-- more -->

> OPTIONS="--default-ulimit nofile=1024:4096 **--registry-mirror=<国内镜像地址>**"

*注意*

*/etc/sysconfig/docker是Amazon Linux的docker配置文件位置，不同发行版的位置甚至文件格式都不太一样。*

*OPTIONS是Amazon Linux上用的参数，某些发行版会用不一样的参数名（如DOCKER_OPTS）*

*在修改前，应该搜索确认自己所用的发行版对应的参数值。*

## 重启docker

```
sudo service docker restart
```

# 可用镜像

docker官方国内镜像：
https://registry.docker-cn.com

163：
http://hub-mirror.c.163.com

ustc：
https://docker.mirrors.ustc.edu.cn

阿里云：
需要申请个人专用链接

# 参考链接

- [为aws中国配置docker镜像加速](https://yryz.net/post/aws-china-docker.html)
- [国内 docker 仓库镜像对比](http://ieevee.com/tech/2016/09/28/docker-mirror.html)
- [使用阿里云Docker镜像加速](http://warjiang.github.io/devcat/2016/11/28/%E4%BD%BF%E7%94%A8%E9%98%BF%E9%87%8C%E4%BA%91Docker%E9%95%9C%E5%83%8F%E5%8A%A0%E9%80%9F/)
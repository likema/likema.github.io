---
layout: post
title: 在Docker容器中创建GlusterFS
date: 2017-03-02 18:06:07
comments: true
categories: [GlusterFS, Docker]
---

本文展示如何在一台Linux物理机/虚拟机上，创建GlusterFS集群。目的在于测试和学习GlusterFS，而非将GlusterFS应用生产环境。

## 下载节点容器镜像

```bash
docker pull gluster/gluster-centos
```

## 创建节点容器实例

```bash
for i in `seq 1 3`; do
        docker run -d --privileged=true --name gluster$i --hostname=gluster$i \
                -v /etc/glusterfs$i:/etc/glusterfs:z \
                -v /var/lib/glusterd$i:/var/lib/glusterd:z \
                -v /var/log/glusterfs$i:/var/log/glusterfs:z \
                -v /srv/glusterfs$i:/srv/glusterfs:z \
                -v /sys/fs/cgroup:/sys/fs/cgroup:ro \
                -v /dev:/dev \
                gluster/gluster-centos
done
```

## 组建集群

```bash
docker exec -ti gluster1 /bin/bash
```

```bash
gluster peer probe <gluster2 ip addr>
gluster peer probe <gluster3 ip addr>
```

## 创建卷

### 冗余卷 (replica)

```bash
gluster peer volume create v1 replica 3 172.17.0.{2,3,4}:/srv/glusterfs/v1 force
```

### 条带卷 (stripe)

```bash
gluster peer volume create v2 strip 3 172.17.0.{2,3,4}:/srv/glusterfs/v2 force
```

### 纠删码卷 (disperse)

```bash
gluster peer volume create e3 disperse 3 redundancy 1 172.17.0.{2,3,4}:/srv/glusterfs/v3 force
```

* 由于这些容器实例的/srv/glusterfst与/在同一个分区，故需要指定force参数。
* 创建容器时，将host的/srv/glusterfs相关目录绑定至容器/srv/glusterfs是为了避免[no xattr support in Docker #1070](https://github.com/docker/docker/issues/1070)

## 挂载卷

### 通过FUSE挂载

```bash
mount -t glusterfs 172.17.0.2:/v1 /mnt/glusterfs/
```

## 脚本化

为了便于测试，我将上述诸多过程归纳成脚本：[gluster_docker](https://gist.github.com/likema/69f6617c7567766302ec1ee4a53a0f6c#file-gluster_docker)

## Gluster Web Interface

[gluster-web-interface](https://github.com/oss2016summer/gluster-web-interface)是一个管理GlusterFS的Web应用应用，它基于Ruby on Rails实现。

为了简化其安装，我创建其docker镜像[docker-gluster-web-interface](https://hub.docker.com/r/like/gluster-web-interface/)：

```bash
docker run -it -p 3000:3000 like/gluster-web-interface
```


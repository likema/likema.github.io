---
layout: post
title: 在Docker容器中创建GlusterFS
date: 2017-03-02 18:06:07
comments: true
categories: [GlusterFS, Docker]
---

本文展示如何在一台Linux物理机/虚拟机上，创建GlusterFS集群。目的在于测试和学习GlusterFS，而非将GlusterFs应用生产环境。

## 下载节点容器镜像

```bash
docker pull gluster/gluster-centos
```

## 创建元数据目录

```bash
mkdir -p /etc/glusterfs{1,2,3} /var/lib/glusterd{1,2,3} /var/log/glusterfs{1,2,3}
```

## 创建节点容器实例

```bash
for i in `seq 1 3`; do
        mkdir -p $etc$i $var$i $log$i
        docker run -d --privileged=true --name gluster$i --hostname=gluster$i \
                -v /etc/glusterfs$i:/etc/glusterfs:z \
                -v /var/lib/glusterd$i:/var/lib/glusterd:z \
                -v /var/log/glusterfs$i:/var/log/glusterfs:z \
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
gluster peer volume create v1 replica 3 172.17.0.{2,3,4}:/mnt/v1 force
```

### 条带卷 (stripe)

```bash
gluster peer volume create v2 strip 3 172.17.0.{2,3,4}:/mnt/v2 force
```

### 纠删码卷 (disperse)

```bash
gluster peer volume create e3 disperse 3 redundancy 1 172.17.0.{2,3,4}:/mnt/v3 force
```

* 由于这些容器实例的/mnt与/在同一个分区，故需要指定force参数。
* 创建卷时，目录/mnt/v{1,2,3}将被gluster自动创建，前提是父目录 (/mnt)已存在

## 挂载卷

### 通过FUSE挂载

```bash
mount -t glusterfs 172.17.0.2:/v1 /mnt/glusterfs/
```

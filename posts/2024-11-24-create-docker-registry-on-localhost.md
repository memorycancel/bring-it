---
title: 使用 docker 在本地服务器搭建 docker registry
layout: home
---

# 使用 docker 在本地服务器搭建 docker registry

2024-11-24 18：00

标题有点拗口，没错，就是使用本地的docker启动一个registry容器，
然后在本地编译镜像image，推送到本地的registry用作为本地测试。

首先就是安装docker，不作赘述。
值得一提的是docker安装后默认使用Ubuntu的root用户group。
需要将当前普通用户加入到docker用户组：

```bash
sudo gpasswd -a $USER docker
newgrp docker
docker ps
```

由于网络完全原因，dockerhub被墙了，需要修改默认registry为国内镜像：

```bash
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<EOF
{
    "registry-mirrors": [
        "https://hub-mirror.c.163.com",
        "https://mirror.ccs.tencentyun.com",
        "https://05f073ad3c0010ea0f4bc00b7105ec20.mirror.swr.myhuaweicloud.com",
        "https://registry.docker-cn.com",
        "https://docker.m.daocloud.io",
        "https://docker.1panel.live",
        "https://hub.rat.dev",
        "https://dockerpull.com",
        "https://dockerproxy.cn",
        "https://docker.rainbond.cc",
        "https://docker.udayun.com",
        "https://docker.211678.top"
    ]
}
EOF

systemctl restart docker
```

最后只需要下载 registry image和启动他，然后下载一个ubuntu image，然后推送到本地registry作为测试：

```bash
docker run -d -p 5000:5000 --name registry registry:2.7
docker logs -f registry
docker pull ubuntu
docker tag ubuntu localhost:5000/ubuntu
docker push localhost:5000/ubuntu
```

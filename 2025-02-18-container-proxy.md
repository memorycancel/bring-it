---
title: 使用 proxy 解决 docker 镜像的网络下载问题
layout: home
---

# 使用 proxy 解决 docker 镜像的网络下载问题

2025-02-18 20:00

3个问题：

+ 在本地主机安装了docker后经常会遇到网络原因拉不下来镜像。
+ 在远程服务器主机安装了docker后经常会遇到网络原因拉不下来镜像。
+ 使用 kubernetes 时，在 control panel 容器内的 docker 因为网络原因拉不下来镜像。

## 1 解决本地主机 docker 代理

在本地主机安装了docker后经常会遇到网络原因拉不下来镜像。主要要2大步骤：

### 1.1 本地主机安装代理

在本机安装代理客户端服务。例如 [https://github.com/XTLS/Xray-core](https://github.com/XTLS/Xray-core)。
代理服务器可以购买机场，或者自己搭建。
在主机配置代理参考[2024-11-12-Linux-startup-script](2024-11-12-Linux-startup-script)

### 1.2 修改 docker 后台服务代理配置

docker 里挂代理：

```shell
mkdir -p /etc/systemd/system/docker.service.d
vi /etc/systemd/system/docker.service.d/http-proxy.conf
# 配置文件 http-proxy.conf 填入以下内容，保存：
[Service]
Environment="HTTP_PROXY=http://127.0.0.1:10809"
Environment="HTTPS_PROXY=http://127.0.0.1:10809"

# 重新加载配置文件，重启 dockerd：
systemctl daemon-reload
systemctl restart docker

#（此时 docker pull 应该没问题了）
```
## 2 解决远程主机（服务器） docker 代理

在远程主机安装了docker后经常会遇到网络原因拉不下来镜像。主要要3大步骤（多了一步是将本地的代理开一个ssh隧道给远程服务器）：

### 2.1 本地主机代理

第一步与前文1.1一样

### 2.2 服务器挂代理

前提本地电脑已开梯子+允许局域网+端口号 10809，然后通过 ssh 隧道将本机代理服务传递给服务器。

```shell
# 1.先在本地电脑 cmd 执行：
ssh -R 10809:127.0.0.1:10809 -q -C -N username@remote_server
# （ username 和 remote_server 按实际修改，输入密码回车后没提示，不用管，此时 cmd 不要关闭）
# 2.再在服务器上执行：
export ALL_PROXY=socks5://127.0.0.1:10809
# （此时服务器 curl -v google.com 应该能返回 301 了）
```

### 2.3 修改 docker 后台服务代理配置

参考前文 1.2 章节。


## 3 在容器内使用主机代理服务

`kubectl describe pods`，在使用 minikube 搭建 kubernetes 集群时会遇到这个错误：

```text
Warning  Failed          63m (x3 over 65m)    kubelet            Failed to pull image "registry.k8s.io/e2e-test-images/agnhost:2.39": Error response from daemon: Head "https://us-west2-docker.pkg.dev/v2/k8s-artifacts-prod/images/e2e-test-images/agnhost/manifests/2.39": dial tcp 142.250.107.82:443: i/o timeout
```

这个错误表明 Kubernetes 节点（通过 kubelet）在尝试从 registry.k8s.io 拉取镜像时遇到了网络问题，具体表现为连接超时（i/o timeout）。这通常是由于网络访问限制或代理配置不正确导致的。

进入minikube容器，测试网络连接，确保允许访问以下地址：
```shell
minikube ssh
curl -I https://registry.k8s.io
curl -I us-west2-docker.pkg.dev # （Google Artifact Registry） 
```
此时发现是不通的与报错一样。

### 3.1 确保本机网络代理服务开放

为了解决这个问题，需要将主机的代理服务映射到容器内。通过`ip addr`命令，找到主机的 ip 为：`192.168.0.100`。进入到 control panel 容器内测试网络联通：

```shell
minikube ssh
# 或者 docker ps 然后 docker exec -ti [k8s-minikube-container-id] sh
ping 192.168.0.100
```
会发现网络是通的，这时候只需要修改 xray 的 config.json 文件，确保网络代理服务是绑定在 `0.0.0.0`上启动。这样通过 `192.168.0.100:10809` 和 `127.0.0.1:10809` 都可以访问到网络代理服务。

### 3.2 给minikube容器挂代理

```shell
# export HTTP_PROXY=http://127.0.0.1:10809
# export HTTPS_PROXY=http://127.0.0.1:10809
# export NO_PROXY=localhost,127.0.0.1,10.96.0.0/12,192.168.59.0/24,192.168.49.0/24,192.168.39.0/24
minikube start --vm-driver=docker \
  --docker-env HTTP_PROXY=http://192.168.0.100:10809 \
  --docker-env HTTPS_PROXY=http://192.168.0.100:10809 \
  --docker-env NO_PROXY=localhost,127.0.0.1,10.96.0.0/12,192.168.59.0/24,192.168.49.0/24,192.168.39.0/24

minikube ssh
# 或者 docker ps 然后 docker exec -ti [k8s-minikube-container-id] sh

https_proxy=192.168.0.100:10809 curl -I https://us-west2-docker.pkg.dev
```

### 3.3 给 minikube容器里面的docker 里挂代理

1.创建目录：mkdir -p /etc/systemd/system/docker.service.d
2.创建配置文件： /etc/systemd/system/docker.service.d/http-proxy.conf
3.配置文件 http-proxy.conf 填入以下内容，保存：

```
[Service]
Environment="HTTP_PROXY=http://192.168.0.100:10809"
Environment="HTTPS_PROXY=http://192.168.0.100:10809"
```
这样容器内的docker的服务也挂上了与本机一样的网络代理。

### 3.4 使用本地 insecure-registry

还有一种方法是在本地下载，集群通过`--insecure-registry "192.168.0.100:5000`访问本地的registry。

参考：[https://minikube.sigs.k8s.io/docs/handbook/registry/](https://minikube.sigs.k8s.io/docs/handbook/registry/)


在本地准备好镜像：

```shell
docker tag memorycancel/rails-8-demo 192.168.0.100:5000/memorycancel/rails-8-demo
docker push 192.168.0.100:5000/memorycancel/rails-8-demo
```

然后重新启动集群：

```shell
# 在使用前要确保先删除原容器。
minikube delete

minikube start --vm-driver=docker \
  --docker-env HTTP_PROXY=http://192.168.0.100:10809 \
  --docker-env HTTPS_PROXY=http://192.168.0.100:10809 \
  --docker-env NO_PROXY=localhost,127.0.0.1,10.96.0.0/12,192.168.59.0/24,192.168.49.0/24,192.168.39.0/24 \
  --insecure-registry "192.168.0.100:5000"

# 这个时候集群内部已经可以 pull 了

minikube ssh
docker pull 192.168.0.100:5000/memorycancel/rails-8-demo
```

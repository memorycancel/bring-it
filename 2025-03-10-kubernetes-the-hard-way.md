---
title: Kubernetes 集群中的 worker 节点 containerd 代理配置
layout: home
---

# Kubernetes 集群中的 worker 节点 containerd 代理配置

2025-03-11 02:00


```shell
sudo mkdir -p /etc/systemd/system/containerd.service.d/
sudo vi /etc/systemd/system/containerd.service.d/http-proxy.conf
```

```ini
[Service]
Environment="HTTP_PROXY=http://192.168.0.100:10809"
Environment="HTTPS_PROXY=http://192.168.0.100:10809"
Environment="NO_PROXY=localhost"
```

```shell
sudo systemctl daemon-reload
sudo systemctl restart containerd
```


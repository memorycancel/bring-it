---
title: 使用 minikube 部署 rails 8 应用
layout: home
---

# 使用 minikube 部署 rails 8 应用

2025-02-23 23:00

## 1 准备

+ Kubernetes 集群（minikube）
+ rails 8 应用

### 1.1 Kubernetes 集群

[使用本地-insecure-registry](2025-02-18-container-proxy#34-使用本地-insecure-registry)

{: .important :}
需要使用本地 registry ，所以启动集群时，控制节点的docker（minikube）需要加上 `--insecure-registry` 选项。

### 1.2 rails 8 应用镜像

rails 8 默认使用 kamal 部署，项目中默认已经写好了`dockerfile`，因此可以拿来就用。
[https://raw.githubusercontent.com/memorycancel/rails-8-demo/refs/heads/main/Dockerfile](https://raw.githubusercontent.com/memorycancel/rails-8-demo/refs/heads/main/Dockerfile)

参考注释，需要build两个镜像：

+ app： rails 8 应用镜像
+ solid： solid 镜像（queue cache cable）依赖此服务

```shell
docker build -t rails-8-demo .
docker tag memorycancel/rails-8-demo 192.168.0.100:5000/memorycancel/rails-8-demo
docker push 192.168.0.100:5000/memorycancel/rails-8-demo

docker build -t rails_solid .
docker tag rails_solid 192.168.0.100:5000/memorycancel/rails-8-demo/rails_solid
docker push 192.168.0.100:5000/memorycancel/rails-8-demo/rails_solid
```

## 2 部署

### 2.1 编写 deployments 部署文件

有 3 个要点：

1. 需要挂载一个volume存放sqlite-data（storage）
2. 生产环境需要传递 RAILS_MASTER_KEY 环境变量
3. 在启动主应用容器之前，需要 solid 服务作为sidecar容器（initContainer）

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: rails-8-demo
spec:
  replicas: 1
  selector:
    matchLabels:
      app: rails-8-demo
  template:
    metadata:
      labels:
        app: rails-8-demo
    spec:
      volumes:
        - name: sqlite-data
          emptyDir: {}
      containers:
        - name: rails-8-demo
          image: 192.168.0.100:5000/memorycancel/rails-8-demo
          volumeMounts:
            - name: sqlite-data
              mountPath: /rails/storage
          env:
            - name: RAILS_MASTER_KEY
              valueFrom:
                configMapKeyRef:
                  key: rails-master-key
                  name: rails-master-key
          ports:
            - containerPort: 3000
      initContainers:
        - name: rails-8-demo-solid
          image: 192.168.0.100:5000/memorycancel/rails-8-demo/rails_solid
          restartPolicy: Always
          volumeMounts:
            - name: sqlite-data
              mountPath: /rails/storage
          env:
            - name: RAILS_MASTER_KEY
              valueFrom:
                configMapKeyRef:
                  key: rails-master-key
                  name: rails-master-key
          command:
            - /rails/bin/jobs
            - start

```

## 2.2 启动应用

```shell
kubectl create configmap rails-master-key --from-literal=rails-master-key=`cat config/master.key`
kubectl get configmap

kubectl apply -f kubernetes/deploy.yaml
# kubectl delete deployments rails-8-demo
kubectl describe deployment rails-8-demo
kubectl logs deployment/rails-8-demo
kubectl get pods -l app=rails-8-demo
# kubectl describe pod rails-8-demo-6ffd78f64-c98bp

kubectl expose deployment/rails-8-demo --type="LoadBalancer" --port 3000
kubectl port-forward service/rails-8-demo 3000:3000

# 清理
kubectl delete deployments rails-8-demo
kubectl delete services rails-8-demo
```

示例项目地址： [https://github.com/memorycancel/rails-8-demo/tree/main/kubernetes](https://github.com/memorycancel/rails-8-demo/tree/main/kubernetes)

{: .note :}
这是单节点部署仅供测试，多节点部署需要添加持久卷并共享，进一步使用 postgresql 数据库集群。

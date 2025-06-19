---
title: 使用 Kubernetes 中的持久卷 Persistent Volumes
layout: home
---

# 使用 Kubernetes 中的持久卷 Persistent Volumes

2025-02-24 23:00

## 1 什么是持久卷

存储的管理是一个与计算实例的管理完全不同的问题。 为了实现将存储的准备和实现抽象出来。引入了两个新的 API 资源：PersistentVolume 和 PersistentVolumeClaim。

+ 持久卷（PersistentVolume，PV） ：是集群中的一块存储，可以由管理员事先制备， 或者使用存储类（Storage Class）来动态制备。 持久卷是集群资源，就像节点也是集群资源一样。PV 持久卷和普通的 Volume 一样， 也是使用卷插件来实现的，只是它们拥有独立于任何使用 PV 的 Pod 的生命周期。（更长）
+ 持久卷申领（PersistentVolumeClaim，PVC） 表达的是用户对存储的请求。概念上与 Pod 类似。 Pod 会耗用节点资源，而 PVC 申领会耗用 PV 资源（存储空间）。Pod 可以请求特定数量的资源（CPU 和内存）。同样 PVC 申领也可以请求特定的大小和访问模式 （例如，可以挂载为 ReadWriteOnce、ReadOnlyMany、ReadWriteMany 或 ReadWriteOncePod）。

尽管 PersistentVolumeClaim 允许用户消耗抽象的存储资源， 常见的情况是针对不同的问题用户需要具有不同属性（如，性能、协议、格式）的 PersistentVolume 卷。 集群管理员需要能够提供不同性质的 PersistentVolume， 并且这些 PV 卷之间的差别不仅限于卷大小和访问模式，同时又不能将卷是如何实现的这些细节暴露给用户。 为了满足这类需求，就有了存储类（StorageClass） 资源。

参考原文[https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/](https://kubernetes.io/zh-cn/docs/concepts/storage/persistent-volumes/)

## 2 持久卷类型与存储类 Persistent Volume（PV）and Storage Class

### 2.1 存储类 StorageClass

{: .important :}
StorageClass用于抽象底层存储系统的细节，允许动态创建PV，无需管理员手动介入。用户通过Persistent Volume Claim（PVC）请求存储资源时，StorageClass根据预定义的配置自动创建PV。

核心特性：

#### 动态供应（Dynamic Provisioning）

+ 当PVC引用StorageClass时，Kubernetes自动调用关联的Provisioner创建PV。
+ 示例：AWS EBS、GCP Persistent Disk等云存储的动态创建。

#### 参数化配置

+ 通过parameters字段定义存储后端的特性（如磁盘类型、区域、IOPS等）。
+ 示例：AWS gp2 StorageClass可能指定type: gp2。

#### 回收策略（Reclaim Policy）

+ 定义PVC删除后PV的处理方式（Delete或Retain）。
+ 云存储通常设置为Delete以自动清理资源。

#### 绑定模式（Volume Binding Mode）

+ 控制PV绑定时机（如WaitForFirstConsumer延迟绑定至Pod调度）。

示例：

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: aws-gp2
provisioner: ebs.csi.aws.com
parameters:
  type: gp2
reclaimPolicy: Delete
volumeBindingMode: WaitForFirstConsumer
```

### 2.2 持久卷类型 Persistent Volume（PV）types

PV类型对应具体的存储后端技术，决定了存储的访问方式、性能及使用场景。常见类型包括：

#### 1. 云提供商存储
+ AWS EBS：块存储，支持ReadWriteOnce，动态供应需CSI驱动。
+ GCP Persistent Disk：类似EBS，适用于GKE集群。
+ Azure Disk：Azure上的块存储服务。

#### 2. 网络存储
+ NFS：文件存储，支持ReadWriteMany，需手动或使用外部Provisioner（如NFS subdir）。
+ iSCSI：基于IP的块存储，适合跨节点共享。
+ Ceph/GlusterFS：分布式存储，支持高可用和扩展性。

#### 3. 本地存储
+ hostPath：节点本地目录，仅限测试（数据与节点生命周期绑定）。
+ local：正式本地卷，需静态创建PV并指定节点亲和性。

#### 4. 其他类型
+ CSI驱动扩展：支持自定义存储后端（如Portworx、Rook）。
+ Secret/ConfigMap：特殊类型，挂载配置数据而非持久化存储。

### 2.3 StorageClass与PV类型的关系

+ Provisioner驱动：StorageClass通过provisioner字段关联到具体PV类型（如ebs.csi.aws.com对应AWS EBS）。
+ 静态与动态供应
	+ 静态：管理员手动创建PV，StorageClass可选用于分类。
	+ 动态：StorageClass自动创建PV，无需手动干预。
+ 参数传递：StorageClass的parameters将配置传递给底层存储系统（如云磁盘类型、区域）。

## 3 使用场景

场景	 | StorageClass作用	 | PV类型选择
:-------------------------:|:-------------------------:|:-------------------------:
动态创建云磁盘	 | 定义云参数（类型、区域）	 | AWS EBS、GCP PD等
共享文件存储	 | 配置NFS服务器地址和路径	 | NFS、CephFS
高性能本地SSD	 | 指定本地卷Provisioner和节点亲和性	 | local卷类型
跨可用区持久化	 | 设置volumeBindingMode: WaitForFirstConsumer	 | 云存储（如EBS跨区挂载）

## 4 实操：配置 Pod 以使用 PersistentVolume 作为存储

原文参考：[https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-persistent-volume-storage/](https://kubernetes.io/zh-cn/docs/tasks/configure-pod-container/configure-persistent-volume-storage/)

3个步骤：

1. 作为管理员创建 PersistentVolume。该卷不与任何 Pod 关联。
2. 以开发人员或者集群用户的角色创建一个 PersistentVolumeClaim， 自动绑定到合适的 PersistentVolume。
3. 创建一个使用以上 PersistentVolumeClaim 作为存储的 Pod。

开始之前先准备好 minikube 集群：[使用本地-insecure-registry](2025-02-18-container-proxy#34-使用本地-insecure-registry)

```shell
# 在 minikube 集群中创建
minikube ssh
sudo mkdir /mnt/data
sudo sh -c "echo 'Hello from Kubernetes storage' > /mnt/data/index.html"
cat /mnt/data/index.html
# 关闭节点的 Shell。

# 1. 创建 PersistentVolume
# # pods/storage/pv-volume.yaml
# apiVersion: v1
# kind: PersistentVolume
# metadata:
#   name: task-pv-volume
#   labels:
#     type: local
# spec:
#   storageClassName: manual
#   capacity:
#     storage: 10Gi
#   accessModes:
#     - ReadWriteOnce
#   hostPath:
#     path: "/mnt/data"
kubectl apply -f https://k8s.io/examples/pods/storage/pv-volume.yaml
kubectl get pv task-pv-volume

# 2. 创建 PersistentVolumeClaim
# # pods/storage/pv-claim.yaml
# apiVersion: v1
# kind: PersistentVolumeClaim
# metadata:
#   name: task-pv-claim
# spec:
#   storageClassName: manual
#   accessModes:
#     - ReadWriteOnce
#   resources:
#     requests:
#       storage: 3Gi
kubectl apply -f https://k8s.io/examples/pods/storage/pv-claim.yaml
kubectl get pv task-pv-volume
kubectl get pvc task-pv-claim

# 3. 创建 Pod
# # pods/storage/pv-pod.yaml
# apiVersion: v1
# kind: Pod
# metadata:
#   name: task-pv-pod
# spec:
#   volumes:
#     - name: task-pv-storage
#       persistentVolumeClaim:
#         claimName: task-pv-claim
#   containers:
#     - name: task-pv-container
#       image: nginx
#       ports:
#         - containerPort: 80
#           name: "http-server"
#       volumeMounts:
#         - mountPath: "/usr/share/nginx/html"
#           name: task-pv-storage
kubectl apply -f https://k8s.io/examples/pods/storage/pv-pod.yaml
kubectl get pod task-pv-pod
kubectl exec -it task-pv-pod -- /bin/bash
apt update
apt install curl
curl http://localhost/

# 清理
kubectl delete pod task-pv-pod
kubectl delete pvc task-pv-claim
kubectl delete pv task-pv-volume
sudo rm /mnt/data/index.html
sudo rmdir /mnt/data
```

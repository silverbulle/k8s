# Kubernetes 学习笔记

> 面向有容器基础的开发者，系统掌握 K8s 架构、资源模型、核心流程与实战案例。

## 学习路线

```
第一阶段：理解集群架构（基础设施层）
  01 → 01a → 02
  掌握控制平面和工作节点的核心组件，理解 CRD 扩展机制

第二阶段：掌握资源模型（平台资源层）
  03 → 04 → 05
  学习 K8s 提供的各类资源抽象：工作负载、网络、配置与存储

第三阶段：理解核心流程（业务流程层）
  06
  串联各组件，理解 Pod 创建、服务发现、滚动更新、弹性伸缩的完整链路

第四阶段：实战案例（综合应用层）
  07 → 08 → 09 → 10
  通过真实场景将前面所有知识融会贯通
```

## 目录

### 第一部分：集群架构

| 章节 | 内容 | 阅读时间 |
|------|------|---------|
| [01-控制平面组件](01-控制平面组件.md) | kube-apiserver, etcd, kube-scheduler, kube-controller-manager | ~20 分钟 |
| [01a-CRD与Operator模式](01a-CRD与Operator模式.md) | CRD 定义、Operator 模式、kubebuilder 实战 | ~25 分钟 |
| [02-工作节点组件](02-工作节点组件.md) | kubelet, kube-proxy, 容器运行时 | ~15 分钟 |

### 第二部分：资源模型

| 章节 | 内容 | 阅读时间 |
|------|------|---------|
| [03-工作负载资源](03-工作负载资源.md) | Pod, ReplicaSet, Deployment, StatefulSet, DaemonSet, Job/CronJob | ~25 分钟 |
| [04-服务与网络](04-服务与网络.md) | Service, Ingress, DNS, NetworkPolicy | ~20 分钟 |
| [05-配置与存储](05-配置与存储.md) | ConfigMap, Secret, Volume, PV/PVC, StorageClass | ~20 分钟 |

### 第三部分：核心业务流程

| 章节 | 内容 | 阅读时间 |
|------|------|---------|
| [06-核心业务流程](06-核心业务流程.md) | Pod 创建流程, 服务发现, 滚动更新, 弹性伸缩 | ~25 分钟 |

### 第四部分：实战案例

| 章节 | 内容 | 阅读时间 |
|------|------|---------|
| [07-案例-Web应用部署](07-案例-Web应用部署.md) | Deployment + Service + Ingress 完整发布 | ~20 分钟 |
| [08-案例-有状态服务](08-案例-有状态服务.md) | StatefulSet + PV 部署 MySQL | ~20 分钟 |
| [09-案例-微服务架构](09-案例-微服务架构.md) | 多服务编排、服务发现、配置管理 | ~25 分钟 |
| [10-案例-弹性伸缩](10-案例-弹性伸缩.md) | HPA + VPA + Cluster Autoscaler | ~20 分钟 |

## 前置知识

- Docker / 容器基础（镜像、容器、Dockerfile）
- Linux 基础（网络、进程、文件系统）
- YAML 语法

## 参考资源

- [Kubernetes 官方文档](https://kubernetes.io/docs/)
- [Kubernetes 架构设计文档](https://github.com/kubernetes/community/tree/master/contributors/design-proposals)
- 《Kubernetes in Action》

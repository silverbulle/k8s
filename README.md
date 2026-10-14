# Kubernetes 学习笔记

> 面向有容器基础的开发者，系统掌握 K8s 架构、资源模型、核心流程与实战案例。

## 学习路线

```
第一阶段：理解集群架构（基础设施层）
  01 → 01a → 01b → 02
  掌握控制平面和工作节点的核心组件，理解 CRD 机制与安全证书体系

第二阶段：掌握资源模型（平台资源层）
  03 → 04 → 05 → 05b
  学习 K8s 提供的各类资源抽象：工作负载、网络、配置与存储、GPU 与高性能互联

第三阶段：理解核心流程与工程交付（业务流程层）
  06 → 06a
  串联各组件理解 Pod 生命周期、驱逐、服务发现、滚动更新与弹性伸缩，掌握 Kustomize 与 Helm 多环境交付

第四阶段：实战案例与生产排障（综合应用层）
  07 → 08 → 09 → 10 → 11
  通过真实场景将前面所有知识融会贯通，并沉淀生产级故障排查全景速查与 kubectl debug 技巧
```

## 目录

### 第一部分：集群架构

| 章节 | 内容 | 阅读时间 |
|------|------|---------|
| [01-控制平面组件](01-控制平面组件.md) | kube-apiserver, etcd, kube-scheduler, kube-controller-manager | ~25 分钟 |
| [01a-CRD与Operator模式](01a-CRD与Operator模式.md) | CRD 定义、Operator 模式、kubebuilder 实战 | ~30 分钟 |
| [01b-安全与证书体系](01b-安全与证书体系.md) | K8s 三大证书体系 (PKI)、双向 mTLS 认证、报错排查 | ~15 分钟 |
| [02-工作节点组件](02-工作节点组件.md) | kubelet, kube-proxy, 容器运行时, APIServer 本地代理 | ~25 分钟 |

### 第二部分：资源模型

| 章节 | 内容 | 阅读时间 |
|------|------|---------|
| [03-工作负载资源](03-工作负载资源.md) | Pod, SecurityContext/PSA, Deployment, StatefulSet, DaemonSet, Job/CronJob | ~30 分钟 |
| [04-服务与网络](04-服务与网络.md) | CNI (Calico), Service, Ingress, CoreDNS, NetworkPolicy | ~45 分钟 |
| [05-配置与存储](05-配置与存储.md) | ConfigMap, Secret, PV/PVC, StorageClass, CSI 原理, ResourceQuota/LimitRange | ~35 分钟 |
| [05b-GPU与高性能互联](05b-GPU与高性能互联.md) | GPU 调度, RDMA, NCCL, 数据并行, 集合通信 | ~40 分钟 |

### 第三部分：核心业务流程与工程交付

| 章节 | 内容 | 阅读时间 |
|------|------|---------|
| [06-核心业务流程](06-核心业务流程.md) | Pod 创建与状态流转, 资源驱逐, 服务发现, 滚动更新, 弹性伸缩 | ~35 分钟 |
| [06a-应用打包与多环境管理](06a-应用打包与多环境管理.md) | Kustomize (Base/Overlays), Helm (Chart/Release/Rollback), 与 Operator 选型边界 | ~25 分钟 |

### 第四部分：实战案例与生产排障

| 章节 | 内容 | 阅读时间 |
|------|------|---------|
| [07-案例-Web应用部署](07-案例-Web应用部署.md) | Deployment + Service + Ingress 完整发布 | ~20 分钟 |
| [08-案例-有状态服务](08-案例-有状态服务.md) | StatefulSet + PV + RBAC 部署 MySQL 主从复制集群 | ~25 分钟 |
| [09-案例-微服务架构](09-案例-微服务架构.md) | 多服务编排、服务发现、配置管理、Prometheus/Jaeger 可观测性 | ~25 分钟 |
| [10-案例-弹性伸缩](10-案例-弹性伸缩.md) | HPA + VPA + Cluster Autoscaler + PDB | ~20 分钟 |
| [11-故障排查全景速查](11-故障排查全景速查.md) | 八大高频故障诊断决策树、Exit Code 速查、kubectl debug 临时容器实战 | ~25 分钟 |

## 前置知识

- Docker / 容器基础（镜像、容器、Dockerfile）
- Linux 基础（网络、进程、文件系统）
- YAML 语法

## 参考资源

- [Kubernetes 官方文档](https://kubernetes.io/docs/)
- [Kubernetes 架构设计文档](https://github.com/kubernetes/community/tree/master/contributors/design-proposals)
- 《Kubernetes in Action》

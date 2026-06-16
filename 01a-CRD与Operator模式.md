# CRD 与 Operator 模式

CRD（Custom Resource Definition）是 K8s 最强大的扩展机制之一。它允许你定义自己的资源类型，让 K8s 像管理内置资源（Deployment、Service）一样管理你的自定义资源。

## 为什么需要 CRD

K8s 内置资源（Pod、Deployment、Service 等）覆盖了通用场景，但实际业务中经常遇到：

- **重复组合**：每次部署 Web 应用都要写 Deployment + Service + Ingress + HPA 四个 YAML
- **业务抽象缺失**：没有 "数据库集群"、"消息队列" 这样的高层概念
- **运维逻辑无法编码**：备份策略、故障切换、证书轮换等运维知识只存在于人脑中

CRD 让你**把运维知识编码为 K8s 资源**，通过声明式 API 和自动化控制循环来管理。

---

## 1. 核心概念

### 资源与控制器

```
┌─────────────────────────────────────────────────────┐
│  内置资源（K8s 原生提供）                              │
│                                                     │
│  Pod ────────────► kubelet（控制循环）                │
│  Deployment ─────► Deployment Controller（控制循环）  │
│  Service ────────► Endpoints Controller（控制循环）   │
│                                                     │
│  每种资源都有一个 Controller 负责将实际状态推向期望状态  │
└─────────────────────────────────────────────────────┘

┌─────────────────────────────────────────────────────┐
│  自定义资源（你定义的）                                │
│                                                     │
│  WebApp (CRD) ──► 你的 Controller（控制循环）         │
│  Database (CRD) ► 你的 Controller（控制循环）         │
│  Certificate ───► Cert-Manager Controller            │
│                                                     │
│  CRD 只是定义资源格式，还需要 Controller 赋予它行为    │
└─────────────────────────────────────────────────────┘
```

**关键理解：**
- **CRD** = 资源定义（Schema），告诉 K8s "这种资源长什么样"
- **CR（Custom Resource）** = 资源实例，用户创建的具体对象
- **Controller** = 控制循环，负责让实际状态匹配期望状态
- **Operator** = CRD + Controller 的打包组合，通常用于管理复杂有状态应用

### K8s 扩展机制全景

```
                    ┌─────────────────────┐
                    │   kube-apiserver    │
                    │                     │
   ┌───────────────►│  内置 API           │◄──────────────────┐
   │                │  /api/v1/...        │                   │
   │                │  /apis/apps/v1/...  │                   │
   │                └─────────┬───────────┘                   │
   │                          │                               │
   │                ┌─────────┼───────────┐                   │
   │                │                     │                   │
   │    CRD 注册     │    API 聚合层       │                   │
   │  (无需写代码)   │  (需要写 API Server) │                   │
   │                │                     │                   │
   │                ▼                     ▼                   │
   │    ┌───────────────┐     ┌────────────────┐             │
   │    │ 自定义资源     │     │ 聚合 API Server │             │
   │    │ (存在 etcd)   │     │ (独立进程)      │             │
   │    └───────────────┘     └────────────────┘             │
   │                                                         │
   │    典型：Cert-Manager,     典型：Metrics Server,         │
   │    Prometheus Operator     KubeAggregator               │
   │                                                         │
   └──────────── Controller 监听并响应资源变更 ──────────────┘
```

**CRD 是最常用的扩展方式**——不需要编写额外的 API Server，定义一个 YAML 就能注册新资源类型，数据直接存储在 etcd 中。

---

## 2. 定义一个 CRD

### 完整示例：WebApp CRD

假设我们经常部署 Web 应用，每次都写一套 Deployment + Service + Ingress。我们定义一个 `WebApp` 资源来抽象这些细节。

```yaml
# webapp-crd.yaml
apiVersion: apiextensions.k8s.io/v1
kind: CustomResourceDefinition
metadata:
  name: webapps.example.com          # 格式: <plural>.<group>
spec:
  group: example.com                  # API Group（类似 apps, batch）
  names:
    kind: WebApp                      # 资源类型名（单数，PascalCase）
    plural: webapps                   # 复数形式
    singular: webapp                  # 单数形式
    shortNames:                       # kubectl 缩写
    - wa
  scope: Namespaced                   # Namespaced 或 Cluster
  versions:
  - name: v1                          # API 版本
    served: true                      # 是否提供服务
    storage: true                     # 是否为存储版本（只能有一个）

    # Schema 定义（OpenAPI v3 格式）
    schema:
      openAPIV3Schema:
        type: object
        properties:
          spec:
            type: object
            required: [image]
            properties:
              image:
                type: string
                description: "容器镜像地址"
              replicas:
                type: integer
                minimum: 1
                maximum: 50
                default: 2
                description: "副本数"
              port:
                type: integer
                default: 8080
                description: "容器端口"
              domain:
                type: string
                description: "域名（配置 Ingress 用）"
              env:
                type: array
                items:
                  type: object
                  properties:
                    name:
                      type: string
                    value:
                      type: string
              resources:
                type: object
                properties:
                  cpu:
                    type: string
                    default: "100m"
                  memory:
                    type: string
                    default: "128Mi"

          # status 子资源（由 Controller 更新）
          status:
            type: object
            properties:
              phase:
                type: string
                enum: [Pending, Running, Updating, Failed]
              availableReplicas:
                type: integer
              conditions:
                type: array
                items:
                  type: object
                  properties:
                    type:
                      type: string
                    status:
                      type: string
                    lastTransitionTime:
                      type: string
                      format: date-time
                    reason:
                      type: string
                    message:
                      type: string

    # 子资源
    subresources:
      status: {}                      # 启用 /status 子资源
      scale:                          # 启用 /scale 子资源（支持 kubectl scale）
        specReplicasPath: .spec.replicas
        statusReplicasPath: .status.availableReplicas

    # 打印列（kubectl get 时显示）
    additionalPrinterColumns:
    - name: Replicas
      type: integer
      jsonPath: .spec.replicas
    - name: Available
      type: integer
      jsonPath: .status.availableReplicas
    - name: Domain
      type: string
      jsonPath: .spec.domain
    - name: Phase
      type: string
      jsonPath: .status.phase
    - name: Age
      type: date
      jsonPath: .metadata.creationTimestamp
```

### 关键概念解析

**Group/Version/Kind（GVK）：**

```
每个 K8s 资源都有唯一的 GVK 标识：

内置资源：
  Deployment → apps/v1/Deployment
  Service    → v1/Service (core group)
  Ingress    → networking.k8s.io/v1/Ingress

自定义资源：
  WebApp     → example.com/v1/WebApp
                ↑group    ↑version ↑kind
```

**scope（作用域）：**

| 类型 | 说明 | 示例 |
|------|------|------|
| `Namespaced` | 属于某个 Namespace | WebApp, Database |
| `Cluster` | 集群级别，不属于任何 Namespace | Node, StorageClass |

**Schema 校验：**

CRD 使用 OpenAPI v3 Schema 定义字段类型和约束。创建 CR 时，apiserver 会自动校验格式是否合法：

```bash
# 如果 replicas 设为字符串而不是整数
kubectl apply -f - <<EOF
apiVersion: example.com/v1
kind: WebApp
metadata:
  name: test
spec:
  image: nginx:latest
  replicas: "three"          # 不合法！Schema 要求 integer
EOF

# apiserver 会返回错误：
# The WebApp "test" is invalid: spec.replicas: Invalid value: "string": spec.replicas in body must be of type integer
```

**版本管理：**

```yaml
versions:
- name: v1alpha1              # 实验版本
  served: true
  storage: false              # 不是存储版本
- name: v1beta1               # Beta 版本
  served: true
  storage: false
- name: v1                    # 稳定版本
  served: true
  storage: true               # 存储版本（etcd 中存这个版本）
```

不同版本之间需要 **Conversion Webhook** 做自动转换，让用户可以用任意版本访问同一资源。

---

## 3. 创建自定义资源（CR）

CRD 注册后，就可以像使用内置资源一样创建 CR：

```yaml
# my-webapp.yaml
apiVersion: example.com/v1
kind: WebApp
metadata:
  name: shop-frontend
  namespace: production
spec:
  image: myregistry/shop-frontend:2.1.0
  replicas: 3
  port: 3000
  domain: shop.example.com
  resources:
    cpu: "200m"
    memory: "256Mi"
  env:
  - name: API_URL
    value: "http://api-svc:8080"
  - name: LOG_LEVEL
    value: "info"
```

```bash
# 注册 CRD
kubectl apply -f webapp-crd.yaml

# 创建 CR
kubectl apply -f my-webapp.yaml

# 查看（支持我们定义的打印列）
kubectl get webapps -n production
NAME              REPLICAS   AVAILABLE   DOMAIN              PHASE     AGE
shop-frontend     3          3           shop.example.com    Running   5m

# 使用短名称
kubectl get wa -n production

# 查看详情
kubectl describe webapp shop-frontend -n production

# 编辑
kubectl edit webapp shop-frontend

# 删除（级联删除由 Controller 创建的子资源）
kubectl delete webapp shop-frontend
```

---

## 4. Operator 模式：CRD + Controller

CRD 只定义了"资源长什么样"，但**不会自动做任何事**。需要一个 Controller 监听 CR 的变更并执行实际操作。

**CRD + Controller = Operator**

### Controller 的工作流程

```
用户创建/更新 WebApp CR
        │
        ▼
apiserver 写入 etcd
        │
        ▼
Controller Watch 到变更
        │
        ▼
┌─────────────────────────────────────────────────┐
│              Reconciliation Loop                  │
│                                                 │
│  1. 读取 WebApp CR 的 spec（期望状态）            │
│  2. 读取当前实际资源（Deployment/Service/Ingress） │
│  3. 计算差异                                     │
│  4. 创建/更新/删除子资源使实际状态匹配期望状态      │
│  5. 更新 WebApp CR 的 status                     │
│                                                 │
│  循环执行，直到实际状态 == 期望状态                 │
└─────────────────────────────────────────────────┘
```

### Controller 伪代码

```python
class WebAppController:
    def reconcile(self, webapp):
        """核心调谐逻辑，每次 WebApp CR 变更时调用"""

        name = webapp.metadata.name
        namespace = webapp.metadata.namespace
        spec = webapp.spec

        # 1. 确保 Deployment 存在且与 spec 一致
        desired_deployment = self.build_deployment(webapp)
        current_deployment = self.get_deployment(name, namespace)

        if current_deployment is None:
            self.create_deployment(desired_deployment)
        elif not self.matches(current_deployment, desired_deployment):
            self.update_deployment(desired_deployment)

        # 2. 确保 Service 存在
        desired_service = self.build_service(webapp)
        current_service = self.get_service(name, namespace)

        if current_service is None:
            self.create_service(desired_service)

        # 3. 如果指定了 domain，确保 Ingress 存在
        if spec.domain:
            desired_ingress = self.build_ingress(webapp)
            current_ingress = self.get_ingress(name, namespace)

            if current_ingress is None:
                self.create_ingress(desired_ingress)
            elif not self.matches(current_ingress, desired_ingress):
                self.update_ingress(desired_ingress)

        # 4. 更新 status
        available = self.get_deployment_available_replicas(name, namespace)
        self.update_status(webapp, {
            "phase": "Running" if available == spec.replicas else "Updating",
            "availableReplicas": available
        })

    def build_deployment(self, webapp):
        """根据 WebApp CR 构建期望的 Deployment"""
        return {
            "apiVersion": "apps/v1",
            "kind": "Deployment",
            "metadata": {
                "name": webapp.metadata.name,
                "namespace": webapp.metadata.namespace,
                "labels": {"app": webapp.metadata.name},
                "ownerReferences": [{                    # 关键：设置 OwnerReference
                    "apiVersion": "example.com/v1",
                    "kind": "WebApp",
                    "name": webapp.metadata.name,
                    "uid": webapp.metadata.uid
                }]
            },
            "spec": {
                "replicas": webapp.spec.replicas,
                "selector": {"matchLabels": {"app": webapp.metadata.name}},
                "template": {
                    "metadata": {"labels": {"app": webapp.metadata.name}},
                    "spec": {
                        "containers": [{
                            "name": "app",
                            "image": webapp.spec.image,
                            "ports": [{"containerPort": webapp.spec.port}],
                            "resources": {
                                "requests": {
                                    "cpu": webapp.spec.resources.cpu,
                                    "memory": webapp.spec.resources.memory
                                }
                            },
                            "env": webapp.spec.env or []
                        }]
                    }
                }
            }
        }
```

### OwnerReference：级联删除与垃圾回收

Controller 创建子资源时必须设置 `ownerReferences`，这带来两个关键能力：

```
WebApp (Owner)
  ├── Deployment (ownerReferences → WebApp)
  ├── Service    (ownerReferences → WebApp)
  └── Ingress    (ownerReferences → WebApp)

当 WebApp 被删除时：
  K8s Garbage Collector 自动删除所有子资源（级联删除）
  无需 Controller 手动清理
```

```bash
# 删除 WebApp → 自动删除 Deployment、Service、Ingress
kubectl delete webapp shop-frontend
# deployment.apps "shop-frontend" deleted
# service "shop-frontend" deleted
# ingress.networking.k8s.io "shop-frontend" deleted
```

---

## 5. 用 kubebuilder 构建 Operator

实际开发 Operator 通常使用 **kubebuilder** 或 **operator-sdk** 脚手架工具。以下是完整流程：

### 初始化项目

```bash
# 安装 kubebuilder
go install sigs.k8s.io/kubebuilder/v4/cmd/kubebuilder@latest

# 初始化项目
mkdir webapp-operator && cd webapp-operator
kubebuilder init --domain example.com --repo github.com/yourname/webapp-operator

# 创建 API（CRD + Controller 骨架）
kubebuilder create api --group example --version v1 --kind WebApp
```

### Controller 核心代码

```go
// controllers/webapp_controller.go
package controllers

import (
    "context"
    appsv1 "k8s.io/api/apps/v1"
    corev1 "k8s.io/api/core/v1"
    netv1 "k8s.io/api/networking/v1"
    "k8s.io/apimachinery/pkg/api/errors"
    metav1 "k8s.io/apimachinery/pkg/apis/meta/v1"
    "k8s.io/apimachinery/pkg/types"
    ctrl "sigs.k8s.io/controller-runtime"
    "sigs.k8s.io/controller-runtime/pkg/client"

    examplev1 "github.com/yourname/webapp-operator/api/v1"
)

type WebAppReconciler struct {
    client.Client
}

// Reconcile 是核心调谐函数
func (r *WebAppReconciler) Reconcile(ctx context.Context, req ctrl.Request) (ctrl.Result, error) {
    // 1. 获取 WebApp CR
    var webapp examplev1.WebApp
    if err := r.Get(ctx, req.NamespacedName, &webapp); err != nil {
        if errors.IsNotFound(err) {
            return ctrl.Result{}, nil  // CR 被删除了，不需要做任何事
        }
        return ctrl.Result{}, err
    }

    // 2. 调谐 Deployment
    if err := r.reconcileDeployment(ctx, &webapp); err != nil {
        return ctrl.Result{}, err
    }

    // 3. 调谐 Service
    if err := r.reconcileService(ctx, &webapp); err != nil {
        return ctrl.Result{}, err
    }

    // 4. 调谐 Ingress（如果有 domain）
    if webapp.Spec.Domain != "" {
        if err := r.reconcileIngress(ctx, &webapp); err != nil {
            return ctrl.Result{}, err
        }
    }

    // 5. 更新 Status
    deployment := &appsv1.Deployment{}
    r.Get(ctx, types.NamespacedName{Name: webapp.Name, Namespace: webapp.Namespace}, deployment)
    webapp.Status.AvailableReplicas = deployment.Status.AvailableReplicas
    if deployment.Status.AvailableReplicas == *deployment.Spec.Replicas {
        webapp.Status.Phase = "Running"
    } else {
        webapp.Status.Phase = "Updating"
    }
    r.Status().Update(ctx, &webapp)

    return ctrl.Result{}, nil
}

func (r *WebAppReconciler) reconcileDeployment(ctx context.Context, webapp *examplev1.WebApp) error {
    desired := &appsv1.Deployment{
        ObjectMeta: metav1.ObjectMeta{
            Name:      webapp.Name,
            Namespace: webapp.Namespace,
        },
        Spec: appsv1.DeploymentSpec{
            Replicas: &webapp.Spec.Replicas,
            Selector: &metav1.LabelSelector{
                MatchLabels: map[string]string{"app": webapp.Name},
            },
            Template: corev1.PodTemplateSpec{
                ObjectMeta: metav1.ObjectMeta{
                    Labels: map[string]string{"app": webapp.Name},
                },
                Spec: corev1.PodSpec{
                    Containers: []corev1.Container{{
                        Name:  "app",
                        Image: webapp.Spec.Image,
                        Ports: []corev1.ContainerPort{{ContainerPort: int32(webapp.Spec.Port)}},
                    }},
                },
            },
        },
    }

    // 设置 OwnerReference（实现级联删除）
    ctrl.SetControllerReference(webapp, desired, r.Scheme())

    // 创建或更新
    existing := &appsv1.Deployment{}
    err := r.Get(ctx, types.NamespacedName{Name: desired.Name, Namespace: desired.Namespace}, existing)
    if errors.IsNotFound(err) {
        return r.Create(ctx, desired)
    }
    existing.Spec = desired.Spec
    return r.Update(ctx, existing)
}
```

### 构建与部署

```bash
# 生成 CRD manifests
make manifests

# 构建镜像
make docker-build IMG=myregistry/webapp-operator:1.0.0

# 推送到仓库
make docker-push IMG=myregistry/webapp-operator:1.0.0

# 部署到集群（CRD + Controller Deployment + RBAC）
make deploy IMG=myregistry/webapp-operator:1.0.0

# 验证
kubectl get crd webapps.example.com
kubectl get pods -n webapp-operator-system
```

---

## 6. 生态中的经典 CRD 案例

| 项目 | CRD | 用途 |
|------|-----|------|
| **Cert-Manager** | `Certificate` | 声明式管理 TLS 证书，自动申请和续期 |
| **Prometheus Operator** | `Prometheus`, `ServiceMonitor` | 声明式配置监控目标和告警规则 |
| **Argo CD** | `Application` | GitOps 声明式持续交付 |
| **Istio** | `VirtualService`, `DestinationRule` | 服务网格流量管理 |
| **Crossplane** | 各种云资源 CRD | 用 K8s API 管理云基础设施（AWS RDS、S3 等） |
| **Knative** | `Service`, `Revision` | Serverless 自动伸缩到零 |

### 案例：Cert-Manager 的 Certificate CRD

```yaml
# 只需声明"我要一个证书"，Cert-Manager Operator 自动完成申请和续期
apiVersion: cert-manager.io/v1
kind: Certificate
metadata:
  name: shop-tls
  namespace: production
spec:
  secretName: shop-tls-secret       # 证书存到这个 Secret 中
  issuerRef:
    name: letsencrypt-prod          # 使用 Let's Encrypt
    kind: ClusterIssuer
  dnsNames:
  - shop.example.com
  - www.shop.example.com
```

Cert-Manager Controller 自动：
1. 生成私钥
2. 向 Let's Encrypt 申请证书（ACME 协议）
3. 完成 DNS/HTTP 验证
4. 将证书写入 Secret
5. **到期前 30 天自动续期**

### 案例：Prometheus Operator 的 ServiceMonitor

```yaml
# 声明"监控这些 Pod"，无需手动编辑 Prometheus 配置
apiVersion: monitoring.coreos.com/v1
kind: ServiceMonitor
metadata:
  name: webapp-monitor
spec:
  selector:
    matchLabels:
      app: webapp
  endpoints:
  - port: metrics
    interval: 15s
    path: /metrics
```

Prometheus Operator 自动将 ServiceMonitor 转换为 Prometheus 的 scrape 配置。

---

## 7. CRD 设计最佳实践

### 1. 声明式优于命令式

```yaml
# ✗ 不好：命令式（告诉系统怎么做）
spec:
  action: create_deployment
  steps:
  - run: kubectl create deployment ...
  - run: kubectl expose ...

# ✓ 好：声明式（告诉系统要什么结果）
spec:
  image: myapp:1.0
  replicas: 3
  domain: app.example.com
```

### 2. Status 与 Spec 分离

```yaml
spec:          # 用户写的（期望状态）
  replicas: 3

status:        # Controller 写的（实际状态）
  availableReplicas: 3
  phase: Running
  conditions:
  - type: Ready
    status: "True"
    lastTransitionTime: "2024-01-15T10:00:00Z"
```

### 3. 使用 Conditions 报告详细状态

```yaml
status:
  conditions:
  - type: DeploymentReady
    status: "True"
    reason: MinimumReplicasAvailable
    message: "Deployment has minimum availability"
  - type: IngressReady
    status: "False"
    reason: DNSNotResolved
    message: "DNS resolution failed for shop.example.com"
  - type: CertificateReady
    status: "True"
    reason: CertificateIssued
    message: "TLS certificate issued successfully"
```

### 4. 支持 kubectl scale

```yaml
subresources:
  scale:
    specReplicasPath: .spec.replicas
    statusReplicasPath: .status.availableReplicas
```

```bash
# 用户可以直接使用 kubectl scale
kubectl scale webapp shop-frontend --replicas=5
```

### 5. 使用 Finalizer 处理清理逻辑

```go
// 在删除前执行清理（比如删除外部资源）
func (r *WebAppReconciler) reconcileDelete(ctx context.Context, webapp *examplev1.WebApp) error {
    if controllerutil.ContainsFinalizer(webapp, "example.com/webapp-cleanup") {
        // 执行清理逻辑（如删除外部 DNS 记录）
        if err := r.cleanupExternalResources(webapp); err != nil {
            return err
        }
        // 移除 Finalizer，允许 K8s 完成删除
        controllerutil.RemoveFinalizer(webapp, "example.com/webapp-cleanup")
        r.Update(ctx, webapp)
    }
    return nil
}
```

---

## 本章小结

| 概念 | 一句话总结 |
|------|-----------|
| **CRD** | 自定义资源定义，扩展 K8s API |
| **CR** | CRD 的实例，用户创建的具体对象 |
| **Controller** | 控制循环，将实际状态推向期望状态 |
| **Operator** | CRD + Controller 的打包组合 |
| **OwnerReference** | 级联删除，Owner 被删时子资源自动清理 |
| **Finalizer** | 删除前的清理钩子，用于处理外部资源 |
| **kubebuilder** | Operator 开发脚手架工具 |

**回到目录：[README](README.md)** | **相关章节：[01-控制平面组件](01-控制平面组件.md)**

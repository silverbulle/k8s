# 07 - 案例：Web 应用部署

本案例演示如何将一个 Web 应用完整部署到 K8s 集群，涵盖 Deployment、Service、Ingress、ConfigMap、Secret 和 HPA。

## 场景描述

部署一个前后端分离的 Web 应用：

```
                   ┌──────────────┐
    用户请求 ────► │   Ingress    │
                   │  (Nginx IC)  │
                   └──────┬───────┘
                          │
              ┌───────────┴───────────┐
              ▼                       ▼
    ┌──────────────────┐    ┌──────────────────┐
    │  Frontend (Vue)  │    │  Backend (API)   │
    │  Deployment      │    │  Deployment      │
    │  + Service       │    │  + Service       │
    └──────────────────┘    └────────┬─────────┘
                                     │
                                     ▼
                            ┌──────────────────┐
                            │  Redis Cache     │
                            │  Deployment      │
                            │  + Service       │
                            └──────────────────┘
```

---

## 第一步：命名空间与配置

```yaml
# 01-namespace.yaml
apiVersion: v1
kind: Namespace
metadata:
  name: webapp
---
# 02-configmap.yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
  namespace: webapp
data:
  APP_ENV: "production"
  LOG_LEVEL: "info"
  REDIS_HOST: "redis-svc.webapp.svc.cluster.local"
  REDIS_PORT: "6379"
---
# 03-secret.yaml
apiVersion: v1
kind: Secret
metadata:
  name: app-secret
  namespace: webapp
type: Opaque
stringData:                    # stringData 自动 Base64 编码
  JWT_SECRET: "your-jwt-secret-key"
  API_KEY: "sk-abc123xyz"
```

---

## 第二步：部署后端 API

```yaml
# 04-backend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: backend
  namespace: webapp
  labels:
    app: backend
spec:
  replicas: 3
  selector:
    matchLabels:
      app: backend
  strategy:
    type: RollingUpdate
    rollingUpdate:
      maxSurge: 1
      maxUnavailable: 0
  template:
    metadata:
      labels:
        app: backend
        version: v1
    spec:
      containers:
      - name: backend
        image: myregistry/backend:1.0.0
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: APP_ENV
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_ENV
        - name: JWT_SECRET
          valueFrom:
            secretKeyRef:
              name: app-secret
              key: JWT_SECRET
        - name: REDIS_HOST
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: REDIS_HOST
        resources:
          requests:
            cpu: "100m"
            memory: "128Mi"
          limits:
            cpu: "500m"
            memory: "256Mi"
        readinessProbe:
          httpGet:
            path: /api/health
            port: 8080
          initialDelaySeconds: 5
          periodSeconds: 10
        livenessProbe:
          httpGet:
            path: /api/health
            port: 8080
          initialDelaySeconds: 15
          periodSeconds: 20
          failureThreshold: 3
        startupProbe:
          httpGet:
            path: /api/health
            port: 8080
          failureThreshold: 30
          periodSeconds: 5
        lifecycle:
          preStop:
            exec:
              command: ["/bin/sh", "-c", "sleep 10"]
      terminationGracePeriodSeconds: 40
---
# 05-backend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: backend-svc
  namespace: webapp
spec:
  type: ClusterIP
  selector:
    app: backend
  ports:
  - port: 80
    targetPort: 8080
    protocol: TCP
    name: http
```

---

## 第三步：部署前端

```yaml
# 06-frontend-deployment.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: frontend
  namespace: webapp
spec:
  replicas: 2
  selector:
    matchLabels:
      app: frontend
  template:
    metadata:
      labels:
        app: frontend
    spec:
      containers:
      - name: frontend
        image: myregistry/frontend:1.0.0
        ports:
        - containerPort: 80
        resources:
          requests:
            cpu: "50m"
            memory: "64Mi"
          limits:
            cpu: "200m"
            memory: "128Mi"
        readinessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 5
        livenessProbe:
          httpGet:
            path: /
            port: 80
          periodSeconds: 10
---
# 07-frontend-service.yaml
apiVersion: v1
kind: Service
metadata:
  name: frontend-svc
  namespace: webapp
spec:
  type: ClusterIP
  selector:
    app: frontend
  ports:
  - port: 80
    targetPort: 80
```

---

## 第四步：部署 Redis 缓存

```yaml
# 08-redis.yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
  namespace: webapp
spec:
  replicas: 1
  selector:
    matchLabels:
      app: redis
  template:
    metadata:
      labels:
        app: redis
    spec:
      containers:
      - name: redis
        image: redis:7-alpine
        ports:
        - containerPort: 6379
        command: ["redis-server", "--maxmemory", "256mb", "--maxmemory-policy", "allkeys-lru"]
        resources:
          requests:
            cpu: "50m"
            memory: "128Mi"
          limits:
            cpu: "200m"
            memory: "300Mi"
        readinessProbe:
          tcpSocket:
            port: 6379
          periodSeconds: 5
---
apiVersion: v1
kind: Service
metadata:
  name: redis-svc
  namespace: webapp
spec:
  selector:
    app: redis
  ports:
  - port: 6379
    targetPort: 6379
```

---

## 第五步：配置 Ingress

```yaml
# 09-ingress.yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: webapp-ingress
  namespace: webapp
  annotations:
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
    nginx.ingress.kubernetes.io/proxy-body-size: "10m"
    # 限流：每秒 20 个请求每 IP
    nginx.ingress.kubernetes.io/limit-rps: "20"
    nginx.ingress.kubernetes.io/limit-burst-multiplier: "5"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - app.example.com
    secretName: webapp-tls
  rules:
  - host: app.example.com
    http:
      paths:
      # API 请求 → 后端服务
      - path: /api
        pathType: Prefix
        backend:
          service:
            name: backend-svc
            port:
              number: 80
      # 其他请求 → 前端服务
      - path: /
        pathType: Prefix
        backend:
          service:
            name: frontend-svc
            port:
              number: 80
```

创建 TLS 证书 Secret：

```bash
kubectl create secret tls webapp-tls \
  --cert=certs/app.example.com.crt \
  --key=certs/app.example.com.key \
  -n webapp
```

---

## 第六步：配置 HPA 自动伸缩

```yaml
# 10-hpa.yaml
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: backend-hpa
  namespace: webapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: backend
  minReplicas: 3
  maxReplicas: 15
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
  behavior:
    scaleDown:
      stabilizationWindowSeconds: 300
    scaleUp:
      stabilizationWindowSeconds: 0
---
apiVersion: autoscaling/v2
kind: HorizontalPodAutoscaler
metadata:
  name: frontend-hpa
  namespace: webapp
spec:
  scaleTargetRef:
    apiVersion: apps/v1
    kind: Deployment
    name: frontend
  minReplicas: 2
  maxReplicas: 10
  metrics:
  - type: Resource
    resource:
      name: cpu
      target:
        type: Utilization
        averageUtilization: 70
```

---

## 部署执行

```bash
# 按顺序部署
kubectl apply -f 01-namespace.yaml
kubectl apply -f 02-configmap.yaml
kubectl apply -f 03-secret.yaml
kubectl apply -f 04-backend-deployment.yaml
kubectl apply -f 05-backend-service.yaml
kubectl apply -f 06-frontend-deployment.yaml
kubectl apply -f 07-frontend-service.yaml
kubectl apply -f 08-redis.yaml
kubectl apply -f 09-ingress.yaml
kubectl apply -f 10-hpa.yaml

# 或者一次性部署（按文件名排序）
kubectl apply -f . -n webapp

# 检查状态
kubectl get all -n webapp
kubectl get ingress -n webapp
kubectl get hpa -n webapp
```

---

## 验证与测试

```bash
# 1. 检查 Pod 状态
kubectl get pods -n webapp
NAME                        READY   STATUS    RESTARTS   AGE
backend-5d8f7b-abc          1/1     Running   0          2m
backend-5d8f7b-def          1/1     Running   0          2m
backend-5d8f7b-ghi          1/1     Running   0          2m
frontend-7a9c3e-jkl         1/1     Running   0          2m
frontend-7a9c3e-mno         1/1     Running   0          2m
redis-4b6d2f-pqr            1/1     Running   0          2m

# 2. 验证 Service
kubectl get svc -n webapp
NAME           TYPE        CLUSTER-IP       PORT(S)
backend-svc    ClusterIP   10.96.100.10     80/TCP
frontend-svc   ClusterIP   10.96.100.20     80/TCP
redis-svc      ClusterIP   10.96.100.30     6379/TCP

# 3. 测试 API 访问
kubectl exec -it backend-5d8f7b-abc -n webapp -- curl http://backend-svc/api/health
# {"status":"ok","version":"1.0.0"}

# 4. 端口转发测试
kubectl port-forward svc/frontend-svc 8080:80 -n webapp
# 浏览器打开 http://localhost:8080

# 5. 查看 Ingress
kubectl get ingress -n webapp
NAME              HOSTS             ADDRESS         PORTS
webapp-ingress    app.example.com   52.1.2.3        80, 443

# 6. 测试外部访问
curl -k https://app.example.com/api/health
curl -k https://app.example.com/

# 7. 模拟压力测试（验证 HPA）
kubectl run load-test --image=busybox -n webapp --rm -it -- \
  sh -c "while true; do wget -q -O- http://backend-svc/api/health; done"

# 另一个终端观察 HPA
kubectl get hpa -n webapp -w
NAME           REFERENCE             TARGETS   MINPODS   MAXPODS   REPLICAS
backend-hpa    Deployment/backend    85%/70%   3         15        3→5→8
```

---

## 发布新版本

```bash
# 更新后端镜像版本
kubectl set image deployment/backend \
  backend=myregistry/backend:2.0.0 \
  -n webapp

# 观察滚动更新
kubectl rollout status deployment/backend -n webapp

# 如果有问题，回滚
kubectl rollout undo deployment/backend -n webapp
```

---

## 知识点回顾

| 使用的资源 | 作用 |
|-----------|------|
| Namespace | 隔离 webapp 的资源 |
| ConfigMap | 应用配置注入 |
| Secret | 敏感信息管理 |
| Deployment | 管理 Pod 副本，支持滚动更新 |
| Service (ClusterIP) | 集群内部服务发现 |
| Ingress | HTTP 路由 + TLS 终止 |
| HPA | 基于 CPU 的自动水平伸缩 |

**下一章：[08-案例-有状态服务](08-案例-有状态服务.md)** →

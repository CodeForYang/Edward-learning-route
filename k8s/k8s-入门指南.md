# ☸️ Kubernetes 入门学习路线

> 适合：有基础 Docker 概念 + AWS 免费账号 + macOS
> 工具：Kind（Kubernetes in Docker）本地学习，可选拓展到 AWS EKS
> 周期：7-10 天，每天 1-2 小时

---

## 📋 目录

1. [学习路线总览](#学习路线总览)
2. [前置准备](#前置准备)
3. [Day 1：K8s 架构与环境搭建](#day-1k8s-架构与环境搭建)
4. [Day 2：Pod — 最小的计算单元](#day-2pod--最小的计算单元)
5. [Day 3：Deployment — 管理 Pod 的生命周期](#day-3deployment--管理-pod-的生命周期)
6. [Day 4：Service — 网络与访问](#day-4service--网络与访问)
7. [Day 5：ConfigMap & Secret — 配置管理](#day-5configmap--secret--配置管理)
8. [Day 6：存储卷（Volumes & PVC）](#day-6存储卷volumes--pvc)
9. [Day 7：健康检查与故障排查](#day-7健康检查与故障排查)
10. [Day 8：整合 Demo — 完整应用部署](#day-8整合-demo--完整应用部署)
11. [Bonus：尝试 AWS EKS（可选）](#bonus尝试-aws-eks可选)
12. [附录：K8s 命令速查表](#附录k8s-命令速查表)

---

## 学习路线总览

```
Day 1 ───── 架构理解 + 搭环境（Kind + kubectl）
  │
Day 2 ───── Pod 核心概念 + YAML 声明式
  │
Day 3 ───── Deployment（滚动更新、扩缩容、回滚）
  │
Day 4 ───── Service（ClusterIP / NodePort / LoadBalancer）
  │
Day 5 ───── ConfigMap + Secret（配置与敏感信息）
  │
Day 6 ───── 存储卷（Volumes + PVC 持久化）
  │
Day 7 ───── 健康检查（Probes）+ kubectl debug
  │
Day 8 ───── 🎯 整合 Demo（Flask + Redis + 全套配置）
  │
Bonus ───── EKS 体验
```

---

## 前置准备

### 硬件/软件要求

- macOS（Intel 或 Apple Silicon）
- Docker Desktop（Docker 入门指南中已安装）
- 终端（iTerm2 或自带 Terminal）
- 4GB+ 可用内存（Kind 集群运行时占用 ~1-2GB）

### 安装工具

#### 1️⃣ 安装 kubectl

```bash
# 使用 Homebrew 安装
brew install kubectl

# 验证
kubectl version --client
# 输出类似：Client Version: v1.30.0
```

#### 2️⃣ 安装 Kind

```bash
# 使用 Homebrew 安装
brew install kind

# 验证
kind version
# 输出类似：kind v0.23.0 go1.21.7 darwin/arm64
```

#### 3️⃣ 验证 Docker Desktop 正在运行

```bash
docker info | head -5
# 应该能看到服务器信息，说明 Docker 引擎在跑
```

> 💡 Kind 的工作原理：Kubernetes 控制平面和 Worker 节点都跑在 **Docker 容器** 里。所以 Docker Desktop 是 Kind 的底层依赖。

---

## Day 1：K8s 架构与环境搭建

### 1.1 K8s 是什么？

**Kubernetes（K8s）** 是一个容器编排平台，回答三个核心问题：

```
💡 你有很多容器要跑
❓ 放哪台机器？  →  调度器帮你决定
❓ 挂了怎么办？  →  控制器帮你重启
❓ 怎么访问？    →  Service 帮你做网络
```

### 1.2 核心架构一览

```
┌─────────────────────────────────────────────────┐
│                  Control Plane                    │
│  ┌──────────┐  ┌──────────┐  ┌────────────────┐ │
│  │ API      │  │ Scheduler │  │ Controller     │ │
│  │ Server   │  │           │  │ Manager        │ │
│  └────┬─────┘  └──────────┘  └────────────────┘ │
│       │                                            │
│  ┌────▼─────┐                                     │
│  │ etcd     │  (键值存储，集群状态)                  │
│  └──────────┘                                     │
└─────────────────────────────────────────────────┘
         │ API Server（HTTPS）
         ▼
┌───────────────────┐   ┌───────────────────┐
│  Worker Node 1    │   │  Worker Node 2    │
│  ┌───────────┐    │   │  ┌───────────┐    │
│  │  Pod      │    │   │  │  Pod      │    │
│  │ ┌───────┐ │    │   │  │ ┌───────┐ │    │
│  │ │容器A   │ │    │   │  │ │容器B   │ │    │
│  │ └───────┘ │    │   │  │ └───────┘ │    │
│  └───────────┘    │   │  └───────────┘    │
│  ┌───────────┐    │   │  ┌───────────┐    │
│  │ kubelet   │    │   │  │ kubelet   │    │
│  └───────────┘    │   │  └───────────┘    │
│  ┌───────────┐    │   │  ┌───────────┐    │
│  │ kube-proxy│    │   │  │ kube-proxy│    │
│  └───────────┘    │   │  └───────────┘    │
└───────────────────┘   └───────────────────┘
```

**关键组件速记：**

| 组件 | 角色 | 一句话 |
|------|------|--------|
| **API Server** | 集群大门 | 所有操作都通过它（kubectl 就是跟它说话）|
| **etcd** | 集群数据库 | 存所有状态（谁在跑、在哪跑）|
| **Scheduler** | 调度员 | 新 Pod 放哪个节点？|
| **Controller Manager** | 监工 | 确保实际状态=期望状态 |
| **kubelet** | 节点代理人 | 每个节点一个，跟 API Server 通信 |
| **kube-proxy** | 网络代理 | 处理网络规则 |

### 1.3 创建你的第一个 Kind 集群

```bash
# 创建一个最简单的 Kind 集群
kind create cluster

# 输出：
# Creating cluster "kind" ...
# ✓ Ensuring node image (kindest/node:v1.30.0) 🖼
# ✓ Preparing nodes 📦
# ✓ Writing configuration 📜
# ✓ Starting control-plane 🕹️
# ✓ Installing CNI 🔌
# ✓ Installing StorageClass 💾
# Set kubectl context to "kind-kind"
```

查看集群状态：

```bash
# 查看集群节点
kubectl get nodes
# NAME                 STATUS   ROLES           AGE   VERSION
# kind-control-plane   Ready    control-plane   1m    v1.30.0

# 查看集群信息
kubectl cluster-info
# Kubernetes control plane is running at https://127.0.0.1:6443
# CoreDNS is running at https://127.0.0.1:6443/api/...

# 查看命名空间
kubectl get namespaces
# NAME              STATUS   AGE
# default           Active   2m
# kube-node-lease   Active   2m
# kube-public       Active   2m
# kube-system       Active   2m
```

### 1.4 认识 kubectl

kubectl 是你跟 K8s 对话的 CLI 工具。

```
kubectl <动作> <资源类型> [资源名称] [选项]
   │         │            │
  get      pod          my-pod
  create   deployment   --namespace=dev
  delete   service
  apply    configmap
  describe  (所有资源)
  logs     (仅 Pod)
  exec     (仅 Pod)
```

**最重要的一条原则：** `kubectl apply -f <yaml>` 是你用得最多的命令。

### 1.5 清理当前环境

```bash
# 删除 Kind 集群（今天结束时做）
kind delete cluster

# 以后再创建
kind create cluster
```

### ✅ Day 1 学习成果检查

- [ ] 理解了 K8s Control Plane / Worker Node 的基本架构
- [ ] 安装了 kubectl + kind
- [ ] 能创建和删除 Kind 集群
- [ ] 会用 `kubectl get nodes` 和 `kubectl cluster-info`
- [ ] 理解了 `kubectl <动作> <资源>` 的命令模式

---

## Day 2：Pod — 最小的计算单元

### 2.1 什么是 Pod？

**Pod** 是 K8s 中最小的部署单元，一个 Pod 包含一个或多个容器。

```
┌─────────────────────────┐
│         Pod             │
│  ┌───────┐  ┌───────┐  │
│  │ 容器A  │  │ 容器B  │  │ ← 共享网络（127.0.0.1）
│  │ (app)  │  │(sidecar)│  ← 共享存储卷
│  └───────┘  └───────┘  │
│    IP: 10.244.0.5      │ ← Pod 有自己的 IP
└─────────────────────────┘
```

**关键理解：**
- Pod 是"逻辑主机"——里面的容器共享网络栈和存储
- 同一个 Pod 的容器可以通过 `localhost` 互相访问
- Pod 是短暂的——它可能随时被销毁和重建（IP 会变！）

### 2.2 两种创建方式

#### 方式一：命令式（Imperative）

```bash
# 创建一个 Nginx Pod
kubectl run nginx-pod --image=nginx:latest

# 查看 Pod
kubectl get pods
# NAME        READY   STATUS    RESTARTS   AGE
# nginx-pod   1/1     Running   0          10s

# 查看详细信息
kubectl get pod nginx-pod -o wide
# NAME        READY   STATUS    ...  IP           NODE
# nginx-pod   1/1     Running   ...  10.244.0.4   kind-control-plane

# 查看 Pod 完整描述（包含事件）
kubectl describe pod nginx-pod
```

#### 方式二：声明式（Declarative — 推荐！）

创建 `~/k8s-learning/pod-nginx.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: nginx-pod
  labels:
    app: nginx
    env: dev
spec:
  containers:
    - name: nginx
      image: nginx:latest
      ports:
        - containerPort: 80
```

应用：

```bash
mkdir -p ~/k8s-learning && cd ~/k8s-learning

kubectl apply -f pod-nginx.yaml

# 查看 Pod
kubectl get pods
```

> 💡 **为什么推荐声明式？**
> - 文件可以版本控制（Git）
> - 可重复、可共享
> - `kubectl apply` 是幂等的（跑多次结果一致）
> - 生产环境几乎只用声明式

### 2.3 YAML 结构速记

```yaml
apiVersion: v1            # API 版本（不同资源版本不同）
kind: Pod                 # 资源类型
metadata:                 # 元数据
  name: my-pod
  labels:                 # 标签（重要！用于选择器）
    app: my-app
spec:                     # 期望状态
  containers:             # 容器列表
    - name: app
      image: nginx
```

> 💡 **思考：为什么要 labels？**
> 想像有 100 个 Pod 在跑，Service、Deployment 如何知道哪些 Pod 归它管？答案就是 **Labels + Selector**。

### 2.4 与 Pod 交互

```bash
# 查看日志
kubectl logs nginx-pod

# 持续跟踪日志
kubectl logs -f nginx-pod

# 进入 Pod 内部
kubectl exec -it nginx-pod -- bash
# 进入后可以 ls、cat 等操作
# exit 退出

# 端口转发（把 Pod 的 80 端口映射到本机 8080）
kubectl port-forward pod/nginx-pod 8080:80
# 然后浏览器访问 http://localhost:8080
```

### 2.5 多容器 Pod

创建 `pod-multi.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: multi-pod
spec:
  containers:
    - name: app
      image: nginx:latest
      ports:
        - containerPort: 80
    - name: sidecar
      image: alpine:latest
      command: ["sh", "-c", "sleep 3600"]
```

```bash
kubectl apply -f pod-multi.yaml
kubectl get pods -w    # -w 持续观察变化

# 查看两个容器各自的日志
kubectl logs multi-pod -c app
kubectl logs multi-pod -c sidecar
```

### 2.6 删除 Pod

```bash
# 删除特定 Pod
kubectl delete pod nginx-pod

# 通过文件删除（推荐）
kubectl delete -f pod-nginx.yaml
```

**⚠️ 关键理解：Pod 不会自愈**

```bash
# 删除一个由 Deployment 创建的 Pod → 会自动重建
# 删除一个独立 Pod → 它就真没了（不会复活）
kubectl delete pod nginx-pod  # 这就是"Pod 是短暂的"含义
```

### ✅ Day 2 学习成果检查

- [ ] 理解 Pod 是什么（最小部署单元、共享网络/存储）
- [ ] 会用 `kubectl run` 命令式创建 Pod
- [ ] 会写 YAML 并用 `kubectl apply -f` 声明式创建
- [ ] 会用 `kubectl logs`、`exec`、`port-forward`
- [ ] 理解 Pod 是短暂的，不会自愈
- [ ] 理解 Labels 的基本概念

---

## Day 3：Deployment — 管理 Pod 的生命周期

### 3.1 为什么需要 Deployment？

```
没有 Deployment：
  Pod A ─── 如果挂了 → 真的挂了 ❌
  
有 Deployment：
  Deployment 要求 3 个副本
  ├── Pod A  如果 A 挂了
  ├── Pod B  → Deployment 自动创建 Pod D
  └── Pod C  ✅ 始终保持 3 个
```

**Deployment 的功能：**
- 确保指定数量的 Pod 在运行（自愈）
- 支持滚动更新（逐步替换旧版本）
- 支持版本回滚
- 支持扩缩容

### 3.2 创建第一个 Deployment

创建 `deploy-nginx.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3                    # 期望 3 个副本
  selector:
    matchLabels:
      app: nginx                 # 管理带有 app: nginx 标签的 Pod
  template:
    metadata:
      labels:
        app: nginx               # Pod 的标签
    spec:
      containers:
        - name: nginx
          image: nginx:latest
          ports:
            - containerPort: 80
```

```bash
kubectl apply -f deploy-nginx.yaml

# 查看 Deployment
kubectl get deployments
# NAME               READY   UP-TO-DATE   AVAILABLE   AGE
# nginx-deployment   3/3     3            3           10s

# 查看背后的 ReplicaSet
kubectl get replicasets
# NAME                          DESIRED   CURRENT   READY   AGE
# nginx-deployment-7c79c4bf97   3         3         3       10s

# 查看它管理的 Pod（注意 Pod 名称的后缀）
kubectl get pods
# NAME                                READY   STATUS    RESTARTS   AGE
# nginx-deployment-7c79c4bf97-k4n7d   1/1     Running   0          20s
# nginx-deployment-7c79c4bf97-m8x3p   1/1     Running   0          20s
# nginx-deployment-7c79c4bf97-pq2jv   1/1     Running   0          20s
```

### 3.3 验证自愈能力

```bash
# 手动删除一个 Pod
kubectl delete pod nginx-deployment-7c79c4bf97-k4n7d

# 快速看变化
kubectl get pods -w
# 你会看到：旧的 Terminating → 新的 Pending → Running
# 始终保持在 3 个副本！
```

### 3.4 扩缩容

```bash
# 方法一：修改 YAML 文件中的 replicas，然后重新 apply
# 把 replicas: 3 改成 replicas: 5
kubectl apply -f deploy-nginx.yaml

# 方法二：命令式扩缩（快速测试用）
kubectl scale deployment nginx-deployment --replicas=5

# 观察变化
kubectl get pods

# 缩回去
kubectl scale deployment nginx-deployment --replicas=2
```

### 3.5 滚动更新

创建 `deploy-nginx-v2.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 4
  selector:
    matchLabels:
      app: nginx
  template:
    metadata:
      labels:
        app: nginx
    spec:
      containers:
        - name: nginx
          image: nginx:alpine       # 换成 alpine 版本（更小）
          ports:
            - containerPort: 80
```

```bash
# 执行滚动更新
kubectl apply -f deploy-nginx-v2.yaml

# 观察更新过程
kubectl rollout status deployment nginx-deployment
# Waiting for deployment "nginx-deployment" rollout to finish: 2 of 4 updated replicas...
# deployment "nginx-deployment" successfully rolled out

# 查看更新历史
kubectl rollout history deployment nginx-deployment
```

**滚动更新策略（可配置）：**

```yaml
spec:
  strategy:
    type: RollingUpdate         # 默认策略
    rollingUpdate:
      maxUnavailable: 1         # 更新中最多允许 1 个 Pod 不可用
      maxSurge: 1               # 可以多创建 1 个额外 Pod
```

另一种策略：

```yaml
spec:
  strategy:
    type: Recreate              # 先全部删除，再全部重建（有停机时间）
```

### 3.6 回滚

```bash
# 回滚到上一个版本
kubectl rollout undo deployment nginx-deployment

# 回滚到指定版本（先用 history 看版本号）
kubectl rollout history deployment nginx-deployment
kubectl rollout undo deployment nginx-deployment --to-revision=1
```

### 3.7 理解：Deployment → ReplicaSet → Pod

```
Deployment
  │  管理 ReplicaSet 的版本和更新策略
  ▼
ReplicaSet（nginx-deployment-7c79c4bf97）
  │  确保 N 个 Pod 在运行
  ▼
Pod（nginx-deployment-7c79c4bf97-xxxxx）
  │  承载容器
  ▼
Container（nginx）
```

- 每次更新镜像 → 创建新的 ReplicaSet
- 回滚 → 把流量切回旧 ReplicaSet
- 旧 ReplicaSet 的副本数归零，但保留 YAML 定义（方便再回滚）

### ✅ Day 3 学习成果检查

- [ ] 理解 Deployment → ReplicaSet → Pod 的层级关系
- [ ] 会创建 Deployment 并指定副本数
- [ ] 验证了 Pod 的自愈能力（删了自动重建）
- [ ] 会扩缩容（scale up/down）
- [ ] 会执行滚动更新和回滚
- [ ] 理解 RollingUpdate 和 Recreate 的区别

---

## Day 4：Service — 网络与访问

### 4.1 问题：为什么需要 Service？

```
场景：
  浏览器 ─？─→ Pod A（IP: 10.244.0.5）
              Pod B（IP: 10.244.0.6）
              Pod C（IP: 10.244.0.7）

问题 1：Pod IP 是动态的（重启就变）
问题 2：3 个副本之间如何负载均衡？
问题 3：外界如何访问集群内的 Pod？

答案：Service！
```

### 4.2 三种 Service 类型

| 类型 | 访问方式 | 用途 |
|------|---------|------|
| **ClusterIP**（默认）| 集群内部 IP 访问 | 内部服务间调用 |
| **NodePort** | `节点IP:端口` 访问 | 外部访问（开发测试） |
| **LoadBalancer** | 云厂商负载均衡器 | 对外暴露服务（生产） |

### 4.3 ClusterIP（内部访问）

创建 `service-clusterip.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-service
spec:
  selector:
    app: nginx               # 选择带有 app: nginx 标签的 Pod
  ports:
    - port: 80               # Service 的端口
      targetPort: 80         # 转发到 Pod 的端口
  type: ClusterIP
```

```bash
kubectl apply -f service-clusterip.yaml

# 查看 Service
kubectl get services
# NAME            TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)   AGE
# nginx-service   ClusterIP   10.96.123.45   <none>        80/TCP    10s

# 测试内部访问（启动一个临时 Pod 来发请求）
kubectl run test-pod --image=busybox -it --rm --restart=Never -- \
  wget -qO- http://nginx-service
# 或者用 nslookup 验证 DNS
kubectl run test-pod --image=busybox -it --rm --restart=Never -- \
  nslookup nginx-service
```

### 4.4 NodePort（外部访问）

创建 `service-nodeport.yaml`：

```yaml
apiVersion: v1
kind: Service
metadata:
  name: nginx-nodeport
spec:
  selector:
    app: nginx
  ports:
    - port: 80
      targetPort: 80
      nodePort: 30080           # 节点上的端口（范围 30000-32767，不指定则随机）
  type: NodePort
```

```bash
kubectl apply -f service-nodeport.yaml

# 查看 Service
kubectl get svc
# NAME             TYPE        CLUSTER-IP     EXTERNAL-IP   PORT(S)          AGE
# nginx-service    ClusterIP   10.96.123.45   <none>        80/TCP           5m
# nginx-nodeport   NodePort    10.96.67.89    <none>        80:30080/TCP     10s

# Kind 环境中访问 NodePort（需要端口映射）
# 先查一下 Kind 节点的 IP
kubectl get nodes -o wide

# 因为 Kind 节点也是容器，需要把容器端口映射到本机
docker ps
# 找到 kind-control-plane 容器的端口映射

# 或者用 kubectl port-forward 直接访问 Service
kubectl port-forward service/nginx-nodeport 8080:80
# 然后浏览器访问 http://localhost:8080 ✅
```

### 4.5 理解 Labels & Selector（重要！）

Service 通过 Label Selector 找到后端 Pod：

```
Service (nginx-service)
  selector:
    app: nginx          ← 匹配所有带 app: nginx 标签的 Pod
        │
        ▼
Pod A  ─── labels: {app: nginx, env: dev}      ← 匹配 ✅
Pod B  ─── labels: {app: nginx, env: prod}     ← 匹配 ✅
Pod C  ─── labels: {app: busybox}              ← 不匹配 ❌
```

**实战验证：**

```bash
# 查看 Service 后端有哪些 Pod
kubectl get endpoints nginx-service
# NAME            ENDPOINTS                                   AGE
# nginx-service   10.244.0.5:80,10.244.0.6:80,10.244.0.7:80  5m

# 给其中一个 Pod 改标签（观察它被移出 Service）
kubectl label pod nginx-deployment-xxxxx app=not-nginx --overwrite
kubectl get endpoints nginx-service  # 会少一个 Endpoint！

# 改回来
kubectl label pod nginx-deployment-xxxxx app=nginx --overwrite
```

### 4.6 Service 的负载均衡

```bash
# 在一个终端开循环请求
kubectl run test-pod --image=busybox -it --rm --restart=Never -- sh
# 在容器内执行：
# while true; do wget -qO- http://nginx-service; sleep 1; done

# 另一个终端看 Nginx 日志
kubectl logs -l app=nginx --tail=5 --prefix
```

### 4.7 完整练习：从 Deployment 到 Service

创建 `complete-demo.yaml`：

```yaml
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-world
spec:
  replicas: 3
  selector:
    matchLabels:
      app: hello
  template:
    metadata:
      labels:
        app: hello
    spec:
      containers:
        - name: hello
          image: hashicorp/http-echo:latest
          args: ["-text=Hello Kubernetes!"]
          ports:
            - containerPort: 5678
---
apiVersion: v1
kind: Service
metadata:
  name: hello-service
spec:
  selector:
    app: hello
  ports:
    - port: 80
      targetPort: 5678
  type: NodePort
```

```bash
kubectl apply -f complete-demo.yaml

# 测试
kubectl get pods
kubectl port-forward service/hello-service 8080:80
# 浏览器访问 http://localhost:8080 → 看到 "Hello Kubernetes!"
```

### ✅ Day 4 学习成果检查

- [ ] 理解 Service 解决了什么问题（动态 IP、负载均衡、外部访问）
- [ ] 理解 ClusterIP / NodePort / LoadBalancer 的用途
- [ ] 理解 Labels & Selector 的工作原理（能动手验证）
- [ ] 会使用 `kubectl get endpoints` 查看后端
- [ ] 会用 `kubectl port-forward` 访问 Service

---

## Day 5：ConfigMap & Secret — 配置管理

### 5.1 问题：硬编码的坏处

```yaml
# ❌ 硬编码配置
env:
  - name: DB_HOST
    value: "10.244.0.50"
  - name: DB_PORT
    value: "3306"
  - name: APP_MODE
    value: "production"

# ✅ 解耦配置
envFrom:
  - configMapRef:
      name: app-config
```

### 5.2 ConfigMap（非敏感配置）

#### 创建 ConfigMap

```bash
# 方式一：从字面量创建
kubectl create configmap app-config --from-literal=APP_MODE=dev \
  --from-literal=LOG_LEVEL=debug

# 方式二：从文件创建
echo "DB_HOST=localhost" > db.env
echo "DB_PORT=3306" >> db.env
kubectl create configmap db-config --from-env-file=db.env

# 方式三：声明式（推荐）
```

创建 `configmap-demo.yaml`：

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_MODE: dev
  LOG_LEVEL: debug
  config.json: |                 # 可以存完整配置文件
    {
      "timeout": 30,
      "retries": 3
    }
```

```bash
kubectl apply -f configmap-demo.yaml

# 查看
kubectl get configmaps
kubectl describe configmap app-config
```

#### 在 Pod 中使用 ConfigMap

创建 `pod-with-config.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-pod
spec:
  containers:
    - name: test
      image: busybox
      command: ["sh", "-c", "env | grep APP_ && sleep 3600"]
      envFrom:                      # 注入所有键值对
        - configMapRef:
            name: app-config
      env:
        - name: APP_MODE            # 也可以只注入特定键
          valueFrom:
            configMapKeyRef:
              name: app-config
              key: APP_MODE
```

```bash
kubectl apply -f pod-with-config.yaml

# 查看环境变量是否注入
kubectl logs config-pod
# APP_MODE=dev
# LOG_LEVEL=debug
```

#### 挂载为文件

创建 `pod-with-config-file.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: config-file-pod
spec:
  containers:
    - name: test
      image: busybox
      command: ["sh", "-c", "cat /etc/config/config.json && sleep 3600"]
      volumeMounts:
        - name: config-volume
          mountPath: /etc/config
  volumes:
    - name: config-volume
      configMap:
        name: app-config
```

### 5.3 Secret（敏感配置）

Secret 和 ConfigMap 用法几乎一样，但：

- 数据以 Base64 编码存储（不是加密，只是编码！）
- 适合存密码、API Key、证书
- 有更强的访问控制（RBAC）

```bash
# 创建 Secret（命令式）
kubectl create secret generic db-secret \
  --from-literal=DB_USER=admin \
  --from-literal=DB_PASSWORD=s3cret
```

创建 `secret-demo.yaml`：

```yaml
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque                # 通用类型
data:
  DB_USER: YWRtaW4=         # echo -n "admin" | base64
  DB_PASSWORD: czNjcmV0     # echo -n "s3cret" | base64
```

```bash
# 或者用 stringData（不用自己 base64）
kubectl apply -f - <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: db-secret
type: Opaque
stringData:
  DB_USER: admin
  DB_PASSWORD: s3cret
EOF
```

#### 在 Pod 中使用 Secret

创建 `pod-with-secret.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: secret-pod
spec:
  containers:
    - name: test
      image: busybox
      command: ["sh", "-c", "env | grep DB_ && sleep 3600"]
      envFrom:
        - secretRef:
            name: db-secret
```

```bash
kubectl apply -f pod-with-secret.yaml
kubectl logs secret-pod
# DB_USER=admin
# DB_PASSWORD=s3cret
```

### 5.4 完整练习：带配置的 Web 应用

创建 `hello-config.yaml`：

```yaml
---
apiVersion: v1
kind: ConfigMap
metadata:
  name: hello-config
data:
  HELLO_TEXT: "你好，Kubernetes！"
  HELLO_BG_COLOR: "#4CAF50"
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: hello-config-deploy
spec:
  replicas: 2
  selector:
    matchLabels:
      app: hello-config
  template:
    metadata:
      labels:
        app: hello-config
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo:latest
          args:
            - -text="$(HELLO_TEXT)"
          ports:
            - containerPort: 5678
          envFrom:
            - configMapRef:
                name: hello-config
---
apiVersion: v1
kind: Service
metadata:
  name: hello-config-svc
spec:
  selector:
    app: hello-config
  ports:
    - port: 80
      targetPort: 5678
  type: NodePort
```

```bash
kubectl apply -f hello-config.yaml
kubectl port-forward service/hello-config-svc 8080:80
# 访问 http://localhost:8080
```

### ✅ Day 5 学习成果检查

- [ ] 理解 ConfigMap 和 Secret 的用途
- [ ] 会创建 ConfigMap/Secret（命令式 + 声明式）
- [ ] 会以环境变量方式注入到 Pod
- [ ] 会以文件挂载方式注入到 Pod
- [ ] 理解 Secret 只是 Base64 编码不是真加密

---

## Day 6：存储卷（Volumes & PVC）

### 6.1 问题：容器重启后数据会丢

```
容器日志写入 /var/log/app.log
         │
Pod 重启 → 新容器从镜像重新创建
         │
/var/log 是容器内部文件系统 → 日志丢了！😱
         │
解决方案 → Volume（卷）
```

### 6.2 emptyDir（临时存储）

Pod 内的容器共享，Pod 删除时数据消失。

创建 `pod-emptydir.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: shared-vol-pod
spec:
  containers:
    - name: writer
      image: alpine
      command: ["sh", "-c", "while true; do echo $(date) >> /data/timestamps.log; sleep 2; done"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
    - name: reader
      image: alpine
      command: ["sh", "-c", "tail -f /data/timestamps.log"]
      volumeMounts:
        - name: shared-data
          mountPath: /data
  volumes:
    - name: shared-data
      emptyDir: {}
```

```bash
kubectl apply -f pod-emptydir.yaml

# 查看读者容器的输出（能看到写入容器的每 2 秒追加的时间戳）
kubectl logs -f shared-vol-pod -c reader
```

> 💡 典型用途：Sidecar 共享日志、缓存、临时文件交换

### 6.3 hostPath（节点本地存储）

把宿主机目录挂载到 Pod（Kind 里是 Docker 容器内部路径）。

> ⚠️ hostPath 在单节点学习环境中很方便，但**生产环境不推荐**（Pod 调度到不同节点就找不到数据了）。

创建 `pod-hostpath.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: hostpath-pod
spec:
  containers:
    - name: test
      image: alpine
      command: ["sh", "-c", "echo 'Hello from hostPath!' > /host/data.txt; sleep 3600"]
      volumeMounts:
        - name: host-storage
          mountPath: /host
  volumes:
    - name: host-storage
      hostPath:
        path: /tmp/k8s-data
        type: DirectoryOrCreate
```

```bash
kubectl apply -f pod-hostpath.yaml

# 进入 Kind 节点容器确认数据真写到了宿主机
docker exec kind-control-plane cat /tmp/k8s-data/data.txt
# Hello from hostPath!
```

### 6.4 PV & PVC（持久化存储 — 重点！）

**核心概念：**

```
管理员                            用户
  │                                │
  ▼                                ▼
PersistentVolume  ←── 绑定 ──→  PersistentVolumeClaim  ←── 使用 ──→ Pod
（存储资源）       │           （存储申请）
                  │
           StorageClass（动态供给）
```

- **PV（PersistentVolume）**：集群级别的存储资源（就像是一个"云硬盘"）
- **PVC（PersistentVolumeClaim）**：用户对存储的申请（"我要 1GB 的快速存储"）
- Pod 通过 PVC 来使用存储

#### 在 Kind 中体验 PV + PVC

创建 `pv-demo.yaml`：

```yaml
---
# 创建一个 PV（使用 hostPath 模拟，实际环境用云硬盘）
apiVersion: v1
kind: PersistentVolume
metadata:
  name: local-pv
spec:
  capacity:
    storage: 1Gi
  accessModes:
    - ReadWriteOnce          # 单节点读写
  hostPath:
    path: /tmp/k8s-pv-data   # Kind 节点容器的路径
---
# 申请使用 500Mi
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: local-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 500Mi
---
# Pod 使用 PVC
apiVersion: v1
kind: Pod
metadata:
  name: pv-pod
spec:
  containers:
    - name: app
      image: alpine
      command: ["sh", "-c", "echo '持久化数据！' > /data/test.txt; cat /data/test.txt; sleep 3600"]
      volumeMounts:
        - name: storage
          mountPath: /data
  volumes:
    - name: storage
      persistentVolumeClaim:
        claimName: local-pvc
```

```bash
kubectl apply -f pv-demo.yaml

# 查看状态
kubectl get pv
# NAME       CAPACITY   ACCESS MODES   RECLAIM POLICY   STATUS   CLAIM
# local-pv   1Gi        RWO            Retain           Bound    default/local-pvc

kubectl get pvc
# NAME        STATUS   VOLUME     CAPACITY   ACCESS MODES
# local-pvc   Bound    local-pv   1Gi        RWO

# 验证数据写入
kubectl logs pv-pod
# 持久化数据！

# 删除 Pod 和 PVC
kubectl delete pod pv-pod
kubectl delete pvc local-pvc
kubectl delete pv local-pv
```

> 💡 **PV 的回收策略（Reclaim Policy）：**
> - `Retain`：PVC 删除后，PV 保留数据（需要管理员手动清理）
> - `Delete`：PVC 删除后，PV 和数据一起删除
> - `Recycle`：PVC 删除后，清空数据再复用（已废弃）

### 6.5 Kind 中的 StorageClass（动态供给）

Kind 自带一个 `standard` StorageClass，支持动态 PV 供给：

```bash
# 查看默认 StorageClass
kubectl get storageclass
# NAME                 PROVISIONER             RECLAIMPOLICY   VOLUMEBINDINGMODE
# standard (default)   rancher.io/local-path   Delete          WaitForFirstConsumer
```

使用动态 PVC 不需要先创建 PV：

```yaml
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: dynamic-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
  storageClassName: standard    # 明确使用哪个 StorageClass
```

### 6.6 实战：Redis 数据持久化

创建 `redis-with-pvc.yaml`：

```yaml
---
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-data
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
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
          volumeMounts:
            - name: data
              mountPath: /data
  volumeClaimTemplates: []
---
# 这里用完整的写法带 volume
```

简化版本（Deployment 里直接引用已创建的 PVC）：

> 注意：该 Deployment 依赖一个已存在的 PVC `redis-data`。
> 如果希望 YAML 文件自包含，需要在同一个多文档 YAML 中先定义 `PersistentVolumeClaim`。

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
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
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: redis-data
---
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:
    app: redis
  ports:
    - port: 6379
```

```bash
kubectl apply -f redis-with-pvc.yaml

# 往 Redis 写数据
kubectl run -it --rm redis-client --image=redis:7-alpine -- redis-cli -h redis
# > SET visits 100
# > GET visits
# > SAVE
# > exit

# 删除 Redis Pod（验证持久化）
kubectl delete pod -l app=redis
# 新的 Pod 启动后，之前的数据还在！
```

### ✅ Day 6 学习成果检查

- [ ] 理解 emptyDir、hostPath 的场景和局限
- [ ] 理解 PV / PVC / StorageClass 的关系
- [ ] 会创建 PVC 并在 Pod 中使用
- [ ] 验证了 Redis 数据持久化（Pod 重启数据不丢）

---

## Day 7：健康检查与故障排查

### 7.1 问题场景

```
Pod 启动了，但：
  - 应用还在初始化（没准备好接受流量）
  - 应用僵死了（进程还在但无响应）
  - 初始化需要下载资源，很慢

K8s 需要知道容器到底"活着"和"准备好了"没有。
```

### 7.2 三种探针（Probe）

| 探针 | 作用 | 如果失败 |
|------|------|---------|
| **Liveness Probe**（存活） | 检查容器是否活着 | 重启容器 |
| **Readiness Probe**（就绪） | 检查容器能否处理请求 | 从 Service 摘除 |
| **Startup Probe**（启动） | 检查容器是否已完成初始化 | 杀死重启（慢启动应用用） |

```
         ┌──────────────────────┐
         │   Pod 生命周期       │
         │                      │
         │   ┌──────────┐      │
         │   │  Pending  │      │  ← 正在调度/拉镜像
         │   └────┬─────┘      │
         │        ▼            │
         │   ┌──────────┐      │
         │   │  Running  │      │
 Start ──┼──►│ (启动中)  │      │  ← Startup Probe 通过后进入就绪检查
         │   └────┬─────┘      │
         │        ▼            │
         │   ┌──────────┐      │
         │   │  Running  │      │
         │   │ (就绪)    │      │  ← 加入 Service 后端
         │   └──────────┘      │
         │                      │
         │   ↑ Liveness Probe   │  ← 每隔 N 秒检查
         │   ↑ Readiness Probe  │  ← 每隔 N 秒检查
         └──────────────────────┘
```

### 7.3 完整示例

创建 `probe-demo.yaml`：

```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: probe-demo
spec:
  replicas: 2
  selector:
    matchLabels:
      app: probe-demo
  template:
    metadata:
      labels:
        app: probe-demo
    spec:
      containers:
        - name: app
          image: hashicorp/http-echo:latest
          args: ["-text=Healthy!"]
          ports:
            - containerPort: 5678
          livenessProbe:                          # 存活检查
            httpGet:
              path: /
              port: 5678
            initialDelaySeconds: 5                # 启动后等 5 秒再开始检查
            periodSeconds: 10                     # 每 10 秒检查一次
            timeoutSeconds: 3                     # 超时 3 秒
            failureThreshold: 3                   # 连续 3 次失败才重启
          readinessProbe:                         # 就绪检查
            httpGet:
              path: /
              port: 5678
            initialDelaySeconds: 3
            periodSeconds: 5
      ```


```bash
kubectl apply -f probe-demo.yaml

# 观察 Pod 状态变化
kubectl get pods -w
# 你会看到：Pending → Running(0/1 ready) → Running(1/1 ready)
#            启动中             还没就绪        就绪完成 ✅

# 检查详情（会显示 Probe 信息）
kubectl describe pod probe-demo-xxxxx
# 在 Events 中可以看到：
#   Readiness probe ...   (就绪检查通过)
#   Liveness probe ...    (存活检查通过)
```

### 7.4 非 HTTP 探针

```yaml
# TCP 探针（适用于非 HTTP 服务，如 Redis）
livenessProbe:
  tcpSocket:
    port: 6379
  initialDelaySeconds: 5
  periodSeconds: 10

# 命令探针（通用，执行自定义脚本）
livenessProbe:
  exec:
    command:
      - cat
      - /tmp/healthy
  initialDelaySeconds: 5
  periodSeconds: 10
```

### 7.5 故障排查工具箱

```bash
# 1️⃣ 查看 Pod 当前状态
kubectl get pods

# 2️⃣ 查看详情（事件、原因、上次状态）
kubectl describe pod <pod-name>

# 3️⃣ 查看日志
kubectl logs <pod-name>
kubectl logs --previous <pod-name>    # 查看上次容器的日志（崩溃后）

# 4️⃣ 跟踪事件（全集群）
kubectl get events --sort-by='.lastTimestamp'

# 5️⃣ 查看 Pod YAML（完整配置）
kubectl get pod <pod-name> -o yaml

# 6️⃣ 进入 Pod 内部调试
kubectl exec -it <pod-name> -- sh

# 7️⃣ 临时调试 Pod（像 Sidecar 一样共享网络）
kubectl debug -it <pod-name> --image=busybox --target=<container-name>

# 8️⃣ 查看节点资源
kubectl top nodes
kubectl top pods
```

### 7.6 模拟故障练习

```bash
# 创建一个带健康检查的 Pod
kubectl apply -f - <<EOF
apiVersion: v1
kind: Pod
metadata:
  name: probe-test
spec:
  containers:
    - name: app
      image: alpine
      command:
        - sh
        - -c
        - |
          # 启动时创建 /tmp/healthy，30 秒后删除（模拟故障）
          touch /tmp/healthy
          sleep 30
          rm /tmp/healthy
          sleep 99999
      livenessProbe:
        exec:
          command:
            - cat
            - /tmp/healthy
        initialDelaySeconds: 5
        periodSeconds: 5
EOF

# 观察：30 秒后探针失败 → 容器被重启
kubectl get pods -w
# probe-test   1/1     Running
# probe-test   0/1     Running   (探针失败，但容器还在)
# probe-test   1/1     Running   (重启完成，重新创建了 /tmp/healthy)
```

### ✅ Day 7 学习成果检查

- [ ] 理解 Liveness / Readiness / Startup 三种探针的区别
- [ ] 会配置 HTTP 和 TCP 探针
- [ ] 会使用 `kubectl describe` / `logs` / `events` 排查问题
- [ ] 验证了健康检查自动重启容器的行为

---

## Day 8：整合 Demo — 完整应用部署

### 🎯 目标

把之前学的内容整合起来，部署一个完整的 **计数访问应用**：

```
                  ┌─────────┐
 浏览器 ──→  NodePort ──→  Flask Web App ──→ Redis（计数器）
                  │                        │
                  │                   ┌────┴────┐
                  │                   │  PVC    │ ← 数据持久化
                  │                   └─────────┘
                  │
             ┌────┴────┐
             │ConfigMap│ ← 应用配置
             └─────────┘
```

### 8.1 构建 Flask 应用

```bash
mkdir -p ~/k8s-demo && cd ~/k8s-demo
```

创建 `app.py`：

```python
import os
import redis
from flask import Flask

app = Flask(__name__)

redis_host = os.getenv("REDIS_HOST", "redis")
redis_port = int(os.getenv("REDIS_PORT", "6379"))
app_name = os.getenv("APP_NAME", "K8s Demo")
bg_color = os.getenv("BG_COLOR", "#f0f0f0")

r = redis.Redis(host=redis_host, port=redis_port, decode_responses=True)

@app.route("/")
def hello():
    try:
        count = r.incr("visits")
        return f"""
        <html>
        <body style="background:{bg_color}; 
                     font-family:sans-serif; 
                     display:flex; 
                     justify-content:center; 
                     align-items:center; 
                     height:100vh;">
          <div style="text-align:center; background:white; padding:40px; border-radius:10px;
                      box-shadow: 0 4px 6px rgba(0,0,0,0.1);">
            <h1 style="color:#333;">{app_name}</h1>
            <p style="font-size:48px; margin:20px 0; color:#4CAF50;">{count}</p>
            <p style="color:#666;">页面访问次数</p>
          </div>
        </body>
        </html>
        """
    except Exception as e:
        return f"<h1>Redis 连接失败: {e}</h1>", 500

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

创建 `requirements.txt`：

```
flask>=3.0
redis>=5.0
```

创建 `Dockerfile`：

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install --no-cache-dir -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

### 8.2 把镜像加载到 Kind

Kind 集群默认无法直接访问本地 Docker 镜像，需要手动加载：

```bash
# 构建镜像
docker build -t k8s-demo-app .

# 加载到 Kind 集群
kind load docker-image k8s-demo-app

# 验证
docker images k8s-demo-app
```

### 8.3 创建 K8s 资源文件

创建 `deploy-all.yaml`（一个文件包含所有资源）：

```yaml
---
# ────────── Redis 持久化存储 ──────────
apiVersion: v1
kind: PersistentVolumeClaim
metadata:
  name: redis-pvc
spec:
  accessModes:
    - ReadWriteOnce
  resources:
    requests:
      storage: 1Gi
---
# ────────── ConfigMap ──────────
apiVersion: v1
kind: ConfigMap
metadata:
  name: app-config
data:
  APP_NAME: "K8s 计数 Demo"
  BG_COLOR: "#e8f5e9"
---
# ────────── Redis Deployment ──────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: redis
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
          livenessProbe:
            tcpSocket:
              port: 6379
            initialDelaySeconds: 5
            periodSeconds: 10
          readinessProbe:
            tcpSocket:
              port: 6379
            initialDelaySeconds: 3
            periodSeconds: 5
          volumeMounts:
            - name: data
              mountPath: /data
      volumes:
        - name: data
          persistentVolumeClaim:
            claimName: redis-pvc
---
# ────────── Redis Service ──────────
apiVersion: v1
kind: Service
metadata:
  name: redis
spec:
  selector:
    app: redis
  ports:
    - port: 6379
  type: ClusterIP
---
# ────────── Flask Deployment ──────────
apiVersion: apps/v1
kind: Deployment
metadata:
  name: flask-app
spec:
  replicas: 2
  selector:
    matchLabels:
      app: flask
  template:
    metadata:
      labels:
        app: flask
    spec:
      containers:
        - name: flask
          image: k8s-demo-app
          ports:
            - containerPort: 5000
          envFrom:
            - configMapRef:
                name: app-config
          env:
            - name: REDIS_HOST
              value: redis
          livenessProbe:
            httpGet:
              path: /
              port: 5000
            initialDelaySeconds: 10
            periodSeconds: 15
          readinessProbe:
            httpGet:
              path: /
              port: 5000
            initialDelaySeconds: 5
            periodSeconds: 5
---
# ────────── Flask Service ──────────
apiVersion: v1
kind: Service
metadata:
  name: flask-service
spec:
  selector:
    app: flask
  ports:
    - port: 80
      targetPort: 5000
  type: NodePort
```

### 8.4 一键部署

```bash
# 部署全部
kubectl apply -f deploy-all.yaml

# 观察启动过程
kubectl get pods -w

# 所有 Pod 都 Running 后
kubectl get pods
# NAME                          READY   STATUS    RESTARTS   AGE
# flask-app-6b5f8f7d89-9nwpl   1/1     Running   0          30s
# flask-app-6b5f8f7d89-vxf8j   1/1     Running   0          30s
# redis-7d8b6f85b6-kxvqf       1/1     Running   0          45s

# 访问应用
kubectl port-forward service/flask-service 8080:80
# 浏览器打开 http://localhost:8080 🎉
```

### 8.5 验证各项功能

```bash
# 1. 访问计数（每刷新一次 +1）
# 打开 http://localhost:8080

# 2. 配置注入验证
kubectl get configmap app-config -o yaml

# 3. 验证自愈
kubectl delete pod -l app=flask
kubectl get pods -w
# 自动重建，且 Redis 计数不丢

# 4. 验证数据持久化
kubectl delete pod -l app=redis
kubectl get pods -w
# Redis 重建后，之前已有的计数还在

# 5. 更新配置（实时生效需要重建 Pod）
kubectl patch configmap app-config -p '{"data":{"BG_COLOR":"#ffebee"}}'
kubectl delete pod -l app=flask    # 重启让 ConfigMap 生效

# 6. 扩缩容
kubectl scale deployment flask-app --replicas=5
kubectl get pods -w
```

### 8.6 清理

```bash
# 删除 Demo
kubectl delete -f deploy-all.yaml
kubectl delete pvc --all    # 清理 PVC（数据会丢）

# 或者删除整个集群
kind delete cluster
```

### ✅ Day 8 学习成果检查

- [ ] 构建了完整的 Demo 应用（Flask + Redis）
- [ ] 把本地镜像加载到 Kind 集群
- [ ] 验证了 ConfigMap、PVC、Probe 等功能的集成
- [ ] 验证了计数持久化（Redis Pod 重启后数据不丢）
- [ ] 验证了应用自愈（Flask Pod 删除后自动重建）

---

## Bonus：尝试 AWS EKS（可选）

> 🎯 本节目标：在 **东京（ap-northeast-1）** 区域创建 **2 个 EC2 节点** 的 EKS 集群，模拟真实运维场景。
> 你将实操：多节点调度、节点维护（Cordon/Drain）、污点与容忍度、Cluster Autoscaler — 这些是 K8s 管理员每天都会面对的操作。

---

### 0. 💰 先看预算：在东京跑一个 EKS 集群要花多少钱？

> ⚠️ 你已升级为 AWS 付费用户，以下为 **ap-northeast-1（东京）** 区域的**按量计费**价格，供你决策时参考。

| 资源 | 规格 | 单价 | 月估费（730 h） | 说明 |
| --- | --- | --- | --- | --- |
| **EKS 控制平面** | 每集群 | $0.10/h | **$73.00** | 只要集群存在就收费，与节点数无关 |
| **EC2 节点×2** | t3.small（2vCPU/2GB） | ~$0.0252/h×2 | **~$36.79** | 学习够用，见下文选项 |
| **EC2 节点×2** | t3.medium（2vCPU/4GB） | ~$0.0504/h×2 | **~$73.58** | 推荐 — 跑完整 Demo 更宽裕 |
| **EBS 根卷×2** | gp3, 30 GB | $0.09/GB-month×60GB | **~$5.40** | 每个节点默认 30 GB |
| **ALB**（LoadBalancer） | 1 个 | ~$0.0252/h+LCU | **~$22** | 仅当创建 `type: LoadBalancer` 时产生 |
| **NAT Gateway** | 1 个（eksctl 默认创建） | ~$0.062/h | **~$45.26** | ⚠️ 这是容易被忽略的大头！见下文 |

**🔑 省钱的三个关键：**

1. **节点类型选择** — 你的场景是学习，推荐 **t3.small（$0.025/h）** 或 **t3.medium（$0.05/h）**，性价比最高。
2. **用完即删** — EKS 集群即使没有跑 Pod，控制平面的 $0.10/h 也在计费。**不使用时务必 `eksctl delete cluster`。**
3. **注意 NAT Gateway** — eksctl 默认创建的 VPC 带 NAT Gateway（~$45/月），如果你用 `--vpc-nat-mode Disable` 可以省掉，但私网节点将无法访问外网（拉镜像需要加 `--with-oidc` 配合 ECR 或其他方案）。为简化，下面的教程使用 eksctl 默认配置（含 NAT）。

**📊 不同配置的月成本估算（东京区域）：**

| 配置 | EKS | EC2×2 | EBS×2 | ALB | NAT | **月合计** |
| --- | --- | --- | --- | --- | --- | --- |
| 🟢 **最省：2×t3.small + 不创建 LB** | $73 | $36.79 | $5.40 | $0 | $45 | **~$160** |
| 🟡 **推荐：2×t3.medium + 创建 LB 体验** | $73 | $73.58 | $5.40 | $22 | $45 | **~$219** |
| 🔴 **2×t3.large + LB** | $73 | $147.20 | $5.40 | $22 | $45 | **~$293** |

> 💡 **短期体验建议**：完整跟着教程走一遍大约 2-3 小时，用 t3.small 节点 ≈ 花费 **$0.10（EKS）+ $0.05（EC2）× 3h = $0.25**。加上 EBS 和 NAT 的按小时比例 ≈ **不到 $1**。重点是**用完立刻删除**。

---

### 1. 前置准备

#### 安装/更新工具

```bash
# 安装或更新 eksctl
brew install eksctl
# 或升级
brew upgrade eksctl

# 安装 AWS CLI（如果还没有）
brew install awscli

# 验证
eksctl version
aws --version
kubectl version --client
```

#### 配置 AWS 凭证

```bash
# 确保配置了东京区域
aws configure
# AWS Access Key ID [****************]: <你的 Key>
# AWS Secret Access Key [****************]: <你的 Secret>
# Default region name [us-east-1]: ap-northeast-1    ← 设为东京！
# Default output format [json]: json

# 验证身份
aws sts get-caller-identity
```

---

### 2. 在东京创建 EKS 集群（2 节点）

#### 方式 A：一键创建（快速，适合首次体验）

```bash
eksctl create cluster \
  --name my-k8s-tokyo \
  --region ap-northeast-1 \
  --nodegroup-name standard-workers \
  --node-type t3.medium \
  --nodes 2 \
  --nodes-min 2 \
  --nodes-max 4 \
  --managed \
  --version 1.31

# ⏳ 等待 15-20 分钟...
# 这个期间 eksctl 在帮你做：
#   1. 创建 CloudFormation 堆栈（VPC、子网、安全组）
#   2. 创建 EKS 控制平面
#   3. 创建托管节点组（Auto Scaling Group + 2 台 EC2）
#   4. 配置 aws-auth ConfigMap
#   5. 写入 kubeconfig
```

#### 方式 B：使用 YAML 配置文件（推荐，可重复使用）

创建 `eks-cluster.yaml`：

```yaml
apiVersion: eksctl.io/v1alpha5
kind: ClusterConfig

metadata:
  name: my-k8s-tokyo
  region: ap-northeast-1          # 东京区域
  version: "1.31"

managedNodeGroups:
  - name: workers-1a              # 第一可用区
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 2
    maxSize: 4
    availabilityZones: ["ap-northeast-1a"]
    labels:
      role: worker
      zone: az1
    tags:
      Environment: learning

  - name: workers-1c              # 第二可用区 — 高可用！
    instanceType: t3.medium
    desiredCapacity: 2
    minSize: 2
    maxSize: 4
    availabilityZones: ["ap-northeast-1c"]
    labels:
      role: worker
      zone: az2
    tags:
      Environment: learning

# 如果你想省 NAT Gateway 费用（但注意：私网节点无法直接从外网拉镜像）
# vpc:
#   nat:
#     mode: Disable
```

```bash
# 用配置文件创建
eksctl create cluster -f eks-cluster.yaml

# 这种方式好处：可版本控制，重复创建一致的环境
```

#### 验证集群

```bash
# 查看 kubectl 上下文
kubectl config current-context
# 输出类似：arn:aws:eks:ap-northeast-1:123456789:cluster/my-k8s-tokyo

# 查看节点
kubectl get nodes -o wide
# NAME                                                STATUS   ROLES    AGE   VERSION    INTERNAL-IP    EXTERNAL-IP     OS-IMAGE
# ip-192-168-11-22.ap-northeast-1.compute.internal    Ready    <none>   5m    v1.31.xx   192.168.11.22   xx.xx.xx.xx    Amazon Linux 2
# ip-192-168-33-44.ap-northeast-1.compute.internal    Ready    <none>   5m    v1.31.xx   192.168.33.44   xx.xx.xx.xx    Amazon Linux 2

# 查看 AWS 上对应的 EC2（在另一个终端验证）
aws ec2 describe-instances --region ap-northeast-1 \
  --filters "Name=tag:eks:cluster-name,Values=my-k8s-tokyo" \
  --query "Reservations[].Instances[].{ID:InstanceId,Type:InstanceType,State:State.Name,IP:PrivateIpAddress}" \
  --output table
```

---

### 3. 第一天：认识你的集群

#### 3.1 看节点详情

```bash
# 节点标签
kubectl describe node ip-192-168-11-22.ap-northeast-1.compute.internal | grep -A 5 "Labels:"

# 节点资源容量
kubectl describe node ip-192-168-11-22.ap-northeast-1.compute.internal | grep -A 10 "Capacity:"

# 节点上的 Pod
kubectl get pods --all-namespaces -o wide --field-selector spec.nodeName=ip-192-168-11-22.ap-northeast-1.compute.internal
```

#### 3.2 通过 SSM Session Manager 连接到 EC2（无需 SSH 密钥）

AWS 托管的 EKS 节点默认安装了 SSM Agent，推荐使用 Session Manager 连接：

```bash
# 先找到节点的实例 ID（注意把 workers-1a 换成你的节点组名）
INSTANCE_ID=$(aws ec2 describe-instances \
  --region ap-northeast-1 \
  --filters "Name=tag:eks:nodegroup-name,Values=workers-1a" \
  --query "Reservations[*].Instances[*].InstanceId" \
  --output text)
echo $INSTANCE_ID

# 启动 Session Manager 会话
aws ssm start-session --region ap-northeast-1 --target $INSTANCE_ID

# 进入后可以：
sh-5.2$ sudo -i              # 切 root
sh-5.2$ crictl ps            # 查看运行中的容器（containerd 容器运行时）
sh-5.2$ crictl images        # 查看已拉取的镜像
sh-5.2$ ctr -n k8s.io container ls   # crictl 的替代（某些节点可能未装 crictl）
sh-5.2$ ctr -n k8s.io image ls       # 用 containerd 原生 CLI 查看镜像
sh-5.2$ journalctl -u kubelet --no-pager | tail -20   # 查看 kubelet 日志
sh-5.2$ cat /var/log/containers/*.log | tail -20      # 查看具体容器日志
sh-5.2$ lsblk                # 查看磁盘（能看到 EBS 卷）
sh-5.2$ exit                 # 退出 SSM
```

> 💡 SSM Session Manager 的好处：不需要管理 SSH 密钥对，基于 IAM 权限控制，所有操作都有 CloudTrail 审计日志。

#### 3.3 在 AWS 控制台查看

在浏览器打开（或直接点击以下链接）：

- **EKS 控制台** — 查看集群状态和节点组
- **EC2 控制台** — 查看实际的虚拟机实例
- **CloudFormation** — 查看 eksctl 创建的堆栈

> 链接：`https://console.aws.amazon.com/eks/home?region=ap-northeast-1`

---

### 4. 第二天：真实世界的多节点运维

#### 4.1 部署我们之前造的 Demo 应用

```bash
# 在本地重建 ~/k8s-demo 或直接使用公共 Nginx 镜像
kubectl create deployment web-demo --image=nginx:alpine --replicas=6

# 看看 Pod 是怎么分布在 2 个节点上的
kubectl get pods -o wide
# NAME                       READY   STATUS    RESTARTS   AGE   IP               NODE
# web-demo-6b5f8f7d89-9abc   1/1     Running   0          10s   192.168.11.50    ip-192-168-11-22
# web-demo-6b5f8f7d89-9def   1/1     Running   0          10s   192.168.33.60    ip-192-168-33-44
# web-demo-6b5f8f7d89-9ghi   1/1     Running   0          10s   192.168.33.61    ip-192-168-33-44
# web-demo-6b5f8f7d89-9jkl   1/1     Running   0          10s   192.168.11.51    ip-192-168-11-22
# ...

# 看到没？K8s Scheduler 自动把 6 个副本分布到 2 个节点！
```

#### 4.2 🛠️ 节点维护：Cordon & Drain（日常高频操作！）

**场景**：给一台 EC2 打安全补丁，需要将上面的 Pod 赶走。

```bash
# 第一步：标记节点为不可调度（新的 Pod 不会调度上来）
kubectl cordon ip-192-168-33-44.ap-northeast-1.compute.internal
# node/ip-192-168-33-44.ap-northeast-1.compute.internal cordoned

# 查看节点状态（变为 SchedulingDisabled）
kubectl get nodes
# NAME                     STATUS                     ROLES    AGE
# ip-192-168-11-22         Ready                      <none>   1h
# ip-192-168-33-44         Ready,SchedulingDisabled   <none>   1h   ← 注意变化

# 第二步：驱逐该节点上的 Pod（滚动到另一个节点）
kubectl drain ip-192-168-33-44.ap-northeast-1.compute.internal \
  --ignore-daemonsets \
  --delete-emptydir-data

# 观察 Pod 迁移
kubectl get pods -o wide -w
# 你会看到：被驱逐的 Pod Terminating → 新的 Pod 在另一个节点上 Running ✅

# 假设补丁打完了，恢复节点
kubectl uncordon ip-192-168-33-44.ap-northeast-1.compute.internal
# node/ip-192-168-33-44.ap-northeast-1.compute.internal uncordoned

# 新的 Pod 现在又可以调度上去了
```

> ⚠️ **关键理解**：`cordon` 只阻止新 Pod 调度，`drain` 才会驱逐已有 Pod。
> 生产流程：`cordon → drain → 运维操作 → uncordon`

#### 4.3 🏷️ Taints & Tolerations（给节点打污点）

**场景**：有些节点是"特殊"的（比如有 GPU、SSD），只有特定应用能用。

```bash
# 给一个节点打污点：只有 toleration 的 Pod 才能上去
kubectl taint nodes ip-192-168-11-22.ap-northeast-1.compute.internal dedicated=gpu:NoSchedule

# 创建一个没有 toleration 的 Pod → 它不会被调度到该节点
kubectl run test-regular --image=nginx --restart=Never
kubectl get pod test-regular -o wide  # 你会发现它在另一个节点上

# 创建带有 toleration 的 Pod → 它可以上去
cat <<EOF | kubectl apply -f -
apiVersion: v1
kind: Pod
metadata:
  name: test-gpu
spec:
  tolerations:
    - key: dedicated
      operator: Equal
      value: gpu
      effect: NoSchedule
  containers:
    - name: nginx
      image: nginx:alpine
EOF

kubectl get pod test-gpu -o wide  # Pod 被调度到了带污点的节点 ✅

# 清理污点
kubectl taint nodes ip-192-168-11-22.ap-northeast-1.compute.internal dedicated=gpu:NoSchedule-
```

#### 4.4 🎯 Node Affinity（偏好调度）

**场景**：你想让某些 Pod 优先跑到某台机器上（比如离 Redis 近的节点）。

```bash
# 给节点贴标签
kubectl label nodes ip-192-168-11-22.ap-northeast-1.compute.internal disk-type=ssd

# 创建节点亲和性的 Deployment
cat <<EOF | kubectl apply -f -
apiVersion: apps/v1
kind: Deployment
metadata:
  name: affinity-demo
spec:
  replicas: 3
  selector:
    matchLabels:
      app: affinity-demo
  template:
    metadata:
      labels:
        app: affinity-demo
    spec:
      affinity:
        nodeAffinity:
          preferredDuringSchedulingIgnoredDuringExecution:
            - weight: 100
              preference:
                matchExpressions:
                  - key: disk-type
                    operator: In
                    values:
                      - ssd
      containers:
        - name: app
          image: nginx:alpine
EOF

# 观察调度
kubectl get pods -o wide
```

> 💡 nodeAffinity 常用类型：
> - `requiredDuringSchedulingIgnoredDuringExecution`：**硬性要求**，必须匹配
> - `preferredDuringSchedulingIgnoredDuringExecution`：**软偏好**，尽量匹配

#### 4.5 📈 演示 Cluster Autoscaler

**场景**：流量突增，Pod 太多，现有节点不够了 → 自动增加 EC2。

EKS 托管节点组自带 Cluster Autoscaler 支持：

```bash
# 1. 安装 IAM OIDC Provider（搭桥，让 AWS 信任 EKS 签发的 Token）
eksctl utils associate-iam-oidc-provider \
  --region ap-northeast-1 \
  --cluster my-k8s-tokyo \
  --approve

# 2. （⚠️ 易遗漏！）创建 IAM 角色并绑定到 Service Account
#    这样 Cluster Autoscaler Pod 才能拿到权限去调用 AWS API 创建 EC2
eksctl create iamserviceaccount \
  --cluster my-k8s-tokyo \
  --namespace kube-system \
  --name cluster-autoscaler \
  --attach-policy-arn arn:aws:iam::aws:policy/AmazonEKSClusterAutoscalerPolicy \
  --override-existing-serviceaccounts \
  --approve

# 3. 部署 Cluster Autoscaler
kubectl apply -f https://raw.githubusercontent.com/kubernetes/autoscaler/master/cluster-autoscaler/cloudprovider/aws/examples/cluster-autoscaler-autodiscover.yaml

# 4. 给 Cluster Autoscaler 打上注解（指定集群名）
kubectl -n kube-system annotate deployment.apps/cluster-autoscaler \
  cluster-autoscaler.kubernetes.io/safe-to-evict="false"

# 5. 编辑 Deployment，将 <YOUR CLUSTER NAME> 替换为 my-k8s-tokyo
kubectl -n kube-system edit deployment.apps/cluster-autoscaler
# 找到 --node-group-auto-discovery 那一行，修改 clusterName

# 6. 触发扩容测试
kubectl scale deployment web-demo --replicas=50

# 观察节点数自动增加
kubectl get nodes -w
# 几分钟后你会看到新的 EC2 被创建并加入集群 🚀

# 7. 缩回去
kubectl scale deployment web-demo --replicas=3
# 等待几分钟，多余节点会被 Cluster Autoscaler 自动回收
```

#### 4.6 从本地 Kind 切换到 EKS

```bash
# 查看所有上下文（EKS 上下文名格式：<用户名>@<集群名>.<区域>.eksctl.io）
kubectl config get-contexts

# 切换到 EKS（把 <用户名> 换成你的 IAM 用户名）
kubectl config use-context <用户名>@my-k8s-tokyo.ap-northeast-1.eksctl.io

# 切回 Kind（把 my-cluster 换成你的 Kind 集群名）
kubectl config use-context kind-my-cluster

# 或者用别名快速切换（先查好你的上下文名）
alias k8s-kind='kubectl config use-context kind-my-cluster'
alias k8s-eks='kubectl config use-context <用户名>@my-k8s-tokyo.ap-northeast-1.eksctl.io'

# 加入 shell 配置文件（zshrc）方便以后使用
echo 'alias k8s-kind="kubectl config use-context kind-${KIND_CLUSTER_NAME}"' >> ~/.zshrc
echo 'alias k8s-eks="kubectl config use-context <用户名>@my-k8s-tokyo.ap-northeast-1.eksctl.io"' >> ~/.zshrc
source ~/.zshrc
```

---

### 5. 体验 LoadBalancer Service

在 EKS 中，`type: LoadBalancer` 会真正创建一个 **AWS ALB（应用负载均衡器）**：

```bash
# 先确保有运行中的 Pod
kubectl get pods

# 创建 LoadBalancer Service
kubectl apply -f - <<EOF
apiVersion: v1
kind: Service
metadata:
  name: web-lb
  annotations:
    service.beta.kubernetes.io/aws-load-balancer-type: external
    service.beta.kubernetes.io/aws-load-balancer-scheme: internet-facing
    service.beta.kubernetes.io/aws-load-balancer-nlb-target-type: ip
spec:
  selector:
    app: web-demo
  ports:
    - port: 80
      targetPort: 80
  type: LoadBalancer
EOF

# 等待分配 DNS 名称（通常 1-2 分钟）
kubectl get svc web-lb -w
# NAME     TYPE           CLUSTER-IP     EXTERNAL-IP                                                             PORT(S)
# web-lb   LoadBalancer   10.100.20.10   xxxxx-xxxxx.ap-northeast-1.elb.amazonaws.com                            80:30123/TCP

# 用浏览器或 curl 访问
curl http://xxxxx-xxxxx.ap-northeast-1.elb.amazonaws.com

# 验证负载均衡效果（连续请求，看到两个 Pod 的 IP 轮流响应）
for i in {1..10}; do
  kubectl get pods -o wide | grep web-demo
  curl -s http://xxxxx-xxxxx.ap-northeast-1.elb.amazonaws.com | head -1
done
```

> 💡 **对比 Kind**：在 Kind 中 NodePort 只能通过 `port-forward` 访问，而 EKS 上的 LoadBalancer 会创建真正的 AWS 负载均衡器，让外部直接访问你的应用。

---

### 6. 故障演练：模拟节点故障

#### 场景一：手动停止一台 EC2

不要通过 kubectl 删除节点，而是去 AWS 控制台或 CLI 停止一个 EC2 实例：

```bash
# 找到节点对应的 EC2 实例 ID
NODE_NAME=$(kubectl get nodes -o json | jq -r '.items[0].metadata.name')
echo "K8s 节点名: $NODE_NAME"

# 从节点名找到 AWS EC2 实例 ID（格式：i-xxxx）
INSTANCE_ID=$(aws ec2 describe-instances \
  --region ap-northeast-1 \
  --filters "Name=private-dns-name,Values=${NODE_NAME}" \
  --query "Reservations[0].Instances[0].InstanceId" \
  --output text)
echo "EC2 实例 ID: $INSTANCE_ID"

# 在另一个终端观察
kubectl get nodes -w

# 停止该 EC2 实例（模拟节点故障）
aws ec2 stop-instances \
  --region ap-northeast-1 \
  --instance-ids $INSTANCE_ID

# 观察：
# 1. kubectl 中节点状态变为 NotReady
# 2. 该节点上的 Pod 先变成 Unknown，然后被重新调度到另一个节点
# 3. 如果是托管节点组，Auto Scaling Group 会自动重建一台新的 EC2 🎉

kubectl get pods -o wide -w
```

#### 场景二：观察 Pod 重新调度

```bash
# 在一个窗口观察 Pod 状态
kubectl get pods -o wide -w

# 在另一个窗口删除一个节点上的所有 Pod 模拟
kubectl delete pod --all --field-selector spec.nodeName=<某个节点名>
# 看 Deployment 的控制器自动重建 ✅
```

---

### 7. 清理（非常重要！）

```bash
# 方式一：一键删除整个集群（推荐）
eksctl delete cluster --name my-k8s-tokyo --region ap-northeast-1

# 方式二：如果使用 YAML 配置文件创建
eksctl delete cluster -f eks-cluster.yaml

# 验证清理完成
kubectl config get-contexts          # 应该看到 EKS 上下文已消失
aws ec2 describe-instances \
  --region ap-northeast-1 \
  --filters "Name=tag:eks:cluster-name,Values=my-k8s-tokyo"  # 应该返回空
```

> ⚠️ **eksctl delete cluster 会清理哪些资源：**
> - ✅ EKS 控制平面
> - ✅ 所有 EC2 节点
> - ✅ Auto Scaling Group
> - ✅ VPC + 子网 + 安全组
> - ✅ NAT Gateway + 弹性 IP
> - ✅ ALB / NLB（ELB）
> - ✅ IAM 角色和策略
> - ✅ CloudFormation 堆栈

---

### 8. 💳 东京区域 EKS 详细成本分析

#### AWS 付费计费项（东京 ap-northeast-1）

以下为 2025/2026 年参考价格，实际以 [AWS 定价页面](https://aws.amazon.com/eks/pricing/) 为准：

| 组件 | 计费方式 | 单价 | 2 小时体验 | 1 天（8h） | 1 个月（730h） |
| --- | --- | --- | --- | --- | --- |
| **EKS 控制平面** | 按小时 | $0.10/h | $0.20 | $0.80 | $73.00 |
| **EC2 t3.small×2** | 按小时 | ~$0.0252/h | $0.10 | $0.40 | $36.79 |
| **EC2 t3.medium×2** | 按小时 | ~$0.0504/h | $0.20 | $0.81 | $73.58 |
| **EBS gp3 30GB×2** | 按 GB/月 | $0.09/GB/月 | — | $0.03 | $5.40 |
| **ALB（可选）** | 按小时+LCU | ~$0.0252/h | $0.05 | $0.20 | ~$22 |
| **NAT Gateway×1** | 按小时 | ~$0.062/h | $0.12 | $0.50 | $45.26 |
| **数据传输** | 按 GB | 出站 $0.09/GB 起 | 忽略 | 忽略 | 视流量 |

#### 实战成本预估（短期学习）

| 场景 | EC2 规格 | 使用时长 | 预估总费用 |
| --- | --- | --- | --- |
| 🟢 最低体验 | t3.small×2，无 LB | 3 小时 | **~$0.80** |
| 🟡 完整教程 | t3.medium×2，含 LB | 3 小时 | **~$1.05** |
| 🔵 长期实验 | t3.small×2，无 LB | 一周（~20h 实际使用） | **~$5.30** |

> 💡 **省钱核心**：
> 1. **用完即删集群**：不删的话，即使所有 Pod 删光，EKS 控制平面和 NAT Gateway 也在计费
> 2. **不使用 LoadBalancer**：不创建 LoadBalancer 类型的 Service 就不会产生 ALB/NLB 费用
> 3. **t3.small 够学习用**：2 个 t3.small 总共 4GB 内存，跑几个 Nginx 或你的 Flask Demo 完全够用
> 4. **善用 `kubectl` 操作**：大部分学习和探索可以通过 kubectl 完成，不需要长时间开着集群

---

### ✅ Bonus 成果检查

- [ ] 在 ap-northeast-1（东京）创建了 2 节点的 EKS 集群
- [ ] 能通过 SSM Session Manager 连接到 EC2 节点查看日志
- [ ] 观察了 Pod 在 2 个节点上的调度分布
- [ ] 掌握了 `cordon / drain / uncordon` 节点维护流程
- [ ] 理解了 Taints & Tolerations 的用途（特殊节点限制调度）
- [ ] 尝试了 nodeAffinity 定向调度
- [ ] 在 EKS 上体验了 LoadBalancer Service（真正的 AWS ALB）
- [ ] 模拟了节点故障，观察了 Pod 重新调度
- [ ] 💰 理解了东京区域的成本结构并**清理了集群**

---

## 附录：K8s 命令速查表

### 日常操作

```bash
# ───────── 资源管理 ─────────
kubectl apply -f <file.yaml>          # 创建/更新资源
kubectl delete -f <file.yaml>         # 删除资源
kubectl delete pod <name>             # 删除 Pod
kubectl get <资源类型>                 # 查看资源列表

# ───────── 常用查看 ─────────
kubectl get pods                      # 查看所有 Pod
kubectl get pods -w                   # 持续观察变化
kubectl get pods -o wide              # 显示 IP 和节点
kubectl get pods --all-namespaces     # 查看所有命名空间
kubectl get pods -l app=myapp         # 按标签过滤
kubectl describe pod <name>           # 查看详情
kubectl get events                    # 查看事件

# ───────── 与 Pod 交互 ─────────
kubectl logs <pod>                    # 查看日志
kubectl logs -f <pod>                 # 持续跟踪
kubectl logs <pod> -c <容器名>        # 多容器 Pod 指定容器
kubectl exec -it <pod> -- sh          # 进入 Pod
kubectl port-forward pod/<pod> 8080:80    # 端口转发（Pod）
kubectl port-forward svc/<svc> 8080:80    # 端口转发（Service）

# ───────── Deployment 操作 ─────────
kubectl scale deploy/<name> --replicas=5     # 扩缩容
kubectl rollout status deploy/<name>         # 查看部署状态
kubectl rollout history deploy/<name>        # 查看历史版本
kubectl rollout undo deploy/<name>           # 回滚到上一版本
kubectl rollout undo deploy/<name> --to-revision=2  # 回滚到指定版本
kubectl set image deploy/<name> <容器>=<新镜像>  # 更新镜像（命令式）
```

### 资源类型速记

```bash
kubectl api-resources | head -20

# 常用缩写：
po    = Pod
deploy = Deployment
svc   = Service
cm    = ConfigMap
secret = Secret
pv    = PersistentVolume
pvc   = PersistentVolumeClaim
ns    = Namespace
ep    = Endpoints
no    = Node
rs    = ReplicaSet
```

### YAML 字段速查

```yaml
# 每一类资源的必要字段
# ──────── Pod ────────
apiVersion: v1
kind: Pod
metadata:
  name: <字符串>
  labels:
    <key>: <value>
spec:
  containers:
    - name: <字符串>
      image: <镜像名:标签>

# ──────── Deployment ────────
apiVersion: apps/v1          # 注意！不是 v1
kind: Deployment
spec:
  replicas: <数字>
  selector:
    matchLabels:
      <key>: <value>
  template:
    metadata:
      labels:
        <key>: <value>      # 必须跟 selector 一致
    spec:
      containers: [...]

# ──────── Service ────────
apiVersion: v1
kind: Service
spec:
  selector:
    <key>: <value>          # 通过标签选 Pod
  ports:
    - port: <数字>            # Service 端口
      targetPort: <数字>      # Pod 端口
  type: ClusterIP | NodePort | LoadBalancer

# ──────── ConfigMap / Secret ────────
apiVersion: v1
kind: ConfigMap              # 或 Secret
data:
  KEY: value                 # Secret 需要用 base64
```

### Kind 集群管理

```bash
# 创建/删除
kind create cluster
kind create cluster --name my-cluster
kind delete cluster

# 加载本地镜像到 Kind
kind load docker-image my-app:latest
kind load docker-image my-app:latest --name my-cluster

# 查看集群节点容器
docker ps --filter name=kind-
```

---

## 🎉 结语

走到这里，你已经：

- ✅ 理解了 K8s 的核心概念（Pod、Deployment、Service、ConfigMap、Secret、Volume）
- ✅ 能在本地用 Kind 搭建开发环境
- ✅ 会写 YAML 声明式管理资源
- ✅ 掌握了健康检查、滚动更新、回滚等运维操作
- ✅ 能部署一个完整的 Web 应用（Flask + Redis）
- ✅ 体验了 AWS EKS 的云端部署

**下一步可以探索的方向：**
- 📦 **Helm**：K8s 的包管理器（像 apt/brew 一样安装应用）
- 🏗️ **Kustomize**：声明式配置管理
- 🔍 **Prometheus + Grafana**：K8s 监控
- 🔐 **RBAC**：权限控制
- 🚀 **GitOps（ArgoCD）**：以 Git 为真理来源的持续部署

---

> **遇到问题先查：** `kubectl describe` + `kubectl logs` + `kubectl get events` 解决 80% 的问题。

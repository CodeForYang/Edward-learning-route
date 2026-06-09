# 🚀 本地体验 Kubernetes：用 kind 运行带 nodeSelector 的 Pod

> **目标**：在你自己电脑上跑一个迷你 K8s 集群，体验“调度器找 Node → 拉镜像 → 运行容器”的全过程。  
> **适合**：已有 Docker 基础，想理解 K8s 调度逻辑的你。  
> **耗时**：约 20 分钟（取决于网络）。

## 📦 准备工作

### 1. 确认 Docker 已运行
```bash
docker --version
docker ps
```
确保 Docker Desktop 状态栏图标静止（引擎已启动）。

### 2. 安装 kind（Kubernetes in Docker）

**macOS 用 Homebrew：**
```bash
brew install kind
```

**验证安装：**
```bash
kind --version
# 输出类似：kind version 0.20.0
```

> 💡 **kind 原理**：在你本机 Docker 里启动几个“容器节点”，每个容器模拟一台 K8s 的 Node（虚拟机）。这样不占太多资源，却能体验完整集群。

## 🎯 第一步：创建一个带不同标签的 2 节点集群

正常情况下你只需要 `kind create cluster`，但为了演示 `nodeSelector`，我们手动给节点打标签。

### 1.1 编写 kind 配置文件

新建文件 `kind-config.yaml`：

```yaml
kind: Cluster
apiVersion: kind.x-k8s.io/v1alpha4
nodes:
  - role: control-plane           # 控制平面节点（类似 Jenkins Master）
  - role: worker                  # 工作节点1（类似 Jenkins Agent）
    extraMounts:
      - hostPath: /var/run/docker.sock
        containerPath: /var/run/docker.sock
  - role: worker                  # 工作节点2
    labels:
      mydisk: ssd                 # 👈 给这个节点打上自定义标签
```

### 1.2 创建集群

```bash
kind create cluster --name demo --config kind-config.yaml
```

等待 2-3 分钟，看到 `Cluster creation complete` 即成功。

### 1.3 验证节点状态

```bash
kubectl get nodes
```

输出示例：
```
NAME                 STATUS   ROLES           AGE   VERSION
demo-control-plane   Ready    control-plane   2m   v1.27.3
demo-worker          Ready    <none>          2m   v1.27.3
demo-worker2         Ready    <none>          2m   v1.27.3
```

### 1.4 查看节点标签（关键！）

```bash
kubectl get nodes --show-labels
```

你会看到 `demo-worker2` 有 `mydisk=ssd` 标签，而 `demo-worker` 没有。  
这就是你的 **Jenkins Label** 在 K8s 里的实现。

## 🧪 第二步：运行一个普通 Pod（不加 nodeSelector）

先看默认调度行为——不加约束，调度器随机选一个 worker。

### 2.1 创建一个测试 Pod

新建 `pod-nginx.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: test-nginx
spec:
  containers:
  - name: nginx
    image: nginx:alpine
    ports:
    - containerPort: 80
```

### 2.2 部署到集群

```bash
kubectl apply -f pod-nginx.yaml
```

### 2.3 查看 Pod 被调度到了哪个 Node

```bash
kubectl get pod -o wide
```

输出类似：
```
NAME         READY   STATUS    RESTARTS   AGE   IP           NODE             
test-nginx   1/1     Running   0          10s   10.244.1.2   demo-worker      # 可能被调度到 worker1
```

> **结论**：没有 `nodeSelector` 时，调度器随机选一个健康节点（通常是负载最低的）。

### 2.4 清理这个 Pod

```bash
kubectl delete pod test-nginx
```

## 🎯 第三步：运行带 nodeSelector 的 Pod（找到带 `mydisk=ssd` 的 Node）

现在模拟你的需求：“必须运行在 **有 SSD 硬盘** 的机器上”。

### 3.1 编写带 nodeSelector 的 Pod

新建 `pod-selector.yaml`：

```yaml
apiVersion: v1
kind: Pod
metadata:
  name: ssd-pod
spec:
  nodeSelector:
    mydisk: ssd              # 👈 只调度到有 mydisk=ssd 标签的节点
  containers:
  - name: nginx
    image: nginx:alpine
```

### 3.2 部署

```bash
kubectl apply -f pod-selector.yaml
```

### 3.3 验证 Pod 是否跑在 `demo-worker2` 上

```bash
kubectl get pod -o wide
```

你应该看到：
```
NAME      READY   STATUS    RESTARTS   AGE   IP            NODE             
ssd-pod   1/1     Running   0          8s    10.244.2.2    demo-worker2     ✅ 带标签的节点
```

如果 `demo-worker2` 一直 NotReady（很少见），调度器会等待它恢复，Pod 会卡在 `Pending`。

### 3.4 查看调度事件（理解“找机器”过程）

```bash
kubectl describe pod ssd-pod
```

注意看 **Events** 部分：
```
Events:
  Type    Reason     Age   From               Message
  ----    ------     ----  ----               -------
  Normal  Scheduled  10s   default-scheduler  Successfully assigned default/ssd-pod to demo-worker2
```
这里 `default-scheduler` 就是 **K8s 控制平面的调度器**，它找到了匹配 `mydisk=ssd` 的节点。

## 🧹 第四步：清理实验环境

```bash
# 删除 Pod
kubectl delete -f pod-selector.yaml

# 删除整个 kind 集群（释放 Docker 资源）
kind delete cluster --name demo
```

## 📖 概念回顾（对照 Jenkins）

| Jenkins                       | Kubernetes (本实验)                    |
|-------------------------------|----------------------------------------|
| 给 Agent 打 Label（如 `ssd`） | 给 Node 打标签 `kubectl label nodes`   |
| `agent { label 'ssd' }`       | `nodeSelector: { mydisk: ssd }`        |
| Master 找到匹配的 Agent       | Scheduler 找到带 `mydisk=ssd` 的 Node  |
| Agent 拉取代码 + 执行流水线   | Node 上的 kubelet 拉镜像 + 运行容器    |

## 🔧 扩展练习（可选）

### 1. 动态给现有节点打标签
```bash
# 给 demo-worker 也加上 ssd 标签
kubectl label nodes demo-worker mydisk=ssd

# 再创建一个无 nodeSelector 的 Pod，它可能被调度到任意 ssd 节点
```

### 2. 模拟“找不到匹配节点”
```bash
# 修改 nodeSelector 为一个不存在的标签
# nodeSelector:
#   disk: ultra-ssd   # 没有节点有这个标签

kubectl apply -f pod-selector.yaml
kubectl get pod   # 会一直处于 Pending
kubectl describe pod ssd-pod | grep -A 10 Events
# 你会看到：0/3 nodes are available: 3 node(s) didn't match Pod's node affinity/selector.
```

### 3. 使用更优雅的 nodeAffinity（进阶）
```yaml
spec:
  affinity:
    nodeAffinity:
      requiredDuringSchedulingIgnoredDuringExecution:
        nodeSelectorTerms:
        - matchExpressions:
          - key: mydisk
            operator: In
            values:
            - ssd
```
效果同 `nodeSelector`，但支持 `NotIn`、`Exists` 等复杂逻辑。

## ❓ 常见问题

**Q: 为什么 kind 创建节点时需要挂载 `/var/run/docker.sock`？**  
A: kind 内部使用 Docker 来运行容器，挂载 socket 让节点内的 kubelet 能调用宿主机的 Docker 创建 Pod 里的容器。

**Q: 生产环境也用 kind 吗？**  
A: 不，kind 只用于本地开发/测试。生产用托管 K8s（如 AWS EKS、Google GKE）或自建集群（kubeadm）。

**Q: 怎么看到“拉取代码”（git clone）这个过程？**  
A: K8s 原生不拉代码，它拉的是镜像（`docker pull`）。如果你想模拟 Jenkins 那种从 git 拉代码编译，可以把编译命令写在 Pod 的 `command` 里（如 `git clone && make`），但更好的实践是提前构建好镜像。

## 🎉 总结

你已经用 **kind** 在本地完整模拟了：
1. 创建带标签的 K8s 节点（≈ Jenkins Agent Label）。
2. 编写 Pod 并指定 `nodeSelector`（≈ Jenkins agent label 约束）。
3. 观察调度器如何找到匹配节点并运行容器。
4. 清理实验环境。

现在你应该能直观理解：**Node 就是一台机器，nodeSelector 就是 Label 匹配规则，调度器就是那个负责任务分配的 Master**。

下一步可尝试：用 `kubectl scale` 扩容 Pod 数量，看看调度器如何将副本分散到不同 Node（如果你有两个以上 worker）。
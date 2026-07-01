# Crossplane 入门指南 —— K8s 原生的 IaC 方案

> **前置知识**：本文假设你已完成 [Terraform 入门指南](terraform-入门指南.md) 的 **Day 7（EKS）**，并熟悉 Kubernetes 基础概念（Pod、Deployment、CRD）。如果你对 K8s 还不熟，建议先学完 K8s 基础再回来。
>
> **一句话**：在 Kubernetes 里用 `kubectl apply` 创建云资源（RDS、S3、VPC），云资源像 Pod 一样被 K8s 管理。

---

## 1. 它解决了什么问题？

你的团队已经全面拥抱 Kubernetes，但云资源还得用 Terraform 管：

```
# 当前的两套工具、两个流程

# 流程 A：K8s 资源
kubectl apply -f deployment.yaml    # 部署应用
kubectl apply -f service.yaml       # 暴露服务

# 流程 B：云资源
terraform apply -auto-approve        # 创建 RDS 数据库

# 痛点：
# 1. 两套工具、两套认证、两套 CI/CD 流程
# 2. 开发和运维各会一套，沟通成本高
# 3. GitOps（如 ArgoCD）只能管 K8s 资源，管不了云资源
```

**Crossplane 的解法**：把 AWS / GCP / Azure 的云资源变成 **Kubernetes 的自定义资源**。

```bash
# 统一用 K8s 的方式：kubectl apply 解决一切

kubectl apply -f deployment.yaml    # 创建应用
kubectl apply -f rds.yaml           # 创建数据库
kubectl apply -f bucket.yaml        # 创建 S3 桶

# 没有任何区别！
# 所有资源都被 ArgoCD 跟踪、在 K8s API 里可见
```

---

## 2. 架构理解

```
你的 K8s 集群
┌──────────────────────────────────────────────────────┐
│                                                       │
│  你执行 kubectl apply -f rds.yaml                     │
│          │                                            │
│          ▼                                            │
│  ┌────────────────────┐                              │
│  │ RDSInstance (CR)   │  ◄── K8s 里的"自定义资源"      │
│  │ (my-database)      │     像 Deployment、Pod 一样    │
│  └────────┬───────────┘                              │
│           │                                           │
│           ▼                                           │
│  ┌────────────────────┐                              │
│  │  Crossplane        │  ◄── K8s 里的"控制器"          │
│  │  Provider (AWS)    │     像 Ingress Controller 一样 │
│  └────────┬───────────┘                              │
│           │                                           │
└───────────┼───────────────────────────────────────────┘
            │ Crossplane 调用 AWS API
            ▼
┌──────────────────────────┐
│  AWS 上的真实 RDS 实例    │  ◄── AWS 控制台里也能看到
│  (db-xxxxxxxxx)          │
└──────────────────────────┘
```

**关键区别**：

| | Terraform | Crossplane |
|--|-----------|------------|
| 谁执行 | 你在本机或 CI 运行 | K8s 里的控制器自动运行 |
| 状态存哪 | S3 / Consul 的 state 文件 | 存在 K8s 的 CR（自定义资源）状态里 |
| 触发时机 | 你执行 `terraform apply` | CR 被创建后，自动触发 |
| 持续同步 | 不，再 plan 才知道 | 是，Crossplane 持续监听并纠正漂移 |

---

## 3. 安装 Crossplane

> **前提条件**：你有一个运行中的 K8s 集群（minikube / EKS / 任何集群都可以）。

```bash
# ===== 第一步：用 Helm 安装 Crossplane =====

# 添加 Helm 仓库
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update

# 安装到 crossplane-system 命名空间
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace

# 查看安装状态
kubectl get pods -n crossplane-system
# 应该看到 crossplane-xxx 和 provider-xxx 的 Pod 在运行

# ===== 第二步：安装 AWS Provider =====
# Crossplane 本身不知道 AWS，需要装 Provider 来扩展能力

cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-aws:v1.12.0
EOF

# 查看 Provider 状态（等变成 HEALTHY）
kubectl get provider

# ===== 第三步：配置 AWS 凭证 =====
# Crossplane 需要 AWS 密钥才能调 AWS API

# 创建本地凭证文件
cat <<EOF > aws-creds.conf
[default]
aws_access_key_id = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY
EOF

# 把凭证存为 K8s Secret
kubectl create secret generic aws-creds \
  -n crossplane-system \
  --from-file=credentials=./aws-creds.conf

# 告诉 Crossplane 使用这个 Secret
cat <<EOF | kubectl apply -f -
apiVersion: aws.upbound.io/v1beta1
kind: ProviderConfig
metadata:
  name: default
spec:
  credentials:
    source: Secret
    secretRef:
      namespace: crossplane-system
      name: aws-creds
      key: credentials
EOF
```

---

## 4. 第一个例子：用 Crossplane 创建 S3 存储桶

```yaml
# s3-bucket.yaml
# 完全用 K8s 的方式声明一个 S3 存储桶

apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: my-crossplane-bucket-demo

  # 标签 —— 和在 K8s 里打标签一样
  labels:
    environment: demo
    managed-by: crossplane
spec:
  # --- S3 桶的配置 ---
  forProvider:
    # 私有桶
    acl: "private"

    # 开启版本控制
    versioningConfiguration:
      - status: Enabled

    # 服务端加密
    serverSideEncryptionConfiguration:
      - rule:
          applyServerSideEncryptionByDefault:
            sseAlgorithm: AES256

    # 标签（会同步到 AWS）
    tagging:
      tagSet:
        - key: Environment
          value: demo
        - key: ManagedBy
          value: crossplane

  # 引用我们之前配置的 AWS Provider
  providerConfigRef:
    name: default
```

部署并查看：

```bash
# 创建 S3 桶
kubectl apply -f s3-bucket.yaml

# 查看所有 Bucket 资源（等效于 terraform state list）
kubectl get bucket

# 输出：
# NAME                           READY   SYNCED   AGE
# my-crossplane-bucket-demo      True    True     30s

# READY  = True → AWS 资源已创建成功
# SYNCED = True → Crossplane 持续同步，没有漂移

# 查看详细信息（包含 AWS 返回的状态）
kubectl describe bucket my-crossplane-bucket-demo

# 删除 S3 桶（等效于 terraform destroy）
kubectl delete bucket my-crossplane-bucket-demo
```

---

## 5. 第二个例子：创建 VPC + EC2（完整基础设施）

```yaml
# infra.yaml
# 用 K8s 的方式创建 VPC、子网、安全组、EC2

---
# ==== 1. VPC ====
apiVersion: ec2.aws.upbound.io/v1beta1
kind: VPC
metadata:
  name: demo-vpc
spec:
  forProvider:
    cidrBlock: "10.0.0.0/16"
    enableDnsSupport: true
    enableDnsHostnames: true
    tags:
      Name: demo-vpc
  providerConfigRef:
    name: default
---
# ==== 2. 公有子网 ====
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Subnet
metadata:
  name: demo-public-subnet
spec:
  forProvider:
    # 引用 VPC —— 写被引用资源的 name
    vpcIdRef:
      name: demo-vpc
    cidrBlock: "10.0.1.0/24"
    availabilityZone: "ap-northeast-1a"
    mapPublicIpOnLaunch: true
  providerConfigRef:
    name: default
---
# ==== 3. 安全组 ====
apiVersion: ec2.aws.upbound.io/v1beta1
kind: SecurityGroup
metadata:
  name: demo-web-sg
spec:
  forProvider:
    vpcIdRef:
      name: demo-vpc
	      groupDescription: "Allow HTTP and SSH"
	      # ⚠️ 生产环境不应将 SSH（22）开放到全互联网
	      # 应限制来源 IP 或使用 AWS SSM 代替 SSH
	      ingress:
	      - fromPort: 22
        toPort: 22
        protocol: tcp
        cidrBlocks:
          - "0.0.0.0/0"
      - fromPort: 80
        toPort: 80
        protocol: tcp
        cidrBlocks:
          - "0.0.0.0/0"
    egress:
      - fromPort: 0
        toPort: 0
        protocol: "-1"
        cidrBlocks:
          - "0.0.0.0/0"
  providerConfigRef:
    name: default
---
# ==== 4. EC2 实例 ====
apiVersion: ec2.aws.upbound.io/v1beta1
kind: Instance
metadata:
  name: demo-web-server
spec:
  forProvider:
    # ⚠️ 这个 AMI ID 是 us-east-1 区域的，切换到其他区域需替换
    # 推荐用 SSM 参数动态获取：aws ssm get-parameters --names /aws/service/ami-amazon-linux-latest/al2023-ami-kernel-default-x86_64
    ami: "ami-0c55b159cbfafe1f0"
    instanceType: "t3.micro"
    subnetIdRef:
      name: demo-public-subnet
    securityGroupRefs:
      - name: demo-web-sg
    associatePublicIpAddress: true
    tags:
      Name: demo-web-server
    # 启动脚本
    userData: |
      #!/bin/bash
      yum install -y httpd   # ⚠️ 如果用 Amazon Linux 2023 需替换为 dnf
      systemctl start httpd
      echo "Hello from Crossplane!" > /var/www/html/index.html
  providerConfigRef:
    name: default
```

部署：

```bash
# 一次性创建所有资源
kubectl apply -f infra.yaml

# Crossplane 会自动解析依赖关系：
# 1. 先创建 VPC（因为被 Subnet 和 SecurityGroup 引用）
# 2. 然后创建 Subnet 和 SecurityGroup
# 3. 最后创建 EC2 Instance

# 查看所有资源状态
kubectl get vpc,subnet,securitygroup,instance

# 查看某个资源详情（如果卡住了）
kubectl describe instance demo-web-server
```

---

## 6. 核心概念：Composition（资源组合）

这是 Crossplane **最强的能力**——把一组云资源打包成一个"产品"。

### 6.1 为什么要 Composition？

```
没有 Composition 的时候：
用户需要了解 VPC、Subnet、SecurityGroup、RDS 的所有细节
写 50 行 YAML 才能创建一个数据库

用了 Composition 之后：
平台团队定义好"数据库产品"
用户只需要写 3 行 YAML
```

### 6.2 三步实现 Composition

#### 第 1 步：定义"产品"的 API（XRD）

```yaml
# xrd-database.yaml
# 告诉 K8s：以后有一个叫 XDatabase 的自定义资源

apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.database.example.org
spec:
  group: database.example.org          # API 分组
  names:
    kind: XDatabase                    # 资源名字（类比 Pod、Deployment）
    plural: xdatabases
  versions:
    - name: v1alpha1
      schema:
        openAPIV3Schema:
          type: object
          properties:
            spec:
              type: object
              properties:
                # 用户只需要传这三个参数
                storageGB:
                  type: integer
                  default: 20          # 默认 20GB
                engine:
                  type: string
                  default: "postgres"
                engineVersion:
                  type: string
                  default: "13"
```

#### 第 2 步：定义"产品"由哪些资源组成（Composition）

```yaml
# composition-database.yaml
# 告诉 Crossplane：一个 XDatabase 由 安全组 + RDS 实例 组成

apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xdatabases.aws.database.example.org
spec:
  compositeTypeRef:
    apiVersion: database.example.org/v1alpha1
    kind: XDatabase

  # 这个产品包含的资源列表
  resources:
    # 子资源 1：安全组
    - name: database-security-group
      base:
        apiVersion: ec2.aws.upbound.io/v1beta1
        kind: SecurityGroup
        spec:
          forProvider:
            groupDescription: "Security group for the database"
            ingress:
              - fromPort: 5432
                toPort: 5432
                protocol: tcp
                cidrBlocks:
                  - "10.0.0.0/8"

    # 子资源 2：RDS 实例
    - name: rds-instance
      base:
        apiVersion: rds.aws.upbound.io/v1beta1
        kind: Instance
        spec:
          forProvider:
            dbInstanceClass: db.t3.small
            allocatedStorage: 20
            engine: postgres
            engineVersion: "13"
            skipFinalSnapshotBeforeDeletion: true

      # 把用户的参数"注入"到这个资源里
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.storageGB"       # 用户填的 storageGB
          toFieldPath: "spec.forProvider.allocatedStorage"  # → 注入到这个字段
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.engine"
          toFieldPath: "spec.forProvider.engine"
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.engineVersion"
          toFieldPath: "spec.forProvider.engineVersion"
```

#### 第 3 步：用户使用这个"产品"

```bash
# 应用 XRD 和 Composition（这是平台团队做的事）
kubectl apply -f xrd-database.yaml
kubectl apply -f composition-database.yaml

# 等待 Crossplane 注册新资源类型
kubectl get crd | grep xdatabase
```

```yaml
# 用户创建数据库
# 只需要 3 个关键参数！

apiVersion: database.example.org/v1alpha1
kind: XDatabase
metadata:
  name: my-app-db
spec:
  storageGB: 50
  engine: postgres
  engineVersion: "14"
```

```bash
# 用户部署
kubectl apply -f my-database.yaml

# 查看"产品"状态
kubectl get xdatabase

# Crossplane 自动在后台创建了：
# - 一个安全组（5432 端口）
# - 一个 RDS PostgreSQL 实例（50GB）
```

### 6.3 Composition 的意义

```
没有 Composition：                      有 Composition：

用户（面向复杂资源）                     用户（面向产品）
├── kubectl apply Subnet               ├── kubectl apply XDatabase
├── kubectl apply SecurityGroup         └── 写 3 行 YAML
├── kubectl apply RDSInstance                         
└── 要懂所有参数的细节                      平台团队（封装复杂性）
                                          ├── 定义 API（XRD）
                                          └── 定义实现（Composition）
```

**Composition 的资源流转关系（加深理解）：**

```
用户 (kubectl apply)
        |
        v
+-- Claim (XDatabase)  ----------------------+
|   spec: { storageGB: 50 }                  |  ← 用户创建的自定义资源
+-----------------------+--------------------+
                        | Crossplane 控制器自动转换
                        v
+-- CompositeResource  ----------------------+
|   spec: { storageGB: 50 }                  |  ← 平台定义的资源模板实例
+-----------------------+--------------------+
                        | 按 Composition 定义展开
                        v
+-- Managed Resource 1 --+  +-- Managed Resource 2 --+
| SecurityGroup          |  | RDSInstance             |  ← 真实云资源
| forProvider: ...       |  | forProvider: ...        |
+------------------------+  +-------------------------+
                        |                         |
                        v                         v
+-- AWS SecurityGroup --+  +-- AWS RDS Instance --+
|                        |  |                      |  ← 真实的 AWS 资源
+------------------------+  +----------------------+
```

---

## 7. Crossplane vs Terraform

| 对比维度 | Terraform | Crossplane |
|---------|-----------|------------|
| **执行环境** | 本地 / CI 管道 | K8s 集群内部 |
| **声明方式** | HCL 文件 | K8s CRD（YAML） |
| **状态管理** | State 文件（S3/Consul） | 存在 K8s 的 CR 状态里 |
| **触发方式** | 手动执行 `terraform apply` | CR 被创建后自动触发 |
| **持续同步** | 不自动（需再跑 plan） | 自动（Continuously Reconciling） |
| **GitOps 集成** | 需要额外工具 | 天然支持（ArgoCD 直接管 CR） |
| **适用场景** | 个人 / CI / 任何环境 | 已用 K8s 的团队 |
| **依赖管理** | depends_on | K8s 原生的资源引用（ref） |
| **学习曲线** | 需学 HCL | 需懂 K8s 概念 |

---

## 8. 快速参考

```bash
# 基础操作（和普通 K8s 资源完全一样）
kubectl get bucket                     # 列出 S3 桶
kubectl get vpc                        # 列出 VPC
kubectl get instance                   # 列出 EC2
kubectl get xdatabase                  # 列出自定义产品

kubectl describe bucket my-bucket      # 查看详情
kubectl delete bucket my-bucket        # 删除（销毁 AWS 资源）
kubectl apply -f infra.yaml            # 创建 / 更新

# Crossplane 特有
kubectl get provider                   # 查看已安装的 Provider
kubectl get providerconfig             # 查看 Provider 配置
kubectl get composition                # 查看已定义的 Composition
kubectl get compositeresourcedefinition # 查看已定义的 XRD

# 查看 Crossplane 自身运行状态
kubectl get pods -n crossplane-system
kubectl logs -n crossplane-system deployment/crossplane
```

---

## 9. 什么时候学 Crossplane？

**推荐用 Crossplane 的场景**：
- 你的团队已经在使用 Kubernetes
- 你已经在用 ArgoCD / Flux 做 GitOps（Crossplane + ArgoCD 是黄金组合：ArgoCD watch 你仓库中的 YAML → 自动 apply 到 K8s → Crossplane 控制器负责创建/更新/删除云资源，实现了云资源层面的 GitOps）
- 你想统一管理"容器资源"和"云资源"
- 你有平台工程团队，想向开发者提供"数据库即服务"的体验

**不推荐 Crossplane 的场景**：
- 没有 K8s 集群（为了用 Crossplane 专门搭集群，成本太高）
- 团队不熟悉 K8s（学习曲线太陡）
- 只是个人学习 IaC（Terraform 更直接）

**Crossplane 的学习前提**：
```
┌─────────────────────────────┐
│  先掌握这些 K8s 基础概念     │
├─────────────────────────────┤
│ ✅ Pod / Deployment / Service│
│ ✅ kubectl 基本操作          │
│ ✅ K8s CRD 和 Controller    │
│ ✅ Helm 或 Kustomize        │
│ ✅ GitOps（ArgoCD / Flux）  │
└─────────────────────────────┘
```

---

> **一句话总结**：Crossplane 适合"已经在 K8s 里了，想把云资源也拉进来一起管"的团队。不是 Terraform 的替代品，而是不同场景的工具。

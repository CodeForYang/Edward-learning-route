# Crossplane 入门指南 —— K8s 原生的 IaC 方案

> **前置知识**：本文假设你已完成 [Terraform 入门指南](terraform-入门指南.md) 的 **Day 7（EKS）**，并熟悉 Kubernetes 基础概念（Pod、Deployment、CRD）。如果你对 K8s 还不熟，建议先学完 K8s 基础再回来。
>
> **一句话**：在 Kubernetes 里用 `kubectl apply` 创建云资源（RDS、S3、VPC），云资源像 Pod 一样被 K8s 管理。

---

## 1️⃣ 搭建学习阶梯

> 将 Crossplane 技能拆解为 5 个递进等级，每个等级标注了**掌握内容**、**常见错误**和**进阶标准**。按顺序学习，不要跳级。

---

### Lv 1 —— 认知层：Crossplane 是什么？

**平时要做什么**
- 理解「两套工具、两个流程」的痛点（K8s 资源用 kubectl、云资源用 Terraform）
- 掌握 Crossplane 的核心理念：把云资源变成 K8s 自定义资源（CRD）
- 理解架构：CR（自定义资源）→ Provider（控制器）→ 真实云 API
- 区分 Crossplane 和 Terraform 的定位差异

**Crossplane 解决了什么问题**

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

**核心架构**

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

**常见错误**
- ❌ 以为 Crossplane 是 Terraform 的替代品 —— 它们是不同场景的工具，Crossplane 适合已用 K8s 的团队
- ❌ 没有 K8s 基础就想学 Crossplane —— 必须先理解 Pod、CRD、Controller 等 K8s 概念
- ❌ 忽略持续同步特性 —— Crossplane 会自动纠正漂移，Terraform 不会
- ❌ 把 Crossplane 当"云资源管理工具"而不是"K8s 扩展"来理解

**进阶标准**：你能画出一张图，说明 CR → Provider → AWS API 的调用链路。

---

### Lv 2 —— 操作层：安装并创建第一个云资源

**平时要做什么**
- 用 Helm 安装 Crossplane 到 K8s 集群
- 安装 AWS Provider 并配置凭证
- 用 `kubectl apply` 创建 S3 存储桶
- 用 `kubectl get/describe/delete` 管理云资源

**安装 Crossplane**

```bash
# ===== 第一步：用 Helm 安装 Crossplane =====
helm repo add crossplane-stable https://charts.crossplane.io/stable
helm repo update
helm install crossplane crossplane-stable/crossplane \
  --namespace crossplane-system \
  --create-namespace

# 查看安装状态
kubectl get pods -n crossplane-system

# ===== 第二步：安装 AWS Provider =====
cat <<EOF | kubectl apply -f -
apiVersion: pkg.crossplane.io/v1
kind: Provider
metadata:
  name: provider-aws
spec:
  package: xpkg.upbound.io/crossplane-contrib/provider-aws:v1.12.0
EOF

kubectl get provider  # 等变成 HEALTHY

# ===== 第三步：配置 AWS 凭证 =====
cat <<EOF > aws-creds.conf
[default]
aws_access_key_id = YOUR_ACCESS_KEY
aws_secret_access_key = YOUR_SECRET_KEY
EOF

kubectl create secret generic aws-creds \
  -n crossplane-system \
  --from-file=credentials=./aws-creds.conf

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

**第一个例子：创建 S3 存储桶**

```yaml
# s3-bucket.yaml
apiVersion: s3.aws.upbound.io/v1beta1
kind: Bucket
metadata:
  name: my-crossplane-bucket-demo
  labels:
    environment: demo
    managed-by: crossplane
spec:
  forProvider:
    acl: "private"
    versioningConfiguration:
      - status: Enabled
    serverSideEncryptionConfiguration:
      - rule:
          applyServerSideEncryptionByDefault:
            sseAlgorithm: AES256
    tagging:
      tagSet:
        - key: Environment
          value: demo
  providerConfigRef:
    name: default
```

```bash
# 部署
kubectl apply -f s3-bucket.yaml

# 查看状态
kubectl get bucket
# NAME                           READY   SYNCED   AGE
# my-crossplane-bucket-demo      True    True     30s
# READY  = True → AWS 资源已创建成功
# SYNCED = True → Crossplane 持续同步，没有漂移

kubectl describe bucket my-crossplane-bucket-demo

# 删除（等价于 terraform destroy）
kubectl delete bucket my-crossplane-bucket-demo
```

**常见错误**
- ❌ `READY=False` 不 debug 就重试 —— 用 `kubectl describe bucket xxx` 看具体错误信息
- ❌ AWS 凭证配置不对 —— 检查 Secret 内容和 ProviderConfig 引用
- ❌ 把资源删了就以为全删了 —— 检查 `kubectl get bucket` 是否真的空了
- ❌ 不理解 `READY` 和 `SYNCED` 的区别 —— READY 是资源存在，SYNCED 是状态一致

**进阶标准**：你能在 5 分钟内从一个空集群开始，完成 Crossplane 安装 + 创建 S3 存储桶 + 验证 + 删除的完整流程。

---

### Lv 3 —— 组合层：用 Composition 封装云产品

**平时要做什么**
- 理解 Composition 的设计思想：把一组云资源打包成"产品"
- 掌握 XRD（CompositeResourceDefinition）的定义方式
- 学会编写 Composition 并用 `patches` 传递用户参数
- 知道 Claim → CompositeResource → Managed Resource 的层层展开关系

**为什么需要 Composition？**

```
没有 Composition 的时候：
用户需要了解 VPC、Subnet、SecurityGroup、RDS 的所有细节
写 50 行 YAML 才能创建一个数据库

用了 Composition 之后：
平台团队定义好"数据库产品"
用户只需要写 3 行 YAML
```

**第 1 步：定义"产品"的 API（XRD）**

```yaml
# xrd-database.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: CompositeResourceDefinition
metadata:
  name: xdatabases.database.example.org
spec:
  group: database.example.org
  names:
    kind: XDatabase
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
                storageGB:
                  type: integer
                  default: 20
                engine:
                  type: string
                  default: "postgres"
                engineVersion:
                  type: string
                  default: "13"
```

**第 2 步：定义"产品"由哪些资源组成（Composition）**

```yaml
# composition-database.yaml
apiVersion: apiextensions.crossplane.io/v1
kind: Composition
metadata:
  name: xdatabases.aws.database.example.org
spec:
  compositeTypeRef:
    apiVersion: database.example.org/v1alpha1
    kind: XDatabase

  resources:
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
      patches:
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.storageGB"
          toFieldPath: "spec.forProvider.allocatedStorage"
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.engine"
          toFieldPath: "spec.forProvider.engine"
        - type: FromCompositeFieldPath
          fromFieldPath: "spec.engineVersion"
          toFieldPath: "spec.forProvider.engineVersion"
```

**第 3 步：用户使用**

```yaml
# 用户只需要 3 个关键参数
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
# 部署
kubectl apply -f my-database.yaml
kubectl get xdatabase  # 查看"产品"状态
```

**Composition 资源流转关系**

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
+------------------------+  +-------------------------+
                        |                         |
                        v                         v
+-- AWS SecurityGroup --+  +-- AWS RDS Instance --+
|                        |  |                      |  ← 真实的 AWS 资源
+------------------------+  +----------------------+
```

**常见错误**
- ❌ XRD 和 Composition 一次性写完再 deploy —— 先创建 XRD，等 CRD 注册成功后再创建 Composition
- ❌ `patches` 的 `fromFieldPath` 写错 —— Crossplane 不会警告，只会默默不注入
- ❌ 一次给用户太多参数选择 —— Composition 的初衷就是简化，保留 3-5 个参数就够了
- ❌ 忽略 `base` 字段的默认值 —— base 里写的值就是默认值，被 patches 覆盖

**进阶标准**：你能为团队的"标准 Web 服务"定义一个 Composition：包含 VPC + 子网 + 安全组 + EC2，用户只需要传环境名和实例规格。

---

### Lv 4 —— 对比层：Crossplane vs Terraform

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

**常见错误**
- ❌ 在 Crossplane 里强行用 Composition 封装一切 —— 简单的单资源直接用 YAML 定义即可
- ❌ 以为 Crossplane 能完全替代 Terraform —— 有些 Terraform provider 的成熟度远超 Crossplane
- ❌ 忽略 Crossplane 持续同步的副作用 —— 手动修改 AWS 资源会被 Crossplane 自动纠正回 CR 定义的状态

**进阶标准**：你能根据团队情况（是否有 K8s、是否用 GitOps、团队技术栈），判断该用 Crossplane 还是 Terraform。

---

### Lv 5 —— 生产层：GitOps 集成与生产考量

**平时要做什么**
- 理解 Crossplane + ArgoCD 的黄金组合
- 知道 Crossplane 的学习前提（K8s 基础概念列表）
- 了解什么场景推荐、什么场景不推荐 Crossplane

**Crossplane + ArgoCD 的黄金组合**

```
ArgoCD watch 你仓库中的 YAML → 自动 apply 到 K8s → Crossplane 控制器负责创建/更新/删除云资源

这样就实现了"云资源层面的 GitOps"：你的 S3、RDS、VPC 全部在 Git 仓库里、全部被 ArgoCD 跟踪。
```

**什么时候学 Crossplane？**

**推荐用 Crossplane**：
- 你的团队已经在使用 Kubernetes
- 你已经在用 ArgoCD / Flux 做 GitOps
- 你想统一管理"容器资源"和"云资源"
- 你有平台工程团队，想向开发者提供"数据库即服务"的体验

**不推荐 Crossplane**：
- 没有 K8s 集群（为了用 Crossplane 专门搭集群，成本太高）
- 团队不熟悉 K8s（学习曲线太陡）
- 只是个人学习 IaC（Terraform 更直接）

**Crossplane 的学习前提**

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

**常见错误**
- ❌ 团队没 K8s 硬上 Crossplane —— 额外维护 K8s 集群的成本可能超过收益
- ❌ 用 Crossplane 管理"非云资源"（如 DNS 记录、SaaS 配置）—— 不是所有资源都适合用 CRD 封装
- ❌ 不设置资源回收策略 —— 删除 CR 会同步删除云资源，要小心

**进阶标准**：你能画出 Crossplane + ArgoCD 的集成架构图，并说明从 Git 提交到云资源创建完整的自动化链路。

---

> **一句话总结**：Crossplane 适合"已经在 K8s 里了，想把云资源也拉进来一起管"的团队。不是 Terraform 的替代品，而是不同场景的工具。

---

## 2️⃣ 聚焦核心 20%

> Crossplane 中最重要的 20% 内容：**安装 + CR 创建云资源 + Composition 封装**。这 20% 覆盖了日常使用中 80% 的场景。

---

| 次数 | 主题 | 核心内容 | 练习资源 | 复盘问题 |
|:----:|------|----------|----------|----------|
| 1 | 安装 Crossplane + AWS Provider | Helm 安装、Provider 部署、凭证配置 | 在一个 K8s 集群上部署 Crossplane + AWS Provider | `kubectl get provider` 返回什么才表示安装成功？ |
| 2 | 创建第一个 S3 桶 | 编写 Bucket CR、kubectl apply、查看状态、删除 | 创建、查看、删除一个 S3 桶 | `READY=True` 和 `SYNCED=True` 各代表什么？ |
| 3 | 创建完整基础设施 | VPC + Subnet + SecurityGroup + EC2 的 YAML 组合 | 部署 infra.yaml 创建一套完整网络 + EC2 | 如果 EC2 创建失败，你第一步该查什么？ |
| 4 | 理解 Composition 思想 | 平台工程分层、用户 vs 平台团队的责任划分 | 画一张"没有 Composition vs 有 Composition"的对比图 | Composition 的本质是降低了谁的负担？ |
| 5 | 编写 XRD | CompositeResourceDefinition 的定义方式和 schema | 定义一个"MySQL 数据库产品"的 XRDatabase | 为什么 XRD 需要 OpenAPI schema？ |
| 6 | 编写 Composition | resources 列表、base 定义、patches 注入 | 写一个包含 Subnet + RDS 的 Composition | 如果用户没有传 storageGB，会用什么值？ |
| 7 | 用户使用 Composition | 用 XRDatabase 声明 3 行 YAML 创建完整数据库 | 用 XRDatabase 创建一个 50GB PostgreSQL | Crossplane 在后台自动产生了哪些资源？ |
| 8 | 深入学习 Patches | FromCompositeFieldPath、ToCompositeFieldPath 等 | 尝试给 Composition 添加一个 engineVersion 的 patch | patches 的 fromFieldPath 写错了会怎样？ |
| 9 | Crossplane vs Terraform | 对比每个维度，理解各自适用场景 | 画一张对比表，列出你的项目最适合用哪个 | 你的场景更适合 Crossplane 还是 Terraform？ |
| 10 | 综合实战：GitOps 集成 | Crossplane + ArgoCD 的完整流程 | 模拟一个 Git 仓库→ArgoCD→Crossplane→AWS 的链路 | 如果手动在 AWS 改了资源，Crossplane 会怎么做？ |

> **学习建议**：每次学完后花 5 分钟回答"复盘问题"列。答不出来就回看对应内容。

---

## 3️⃣ AI 考官测试

> 学完以上内容后，告诉 AI："开始 Crossplane 考官测试"，按以下流程进行：

**测试规则**
1. AI 一次只问 **一个问题**
2. 从**简单到困难**逐步深入
3. 你回答后，AI 立即给出**反馈** + **针对性讲解**

**测试范围**
- 第一轮：概念题（Crossplane 的核心架构是什么？CR 和 CRD 的区别？）
- 第二轮：操作题（写出安装 Crossplane 和 AWS Provider 的完整命令）
- 第三轮：场景题（"你的团队有 K8s 想做 GitOps，云资源该怎么管理？"）
- 第四轮：设计题（"设计一个 Composition，让开发者一行 YAML 就能创建一个带 VPC 的 RDS"）

**启动方式**
> "开始 Crossplane 考官测试，从简单概念题开始。"

---

## 4️⃣ 制作速查表

### 速查表 A：基础操作

| 项目 | 内容 |
|------|------|
| **一句话** | Crossplane 把云资源变成 K8s 自定义资源（CRD），用 `kubectl apply` 就能创建 AWS/GCP/Azure 资源 |
| **核心概念** | CR（自定义资源）= 云资源的声明 · Provider = 调用云 API 的控制器 · Composition = 资源组合模板 · XRD = 自定义 API 定义 |
| **真实案例** | 平台团队定义一个 XRDS 产品，开发者在 YAML 里填 `storageGB: 50`，Crossplane 自动创建 SecurityGroup + RDS 实例 |

**基础命令**

```bash
# 安装
helm install crossplane crossplane-stable/crossplane --namespace crossplane-system --create-namespace

# Provider 管理
kubectl get provider                    # 查看 Provider 状态
kubectl get providerconfig             # 查看凭证配置

# 资源管理（和普通 K8s 资源一样）
kubectl get bucket                     # 列出 S3 桶
kubectl get vpc                        # 列出 VPC
kubectl get instance                   # 列出 EC2
kubectl get xdatabase                  # 列出自定义产品
kubectl describe bucket my-bucket      # 查看详情
kubectl delete bucket my-bucket        # 删除（销毁 AWS 资源）

# Composition
kubectl get composition                # 查看已定义的 Composition
kubectl get compositeresourcedefinition # 查看已定义的 XRD
```

**常见错误**
- ❌ CR 创建后 `READY=False` 不 debug —— 用 `kubectl describe` 看 events
- ❌ 装完 Crossplane 不装 Provider —— 装完不会自动有 AWS 能力
- ❌ 手动改 AWS 资源，Crossplane 会自动改回来（持续同步）
- ❌ 删除 CR 会同步删除 AWS 资源，不是软删除

**检查清单**
- [ ] K8s 集群是否就绪？
- [ ] Crossplane 是否已安装并运行？
- [ ] AWS Provider 是否已安装（`kubectl get provider`）？
- [ ] AWS 凭证 Secret 是否正确创建？
- [ ] ProviderConfig 是否正确引用 Secret？
- [ ] CR 创建后 `READY=True` 且 `SYNCED=True`？
- [ ] Composition 的 `patches` 路径是否正确？

**5 个自测题**
1. 安装 Crossplane 到 crossplane-system 命名空间的 Helm 命令是什么？
2. 创建一个 S3 存储桶的 CR，要填哪几个核心字段？
3. `kubectl get bucket` 返回的 READY 和 SYNCED 分别代表什么？
4. Composition 中的 patches 的作用是什么？
5. 删除 CR 后，对应的 AWS 资源会被怎样处理？

---

### 速查表 B：Composition

| 项目 | 内容 |
|------|------|
| **一句话** | Composition 把一组云资源打包成"产品"，用户只需要填 3-5 个参数就能创建复杂的基础设施 |
| **三步流程** | XRD（定义 API）→ Composition（定义实现）→ Claim（用户使用）|
| **核心机制** | patches 把用户参数注入到资源定义中 · dependsOn 控制资源创建顺序 · base 提供默认值 |

**常见错误**
- ❌ XRD 还没注册成功就创建 Composition —— 等 CRD 就绪
- ❌ patches 路径写错，Crossplane 不报错 —— 用 describe 检查是否注入成功
- ❌ base 中遗漏关键字段 —— 被 patches 覆盖不了的字段必须写在 base 里
- ❌ 给用户太多参数选项 —— 违背 Composition"简化"的初衷

**检查清单**
- [ ] XRD 的 OpenAPI schema 是否正确？
- [ ] Composition 的 `compositeTypeRef` 是否指向正确的 XRD？
- [ ] 每个 Managed Resource 的 base 是否包含完整的默认值？
- [ ] patches 的 `fromFieldPath` 和 `toFieldPath` 是否匹配？
- [ ] 用户侧的 Claim 能否只用 3-5 个参数完成？

**5 个自测题**
1. XRD（CompositeResourceDefinition）的作用是什么？
2. Composition 中的 patches 有哪几种类型？
3. 用户创建 XDatabase 后，Crossplane 自动创建了哪几类资源？
4. 为什么 Composition 被称为"平台工程"的核心工具？
5. 如何查看一个 XDatabase 实例的详细状态？

---

## 5️⃣ 筛选优质资源

| # | 资源 | 类型 | 难度 | 适合人群 | 怎么用 |
|:-:|------|------|:----:|----------|--------|
| 1 | [Crossplane 官方入门](https://docs.crossplane.io/latest/getting-started/) | 交互式教程 | ⭐⭐ | 零基础入门 | 跟着官方 quickstart 走一遍，配合本文 Lv1-Lv2 |
| 2 | [Crossplane Composition 文档](https://docs.crossplane.io/latest/concepts/composition/) | 官方文档 | ⭐⭐⭐ | 平台工程团队 | 学完 Lv3 后精读，理解 Composition 的所有细节 |
| 3 | [Upbound 官方课程](https://www.upbound.io/learn) | 在线课程 | ⭐⭐⭐ | 想系统学习的用户 | 按课程顺序学，覆盖从安装到生产部署 |
| 4 | [Crossplane vs Terraform 对比](https://docs.crossplane.io/latest/faq/) | FAQ | ⭐⭐ | 做技术选型的读者 | 读 FAQ 中的对比部分，结合本文 Lv4 |
| 5 | [Crossplane + ArgoCD 集成指南](https://crossplane.github.io/docs/v1.12/guides/argocd.html) | 实践指南 | ⭐⭐⭐⭐ | 想上生产 GitOps 的团队 | 学了 Lv5 后按指南一步步搭建 |

**不推荐的资源**
- 旧的 Crossplane v0.x 博客文章（API 变化大，旧语法已废弃）
- 过于简短的短视频教程（Crossplane 概念多，需要系统学习）

---

### 7 天学习路径

| 天 | 学习内容 | 使用资源 | 预计时间 |
|:--:|----------|----------|:--------:|
| Day 1 | 概念理解 + 架构（Lv 1） | 资源 #1 官方入门 + 本文 Lv1 | 1.5h |
| Day 2 | 安装 + 第一个 S3（Lv 2） | 资源 #1 Quickstart | 2h |
| Day 3 | VPC + EC2 完整示例（Lv 2） | 本文完整 YAML 示例 | 2h |
| Day 4 | Composition 概念 + XRD（Lv 3） | 资源 #2 Composition 文档 | 2h |
| Day 5 | Composition 实践 + patches（Lv 3） | 资源 #3 Upbound 课程 | 2h |
| Day 6 | 对比 + 技术选型（Lv 4） | 资源 #4 FAQ 对比 + 本文 Lv4 | 1.5h |
| Day 7 | GitOps 集成 + 生产考量（Lv 5） | 资源 #5 ArgoCD 集成指南 | 1.5h |

> **建议**：Day 1-3 上手操作，Day 4-5 深入 Composition，Day 6-7 对比和生产。每天不超过 2 小时。

---

## 6️⃣ 费曼循环巩固

> 费曼学习法：**如果你不能简单地解释它，你就没有真正理解它。**

### 用 12 岁孩子能懂的话来说

**Crossplane 是什么？**

想象你有一个**积木收纳盒**，每个格子里放了不同类型的积木（Pod、Service、Deployment）。你平时只用这个盒子里的积木搭东西。

现在你想用**外面的积木**（云资源：S3 桶、RDS 数据库），但外面的积木和盒子里的积木不兼容，你得用另一套工具去拿。

Crossplane 就是这个收纳盒的**扩展包**——它把外面的积木也做成盒子里的积木那样，从此你只需要一种方法就拿到所有积木。

**Crossplane 怎么工作的？**

你写一张纸条"我要一个 S3 存储桶"（这就是 CR），然后把纸条放进收纳盒里。

Crossplane 就像一个聪明的机器人，看到你的纸条后，它就跑到 AWS 那里帮你真的建了一个 S3 桶，然后把"建好了"的标签贴在纸条上。

从此这个 S3 桶就在收纳盒的"监视"下——如果你故意去 AWS 控制台改了什么，机器人会把它改回来，因为纸条上写的是"私有桶"，不能变成"公开桶"。

**Composition 是什么？**

想象你要搭一个"网站"（需要 VPC + 子网 + 安全组 + EC2 + RDS），以前得写 5 张纸条。

平台团队做了一件好事：他们定义了一个叫"标准网站"的新积木类型。你只要写一张纸条"我要一个标准网站，名字叫 my-app"，Crossplane 机器人就会自动帮你搭好那 5 样东西。

这张"一张纸条换一堆东西"的能力，就叫 Composition。

**Crossplane 和 Terraform 有什么不同？**

Terraform 像是一个**外挂工具箱**——你需要的时候打开它，用完收起来。

Crossplane 像是**收纳盒自带的扩展抽屉**——它就在收纳盒里，永远在监视着，不需要手动打开关闭。

如果你已经在用收纳盒（K8s），用 Crossplane 会很自然。如果你没有收纳盒（没有 K8s），专门去买一个收纳盒来装 Crossplane 这个扩展抽屉，成本太高了。

### 你的费曼练习

现在轮到你。按以下步骤做：

1. **复述**：合上文档，用自己的话给一个懂 K8s 但没听过 Crossplane 的同事解释
2. **检查漏洞**：对照以下问题

**自检问题**
- ❓ Crossplane 和 Terraform 的核心区别是什么？
- ❓ CR → Provider → AWS API 这条链路中，每个角色做了什么？
- ❓ Composition 解决了什么问题？（提示：平台团队 vs 开发者的分工）
- ❓ 为什么 Crossplane 天然适合 GitOps？
- ❓ 什么情况下不应该用 Crossplane？

### 循环方法

```
Round 1: 你用自己的话说 → 我发现漏洞并指出 → 你去查资料修正
Round 2: 你重新说（这次更准确） → 我再指出改进点
Round 3: 你再说 → 漏洞越来越少 → 你真正掌握了 ✅
```

开始费曼循环，告诉我："开始 Crossplane 费曼循环，我先用自己的话解释 Crossplane 是什么。"

---

> **最后说一句**：Crossplane 最核心的思想不是"用 YAML 管理云资源"，而是**"把云资源融入 K8s 的声明式管理体系中"**。懂了这一层，你就懂了 Crossplane 存在的意义。

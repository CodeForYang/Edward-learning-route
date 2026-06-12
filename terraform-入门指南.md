# 🏗️ Terraform 入门学习指南

> 适合：有基础 Docker/K8s 概念 + AWS 免费账号 + macOS
> 特点：和 K8s YAML 一样是声明式，但管理的是"云基础设施"而非"集群内部资源"
> 周期：7-10 天，每天 1-2 小时

---

## 📋 目录

1. [学习路线总览](#学习路线总览)
2. [前置准备](#前置准备)
3. [Day 1：认识 Terraform 和 IaC](#day-1认识-terraform-和-iac)
4. [Day 2：HCL 语法核心](#day-2hcl-语法核心)
5. [Day 3：变量、输出与数据源](#day-3变量输出与数据源)
6. [Day 4：状态管理（最重要的概念）](#day-4状态管理最重要的概念)
7. [Day 5：从单文件到模块化](#day-5从单文件到模块化)
8. [Day 6：实战 — 用 Terraform 创建 AWS 基础设施](#day-6实战--用-terraform-创建-aws-基础设施)
9. [Day 7：Terraform + K8s：创建 EKS 集群](#day-7terraform--k8s创建-eks-集群)
10. [Day 8：多环境管理与工作流](#day-8多环境管理与工作流)
11. [附录：常用命令速查](#附录常用命令速查)

---

## 学习路线总览

```
Day 1 ───── 什么是 IaC + 安装 Terraform + 跑通第一个例子
  │
Day 2 ───── HCL 语法：resource、provider、数据类型、表达式
  │
Day 3 ───── 变量、输出、数据源（让配置可复用）
  │
Day 4 ───── 状态管理（State）：本地 → 远程，锁定，导入（⚠️ 最重要）
  │
Day 5 ───── 模块化（Module）：写自己的模块，用别人的模块
  │
Day 6 ───── 🎯 实战：用 Terraform 创建 VPC + EC2 + RDS
  │
Day 7 ───── 🎯 K8s 联动：用 Terraform 创建 EKS 集群
  │
Day 8 ───── 多环境（dev/staging/prod）+ 工作流规范
```

---

## 前置准备

### 硬件/软件要求

- macOS（Intel 或 Apple Silicon）
- 终端
- AWS 账号（免费套餐即可，Day 6-7 用到）
- 你已经熟悉的 K8s/Docker 知识会很有帮助

### 安装工具

#### 1️⃣ 安装 Terraform

```bash
# 使用 Homebrew 安装
brew install terraform

# 验证
terraform version
# 输出类似：Terraform v1.9.0
```

#### 2️⃣ 安装 AWS CLI（Day 6 起用到）

```bash
brew install awscli

# 配置凭证
aws configure
# AWS Access Key ID [****************]: <你的 Key>
# AWS Secret Access Key [****************]: <你的 Secret>
# Default region name [us-east-1]: ap-northeast-1
# Default output format [json]: json

# 验证
aws sts get-caller-identity
```

#### 3️⃣ 了解你的文本编辑器

Terraform 文件后缀为 `.tf`，推荐安装 VSCode 的 **HashiCorp Terraform** 插件（语法高亮、自动补全）。

---

## Day 1：认识 Terraform 和 IaC

### 1.1 什么是 IaC？

**基础设施即代码（Infrastructure as Code）**——用代码来描述和管理你的云资源。

```
传统方式（手动）：
  登录 AWS 控制台 → 点"创建 EC2"→ 选镜像 → 配安全组 → 等创建
  → 另一同事也手动配 → 配置漂移 → "我记得上周不是这么配的啊？"
  
IaC 方式（Terraform）：
  写 main.tf → git commit → terraform apply
  → 全员一致 → 可版本控制 → 可代码审查 → 可重复创建
```

### 1.2 和 K8s YAML 的思想对比

如果你已经学过 K8s，这个对比能帮你快速理解：

```
K8s YAML：                    Terraform HCL：
────────                      ────────────
apiVersion: apps/v1           terraform {
kind: Deployment                required_providers {
spec:                             aws = {
  replicas: 3                       source  = "hashicorp/aws"
  template:                         version = "~> 5.0"
    spec:                        }
      containers:               }
        - name: app
          image: nginx
                                resource "aws_instance" "web" {
                                  ami           = "ami-xxx"
                                  instance_type = "t3.micro"
                                }

共同点：
  ✅ 声明式（Declarative）——你说"要什么"，工具算"怎么做"
  ✅ 幂等（Idempotent）     ——跑一次和跑多次结果一样
  ✅ 可版本控制              ——放 Git 里管理

不同点：
  ❗ K8s 管"集群内部的资源"（Pod/Service/Deployment）
  ❗ Terraform 管"基础设施"（服务器/网络/数据库/K8s 集群本身）
```

### 1.3 你的第一个 Terraform 配置

创建 `~/terraform-learning/` 目录并进入：

```bash
mkdir -p ~/terraform-learning && cd ~/terraform-learning
```

创建 `main.tf`：

```hcl
# 指定 Provider（相当于告诉 Terraform 我要管哪个平台）
terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
  }
}

# 创建一个本地文件
resource "local_file" "hello" {
  content  = "Hello Terraform!"
  filename = "${path.module}/hello.txt"
}
```

> 💡 这里用 `local` provider 而不是云平台，让你**不花钱**就能跑通第一个例子。

```bash
# 1. 初始化（下载 provider 插件）
terraform init

# 输出：
# Initializing the backend...
# Initializing provider plugins...
# - Installing hashicorp/local v2.5.1...
# Terraform has been successfully initialized!

# 2. 预览执行计划（看看 Terraform 要做什么）
terraform plan

# 输出：
# Plan: 1 to add, 0 to change, 0 to destroy.
# 注意：它只会告诉你，不会真做

# 3. 执行
terraform apply

# 会提示你确认，输入 yes 回车
# 输出：
# Apply complete! Resources: 1 added, 0 changed, 0 destroyed.

# 验证：文件确实被创建了
cat hello.txt
# Hello Terraform!
```

### 1.4 核心概念：Desired State 与 Current State

```
你写的内容（Desired State）：        Terraform 的运作：
┌────────────────────────┐          ┌────────────────────┐
│ resource "local_file"  │  ──→     │ 读取 main.tf       │
│   content = "..."      │          │ 读取当前 state     │
│   filename = "..."     │          │ 对比差异            │
└────────────────────────┘          │ 算出执行计划 (plan) │
                                    │ 执行变更 (apply)    │
                                    └────────────────────┘
```

**关键理解：** Terraform 不是"执行指令"，而是"计算差异"。它比较你写的配置（Desired）和实际存在的资源（Current），算出最小的变更步骤。

### 1.5 清理资源

```bash
# 销毁所有 Terraform 管理的资源
terraform destroy

# 会提示确认，输入 yes
# 本地文件 hello.txt 被删除 ✅
```

### 1.6 K8s vs Terraform 操作对比

| 操作 | K8s 命令 | Terraform 命令 |
|------|----------|---------------|
| 初始化 | — | `terraform init` |
| 预览 | `kubectl apply --dry-run=client` | `terraform plan` |
| 应用 | `kubectl apply -f` | `terraform apply` |
| 删除 | `kubectl delete -f` | `terraform destroy` |
| 查看状态 | `kubectl get` | `terraform show` |
| 查看当前配置 | `kubectl get -o yaml` | `terraform state list` |

### ✅ Day 1 学习成果检查

- [ ] 理解了什么是 IaC 和声明式配置
- [ ] 安装了 Terraform
- [ ] 跑通了第一个例子（创建/查看/销毁本地文件）
- [ ] 理解了 Desired State vs Current State 的概念
- [ ] 理解了 `terraform init` / `plan` / `apply` / `destroy` 的基本流程

---

## Day 2：HCL 语法核心

### 2.1 HCL vs YAML 对照

```
K8s YAML 语法：               Terraform HCL 语法：
─────────────                 ─────────────────
apiVersion: v1               # 块（Block）用花括号
kind: Pod                    resource "type" "name" {
metadata:                      argument = "value"
  name: my-pod                 嵌套块 {
spec:                            key = "value"
  containers:                  }
    - name: app              }
      image: nginx
```

**关键区别：**
- HCL 是配置语言（有逻辑、表达式、函数）
- YAML 是数据序列化格式（纯数据结构）
- HCL 支持 `for` 循环、`if` 条件、函数调用 —— **YAML 不行**

### 2.2 HCL 基础结构

```hcl
# ──── 块（Block）──── Terraform 的基本组成单元
resource "aws_instance" "web" {
  # ↑ 块类型    ↑ 块标签（Local Name）
  
  # ──── 参数（Argument）────
  ami           = "ami-0c55b159cbfafe1f0"
  instance_type = "t2.micro"
  
  # ──── 嵌套块（Nested Block）────
  tags = {
    Name = "Web Server"
    Env  = "dev"
  }
}
```

**块的三种常见类型：**

```hcl
# 1. 资源块 — 描述你要创建的资源
resource "aws_vpc" "main" {
  cidr_block = "10.0.0.0/16"
}

# 2. 数据源块 — 查询已有资源的信息
data "aws_ami" "ubuntu" {
  most_recent = true
  owners      = ["099720109477"]
}

# 3. 变量块 — 定义输入参数
variable "region" {
  type    = string
  default = "ap-northeast-1"
}
```

### 2.3 引用与表达式

这是 HCL 和纯 YAML 最大的区别——**HCL 可以引用其他资源的值**：

```hcl
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id    # 引用数据源
  instance_type = var.instance_type         # 引用变量
}

resource "aws_eip" "web_ip" {
  instance = aws_instance.web.id            # 引用另一个资源
  # ↑ Terraform 自动推导依赖关系：必须先创建实例，再创建弹性 IP
}
```

**引用语法：**

| 表达式 | 含义 |
|--------|------|
| `aws_instance.web.id` | 资源的属性 |
| `data.aws_ami.ubuntu.id` | 数据源的属性 |
| `var.instance_type` | 变量的值 |
| `local.env_name` | 本地值的值 |
| `module.vpc.vpc_id` | 模块的输出 |

### 2.4 依赖关系（Dependency）

```hcl
# Terraform 会自动分析引用关系来构建依赖图
# 但有时候你需要显式声明（比如 A 不直接引用 B，但需要等 B 先创建）

resource "aws_s3_bucket" "logs" {
  bucket = "my-app-logs"
}

resource "aws_instance" "web" {
  ami           = "ami-xxx"
  instance_type = "t3.micro"
  
  depends_on = [                     # 显式依赖
    aws_s3_bucket.logs               # 先等 S3 创建完再创建 EC2
  ]
}
```

> 💡 **最佳实践**：尽量用自然引用（隐式依赖），少用 `depends_on`。依赖关系越明确，Terraform 的执行计划越高效。

### 2.5 常用内置函数

```hcl
locals {
  # 字符串拼接
  name      = "${var.project}-${var.environment}"   # "myapp-dev"
  
  # 或者用 format
  name2     = format("%s-%s", var.project, var.environment)
  
  # 列表操作
  all_zones = ["a", "b", "c"]
  
  # 合并 map
  merged_tags = merge(
    var.default_tags,
    { Name = local.name }
  )
  
  # 条件表达式（三目运算）
  instance_size = var.env == "prod" ? "t3.large" : "t3.micro"
  #              ↑ 条件              ↑ 真时取值     ↑ 假时取值
}
```

### 2.6 完整的组合例子

创建 `~/terraform-learning/day2-demo.tf`：

```hcl
terraform {
  required_providers {
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}

# 生成随机字符串
resource "random_string" "suffix" {
  length  = 6
  special = false
  upper   = false
}

# 创建两个文件，文件名中引用随机值
resource "local_file" "config" {
  content = <<-EOF
    server_name=web-${random_string.suffix.result}
    environment=${var.env}
    log_level=${local.log_level}
  EOF
  filename = "${path.module}/config-${random_string.suffix.result}.txt"
}

# 变量
variable "env" {
  type    = string
  default = "dev"
}

# 本地值
locals {
  log_level = var.env == "prod" ? "warn" : "debug"
}

# 输出
output "created_file" {
  value       = local_file.config.filename
  description = "刚刚创建的文件路径"
}
```

然后创建 `terraform.tfvars`（变量值文件）：

```hcl
env = "staging"
```

运行：

```bash
terraform init
terraform plan
terraform apply

# 看看生成的文件
cat config-*.txt
```

### ✅ Day 2 学习成果检查

- [ ] 理解了 Block、Argument、Nested Block 的概念
- [ ] 理解了如何跨资源引用属性（`resource.name.attribute`）
- [ ] 理解隐式依赖和显式依赖（`depends_on`）
- [ ] 会使用基本的内置函数（format、merge、条件表达式）
- [ ] 会创建 `terraform.tfvars` 文件来覆盖变量默认值

---

## Day 3：变量、输出与数据源

### 3.1 变量（Variable）—— 让你的配置可复用

```hcl
# 定义变量
variable "instance_type" {
  description = "EC2 实例规格"            # 说明
  type        = string                    # 类型约束
  default     = "t3.micro"               # 默认值（可选）
  
  validation {                           # 自定义校验（可选）
    condition     = contains(["t3.micro", "t3.small", "t3.medium"], var.instance_type)
    error_message = "实例类型必须是 t3.micro / t3.small / t3.medium 之一。"
  }
}
```

**变量的类型：**

```hcl
variable "name"       { type = string }
variable "count"      { type = number }
variable "enabled"    { type = bool }
variable "tags"       { type = map(string) }
variable "azs"        { type = list(string) }
variable "instance"   { type = object({
                          size = string
                          ami  = string
                        }) }
```

**给变量赋值的方式（优先级从低到高）：**

```bash
# 方式 1：默认值（代码中写 default）
variable "env" { default = "dev" }

# 方式 2：terraform.tfvars 文件（推荐）
# terraform.tfvars
env = "staging"

# 方式 3：环境变量
export TF_VAR_env=prod

# 方式 4：命令行参数（优先级最高）
terraform apply -var="env=prod"
```

### 3.2 输出（Output）—— 让 Terraform 告诉你结果

就像函数的返回值，配置执行完后暴露一些有用的信息：

```hcl
# 在 main.tf 中定义输出
output "instance_ip" {
  value       = aws_instance.web.public_ip
  description = "Web 服务器的公网 IP"
  sensitive   = false    # 如果 true，会在日志中隐藏值
}
```

```bash
# 查看输出
terraform output          # 列出所有输出
terraform output instance_ip  # 只看这个
```

### 3.3 数据源（Data Source）—— 查询已有资源

数据源让你**读取**已在云平台上存在的资源信息，而不是创建新的：

```hcl
# 查询当前 AWS 账号信息
data "aws_caller_identity" "current" {}

# 查询最新的 Ubuntu AMI
data "aws_ami" "ubuntu" {
  most_recent = true
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-24.04-*"]
  }
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
  owners = ["099720109477"]  # Canonical
}

# 在资源中使用
resource "aws_instance" "web" {
  ami           = data.aws_ami.ubuntu.id      # 引用数据源
  instance_type = "t3.micro"
}

output "account_id" {
  value = data.aws_caller_identity.current.account_id
}
```

> 💡 **数据源 vs 资源：**
> - `resource` = **创建**新资源
> - `data` = **读取**已有资源
>
> 数据源让你的配置可以引用外部资源——比如引用一个由其他人（或另一个 Terraform 项目）创建的 VPC。

### 3.4 本地值（Local）—— 变量计算的中间结果

```hcl
locals {
  # 组合变量
  name = format("%s-%s", var.project, var.env)
  
  # 根据环境计算不同的值
  instance_type = var.env == "prod" ? "t3.large" : "t3.micro"
  replicas      = var.env == "prod" ? 3 : 1
  
  # 合并标签
  tags = merge(var.default_tags, {
    Name = local.name
  })
}

# 使用 local
resource "aws_instance" "web" {
  instance_type = local.instance_type
  tags          = local.tags
}
```

> `locals` 和 `var` 的区别：
> - `var` = 外部输入的参数（用户赋值）
> - `local` = 内部计算的中间值（由其他变量计算得来）

### ✅ Day 3 学习成果检查

- [ ] 会定义和使用不同类型的变量
- [ ] 理解了变量的 4 种赋值方式及优先级
- [ ] 会定义 output 并用 `terraform output` 查看
- [ ] 会使用数据源查询已有资源
- [ ] 会使用 locals 计算中间值

---

## Day 4：状态管理（最重要的概念）

### 4.1 什么是 State？

```hcl
# 你写的内容
resource "aws_instance" "web" {
  ami           = "ami-xxx"
  instance_type = "t3.micro"
}
```

Terraform 运行后，会创建一个 `terraform.tfstate` 文件。里面记录了**实际存在的资源信息**：

```json
{
  "resources": [
    {
      "type": "aws_instance",
      "name": "web",
      "instances": [
        {
          "attributes": {
            "id": "i-0a1b2c3d4e5f",
            "ami": "ami-xxx",
            "public_ip": "54.123.45.67"
          }
        }
      ]
    }
  ]
}
```

**Terraform 的工作循环：**

```
  main.tf（你写的 Desired State）
     │
     ▼
  读取 terraform.tfstate（当前实际状态）
     │
     ▼
  对比差异
     │
     ▼
  生成执行计划（plan）
     │
     ▼
  执行变更（apply）
     │
     ▼
  更新 terraform.tfstate ✅
```

### 4.2 为什么 State 管理是 Terraform 最大的坑？

**场景一：你没 commit tfstate**

```bash
你：  terraform apply  → 创建了 EC2 → state 记了 i-xxx
同事：没拿到你的 state → 又创建了一台 EC2
→ 两个人各管各的，谁也看不见谁
```

**场景二：tfstate 丢了**

```bash
# 你不小心删了 terraform.tfstate
rm terraform.tfstate

# 再跑 terraform plan
# → Terraform 以为你啥也没创建过
# → 计划里说"再创建一台"
# → 但 AWS 上实际已经有一台了
# → 你花了双份的钱，或者运行报错
```

**场景三：多人在 AWS 手动操作**

```bash
# 同事去 AWS 控制台手工删了一个安全组
# Terraform 不知道 → tfstate 里还记着那个安全组
# 下次 apply → Terraform 尝试操作一个不存在的资源 → 报错 ❌
```

**解决：远程状态 + 状态锁定**

### 4.3 远程状态（Remote State）

本地 `terraform.tfstate` 只适合一个人玩玩。团队协作必须用远程存储：

```
                    ┌────────────┐
                    │  S3 Bucket  │ ← 存 tfstate 文件
                    │  (中央存储) │
                    └─────┬──────┘
                          │
              ┌───────────┼───────────┐
              │           │           │
         你拉取 state   CI/CD 拉取  同事拉取
```

创建 `backend.tf`：

```hcl
terraform {
  backend "s3" {
    bucket         = "my-terraform-state"           # S3 桶
    key            = "prod/terraform.tfstate"       # 路径（区分环境）
    region         = "ap-northeast-1"               # 区域
    encrypt        = true                           # 加密
    dynamodb_table = "terraform-state-lock"         # 锁定表（防并发）
  }
}
```

**配套 DynamoDB 锁表**（先手动创建，或者用另一个 Terraform 项目创建）：

```bash
aws dynamodb create-table \
  --table-name terraform-state-lock \
  --attribute-definitions AttributeName=LockID,AttributeType=S \
  --key-schema AttributeName=LockID,KeyType=HASH \
  --billing-mode PAY_PER_REQUEST \
  --region ap-northeast-1
```

> 💡 S3 负责"存文件"，DynamoDB 负责"上锁"——两人同时 apply 时，只有一个人能拿到锁。

### 4.4 状态操作命令

```bash
# 列出 state 中的所有资源
terraform state list

# 查看某个资源在 state 中的详情
terraform state show aws_instance.web

# 把已有资源导入 Terraform 管理
terraform import aws_instance.web i-0a1b2c3d4e5f

# 从 state 中移除资源（但不真删资源）
terraform state rm aws_instance.web

# 移动资源在 state 中的位置（比如重构后改名）
terraform state mv aws_instance.web aws_instance.old_web
```

### 4.5 常见故障处理

```bash
# 场景：state 损坏或丢失
# 方案：用实际资源恢复 state
terraform import <resource_type>.<name> <resource_id>

# 场景：state 和实际不一致
# 方案：刷新 state
terraform refresh    # 用实际资源更新 state

# 场景：远程 state 读取失败
# 方案：重新拉取
terraform init -reconfigure
```

### ✅ Day 4 学习成果检查

- [ ] 理解 State 文件的作用和内容结构
- [ ] 理解为什么多人协作时必须用远程 State
- [ ] 会配置 S3 远程后端 + DynamoDB 锁表
- [ ] 掌握 `terraform state list` / `show` / `import` / `rm`
- [ ] 理解 State 损坏或丢失后的恢复思路

---

## Day 5：从单文件到模块化

### 5.1 为什么需要模块？

```
单文件（❌ 不推荐用于真实项目）：

main.tf                     ← 一坨，500 行
variables.tf                ← 变量混在一起
terraform.tfvars            ← 变量值
outputs.tf                  ← 输出
```

```
模块化（✅ 推荐）：

modules/
├── networking/
│   ├── main.tf              ← VPC、子网、路由表
│   ├── variables.tf
│   └── outputs.tf
├── compute/
│   ├── main.tf              ← EC2、Auto Scaling
│   ├── variables.tf
│   └── outputs.tf
└── database/
    ├── main.tf              ← RDS
    ├── variables.tf
    └── outputs.tf

environments/
├── dev/
│   └── main.tf              ← 调用各模块，传参数
└── prod/
    └── main.tf              ← 调用各模块，传不同的参数
```

### 5.2 创建一个简单的模块

创建 `~/terraform-learning/modules/networking/main.tf`：

```hcl
variable "vpc_cidr" {
  type = string
}

variable "env" {
  type = string
}

resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.env}-vpc"
    Env  = var.env
  }
}

resource "aws_subnet" "public" {
  count             = 2
  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.env}-public-${count.index}"
    Env  = var.env
  }
}

data "aws_availability_zones" "available" {
  state = "available"
}

output "vpc_id" {
  value = aws_vpc.main.id
}

output "subnet_ids" {
  value = aws_subnet.public[*].id
}
```

### 5.3 使用模块

创建 `~/terraform-learning/environments/dev/main.tf`：

```hcl
terraform {
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-1"
}

# 调用网络模块
module "networking" {
  source   = "../../modules/networking"
  vpc_cidr = "10.0.0.0/16"
  env      = "dev"
}

# 使用模块的输出
output "created_vpc_id" {
  value = module.networking.vpc_id
}
```

### 5.4 使用 Terraform Registry 的公共模块

别人写好的模块，直接拿来用：

```hcl
# 标准 VPC 模块（来自 Terraform Registry）
module "vpc" {
  source = "terraform-aws-modules/vpc/aws"   # registry 上的模块路径
  version = "~> 5.0"

  name = "my-vpc"
  cidr = "10.0.0.0/16"

  azs             = ["ap-northeast-1a", "ap-northeast-1c"]
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway = true
  enable_vpn_gateway = false

  tags = {
    Environment = "dev"
  }
}
```

> 💡 Terraform Registry（registry.terraform.io）就像 Docker Hub——上面有官方和社区维护的各类模块。

### ✅ Day 5 学习成果检查

- [ ] 理解模块化的必要性和目录结构规范
- [ ] 会创建自己的模块（source 用本地路径）
- [ ] 会使用 `source` 引用本地和 Registry 上的模块
- [ ] 理解模块的输入（variables）和输出（outputs）

---

## Day 6：实战 — 用 Terraform 创建 AWS 基础设施

> 🎯 本日目标：用 Terraform 在东京区域创建一整套基础设施：VPC + 子网 + EC2 + RDS + 安全组。
> 这是生产环境中最常见的资源组合，也是面试中高频考察的内容。

### 6.1 项目结构

```
~/terraform-learning/real-demo/
├── main.tf              # 主配置
├── variables.tf          # 变量定义
├── terraform.tfvars      # 变量值
├── outputs.tf            # 输出
└── provider.tf           # Provider 配置
```

### 6.2 编写配置

创建 `provider.tf`：

```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = var.region
}
```

创建 `variables.tf`：

```hcl
variable "region" {
  description = "AWS 区域"
  type        = string
  default     = "ap-northeast-1"
}

variable "project" {
  description = "项目名称"
  type        = string
  default     = "terraform-demo"
}

variable "env" {
  description = "环境名称"
  type        = string
  default     = "dev"
}

variable "db_password" {
  description = "数据库密码"
  type        = string
  sensitive   = true   # 标记为敏感，日志中隐藏
}
```

创建 `main.tf`（这是核心——包含所有资源）：

```hcl
# ────────────────────────────────────────────
# 1. VPC（虚拟私有网络）
# ────────────────────────────────────────────
resource "aws_vpc" "main" {
  cidr_block           = "10.0.0.0/16"
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.project}-${var.env}-vpc"
  }
}

# ────────────────────────────────────────────
# 2. 子网（2 个公有子网，2 个私有子网）
# ────────────────────────────────────────────
data "aws_availability_zones" "available" {
  state = "available"
}

# 公有子网（放 Web 服务器、负载均衡）
resource "aws_subnet" "public" {
  count = 2

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  map_public_ip_on_launch = true   # 自动分配公网 IP

  tags = {
    Name = "${var.project}-${var.env}-public-${count.index}"
  }
}

# 私有子网（放数据库、内部服务）
resource "aws_subnet" "private" {
  count = 2

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 2)
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.project}-${var.env}-private-${count.index}"
  }
}

# ────────────────────────────────────────────
# 3. 互联网网关（让 VPC 能访问公网）
# ────────────────────────────────────────────
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id

  tags = {
    Name = "${var.project}-${var.env}-igw"
  }
}

# ────────────────────────────────────────────
# 4. 路由表（公有子网连 IGW，私有子网连 NAT）
# ────────────────────────────────────────────
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  route {
    cidr_block = "0.0.0.0/0"
    gateway_id = aws_internet_gateway.main.id
  }

  tags = {
    Name = "${var.project}-${var.env}-public-rt"
  }
}

resource "aws_route_table_association" "public" {
  count          = 2
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# ────────────────────────────────────────────
# 5. 安全组（防火墙规则）
# ────────────────────────────────────────────

# Web 服务器安全组（开放 80 和 443）
resource "aws_security_group" "web" {
  name        = "${var.project}-${var.env}-web-sg"
  description = "Allow HTTP/HTTPS"
  vpc_id      = aws_vpc.main.id

  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"
    cidr_blocks = ["0.0.0.0/0"]
  }

  tags = {
    Name = "${var.project}-${var.env}-web-sg"
  }
}

# 数据库安全组（只允许来自 Web 安全组的流量）
resource "aws_security_group" "db" {
  name        = "${var.project}-${var.env}-db-sg"
  description = "Allow DB access from web tier"
  vpc_id      = aws_vpc.main.id

  ingress {
    description     = "MySQL"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]   # 只放行 Web 安全组的流量
  }

  tags = {
    Name = "${var.project}-${var.env}-db-sg"
  }
}

# ────────────────────────────────────────────
# 6. EC2 实例（Web 服务器）
# ────────────────────────────────────────────

# 查找最新的 Amazon Linux 2 AMI
data "aws_ami" "amazon_linux" {
  most_recent = true
  owners      = ["amazon"]

  filter {
    name   = "name"
    values = ["amzn2-ami-hvm-*-x86_64-gp2"]
  }
}

resource "aws_instance" "web" {
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t3.micro"
  subnet_id              = aws_subnet.public[0].id
  vpc_security_group_ids = [aws_security_group.web.id]
  associate_public_ip_address = true

  user_data = <<-EOF
    #!/bin/bash
    yum update -y
    yum install -y httpd
    systemctl start httpd
    systemctl enable httpd
    echo "<h1>Hello from Terraform!</h1>" > /var/www/html/index.html
  EOF

  tags = {
    Name = "${var.project}-${var.env}-web-server"
  }
}

# ────────────────────────────────────────────
# 7. RDS 数据库（MySQL）
# ────────────────────────────────────────────

resource "aws_db_subnet_group" "main" {
  name       = "${var.project}-${var.env}-db-subnet-group"
  subnet_ids = aws_subnet.private[*].id

  tags = {
    Name = "${var.project}-${var.env}-db-subnet-group"
  }
}

resource "aws_db_instance" "main" {
  identifier = "${var.project}-${var.env}-mysql"

  engine         = "mysql"
  engine_version = "8.0"
  instance_class = "db.t3.micro"

  db_name  = "appdb"
  username = "admin"
  password = var.db_password

  db_subnet_group_name   = aws_db_subnet_group.main.name
  vpc_security_group_ids = [aws_security_group.db.id]

  skip_final_snapshot  = true       # 学习环境跳过快照
  publicly_accessible  = false      # 不开放公网访问
  allocated_storage    = 20

  tags = {
    Name = "${var.project}-${var.env}-mysql"
  }
}
```

创建 `outputs.tf`：

```hcl
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

output "web_server_ip" {
  description = "Web 服务器公网 IP"
  value       = aws_instance.web.public_ip
}

output "db_endpoint" {
  description = "数据库连接地址"
  value       = aws_db_instance.main.endpoint
  sensitive   = true
}

output "web_server_url" {
  value = "http://${aws_instance.web.public_ip}"
}
```

创建 `terraform.tfvars`：

```hcl
region      = "ap-northeast-1"
project     = "terraform-demo"
env         = "dev"
db_password = "SafePassword123!"    # ⚠️ 真实项目不要硬编码！
```

### 6.3 部署

```bash
cd ~/terraform-learning/real-demo

terraform init

terraform plan    # 先看计划，确认没有意外

terraform apply   # 输入 yes
```

部署后访问 Web 服务器：

```bash
# 查看分配的 IP
terraform output web_server_ip

# 用 curl 或浏览器访问
curl http://<输出的 IP>
# 应该看到：<h1>Hello from Terraform!</h1>

# 查看所有输出
terraform output
```

### 6.4 理解 Terraform 的执行依赖图

当你执行 `terraform apply` 时，Terraform 自动推导出的依赖关系是这样的：

```
aws_vpc.main
  ├── aws_subnet.public[*]      ← 需要 VPC
  ├── aws_subnet.private[*]     ← 需要 VPC
  ├── aws_internet_gateway.main ← 需要 VPC
  └── aws_security_group.web    ← 需要 VPC
      └── aws_security_group.db ← 需要 aws_security_group.web（引用其 ID）
          └── aws_db_instance.main  ← 需要安全组 + 子网组

aws_internet_gateway.main
  └── aws_route_table.public    ← 需要 IGW
      └── aws_route_table_association.public[*]

aws_subnet.public[*]
  └── aws_instance.web          ← 需要子网
```

Terraform 会按依赖顺序创建，**逆序销毁**。这就是声明式的威力——你不用写"先创建 A，再创建 B"，Terraform 自己会算。

### 6.5 清理资源

```bash
terraform destroy
# 输入 yes

# 确认全部删除
aws ec2 describe-instances --region ap-northeast-1 --filters "Name=tag:Name,Values=terraform-demo-dev-*"
# 应该返回空
```

### ✅ Day 6 学习成果检查

- [ ] 能用 Terraform 创建 VPC + 子网 + 路由表
- [ ] 能创建安全组并理解安全组间引用
- [ ] 能创建 EC2 实例（含 User Data 脚本）
- [ ] 能创建 RDS 数据库（含子网组）
- [ ] 理解了 `cidrsubnet` 等函数的作用
- [ ] 理解了 Terraform 自动构建依赖图的原理

---

## Day 7：Terraform + K8s — 创建 EKS 集群

> 🎯 本日目标：用 Terraform 在 AWS 上创建一个完整的 EKS 集群。
> 这是 Terraform 在 K8s 场景中最常见的用途——和你之前 K8s 入门指南中的 Bonus 部分做的同一件事，但用 Terraform 实现。

### 7.1 为什么要用 Terraform 创建 EKS 而非 eksctl？

```
eksctl:                     Terraform:
────────                    ─────────
✅ 简单快速                   ✅ 可以管理 EKS 之外的资源（VPC/RDS/等）
❌ 不能管其他 AWS 资源          ✅ 统一的 IaC 工具
❌ 定制化有限                   ✅ 细粒度控制
❌ 状态管理不完善               ✅ 完整的 State 管理
```

**真实工作流：** 用 Terraform 创建 EKS 集群，等集群就绪后，再用 Helm/ArgoCD 部署应用到集群内部。

### 7.2 最小 EKS 配置

创建 `~/terraform-learning/eks-demo/` 目录：

```bash
mkdir -p ~/terraform-learning/eks-demo && cd $_
```

创建 `main.tf`：

```hcl
terraform {
  required_version = ">= 1.0"
  required_providers {
    aws = {
      source  = "hashicorp/aws"
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-1"
}

# ────────────────────────────────────────────
# 1. VPC（EKS 需要 VPC 支持）
# ────────────────────────────────────────────
data "aws_availability_zones" "available" {}

module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"
  version = "~> 5.0"

  name = "eks-demo-vpc"
  cidr = "10.0.0.0/16"

  azs             = slice(data.aws_availability_zones.available.names, 0, 2)
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]

  enable_nat_gateway   = true
  enable_dns_hostnames = true

  public_subnet_tags = {
    "kubernetes.io/role/elb" = "1"
  }

  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = "1"
  }
}

# ────────────────────────────────────────────
# 2. EKS 集群
# ────────────────────────────────────────────
module "eks" {
  source  = "terraform-aws-modules/eks/aws"
  version = "~> 20.0"

  cluster_name    = "eks-demo-tokyo"
  cluster_version = "1.30"

  cluster_endpoint_public_access = true

  # 指定子网（控制平面使用）
  vpc_id     = module.vpc.vpc_id
  subnet_ids = module.vpc.private_subnets

  # 节点组
  eks_managed_node_groups = {
    main = {
      desired_size = 2
      min_size     = 1
      max_size     = 4

      instance_types = ["t3.medium"]

      tags = {
        Role = "worker"
      }
    }
  }

  tags = {
    Environment = "dev"
  }
}
```

创建 `outputs.tf`：

```hcl
output "cluster_name" {
  description = "EKS 集群名称"
  value       = module.eks.cluster_name
}

output "cluster_endpoint" {
  description = "EKS API Server 地址"
  value       = module.eks.cluster_endpoint
}

output "configure_kubectl" {
  description = "配置 kubectl 的命令"
  value       = "aws eks update-kubeconfig --region ap-northeast-1 --name ${module.eks.cluster_name}"
}
```

### 7.3 部署

```bash
cd ~/terraform-learning/eks-demo

terraform init
terraform plan
terraform apply

# 输出中会看到：
# configure_kubectl = "aws eks update-kubeconfig ..."
```

### 7.4 配置 kubectl 连接到新建的集群

```bash
# 使用 Terraform 输出的命令
aws eks update-kubeconfig --region ap-northeast-1 --name eks-demo-tokyo

# 验证连接
kubectl get nodes
# NAME                             STATUS   ROLES    AGE   VERSION
# ip-10-0-1-xx.ec2.internal       Ready    <none>   5m    v1.30.x
# ip-10-0-2-xx.ec2.internal       Ready    <none>   5m    v1.30.x

# 测试——部署一个 Nginx
kubectl create deployment hello-eks --image=nginx:alpine --replicas=3
kubectl get pods
```

### 7.5 对比：eksctl vs Terraform

对照你 K8s 入门指南中的 Bonus 部分：

```
eksctl 创建集群：
  eksctl create cluster --name my-k8s-tokyo ...
  一行命令 → 自动创建 VPC + EKS + 节点组
  但：不能细粒度控制，不能和其他资源联动

Terraform 创建集群：
  可以精细控制每个组件
  可以和其他资源（RDS、IAM 策略、S3）整合
  可以版本管理、代码评审
  缺点：更啰嗦，需要了解更多的底层概念
```

### 7.6 清理

```bash
terraform destroy
```

> ⚠️ EKS 控制平面每小时的费用 (~$0.10/h) 在你跑 `terraform destroy` 前会持续产生，不像 Kind 那样免费，用完务必销毁。

### ✅ Day 7 学习成果检查

- [ ] 会用 Terraform 创建 VPC（使用 terraform-aws-modules/vpc/aws 模块）
- [ ] 会用 Terraform 创建 EKS 集群（使用 terraform-aws-modules/eks/aws 模块）
- [ ] 建立了 Terraform 和 K8s 的关联：Terraform 管基础设施，K8s 管应用
- [ ] 理解了 eksctl 和 Terraform 创建 EKS 的差异和适用场景

---

## Day 8：多环境管理与工作流

### 8.1 问题：如何管理多个环境？

```bash
# 简单场景：你有一个项目，需要放在不同环境
dev/       → 1 台 t3.micro，个人开发
staging/   → 2 台 t3.small，测试环境
prod/      → 5 台 t3.large，生产环境
```

核心问题：
- **代码如何复用？** 不想把同样配置写三遍
- **变量如何隔离？** 每个环境有不同的值
- **State 如何隔离？** dev 的变更不能影响 prod

### 8.2 方案一：目录结构分离（推荐）

```
environments/
├── dev/
│   ├── main.tf           # 调用 Module，传 dev 参数
│   ├── terraform.tfvars  # dev 变量值
│   └── backend.tf        # S3 路径：dev/terraform.tfstate
├── staging/
│   ├── main.tf
│   ├── terraform.tfvars
│   └── backend.tf        # S3 路径：staging/terraform.tfstate
└── prod/
    ├── main.tf
    ├── terraform.tfvars
    └── backend.tf        # S3 路径：prod/terraform.tfstate
```

每个环境的 `main.tf` 共享同一个模块，但传递不同参数：

```hcl
# environments/dev/main.tf
module "app" {
  source        = "../../modules/app"
  env           = "dev"
  instance_type = "t3.micro"
  replicas      = 1
}
```

```hcl
# environments/prod/main.tf
module "app" {
  source        = "../../modules/app"
  env           = "prod"
  instance_type = "t3.large"
  replicas      = 5
}
```

```hcl
# environments/dev/backend.tf
terraform {
  backend "s3" {
    bucket = "my-tfstate"
    key    = "dev/terraform.tfstate"    # ← 不同的路径！
    region = "ap-northeast-1"
    dynamodb_table = "terraform-lock"
  }
}
```

### 8.3 方案二：Workspace（适合较简单的场景）

```bash
# Terraform 内置的 workspace 机制
terraform workspace new dev
terraform workspace new staging
terraform workspace new prod

# 切换到某个 workspace
terraform workspace select dev

# 当前 workspace
terraform workspace show
```

在代码中获取当前 workspace：

```hcl
# 根据 workspace 不同，实例类型不同
locals {
  instance_type = {
    dev     = "t3.micro"
    staging = "t3.small"
    prod    = "t3.large"
  }[terraform.workspace]
}
```

> ⚠️ Workspace 适合简单场景，复杂多环境推荐用**目录结构**，因为目录结构更清晰、更不容易误操作（在 dev 目录下 apply 不会影响到 prod 的 state）。

### 8.4 安全实践

#### 敏感信息处理

```hcl
# ❌ 不要硬编码
variable "db_password" {
  default = "SuperSecret123!"    # ❌ 提交到 Git 了！
}

# ✅ 用环境变量传入
export TF_VAR_db_password=SuperSecret123!
terraform apply

# ✅ 或使用 AWS Secrets Manager
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "my-db-password"
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}
```

#### .gitignore

```gitignore
# .gitignore
.terraform/
*.tfstate
*.tfstate.*
crash.log
override.tf
terraform.rc
```

### 8.5 CI/CD 集成（GitOps 方式）

```
开发者 Push 代码到 Git
        │
        ▼
CI/CD 触发（GitHub Actions / GitLab CI）
        │
        ▼
terraform plan         ← 在 PR 里审查 plan 输出
        │
        ▼
人工审核后合并到 main
        │
        ▼
terraform apply        ← 自动执行
```

#### GitHub Actions 示例

```yaml
name: Terraform
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

jobs:
  terraform:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: hashicorp/setup-terraform@v3

      - name: Terraform Init
        run: terraform init
        working-directory: environments/${{ github.ref_name }}

      - name: Terraform Plan
        run: terraform plan
        working-directory: environments/${{ github.ref_name }}

      - name: Terraform Apply
        if: github.ref == 'refs/heads/main'
        run: terraform apply -auto-approve
        working-directory: environments/${{ github.ref_name }}
```

### ✅ Day 8 学习成果检查

- [ ] 理解多环境管理的两种方案（目录结构 vs Workspace）
- [ ] 理解如何复用 Module 来管理不同环境
- [ ] 知道如何处理敏感信息（环境变量 / Secrets Manager）
- [ ] 了解 Terraform + CI/CD 的基本工作流

---

## 附录：常用命令速查

### 基础工作流

```bash
# ───────── 初始化 ─────────
terraform init                           # 初始化（下载 provider）
terraform init -reconfigure              # 重新配置后端（解决 state 问题）

# ───────── 执行计划 ─────────
terraform plan                           # 预览变更
terraform plan -out=plan.tfplan          # 保存计划到文件
terraform plan -var="env=prod"           # 指定变量

# ───────── 应用 ─────────
terraform apply                          # 执行变更（需要确认）
terraform apply -auto-approve            # 自动确认（CI/CD 用）
terraform apply plan.tfplan              # 用已保存的计划执行

# ───────── 销毁 ─────────
terraform destroy                        # 销毁所有资源
terraform destroy -target=aws_instance.web  # 只销毁特定资源

# ───────── 状态 ─────────
terraform state list                     # 列出所有资源
terraform state show aws_instance.web    # 查看资源详情
terraform state rm aws_instance.web      # 从 state 移除（不真删）
terraform import aws_instance.web i-xxx  # 导入已有资源

# ───────── 查看 ─────────
terraform output                         # 列出所有输出
terraform show                           # 查看当前 state
terraform graph                          # 输出依赖图（可使用 graphviz 可视化）
```

### 核心概念速查表

| 概念 | 一句话 | 语法示例 |
|------|--------|---------|
| **Provider** | 告诉 Terraform 管哪个平台 | `aws = { source = "hashicorp/aws" }` |
| **Resource** | 要创建的具体资源 | `resource "aws_instance" "web" {}` |
| **Data Source** | 查询已有资源 | `data "aws_ami" "ubuntu" {}` |
| **Variable** | 输入参数 | `variable "name" { type = string }` |
| **Local** | 本地计算值 | `locals { name = format(...) }` |
| **Output** | 返回值 | `output "ip" { value = ... }` |
| **Module** | 可复用的封装 | `module "vpc" { source = "./modules/vpc" }` |
| **Backend** | State 存储位置 | `backend "s3" { bucket = "..." }` |

### 常见错误排查

```bash
# 错误：Backend 配置变更
terraform init -reconfigure

# 错误：State 锁定（有人正在 apply）
# 等对方完成，或强制解锁
terraform force-unlock <LOCK_ID>

# 错误：资源已存在
terraform import <resource_type>.<name> <id>

# 错误：Provider 版本不兼容
# 更新 provider 版本或锁定版本
```

---

## 🎉 结语

走到这里，你已经：

- ✅ 理解了 IaC 思想和 Terraform 声明式配置
- ✅ 掌握了 HCL 语法（resource、variable、data、output、local）
- ✅ 理解了 State 管理的重要性和远程后端配置
- ✅ 会写可复用的 Module
- ✅ 能用 Terraform 创建 AWS 基础设施（VPC + EC2 + RDS）
- ✅ 能用 Terraform 创建 EKS 集群（联动 K8s）
- ✅ 理解多环境管理的最佳实践

**下一步可以探索的方向：**
- 🔄 **Terragrunt**：Terraform 的 DRY 工具，减少重复代码
- 🔐 **Vault**：HashiCorp 的密钥管理工具（和 Terraform 同家）
- 🏗️ **Pulumi**：用通用编程语言（TypeScript/Python/Go）写 IaC
- 🚀 **Crossplane**：K8s 原生的 IaC 方案（直接在 K8s 里声明云资源）

---

> **遇到问题先查：** `terraform plan` 可以看到变更预览；`terraform state list` 看当前管的资源；`terraform validate` 检查语法；这三个解决 80% 的疑惑。

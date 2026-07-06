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
terraform {
  # required_providers 声明该项目需要哪些 provider 插件
  required_providers {
    local = {
      # source 指定 provider 的来源路径：命名空间/类型
      source  = "hashicorp/local"
      # version 用悲观约束 ~> 2.5，即 >= 2.5 且 < 3.0
      version = "~> 2.5"
    }
  }
}

# resource 块定义要创建的资源：类型是 local_file，名称是 hello
resource "local_file" "hello" {
  # content 指定文件的文本内容
  content  = "Hello Terraform!"
  # filename 指定文件在本地磁盘的路径，path.module 表示当前模块目录
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

> ⚠️ **`terraform destroy` 是不可逆操作**。它会删除所有当前 Terraform 项目管理的资源。在真实云环境中，这意味着服务器、数据库、存储都会被彻底删除，**无法恢复**。始终在操作前通过 `terraform plan` 确认要删除的内容，生产环境务必三思后行。

### 1.6 K8s vs Terraform 操作对比

| 操作 | K8s 命令 | Terraform 命令 |
|------|----------|---------------|
| 初始化 | — | `terraform init` |
| 预览 | `kubectl apply --dry-run=client` | `terraform plan` |
| 应用 | `kubectl apply -f` | `terraform apply` |
| 删除 | `kubectl delete -f` | `terraform destroy` |
| 查看状态 | `kubectl get` | `terraform show` |
| 查看当前配置 | `kubectl get -o yaml` | `terraform state list` |

### 📌 建立好习惯：每天都用 `fmt` + `validate`

从今天开始，把下面两个命令变成肌肉记忆：

```bash
# 1. 格式化代码（类似 prettier / gofmt）
terraform fmt
# 自动修正所有 .tf 文件的缩进、空格对齐，让团队代码风格统一

# 2. 检查语法（类似编译器的类型检查）
terraform validate
# 检查 HCL 语法、属性名称、类型是否匹配
# 能在 apply 之前发现大部分低级错误
```

> 💡 建议把这两个命令写入你的编辑器保存钩子或 pre-commit hook，这样每次修改代码后自动检查。

### ✅ Day 1 学习成果检查

- [ ] 理解了什么是 IaC 和声明式配置
- [ ] 安装了 Terraform
- [ ] 跑通了第一个例子（创建/查看/销毁本地文件）
- [ ] 理解了 Desired State vs Current State 的概念
- [ ] 理解了 `terraform init` / `plan` / `apply` / `destroy` / `fmt` / `validate` 的基本流程
- [ ] 养成了编写代码后先 `terraform fmt` 再 `terraform validate` 的习惯

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

### 2.5 `count` vs `for_each`——批量创建资源

Terraform 提供两种批量创建资源的机制，**适用场景完全不同**：

```hcl
# ──── count：按"编号"创建一组相似的资源 ────
# 适合：所有副本配置完全一样

variable "subnet_cidrs" {
  type    = list(string)
  default = ["10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"]
}

resource "aws_subnet" "by_count" {
  count      = length(var.subnet_cidrs)     # 创建 3 个子网
  vpc_id     = aws_vpc.main.id
  cidr_block = var.subnet_cidrs[count.index] # 用 count.index 取编号
}

# ──── for_each：按"键"管理一组不同的资源 ────
# 适合：每个副本的配置不同，需要通过"键"来引用

variable "subnet_configs" {
  type = map(object({
    cidr = string
    az   = string
  }))
  default = {
    "subnet-a" = { cidr = "10.0.1.0/24", az = "ap-northeast-1a" }
    "subnet-b" = { cidr = "10.0.2.0/24", az = "ap-northeast-1c" }
  }
}

resource "aws_subnet" "by_for_each" {
  for_each         = var.subnet_configs
  vpc_id           = aws_vpc.main.id
  cidr_block       = each.value.cidr          # each.value 取当前项的值
  availability_zone = each.value.az
  tags = {
    Name = each.key                            # each.key 取当前项的键
  }
}
```

**为什么这个区别很重要？**

| 场景 | 用 `count` | 用 `for_each` |
|------|-----------|--------------|
| 列表中间插了一个元素 | 所有后续资源的 `count.index` 变化 → 会触发更新/重建 ❌ | 键不变 → 不受影响 ✅ |
| 删除列表中某个元素 | 需要手动 `terraform state rm` 移除旧的 | 自动移除对应的键 |
| 从代码中引用某个资源 | `aws_subnet.by_count[1]`（下标脆弱） | `aws_subnet.by_for_each["subnet-b"]`（键稳定） |

> 💡 **经验法则**：如果资源列表可能会增删改（大多数真实场景），优先用 `for_each`。只在你确定列表永远不变的场景下用 `count`。

### 2.6 常用内置函数

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
      # source 指定 provider 来源：hashicorp 官方维护的 random provider
      source  = "hashicorp/random"
      # version 悲观约束 ~> 3.6，表示 >= 3.6 且 < 4.0
      version = "~> 3.6"
    }
  }
}

# resource "random_string" 生成一个随机字符串，常用于创建唯一标识符
resource "random_string" "suffix" {
  length  = 6          # 生成长的随机字符串
  special = false      # 不含特殊字符
  upper   = false      # 只含小写字母
}

# resource "local_file" 创建一个配置文件，内容引用随机字符串和变量
resource "local_file" "config" {
  # content 使用 heredoc 语法（<<-EOF）编写多行文本
  content = <<-EOF
    server_name=web-${random_string.suffix.result}
    environment=${var.env}
    log_level=${local.log_level}
  EOF
  # filename 也包含随机后缀，每次 apply 都会生成不同的文件名
  filename = "${path.module}/config-${random_string.suffix.result}.txt"
}

# variable 定义输入参数 env，调用方可以覆盖默认值
variable "env" {
  type    = string     # 类型为字符串
  default = "dev"      # 默认值 dev，也可通过 tfvars / -var 覆盖
}

# locals 定义本地计算值，仅在当前模块内可见
locals {
  # 条件表达式：如果是 prod 环境用 warn，否则用 debug
  log_level = var.env == "prod" ? "warn" : "debug"
}

# output 暴露执行结果给调用方
output "created_file" {
  value       = local_file.config.filename   # 输出生成的文件路径
  description = "刚刚创建的文件路径"                # 可读说明
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
variable "name"       { type = string }       # 字符串类型
variable "count"      { type = number }       # 数字类型
variable "enabled"    { type = bool }         # 布尔类型（true / false）
variable "tags"       { type = map(string) }  # 字典/映射类型（键=字符串，值=字符串）
variable "azs"        { type = list(string) } # 字符串列表
variable "instance"   { type = object({       # 对象类型（包含多个命名字段）
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
# output 块暴露 Terraform 执行后的结果给用户或其他模块使用
output "instance_ip" {
  # value 指定要暴露的属性值，这里引用 EC2 实例的公网 IP
  value       = aws_instance.web.public_ip
  # description 提供可读说明，terraform output 时会显示
  description = "Web 服务器的公网 IP"
  # sensitive 标记是否敏感：true 时日志中隐藏值，但 state 仍存明文
  sensitive   = false
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
# data 块声明数据源：查询当前 AWS 账号信息，无需传参
data "aws_caller_identity" "current" {}

# data 块查询最新 Ubuntu 24.04 AMI 镜像 ID
data "aws_ami" "ubuntu" {
  # most_recent = true 表示取符合条件的最新版本
  most_recent = true
  # filter 按"名称"筛选：匹配 ubuntu 24.04 的 HVM SSD 镜像
  filter {
    name   = "name"
    values = ["ubuntu/images/hvm-ssd/ubuntu-24.04-*"]
  }
  # filter 按虚拟化类型筛选：只取 HVM（硬件辅助虚拟化）
  filter {
    name   = "virtualization-type"
    values = ["hvm"]
  }
  # owners 指定 AMI 拥有者：099720109477 是 Canonical（Ubuntu 官方）的 AWS 账号
  owners = ["099720109477"]
}

# resource 块使用数据源查询到的值创建资源
resource "aws_instance" "web" {
  # data.aws_ami.ubuntu.id 引用数据源返回的 AMI ID——不硬编码镜像 ID
  ami           = data.aws_ami.ubuntu.id
  instance_type = "t3.micro"
}

# output 暴露数据源查询到的 AWS 账号 ID
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
# locals 块定义模块内部的本地计算值，不可从外部覆盖
locals {
  # format 函数用于字符串模板拼接，等价于 "${var.project}-${var.env}"
  name = format("%s-%s", var.project, var.env)
  
  # 条件表达式（三目运算）：生产环境用 t3.large，否则用 t3.micro
  instance_type = var.env == "prod" ? "t3.large" : "t3.micro"
  replicas      = var.env == "prod" ? 3 : 1
  
  # merge 函数合并多个 map：将自定义标签 Name 添加到默认标签中
  tags = merge(var.default_tags, {
    Name = local.name
  })
}

# 在 resource 中引用 locals 的值
resource "aws_instance" "web" {
  # local.instance_type 引用上面定义的本地值
  instance_type = local.instance_type
  # local.tags 引用合并后的完整标签集
  tags          = local.tags
}
```

> `locals` 和 `var` 的区别：
> - `var` = 外部输入的参数（用户赋值）
> - `local` = 内部计算的中间值（由其他变量计算得来）

### 3.5 综合实操演练

把前面学到的变量、输出、数据源、本地值全部放到一个完整的例子里，**不花钱就能跑通**。

---

#### 📁 文件结构

```
~/terraform-learning/
├── day3-demo.tf          # 主配置（复制下面的完整代码）
├── terraform.tfvars      # 变量值文件
└── terraform.tfstate     # 自动生成（State 文件）
```

---

#### 📄 创建 `day3-demo.tf`

```hcl
# ============================================================
# Day 3 实操练习：变量 + 输出 + 数据源 + 本地值
# ============================================================
# 练习目标：
#   1. 定义多种类型的变量（string / number / bool / list / map / object）
#   2. 使用变量校验（validation）
#   3. 使用本地值（locals）计算中间值
#   4. 使用数据源（data）读取已有资源
#   5. 使用输出（output）暴露结果
# ============================================================

terraform {
  required_providers {
    local = {
      source  = "hashicorp/local"
      version = "~> 2.5"
    }
    random = {
      source  = "hashicorp/random"
      version = "~> 3.6"
    }
  }
}

# ─────────────────────────────────────
# 1. 变量（Variable）—— 定义输入参数
# ─────────────────────────────────────

variable "project_name" {
  description = "项目名称"
  type        = string
  default     = "terraform-day3"
}

variable "environment" {
  description = "运行环境"
  type        = string
  # ⚠️ 注意：没有 default，apply 时必须赋值，否则 Terraform 会提示输入

  validation {
    condition     = contains(["dev", "staging", "prod"], var.environment)
    error_message = "环境必须是 dev、staging 或 prod 之一。"
  }
}

variable "replicas" {
  description = "实例副本数"
  type        = number
  default     = 1
}

variable "enable_monitoring" {
  description = "是否启用监控"
  type        = bool
  default     = true
}

variable "tags" {
  description = "资源标签"
  type        = map(string)
  default = {
    Owner = "terraform-learner"
  }
}

variable "availability_zones" {
  description = "可用区列表"
  type        = list(string)
  default     = ["ap-northeast-1a", "ap-northeast-1c"]
}

variable "instance_config" {
  description = "实例配置"
  type = object({
    size = string
    disk = number
  })
  default = {
    size = "t3.micro"
    disk = 20
  }
}

# ─────────────────────────────────────
# 2. 本地值（Local）—— 计算中间值
# ─────────────────────────────────────

locals {
  # string 拼接
  full_name = "${var.project_name}-${var.environment}"

  # 条件表达式（三目运算）
  log_level = var.environment == "prod" ? "warn" : "debug"

  # 数字运算：prod 环境副本数翻倍
  replica_count = var.environment == "prod" ? var.replicas * 2 : var.replicas

  # 合并 map
  all_tags = merge(var.tags, {
    Name        = local.full_name
    Environment = var.environment
    Monitoring  = var.enable_monitoring ? "on" : "off"
  })

  # for 表达式遍历列表
  formatted_zones = [for az in var.availability_zones : upper(az)]
}

# ─────────────────────────────────────
# 3. 资源（Resource）—— 创建资源
# ─────────────────────────────────────

resource "random_string" "suffix" {
  length  = 8
  special = false
  upper   = false
}

resource "local_file" "config" {
  content = <<-EOF
# ${local.full_name} 配置文件
project=${var.project_name}
env=${var.environment}
log_level=${local.log_level}
replicas=${local.replica_count}
instance_size=${var.instance_config.size}
disk_size=${var.instance_config.disk}
monitoring=${var.enable_monitoring}
zones=${join(", ", local.formatted_zones)}
suffix=${random_string.suffix.result}
EOF
  filename = "${path.module}/${local.full_name}.conf"
}

resource "local_file" "tags" {
  content  = jsonencode(local.all_tags)
  filename = "${path.module}/${local.full_name}-tags.json"
}

# ─────────────────────────────────────
# 4. 数据源（Data）—— 读取已有资源
# ─────────────────────────────────────

data "local_file" "config_content" {
  filename = local_file.config.filename
}

# ─────────────────────────────────────
# 5. 输出（Output）—— 暴露结果
# ─────────────────────────────────────

output "generated_config" {
  value       = local_file.config.filename
  description = "生成的配置文件路径"
}

output "config_file_content" {
  value       = data.local_file.config_content.content
  description = "配置文件内容"
}

output "final_tags" {
  value       = local.all_tags
  description = "最终合并后的标签"
}

output "replica_count" {
  value       = local.replica_count
  description = "根据环境计算的副本数"
}

output "formatted_availability_zones" {
  value       = local.formatted_zones
  description = "大写格式化后的可用区"
}

output "db_connection_string" {
  value       = "postgresql://admin:${random_string.suffix.result}@${local.full_name}.rds.amazonaws.com:5432/appdb"
  description = "数据库连接字符串"
  sensitive   = true
}
```

---

#### 📄 创建 `terraform.tfvars`

```hcl
# ─────────────────────────────────────
# 变量值文件：覆盖变量的默认值
# ─────────────────────────────────────
environment = "staging"
replicas    = 2
tags = {
  Owner   = "terraform-learner"
  Team    = "platform"
}

# 你也可以试试改成这些值再跑一次：
# environment = "prod"
# replicas    = 3
```

> 💡 `environment` 变量没有设 `default`，所以必须通过 `terraform.tfvars`（或 `-var` / 环境变量）提供值，否则 Terraform 会在 `apply` 时交互式提示输入——这是故意设计的，让你体验变量必须赋值的场景。

---

#### ▶️ 运行步骤

```bash
cd ~/terraform-learning

# 如果之前 Day 2 的资源还在，先清理
terraform destroy

# 1️⃣ 初始化（下载 provider 插件）
terraform init

# 2️⃣ 预览执行计划
terraform plan

# 3️⃣ 执行（会自动读取 terraform.tfvars）
terraform apply
# 输入 yes

# 4️⃣ 查看生成的文件
ls *.conf *.json
cat terraform-day3-staging.conf
cat terraform-day3-staging-tags.json

# 5️⃣ 查看所有输出
terraform output

# 6️⃣ 查看单个输出
terraform output replica_count

# 7️⃣ sensitive 值不会直接显示
terraform output db_connection_string
# 输出：╷
#       │ Warning: Output refers to sensitive value
#       │ (value is hidden unless you run with -json)

# 想看也得加 -json：
terraform output -json db_connection_string

# 8️⃣ 查看数据源读到的内容
terraform output config_file_content
```

---

#### 🔄 试一下变量覆盖

不修改文件，直接通过命令行覆盖变量：

```bash
# 用 prod 配置重新 apply
terraform apply -var="environment=prod" -var="replicas=3"

# 观察变化：
#   - log_level 从 debug → warn
#   - replica_count 从 2 → 6（翻倍）
#   - 文件名从 staging → prod
#   - 生成了新的 .conf 和 .json 文件

# 查看生成的新文件
cat terraform-day3-prod.conf
```

也可以用环境变量：

```bash
export TF_VAR_environment=prod
terraform plan     # 此时用的是 prod
unset TF_VAR_environment
```

---

#### 🧹 清理

```bash
terraform destroy
# 或直接删文件
rm -f terraform-day3-*.conf terraform-day3-*.json
```

---

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
  "resources": [                            # 所有被管理的资源列表
    {
      "type": "aws_instance",               # 资源类型：EC2 实例
      "name": "web",                        # 资源在代码中的命名（local name）
      "instances": [                        # 该资源的实例列表（count/for_each 创建多个）
        {
          "attributes": {                   # 资源的实际属性值，与云平台真实状态一致
            "id": "i-0a1b2c3d4e5f",        # AWS 分配的唯一实例 ID
            "ami": "ami-xxx",               # 创建时使用的 AMI 镜像 ID
            "public_ip": "54.123.45.67"     # AWS 自动分配的公网 IP
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
    bucket         = "my-terraform-state"           # S3 桶（需提前创建）
    key            = "prod/terraform.tfstate"       # 路径（区分环境）
    region         = "ap-northeast-1"               # 区域
    encrypt        = true                           # 加密
    use_lockfile   = true                           # 使用文件锁（替代已废弃的 dynamodb_table）
  }
}
```

> ⚠️ **先有鸡还是先有蛋？** S3 桶需要**提前手动创建**——Terraform 不能自己创建自己存 state 的桶。
> 只需要初始化时做一次：

```bash
# 在配置 backend.tf 之前，先手动创建 S3 桶（桶名需全局唯一）
aws s3 mb s3://你的名字-terraform-state --region ap-northeast-1

# 启用版本控制（防止误删/误改 state）
aws s3api put-bucket-versioning \
  --bucket 你的名字-terraform-state \
  --versioning-configuration Status=Enabled

# 然后在 backend.tf 里填上你的桶名
```

这步也叫 **bootstrap（引导初始化）**——整个项目中**只需要手动做一次**，之后 Terraform 的所有操作都自动读写这个桶。

> 💡 S3 桶负责"存文件"，`use_lockfile = true` 负责"上锁"——多人同时 apply 时，只有一个人能操作，其他人会等锁释放。`dynamodb_table` 参数在 AWS provider v6 中已废弃，改为 `use_lockfile`。

> ⚠️ **State 文件安全警告**：`terraform.tfstate` 中**可能包含明文敏感信息**——数据库密码、IAM 密钥、私钥、连接字符串等。如果你用了 `sensitive = true` 标记输出，Terraform 会在日志中隐藏它，但 state 文件里仍然是明文。因此：
> - ✅ 对 S3 后端**启用存储桶版本控制**（`aws s3api put-bucket-versioning`），意外删改 state 时可恢复
> - ✅ 使用 **S3 桶策略限制访问**——只有需要的人能读 state
> - ✅ 生产环境考虑用 **Terraform Cloud / Enterprise**，其 state 始终加密存储且支持审计
> - ❌ 不要将 `terraform.tfstate` 提交到 Git
> 
> 后续 Day 8 会讲如何通过 Vault 等工具避免敏感信息进入 state。

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
# variable 定义模块的输入参数：VPC 的 CIDR 地址段，调用方必须传入
variable "vpc_cidr" {
  type = string
}

# variable 定义环境名称（如 dev/prod），用于资源命名和标签隔离
variable "env" {
  type = string
}

# resource 创建 VPC（虚拟私有网络），这是网络层的基础
resource "aws_vpc" "main" {
  # cidr_block 指定 VPC 的 IP 地址范围，如 "10.0.0.0/16"
  cidr_block           = var.vpc_cidr
  # enable_dns_support 启用 DNS 解析（默认就是 true，显式写出更清晰）
  enable_dns_support   = true
  # enable_dns_hostnames 启用 DNS 主机名（为 EC2 分配 DNS 名称）
  enable_dns_hostnames = true

  # tags 给资源打标签，便于在 AWS 控制台识别和成本分配
  tags = {
    Name = "${var.env}-vpc"
    Env  = var.env
  }
}

# resource 创建两个公有子网，用于放置需要公网访问的资源
resource "aws_subnet" "public" {
  count             = 2                                                     # 创建 2 个子网，分布在不同的可用区
  vpc_id            = aws_vpc.main.id                                       # 关联到上面创建的 VPC
  cidr_block        = cidrsubnet(var.vpc_cidr, 8, count.index)             # cidrsubnet 自动划分 CIDR 段
  availability_zone = data.aws_availability_zones.available.names[count.index]  # 轮询分配到各个可用区

  # tags 用 count.index 区分不同子网
  tags = {
    Name = "${var.env}-public-${count.index}"
    Env  = var.env
  }
}

# data 块查询当前区域有哪些可用区（AZ），供子网创建时使用
data "aws_availability_zones" "available" {
  state = "available"       # 只查询状态为 available 的可用区
}

# output 暴露 VPC ID，供调用模块的一方使用
output "vpc_id" {
  value = aws_vpc.main.id
}

# output 暴露所有公有子网的 ID 列表（[*] 是 splat 表达式，提取所有实例的 id 属性）
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
      source  = "hashicorp/aws"   # module 本身不声明 provider，但调用方需要
      version = "~> 5.0"
    }
  }
}

provider "aws" {
  region = "ap-northeast-1"       # 模块内资源将创建在此区域
}

# module 块调用自定义模块：source 使用本地路径引用
module "networking" {
  # source 指向模块目录，支持本地路径或 Registry URL
  source   = "../../modules/networking"
  # vpc_cidr 传给模块的 variable "vpc_cidr"
  vpc_cidr = "10.0.0.0/16"
  # env 传给模块的 variable "env"
  env      = "dev"
}

# output 引用模块的输出值，供调用方或其他模块使用
output "created_vpc_id" {
  value = module.networking.vpc_id   # 模块的 output "vpc_id" 暴露的值
}
```

### 5.4 使用 Terraform Registry 的公共模块

别人写好的模块，直接拿来用：

```hcl
# module 块引用 Terraform Registry 上的公共模块，开箱即用
module "vpc" {
  # source 格式：命名空间/模块名/提供商（terraform-aws-modules/vpc/aws）
  source = "terraform-aws-modules/vpc/aws"
  # version 指定模块版本，与 provider 一样用悲观约束
  version = "~> 5.0"

  name = "my-vpc"               # VPC 名称标签
  cidr = "10.0.0.0/16"          # VPC 的 CIDR 地址段

  azs             = ["ap-northeast-1a", "ap-northeast-1c"]   # 使用的可用区列表
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]           # 私有子网 CIDR
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]       # 公有子网 CIDR

  enable_nat_gateway = true      # 自动创建 NAT 网关（私有子网访问公网用）
  enable_vpn_gateway = false     # 不创建 VPN 网关

  tags = {
    Environment = "dev"
  }
}
```

> 💡 Terraform Registry（registry.terraform.io）就像 Docker Hub——上面有官方和社区维护的各类模块。

### 5.5 `lifecycle` 元参数——控制资源创建/销毁行为

`lifecycle` 是你生产环境中**必须掌握**的元参数，用于控制 Terraform 在变更资源时的行为：

```hcl
resource "aws_instance" "web" {
  ami           = "ami-xxx"
  instance_type = "t3.micro"

  lifecycle {
    # ─── prevent_destroy：防止误删 ───
    # 设置后，任何试图 destroy 这个资源的 plan 都会直接报错
    # 适用于数据库、生产环境核心资源
    prevent_destroy = true

    # ─── create_before_destroy：先建后删 ───
    # 资源更新时，先创建新的再删除旧的，实现零停机
    # 适用于 ALB、安全组等需要不停机的资源
    create_before_destroy = true

    # ─── ignore_changes：忽略特定属性的变化 ───
    # 资源创建后，某些属性如果被外部修改（如 AWS 控制台手动调整），
    # Terraform 不会尝试改回来
    ignore_changes = [
      ami,              # AMI 版本可能被外部自动更新
      user_data,        # 启动脚本可能被运维脚本修改
      tags,             # 标签可能被其他工具管理
    ]
  }
}
```

**三个 `lifecycle` 规则的典型用法：**

| 规则 | 作用 | 典型场景 |
|------|------|----------|
| `prevent_destroy = true` | 阻止删除 | 生产数据库、有状态服务 |
| `create_before_destroy = true` | 先新建后销毁（零停机更新） | 负载均衡器、安全组 |
| `ignore_changes = [...]` | 忽略外部对某些属性的修改 | AMI 自动更新、外部标签管理 |

> ⚠️ `prevent_destroy` 只能阻止 `terraform destroy` 和 `terraform apply` 中删除该资源的操作，但如果你手动改了代码中该资源的名称再 apply，Terraform 还是会试图删除旧资源创建新资源——这时 `create_before_destroy` 可以保护你不中断服务。

### ✅ Day 5 学习成果检查

- [ ] 理解模块化的必要性和目录结构规范
- [ ] 会创建自己的模块（source 用本地路径）
- [ ] 会使用 `source` 引用本地和 Registry 上的模块
- [ ] 理解模块的输入（variables）和输出（outputs）
- [ ] 掌握了 `lifecycle` 的三个元参数（prevent_destroy / create_before_destroy / ignore_changes）的用法

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
  required_version = "~> 1.9"          # 建议用悲观约束：>= 1.9, < 2.0，防止大版本升级导致意外
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
  cidr_block           = "10.0.0.0/16"     # VPC 的 IP 地址段，最多 65536 个 IP
  enable_dns_support   = true               # 启用 DNS 解析
  enable_dns_hostnames = true               # 为 EC2 自动分配 DNS 名称

  tags = {
    Name = "${var.project}-${var.env}-vpc"
  }
}

# ────────────────────────────────────────────
# 2. 子网（2 个公有子网，2 个私有子网）
# ────────────────────────────────────────────
# 查询当前区域有哪些可用区
data "aws_availability_zones" "available" {
  state = "available"                        # 只取正常运行的可用区
}

# 公有子网（放 Web 服务器、负载均衡）
resource "aws_subnet" "public" {
  count = 2                                  # 创建 2 个，分布在不同的可用区

  vpc_id            = aws_vpc.main.id        # 关联到上面创建的 VPC
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index)  # 自动划分子网段
  availability_zone = data.aws_availability_zones.available.names[count.index]  # 轮询分配可用区

  map_public_ip_on_launch = true             # 在该子网创建的 EC2 自动分配公网 IP

  tags = {
    Name = "${var.project}-${var.env}-public-${count.index}"
  }
}

# 私有子网（放数据库、内部服务）
resource "aws_subnet" "private" {
  count = 2                                  # 创建 2 个私有子网

  vpc_id            = aws_vpc.main.id
  cidr_block        = cidrsubnet(aws_vpc.main.cidr_block, 8, count.index + 2)  # 偏移 2 避免与公有子网 CIDR 冲突
  availability_zone = data.aws_availability_zones.available.names[count.index]

  tags = {
    Name = "${var.project}-${var.env}-private-${count.index}"
  }
}

# ────────────────────────────────────────────
# 3. 互联网网关（让 VPC 能访问公网）
# ────────────────────────────────────────────
resource "aws_internet_gateway" "main" {
  vpc_id = aws_vpc.main.id                    # 挂载到 VPC，VPC 内的子网才能通过它访问公网

  tags = {
    Name = "${var.project}-${var.env}-igw"
  }
}

# ────────────────────────────────────────────
# 4. 路由表（公有子网连 IGW，私有子网连 NAT）
# ────────────────────────────────────────────
resource "aws_route_table" "public" {
  vpc_id = aws_vpc.main.id

  # route 块定义路由规则：所有流量（0.0.0.0/0）走互联网网关
  route {
    cidr_block = "0.0.0.0/0"                 # 目标网段：0.0.0.0/0 表示所有公网地址
    gateway_id = aws_internet_gateway.main.id  # 下一跳：互联网网关
  }

  tags = {
    Name = "${var.project}-${var.env}-public-rt"
  }
}

# 将公有路由表关联到每个公有子网
resource "aws_route_table_association" "public" {
  count          = 2                          # 2 个公有子网各关联一次
  subnet_id      = aws_subnet.public[count.index].id
  route_table_id = aws_route_table.public.id
}

# ────────────────────────────────────────────
# 5. 安全组（防火墙规则）
# ────────────────────────────────────────────

# Web 服务器安全组（开放 80 和 443，⚠️ 生产环境不应开放 SSH 22 端口到全互联网）
# 生产环境最佳实践：EC2 放在私有子网，通过 ALB/NLB 暴露服务
# SSH 管理应通过 AWS Systems Manager Session Manager 或 VPN + 堡垒机
resource "aws_security_group" "web" {
  name        = "${var.project}-${var.env}-web-sg"
  description = "Allow HTTP/HTTPS"
  vpc_id      = aws_vpc.main.id               # 安全组属于哪个 VPC

  # ingress 块定义入站规则：允许来自任意 IP 的 HTTP 流量
  ingress {
    description = "HTTP"
    from_port   = 80
    to_port     = 80
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]              # 允许所有来源 IP
  }

  # ingress 定义 HTTPS 入站规则
  ingress {
    description = "HTTPS"
    from_port   = 443
    to_port     = 443
    protocol    = "tcp"
    cidr_blocks = ["0.0.0.0/0"]
  }

  # egress 块定义出站规则：允许所有出站流量（-1 表示所有协议）
  egress {
    from_port   = 0
    to_port     = 0
    protocol    = "-1"                        # -1 表示所有协议
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

  # ingress 只允许 Web 安全组所在的资源访问 MySQL 端口
  ingress {
    description     = "MySQL"
    from_port       = 3306
    to_port         = 3306
    protocol        = "tcp"
    security_groups = [aws_security_group.web.id]   # 通过安全组 ID 引用，而非开放给全网
  }

  tags = {
    Name = "${var.project}-${var.env}-db-sg"
  }
}

# ────────────────────────────────────────────
# 6. EC2 实例（Web 服务器）
# ────────────────────────────────────────────

# 查找最新的 Amazon Linux 2023 AMI（⚠️ Amazon Linux 2 已于 2025 年结束标准支持，新项目请用 AL2023）
data "aws_ami" "amazon_linux" {
  most_recent = true                             # 取最新版本
  owners      = ["amazon"]                       # 限定 AWS 官方提供的 AMI

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]             # 通配匹配 AL2023 的 x86 架构镜像名
  }
}

resource "aws_instance" "web" {
  # ami 引用数据源查询到的 AMI ID，不硬编码具体值
  ami                    = data.aws_ami.amazon_linux.id
  instance_type          = "t3.micro"            # 实例规格
  subnet_id              = aws_subnet.public[0].id  # 放到第一个公有子网中
  vpc_security_group_ids = [aws_security_group.web.id]  # 关联 Web 安全组
  associate_public_ip_address = true             # 自动分配公网 IP

  # user_data 是实例启动时执行的脚本（首次启动运行一次）
  user_data = <<-EOF
    #!/bin/bash
    # Amazon Linux 2023 已用 dnf 替代 yum
    dnf update -y                                # 更新系统包
    dnf install -y httpd                         # 安装 Apache Web 服务器
    systemctl start httpd                        # 启动 Apache 服务
    systemctl enable httpd                       # 设置开机自启
    echo "<h1>Hello from Terraform!</h1>" > /var/www/html/index.html  # 写入测试页面
  EOF

  tags = {
    Name = "${var.project}-${var.env}-web-server"
  }
}

# ────────────────────────────────────────────
# 7. RDS 数据库（MySQL）
# ────────────────────────────────────────────

# 数据库子网组：RDS 需要关联到至少两个私有子网实现高可用
resource "aws_db_subnet_group" "main" {
  name       = "${var.project}-${var.env}-db-subnet-group"
  subnet_ids = aws_subnet.private[*].id          # 引用上面创建的所有私有子网

  tags = {
    Name = "${var.project}-${var.env}-db-subnet-group"
  }
}

resource "aws_db_instance" "main" {
  identifier = "${var.project}-${var.env}-mysql"  # RDS 实例标识符，在 AWS 控制台中显示

  engine         = "mysql"
  engine_version = "8.0"                          # ⚠️ 引擎版本会随时间更新，apply 前用 aws cli 确认最新版
  instance_class = "db.t4g.micro"                 # ⚠️ RDS 没有 db.t3.micro！db.t4g.micro 是最小免费规格

  db_name  = "appdb"                              # 数据库名称（连接时用）
  username = "admin"                              # 数据库管理员用户名
  password = var.db_password                      # 密码从变量传入，避免硬编码

  db_subnet_group_name   = aws_db_subnet_group.main.name   # 关联到上面定义的子网组
  vpc_security_group_ids = [aws_security_group.db.id]      # 关联数据库安全组

  skip_final_snapshot  = true                     # 删除时跳过创建最终快照（学习环境避免残留）
  publicly_accessible  = false                    # 不开放公网访问，仅 VPC 内部可连接
  allocated_storage    = 20                       # 分配 20GB 存储空间

  tags = {
    Name = "${var.project}-${var.env}-mysql"
  }
}
```

创建 `outputs.tf`：

```hcl
# 输出 VPC 的 ID，供其他 Terraform 项目或模块引用
output "vpc_id" {
  description = "VPC ID"
  value       = aws_vpc.main.id
}

# 输出 Web 服务器的公网 IP，方便直接 SSH 或浏览器访问
output "web_server_ip" {
  description = "Web 服务器公网 IP"
  value       = aws_instance.web.public_ip
}

# 输出数据库连接地址，供应用配置使用（标记 sensitive 防止日志泄露）
output "db_endpoint" {
  description = "数据库连接地址"
  value       = aws_db_instance.main.endpoint
  sensitive   = true                    # true 表示在控制台输出中隐藏具体值
}

# 输出可直接访问的 URL 链接（拼接 IP 生成）
output "web_server_url" {
  value = "http://${aws_instance.web.public_ip}"
}
```

创建 `terraform.tfvars`：

```hcl
region      = "ap-northeast-1"        # AWS 区域：东京
project     = "terraform-demo"        # 项目名称，用于资源命名前缀
env         = "dev"                   # 环境标识，用于资源隔离
db_password = "SafePassword123!"      # ⚠️ 真实项目不要硬编码！应使用环境变量或 Secrets Manager
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
  required_version = "~> 1.9"          # 建议用悲观约束：>= 1.9, < 2.0，防止大版本升级导致意外
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
# 查询当前区域有哪些可用区，用于后续分配子网
data "aws_availability_zones" "available" {}

# 使用 Terraform Registry 上的 VPC 模块快速创建生产级网络
module "vpc" {
  source  = "terraform-aws-modules/vpc/aws"    # Registry 路径
  version = "~> 5.0"

  name = "eks-demo-vpc"                        # VPC 名称标签
  cidr = "10.0.0.0/16"                         # VPC 地址段

  azs             = slice(data.aws_availability_zones.available.names, 0, 2)   # 取前 2 个可用区
  private_subnets = ["10.0.1.0/24", "10.0.2.0/24"]                            # 私有子网
  public_subnets  = ["10.0.101.0/24", "10.0.102.0/24"]                        # 公有子网

  enable_nat_gateway   = true                  # 创建 NAT 网关（EKS 节点需要访问 ECR/ECR）
  enable_dns_hostnames = true                  # 启用 DNS 主机名

  # EKS 需要的标签：公有子网标记为 LoadBalancer 角色
  public_subnet_tags = {
    "kubernetes.io/role/elb" = "1"
  }

  # EKS 需要的标签：私有子网标记为 Internal ELB 角色
  private_subnet_tags = {
    "kubernetes.io/role/internal-elb" = "1"
  }
}

# ────────────────────────────────────────────
# 2. EKS 集群
# ────────────────────────────────────────────
# 使用 Terraform Registry 上的 EKS 模块创建托管 K8s 集群
module "eks" {
  source  = "terraform-aws-modules/eks/aws"    # Registry 路径
  version = "~> 20.0"

  cluster_name    = "eks-demo-tokyo"            # 集群名称，后续 kubectl 连接用
  cluster_version = "1.30"                      # K8s 版本

  cluster_endpoint_public_access = true          # 允许从公网访问 API Server

  # vpc_id 和 subnet_ids 引用上面 VPC 模块的输出
  vpc_id     = module.vpc.vpc_id
  # EKS 控制平面部署在私有子网中
  subnet_ids = module.vpc.private_subnets

  # eks_managed_node_groups 定义托管节点组
  eks_managed_node_groups = {
    main = {                                     # 节点组名称
      desired_size = 2                           # 期望节点数（初始运行 2 个）
      min_size     = 1                           # 最小节点数（缩容不低于 1）
      max_size     = 4                           # 最大节点数（扩容不超过 4）

      instance_types = ["t3.medium"]             # 节点实例规格

      tags = {
        Role = "worker"                          # 标记为 Worker 节点
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
│   └── backend.tf        # S3 路径：dev/terraform.tfstate
├── staging/
│   ├── main.tf
│   └── backend.tf        # S3 路径：staging/terraform.tfstate
└── prod/
    ├── main.tf
    └── backend.tf        # S3 路径：prod/terraform.tfstate

modules/                  # ← 各环境共享的模块
├── networking/           # ← Day 5 创建的 VPC + 子网模块，此处复用
│   ├── main.tf
│   ├── variables.tf
│   └── outputs.tf
└── app/
    ├── main.tf           # EC2 + 安全组 + ALB
    ├── variables.tf      # 输入参数
    └── outputs.tf        # 输出值
```

两个模块都在 `modules/` 目录下，各环境共享同一套模板。

#### 📁 `modules/networking/` — 网络层模块

这个模块在 [Day 5](#52-创建一个简单的模块) 中已经详细写过，此处按标准结构拆为三个文件：

**`modules/networking/variables.tf`**：

```hcl
variable "vpc_cidr" {
  description = "VPC 的 CIDR 地址段，如 10.0.0.0/16"
  type        = string
}

variable "env" {
  description = "环境名称（dev / staging / prod），用于资源命名和标签隔离"
  type        = string
}
```

**`modules/networking/main.tf`**：

```hcl
# 查询当前区域有哪些可用区
data "aws_availability_zones" "available" {
  state = "available"
}

# 创建 VPC
resource "aws_vpc" "main" {
  cidr_block           = var.vpc_cidr
  enable_dns_support   = true
  enable_dns_hostnames = true

  tags = {
    Name = "${var.env}-vpc"
    Env  = var.env
  }
}

# 创建两个公有子网（分布在不同的可用区）
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
```

**`modules/networking/outputs.tf`**：

```hcl
output "vpc_id" {
  description = "VPC ID，供 app 模块引用"
  value       = aws_vpc.main.id
}

output "subnet_ids" {
  description = "公有子网 ID 列表，供 app 模块挂载 ALB 和 EC2"
  value       = aws_subnet.public[*].id
}
```

#### 📁 `modules/app/` — 应用层模块

先来看这个共享模块的定义：

创建 `~/terraform-learning/modules/app/variables.tf`：

```hcl
# modules/app/variables.tf
# 定义这个模块需要外部传入哪些参数

variable "env" {
  description = "环境名称（dev / staging / prod），用于命名和标签隔离"
  type        = string
}

variable "instance_type" {
  description = "EC2 实例规格，不同环境可以用不同规格"
  type        = string
}

variable "replicas" {
  description = "EC2 实例数量，生产环境通常比开发环境多"
  type        = number
}

variable "vpc_id" {
  description = "目标 VPC ID，由调用方传入（引用 networking 模块的输出）"
  type        = string
}

variable "public_subnet_ids" {
  description = "公有子网 ID 列表，用于挂载 ALB 和 EC2"
  type        = list(string)
}
```

创建 `~/terraform-learning/modules/app/main.tf`：

```hcl
# modules/app/main.tf
# 定义一个"应用"模块：安全组 + EC2 实例 + 负载均衡
# 不同环境通过传递不同参数来复用这个模板

# ────────────────────────────────────────────
# 1. 安全组：允许 HTTP（80）入站
# ────────────────────────────────────────────
resource "aws_security_group" "web" {
  name        = "${var.env}-web-sg"          # 安全组名称，按环境区分
  description = "Allow HTTP inbound traffic"
  vpc_id      = var.vpc_id                   # 模块输入：目标 VPC

  ingress {
    description = "HTTP from anywhere"       # 入站规则说明
    from_port   = 80                         # 起始端口
    to_port     = 80                         # 结束端口（80-80 即只开 80 端口）
    protocol    = "tcp"                      # TCP 协议
    cidr_blocks = ["0.0.0.0/0"]             # 允许所有来源 IP
  }

  egress {
    description = "Allow all outbound"       # 出站规则说明
    from_port   = 0                          # 0 表示所有端口
    to_port     = 0
    protocol    = "-1"                       # -1 表示所有协议
    cidr_blocks = ["0.0.0.0/0"]             # 允许访问所有目标
  }

  tags = {
    Name        = "${var.env}-web-sg"
    Environment = var.env
  }
}

# ────────────────────────────────────────────
# 2. 应用负载均衡（ALB）
# ────────────────────────────────────────────
resource "aws_lb" "app" {
  name               = "${var.env}-app-alb"  # ALB 名称，按环境区分
  internal           = false                 # false = 公网 ALB，可通过互联网访问
  load_balancer_type = "application"         # 应用层负载均衡（HTTP/HTTPS）
  security_groups    = [aws_security_group.web.id]  # 关联 Web 安全组
  subnets            = var.public_subnet_ids         # 部署到公有子网

  tags = {
    Name        = "${var.env}-app-alb"
    Environment = var.env
  }
}

resource "aws_lb_target_group" "app" {
  name     = "${var.env}-app-tg"             # 目标组名称
  port     = 80                              # 后端服务端口（EC2 上 Apache 监听的端口）
  protocol = "HTTP"                          # 健康检查和后端通信使用 HTTP
  vpc_id   = var.vpc_id                      # 目标组所属 VPC

  # health_check 定义 ALB 如何检测后端 EC2 的健康状态
  health_check {
    enabled             = true               # 启用健康检查
    healthy_threshold   = 2                  # 连续 2 次成功即视为健康
    unhealthy_threshold = 2                  # 连续 2 次失败即视为不健康
    interval            = 30                 # 每 30 秒检查一次
    path                = "/"                # 检查根路径的 HTTP 响应
  }

  tags = {
    Name        = "${var.env}-app-tg"
    Environment = var.env
  }
}

resource "aws_lb_listener" "app" {
  # 关联到之前创建的 ALB，指定接收流量的入口
  load_balancer_arn = aws_lb.app.arn
  # 监听 80 端口，接收来自用户的 HTTP 请求
  port              = "80"
  # 使用 HTTP 协议（若使用 HTTPS 需搭配 aws_lb_listener_certificate）
  protocol          = "HTTP"

  # default_action 定义没有匹配任何规则时的默认行为
  default_action {
    # 将所有流量转发到后端的目标组
    type             = "forward"
    # 指定上一步创建的目标组，明确请求要发往哪组后端服务器
    target_group_arn = aws_lb_target_group.app.arn
  }
}

# ────────────────────────────────────────────
# 3. EC2 实例（数量由 replicas 控制）
# ────────────────────────────────────────────
resource "aws_instance" "web" {
  # count 控制创建几台：dev 传 1 台，prod 传 5 台
  count = var.replicas

  ami           = data.aws_ami.amazon_linux.id    # 引用数据源查询到的最新 AL2023 AMI ID
  instance_type = var.instance_type                # 实例规格由调用方传入（dev=微型，prod=大型）

  # 放到公有子网，并关联安全组
  subnet_id              = var.public_subnet_ids[count.index % length(var.public_subnet_ids)]
  # 关联 Web 安全组，开放 80 端口的入站流量
  vpc_security_group_ids = [aws_security_group.web.id]
  # 自动分配公网 IP，用户可以直接通过浏览器访问 EC2 上的 Web 服务
  associate_public_ip_address = true

  # 启动时安装 HTTP 服务
  user_data = <<-EOF
    #!/bin/bash
    dnf install -y httpd                         # Amazon Linux 2023 用 dnf 替代 yum
    systemctl start httpd                         # 启动 Apache 服务
    systemctl enable httpd                        # 设置开机自启
    echo "Hello from ${var.env} environment (instance ${count.index + 1})" > /var/www/html/index.html
  EOF

  tags = {
    Name        = "${var.env}-web-${count.index + 1}"   # 例如 dev-web-1, dev-web-2
    Environment = var.env
  }
}

# 把 EC2 注册到目标组（ALB 才能把流量转发过来）
resource "aws_lb_target_group_attachment" "web" {
  count            = var.replicas                        # 每台 EC2 都注册一次
  target_group_arn = aws_lb_target_group.app.arn         # 关联到上面创建的目标组
  target_id        = aws_instance.web[count.index].id    # 当前 EC2 实例的 ID
  port             = 80                                  # 目标端口 80（EC2 上 Apache 监听的端口）
}

# 获取最新的 Amazon Linux 2023 AMI（⚠️ Amazon Linux 2 已结束标准支持，请用 AL2023）
data "aws_ami" "amazon_linux" {
  most_recent = true                                    # 取最新版本
  owners      = ["amazon"]                              # 限定 AWS 官方提供的 AMI

  filter {
    name   = "name"
    values = ["al2023-ami-*-x86_64"]                    # 匹配 AL2023 x86 架构的镜像名称
  }
}
```

创建 `~/terraform-learning/modules/app/outputs.tf`：

```hcl
# modules/app/outputs.tf
# 暴露给调用方使用的输出值

output "alb_dns_name" {
  description = "ALB 的 DNS 域名，访问这个域名就能到达应用"
  value       = aws_lb.app.dns_name
}

output "instance_ids" {
  description = "创建的 EC2 实例 ID 列表"
  value       = aws_instance.web[*].id
}

output "security_group_id" {
  description = "Web 安全组 ID"
  value       = aws_security_group.web.id
}
```

有了这个模块后，每个环境的核心 `main.tf` 就非常简洁了——直接给模块传值：

```hcl
# environments/dev/main.tf

# 先创建网络层（VPC + 子网）
module "networking" {
  source   = "../../modules/networking"
  vpc_cidr = "10.0.0.0/16"
  env      = "dev"
}

# 再创建应用层（安全组 + ALB + EC2），引用 networking 的输出
module "app" {
  source        = "../../modules/app"

  env           = "dev"
  instance_type = "t3.micro"
  replicas      = 1

  vpc_id             = module.networking.vpc_id
  public_subnet_ids  = module.networking.subnet_ids
}
```

```hcl
# environments/prod/main.tf

module "networking" {
  source   = "../../modules/networking"
  vpc_cidr = "10.0.0.0/16"
  env      = "prod"
}

module "app" {
  source        = "../../modules/app"

  env           = "prod"
  instance_type = "t3.large"
  replicas      = 5

  vpc_id             = module.networking.vpc_id
  public_subnet_ids  = module.networking.subnet_ids
}
```

> 💡 两个环境的差异只在 `main.tf` 的传值上体现。`modules/app/variables.tf` 是唯一的接口定义，环境层不需要再重复声明变量。

#### 📄 `backend.tf`（以 dev 为例，各环境路径不同）

> ⚠️ 在写 `backend.tf` 之前，先手动创建好 S3 桶（参考 Day 4 的 bootstrap 说明），然后把 `bucket` 名字改成你创建的桶名。

```hcl
# environments/dev/backend.tf
terraform {
  backend "s3" {
    bucket = "你的名字-terraform-state"   # 改为你手动创建的桶名
    key    = "dev/terraform.tfstate"     # ← 不同的路径！dev/staging/prod 各不同
    region = "ap-northeast-1"
    use_lockfile = true                  # 启用 state 锁定，防并发冲突
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
# locals 根据当前 workspace 动态选择实例类型
locals {
  # 用 map 查找的方式：terraform.workspace 返回当前 workspace 名称
  instance_type = {
    dev     = "t3.micro"      # 开发环境用最小规格
    staging = "t3.small"      # 测试环境稍大
    prod    = "t3.large"      # 生产环境用高性能规格
  }[terraform.workspace]       # 用当前 workspace 名作为 key 查表取值
}
```

> ⚠️ Workspace 适合简单场景，复杂多环境推荐用**目录结构**，因为目录结构更清晰、更不容易误操作（在 dev 目录下 apply 不会影响到 prod 的 state）。

### 8.4 安全实践

#### 敏感信息处理

**❌ 不要硬编码**——写在 `variables.tf` 的 `default` 里会提交到 Git：

```hcl
# ❌ variables.tf —— 密码写在 default 里，会提交到 Git 暴露
variable "db_password" {
  default = "SuperSecret123!"
}
```

**✅ 方式一：环境变量**——`variables.tf` 声明变量但不给 default，运行时通过终端传入：

```hcl
# ✅ variables.tf —— 只声明类型，不设 default
variable "db_password" {
  type      = string
  sensitive = true    # 日志中隐藏值，但 state 文件仍存明文
}
```

```bash
# ✅ 终端 —— 运行时设置，不进入版本控制
export TF_VAR_db_password=SuperSecret123!
terraform apply        # Terraform 自动读取 TF_VAR_ 前缀的环境变量
```

**✅ 方式二：AWS Secrets Manager**——直接写在 `main.tf` 中，`variables.tf` 不再需要 `db_password` 变量：

```hcl
# ✅ main.tf —— 从 Secrets Manager 读取密码，避免任何位置明文存储
data "aws_secretsmanager_secret_version" "db_password" {
  secret_id = "my-db-password"  # Secrets Manager 中的密钥名称
}

resource "aws_db_instance" "main" {
  password = data.aws_secretsmanager_secret_version.db_password.secret_string
}

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

### 8.5 跨环境共享：`terraform_remote_state` 数据源

当你的项目拆分为多个独立的 Terraform 项目（如 `networking/`、`services/`）时，不同项目之间需要共享输出值。`terraform_remote_state` 数据源就是干这个的：

```hcl
# data "terraform_remote_state" 读取其他 Terraform 项目生成的 state 文件
# 这样不同项目之间可以共享输出值，无需硬编码
data "terraform_remote_state" "networking" {
  backend = "s3"                              # 与目标项目相同的后端类型

  config = {
    bucket = "my-company-terraform-state"     # 目标项目 state 所在的 S3 桶
    key    = "networking/terraform.tfstate"   # 目标项目 state 的路径（不同环境不同 key）
    region = "ap-northeast-1"                 # S3 桶所在区域
  }
}

# 使用另一个项目的输出：通过 outputs.xxx 访问目标项目的 output 块
resource "aws_instance" "web" {
  subnet_id = data.terraform_remote_state.networking.outputs.public_subnet_ids[0]
  #           ↑                                   ↑
  #           数据源引用                           networking 项目中定义的 output 名
}
```

> ⚠️ **安全注意事项**：使用 `terraform_remote_state` 意味着你可以读取另一个项目的 state 文件。确保 S3 桶的访问策略限制哪些人可以读取 state。

### 8.6 CI/CD 集成（GitOps 方式）

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

#### GitHub Actions 示例（安全版本）

> ⚠️ **使用 OIDC 替代静态密钥**：下面示例假设通过 OIDC（OpenID Connect）进行 AWS 认证，而不是在 CI 中存储 AWS 密钥。这是当前的安全最佳实践——OIDC 让 CI 不需要保存任何长期有效的凭据。GitHub/AWS/GitLab 等都支持。

```yaml
name: Terraform
on:
  push:
    branches: [main]
  pull_request:
    branches: [main]

# OIDC 权限配置（替代静态 AWS 密钥）
permissions:
  id-token: write      # 允许请求 OIDC token
  contents: read        # 允许读取仓库代码
  pull-requests: write  # 允许 plan 输出作为 PR 评论

jobs:
  # Job 1：Format + Validate（轻量级检查，在 plan 之前拦截低级错误）
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - name: Terraform Format
        run: terraform fmt -check -recursive
        # -check：检查格式是否正确，不对则退出（不会自动修改）
        # -recursive：递归检查所有子目录

      - name: Terraform Validate
        run: terraform validate
        # ├── 建议配合 tflint（最佳实践检查）一起使用
        # └── 建议配合 checkov / tfsec（安全扫描）一起使用

  # Job 2：Plan（PR 触发，展示变更预览）
  plan:
    needs: [check]
    if: github.event_name == 'pull_request'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - name: Configure AWS via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-terraform
          aws-region: ap-northeast-1

      - name: Terraform Init
        run: terraform init
        working-directory: environments/${{ github.head_ref || github.ref_name }}

      - name: Terraform Plan
        run: terraform plan -no-color
        working-directory: environments/${{ github.head_ref || github.ref_name }}

      # 可选：将 plan 输出作为 PR 评论
      - name: Post Plan to PR
        uses: actions/github-script@v7
        with:
          script: |
            const fs = require('fs');
            const output = fs.readFileSync('plan_output.txt', 'utf8');
            // 这里可以将 plan 输出发布为 PR 评论

  # Job 3：Apply（仅 main 分支，需要单独的人工审批步骤）
  apply:
    needs: [check]
    if: github.ref == 'refs/heads/main' && github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: hashicorp/setup-terraform@v3

      - name: Configure AWS via OIDC
        uses: aws-actions/configure-aws-credentials@v4
        with:
          role-to-assume: arn:aws:iam::123456789012:role/github-actions-terraform
          aws-region: ap-northeast-1

      - name: Terraform Init
        run: terraform init
        working-directory: environments/main

      - name: Terraform Apply
        run: terraform apply -auto-approve
        working-directory: environments/main
        # ⚠️ -auto-approve 适合 CI/CD 自动化部署
        #   生产环境建议先执行 plan 保存到文件
        #   人工确认后再 apply plan.tfplan
```

> 💡 **推荐检查工具组合**：
> - **[tflint](https://github.com/terraform-linters/tflint)** — 检查 Terraform 代码风格和最佳实践
> - **[checkov](https://www.checkov.io/)** — 基础设施安全扫描（合规、CIS 基线）
> - **[infracost](https://www.infracost.io/)** — PR 中展示 Terraform 变更的成本预估，防止意外超支
> - **[tfsec](https://github.com/aquasecurity/tfsec)** — 专门的安全扫描

> 🔒 **关于 `.terraform.lock.hcl`**：Terraform 会自动生成这个锁文件来锁定 provider 的版本。**请把它提交到 Git 仓库**。这样团队所有成员和 CI 都使用同一版本的 provider，避免"在我的电脑上能跑"的问题。

### ✅ Day 8 学习成果检查

- [ ] 理解多环境管理的两种方案（目录结构 vs Workspace）
- [ ] 理解如何复用 Module 来管理不同环境
- [ ] 知道如何处理敏感信息（环境变量 / Secrets Manager）
- [ ] 了解 Terraform + CI/CD 的基本工作流
- [ ] 会使用 `terraform_remote_state` 跨项目读取 state 输出
- [ ] 了解 tflint / checkov / infracost 等辅助工具

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

---

## 🏁 Capstone 项目：从零搭建完整环境

如果你已经完成以上 8 天的学习，现在尝试**独立完成这个总结项目**来检验所学：

### 项目要求

1. **目录结构**：设计 `modules/` + `environments/` 分离结构
2. **网络模块**：`modules/networking/` — 创建 VPC + 公有/私有子网 + IGW + NAT 网关
3. **计算模块**：`modules/compute/` — 创建 ALB + 安全组 + EC2（使用 `for_each` 管理多台）
4. **数据库模块**：`modules/database/` — 创建 RDS（使用 `random_password` 生成密码）
5. **多环境配置**：为 `dev / staging / prod` 各创建一套 `terraform.tfvars`
6. **远程状态**：配置 S3 后端（提示：先手动创建 S3 桶，再配置 `backend.tf`，详见 Day 4）
7. **资源保护**：给 RDS 实例添加 `lifecycle { prevent_destroy = true }`

### 加分项

- 使用 Git 管理代码，配置 `.gitignore`
- 集成检查工具：运行 `terraform validate`、`tflint`、`checkov`
- 使用 `terraform_remote_state` 数据源跨环境读取 VPC ID
- 将配置部署到 CI/CD（GitHub Actions 或 GitLab CI）

> 💡 遇到困难时回忆：`terraform plan` 预览变更，`terraform state list` 查看管理中的资源，`terraform validate` 检查语法。

---

**下一步可以探索的方向（推荐按此顺序学习）：**

推荐的进阶学习顺序，从最贴近 Terraform 的工具开始，逐步向外扩展：

1. 🔄 **[Terragrunt 入门指南](terraform-进阶-terragrunt.md)** —— **推荐第 1 步**
   → 最贴近 Terraform，只解决代码重复问题，不引入新认知模型，学习成本最低
2. 🔐 **[Vault 入门指南](terraform-进阶-vault.md)** —— **推荐第 2 步**
   → HashiCorp 生态的核心组件，Terraform + Vault 集成是生产环境最佳实践
3. 🏗️ **[Pulumi 入门指南](terraform-进阶-pulumi.md)]** —— **编程背景强，推荐第 3 步**
   → 如果你觉得 HCL 的限制让你烦躁，用通用编程语言写 IaC
4. 🚀 **[Crossplane 入门指南](terraform-进阶-crossplane.md)** —— **K8s 优先团队推荐第 3 步**
   → 如果你已经在用 K8s，想把云资源也拉进来统一管理

> Pulumi 和 Crossplane 是二选一的关系，取决于团队方向：编程背景强选 Pulumi，K8s 原生选 Crossplane。Terragrunt 和 Vault 则是通用补充——运维越多，越早学越好。

---

> **遇到问题先查：** `terraform plan` 可以看到变更预览；`terraform state list` 看当前管的资源；`terraform validate` 检查语法；这三个解决 80% 的疑惑。

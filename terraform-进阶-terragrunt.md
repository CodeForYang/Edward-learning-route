# Terragrunt 入门指南 —— Terraform 的 DRY 利器

> **前置知识**：建议先完成 [Terraform 入门指南](terraform-入门指南.md) 的 **Day 8（多环境管理）**，理解纯 Terraform 如何管理多环境后，再学 Terragrunt 如何简化这一过程。
>
> **一句话**：Terragrunt 让你不用重复写相同的 Terraform 配置，一个根配置统管所有环境。

---

## 1. 它解决了什么问题？

用 Terraform 管理多环境时，你很快会发现同一个痛苦：

```
terraform/
├── dev/
│   ├── main.tf          # 和 staging 的 main.tf 几乎一样
│   ├── variables.tf
│   └── backend.tf
├── staging/
│   ├── main.tf          # 和 dev 的 main.tf 几乎一样
│   ├── variables.tf
│   └── backend.tf
├── prod/
│   ├── main.tf          # 又复制了一遍！
│   ├── variables.tf
│   └── backend.tf
```

**每个环境的配置高度重复**，修改一个就要手动同步到其他环境。

**Terragrunt 的做法**：把公共部分提取成"根配置"，每个环境只写差异部分。

```
terragrunt/
├── terragrunt.hcl           # 全局配置（远程后端、provider 配置）
├── dev/
│   └── terragrunt.hcl      # 只写 dev 特有的变量值
├── staging/
│   └── terragrunt.hcl      # 只写 staging 特有的变量值
└── prod/
    └── terragrunt.hcl      # 只写 prod 特有的变量值
```

---

## 2. 安装

```bash
# macOS（最方便）
brew install terragrunt

# Linux（下载二进制）
wget https://github.com/gruntwork-io/terragrunt/releases/latest/download/terragrunt_linux_amd64
chmod +x terragrunt_linux_amd64
sudo mv terragrunt_linux_amd64 /usr/local/bin/terragrunt

# 验证安装
terragrunt --version
```

> **注意**：Terragrunt 不替换 Terraform，它只是"包裹"在 Terraform 外面。安装 Terragrunt 后，你仍然需要安装 Terraform。

---

## 3. 第一个例子：用 Terragrunt 管理 VPC 模块

### 3.1 目录结构

```
terragrunt-demo/
├── terragrunt.hcl                  # 根配置 —— 所有环境共享
├── modules/
│   └── vpc/
│       └── main.tf                 # 标准的 Terraform 模块（和之前写法完全一样）
└── env/
    ├── dev/
    │   └── vpc/
    │       └── terragrunt.hcl      # dev 环境 VPC 配置
    └── prod/
        └── vpc/
            └── terragrunt.hcl      # prod 环境 VPC 配置
```

### 3.2 编写共享的 Terraform 模块

`modules/vpc/main.tf` —— 这个模块和普通 Terraform 模块**没有任何区别**：

```hcl
# modules/vpc/main.tf
# Terragrunt 不改变模块编写方式，它只管"如何调用模块"

variable "environment" {
  description = "环境名称（dev/staging/prod），用于命名和标签"
  type        = string
}

variable "vpc_cidr" {
  description = "VPC 的网段，例如 10.0.0.0/16"
  type        = string
}

variable "public_subnet_cidrs" {
  description = "公有子网的网段列表，传几个就建几个"
  type        = list(string)
}

variable "private_subnet_cidrs" {
  description = "私有子网的网段列表，传几个就建几个"
  type        = list(string)
}

# --- 资源创建部分（和之前入门指南完全一样）---

resource "aws_vpc" "main" {
  cidr_block = var.vpc_cidr

  tags = {
    Name        = "vpc-${var.environment}"
    Environment = var.environment
    ManagedBy   = "terragrunt"
  }
}

resource "aws_subnet" "public" {
  count = length(var.public_subnet_cidrs)

  vpc_id     = aws_vpc.main.id
  cidr_block = var.public_subnet_cidrs[count.index]

  map_public_ip_on_launch = true

  tags = {
    Name        = "vpc-${var.environment}-public-${count.index}"
    Environment = var.environment
    Tier        = "public"
  }
}

resource "aws_subnet" "private" {
  count = length(var.private_subnet_cidrs)

  vpc_id     = aws_vpc.main.id
  cidr_block = var.private_subnet_cidrs[count.index]

  tags = {
    Name        = "vpc-${var.environment}-private-${count.index}"
    Environment = var.environment
    Tier        = "private"
  }
}

# Terragrunt 可以通过 dependency 读取这些 output
output "vpc_id" {
  value = aws_vpc.main.id
}

output "public_subnet_ids" {
  value = aws_subnet.public[*].id
}

output "private_subnet_ids" {
  value = aws_subnet.private[*].id
}
```

### 3.3 编写根 terragrunt.hcl（全局共享配置）

`terragrunt-demo/terragrunt.hcl` —— 这个文件位于项目根目录，**作用于所有子目录**：

```hcl
# terragrunt-demo/terragrunt.hcl
# 放在项目根目录，所有子目录的 terragrunt.hcl 都会继承这个配置

# --- 远程后端配置（相当于以前每个目录下的 backend.tf）---
remote_state {
  backend = "s3"  # 也可以用 local，但生产环境必须用远程后端

  config = {
    bucket = "my-company-terraform-state"  # S3 桶名（全局唯一）

    # 路径根据当前目录自动生成：
    # dev/vpc 下的配置 → key = "env/dev/vpc/terraform.tfstate"
    key = "${path_relative_to_include()}/terraform.tfstate"

    region         = "ap-northeast-1"
    dynamodb_table = "terraform-state-lock"  # 锁表，防止多人同时操作
    encrypt        = true                     # 传输中加密
  }
}

# --- 自动生成 provider.tf ---
# 这样每个子目录不用重复写 provider 配置
generate "provider" {
  path      = "provider.tf"          # 生成的文件名
  if_exists = "overwrite_terragrunt" # 已存在时就覆盖

  # 这里写的是 Terraform 的 provider 配置语法
  contents = <<EOF
provider "aws" {
  region = "ap-northeast-1"

  default_tags {
    tags = {
      ManagedBy = "terragrunt"
    }
  }
}
EOF
}
```

### 3.4 编写环境级 terragrunt.hcl（只写差异部分）

`terragrunt-demo/env/dev/vpc/terragrunt.hcl` —— dev 环境：

```hcl
# terragrunt-demo/env/dev/vpc/terragrunt.hcl
# 这个文件只写 dev 环境"独特"的部分，不重复 provider 和后端配置

# 引入模块
terraform {
  source = "../../../modules/vpc"  # 指向我们前面写的 VPC 模块
}

# 继承根配置（找父目录的 terragrunt.hcl）
include "root" {
  path = find_in_parent_folders()
}

# 传参给 VPC 模块（填在 variable 块里的值）
inputs = {
  environment = "dev"

  # dev 用较小的网段，节省资源
  vpc_cidr = "10.0.0.0/16"

  # dev 用 2 个可用区即可，降低成本
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.10.0/24", "10.0.20.0/24"]
}
```

`terragrunt-demo/env/prod/vpc/terragrunt.hcl` —— prod 环境：

```hcl
# terragrunt-demo/env/prod/vpc/terragrunt.hcl

terraform {
  source = "../../../modules/vpc"
}

include "root" {
  path = find_in_parent_folders()
}

# prod 和 dev 的区别：更大、更多、更安全
inputs = {
  environment = "prod"

  vpc_cidr = "10.0.0.0/16"

  # prod 需要 3 个可用区，实现高可用
  public_subnet_cidrs = [
    "10.0.1.0/24",
    "10.0.2.0/24",
    "10.0.3.0/24"
  ]
  private_subnet_cidrs = [
    "10.0.10.0/24",
    "10.0.20.0/24",
    "10.0.30.0/24"
  ]
}
```

### 3.5 部署

```bash
# 进入 dev 环境目录
cd terragrunt-demo/env/dev/vpc

# Terragrunt 会自动执行三步：
# 1. 下载模块（类似 terraform init）
# 2. 生成 provider.tf 和 backend.tf
# 3. 执行 terraform plan 或 apply

# 预览（等价于 terraform plan）
terragrunt plan

# 部署（等价于 terraform apply -auto-approve）
terragrunt apply

# 全部部署完
cd ../../prod/vpc
terragrunt plan
terragrunt apply
```

**批量部署：使用 `run-all` 按依赖顺序自动编排**

进入环境根目录，一次部署所有子模块：

```bash
# 进入环境根目录（dev 或 prod）
cd terragrunt-demo/env/dev

# 自动识别依赖顺序，先部署 VPC，再部署依赖 VPC 的模块（如 EKS）
terragrunt run-all apply

# 查看所有子模块的部署状态
terragrunt run-all output

# 销毁时反向顺序执行（先销毁 EKS，再销毁 VPC）
terragrunt run-all destroy
```

> 💡 在真实项目中，`terragrunt run-all apply` 是最大价值所在——你可以在根目录一条命令完成所有环境的编排部署，Terraform 原生没有这个能力。

---

## 4. 核心模式：依赖管理

Terragrunt 一个很强的功能是**自动处理模块之间的依赖关系**。

例如：EKS 集群依赖于 VPC 先创建好。

```hcl
# terragrunt-demo/env/dev/eks/terragrunt.hcl
# EKS 集群依赖 VPC

terraform {
  source = "../../../modules/eks"  # EKS 模块
}

include "root" {
  path = find_in_parent_folders()
}

# --- 声明依赖 ---
# 告诉 Terragrunt：先部署 VPC，再部署 EKS
dependencies {
  paths = ["../vpc"]  # 依赖同级目录下的 vpc
}

# --- 读取被依赖模块的输出 ---
# 通过 dependency 块访问 VPC 模块的 output 值
dependency "vpc" {
  config_path = "../vpc"

  # 可选：mock 值，用于 plan 阶段
  # 即使 VPC 还没部署，也能先 plan（假设值）
  mock_outputs_allowed_terraform_commands = ["plan"]
  mock_outputs = {
    vpc_id             = "fake-vpc-id"
    public_subnet_ids  = ["subnet-fake1", "subnet-fake2"]
    private_subnet_ids = ["subnet-fake3", "subnet-fake4"]
  }
}

inputs = {
  environment = "dev"

  # 从 VPC 模块的输出中读取子网 ID
  # 等价于 Terraform 中的 data.terraform_remote_state
  subnet_ids = dependency.vpc.outputs.private_subnet_ids
}
```

> ⚠️ `mock_outputs` 只在 `plan` 阶段生效，`apply` 时会使用真实值。但 mock 值可能掩盖真正的依赖问题——如果 mock 值和真实值的结构不一致，运行时才会报错。

然后可以用 `run-all` 一次性按依赖顺序部署所有模块：

```bash
# 在 env/dev 目录下执行
cd terragrunt-demo/env/dev

# 按依赖顺序部署所有模块（VPC 先，EKS 后）
terragrunt run-all apply

# 按依赖顺序销毁（EKS 先，VPC 后）
terragrunt run-all destroy
```

---

## 5. 常用命令速查

```bash
# Terragrunt 命令 → 等价 Terraform 命令

terragrunt plan              # → terraform plan
terragrunt apply             # → terraform apply -auto-approve
terragrunt destroy           # → terraform destroy
terragrunt output            # → terraform output
terragrunt state list        # → terraform state list
terragrunt validate          # → terraform validate
terragrunt fmt               # → terraform fmt

# Terragrunt 特有命令

terragrunt run-all plan      # 递归执行所有子目录的 plan
terragrunt run-all apply     # 按依赖顺序部署所有子模块
terragrunt hclfmt            # 格式化所有 .hcl 文件
terragrunt terragrunt-info   # 查看当前目录的 Terragrunt 配置详情
```

---

## 6. Terragrunt vs 原生 Terraform 速览

| 场景 | 原生 Terraform | Terragrunt |
|------|---------------|------------|
| 多环境管理 | 每个环境复制粘贴 | 继承 + 只写差异部分 |
| 远程后端 | 每个模块手写 backend.tf | 根配置统一定义 |
| 依赖编排 | 手动解耦或写 bash 脚本 | `dependencies` 自动解析执行顺序 |
| 变量传递 | 手写 `var = xxx` | `inputs` 自动注入模块变量 |
| 模块版本 | `source = "git::...?ref=v1.0"` | 在 `terraform.source` 里统一定义 |

---

## 7. 什么时候学 Terragrunt？

**建议学**：
- 已经用 Terraform，且管理多个环境（dev/staging/prod）
- 觉得 HCL 重复代码太多
- 团队需要统一后端配置和 provider 版本
- 模块之间有复杂的依赖关系

**不着急学**：
- 项目只有一个环境
- Terraform 配置总共不到 500 行
- 你刚接触 IaC 不到一周（先把 Terraform 基础打牢）

---

> **一句话总结**：Terragrunt 不教你新的 IaC 知识，它只让你写 Terraform 时**少复制粘贴**。

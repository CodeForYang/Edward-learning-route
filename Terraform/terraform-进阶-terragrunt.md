# Terragrunt 入门指南 —— Terraform 的 DRY 利器

> **前置知识**：建议先完成 [Terraform 入门指南](terraform-入门指南.md) 的 **Day 8（多环境管理）**，理解纯 Terraform 如何管理多环境后，再学 Terragrunt 如何简化这一过程。
>
> **一句话**：Terragrunt 让你不用重复写相同的 Terraform 配置，一个根配置统管所有环境。

---

## 1️⃣ 搭建学习阶梯

> 将 Terragrunt 技能拆解为 5 个递进等级，每个等级标注了**掌握内容**、**常见错误**和**进阶标准**。按顺序学习，不要跳级。

---

### Lv 1 —— 认知层：Terragrunt 是什么？

**平时要做什么**
- 理解纯 Terraform 管理多环境的痛苦：大量重复配置
- 掌握 Terragrunt 的核心理念：把公共部分提取成根配置，每个环境只写差异
- 理解 Terragrunt 和 Terraform 的关系：Terragrunt = Terraform 的"包装器"，不是替代品
- 理解 Terragrunt 的目录结构设计原则

**Terragrunt 解决了什么问题**

```
# 纯 Terraform 多环境 —— 大量重复
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

**核心要点**
- Terragrunt **不替换** Terraform，而是"包裹"在外面
- 你的 Terraform 模块**不需要任何修改**就能被 Terragrunt 使用
- Terragrunt 自动生成 `provider.tf` 和 `backend.tf`，子目录不用手写

**常见错误**
- ❌ 以为 Terragrunt 是另一个 IaC 工具 —— 它就是 Terraform 的增强版 shell
- ❌ 装 Terragrunt 但不装 Terraform —— 会报错，Terragrunt 执行时调的是 Terraform
- ❌ 把 Terragrunt 的 .hcl 文件和 Terraform 的 .tf 文件混淆 —— 格式不同，作用不同
- ❌ 不知道"包装器"意味着什么 —— 所有 Terraform 命令都可以通过 Terragrunt 执行

**进阶标准**：你能画一张图，说明 Terragrunt 如何"包裹"Terraform，以及它在多环境管理中处于哪一层。

---

### Lv 2 —— 操作层：安装并管理第一个模块

**平时要做什么**
- 安装 Terragrunt
- 创建标准的目录结构（根配置 + 环境目录 + 模块目录）
- 编写可复用的 Terraform 模块（和之前写法完全一样）
- 编写根 `terragrunt.hcl`（后端 + provider）
- 编写环境级 `terragrunt.hcl`（只写差异输入）
- 使用 `terragrunt plan / apply` 部署

**安装**

```bash
# macOS（最方便）
brew install terragrunt

# Linux（下载二进制）
wget https://github.com/gruntwork-io/terragrunt/releases/latest/download/terragrunt_linux_amd64
chmod +x terragrunt_linux_amd64
sudo mv terragrunt_linux_amd64 /usr/local/bin/terragrunt

# 验证安装（注意：Terragrunt 不替换 Terraform，你仍然需要安装 Terraform）
terragrunt --version
```

**目录结构**

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

**第 1 步：编写共享的 Terraform 模块**

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

**第 2 步：编写根 terragrunt.hcl（全局共享配置）**

```hcl
# terragrunt-demo/terragrunt.hcl
# 放在项目根目录，所有子目录的 terragrunt.hcl 都会继承这个配置

remote_state {
  backend = "s3"
  config = {
    bucket = "my-company-terraform-state"
    key = "${path_relative_to_include()}/terraform.tfstate"
    region         = "ap-northeast-1"
    dynamodb_table = "terraform-state-lock"
    encrypt        = true
  }
}

generate "provider" {
  path      = "provider.tf"
  if_exists = "overwrite_terragrunt"

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

**第 3 步：编写环境级 terragrunt.hcl（只写差异部分）**

```hcl
# terragrunt-demo/env/dev/vpc/terragrunt.hcl

terraform {
  source = "../../../modules/vpc"
}

include "root" {
  path = find_in_parent_folders()
}

inputs = {
  environment = "dev"
  vpc_cidr = "10.0.0.0/16"
  public_subnet_cidrs  = ["10.0.1.0/24", "10.0.2.0/24"]
  private_subnet_cidrs = ["10.0.10.0/24", "10.0.20.0/24"]
}
```

**第 4 步：prod 环境（对比差异）**

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
  public_subnet_cidrs = [
    "10.0.1.0/24", "10.0.2.0/24", "10.0.3.0/24"
  ]
  private_subnet_cidrs = [
    "10.0.10.0/24", "10.0.20.0/24", "10.0.30.0/24"
  ]
}
```

**部署**

```bash
cd terragrunt-demo/env/dev/vpc

# Terragrunt 会自动执行三步：
# 1. 下载模块（类似 terraform init）
# 2. 生成 provider.tf 和 backend.tf
# 3. 执行 terraform plan 或 apply

terragrunt plan      # 预览
terragrunt apply     # 部署

cd ../../prod/vpc
terragrunt plan
terragrunt apply
```

**常见错误**
- ❌ 目录层级写错导致 `find_in_parent_folders()` 找不到根配置 —— 检查目录结构和路径
- ❌ `remote_state` 里的 S3 桶不存在 —— Terraform 不会自动创建，要手动先建好
- ❌ 同时设了 `source` 和 `include` 但继承关系混乱 —— `include` 是继承根配置，`source` 是引用模块
- ❌ 在子目录里直接跑 `terraform apply` 而不是 `terragrunt apply` —— 不会自动生成 provider 和 backend

**进阶标准**：你能独立搭建一个 terragrunt 项目，包含 dev/prod 两个环境的 VPC，并成功部署。

---

### Lv 3 —— 依赖层：模块间依赖管理

**平时要做什么**
- 理解 Terragrunt 的 `dependencies` 和 `dependency` 块
- 学会用 `dependency` 读取其他模块的 output
- 掌握 `mock_outputs` 的用法和风险
- 会用 `run-all` 批量按依赖顺序部署

**为什么需要依赖管理？**

当你的基础设施有依赖关系时（例如：EKS 集群依赖于 VPC 先创建好），Terragrunt 能自动处理顺序。

```hcl
# terragrunt-demo/env/dev/eks/terragrunt.hcl
# EKS 集群依赖 VPC

terraform {
  source = "../../../modules/eks"
}

include "root" {
  path = find_in_parent_folders()
}

# --- 声明依赖顺序 ---
dependencies {
  paths = ["../vpc"]  # 先部署 VPC，再部署 EKS
}

# --- 读取被依赖模块的输出 ---
dependency "vpc" {
  config_path = "../vpc"

  # mock 值：用于 plan 阶段，即使 VPC 还没部署也能先执行方案
  mock_outputs_allowed_terraform_commands = ["plan"]
  mock_outputs = {
    vpc_id             = "fake-vpc-id"
    public_subnet_ids  = ["subnet-fake1", "subnet-fake2"]
    private_subnet_ids = ["subnet-fake3", "subnet-fake4"]
  }
}

inputs = {
  environment = "dev"
  subnet_ids = dependency.vpc.outputs.private_subnet_ids
}
```

**批量部署：run-all 命令**

```bash
# 进入环境根目录
cd terragrunt-demo/env/dev

# 自动识别依赖顺序，先部署 VPC，再部署 EKS
terragrunt run-all apply

# 查看所有子模块的部署状态
terragrunt run-all output

# 销毁时反向顺序执行（先销毁 EKS，再销毁 VPC）
terragrunt run-all destroy
```

> 💡 `terragrunt run-all apply` 是最大价值所在——你可以在根目录一条命令完成所有环境的编排部署，Terraform 原生没有这个能力。

**常见错误**
- ❌ `dependencies` 里路径写错 —— 相对于当前 terragrunt.hcl 的路径
- ❌ 过度依赖 `mock_outputs` —— mock 值可能掩盖真正的依赖问题，如果 mock 值和真实值的结构不一致，运行时才会报错
- ❌ 在 `mock_outputs_allowed_terraform_commands` 里加了 `apply` —— mock 值只在 plan 阶段生效，加上 apply 可能导致用假值创建了错误资源
- ❌ 忘记给模块写 `output` 块 —— `dependency` 读的是模块的 output，没有 output 就无法传递值

**进阶标准**：你能用 `dependencies` + `dependency` 管理 VPC → EKS 的依赖链，并用 `run-all` 一条命令顺序部署。

---

### Lv 4 —— 对比层：Terragrunt vs 原生 Terraform

| 场景 | 原生 Terraform | Terragrunt |
|------|---------------|------------|
| 多环境管理 | 每个环境复制粘贴 | 继承 + 只写差异部分 |
| 远程后端 | 每个模块手写 backend.tf | 根配置统一定义 |
| Provider 配置 | 每个目录写 provider.tf | `generate` 自动生成 |
| 依赖编排 | 手动解耦或写 bash 脚本 | `dependencies` 自动解析执行顺序 |
| 变量传递 | 手写 `var = xxx` | `inputs` 自动注入模块变量 |
| 模块版本 | `source = "git::...?ref=v1.0"` | 在 `terraform.source` 里统一定义 |
| 批量部署 | 写脚本逐个目录执行 | `run-all` 一条命令 |
| 学习成本 | 0（这是你要学的） | 额外学 Terragrunt 的 .hcl 语法 |

**常见错误**
- ❌ 项目没那么复杂就上 Terragrunt —— 单一环境、配置少于 500 行时没有收益
- ❌ 以为 Terragrunt 能减少 Terraform 的复杂度 —— 它只减少重复，不能简化 Terraform 自身的复杂性
- ❌ 把所有逻辑都塞进根 terragrunt.hcl —— 根配置是共享默认值，每个环境的特殊逻辑应该在各自的 terragrunt.hcl

**进阶标准**：你能判断一个项目是否需要 Terragrunt，并给出理由。

---

### Lv 5 —— 生产层：什么时候用 Terragrunt？

**建议用 Terragrunt**：
- 已经用 Terraform，且管理多个环境（dev/staging/prod）
- 觉得 HCL 重复代码太多
- 团队需要统一后端配置和 provider 版本
- 模块之间有复杂的依赖关系

**不着急用 Terragrunt**：
- 项目只有一个环境
- Terraform 配置总共不到 500 行
- 你刚接触 IaC 不到一周（先把 Terraform 基础打牢）

**常见错误**
- ❌ 项目初期就引入 Terragrunt —— 先验证 Terraform 模块本身 OK，再加 Terragrunt
- ❌ 过度依赖 Terragrunt 的"自动化"而忽略了 Terraform 本身的理解 —— 先懂 Terraform 的核心概念
- ❌ 多人协作时目录结构不一致 —— 约定统一的目录规范（团队文档化）

**进阶标准**：你能向一个团队介绍 Terragrunt 的利弊，帮助他们决定是否需要引入。

---

> **一句话总结**：Terragrunt 不教你新的 IaC 知识，它只让你写 Terraform 时**少复制粘贴**。

---

## 2️⃣ 聚焦核心 20%

> Terragrunt 中最重要的 20% 内容：**根配置 + 环境差异 + 依赖管理**。这 20% 覆盖了 Terragrunt 80% 的使用场景。

---

| 次数 | 主题 | 核心内容 | 练习资源 | 复盘问题 |
|:----:|------|----------|----------|----------|
| 1 | 安装 + 概念理解 | 安装 Terragrunt、理解包装器模式、目录结构 | 阅读本文 Lv1，画一张 Terragrunt 和 Terraform 的关系图 | Terragrunt 是替换 Terraform 还是增强？ |
| 2 | 编写 Terraform 模块 | 写一个标准的 VPC 模块（注意：和之前一样，不用改） | 写一个带 variable 和 output 的 VPC 模块 | 这个模块能被 Terraform 直接使用吗？ |
| 3 | 根配置 | `remote_state` + `generate` provider | 写一个根 terragrunt.hcl，定义后端和 provider | `find_in_parent_folders()` 的作用是什么？ |
| 4 | dev 环境配置 | `terraform.source`、`include`、`inputs` | 写 dev 环境的 terragrunt.hcl，部署 dev VPC | `inputs` 和 Terraform 的 `variables` 是什么关系？ |
| 5 | prod 环境配置 | dev vs prod 的差异（网段、可用区数量） | 写 prod 环境的 terragrunt.hcl，部署 prod VPC | dev 和 prod 的 terragrunt.hcl 有哪些重复？怎么消除？ |
| 6 | 依赖管理 | `dependencies` 声明顺序、`dependency` 读取 output | 写一个 EKS 模块，依赖 VPC 模块的输出 | 为什么需要 `dependencies` + `dependency` 两个块？ |
| 7 | mock_outputs | mock 值的用途和风险 | 在 EKS 的 terragrunt.hcl 加入 mock_outputs | mock 值在 apply 阶段会发生什么？ |
| 8 | run-all 批量部署 | `run-all apply`、`run-all destroy` 按顺序执行 | 在 env/dev 目录执行 `run-all apply` | `run-all` 是怎么识别依赖顺序的？ |
| 9 | 全面对比 | Terragrunt vs 原生 Terraform | 画一张对比表 | 你的项目需要 Terragrunt 吗？为什么？ |
| 10 | 综合实战 | 从零搭建完整的多环境 Terragrunt 项目 | dev/prod 两套环境，VPC + EKS 完整依赖链 | 如果新增一个 staging 环境，你需要改哪些文件？ |

> **学习建议**：前 5 次是 Terragrunt 核心能力，后 5 次是依赖和批量管理。建议 Day 1-5 连续学习，Day 6-10 穿插练习。

---

## 3️⃣ AI 考官测试

> 学完以上内容后，告诉 AI："开始 Terragrunt 考官测试"，按以下流程进行：

**测试规则**
1. AI 一次只问 **一个问题**
2. 从**简单到困难**逐步深入
3. 你回答后，AI 立即给出**反馈** + **针对性讲解**

**测试范围**
- 第一轮：概念题（Terragrunt 和 Terraform 是什么关系？`inputs` 的作用是什么？）
- 第二轮：操作题（写出创建 dev 环境 VPC 的完整目录结构和 terragrunt.hcl）
- 第三轮：场景题（"你有 dev/prod 两个环境，VPC 和 EKS 有依赖关系，怎么组织？"）
- 第四轮：设计题（"一个 10 人团队、5 个环境、20 个模块，用 Terragrunt 后目录该怎么规划？"）

**启动方式**
> "开始 Terragrunt 考官测试，从简单概念题开始。"

---

## 4️⃣ 制作速查表

### 速查表 A：基本操作

| 项目 | 内容 |
|------|------|
| **一句话** | Terragrunt 是 Terraform 的"DRY 包装器"，将公共配置提取为根配置，每个环境只写差异，大幅减少重复代码 |
| **核心文件** | `terragrunt.hcl`（根）= 全局共享 · `terragrunt.hcl`（环境）= 差异部分 · `modules/` = 可复用的 Terraform 模块 |
| **真实案例** | 团队有 dev/staging/prod 三个环境，根配置统一定义 S3 后端和 AWS provider，每个环境只传不同的 `vpc_cidr` 和 `instance_type` |

**常用命令**

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

**常见错误**
- ❌ `find_in_parent_folders()` 找不到根配置 —— 检查目录层级
- ❌ `remote_state` 的 S3 桶没手动创建 —— Terragrunt 不会自动创建
- ❌ 忘了装 Terraform —— Terragrunt 不是独立工具
- ❌ `run-all` 在错误目录执行 —— 在环境根目录（如 `env/dev`）执行，不是在模块目录

**检查清单**
- [ ] Terraform 是否已安装？
- [ ] Terragrunt 是否已安装？
- [ ] 根 `terragrunt.hcl` 中 `remote_state` 的 S3 桶是否已创建？
- [ ] 根 `terragrunt.hcl` 的 `generate "provider"` 配置是否正确？
- [ ] 环境目录的 `include` 是否指向了根配置？
- [ ] 环境目录的 `terraform.source` 是否指向了正确的模块路径？
- [ ] `inputs` 中的变量名是否和模块的 `variable` 定义一致？

**5 个自测题**
1. Terragrunt 和 Terraform 的关系是什么？
2. 根 terragrunt.hcl 中 `remote_state` 块的作用是什么？
3. 环境目录中 `include "root"` 的作用是什么？
4. `inputs` 块里的值和模块的什么对应？
5. dev 和 prod 环境的 terragrunt.hcl 哪个部分应该一样，哪个部分应该不同？

---

### 速查表 B：依赖管理

| 项目 | 内容 |
|------|------|
| **一句话** | `dependencies` 声明模块间的部署顺序，`dependency` 读取被依赖模块的 output 值 |
| **核心块** | `dependencies { paths = [...] }` 定义顺序 · `dependency "vpc" { config_path = "..." }` 读取 output · `mock_outputs` plan 阶段使用假值 |
| **真实案例** | EKS 模块依赖 VPC，在 VPC 部署完成后自动读取 `vpc_id` 和 `subnet_ids` 传给 EKS 模块 |

**常用命令**

```bash
# 批量部署（按依赖顺序）
terragrunt run-all apply
terragrunt run-all destroy  # 反向顺序
terragrunt run-all output   # 查看所有模块输出
terragrunt run-all plan     # 批量预览
```

**常见错误**
- ❌ `dependencies` 路径相对于当前文件 —— 不是绝对路径
- ❌ `dependency` 读不到值 —— 检查被依赖模块是否写了对应的 output
- ❌ `mock_outputs` 加到 `apply` 命令 —— mock 值只应在 plan 阶段生效
- ❌ `run-all apply` 在模块目录里执行 —— 应该在环境根目录（如 `env/dev`）执行

**检查清单**
- [ ] `dependencies` 路径是否正确？
- [ ] 被依赖模块是否有对应的 `output` 块？
- [ ] `dependency` 的 `config_path` 是否指向正确的目录？
- [ ] `mock_outputs` 的结构是否和真实 output 一致？
- [ ] `mock_outputs_allowed_terraform_commands` 是否只包含 `plan`？

**5 个自测题**
1. `dependencies` 和 `dependency` 有什么区别？
2. `dependency.vpc.outputs.vpc_id` 中的 `vpc_id` 来自哪里？
3. `mock_outputs` 的作用是什么？有什么风险？
4. 如果用 `run-all apply` 执行，VPC 和 EKS 谁先部署？
5. 销毁时依赖顺序会怎么变化？

---

## 5️⃣ 筛选优质资源

| # | 资源 | 类型 | 难度 | 适合人群 | 怎么用 |
|:-:|------|------|:----:|----------|--------|
| 1 | [Terragrunt 官方入门](https://terragrunt.gruntwork.io/docs/getting-started/overview/) | 官方文档 | ⭐⭐ | 零基础入门 | 走一遍官方示例，结合本文 Lv1-Lv2 |
| 2 | [Terragrunt 功能文档](https://terragrunt.gruntwork.io/docs/features/) | 官方文档 | ⭐⭐⭐ | 需要深入理解的读者 | 按需阅读，重点关注 `keep`、`dependency`、`run-all` |
| 3 | [Gruntwork 博客：为什么用 Terragrunt](https://blog.gruntwork.io/why-we-use-terragrunt-and-why-you-should-too-6c6b3c1b1c8e) | 博客文章 | ⭐⭐ | 做技术选型的读者 | 学完 Lv1 后阅读，理解设计动机 |
| 4 | [Gruntwork 示例仓库](https://github.com/gruntwork-io/terragrunt-infrastructure-modules-example) | 示例代码 | ⭐⭐⭐ | 需要参考最佳实践的读者 | 直接参考其目录结构和 terragrunt.hcl 写法 |
| 5 | [Gruntwork 基础设施直播模板](https://github.com/gruntwork-io/terragrunt-infrastructure-live-example) | 示例代码 | ⭐⭐⭐⭐ | 准备上生产的团队 | 参考其多环境组织结构、依赖管理的最佳实践 |

**不推荐的资源**
- 过旧的 Terragrunt v0.x 博客（API 变化大，旧语法已废弃）
- 非官方的中文教程（翻译不准确，关键术语理解偏差）

---

### 7 天学习路径

| 天 | 学习内容 | 使用资源 | 预计时间 |
|:--:|----------|----------|:--------:|
| Day 1 | 概念 + 安装（Lv 1） | 资源 #1 官方入门 + 本文 Lv1 | 1h |
| Day 2 | 根配置 + dev 环境（Lv 2） | 资源 #1 + 资源 #4 模块示例 | 2h |
| Day 3 | prod 环境 + 差异对比（Lv 2） | 资源 #5 直播模板参考 | 1.5h |
| Day 4 | 依赖管理（Lv 3） | 资源 #2 功能文档 | 2h |
| Day 5 | mock_outputs + run-all（Lv 3） | 资源 #2 + 动手练习 | 1.5h |
| Day 6 | 对比 + 选型（Lv 4-Lv 5） | 资源 #3 Gruntwork 博客 | 1h |
| Day 7 | 综合实战 | 从零搭建多环境 Terragrunt 项目 | 2h |

> **建议**：Day 1-3 掌握核心配置，Day 4-5 深入依赖管理，Day 6-7 对比和实战。Terragrunt 本身很简单，重点是不停练习目录结构。

---

## 6️⃣ 费曼循环巩固

> 费曼学习法：**如果你不能简单地解释它，你就没有真正理解它。**

### 用 12 岁孩子能懂的话来说

**Terragrunt 是什么？**

你要给三个朋友（dev、staging、prod）各写一封信，告诉他们怎么搭积木。

以前你得写三封几乎一样的信：
- 信 A："用 10.0.0.0/16 网段搭 VPC……配置用 S3 后端……"
- 信 B："用 10.0.0.0/16 网段搭 VPC……配置用 S3 后端……"
- 信 C："用 10.0.0.0/16 网段搭 VPC……配置用 S3 后端……"

Terragrunt 让你做一件事：写一封**公共的信**放在客厅（根配置），然后在每个朋友的信里只写**不一样的部分**："dev 只需要 2 个子网，prod 要 3 个。"公共的部分去客厅看就行了。

**Terragrunt 怎么工作的？**

Terragrunt 不自己搭积木——它只是**帮你准备好材料**（自动生成 provider.tf、backend.tf），然后喊一声"Terraform，开始搭！"

就像你的助理帮你把所有工具摆好了、图纸铺好了，然后说："开始吧"，Terraform 就开始干活。

**依赖管理是什么意思？**

你想先搭 VPC（因为子网和安全组都在 VPC 里），再搭 EKS（因为 EKS 要在 VPC 里运行）。

Terragrunt 说："我帮你看着顺序——等 VPC 搭好了，我自动把 VPC 的 ID 传给 EKS，然后开始搭 EKS。"

你不用手动等着 VPC 搭完再执行下一个命令，Terragrunt 帮你一条龙搞定。

**什么时候不该用 Terragrunt？**

如果你只有一个朋友，只写一封信就够了，没必要搞"公共信 + 个人信"这套。Terragrunt 只有在管理多个环境时才发挥价值。

### 你的费曼练习

现在轮到你。按以下步骤做：

1. **复述**：合上文档，用自己的话给一个会 Terraform 但没用过 Terragrunt 的同事解释
2. **检查漏洞**：对照以下问题

**自检问题**
- ❓ Terragrunt 的核心价值是什么？（用一句话说）
- ❓ 根 terragrunt.hcl 应该放什么？环境级 terragrunt.hcl 应该放什么？
- ❓ `dependencies` 和 `dependency` 的区别？
- ❓ `run-all apply` 和逐个目录 `apply` 有什么不同？
- ❓ 什么情况下 Terragrunt 不带来额外价值？

### 循环方法

```
Round 1: 你用自己的话说 → 我发现漏洞并指出 → 你去查资料修正
Round 2: 你重新说（这次更准确） → 我再指出改进点
Round 3: 你再说 → 漏洞越来越少 → 你真正掌握了 ✅
```

开始费曼循环，告诉我："开始 Terragrunt 费曼循环，我先用自己的话解释 Terragrunt 是什么。"

---

> **最后说一句**：Terragrunt 不教你新的 IaC 知识——它只是让你写 Terraform 时**少复制粘贴**。如果你觉得在 Terraform 里手动复制 backend.tf 和 provider.tf 很烦，你就知道 Terragrunt 是为什么而生的。

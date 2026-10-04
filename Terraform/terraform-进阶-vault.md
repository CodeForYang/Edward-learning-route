# Vault 入门指南 —— 基础设施的密钥管理中心

> **前置知识**：本文假设你已完成 [Terraform 入门指南](terraform-入门指南.md) 中 Day 4（状态管理）和 Day 8（安全实践）的内容，理解 Terraform 中敏感信息处理的痛点。
>
> **一句话**：把密码、API Key、证书统一存在一个安全保险箱里，用的时候申请临时凭证，用完自动销毁。

---

## 1️⃣ 搭建学习阶梯

> 将 Vault 技能拆解为 5 个递进等级，每个等级标注了**掌握内容**、**常见错误**和**进阶标准**。请按顺序学习，不要跳级。

---

### Lv 1 —— 认知层：Vault 是什么？

**平时要做什么**
- 理解传统密钥管理的痛点（硬编码 → 环境变量 → 密钥管理服务的演进）
- 掌握 Vault 的 5 个核心概念：Secret、Path、Engine、Policy、Token
- 区分「静态密钥」与「动态密钥」
- 理解 Vault 的定位：统一密钥管理平台，不绑定特定云厂商

**核心概念对比**

| 概念 | 类比 | 说明 |
|------|------|------|
| **Secret** | 保险柜里的一个格子 | 存放具体的密码 / API Key / 证书 |
| **Path** | 保险柜编号 / 路径 | 按路径组织，例如 `secret/myapp/database` |
| **Engine** | 不同类型的保险柜 | KV（存静态密码）、AWS（动态生成云凭证）、Database（动态生成数据库账号） |
| **Policy** | 权限规则 | 规定谁能读/写哪个路径 |
| **Token** | 保险柜钥匙 | 访问 Vault 时需要出示的凭证 |

**Vault 解决了什么问题**

```hcl
# ❌ 方案 A：硬编码在代码里（不安全，所有人可见）
resource "aws_db_instance" "main" {
  password = "MyP@ssw0rd123!"    # Git 提交后人人都能看
}

# ❌ 方案 B：手动输入（不安全且不可自动化）
# 每次 terraform apply 都需要人工输入密码
export TF_VAR_db_password=<手动输入>
terraform apply                   # CI/CD 里根本没法用

# ❌ 方案 C：存在云厂商的密钥管理服务（厂商锁定）
data "aws_secretsmanager_secret_version" "db" { ... }  # 迁移到 GCP / Azure 就得换方案
```

**常见错误**
- ❌ 把 Vault 和云厂商的 Secrets Manager 对立 —— Vault 是**统一管理**，可同时对接多家
- ❌ 以为 Vault 只能存静态密码 —— 动态密钥才是它最强大的能力
- ❌ 跳过概念直接上手操作 —— 不理解 Engine/Policy/Token 的关系，后面会乱

**进阶标准**：你能用一句话向同事解释清楚 Vault 是什么，以及和 AWS Secrets Manager 的区别。

---

### Lv 2 —— 操作层：安装并读写第一个密钥

**平时要做什么**
- 在本地安装 Vault 并启动开发模式
- 通过 CLI 进行 KV 引擎的读写操作
- 理解 path 的组织方式（`secret/myapp/database`）
- 熟悉 `vault status`、`vault kv` 等基本命令

**安装 + 启动**

```bash
# macOS
brew install vault

# Linux
wget https://releases.hashicorp.com/vault/1.18.0/vault_1.18.0_linux_amd64.zip
unzip vault_*.zip
sudo mv vault /usr/local/bin/

# 验证安装
vault --version

# 启动开发模式（数据在内存里，重启后丢失，不要在生产环境用！）
vault server -dev
```

启动后你会看到两行关键信息：
- `Root Token: hvs.xxxxxxxxxxxx` —— 管理员的"万能钥匙"
- `Unseal Key: xxxxxxxxxxxxxxxx` —— 解封密钥（生产环境要用）

**新开一个终端窗口**，设置连接信息：

```bash
export VAULT_ADDR='http://127.0.0.1:8200'
export VAULT_TOKEN='hvs.xxxxxxxxxxxx'   # 替换成上面输出的 Root Token
vault status
```

**第一个操作：存储和读取静态密钥**

```bash
# --- 写入 ---
vault kv put secret/myapp/database \
  username=admin \
  password=S3cret!

# --- 读取全部字段 ---
vault kv get secret/myapp/database

# --- 只读某个字段（脚本里很有用） ---
vault kv get -field=password secret/myapp/database
# 输出：S3cret!

# --- 列出路径 ---
vault kv list secret/

# --- 删除 ---
vault kv delete secret/myapp/database
```

**常见错误**
- ❌ 忘记设置 `VAULT_ADDR` 和 `VAULT_TOKEN` —— 会连不上 Vault
- ❌ 开发模式测试完直接用于生产 —— dev 模式无认证、无持久化、无高可用
- ❌ `vault kv put` 路径写错导致覆盖其他密钥 —— 路径是精确匹配，没有"目录"保护
- ❌ 把 Root Token 硬编码在脚本里 —— Root Token 是超级管理员权限，泄露等于全盘失守

**进阶标准**：你能不翻文档，流畅完成启动 Vault → 写入密钥 → 读取密钥 → 删除密钥的完整流程。

---

### Lv 3 —— 动态层：Vault 最强大的功能

**平时要做什么**
- 理解动态密钥的核心思想：**临时创建、用完即毁**
- 掌握 Database 引擎的完整流程（配置连接 → 定义角色 → 获取临时凭证）
- 了解 AWS 引擎生成临时云凭证
- 理解 Lease（租约）机制：自动过期时间

**动态密钥 vs 静态密钥**

> 💡 **核心思想**：把"密钥"从静态资产变成动态资产。
>
> **传统密码**：记住一个固定密码，定期人工轮换。泄露了不知道，知道了也持续有效。
> **动态密钥**：每次需要时自动创建临时凭证，用完后自动失效。即使泄露了，也只有短暂的影响窗口。

Vault 支持多种动态引擎：数据库（MySQL/PostgreSQL/MongoDB）、云服务商（AWS/GCP/Azure）、PKI 证书等。下面以 MySQL 为例演示完整流程。

**示例：Vault 动态创建数据库账号**

```bash
# ===== 第一步：开启数据库引擎 =====
vault secrets enable database

# ===== 第二步：告诉 Vault 数据库的连接方式 =====
vault write database/config/my-mysql \
    plugin_name=mysql-database-plugin \
    connection_url="{{username}}:{{password}}@tcp(127.0.0.1:3306)/" \
    allowed_roles="my-role" \
    username="root" \
    password="rootpassword"

# ===== 第三步：定义角色，决定临时账号有什么权限 =====
vault write database/roles/my-role \
    db_name=my-mysql \
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" \
    default_ttl="1h" \
    max_ttl="24h"
# ⚠️ 以上示例中的 root 密码和 GRANT SELECT ON *.* 仅为演示
# 生产环境应使用最小权限账号，避免泄露 root 密码

# ===== 第四步：获取一个临时数据库账号 =====
vault read database/creds/my-role

# 输出：
# Key                Value
# ---                -----
# lease_id           database/creds/my-role/xxx
# lease_duration     1h              ← 1小时后自动过期
# password           xxxxxxxxxx      ← 自动生成的随机密码
# username           v-root-xxxx     ← 自动生成的用户名

# 这个账号完成工作后会自动被删除
# 不需要记密码、不需要轮转密码
```

**示例：Vault 创建临时 AWS 密钥**

```bash
# 开启 AWS 引擎
vault secrets enable aws

# 配置 AWS 根凭证（Vault 用它来生成临时凭证）
vault write aws/config/root \
    access_key=AKIAXXXXXX \
    secret_key=xxxxxxxxxx \
    region=ap-northeast-1

# 定义一个角色：创建 IAM 用户，临时拥有 S3 读权限
vault write aws/roles/s3-reader \
    credential_type=iam_user \
    policy_document=-<<EOF
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": "s3:GetObject",
      "Resource": "*"
    }
  ]
}
EOF

# 获取临时 AWS 凭证（有效期默认 1 小时）
vault read aws/creds/s3-reader
# 输出：
# Key                Value
# ---                -----
# access_key         AKIAxxxxx
# secret_key         xxxxx
# security_token     xxxxx       ← 临时凭证需要 token
# lease_duration     1h
```

**常见错误**
- ❌ 配置 database engine 时直接用 root 账号 —— 应该创建专门的 Vault 管理账号并授予最小权限
- ❌ `default_ttl` 不设置或设得过长 —— 动态密钥的优势在于短暂有效
- ❌ 不理解 Lease 机制 —— 租约到期后凭证会被自动吊销，应用要做好重新获取的逻辑
- ❌ 在同一 Vault 上用同一个角色给所有应用 —— 应该按应用/环境分角色

**进阶标准**：你能配置一个数据库动态密钥角色，让应用每次启动都获取一个 1 小时有效的临时账号。

---

### Lv 4 —— 集成层：在 Terraform 中使用 Vault

**平时要做什么**
- 掌握 Vault Provider 配置方式（address / token / 环境变量）
- 理解两种集成模式：Vault 作为密钥源 vs Terraform 管理 Vault
- 会用 `vault_kv_secret_v2` data source 读取密钥
- 会用 `vault_database_creds` data source 获取临时数据库凭证

**配置 Vault Provider**

```hcl
# vault-provider.tf
# 让 Terraform 可以从 Vault 读取密钥

provider "vault" {
  # 如果设置了 VAULT_ADDR 环境变量，address 可以省略
  address = "http://127.0.0.1:8200"

  # Token 推荐用环境变量 VAULT_TOKEN 传入，不写在代码里
}

# --- 读取静态密钥 ---
data "vault_kv_secret_v2" "db" {
  mount = "secret"
  name  = "myapp/database"  # 对应 vault kv put 的路径
}

# 在资源中引用 Vault 里的密钥
resource "aws_db_instance" "main" {
  # 不再硬编码密码
  password = data.vault_kv_secret_v2.db.data["password"]
}

# --- 获取临时数据库账号（动态密钥）---
data "vault_database_creds" "app" {
  backend = "database"      # 对应上面开启的数据库引擎
  role    = "my-role"       # 对应上面定义的角色
}

# 用临时账号作为 Terraform 的 MySQL provider
provider "mysql" {
  endpoint = "my-db.xxxxx.ap-northeast-1.rds.amazonaws.com:3306"
  username = data.vault_database_creds.app.username
  password = data.vault_database_creds.app.password
}
```

**两种集成模式**

```hcl
# --- 模式 A：Vault 作为密钥源（最常用）---
# Terraform 从 Vault 读取已有密钥，注入到资源中

data "vault_kv_secret_v2" "db" {
  mount = "secret"
  name  = "myapp/database"
}

resource "aws_db_instance" "main" {
  username = data.vault_kv_secret_v2.db.data["username"]
  password = data.vault_kv_secret_v2.db.data["password"]
}

# --- 模式 B：Terraform 管理 Vault 本身（自举模式）---
# 用 Terraform 来配置 Vault，适合新搭建的 Vault 环境

# 开启一个 KV 存储引擎
resource "vault_mount" "secret" {
  path = "secret"
  type = "kv-v2"
}

# 往 Vault 里写密钥
resource "vault_kv_secret_v2" "app_config" {
  mount = vault_mount.secret.path
  name  = "myapp/database"

  data_json = jsonencode({
    username = "admin"
    password = random_password.db.result  # Terraform 生成随机密码，存入 Vault
  })
}

# Terraform 生成随机密码（存在 state 里，但只作为中间步骤）
resource "random_password" "db" {
  length  = 24
  special = true
}
```

**常见错误**
- ❌ 在 provider 配置里硬编码 Vault Token —— 用 `VAULT_TOKEN` 环境变量
- ❌ 没弄清楚模式 A 和 B 的区别 —— A 是读，B 是写。用混了会导致数据覆盖
- ❌ 动态密钥在 Terraform 中只 `apply` 一次 —— 临时凭证过期后需要重新获取
- ❌ 忘记把 Vault Provider 加到 `required_providers` —— 会导致 provider 下载失败

**进阶标准**：你能在 Terraform 中同时使用模式 A（读已有密钥）和模式 B（创建新密钥），并理解各自的适用场景。

---

### Lv 5 —— 生产层：安全部署 Vault

**平时要做什么**
- 了解生产环境的部署要求（至少 3 节点集群、Raft/Consul 存储、TLS）
- 理解 Unseal 流程及意义（Shamir 密钥分割）
- 了解 Auto Unseal 和审计日志
- 知道什么时候该用、什么时候不着急用 Vault

| 要求 | 说明 |
|------|------|
| 至少 3 台服务器 | 高可用集群，一台挂了照常服务 |
| 后端存储 | 用 Consul 或 Raft，不用文件存储 |
| TLS 加密 | 所有通信必须走 HTTPS |
| Unseal 流程 | 启动时需要多人一起输入密钥才能解封 |
| 定期轮换 | Root Token 和 Unseal Key 需要定期更换 |

**Unseal 流程详解**

```bash
# 生产环境启动 Vault 后，它处于"密封"（sealed）状态
# 需要至少 3 个人各自输入一部分密钥才能解封

# 第一人（输入自己的 Key Share）
vault operator unseal <key-share-1>

# 第二人
vault operator unseal <key-share-2>

# 第三人
vault operator unseal <key-share-3>

# 解封完成，Vault 开始提供服务

# 这种机制防止一个人就能拿到所有密钥
# 有点像"三把钥匙才能打开保险柜"
```

> 💡 **生产环境的 Auto Unseal**：手动 unseal 在少量服务器时可行，但集群规模大了以后不可扩展。生产推荐使用 **Auto Unseal**（例如 AWS KMS / Azure Key Vault / GCP Cloud KMS），让 Vault 在启动时自动解封，无需人工介入。
>
> 💡 **审计日志（Audit Device）**：生产环境务必启用审计设备，记录所有对 Vault 的访问请求。这不仅是合规要求（PCI-DSS / SOC 2），也是安全事件溯源的关键数据：`vault audit enable file file_path=/var/log/vault/audit.log`

**什么时候学 Vault？**

**建议学**：
- 公司有安全合规要求（金融、医疗、等保）
- 你的 Terraform 代码里有很多硬编码的密码
- 团队超过 5 人，密钥管理开始混乱
- 需要跨云管理密钥（AWS + GCP + Azure 混用）

**不着急学**：
- 个人项目，只有 1-2 个密码
- Terraform 还只是在本地跑着玩
- 密钥改了就改了，没有轮换需求

**常见错误**
- ❌ 生产环境用 dev 模式启动 —— 数据不持久、无 TLS、无认证
- ❌ 单节点运行 Vault —— 挂了就全瘫
- ❌ 未启用审计日志 —— 出了问题没法溯源
- ❌ 所有团队成员共享 Root Token —— 应该每个人有自己的 Token 和 Policy

**进阶标准**：你能画出一个生产 Vault 集群的架构图（3 节点 + Raft 存储 + TLS + Auto Unseal + Audit），并说明每个组件的作用。

---

> **一句话总结**：Vault 不是一个"要不要装"的东西，而是一个"早晚要面对"的话题。项目早期可以先简单用环境变量管密码，但知道有 Vault 这个方案就行。

---

## 2️⃣ 聚焦核心 20%

> Vault 中最重要的 20% 内容：**静态密钥读写 + 动态密钥获取 + Terraform 集成**。这 20% 覆盖了日常工作中 80% 的使用场景。下面是 10 次 × 2 小时学习计划，将大目标拆解为每天可执行的小任务。

---

| 次数 | 主题 | 核心内容 | 练习资源 | 复盘问题 |
|:----:|------|----------|----------|----------|
| 1 | 安装与环境搭建 | 安装 Vault、启动 dev 模式、设置环境变量、`vault status` | 本地启动一个 dev 模式的 Vault | 启动后你能找到 Root Token 在哪吗？ |
| 2 | 核心概念精讲 | Secret/Path/Engine/Policy/Token 的关系，动静密钥对比 | 画一张 Vault 架构图，标出 5 个概念的位置 | 为什么说动态密钥比静态密钥更安全？ |
| 3 | KV 引擎操作 | `kv put/get/list/delete`、路径组织、字段过滤 | 创建一个层级路径 `secret/team-a/project-b/env-c/db`，写入三对密钥 | 如果路径写错了会发生什么？ |
| 4 | KV 引擎进阶 | 元数据查看、版本回滚、密钥生命周期管理 | 读取 `kv metadata get`，观察版本变化 | 你的密钥过期了该怎么安全删除？ |
| 5 | 数据库动态密钥 | 开启 database 引擎、配置连接、定义角色、获取临时凭证 | 用 Docker 跑一个 MySQL，配置 Vault 动态生成账号 | 临时账号过期后，旧连接会怎样？ |
| 6 | AWS 动态密钥 | 开启 aws 引擎、配置 root 凭证、定义 IAM 角色 | 配置一个仅 S3 读权限的临时 AWS 凭证 | 动态密钥和 IAM Role 有什么区别？ |
| 7 | Terraform 集成 A | 配置 Vault Provider、用 data source 读静态密钥注入资源 | 写一个 Terraform 配置，从 Vault 读密钥创建 RDS | 不用 `VAULT_TOKEN` 环境变量，还能怎么传？ |
| 8 | Terraform 集成 B | 模式 B：Terraform 管理 Vault 引擎和密钥 | 用 Terraform 创建 KV 引擎 + 写入密钥 + 生成密码 | 模式 A 和 B 的使用边界在哪里？ |
| 9 | 生产部署概念 | HA 集群、Unseal 流程、Auto Unseal、审计日志 | 在本地用 Raft 存储启动 3 节点的 Vault 集群（docker-compose） | 为什么需要 Unseal，不能直接启动就提供服务？ |
| 10 | 综合实战 | 设计一个完整方案：Terraform 从 Vault 读密钥创建 AWS 资源 | 模拟一个"创建 EC2 并注入数据库密码"的完整流程 | 如果 Vault 挂了，你的应用该如何降级？ |

> **学习建议**：每次学完后花 5 分钟回答"复盘问题"列的问题。答不出来就回看对应内容，不要往下走。

---

## 3️⃣ AI 考官测试

> 这部分不是要"读"的内容，而是一个**交互规则**。当你学完上面所有内容后，告诉 AI："开始 Vault 考官测试"，然后按以下流程进行：

**测试规则**
1. AI 一次只问 **一个问题**
2. 从**简单到困难**逐步深入
3. 你回答后，AI 立即给出**反馈** + **针对性讲解**
4. 像健身教练一样：做错了马上纠正，不让你带着错误理解继续

**测试范围**
- 第一轮：概念题（什么是 Secret？Path 怎么组织？）
- 第二轮：操作题（写出从 Vault 读取一个密码的完整命令）
- 第三轮：场景题（"你的应用需要在每次部署时获取一个临时数据库账号，该怎么做？"）
- 第四轮：故障题（"Vault 启动后返回 sealed，是什么原因？怎么解决？"）

**你可以这样启动**
> "开始 Vault 考官测试，从简单概念题开始。"

---

## 4️⃣ 制作速查表

### 速查表 A：KV Engine 静态密钥操作

| 项目 | 内容 |
|------|------|
| **一句话** | KV Engine 是 Vault 最基本的密钥存储引擎，用路径组织密钥对，适合存静态密码/API Key |
| **核心命令** | `put` 写入 · `get` 读取 · `list` 列出 · `delete` 删除 · `metadata get` 查看元数据 |
| **真实案例** | 在 CI/CD 中，Terraform 从 `secret/myapp/database` 读取数据库密码注入到资源创建 |

**常见错误**
- ❌ 路径写错导致密钥覆盖（`secret/myapp/db` 和 `secret/myapp/database` 是两个独立路径）
- ❌ 忘记设置 `VAULT_ADDR` 和 `VAULT_TOKEN`
- ❌ 用 Root Token 做日常操作 —— 应该创建有权限限制的 Token

**检查清单**
- [ ] 是否设置了 `VAULT_ADDR` 环境变量？
- [ ] 是否设置了 `VAULT_TOKEN` 环境变量？
- [ ] 路径命名是否遵循了 `team/project/env/service` 规范？
- [ ] 是否为不同的应用分配了不同的 Token？
- [ ] 密钥是否已设置合理的生命周期？

**5 个自测题**
1. 写入一个路径为 `secret/prod/redis/password`、值为 `myp@ss` 的密钥
2. 只读取 `password` 字段，不读取整个密钥
3. 列出 `secret/` 下所有路径
4. 查看密钥的元数据和版本历史
5. 删除该密钥

---

### 速查表 B：动态密钥（Database Engine）

| 项目 | 内容 |
|------|------|
| **一句话** | 动态密钥让 Vault 在需要时自动创建临时凭证，到期自动销毁，无需人工轮换密码 |
| **核心命令** | `vault secrets enable database` · `vault write database/config/...` · `vault write database/roles/...` · `vault read database/creds/...` |
| **真实案例** | 应用每天凌晨自动部署，每次部署 Vault 创建一个 8 小时有效的 MySQL 账号，当天销毁 |

**常见错误**
- ❌ 直接用 root 账号配置数据库连接
- ❌ `default_ttl` 设置为 0 或过长
- ❌ 应用没有处理凭证过期的重试逻辑
- ❌ 多个应用共用一个角色

**检查清单**
- [ ] 数据库引擎是否已启用？
- [ ] 数据库连接配置是否用了 Vault 专用账号（最小权限）？
- [ ] 角色是否设置了合理的 `default_ttl` 和 `max_ttl`？
- [ ] `creation_statements` 是否只授予了必要权限？
- [ ] 应用是否有重试逻辑处理凭证过期？

**5 个自测题**
1. 开启 database 引擎需要什么命令？
2. 定义一个角色，让生成的临时账号只有 SELECT 权限，默认 30 分钟过期
3. 获取一个临时数据库凭证，输出 lease_duration
4. 如果临时账号在 1 小时后过期，应用该怎么做？
5. 关闭 database 引擎的命令是什么？

---

### 速查表 C：Terraform 集成

| 项目 | 内容 |
|------|------|
| **一句话** | Terraform 通过 Vault Provider 读取密钥注入到资源中，也可以反过来管理 Vault 本身的配置 |
| **模式 A 场景** | 密钥已存在 Vault 中，Terraform 只是读取并注入到 AWS RDS / EC2 等资源 |
| **模式 B 场景** | 初始化 Vault 环境，用 Terraform 创建 Engine、写入初始化密钥 |
| **核心资源** | `data.vault_kv_secret_v2` · `data.vault_database_creds` · `resource.vault_mount` · `resource.vault_kv_secret_v2` |

**常见错误**
- ❌ 在 provider 里硬编码 Vault Token
- ❌ 同时使用模式 A 和模式 B 访问同一路径导致冲突
- ❌ 动态密钥在 `terraform apply` 后不再刷新
- ❌ 忘了 `required_providers` 里声明 vault

**检查清单**
- [ ] Vault Token 是否通过环境变量 `VAULT_TOKEN` 传入？
- [ ] 是否在 `required_providers` 中添加了 vault？
- [ ] 使用模式 A 还是 B？是否清楚选择理由？
- [ ] 动态密钥的 data source 是否设置了合理的 `depends_on`？
- [ ] 密钥变更后是否执行了 `terraform plan` 确认影响范围？

**5 个自测题**
1. 写出从 Vault 读取 `secret/myapp/database` 中 password 字段的 data source 代码
2. 模式 A 和模式 B 的根本区别是什么？
3. 如何配置 Terraform 在 apply 时获取临时数据库账号？
4. Hardcode Vault Token 在 provider 里有什么风险？
5. 用 Terraform 创建一个 KV Engine 并写入一个随机密码

---

## 5️⃣ 筛选优质资源

> 信息过载是学习的大敌。从大量资料中精选 5 个，标注难度、适合人群和用法，配合 7 天学习路径。

---

| # | 资源 | 类型 | 难度 | 适合人群 | 怎么用 |
|:-:|------|------|:----:|----------|--------|
| 1 | [Vault官方入门教程](https://developer.hashicorp.com/vault/tutorials/getting-started) | 交互式教程 | ⭐⭐ | 零基础入门 | 跟着一步步敲命令，完成后再做本指南的练习 |
| 2 | [Vault 官方文档：What is Vault](https://developer.hashicorp.com/vault/docs/what-is-vault) | 官方文档 | ⭐⭐ | 需要理论框架 | 读完 Lv 1 概念后精读，理解设计哲学 |
| 3 | [Vault: Secure, Store and Tightly Control Access](https://learn.hashicorp.com/vault) | HashiCorp 官方学习平台 | ⭐⭐⭐ | 想系统学习的初中级用户 | 按 Track 顺序学，覆盖官网教程所有核心 lab |
| 4 | [Terraform Vault Provider 文档](https://registry.terraform.io/providers/hashicorp/vault/latest/docs) | 参考文档 | ⭐⭐⭐ | 需要在 Terraform 中用 Vault 的读者 | 不是从头读，是**查**：用到哪个资源搜哪个 |
| 5 | [Vault Production Hardening Guide](https://developer.hashicorp.com/vault/tutorials/operations/production-hardening) | 生产指南 | ⭐⭐⭐⭐ | 要部署生产的运维 | 学了 Lv 5 后再看，逐条检查你的部署配置 |

**不推荐的资源**
- 过时的博客文章（Vault API 变化快，看旧文章容易学到已被废弃的用法）
- 非官方翻译的中文文档（翻译质量参差不齐，关键术语可能翻错）

---

### 7 天学习路径

| 天 | 学习内容 | 使用资源 | 预计时间 |
|:--:|----------|----------|:--------:|
| Day 1 | 安装 + 核心概念（Lv 1） | 资源 #1 官方入门教程 | 1.5h |
| Day 2 | KV 引擎操作（Lv 2） | 资源 #1 + 本指南速查表 A | 2h |
| Day 3 | 数据库动态密钥（Lv 3） | 资源 #1 的动态密钥教程 | 2h |
| Day 4 | AWS 动态密钥（Lv 3） | 资源 #3 HashiCorp Learn | 1.5h |
| Day 5 | Terraform 集成（Lv 4） | 资源 #4 Provider 文档 | 2h |
| Day 6 | 综合实战 + 故障排查 | 用真实场景串起 Lv 2-Lv 4 | 2h |
| Day 7 | 生产部署概念（Lv 5） | 资源 #5 Production Hardening | 1.5h |

> **建议**：Day 1-5 每天学新内容，Day 6-7 用于串联和巩固。不要一天学超过 2 小时，间隔学习比集中轰炸效果好 3 倍。

---

## 6️⃣ 费曼循环巩固

> 费曼学习法：**如果你不能简单地解释它，你就没有真正理解它。** 下面我用 12 岁孩子能懂的话解释 Vault 的核心概念。读完后，**合上文档，用自己的话复述一遍**，然后来找我，我会指出你的漏洞。

---

### 用 12 岁孩子能懂的话来说

**Vault 是什么？**

想象学校有一个**失物招领处**。每个同学捡到东西都交到这里，有人丢了东西也来这里找。

- 失物招领处 = **Vault**
- 捡到的物品 = **密钥**（密码、API Key）
- 物品所在的架子编号 = **Path**
- 管理员 = **Vault Engine**（不同架子管不同类型的东西）
- 谁能取哪个架子的规则 = **Policy**
- 你想取东西时出示的学生证 = **Token**

一个架子（KV Engine）存的是**固定的物品**（静态密钥）。另一个架子（Database Engine）是你需要时会**现场造一个新东西**给你，你用完了它自动消失（动态密钥）。

**动态密钥为什么厉害？**

传统做法：你告诉全班同学一个**固定的保险柜密码**，每个人都能开，密码永远不会变。一旦有人泄露了密码，全班都危险。

Vault 的做法：每次有人需要开保险柜，Vault 给他发一个**临时的验证码**，5 分钟后自动失效。即使他不小心把验证码发到了群里，5 分钟后也没人能用它开保险柜了。

**Vault 怎么应对"坏人拿到密钥"？**

传统做法：发现泄露 → 人工改密码 → 通知所有人 → 很慢，而且你改密码时可能不知道已经被人用了 3 个月。

Vault 的做法：发现泄露 → 什么都不用做 → 因为那个密钥 30 分钟后就自己失效了。

**生产环境为什么需要 Unseal？**

想象 Vault 是一个装了所有钥匙的保险柜，保险柜本身也是锁着的。每天开机时，需要 3 个各自管一把钥匙的人同时来开锁。这样即使一个管理员被坏人收买了，他也打不开保险柜。

---

### 你的费曼练习

现在轮到你。请按以下步骤做：

1. **复述**：合上这篇文档，用自己的话给一个虚拟的同事解释"Vault 是什么"
2. **检查漏洞**：回来对照以下问题，看自己有没有答不上来的地方

**自检问题**
- ❓ 静态密钥和动态密钥的区别是什么？
- ❓ Vault 用路径（Path）组织密钥，和文件系统的路径有什么异同？
- ❓ Terraform 集成的两种模式分别解决什么问题？
- ❓ 为什么生产环境要有 Unseal 流程？
- ❓ 如果 Vault 宕机了，正在运行的应用程序受影响吗？（提示：已获取的临时凭证在有效期内仍可使用）

> 如果你发现某个问题解释不清，回到对应的学习阶梯层级，重新读一遍。

### 循环方法

```
Round 1: 你用自己的话说 → 我发现漏洞并指出 → 你去查资料修正
Round 2: 你重新说（这次更准确） → 我再指出改进点
Round 3: 你再说 → 我能挑出的漏洞越来越少 → 你真正掌握了 ✅
```

要开始费曼循环，告诉我："开始 Vault 费曼循环，我先用自己的话解释 Vault 是什么。"

---

> **最后说一句**：掌握 Vault 的关键不是记住所有命令，而是理解"把密钥从静态资产变成动态资产"这个核心思想。命令忘了可以查手册，思想对了方向就不会偏。

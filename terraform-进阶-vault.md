# Vault 入门指南 —— 基础设施的密钥管理中心

> **前置知识**：本文假设你已完成 [Terraform 入门指南](terraform-入门指南.md) 中 Day 4（状态管理）和 Day 8（安全实践）的内容，理解 Terraform 中敏感信息处理的痛点。
>
> **一句话**：把密码、API Key、证书统一存在一个安全保险箱里，用的时候申请临时凭证，用完自动销毁。

---

## 1. 它解决了什么问题？

用 Terraform 管密钥时，你会面临两难：

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

**Vault 的解法**：统一管理所有云厂商的密钥，而且能**动态生成临时凭证**。

---

## 2. 安装并启动

```bash
# macOS
brew install vault

# Linux
wget https://releases.hashicorp.com/vault/1.18.0/vault_1.18.0_linux_amd64.zip
unzip vault_*.zip
sudo mv vault /usr/local/bin/

# 验证安装
vault --version
```

### 2.1 启动开发模式（仅用于本地学习）

```bash
# 开发模式：数据存在内存里，重启后丢失
# 不要在生产环境用！

vault server -dev

# 启动后你会看到两行关键信息：
# Root Token: hvs.xxxxxxxxxxxx   ← 管理员的"万能钥匙"
# Unseal Key: xxxxxxxxxxxxxxxx   ← 解封密钥（生产环境要用）
```

**新开一个终端窗口**，设置连接信息：

```bash
# 告诉 vault 命令行工具：我的 Vault 跑在哪里
export VAULT_ADDR='http://127.0.0.1:8200'

# 告诉 vault 命令行工具：用哪个 token 访问
export VAULT_TOKEN='hvs.xxxxxxxxxxxx'   # 替换成上面输出的 Root Token

# 测试连接
vault status
```

---

## 3. 核心概念

| 概念 | 类比 | 说明 |
|------|------|------|
| **Secret** | 保险柜里的一个格子 | 存放具体的密码 / API Key / 证书 |
| **Path** | 保险柜编号 / 路径 | 按路径组织，例如 `secret/myapp/database` |
| **Engine** | 不同类型的保险柜 | KV（存静态密码）、AWS（动态生成云凭证）、Database（动态生成数据库账号） |
| **Policy** | 权限规则 | 规定谁能读/写哪个路径 |
| **Token** | 保险柜钥匙 | 访问 Vault 时需要出示的凭证 |

---

## 4. 第一个例子：存储和读取静态密钥（KV Engine）

```bash
# --- 写入一对密钥 ---
# vault kv put <路径> key1=value1 key2=value2

vault kv put secret/myapp/database \
  username=admin \
  password=S3cret!

# --- 读取所有字段 ---
vault kv get secret/myapp/database

# 输出：
# ====== Data ======
# Key         Value
# ---         -----
# password    S3cret!
# username    admin

# --- 只读取某个字段（脚本里很有用）---
vault kv get -field=password secret/myapp/database
# 输出：S3cret!

# --- 列出所有可用路径 ---
vault kv list secret/

# --- 删除路径 ---
vault kv delete secret/myapp/database
```

---

## 5. 动态密钥 —— Vault 最强大的功能

**传统方案的问题**：密码长期有效。泄露了也不知道，发现时已经造成损失。

**动态密钥的做法**：每次需要时**临时创建**，用完后**自动销毁**。

> 💡 **核心思想**：把"密钥"从静态资产变成动态资产。
> 
> **传统密码**：记住一个固定密码，定期人工轮换。泄露了不知道，知道了也持续有效。
> **动态密钥**：每次需要时自动创建临时凭证，用完后自动失效。即使泄露了，也只有短暂的影响窗口。
>
> Vault 支持多种动态引擎：数据库（MySQL/PostgreSQL/MongoDB）、云服务商（AWS/GCP/Azure）、PKI 证书等。下面以 MySQL 为例演示完整流程。

### 5.1 示例：Vault 动态创建数据库账号

假设你有一个 MySQL 数据库，你想让应用每次启动时获取一个**限时有效**的临时账号。

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
    # 创建临时账号的 SQL
    creation_statements="CREATE USER '{{name}}'@'%' IDENTIFIED BY '{{password}}';GRANT SELECT ON *.* TO '{{name}}'@'%';" \
    default_ttl="1h" \         # 默认 1 小时后过期
    max_ttl="24h"              # 最多续签到 24 小时
    # ⚠️ 以上示例中的 root 密码和 GRANT SELECT ON *.* 仅为演示
    # 生产环境应使用最小权限账号，避免泄露 root 密码

# ===== 第四步：获取一个临时数据库账号 =====
# 每次执行这个命令，Vault 都会在数据库里创建一个真实的 MySQL 账号
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

### 5.2 Vault 还可以创建临时 AWS 密钥

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

---

## 6. 在 Terraform 中集成 Vault

### 6.1 配置 Vault Provider

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

### 6.2 两种集成模式

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

---

## 7. 生产部署须知

```
Vault 在生产环境不要用 dev 模式！你需要：
```

| 要求 | 说明 |
|------|------|
| 至少 3 台服务器 | 高可用集群，一台挂了照常服务 |
| 后端存储 | 用 Consul 或 Raft，不用文件存储 |
| TLS 加密 | 所有通信必须走 HTTPS |
| Unseal 流程 | 启动时需要多人一起输入密钥才能解封 |
| 定期轮换 | Root Token 和 Unseal Key 需要定期更换 |

> 💡 **生产环境的 Auto Unseal**：手动 unseal 在少量服务器时可行，但集群规模大了以后不可扩展。生产推荐使用 **Auto Unseal**（例如 AWS KMS / Azure Key Vault / GCP Cloud KMS），让 Vault 在启动时自动解封，无需人工介入。
>
> 💡 **审计日志（Audit Device）**：生产环境务必启用审计设备，记录所有对 Vault 的访问请求。这不仅是合规要求（PCI-DSS / SOC 2），也是安全事件溯源的关键数据：`vault audit enable file file_path=/var/log/vault/audit.log`

### 7.1 Unseal 流程详解

### 7.1 生产环境的 Unseal 流程

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

---

## 8. 快速参考

```bash
# 密钥读写
vault kv put     secret/<path> key=value    # 写入
vault kv get     secret/<path>              # 读取
vault kv list    secret/                    # 列出
vault kv delete  secret/<path>              # 删除
vault kv metadata get secret/<path>         # 查看元数据

# 引擎管理
vault secrets list                          # 查看已启用的引擎
vault secrets enable <engine>               # 启用一个引擎
vault secrets disable <path>                # 关闭引擎

# 策略管理
vault policy list                           # 列出策略
vault policy read <name>                    # 查看策略详情
vault policy write <name> <file.hcl>        # 写策略（文件里定义权限）

# 认证方式
vault auth list                             # 查看已有认证方式
vault token create                          # 创建新 token
vault token revoke <token>                  # 撤销 token
```

---

## 9. 什么时候学 Vault？

**建议学**：
- 公司有安全合规要求（金融、医疗、等保）
- 你的 Terraform 代码里有很多硬编码的密码
- 团队超过 5 人，密钥管理开始混乱
- 需要跨云管理密钥（AWS + GCP + Azure 混用）

**不着急学**：
- 个人项目，只有 1-2 个密码
- Terraform 还只是在本地跑着玩
- 密钥改了就改了，没有轮换需求

---

> **一句话总结**：Vault 不是一个"要不要装"的东西，而是一个"早晚要面对"的话题。项目早期可以先简单用环境变量管密码，但知道有 Vault 这个方案就行。

# GitHub Actions → AWS Lambda → Jenkins CICD Demo

## 项目概述

模拟真实企业场景：用户 `git push` 到 GitHub 仓库后，触发一整套自动化构建链路。

```
┌──────────┐   push    ┌──────────────────┐   HTTP    ┌──────────────┐
│  开发者    │ ────────→ │  GitHub Actions   │ ────────→ │  API Gateway │
└──────────┘           │  (webhook 接收方)  │          └──────┬───────┘
                       └──────────────────┘                  │
                                                             ▼
                       ┌──────────────────┐          ┌──────────────┐
                       │  SSM Parameter   │ ←─────── │   Lambda     │
                       │  Store (密码存储)  │  取密码   │   (触发器)    │
                       └──────────────────┘          └──────┬───────┘
                                                             │
                                                             ▼
                       ┌──────────────────┐          ┌──────────────┐
                       │   Jenkins (EC2)  │ ←─────── │  构建产物     │
                       │   执行构建任务     │          │  (S3 存档)    │
                       └──────────────────┘          └──────────────┘
```

## 技术栈

| 组件 | 技术 | 用途 |
|------|------|------|
| 代码仓库 | GitHub | 存放源码 + GitHub Actions workflow |
| CI 触发 | GitHub Actions | 接收 push 事件，调用 AWS API Gateway |
| API 入口 | AWS API Gateway | 对外暴露 HTTPS 端点，触发 Lambda |
| 计算 | AWS Lambda (Python 3.11) | 从 SSM 取凭证，调用 Jenkins API 创建构建 |
| 密钥管理 | AWS SSM Parameter Store (SecureString) | 存储 Jenkins 用户名/API Token（永久免费） |
| CI 引擎 | Jenkins (EC2 t2.micro) | 拉代码 → 构建 → 测试 → 归档 |
| 网络 | VPC + Security Group | 安全组控制访问来源 |

## 知识点覆盖

| # | 知识点 | 在哪里体现 |
|---|--------|-----------|
| 1 | Jenkins 密钥配置 | Step 4：创建 API Token + 配置 Credential |
| 2 | Webhook 与 AWS 通信 | Step 3：GitHub Actions 通过 API Gateway 调用 AWS |
| 3 | AWS 保存密码 | Step 5：SSM Parameter Store SecureString |
| 4 | Lambda 触发 Jenkins | Step 6：Lambda 函数调用 Jenkins Remote API |
| 5 | GitHub Actions 构建流程 | Step 7：编写 workflow YAML |

---

## Step 1：创建 IAM 角色和权限

### 1.1 创建 Lambda 执行角色

AWS Console → IAM → Roles → Create Role：

- **信任实体**：Lambda
- **附加策略**：
  - `AWSLambdaBasicExecutionRole`（CloudWatch Logs 写日志）
  - 自定义内联策略 `jenkins-trigger-policy`：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "ssm:GetParameter",
            "Resource": "arn:aws:ssm:*:*:parameter/jenkins/*"
        }
    ]
}
```

> 💡 **注意**：这个策略只允许读取 `/jenkins/` 前缀下的参数，最小权限原则。

- **角色名**：`lambda-jenkins-trigger-role`

### 1.2 创建 GitHub Actions 用的 IAM 用户

IAM → Users → Create User：

- **用户名**：`github-actions-caller`
- **不要勾选** "提供控制台访问"
- **附加策略** → 创建内联策略 `invoke-api-gateway`：

```json
{
    "Version": "2012-10-17",
    "Statement": [
        {
            "Effect": "Allow",
            "Action": "execute-api:Invoke",
            "Resource": "arn:aws:execute-api:*:*:*/*/POST/*"
        }
    ]
}
```

- 创建后生成 **Access Key ID + Secret Access Key**，保存下来（后面配到 GitHub Secrets）

> ⚠️ **安全要点**：这个 IAM 用户**只能**调用 API Gateway，不能做任何其他操作。生产环境中还可以加上 SourceIP 条件限制。

---

## Step 2：启动 EC2 并安装 Jenkins

### 2.1 创建安全组

EC2 → Security Groups → Create：

| 规则 | 类型 | 端口 | 来源 | 说明 |
|------|------|------|------|------|
| 入站 1 | SSH | 22 | 你的 IP/32 | 运维 |
| 入站 2 | Custom TCP | 8080 | 0.0.0.0/0 | Jenkins Web UI（临时，后面要收紧） |
| 入站 3 | Custom TCP | 8080 | Lambda 的 VPC CIDR | Jenkins API 调用 |

> 💡 **安全组概念**：安全组是实例级的"防火墙"。入站规则控制谁能访问你，出站规则控制你能访问谁。默认所有出站放行、所有入站拒绝。

### 2.2 启动 EC2

- **AMI**：Amazon Linux 2023（免费套餐合格）
- **实例类型**：t2.micro（免费套餐，1 vCPU / 1 GB 内存）
- **密钥对**：创建新密钥 `jenkins-key`，下载 `.pem` 文件
- **安全组**：选上一步创建的
- **存储**：20 GB gp2（免费套餐上限 30 GB）

### 2.3 安装 Jenkins

SSH 登录后执行：

```bash
# 1. 更新系统
sudo dnf update -y

# 2. 安装 Java 17（Jenkins 需要）
sudo dnf install java-17-amazon-corretto -y
java -version

# 3. 添加 Jenkins 仓库
sudo wget -O /etc/yum.repos.d/jenkins.repo \
    https://pkg.jenkins.io/redhat-stable/jenkins.repo
sudo rpm --import https://pkg.jenkins.io/redhat-stable/jenkins.io-2023.key

# 4. 安装 Jenkins
sudo dnf install jenkins -y

# 5. 启动 Jenkins
sudo systemctl enable jenkins
sudo systemctl start jenkins
sudo systemctl status jenkins

# 6. 初始密码
sudo cat /var/lib/jenkins/secrets/initialAdminPassword
```

### 2.4 访问 Jenkins

浏览器打开 `http://<EC2-PUBLIC-IP>:8080`：

1. 输入初始密码
2. 选择 "安装推荐插件"
3. 创建管理员用户（记住账号密码）
4. 保持默认 URL，完成安装

> ⚠️ **t2.micro 内存不足提示**：如果 Jenkins 启动很慢或被 kill，创建 swap：
> ```bash
> sudo dd if=/dev/zero of=/swapfile bs=128M count=16
> sudo chmod 600 /swapfile
> sudo mkswap /swapfile
> sudo swapon /swapfile
> # 永久生效：
> echo '/swapfile swap swap defaults 0 0' | sudo tee -a /etc/fstab
> ```

---

## Step 3：配置 Jenkins API Token（密钥配置）

这是 **知识点 1（Jenkins 密钥配置）** 的核心。

### 3.1 创建 API Token

Jenkins → 右上角用户名 → Configure → API Token → Add new Token：

- **名称**：`lambda-trigger`
- 点击 Generate，**立即复制 Token**（只显示一次）

### 3.2 获取 Crumb（CSRF 保护）

Jenkins 默认开启 CSRF 保护，API 调用需要先获取 Crumb：

```bash
# 在 EC2 上测试
curl -u "admin:<API_TOKEN>" \
    "http://localhost:8080/crumbIssuer/api/json"
```

返回：
```json
{
    "_class": "hudson.security.csrf.DefaultCrumbIssuer",
    "crumb": "xxxxxxxxxxxxxxxxxxxxxxxxx",
    "crumbRequestField": "Jenkins-Crumb"
}
```

> 💡 **Crumb 是什么**：Jenkins 的安全机制，每次 POST/PUT 请求都要带上这个 crumb，防止跨站请求伪造攻击。

---

## Step 4：创建一个测试用的 Jenkins Job

创建一个最简单的 Freestyle 项目来验证链路：

1. Jenkins → New Item → 输入名称 `demo-build` → Freestyle project
2. 配置：
   - **Source Code Management**：Git，填你的 demo 仓库地址
   - **Build Steps** → Add build step → Execute shell：
     ```bash
     echo "=== Jenkins Build Started ==="
     echo "Build ID: $BUILD_ID"
     echo "Branch: ${GIT_BRANCH:-unknown}"
     echo "Commit: $(git log -1 --oneline 2>/dev/null || echo 'N/A')"
     echo "=== Building... ==="
     # 模拟构建过程
     sleep 5
     echo "=== Build Complete ==="
     ```
   - **Build Triggers**：勾选 "Trigger builds remotely (e.g., from scripts)"，填入 Token：`remote-trigger-token-123`
   - 保存

3. 手动点一次 "Build Now" 确认能跑通

---

## Step 5：AWS SSM Parameter Store 存储凭证（密码保存）

这是 **知识点 3（AWS 保存密码）** 的核心。

### 5.1 为什么不用 Secrets Manager

| | Secrets Manager | SSM Parameter Store (SecureString) |
|---|---|---|
| 费用 | 30 天试用后 $0.40/密钥/月 | **永久免费** |
| 自动轮转 | ✅ | ❌（需要自己写） |
| 跨账号共享 | ✅ | ❌ |
| 学习目的 | 界面和概念一样 | API 调用模式一样 |
| 适合本 Demo | 可以但不划算 | ✅ **推荐** |

### 5.2 存入凭证

AWS Console → Systems Manager → Parameter Store → Create Parameter：

| 参数名 | 类型 | 值 |
|--------|------|-----|
| `/jenkins/url` | String | `http://<EC2-PRIVATE-IP>:8080` |
| `/jenkins/username` | String | `admin` |
| `/jenkins/api-token` | SecureString | `(你的 API Token)` |
| `/jenkins/job-token` | SecureString | `remote-trigger-token-123` |

> 💡 **String vs SecureString**：SecureString 会用 KMS 加密存储，在 AWS Console 中默认隐藏，API 调用时自动解密。免费套餐包含 KMS 密钥。

### 5.3 验证（用 AWS CLI 在 EC2 上测试）

```bash
# String 类型直接返回
aws ssm get-parameter --name /jenkins/url --query Parameter.Value --output text

# SecureString 解密返回（需要 ssm:GetParameter 权限）
aws ssm get-parameter --name /jenkins/api-token --with-decryption --query Parameter.Value --output text
```

---

## Step 6：编写 Lambda 函数（Lambda 触发 Jenkins）

这是 **知识点 4（Lambda 触发 Jenkins）** 的核心，也是最关键的代码。

### 6.1 Lambda 函数代码

```python
import json
import os
import urllib.request
import urllib.error

import boto3

ssm = boto3.client("ssm")


def get_parameter(name, decrypt=True):
    """从 SSM Parameter Store 获取参数值"""
    resp = ssm.get_parameter(Name=name, WithDecryption=decrypt)
    return resp["Parameter"]["Value"]


def get_jenkins_crumb(jenkins_url, username, api_token):
    """获取 Jenkins CSRF Crumb"""
    url = f"{jenkins_url}/crumbIssuer/api/json"
    req = urllib.request.Request(url)

    # HTTP Basic Auth
    credentials = f"{username}:{api_token}"
    encoded = __import__("base64").b64encode(credentials.encode()).decode()
    req.add_header("Authorization", f"Basic {encoded}")

    try:
        with urllib.request.urlopen(req) as resp:
            data = json.loads(resp.read().decode())
            return data["crumb"]
    except urllib.error.HTTPError as e:
        print(f"获取 crumb 失败: {e.code} {e.reason}")
        raise


def trigger_jenkins_build(jenkins_url, username, api_token, job_name, job_token, params=None):
    """
    触发 Jenkins Job 构建

    调用格式（无参数）:
        POST /job/{job_name}/build?token={job_token}

    或带参数:
        POST /job/{job_name}/buildWithParameters?token={job_token}&PARAM1=val1
    """
    crumb = get_jenkins_crumb(jenkins_url, username, api_token)

    if params:
        param_str = "&".join(f"{k}={v}" for k, v in params.items())
        url = f"{jenkins_url}/job/{job_name}/buildWithParameters?token={job_token}&{param_str}"
    else:
        url = f"{jenkins_url}/job/{job_name}/build?token={job_token}"

    data = json.dumps({}).encode("utf-8")
    req = urllib.request.Request(url, data=data, method="POST")

    credentials = f"{username}:{api_token}"
    encoded = __import__("base64").b64encode(credentials.encode()).decode()
    req.add_header("Authorization", f"Basic {encoded}")
    req.add_header(crumb["crumbRequestField"], crumb["crumb"])

    try:
        with urllib.request.urlopen(req) as resp:
            # build 触发成功返回 201，Location header 包含队列 URL
            return {
                "statusCode": resp.status,
                "queueLocation": resp.headers.get("Location", ""),
            }
    except urllib.error.HTTPError as e:
        error_body = e.read().decode() if e.fp else ""
        print(f"触发构建失败: {e.code} - {error_body}")
        raise


def lambda_handler(event, context):
    """
    Lambda 入口

    期望 event 格式（由 API Gateway 传入）:
    {
        "job_name": "demo-build",          // 必填
        "parameters": {                     // 可选
            "GIT_BRANCH": "main",
            "GIT_COMMIT": "abc123"
        }
    }
    """
    print(f"收到事件: {json.dumps(event)}")

    # 从 SSM 获取所有配置
    jenkins_url = get_parameter("/jenkins/url", decrypt=False)
    username = get_parameter("/jenkins/username", decrypt=False)
    api_token = get_parameter("/jenkins/api-token", decrypt=True)
    job_token = get_parameter("/jenkins/job-token", decrypt=True)

    # 从请求中提取参数
    body = event
    if "body" in event and event["body"]:
        body = json.loads(event["body"]) if isinstance(event["body"], str) else event["body"]

    job_name = body.get("job_name", "demo-build")
    params = body.get("parameters", {})

    try:
        result = trigger_jenkins_build(
            jenkins_url=jenkins_url,
            username=username,
            api_token=api_token,
            job_name=job_name,
            job_token=job_token,
            params=params,
        )

        return {
            "statusCode": 200,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({
                "message": f"Jenkins job '{job_name}' 已触发",
                "queueUrl": result["queueLocation"],
            }),
        }
    except Exception as e:
        return {
            "statusCode": 500,
            "headers": {"Content-Type": "application/json"},
            "body": json.dumps({"error": str(e)}),
        }


# Lambda 执行角色需要的权限：
# - ssm:GetParameter (对 /jenkins/* 路径)
# - logs:CreateLogGroup, logs:CreateLogStream, logs:PutLogEvents
```

### 6.2 创建 Lambda 函数

AWS Console → Lambda → Create Function：

- **名称**：`jenkins-trigger`
- **运行时**：Python 3.11
- **架构**：x86_64
- **执行角色**：选 Step 1 创建的 `lambda-jenkins-trigger-role`
- 粘贴上面的代码
- **超时时间**：改为 30 秒（默认 3 秒不够）
- **环境变量**（可选，也可以纯靠 SSM）：

### 6.3 创建 API Gateway 触发器

在 Lambda 页面 → Add trigger → API Gateway：

- **API 类型**：REST API（不是 HTTP API）
- **安全**：Open（暂时，后面可以加 API Key）
- 创建后你会得到一个 URL：`https://xxxxxxxxxx.execute-api.us-east-1.amazonaws.com/default/jenkins-trigger`

### 6.4 测试 Lambda

在 Lambda Console → Test → 输入测试事件：

```json
{
    "job_name": "demo-build",
    "parameters": {
        "GIT_BRANCH": "main"
    }
}
```

点击 Test，观察返回值。然后去 Jenkins 看是否出现了新的构建。

---

## Step 7：编写 GitHub Actions Workflow（构建流程）

这是 **知识点 5（GitHub Actions 编写构建流程）** 的核心。

### 7.1 准备 Demo 仓库

创建一个简单的 demo 项目（以 Node.js 为例，其他语言同理）：

```
demo-repo/
├── .github/
│   └── workflows/
│       └── ci.yml          # GitHub Actions workflow
├── src/
│   └── index.js
├── package.json
└── README.md
```

### 7.2 配置 GitHub Secrets

GitHub 仓库 → Settings → Secrets and variables → Actions → New repository secret：

| Secret 名 | 值 |
|-----------|-----|
| `AWS_ACCESS_KEY_ID` | Step 1 创建的 IAM 用户的 Access Key |
| `AWS_SECRET_ACCESS_KEY` | 对应的 Secret Key |
| `API_GATEWAY_URL` | API Gateway 的完整 URL |

> 💡 **GitHub Secrets 本身就是密钥管理**：这里的 Secret 在 workflow log 中会自动脱敏（显示为 `***`），即使在日志中打印也不会泄露。

### 7.3 Workflow 文件

`.github/workflows/ci.yml`：

```yaml
name: CI Pipeline

on:
  push:
    branches: [ main, develop ]
  pull_request:
    branches: [ main ]

jobs:
  # ========== Job 1：构建和测试 ==========
  build-and-test:
    runs-on: ubuntu-latest
    steps:
      - name: Checkout 代码
        uses: actions/checkout@v4

      - name: Setup Node.js
        uses: actions/setup-node@v4
        with:
          node-version: "20"
          cache: "npm"

      - name: 安装依赖
        run: npm ci

      - name: 运行 Lint
        run: npm run lint

      - name: 运行测试
        run: npm test

      - name: 构建
        run: npm run build

  # ========== Job 2：触发 Jenkins ==========
  # 只在 push 事件时触发（PR 不需要触发 Jenkins）
  trigger-jenkins:
    needs: build-and-test        # 只有 Job 1 成功后才执行
    if: github.event_name == 'push'
    runs-on: ubuntu-latest
    steps:
      - name: 获取提交信息
        id: commit-info
        run: |
          echo "branch=${GITHUB_REF#refs/heads/}" >> $GITHUB_OUTPUT
          echo "sha=${GITHUB_SHA::7}" >> $GITHUB_OUTPUT
          echo "message=$(git log -1 --pretty=%B | head -1)" >> $GITHUB_OUTPUT

      - name: 调用 API Gateway → Lambda → Jenkins
        run: |
          RESPONSE=$(curl -s -w "\n%{http_code}" -X POST \
            "${{ secrets.API_GATEWAY_URL }}" \
            -H "Content-Type: application/json" \
            -d '{
              "job_name": "demo-build",
              "parameters": {
                "GIT_BRANCH": "${{ steps.commit-info.outputs.branch }}",
                "GIT_COMMIT": "${{ steps.commit-info.outputs.sha }}",
                "GIT_MESSAGE": "${{ steps.commit-info.outputs.message }}",
                "TRIGGERED_BY": "${{ github.actor }}"
              }
            }')

          HTTP_CODE=$(echo "$RESPONSE" | tail -1)
          BODY=$(echo "$RESPONSE" | sed '$d')

          echo "HTTP Status: $HTTP_CODE"
          echo "Response: $BODY"

          if [ "$HTTP_CODE" -ge 400 ]; then
            echo "::error::触发 Jenkins 失败 (HTTP $HTTP_CODE)"
            exit 1
          fi

          echo "✅ Jenkins 构建已触发"

  # ========== Job 3：构建状态检查 ==========
  check-jenkins-result:
    needs: trigger-jenkins
    runs-on: ubuntu-latest
    steps:
      - name: 等待并检查 Jenkins 构建结果（可选增强）
        run: |
          echo "Jenkins 构建已触发，去 http://<EC2-IP>:8080 查看构建结果"
          # 这里可以扩展：轮询 Jenkins API 等待构建完成
```

> 💡 **Workflow 设计要点**：
> - `needs: build-and-test` —— Job 依赖，确保测试通过才触发 Jenkins
> - `if: github.event_name == 'push'` —— 只在 push 时触发，PR 不开 Jenkins
> - `${{ secrets.XXX }}` —— 敏感信息通过 GitHub Secrets 注入，不在代码里硬编码

---

## Step 8：完整链路测试

### 8.1 端到端验证清单

```
[ ] git push 后 GitHub Actions 自动启动
[ ] build-and-test job 通过
[ ] trigger-jenkins job 调用 API Gateway 返回 200
[ ] Lambda 日志（CloudWatch）显示正确读取了 SSM 参数
[ ] Jenkins 出现新的构建记录
[ ] 构建参数（分支名、commit SHA）正确传递
```

### 8.2 调试技巧

| 问题 | 排查方法 |
|------|---------|
| GitHub Actions 报 HTTP 403 | 检查 IAM 用户权限 `execute-api:Invoke` 是否正确 |
| Lambda 报 SSM 权限拒绝 | 检查 Lambda 角色是否有 `ssm:GetParameter` |
| Lambda 超时 | 检查超时设置是否 ≥ 30s，Jenkins EC2 是否可达 |
| Lambda 能调通但 Jenkins 没构建 | 检查 Job Token 是否匹配，Crumb 是否正确获取 |
| API Gateway 报 500 | 去 CloudWatch → Log Groups → `/aws/lambda/jenkins-trigger` 看日志 |

### 8.3 网络连通性检查清单

```
Lambda 能否访问 Jenkins
    │
    ├── Jenkins 在公网？→ 安全组 8080 端口要对 Lambda 出口 IP 放行（或临时 0.0.0.0/0）
    │
    └── Jenkins 在 VPC 内网？→ Lambda 需要配置 VPC 访问（选 Jenkins 所在的 VPC 和子网）
```

对于本 Demo，最简单的方式是：
- Lambda 默认有公网出口
- Jenkins EC2 安全组临时对 `0.0.0.0/0` 开放 8080（学习用途可以接受）
- Lambda 用 EC2 的**公网 IP** 调用 Jenkins

---

## 架构总结

```
                       知识点 2：webhook + aws 通信
                       ═══════════════════════════
                       GitHub Actions workflow
                       (ci.yml 中的 trigger-jenkins job)
                               │
                               │ HTTPS POST (AWS Signature V4 或 API Key)
                               ▼
                       API Gateway REST API
                       (对外暴露端点)
                               │
                               │ AWS 内部调用
                               ▼
  知识点 3                知识点 4
  ═══════                ═══════════════════
  SSM Parameter          Lambda (jenkins-trigger)
  Store                  ① 从 SSM 取凭证
  /jenkins/api-token     ② 获取 Jenkins Crumb
  /jenkins/job-token     ③ POST 触发 Jenkins Job
      │                        │
      │ 取密码                  │ HTTP Basic Auth + Crumb
      └────────────────────────┤
                               ▼
                       知识点 1
                       ════════
                       Jenkins (EC2)
                       - API Token 认证
                       - CSRF Crumb 保护
                       - Remote Trigger Token
                               │
                               ▼
                       构建产物 → S3（可选）
```

---

## 知识点速查表

| # | 知识点 | 核心命令 / 配置 | 文件 / 位置 |
|---|--------|----------------|-------------|
| 1 | Jenkins 密钥配置 | Manage Jenkins → Configure → API Token | Jenkins Web UI |
| 2 | Webhook × AWS 通信 | `curl -X POST <API_GATEWAY_URL>` | `.github/workflows/ci.yml` |
| 3 | AWS 密码存储 | `aws ssm get-parameter --with-decryption` | AWS SSM Parameter Store |
| 4 | Lambda 触发 Jenkins | `urllib.request.urlopen(POST /job/xxx/build)` | Lambda 函数代码 |
| 5 | GitHub Actions 流程 | `on: push → jobs → steps` | `.github/workflows/ci.yml` |

---

## 清理资源（防止意外扣费）

学完后记得清理：

```bash
# 1. 终止 EC2
aws ec2 terminate-instances --instance-ids <instance-id>

# 2. 删除 Lambda
aws lambda delete-function --function-name jenkins-trigger

# 3. 删除 API Gateway（在 Console 中操作）

# 4. 删除 SSM 参数
aws ssm delete-parameter --name /jenkins/url
aws ssm delete-parameter --name /jenkins/username
aws ssm delete-parameter --name /jenkins/api-token
aws ssm delete-parameter --name /jenkins/job-token

# 5. 删除 IAM 角色和用户
aws iam delete-role --role-name lambda-jenkins-trigger-role
aws iam delete-user --user-name github-actions-caller
```

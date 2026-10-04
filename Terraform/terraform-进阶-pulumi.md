# Pulumi 入门指南 —— 用编程语言写 IaC

> **前置知识**：建议先完成 [Terraform 入门指南](terraform-入门指南.md)，理解 IaC 的核心概念（资源、状态、provider）后，再学 Pulumi 会更容易。本文的代码示例会与 Terraform 的对应写法做对比。
>
> **一句话**：不用学 HCL，用你会的 TypeScript/Python/Go 来写基础设施代码。

---

## 1️⃣ 搭建学习阶梯

> 将 Pulumi 技能拆解为 5 个递进等级，每个等级标注了**掌握内容**、**常见错误**和**进阶标准**。按顺序学习，不要跳级。

---

### Lv 1 —— 认知层：Pulumi 是什么？

**平时要做什么**
- 理解 HCL 的局限性（无循环、无 if-else、调试困难、无法复用已有代码）
- 掌握 Pulumi 的核心理念：用编程语言写 IaC
- 理解 Pulumi 与 Terraform 的本质区别（语法不同，但声明式 IaC 思想相同）
- 知道 Pulumi 支持的语言和对应的适用场景

**HCL 的局限——Pulumi 的解法**

```hcl
# HCL 的局限

# ❌ 不能写 for 循环遍历复杂数据结构
resource "aws_subnet" "main" {
  count = 3
  # 想根据一个 map 动态决定每个子网的配置？做不到！
}

# ❌ 没有 if-else 条件判断
# 只能靠 count 和三元表达式变通，可读性差

# ❌ 没有函数复用
# 想封装一段通用逻辑？只能拆成 module，成本高

# ❌ 没有调试工具
# terraform console 只是命令行 REPL，不支持断点

# ❌ 不能复用已有的业务逻辑代码
# 业务代码里已经有的配置常量、工具函数，HCL 里用不上
```

**Pulumi 的做法**：用**你已经会用的编程语言**来写 IaC。

```typescript
// TypeScript 写 IaC —— 熟悉的语法，完整的 IDE 支持
const instanceType = env === "prod" ? "t3.large" : "t3.micro";  // if
const subnets = azs.map((az, i) => ({ az, cidr: `10.0.${i}.0/24` }));  // for
import { getVpcConfig } from "./shared/config";  // 复用已有代码
console.log("Creating VPC:", vpcConfig);  // 调试
```

**常见错误**
- ❌ 以为 Pulumi 是"Terraform 的替代品"—— 核心思想相同（声明式 IaC），只是表达方式不同
- ❌ 以为学了 Pulumi 就不用学 IaC 概念 —— Pulumi 只是换语法，资源模型和 Terraform 完全一致
- ❌ 选择不熟悉的语言学 Pulumi —— 用你最强的语言，不要为了 Pulumi 学一门新语言

**进阶标准**：你能用一句话解释 Pulumi 和 Terraform 的根本区别（语法 vs 语法，而不是能力 vs 能力）。

---

### Lv 2 —— 操作层：安装并创建第一个资源

**平时要做什么**
- 安装 Pulumi CLI 并登录（Pulumi Cloud 或本地模式）
- 使用 `pulumi new` 创建项目
- 编写 TypeScript 代码创建 S3 存储桶
- 掌握 `pulumi up / preview / destroy / stack` 基本命令

**安装**

```bash
# macOS
brew install pulumi

# Linux / Windows 从官网下载
# https://www.pulumi.com/docs/install/

pulumi version

# 登录 Pulumi Cloud（用于存储 state，免费层够用）
pulumi login

# 如果不想用云端，也可以用本地文件
pulumi login --local
```

**第一个例子：创建 S3 存储桶**

```bash
# 创建新项目
mkdir my-first-pulumi && cd my-first-pulumi
pulumi new aws-typescript
# 生成的文件：
# index.ts          ← 主代码文件（类似 Terraform 的 main.tf）
# Pulumi.yaml       ← 项目元数据（项目名、运行时）
# Pulumi.dev.yaml   ← 环境配置（类似 Terraform 的 terraform.tfvars）
```

```typescript
// index.ts
import * as aws from "@pulumi/aws";
import * as pulumi from "@pulumi/pulumi";

const config = new pulumi.Config();
const environment = config.get("environment") || "dev";
const bucketName = `my-app-storage-${environment}-${Date.now()}`;

// 创建 S3 存储桶
const bucket = new aws.s3.Bucket("my-bucket", {
  bucket: bucketName,
  acl: environment === "prod" ? "private" : "public-read",  // 三元表达式
  tags: {
    Name: bucketName,
    Environment: environment,
    ManagedBy: "pulumi",
  },
});

// 条件逻辑直接用 if
if (environment === "prod") {
  const bucketPolicy = new aws.s3.BucketPolicy("enforce-https", {
    bucket: bucket.bucket,
    policy: bucket.arn.apply(arn =>
      JSON.stringify({
        Version: "2012-10-17",
        Statement: [
          {
            Effect: "Deny",
            Principal: "*",
            Action: "s3:*",
            Resource: [`${arn}/*`, arn],
            Condition: {
              Bool: { "aws:SecureTransport": "false" },
            },
          },
        ],
      })
    ),
  });
}

export const bucketName = bucket.bucket;
export const bucketArn = bucket.arn;
```

**部署**

```bash
pulumi stack init dev        # 创建堆栈（类似 terraform workspace）
pulumi config set environment dev  # 设置配置
pulumi preview               # 预览（类似 terraform plan）
pulumi up                    # 部署（类似 terraform apply）
pulumi stack output          # 查看输出
pulumi destroy               # 删除所有资源
pulumi stack rm dev          # 删除堆栈
```

**常见错误**
- ❌ 安装完 Pulumi 没有 `pulumi login` —— 无法保存 state
- ❌ 在一个目录里反复 `pulumi new` —— 会覆盖已有文件
- ❌ 忘记设置 AWS 凭证 —— Pulumi 和 Terraform 一样，底层调用 AWS SDK
- ❌ `pulumi up` 前不 `preview` —— 尤其不熟悉时，preview 可以提前发现错误

**进阶标准**：你能完成从空目录到 S3 桶创建再到销毁的完整流程，且理解每一步对应的 Terraform 概念。

---

### Lv 3 —— 进阶层：编程语言的真正威力

**平时要做什么**
- 用 `map` 和 `for` 批量创建子网
- 用 `if` 实现条件逻辑
- 理解 Pulumi 的 Output 机制
- 掌握 `apply` 和 `interpolate` 处理异步值

**VPC + EC2 完整示例**

```typescript
// vpc-ec2.ts
import * as aws from "@pulumi/aws";
import * as pulumi from "@pulumi/pulumi";

const config = new pulumi.Config();
const env = config.require("environment");

// ===== 创建 VPC =====
const vpc = new aws.ec2.Vpc("main", {
  cidrBlock: "10.0.0.0/16",
  enableDnsHostnames: true,
  tags: { Name: `${env}-vpc` },
});

// ===== 用 map 批量生成子网 =====
const azs = ["ap-northeast-1a", "ap-northeast-1c"];

const publicSubnets = azs.map((az, i) =>
  new aws.ec2.Subnet(`public-${i}`, {
    vpcId: vpc.id,
    cidrBlock: `10.0.${i}.0/24`,
    availabilityZone: az,
    mapPublicIpOnLaunch: true,
    tags: { Name: `${env}-public-${i}` },
  })
);

const privateSubnets = azs.map((az, i) =>
  new aws.ec2.Subnet(`private-${i}`, {
    vpcId: vpc.id,
    cidrBlock: `10.0.${10 + i}.0/24`,
    availabilityZone: az,
    tags: { Name: `${env}-private-${i}` },
  })
);

// ===== 创建安全组 =====
const sg = new aws.ec2.SecurityGroup("web-sg", {
  vpcId: vpc.id,
  description: "Allow HTTP and SSH",
  // ⚠️ SSH 开放到全互联网仅为演示，生产环境应限制来源 IP
  ingress: [
    { protocol: "tcp", fromPort: 22, toPort: 22, cidrBlocks: ["0.0.0.0/0"] },
    { protocol: "tcp", fromPort: 80, toPort: 80, cidrBlocks: ["0.0.0.0/0"] },
  ],
  egress: [{ protocol: "-1", fromPort: 0, toPort: 0, cidrBlocks: ["0.0.0.0/0"] }],
  tags: { Name: `${env}-web-sg` },
});

// ===== 获取 AMI（等价于 Terraform 的 data source）=====
const ami = aws.ec2.getAmiOutput({
  owners: ["amazon"],
  mostRecent: true,
  filters: [
    { name: "name", values: ["amzn2-ami-hvm-*-x86_64-gp2"] },
    { name: "state", values: ["available"] },
  ],
});

// ===== 创建 EC2（if 决定规格）=====
const instanceType = env === "prod" ? "t3.medium" : "t3.micro";

const server = new aws.ec2.Instance("web-server", {
  instanceType: instanceType,
  ami: ami.id,
  subnetId: publicSubnets[0].id,
  vpcSecurityGroupIds: [sg.id],
  associatePublicIpAddress: true,
  userData: `#!/bin/bash
echo "Hello from ${env} environment" > /var/www/html/index.html
yum install -y httpd
systemctl start httpd
systemctl enable httpd
`,
  tags: { Name: `${env}-web-server` },
});

export const vpcId = vpc.id;
export const publicIp = server.publicIp;
export const websiteUrl = pulumi.interpolate`http://${server.publicDns}`;
```

**理解 Output 机制**

```typescript
// 普通代码：不行！因为 vpc.id 是 Output<string>，不是 string
console.log(vpc.id);  // 会打印 "Output<pulumi...>"

// 正确做法 1：apply（类似 Promise.then）
vpc.id.apply(id => { console.log("真实的值是：", id); return id; });

// 正确做法 2：pulumi.interpolate（拼接字符串）
const fullName = pulumi.interpolate`${env}-vpc-${vpc.id}`;

// 正确做法 3：直接传给其他资源的参数（Pulumi 自动处理依赖）
new aws.ec2.Subnet("subnet", { vpcId: vpc.id, cidrBlock: "10.0.1.0/24" });
```

> Output 是 Pulumi 最需要理解的概念：资源是异步创建的，`vpc.id` 返回一个"未来的值"，Pulumi 会在正确的时间解开它。

**常见错误**
- ❌ 直接 `console.log(vpc.id)` 打印 Output 对象 —— 要用 `.apply()`
- ❌ 在普通函数里操作 Output 值但不返回 —— Output 值只在 apply 回调里可用
- ❌ 用 `for` 循环批量创建资源时不注意逻辑名称唯一性 —— 每个资源需要不同的逻辑名称
- ❌ 以为 `.map()` 返回的是普通数组 —— 返回的是 Output 数组，需要特殊处理

**进阶标准**：你能用 `map` 批量创建 3 个子网，并用 `if` 条件决定生产/非生产环境的配置差异。

---

### Lv 4 —— 对比层：Pulumi vs Terraform 全方位对比

| 对比维度 | Terraform | Pulumi |
|---------|-----------|--------|
| **语法** | HCL（DSL，得专门学） | TypeScript / Python / Go（你本来就会的） |
| **IDE 支持** | 有限（HCL 插件不完善） | 完整（VSCode 自动补全、跳转、重构全支持） |
| **循环逻辑** | `count` / `for_each` / `for` 表达式 | `for` / `map` / `filter` / `reduce` 任你用 |
| **条件逻辑** | `count` 变通 + 三元表达式 | `if` / `switch` / 三元，想怎么写就怎么写 |
| **代码复用** | Module（HCL 模块） | npm 包 / Python pip / Go Module |
| **错误处理** | plan 报错就停 | `try/catch` + 日志，可编程处理错误 |
| **调试** | `terraform console`（原始） | `console.log()` + 真实 debugger |
| **类型检查** | `terraform validate` | 编译时直接捕获类型错误 |
| **单元测试** | terratest（第三方工具） | Jest / Mocha / pytest，直接写 |
| **状态管理** | backend 配置（S3/Consul 等） | Pulumi Cloud（默认）或自托管 |

**常见错误**
- ❌ 以为 Pulumi 写了类型检查就自动部署成功 —— 类型检查只保证语法正确，不保证云 API 调用成功
- ❌ 忽略 Pulumi Cloud 的默认状态管理 —— 默认用云端，如果没登录会失败
- ❌ 用 Pulumi 写过于复杂的业务逻辑 —— IaC 代码不该处理业务逻辑，保持简单

**进阶标准**：你能根据自己的编程背景，判断是否值得从 Terraform 切换到 Pulumi。

---

### Lv 5 —— 生产层：什么时候选 Pulumi？

**推荐学 Pulumi**：
- 你有较强的编程背景（TypeScript / Python / Go）
- 你觉得 HCL 的语法限制让你烦躁
- 你的 Terraform 配置里充斥复杂的三元表达式和 for 循环
- 你的团队想用 GitOps 但不想再深入一门新语言

**不推荐 Pulumi**：
- 你是运维出身，对编程语言不太熟悉
- 团队共识是"HCL 就够了，不想引入 Node.js / Python 依赖"
- 你刚入门 IaC，先把 Terraform 的基础概念搞清楚

**常见错误**
- ❌ 在一个纯运维团队强推 Pulumi —— 得考虑团队的技术栈和接受度
- ❌ 所有基础设施全用 Pulumi 接管 —— 渐进式迁移，先试一个小项目
- ❌ 选团队不熟悉的语言写 Pulumi —— 学习 Pulumi 本身就有成本，别叠加语言成本

**进阶标准**：你能给一个同时犹豫 Terraform 和 Pulumi 的团队提供选型建议，说明理由。

---

> **一句话总结**：Terraform 和 Pulumi 的核心思想是一样的（声明式 IaC），只是表达方式不同。选哪个取决于你"用哪只手写字"更舒服。

---

## 2️⃣ 聚焦核心 20%

> Pulumi 中最重要的 20% 内容：**项目创建 + 资源声明 + 编程语言特性（map/if/Output）**。这 20% 覆盖了日常 80% 的使用场景。

---

| 次数 | 主题 | 核心内容 | 练习资源 | 复盘问题 |
|:----:|------|----------|----------|----------|
| 1 | 安装 + 创建项目 | 安装 Pulumi、`pulumi login`、`pulumi new`、文件结构 | 用 `pulumi new aws-typescript` 创建一个新项目 | `Pulumi.yaml` 和 `index.ts` 各有什么作用？ |
| 2 | 第一个 S3 桶 | 编写 Bucket 资源、`pulumi up`、`pulumi stack output` | 创建、查看、销毁一个 S3 桶 | `pulumi up` 和 `terraform apply` 的流程有什么异同？ |
| 3 | 条件逻辑 | `if` 做环境判断、三元表达式 | 创建一个 Bucket，prod 环境强制 HTTPS，dev 环境不强制 | 为什么编程语言的 `if` 比 Terraform 的 `count` 变通更易读？ |
| 4 | 批量创建资源 | `map` + `for` 批量创建子网、动态生成配置 | 用 TypeScript 的 `map` 在 3 个可用区各创建一个子网 | 用 Pulumi 创建 3 个子网和用 Terraform 的 `count` 有什么不同？ |
| 5 | VPC 完整示例 | VPC + Subnet + SecurityGroup + EC2 组合 | 部署完整的 VPC + EC2 示例 | EC2 的 `userData` 在 Pulumi 里怎么写？ |
| 6 | 理解 Output 机制 | Output 的本质、`.apply()`、`pulumi.interpolate` | 用 `.apply()` 打印 VPC 的真实 ID | 为什么 `vpc.id` 不是 string 而是 Output？ |
| 7 | Data Source | 用 `getAmiOutput` 获取 AMI ID（对比 Terraform data source） | 获取最新的 Amazon Linux AMI 并用于 EC2 | Pulumi 的 data source 和 Terraform 的 `data` 块有什么异同？ |
| 8 | 配置管理 | `pulumi config`、Config 对象、secrets 加密 | 用 config 管理数据库密码和 API Key | `pulumi config set --secret` 做了什么？ |
| 9 | 全面对比 | Pulumi vs Terraform 每个维度 | 画一张对比表 | 你的团队该选哪个？为什么？ |
| 10 | 综合实战 | 用 Pulumi 创建一套完整 Web 应用基础设施 | VPC + ALB + ECS + RDS 的完整 TypeScript 代码 | 如果 VPC 已存在，Pulumi 怎么引用已有资源？ |

> **学习建议**：每次学完后花 5 分钟回答"复盘问题"列。答不出来就回看，不往下走。

---

## 3️⃣ AI 考官测试

> 学完以上内容后，告诉 AI："开始 Pulumi 考官测试"，按以下流程进行：

**测试规则**
1. AI 一次只问 **一个问题**
2. 从**简单到困难**逐步深入
3. 你回答后，AI 立即给出**反馈** + **针对性讲解**

**测试范围**
- 第一轮：概念题（Pulumi 的核心思想是什么？Output 机制为什么存在？）
- 第二轮：操作题（写出从零创建一个 S3 桶的完整步骤）
- 第三轮：编码题（用 TypeScript 写一段代码：创建 3 个子网，并根据环境决定实例规格）
- 第四轮：选型题（"一个 5 人运维团队，TypeScript 经验不足，该选 Pulumi 还是 Terraform？"）

**启动方式**
> "开始 Pulumi 考官测试，从简单概念题开始。"

---

## 4️⃣ 制作速查表

### 速查表 A：基本操作

| 项目 | 内容 |
|------|------|
| **一句话** | Pulumi 让你用 TypeScript/Python/Go 写 IaC，享受编程语言的全部能力（循环、条件、调试、复用） |
| **核心命令** | `pulumi new` 创建项目 · `pulumi up` 部署 · `pulumi preview` 预览 · `pulumi destroy` 销毁 · `pulumi stack` 管理环境 |
| **真实案例** | 用 TypeScript 的 `azs.map()` 在 3 个可用区各创建一个子网，用 `if (env === "prod")` 决定是否强制 HTTPS |

**基本命令**

```bash
pulumi new aws-typescript    # 创建新项目
pulumi stack init dev        # 创建 dev 堆栈
pulumi stack select dev      # 切换到 dev
pulumi up                    # 部署
pulumi preview               # 预览
pulumi destroy               # 销毁所有资源
pulumi stack rm dev          # 删除堆栈
pulumi config set key value  # 设置配置
pulumi config                # 查看配置
pulumi stack output          # 查看输出
pulumi state list            # 查看 state 管理的资源
```

**常见错误**
- ❌ 直接 `console.log(vpc.id)` 打印 Output 对象 —— 用 `.apply()`
- ❌ 忘记 `pulumi login` —— 无法保存 state
- ❌ 目录里先有 index.ts 再 `pulumi new` —— 会覆盖已有文件
- ❌ 不 preview 直接 up —— preview 能提前发现错误

**检查清单**
- [ ] 是否已安装 Pulumi CLI？
- [ ] 是否已执行 `pulumi login`（Cloud 或 local）？
- [ ] 项目目录是否已用 `pulumi new` 初始化？
- [ ] AWS 凭证是否已配置？
- [ ] 是否执行了 `pulumi preview` 查看变更？
- [ ] Output 值是否用 `.apply()` 或 `interpolate` 正确处理？

**5 个自测题**
1. 创建一个新的 TypeScript Pulumi 项目用什么命令？
2. 部署和预览分别用什么命令？类似 Terraform 的什么命令？
3. 为什么不能直接 `console.log(vpc.id)`？
4. 如何创建 3 个在不同的可用区的子网？
5. 如何在 prod 环境下创建更大的 EC2 实例？

---

### 速查表 B：Output 与编程模式

| 项目 | 内容 |
|------|------|
| **一句话** | Output 是 Pulumi 中处理"未来值"的机制，类似 Promise，因为资源创建是异步的 |
| **三种用法** | `.apply(fn)` 获取真实值 · `pulumi.interpolate` 拼接字符串 · 直接传给其他资源的参数（自动解包）|

**常见错误**
- ❌ 在普通函数里操作 Output 但不返回 —— 值只在 apply 回调中可用
- ❌ 不理解 Output 和普通类型的区别 —— 没有 `.apply()` 只能看到一个包装对象
- ❌ 在 Output 回调里做副作用操作（写数据库）—— 回调可能被执行多次

**检查清单**
- [ ] 需要打印的值是否用了 `.apply()`？
- [ ] 字符串拼接是否用了 `interpolate`？
- [ ] 传给其他参数的值能否自动解包？如果不能，是否用了 `.apply()`？
- [ ] 条件逻辑是否用了 `if` 而不是 `count` 变通？

**5 个自测题**
1. Output 的设计目的是什么？
2. `.apply()` 和 `pulumi.interpolate` 分别用于什么场景？
3. 如何获取 VPC 的 ID 并打印到控制台？
4. `pulumi.up` 时 Output 没有被解开会发生什么？
5. 如果有 3 个子网，如何获取第一个子网的 ID 传给 EC2？

---

## 5️⃣ 筛选优质资源

| # | 资源 | 类型 | 难度 | 适合人群 | 怎么用 |
|:-:|------|------|:----:|----------|--------|
| 1 | [Pulumi 官方入门教程](https://www.pulumi.com/docs/get-started/) | 交互式教程 | ⭐⭐ | 零基础入门 | 跟着走一遍，选自己最熟悉的语言版本 |
| 2 | [Pulumi AWS 文档](https://www.pulumi.com/registry/packages/aws/) | 参考文档 | ⭐⭐ | 所有用户 | 不是从头读，是**查**：创建什么资源就搜什么 |
| 3 | [Pulumi Outputs 详解](https://www.pulumi.com/docs/concepts/inputs-outputs/) | 官方概念文档 | ⭐⭐⭐ | 被 Output 卡住的读者 | 学完 Lv 3 后精读，彻底理解 Output 机制 |
| 4 | [Pulumi 常见模式](https://www.pulumi.com/docs/guides/adopting/) | 实践指南 | ⭐⭐⭐ | 想从 Terraform 迁移的用户 | 读迁移指南 + 常见模式，了解最佳实践 |
| 5 | [Pulumi Examples 仓库](https://github.com/pulumi/examples) | 代码示例 | ⭐⭐ | 需要参考代码的读者 | 搜你的场景（vpc、eks、rds），直接抄示例改 |

**不推荐的资源**
- 过老的 Pulumi 博客文章（API 变化快，找最新文档）
- 非官方的视频教程（质量参差不齐，可能教错）

---

### 7 天学习路径

| 天 | 学习内容 | 使用资源 | 预计时间 |
|:--:|----------|----------|:--------:|
| Day 1 | 安装 + 创建项目（Lv 1-Lv 2） | 资源 #1 官方入门教程 | 1.5h |
| Day 2 | 第一个 S3 桶（Lv 2） | 资源 #2 AWS 文档 + 本文示例 | 2h |
| Day 3 | 条件逻辑 + 循环（Lv 3） | 本文示例 + 资源 #3 Output 详解 | 2h |
| Day 4 | VPC + EC2 完整示例（Lv 3） | 本文示例 + 资源 #5 GitHub Examples | 2h |
| Day 5 | Output 机制深入（Lv 3） | 资源 #3 Outputs 详解 | 1.5h |
| Day 6 | 对比 + 选型（Lv 4-Lv 5） | 本文对比表 | 1h |
| Day 7 | 综合实战 | 结合本文示例自行搭建 | 2h |

> **建议**：Day 1-2 上手操作，Day 3-5 深入语言特性，Day 6-7 对比和实战。

---

## 6️⃣ 费曼循环巩固

> 费曼学习法：**如果你不能简单地解释它，你就没有真正理解它。**

### 用 12 岁孩子能懂的话来说

**Pulumi 是什么？**

你有一个**积木说明书**（Terraform 的 HCL），说明书上有固定的格式教你写"创建 VPC"、"创建 S3"。但说明书本事有限——你想写"如果今天下雨就建一个小房子，晴天就建一个大房子"，说明书就做不到。

现在有人给了你一个**空白的笔记本**（Pulumi），并说："你用你本来就会的语文（TypeScript）来写说明！"

从此，你再也不用学那套说明书格式了。你想写条件？直接用"如果"。你想写循环？直接用"对于每一个"。你想测试？直接在笔记本上打个勾就可以了。

**Output 是什么？**

想象你在网上买了一个乐高套装。你下单（`new Vpc()`）后，它还没到货，但你有一个**取货码**（Output）。

你不能把这个取货码当乐高来玩——你得等它到了才能拼。但你可以在拿到乐高之前，先把**说明书**写好（`.apply()` 里的代码），等乐高一到，机器就自动按说明书执行。

**Pulumi 和 Terraform 的区别？**

Terraform 像是一个**专用画板**——你只能在它规定的地方画画，画笔只有几种颜色。

Pulumi 像是一个**白板+彩笔**——画板限制了你能画的东西，但白板上你想画什么就画什么。

但注意：这个白板最终还是用来画"基础设施图"的。你不能因为在白板上就画一朵花——Pulumi 的能力是让你画基础设施时更自由，而不是让你做基础设施之外的事。

### 你的费曼练习

现在轮到你。按以下步骤做：

1. **复述**：合上文档，用自己的话给一个会 Terraform 但没听过 Pulumi 的同事解释
2. **检查漏洞**：对照以下问题

**自检问题**
- ❓ Pulumi 解决了 HCL 的哪些局限？
- ❓ Output 是什么？为什么需要 Output？
- ❓ Pulumi 的 `pulumi up` 和 Terraform 的 `terraform apply` 本质上有什么不同？
- ❓ 如果你要用 Pulumi，你会选哪个语言？为什么？
- ❓ 什么情况下你仍然应该选 Terraform 而不是 Pulumi？

### 循环方法

```
Round 1: 你用自己的话说 → 我发现漏洞并指出 → 你去查资料修正
Round 2: 你重新说（这次更准确） → 我再指出改进点
Round 3: 你再说 → 漏洞越来越少 → 你真正掌握了 ✅
```

开始费曼循环，告诉我："开始 Pulumi 费曼循环，我先用自己的话解释 Pulumi 是什么。"

---

> **最后说一句**：Pulumi 最大的价值不是"换一个语法"，而是**让你可以像写应用代码一样写 IaC**。但记住：它仍然是 IaC——声明式、面向资源、幂等——这一点和 Terraform 没有区别。

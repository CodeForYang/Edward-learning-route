# Pulumi 入门指南 —— 用编程语言写 IaC

> **前置知识**：建议先完成 [Terraform 入门指南](terraform-入门指南.md)，理解 IaC 的核心概念（资源、状态、provider）后，再学 Pulumi 会更容易。本文的代码示例会与 Terraform 的对应写法做对比。
>
> **一句话**：不用学 HCL，用你会的 TypeScript/Python/Go 来写基础设施代码。

---

## 1. 它解决了什么问题？

HCL 是 Terraform 用的 DSL（领域特定语言），它能做的事情有限：

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

// 条件判断？直接用 if
const instanceType = env === "prod" ? "t3.large" : "t3.micro";

// 循环？直接用 for
const subnets = azs.map((az, i) => ({
  az,
  cidr: `10.0.${i}.0/24`,
}));

// 复用已有代码？直接 import
import { getVpcConfig } from "./shared/config";

// 调试？console.log + 真实的调试器
console.log("Creating VPC with config:", vpcConfig);
```

---

## 2. 安装

```bash
# macOS
brew install pulumi

# Linux / Windows 从官网下载
# https://www.pulumi.com/docs/install/

# 验证
pulumi version

# 登录 Pulumi Cloud（用于存储 state，免费层够用）
pulumi login

# 如果不想用云端，也可以用本地文件
pulumi login --local
```

---

## 3. 第一个例子：用 TypeScript 创建 S3 存储桶

### 3.1 创建新项目

```bash
# 创建项目目录
mkdir my-first-pulumi && cd my-first-pulumi

# 初始化一个 TypeScript 项目
# 会引导你选择：云厂商（aws）、语言（typescript）、项目名
pulumi new aws-typescript

# 生成的文件说明：
#
# index.ts          ← 主代码文件（类似 Terraform 的 main.tf）
# Pulumi.yaml       ← 项目元数据（项目名、运行时）
# Pulumi.dev.yaml   ← 环境配置（类似 Terraform 的 terraform.tfvars）
# package.json      ← Node.js 项目依赖
# tsconfig.json     ← TypeScript 配置
```

### 3.2 写代码

`index.ts` —— 和 Terraform 的 `main.tf` 对应：

```typescript
// index.ts
// 相当于 Terraform 的 main.tf + variables.tf + outputs.tf

import * as aws from "@pulumi/aws";
import * as pulumi from "@pulumi/pulumi";

// ===== 获取配置（类似 Terraform 的 variable 块）=====

const config = new pulumi.Config();

// 从 pulumi config get 或环境变量读取
const environment = config.get("environment") || "dev";

// 用 TypeScript 的常量生成动态资源名
const bucketName = `my-app-storage-${environment}-${Date.now()}`;

// ===== 创建资源（类似 Terraform 的 resource 块）=====

// 创建 S3 存储桶
// new aws.s3.Bucket("逻辑名称", { 配置 })
const bucket = new aws.s3.Bucket("my-bucket", {
  bucket: bucketName,

  // 直接用三元表达式，不用 count + element()
  acl: environment === "prod" ? "private" : "public-read",

  tags: {
    Name: bucketName,
    Environment: environment,
    ManagedBy: "pulumi",
  },
});

// ---- 条件逻辑直接用 if ----
// 只有生产环境才加访问策略：强制 HTTPS
if (environment === "prod") {
  // 给桶添加加密策略
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

// ===== 输出（类似 Terraform 的 output 块）=====
export const bucketName = bucket.bucket;
export const bucketArn = bucket.arn;
export const environmentUsed = environment;
```

### 3.3 部署

```bash
# 1) 创建堆栈（类似 Terraform workspace）
pulumi stack init dev

# 2) 设置配置参数（类似 terraform.tfvars）
pulumi config set environment dev

# 3) 预览（类似 terraform plan）
pulumi preview

# 4) 部署（类似 terraform apply）
pulumi up
# 会先显示变更清单，确认 Y 后执行
# 后续修改后，也只执行 pulumi up

# 5) 查看输出
pulumi stack output

# 6) 清理
pulumi destroy    # 删除所有资源
pulumi stack rm dev  # 删除堆栈
```

---

## 4. 更复杂的例子：VPC + EC2

这个例子对比 Terraform 和 Pulumi 的写法差异：

```typescript
// vpc-ec2.ts

import * as aws from "@pulumi/aws";
import * as pulumi from "@pulumi/pulumi";

const config = new pulumi.Config();
const env = config.require("environment"); // require = 必填，没有值就报错

// ===== 创建 VPC =====
// 和 Terraform 几乎一一对应，只是语法不同
const vpc = new aws.ec2.Vpc("main", {
  cidrBlock: "10.0.0.0/16",
  enableDnsHostnames: true,
  tags: { Name: `${env}-vpc` },
});

// ===== 用 map 批量生成子网 =====
// 对比 Terraform 的 count + cidrsubnet()
// TypeScript 的数组方法更直观

const azs = ["ap-northeast-1a", "ap-northeast-1c"];

// 公有子网
const publicSubnets = azs.map((az, i) =>
  new aws.ec2.Subnet(`public-${i}`, {
    vpcId: vpc.id,
    cidrBlock: `10.0.${i}.0/24`,
    availabilityZone: az,
    mapPublicIpOnLaunch: true,
    tags: { Name: `${env}-public-${i}` },
  })
);

// 私有子网
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

  // 入站规则：SSH + HTTP + HTTPS
  // ⚠️ SSH（22 端口）开放到全互联网仅为演示，生产环境应限制来源 IP
  ingress: [
    { protocol: "tcp", fromPort: 22, toPort: 22, cidrBlocks: ["0.0.0.0/0"] },
    { protocol: "tcp", fromPort: 80, toPort: 80, cidrBlocks: ["0.0.0.0/0"] },
    { protocol: "tcp", fromPort: 443, toPort: 443, cidrBlocks: ["0.0.0.0/0"] },
  ],

  // 出站规则：全部放行
  egress: [
    { protocol: "-1", fromPort: 0, toPort: 0, cidrBlocks: ["0.0.0.0/0"] },
  ],

  tags: { Name: `${env}-web-sg` },
});

// ===== 获取最新的 Amazon Linux 2 AMI =====
// 等价于 Terraform 的 data "aws_ami" "amazon_linux" {...}
const ami = aws.ec2.getAmiOutput({
  owners: ["amazon"],
  mostRecent: true,
  filters: [
    { name: "name", values: ["amzn2-ami-hvm-*-x86_64-gp2"] },
    { name: "state", values: ["available"] },
  ],
});

// ===== 创建 EC2 实例 =====
// 根据环境决定规格 —— if 条件的天然表达
const instanceType = env === "prod" ? "t3.medium" : "t3.micro";

const server = new aws.ec2.Instance("web-server", {
  instanceType: instanceType,
  ami: ami.id,
  subnetId: publicSubnets[0].id,  // 直接通过数组索引引用
  vpcSecurityGroupIds: [sg.id],
  associatePublicIpAddress: true,

  // 启动脚本
  userData: `#!/bin/bash
echo "Hello from ${env} environment" > /var/www/html/index.html
yum install -y httpd
systemctl start httpd
systemctl enable httpd
`,

  tags: { Name: `${env}-web-server` },
});

// ===== 输出 =====
export const vpcId = vpc.id;
export const publicIp = server.publicIp;
export const websiteUrl = pulumi.interpolate`http://${server.publicDns}`;
```

---

## 5. Pulumi vs Terraform 全方位对比

| 对比维度 | Terraform | Pulumi |
|---------|-----------|--------|
| **语法** | HCL（DSL，得专门学） | TypeScript / Python / Go 等（你本来就会的） |
| **IDE 支持** | 有限（HCL 插件不完善） | 完整（VSCode 自动补全、跳转、重构全支持） |
| **循环逻辑** | `count` / `for_each` / `for` 表达式 | `for` / `map` / `filter` / `reduce` 任你用 |
| **条件逻辑** | `count` 变通 + 三元表达式 | `if` / `switch` / 三元，想怎么写就怎么写 |
| **代码复用** | Module（HCL 模块） | npm 包 / Python pip / Go Module |
| **错误处理** | plan 报错就停 | `try/catch` + 日志，可编程处理错误 |
| **调试** | `terraform console`（原始） | `console.log()` + 真实 debugger |
| **类型检查** | `terraform validate` | 编译时直接捕获类型错误 |
| **单元测试** | terratest（第三方工具） | Jest / Mocha / pytest，直接写 |
| **状态管理** | backend 配置（S3/Consul 等） | Pulumi Cloud（默认）或自托管 |

---

## 6. Pulumi 的"输出"机制（需要理解的概念）

Pulumi 中的 `Output<T>` 是一个特殊概念，类似于 Promise：

```typescript
// 普通代码：不行，因为 bucket.id 是 Output<string>
const vpcId = vpc.id;           // Output<string>，不是 string
console.log(vpcId);             // 会打印 "Output<pulumi...>"，而不是 vpc-xxx

// 正确做法 1：apply（类似 Promise.then）
const vpcIdValue = vpc.id.apply(id => {
  console.log("真实的值是：", id);  // 这里才是真正的字符串
  return id;
});

// 正确做法 2：使用 pulumi.interpolate（拼接字符串）
const fullName = pulumi.interpolate`${env}-vpc-${vpc.id}`;

// 正确做法 3：直接传给其他资源的参数（Pulumi 会自动解开）
new aws.ec2.Subnet("subnet", {
  vpcId: vpc.id,       // 虽然 vpc.id 是 Output，但 Pulumi 会处理依赖
  cidrBlock: "10.0.1.0/24",
});
```

**为什么这样设计？** 因为资源是异步创建的。当你在写 `vpc.id` 的时候，那个 VPC 可能还没创建完，所以它返回一个"未来的值"（Output），Pulumi 自己会在正确的时间解开它。

---

## 7. 常见命令速查

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

---

## 8. 什么时候学 Pulumi？

**推荐学**：
- 你有较强的编程背景（TypeScript / Python / Go）
- 你觉得 HCL 的语法限制让你烦躁
- 你的 Terraform 配置里充斥复杂的三元表达式和 for 循环
- 你的团队想用 GitOps 但不想再深入一门新语言

**不推荐**：
- 你是运维出身，对编程语言不太熟悉
- 团队共识是"HCL 就够了，不想引入 Node.js / Python 依赖"
- 你刚入门 IaC，先把 Terraform 的基础概念搞清楚

---

> **一句话总结**：Terraform 和 Pulumi 的核心思想是一样的（声明式 IaC），只是表达方式不同。选哪个取决于你"用哪只手写字"更舒服。

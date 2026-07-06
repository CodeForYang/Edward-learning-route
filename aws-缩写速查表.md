# ☁️ AWS 缩写速查表

> 面向 Terraform / 云运维场景，列出最常见的 AWS 服务缩写、全称和作用。
> 适合：学习 Terraform、阅读 AWS 文档、面试准备。

---

## 📌 核心基础设施

| 缩写 | 全称 | 一句话作用 |
|------|------|-----------|
| **EC2** | Elastic Compute Cloud | 虚拟服务器，最核心的计算资源 |
| **VPC** | Virtual Private Cloud | 虚拟私有网络，隔离的云上网络环境 |
| **S3** | Simple Storage Service | 对象存储，存文件/图片/备份/静态网站 |
| **EBS** | Elastic Block Store | 块存储（虚拟硬盘），挂载给 EC2 用 |
| **AMI** | Amazon Machine Image | EC2 的镜像模板（含 OS + 预装软件） |
| **ENI** | Elastic Network Interface | 虚拟网卡，挂到 EC2 上提供网络接入 |
| **EIP** | Elastic IP | 固定的公网 IP，可随时绑定/解绑 EC2 |
| **IGW** | Internet Gateway | VPC 访问互联网的大门 |
| **NAT** | Network Address Translation | 让私有子网的资源能访问公网（单向） |
| **AZ** | Availability Zone | 可用区（物理隔离的数据中心），如 ap-northeast-1a |
| **Region** | — | AWS 区域（地理区域），如 ap-northeast-1（东京） |
| **ARN** | Amazon Resource Name | AWS 资源的全局唯一标识符 |

---

## 🌐 网络 & 负载均衡

| 缩写 | 全称 | 一句话作用 |
|------|------|-----------|
| **ALB** | Application Load Balancer | HTTP/HTTPS 应用层负载均衡（第 7 层） |
| **NLB** | Network Load Balancer | TCP/UDP 网络层负载均衡（第 4 层），超低延迟 |
| **CLB** | Classic Load Balancer | 经典负载均衡器（已淘汰，建议用 ALB/NLB 替代） |
| **TG** | Target Group | ALB/NLB 后端目标组，管理流量分发到哪些实例 |
| **ELB** | Elastic Load Balancing | 所有负载均衡器的统称（ALB + NLB + CLB） |
| **Route53** | — | AWS 的 DNS 服务（域名解析 + 流量路由） |
| **CloudFront** | — | CDN 内容分发网络，全球加速静态/动态内容 |
| **SG** | Security Group | 安全组（EC2 级别的虚拟防火墙） |
| **NACL** | Network ACL | 网络访问控制列表（子网级别的防火墙） |
| **VGW** | Virtual Private Gateway | VPN 连接中 VPC 端的网关 |
| **CGW** | Customer Gateway | VPN 连接中本地数据中心端的网关 |
| **DX** | Direct Connect | 专线连接，本地数据中心到 AWS 的专有物理线路 |
| **GTW** | Gateway | 网关的通用缩写（IGW/NAT GW/VPN GW 等） |
| **RT** | Route Table | 路由表，定义子网内的流量怎么走 |
| **RTB** | Route Table Association | 路由表关联，将路由表绑到特定子网 |

---

## 🗄️ 数据库

| 缩写 | 全称 | 一句话作用 |
|------|------|-----------|
| **RDS** | Relational Database Service | 托管关系型数据库（MySQL/PostgreSQL 等） |
| **Aurora** | Amazon Aurora | AWS 自研的高性能数据库（兼容 MySQL/PostgreSQL） |
| **DynamoDB** | — | NoSQL 键值/文档数据库，超低延迟，自动扩缩 |
| **ElastiCache** | — | 托管 Redis/Memcached 缓存服务 |
| **Redshift** | — | 数据仓库，PB 级数据分析 |
| **DMS** | Database Migration Service | 数据库迁移服务（不停机迁移上云） |
| **Neptune** | — | 图数据库服务 |

---

## 🔐 安全 & 认证

| 缩写 | 全称 | 一句话作用 |
|------|------|-----------|
| **IAM** | Identity and Access Management | 用户/角色/权限管理（谁可以做什么） |
| **KMS** | Key Management Service | 托管密钥服务，加密/解密 |
| **Secrets Manager** | — | 托管密钥/密码/API Token 的轮换和存储 |
| **ACM** | AWS Certificate Manager | 免费 SSL/TLS 证书管理 |
| **WAF** | Web Application Firewall | Web 应用防火墙（防 SQL 注入、XSS 等） |
| **Shield** | AWS Shield | DDoS 防护（基础版免费，高级版付费） |
| **GuardDuty** | — | 威胁检测服务（持续监控恶意活动） |
| **CloudTrail** | — | API 调用日志审计（谁在什么时候做了什么操作） |
| **Config** | AWS Config | 资源配置合规检查（资源变化记录 + 规则评估） |
| **STS** | Security Token Service | 临时凭证颁发（跨账号访问用的临时密钥） |

---

## 📦 容器 & 编排

| 缩写 | 全称 | 一句话作用 |
|------|------|-----------|
| **EKS** | Elastic Kubernetes Service | 托管 K8s 集群 |
| **ECS** | Elastic Container Service | 托管 Docker 容器（AWS 自家的编排引擎） |
| **ECR** | Elastic Container Registry | 私有 Docker 镜像仓库 |
| **Fargate** | — | 无服务器容器引擎（不用管 EC2 节点） |
| **EC2** | — | 也可以跑容器（自建 Docker / 自建 K8s） |

---

## 📡 消息 & 事件

| 缩写 | 全称 | 一句话作用 |
|------|------|-----------|
| **SQS** | Simple Queue Service | 消息队列服务（解耦微服务） |
| **SNS** | Simple Notification Service | 消息通知服务（推送邮件/短信/HTTP 等） |
| **EventBridge** | — | 事件总线（连接 AWS 服务和应用） |
| **MQ** | Amazon MQ | 托管 ActiveMQ / RabbitMQ |

---

## 🏗️ 基础设施即代码 & 编排

| 缩写 | 全称 | 一句话作用 |
|------|------|-----------|
| **CFn** | CloudFormation | AWS 自家的 IaC 工具（类似 Terraform） |
| **CDK** | Cloud Development Kit | 用 TypeScript/Python/Java 写 IaC（Terraform 的编程式替代） |
| **OP** | OpsWorks | 配置管理服务（基于 Chef/Puppet） |
| **SSM** | Systems Manager | 运维管理（Session Manager 远程登录、参数存储、补丁管理） |
| **ASG** | Auto Scaling Group | 自动扩缩组（根据负载自动增减 EC2 数量） |
| **LC** | Launch Configuration | 启动配置（ASG 的模板，已淘汰，请用 Launch Template） |
| **LT** | Launch Template | 启动模板（ASG 和 EC2 的模板，支持版本管理） |

---

## 🔄 持续集成 & 部署

| 缩写 | 全称 | 一句话作用 |
|------|------|-----------|
| **CodeCommit** | — | 托管 Git 仓库（类似 GitHub/GitLab） |
| **CodeBuild** | — | 编译构建服务 |
| **CodeDeploy** | — | 自动部署服务 |
| **CodePipeline** | — | CI/CD 管道编排服务 |
| **CodeStar** | — | 一站式 DevOps 项目管理 |

---

## 📊 监控 & 日志

| 缩写 | 全称 | 一句话作用 |
|------|------|-----------|
| **CloudWatch** | — | 监控 + 日志 + 告警的核心服务 |
| **X-Ray** | — | 分布式链路追踪（排性能瓶颈用） |
| **VPC Flow Logs** | — | 记录 VPC 网络流量日志（排查网络问题） |
| **CloudTrail** | — | API 审计日志（已在上方安全类列出） |

---

## 🔗 混合云 & 网络互联

| 缩写 | 全称 | 一句话作用 |
|------|------|-----------|
| **VPN** | Virtual Private Network | 站点到站点 VPN，连接本地和 VPC |
| **DX** | Direct Connect | 专线（前文已列），物理直连，稳定低延迟 |
| **TGW** | Transit Gateway | 网络中转网关，连接多个 VPC/本地网络 |
| **PCA** | Private CA | 私有证书颁发机构（内部 HTTPS 认证） |
| **Privatelink** | AWS PrivateLink | VPC 终端节点服务，私密访问 SaaS |

---

## 💰 账单 & 成本

| 缩写 | 全称 | 一句话作用 |
|------|------|-----------|
| **CE** | Cost Explorer | 成本分析控制台 |
| **Budgets** | AWS Budgets | 预算告警（超支通知） |
| **TAM** | Technical Account Manager | 企业支持的技术客户经理（人） |
| **RI** | Reserved Instance | 预留实例（预付省钱） |
| **SP** | Savings Plans | 灵活省钱计划（替代 RI 的下一代方案） |
| **CUR** | Cost and Usage Report | 详细成本与用量报告（最细粒度的账单数据） |

---

## 📖 面试 / Terraform 中最常出现的缩写 TOP 15

| 排名 | 缩写 | 出现场景 |
|------|------|----------|
| 1 | **EC2** | 几乎所有基础设施模板都有 |
| 2 | **VPC** | 网络层基础，天天见 |
| 3 | **S3** | 存储 + 远程 state 后端 |
| 4 | **IAM** | 权限管理，Terraform provider 配置 |
| 5 | **RDS** | 数据库基座 |
| 6 | **ALB** | 负载均衡，`aws_lb` + `aws_lb_listener` + `aws_lb_target_group` |
| 7 | **EKS** | K8s 集群创建 |
| 8 | **SG** | 安全组，每套环境必有 |
| 9 | **IGW** | VPC 访问公网 |
| 10 | **AZ** | 可用区，多 AZ 部署高可用 |
| 11 | **ARN** | 资源引用时处处用到 |
| 12 | **Route53** | DNS 解析 |
| 13 | **KMS** | 加密相关配置 |
| 14 | **ASG** | 自动扩缩容 |
| 15 | **DynamoDB** | 分布式场景 + Terraform state 锁表（配合 S3 backend） |

---

> 💡 在你 Terraform 入门指南的 Day 6-7 中掌握上面 TOP 15 的用法，已经能覆盖 80% 的日常 AWS IaC 需求。

> ⚠️ AWS 一直在推出新服务，部分缩写（如 CLB）已淘汰，生产环境注意查看官方最新文档。

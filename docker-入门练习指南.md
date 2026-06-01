# 🐳 Docker 入门练习指南

> 适用环境：macOS + Docker Desktop + AWS 免费账号
> 目标：从零基础到能独立部署容器化应用
> 预计耗时：3-5 天（每天 1-2 小时）

---

## 📋 目录

1. [环境准备](#1-环境准备)
2. [Day 1：Docker 基础操作](#2-day-1docker-基础操作)
3. [Day 2：构建自己的镜像](#3-day-2构建自己的镜像)
4. [Day 3：多容器编排（docker-compose）](#4-day-3多容器编排docker-compose)
5. [Day 4：结合 AWS 练手](#5-day-4结合-aws-练手)
6. [Day 5：CI/CD + 自动化部署](#6-day-5cicd--自动化部署)
7. [常见问题与技巧](#7-常见问题与技巧)

---

## 1. 环境准备

### 1.1 装 Docker Desktop

```bash
# 用 Homebrew 安装（推荐）
brew install --cask docker

# 或者手动下载安装：https://www.docker.com/products/docker-desktop/
```

安装后打开 Docker Desktop，等顶部状态栏的鲸鱼图标停止动画（表示 Docker 引擎已启动）。

### 1.2 验证安装

```bash
# 检查版本
docker --version
docker compose version

# 跑通第一个容器
docker run hello-world
```

看到欢迎信息就成功了 ✅

---

## 2. Day 1：Docker 基础操作

### 2.1 核心概念速览

| 概念 | 类比 | 说明 |
|------|------|------|
| **Image（镜像）** | 安装包 / 类 | 只读模板，包含运行环境 + 代码 |
| **Container（容器）** | 运行中的进程 / 实例 | 镜像的运行态，可读写 |
| **Dockerfile** | 安装说明书 | 描述如何构建镜像 |
| **Volume（卷）** | U 盘 | 数据持久化，容器删除后数据还在 |
| **Port Mapping（端口映射）** | 端口转发 | 把容器内的端口暴露到宿主机 |

### 2.2 常用命令练习

开一个终端，跟我敲：

```bash
# ──────────── 镜像相关 ────────────

# 拉取镜像
docker pull nginx:latest
docker pull python:3.11-slim
docker pull mysql:8.0

# 查看本地有哪些镜像
docker images

# 删除镜像
docker rmi nginx:latest

# ──────────── 容器生命周期 ────────────

# 运行一个 Nginx 容器（前台运行）
docker run -p 8080:80 nginx
# 然后打开浏览器访问 http://localhost:8080
# 按 Ctrl+C 停止

# 后台运行（-d = detach）
docker run -d --name my-nginx -p 8080:80 nginx

# 查看运行中的容器
docker ps

# 查看所有容器（包括已停止的）
docker ps -a

# 查看容器日志
docker logs my-nginx

# 进入容器内部（交互式）
docker exec -it my-nginx bash
# 进去后可以 ls /usr/share/nginx/html 看看，exit 退出

# 停止/启动/重启
docker stop my-nginx
docker start my-nginx
docker restart my-nginx

# 删除容器（先停止）
docker rm my-nginx
docker rm -f my-nginx    # 强制删除（运行中也删）

# ──────────── 端口映射练习 ────────────

# 同时跑两个 Nginx，映射不同端口
docker run -d --name web1 -p 8081:80 nginx
docker run -d --name web2 -p 8082:80 nginx
# 分别访问 http://localhost:8081 和 http://localhost:8082

# ──────────── 资源查看 ────────────

# 查看容器资源占用
docker stats

# 查看容器详情
docker inspect web1
```

### 2.3 练习任务 ✅

- [ ] 分别启动 Nginx、Python、MySQL 容器
- [ ] 进入 Nginx 容器，修改 `/usr/share/nginx/html/index.html` 里的内容，刷新浏览器看变化
- [ ] 用 `docker logs` 看 Nginx 的访问日志
- [ ] 用 `docker stop` 停止所有练习容器，用 `docker rm` 清理

---

## 3. Day 2：构建自己的镜像

### 3.1 写你的第一个 Dockerfile

创建一个项目目录：

```bash
mkdir ~/docker-demo && cd ~/docker-demo
```

新建文件 `Dockerfile`：

```dockerfile
# 使用官方 Python 镜像作为基础
FROM python:3.11-slim

# 设置工作目录
WORKDIR /app

# 复制当前目录内容到容器
COPY . .

# 安装依赖（如果有 requirements.txt）
# RUN pip install -r requirements.txt

# 声明容器启动时监听的端口
EXPOSE 5000

# 容器启动时执行的命令
CMD ["python", "-m", "http.server", "5000"]
```

在相同目录下随便放一个测试文件：

```bash
echo "<h1>Hello from Docker!</h1>" > index.html
```

构建并运行：

```bash
# 构建镜像（注意末尾的点）
docker build -t my-web-app .

# 运行
docker run -d --name my-app -p 5000:5000 my-web-app

# 访问 http://localhost:5000 看看效果
```

### 3.2 构建一个真实应用（Python Flask + Redis）

```bash
mkdir ~/flask-demo && cd ~/flask-demo
```

**app.py**：

```python
from flask import Flask
from redis import Redis
import os

app = Flask(__name__)
redis = Redis(host=os.getenv("REDIS_HOST", "localhost"), port=6379)

@app.route("/")
def hello():
    count = redis.incr("visits")
    return f"<h1>Hello from Docker!</h1><p>Visits: {count}</p>"

if __name__ == "__main__":
    app.run(host="0.0.0.0", port=5000)
```

**requirements.txt**：

```
flask>=3.0
redis>=5.0
```

**Dockerfile**：

```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY requirements.txt .
RUN pip install -r requirements.txt
COPY . .
EXPOSE 5000
CMD ["python", "app.py"]
```

构建：

```bash
docker build -t flask-app .
```

### 3.3 镜像优化技巧（进阶）

```dockerfile
# ❌ 不好的做法
FROM python:3.11          # 太大（~1GB）
COPY . .                  # 包含不需要的文件
RUN pip install -r requirements.txt  # 不利用缓存

# ✅ 优化后的做法
FROM python:3.11-slim     # 小镜像（~150MB）
WORKDIR /app
COPY requirements.txt .   # 先 copy 依赖文件
RUN pip install --no-cache-dir -r requirements.txt  # 利用 layer 缓存
COPY . .                  # 最后 copy 代码（经常变）
CMD ["python", "app.py"]
```

使用 `.dockerignore` 文件（和 Dockerfile 同目录）：

```
__pycache__
*.pyc
.git
.env
*.md
```

### 3.4 练习任务 ✅

- [ ] 写一个 Dockerfile 打包任意一个你现有的小项目
- [ ] 用 `docker images` 看看镜像大小，尝试用 slim 版本减小体积
- [ ] 建一个 `.dockerignore` 排除不必要的文件
- [ ] 用 `docker history <镜像名>` 查看镜像的每一层

---

## 4. Day 3：多容器编排（docker-compose）

### 4.1 先理解问题

如果不使用 docker-compose，要让 Flask + Redis 一起工作需要：

```bash
# 先建一个网络
docker network create my-net

# 启动 Redis
docker run -d --name redis --network my-net redis:7-alpine

# 启动 Flask（依赖 Redis）
docker run -d --name flask-app -p 5000:5000 --network my-net -e REDIS_HOST=redis flask-app
```

太麻烦了。用 docker-compose 解决 👇

### 4.2 docker-compose.yml

在 `~/flask-demo/` 目录下新建 `docker-compose.yml`：

```yaml
version: "3.8"

services:
  redis:
    image: redis:7-alpine
    container_name: flask-redis
    restart: always
    volumes:
      - redis-data:/data

  web:
    build: .
    container_name: flask-web
    ports:
      - "5000:5000"
    environment:
      - REDIS_HOST=redis
    depends_on:
      - redis
    restart: always

volumes:
  redis-data:
```

一键启动：

```bash
# 构建并启动所有服务
docker compose up -d

# 访问 http://localhost:5000，每刷新一次计数+1

# 查看日志（同时看两个服务的）
docker compose logs -f

# 停止所有服务
docker compose down

# 停止并删除数据卷
docker compose down -v
```

### 4.3 更完整的实战：WordPress 博客

```yaml
version: "3.8"

services:
  db:
    image: mysql:8.0
    container_name: wp-db
    restart: always
    environment:
      MYSQL_ROOT_PASSWORD: rootpass
      MYSQL_DATABASE: wordpress
      MYSQL_USER: wpuser
      MYSQL_PASSWORD: wppass
    volumes:
      - db-data:/var/lib/mysql

  wordpress:
    image: wordpress:latest
    container_name: wp-app
    ports:
      - "8080:80"
    restart: always
    environment:
      WORDPRESS_DB_HOST: db
      WORDPRESS_DB_USER: wpuser
      WORDPRESS_DB_PASSWORD: wppass
      WORDPRESS_DB_NAME: wordpress
    depends_on:
      - db

volumes:
  db-data:
```

```bash
# 启动 WordPress（MySQL + WordPress 自动关联）
docker compose -f wordpress.yml up -d

# 访问 http://localhost:8080 安装博客
```

> 这是很多应用的标准模式：Web 容器 + 数据库容器，通过 compose 编排。

### 4.4 docker-compose 常用命令

```bash
docker compose up -d          # 后台启动
docker compose down           # 停止并移除容器
docker compose ps             # 查看服务状态
docker compose logs -f        # 查看所有日志
docker compose exec web bash  # 进入 web 服务容器
docker compose restart web    # 只重启 web 服务
docker compose build          # 重新构建镜像
docker compose pull           # 拉取最新的镜像
```

### 4.5 练习任务 ✅

- [ ] 启动 Flask + Redis 的 compose，访问计数页面
- [ ] 启动 WordPress 博客
- [ ] 修改 `docker-compose.yml`，把端口改成别的（比如 8888）
- [ ] 练习 `docker compose logs`、`exec`、`restart` 命令

---

## 5. Day 4：结合 AWS 练手

> 前提：已有 AWS 免费账号

### 5.1 EC2 + Docker（最经典方案）

#### 启动 EC2 实例

1. 登录 AWS 控制台 → EC2 → 启动实例
2. 命名：`docker-playground`
3. AMI：Amazon Linux 2023（免费）
4. 实例类型：`t2.micro` 或 `t3.micro`（免费额度内）
5. **密钥对**：创建或选择已有（用于 SSH 登录，⚠️ 保存好 `.pem` 文件）
6. 安全组规则添加：SSH（22）来源：**你的 IP**（尽量限制来源 IP）
7. 启动

#### 登录并装 Docker

```bash
# 设置权限
chmod 400 ~/Downloads/你的密钥.pem

# SSH 登录
ssh -i ~/Downloads/你的密钥.pem ec2-user@<你的 EC2 公网 IP>

# 安装 Docker
sudo yum install docker -y
sudo systemctl enable docker
sudo systemctl start docker

# 把当前用户加到 docker 组（免 sudo）
sudo usermod -aG docker ec2-user

# 退出重连（使组生效）
exit
```

再次 SSH 登录，验证：

```bash
docker --version
docker run hello-world
```

#### 做点有趣的事

```bash
# 在远程服务器上运行 Nginx
docker run -d --name my-nginx -p 80:80 nginx

# 用浏览器访问 http://<你的 EC2 公网 IP>
# 🎉 你的第一个生产级部署！
```

**安全组注意事项**：记得在 EC2 安全组中开放 80 端口入站规则。

### 5.2 ECR（Elastic Container Registry）

把本地构建的镜像推送到 AWS 仓库：

```bash
# 1️⃣ 在 AWS 控制台创建仓库
# 搜索 ECR → 创建仓库 → 名称：my-app

# 2️⃣ 安装 AWS CLI（本地）
brew install awscli

# 3️⃣ 配置凭证
aws configure
# 输入 Access Key ID / Secret Access Key / 区域（如 us-east-1）

# 4️⃣ 登录 ECR
aws ecr get-login-password --region us-east-1 | \
  docker login --username AWS --password-stdin \
  <你的账号ID>.dkr.ecr.us-east-1.amazonaws.com

# 5️⃣ 给本地镜像打标签
docker tag flask-app:latest \
  <你的账号ID>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest

# 6️⃣ 推送到 ECR
docker push <你的账号ID>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest

# 7️⃣ 在 EC2 上拉取运行（需要 EC2 有对应的 IAM 权限）
# EC2 上也需要先安装 awscli 并配置，或者给 EC2 绑定一个 ECR 读取权限的 IAM Role
docker pull <你的账号ID>.dkr.ecr.us-east-1.amazonaws.com/my-app:latest
```

### 5.3 ECS Fargate（Serverless 容器）

AWS 控制台操作（不需要会代码）：

1. 搜索 **ECS** → 创建集群 → 选择 **Networking only（Fargate）**
2. 创建任务定义（Task Definition）：
   - 启动类型：Fargate
   - 任务角色：无
   - 任务大小：0.5 vCPU / 1GB 内存（免费额度内）
   - 添加容器：nginx:latest，端口映射 80
3. 运行任务 → 选择 VPC/子网 → 安全组开放 80 端口
4. 查看任务状态 → 获取公网 IP → 浏览器访问 🎉

> 💡 Fargate 有免费额度（每月 20 万次 CPU 请求 + 40 万次 GB-秒），玩几次不花钱。

### 5.4 练习任务 ✅

- [ ] 在 EC2 上成功跑一个 Nginx 容器，浏览器能访问
- [ ] 把 Flask 应用部署到 EC2 上运行
- [ ] （选做）推一个镜像到 ECR
- [ ] （选做）用 Fargate 跑一个容器

---

## 6. Day 5：CI/CD + 自动化部署

### 6.1 用 GitHub Actions 自动构建镜像

项目根目录新建 `.github/workflows/docker-build.yml`：

```yaml
name: Docker Build and Push

on:
  push:
    branches: [main]

jobs:
  build:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Build Docker Image
        run: docker build -t my-app .

      - name: Run Tests
        run: |
          docker run -d -p 5000:5000 --name test my-app
          sleep 3
          curl -f http://localhost:5000 || exit 1
```

把代码推送到 GitHub，Actions 会自动构建并测试。

### 6.2 完整 CI/CD（构建 → 推 ECR → 部署 EC2）

```yaml
name: Deploy to EC2

on:
  push:
    branches: [main]

env:
  AWS_REGION: us-east-1
  ECR_REPOSITORY: my-app
  EC2_HOST: ${{ secrets.EC2_HOST }}

jobs:
  deploy:
    runs-on: ubuntu-latest

    steps:
      - uses: actions/checkout@v4

      - name: Configure AWS credentials
        uses: aws-actions/configure-aws-credentials@v4
        with:
          aws-access-key-id: ${{ secrets.AWS_ACCESS_KEY_ID }}
          aws-secret-access-key: ${{ secrets.AWS_SECRET_ACCESS_KEY }}
          aws-region: ${{ env.AWS_REGION }}

      - name: Login to ECR
        id: login-ecr
        uses: aws-actions/amazon-ecr-login@v2

      - name: Build, tag, and push image
        id: build-image
        env:
          ECR_REGISTRY: ${{ steps.login-ecr.outputs.registry }}
          IMAGE_TAG: ${{ github.sha }}
        run: |
          docker build -t $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG .
          docker push $ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG
          echo "image=$ECR_REGISTRY/$ECR_REPOSITORY:$IMAGE_TAG" >> $GITHUB_OUTPUT

      - name: Deploy to EC2
        uses: appleboy/ssh-action@v1.0.0
        with:
          host: ${{ env.EC2_HOST }}
          username: ec2-user
          key: ${{ secrets.EC2_SSH_KEY }}
          script: |
            docker pull ${{ steps.build-image.outputs.image }}
            docker stop my-app || true
            docker rm my-app || true
            docker run -d --name my-app -p 80:5000 ${{ steps.build-image.outputs.image }}
```

> 在 GitHub 仓库的 Settings → Secrets and variables → Actions 中配置 `AWS_ACCESS_KEY_ID`、`AWS_SECRET_ACCESS_KEY`、`EC2_HOST`、`EC2_SSH_KEY`

### 6.3 练习任务 ✅

- [ ] 把你的项目推到 GitHub，配置 GitHub Actions 自动构建
- [ ] （进阶）实现 EC2 自动部署
- [ ] 体会一下：代码 push → 自动构建 → 自动部署 → 访问新版本

---

## 7. 常见问题与技巧

### 🔧 常用清理命令

```bash
# 清理所有停止的容器
docker container prune

# 清理未使用的镜像
docker image prune

# 清理所有未使用的资源（镜像/容器/网络/构建缓存）
docker system prune -a

# 查看磁盘占用
docker system df
```

### 🐛 调试技巧

```bash
# 容器起不来？看日志
docker logs <容器名>

# 进入容器排查
docker exec -it <容器名> bash  # 或 sh（alpine 镜像）

# 查看端口是否映射成功
docker port <容器名>

# 查容器 IP
docker inspect <容器名> | grep IPAddress
```

### ⚠️ AWS 省钱提醒

| 资源 | 省钱建议 |
|------|---------|
| EC2 | 不用的实例要**停止**（Stop）而不是放那里（Stop 不收费，EBS 存储仍收很少费用）|
| ECR | 推送后没用的镜像清理掉 |
| ECS | Fargate 任务跑完后确认任务已停止 |
| 弹性 IP | 不用的弹性 IP 要**释放**（Release），否则每小时收费 |

> 💡 在 AWS 控制台设置 **Billing Alerts**（账单告警），超过 $1 就发邮件提醒，防止意外超支。

### 📚 推荐资源

- [Docker 官方教程](https://docs.docker.com/get-started/) — 交互式教程
- [Play with Docker](https://labs.play-with-docker.com/) — 浏览器里的 Docker 沙箱（免费，不用装环境）
- [Docker Cheat Sheet](https://dockerlabs.collabnix.com/docker/cheatsheet/) — 命令速查表
- [ECS Workshop](https://ecsworkshop.com/) — AWS 容器化工作坊

---

**恭喜 🎉** — 走到这里，你已经具备了 Docker 的核心实战能力！从本地开发到云上部署，这个流程是当前后端开发的标准技能树。

# Mall 项目部署历史记录

## 从 "2026 Qoder CLI" 到 "20260114 Qoder CLI" 的完整操作流程

本文档记录了 mall-admin-web 前端项目从初始部署到更新的完整过程。

---

## 1️⃣ 初始修改（本地）

**时间点：** 2026-01-14 初始部署

**操作内容：**
- 修改文件：`/code/mall-admin-web/src/views/home/index.vue`
- 添加内容：在管理后台首页添加消息框
- 消息内容：`"2026 Qoder CLI"`

**代码修改：**
```vue
<template>
  <div class="app-container">
    <el-alert
      title="2026 Qoder CLI"
      type="info"
      center
      :closable="false"
      show-icon>
    </el-alert>
    <!-- 其他内容 -->
  </div>
</template>
```

**构建和部署：**
```bash
# 构建 Docker 镜像
cd /code/mall-admin-web
docker build -t mall/mall-admin-web:latest .

# 启动容器
docker run -d --name mall-admin-web \
  --network docker_default \
  -p 80:80 \
  mall/mall-admin-web:latest
```

**结果：** 本地运行的容器显示 "2026 Qoder CLI"

---

## 2️⃣ 提交到 GitHub

### 2.1 配置 Git 用户信息

```bash
cd /code/mall-admin-web
git config user.name "lvzhaobo"
git config user.email "lvzhaobo@users.noreply.github.com"
```

### 2.2 创建提交

**添加文件到暂存区：**
```bash
git add config/prod.env.js \
        src/views/home/index.vue \
        Dockerfile \
        nginx.conf
```

**创建提交：**
```bash
git commit -m "Add Docker deployment support and customize admin dashboard

- Add Dockerfile for multi-stage build (Node.js + Nginx)
- Add nginx.conf with API reverse proxy configuration
- Update prod.env.js to use relative API path
- Add welcome message \"2026 Qoder CLI\" to admin home page

🤖 Generated with [Qoder](https://qoder.com)"
```

### 2.3 推送到 GitHub

**修改远程仓库地址：**
```bash
git remote set-url origin https://github.com/lvzhaobo/mall.git
```

**创建并推送新分支：**
```bash
git checkout -b mall-admin-web-deployment
git push origin mall-admin-web-deployment
```

**结果：** 
- 创建了分支：`mall-admin-web-deployment`
- 仓库地址：https://github.com/lvzhaobo/mall.git
- 提交 ID：`a82ccc6`

---

## 3️⃣ 合并到 master 分支

### 3.1 问题发现

远程 master 分支已包含其他内容：
- 后端项目（mall Java 代码）
- AI 相关文档（ai-coding 目录）
- 这些内容与 mall-admin-web 是完全不同的历史

### 3.2 合并操作

**拉取远程 master：**
```bash
git fetch origin master:origin-master
```

**创建合并分支：**
```bash
git checkout -b merge-to-master origin-master
```

**合并代码（允许不相关的历史）：**
```bash
git merge mall-admin-web-deployment --allow-unrelated-histories --no-edit
```

**解决冲突：**
```bash
# 保留 mall-admin-web 版本的文件
git checkout --theirs .gitignore LICENSE README.md
git add .gitignore LICENSE README.md
```

**完成合并提交：**
```bash
git commit -m "Merge mall-admin-web deployment files into master

- Add Docker deployment support (Dockerfile + nginx.conf)
- Add complete mall-admin-web Vue.js application
- Update configuration for production deployment
- Add custom welcome message to admin dashboard

🤖 Generated with [Qoder](https://qoder.com)"
```

**推送到远程 master：**
```bash
git push origin merge-to-master:master
```

**结果：** 
- GitHub master 分支包含了完整的 mall-admin-web 代码
- 提交 ID：`787be6f`

---

## 4️⃣ GitHub 上的代码修改

**时间点：** 推送到 GitHub 之后

**修改内容：**
- 文件位置：`src/views/home/index.vue` 第4行
- 修改前：`title="2026 Qoder CLI"`
- 修改后：`title="20260114 Qoder CLI"`

**修改后的代码：**
```vue
<el-alert
  title="20260114 Qoder CLI"
  type="info"
  center
  :closable="false"
  show-icon>
</el-alert>
```

**修改者：** 可能是通过 GitHub 网页端或其他方式修改

---

## 5️⃣ 拉取 GitHub 更新并重新部署

### 5.1 拉取最新代码

**检查当前状态：**
```bash
cd /code/mall
git status
git remote -v
```

**拉取更新：**
```bash
git pull origin master
```

**更新统计：**
- 更新提交：`11bdb23..8765733`
- 修改文件：237 个文件
- 新增代码：34,245 行插入，188 行删除
- 包含：完整的 mall-admin-web Vue.js 应用代码

### 5.2 重新构建 Docker 镜像

```bash
cd /code/mall
docker build -t mall/mall-admin-web:latest -f Dockerfile .
```

**构建过程：**
- 使用 Node.js 14 构建前端资源
- 使用 Nginx 1.22-alpine 作为运行环境
- 构建时间：约 55 秒
- 最终镜像大小：58.2 MB

### 5.3 重启前端容器

```bash
# 停止并删除旧容器
docker stop mall-admin-web
docker rm mall-admin-web

# 启动新容器
docker run -d --name mall-admin-web \
  --network docker_default \
  -p 80:80 \
  mall/mall-admin-web:latest
```

**验证：**
```bash
docker ps --filter name=mall-admin-web
```

**结果：** 容器成功运行，显示 "20260114 Qoder CLI"

---

## 关键信息总结

### 项目架构

| 组件 | 技术栈 | 端口 | 状态 |
|------|--------|------|------|
| mall-admin（后端） | Spring Boot 2.7.5 + Java 8 | 8080 | 运行中 |
| mall-admin-web（前端） | Vue.js 2 + Element UI | 80 | 运行中 |
| MySQL | 5.7 | 3306 | 运行中 |
| Redis | 7 | 6379 | 运行中 |

### Git 操作时间线

| 阶段 | 位置 | 内容 | 提交/分支 |
|------|------|------|-----------|
| 初始修改 | 本地 `/code/mall-admin-web` | "2026 Qoder CLI" | - |
| 首次提交 | GitHub `mall-admin-web-deployment` | "2026 Qoder CLI" | `a82ccc6` |
| 合并到 master | GitHub `master` | "2026 Qoder CLI" | `787be6f` |
| **远程修改** | GitHub `master` | **"20260114 Qoder CLI"** | `8765733` |
| 本地更新 | 本地 `/code/mall` | "20260114 Qoder CLI" | 已拉取 |
| 重新部署 | Docker 容器 | "20260114 Qoder CLI" | 运行中 |

### 文件路径说明

- **后端项目：** `/code/mall/` (包含 Java Maven 项目)
- **前端源码：** `/code/mall/src/` (Vue.js 组件)
- **首页组件：** `/code/mall/src/views/home/index.vue`
- **Docker 配置：** `/code/mall/Dockerfile` 和 `/code/mall/nginx.conf`

### 重要配置文件

#### 1. Dockerfile（多阶段构建）
```dockerfile
# 第一阶段：构建
FROM node:14 AS builder
WORKDIR /app
COPY package*.json ./
RUN npm config set registry https://registry.npmmirror.com
RUN npm install
COPY . .
RUN npm run build

# 第二阶段：运行
FROM nginx:1.22-alpine
COPY --from=builder /app/dist /usr/share/nginx/html
COPY nginx.conf /etc/nginx/conf.d/default.conf
EXPOSE 80
CMD ["nginx", "-g", "daemon off;"]
```

#### 2. nginx.conf（反向代理配置）
```nginx
server {
    listen 80;
    server_name localhost;

    location / {
        root /usr/share/nginx/html;
        index index.html index.htm;
        try_files $uri $uri/ /index.html;
    }

    # 代理后端 API
    location /api/ {
        proxy_pass http://mall-admin:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }
}
```

#### 3. config/prod.env.js
```javascript
module.exports = {
  NODE_ENV: '"production"',
  BASE_API: '"/api"'  // 使用相对路径，通过 Nginx 反向代理
}
```

---

## 访问信息

- **管理后台：** http://服务器IP
- **登录账号：** admin
- **登录密码：** 871204（已修改）
- **后端 API：** http://服务器IP:8080
- **Swagger 文档：** http://服务器IP:8080/swagger-ui.html

---

## 经验总结

### 1. Docker 部署优势
- **多阶段构建**减少最终镜像大小（从 1.27GB 降至 58.2MB）
- **容器化部署**保证环境一致性
- **网络隔离**通过 Docker 网络实现服务间通信

### 2. Git 协作流程
- 使用分支进行功能开发
- 通过 merge 合并到主分支
- 及时 pull 获取远程更新

### 3. 前后端分离部署
- Nginx 反向代理解决跨域问题
- 前后端独立部署和更新
- 使用 Docker 网络进行服务发现

---

**文档生成时间：** 2026-01-14  
**生成工具：** Qoder CLI  
**项目仓库：** https://github.com/lvzhaobo/mall.git

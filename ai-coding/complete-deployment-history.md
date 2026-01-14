# Mall 项目完整部署历史记录

## 概览

本文档记录了 mall 电商项目从初始部署到多次更新的完整历史，包括所有 Git 操作、Docker 构建、配置修改和问题解决过程。

**项目信息：**
- 项目名称：mall
- 仓库地址：https://github.com/lvzhaobo/mall.git
- 后端技术：Spring Boot 2.7.5 + Java 8 + MySQL 5.7 + Redis 7
- 前端技术：Vue.js 2 + Element UI + Nginx
- 部署方式：Docker 容器化部署
- 记录时间：2026-01-14

---

## 第一阶段：初始环境部署

### 1.1 数据库和缓存部署

**部署 MySQL 5.7：**
```bash
docker-compose up -d mysql
```

**导入数据库：**
```bash
# 导入 mall.sql
docker exec -i mysql mysql -uroot -proot mall < /code/mall/document/sql/mall.sql
```

**部署 Redis 7：**
```bash
docker-compose up -d redis
```

**验证服务：**
```bash
docker ps | grep -E 'mysql|redis'
```

**结果：**
- MySQL 运行在端口 3306
- Redis 运行在端口 6379
- 数据库导入成功，包含 mall 项目所需的所有表结构和初始数据

---

### 1.2 后端服务部署

**构建后端项目：**
```bash
cd /code/mall
mvn clean package -Dmaven.test.skip=true
```

**遇到的问题：**
- 主机环境缺少 Maven 和 Java
- 解决方案：使用 Docker Maven 镜像构建

**使用 Docker 构建：**
```bash
docker run --rm \
  -v /code/mall:/workspace \
  -w /workspace \
  maven:3.8.6-openjdk-8 \
  mvn clean package -Dmaven.test.skip=true
```

**创建后端 Dockerfile：**
```dockerfile
FROM eclipse-temurin:8-jre
VOLUME /tmp
COPY mall-admin/target/mall-admin-1.0-SNAPSHOT.jar app.jar
ENTRYPOINT ["java", "-jar", "/app.jar"]
```

**构建并运行后端容器：**
```bash
# 构建镜像
docker build -t mall/mall-admin:1.0-SNAPSHOT .

# 运行容器
docker run -d \
  --name mall-admin \
  --network docker_default \
  -p 8080:8080 \
  -e SPRING_DATASOURCE_URL="jdbc:mysql://mysql:3306/mall?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Shanghai" \
  -e SPRING_DATASOURCE_USERNAME=root \
  -e SPRING_DATASOURCE_PASSWORD=root \
  -e SPRING_REDIS_HOST=redis \
  mall/mall-admin:1.0-SNAPSHOT
```

**网络问题解决：**
- 问题：容器无法解析 `db` 主机名
- 解决：使用 `docker_default` 网络，配置数据库连接为 `mysql:3306`

**验证后端服务：**
```bash
curl http://localhost:8080/swagger-ui.html
```

**结果：**
- 后端服务成功启动在端口 8080
- Swagger 文档可以正常访问
- API 接口响应正常

---

## 第二阶段：前端项目部署

### 2.1 克隆前端项目

**克隆 mall-admin-web：**
```bash
cd /code
git clone https://github.com/macrozheng/mall-admin-web.git
```

### 2.2 配置前端项目

**修改生产环境配置（config/prod.env.js）：**
```javascript
module.exports = {
  NODE_ENV: '"production"',
  BASE_API: '"/api"'  // 从硬编码的远程 API 改为相对路径
}
```

**创建 Nginx 配置（nginx.conf）：**
```nginx
server {
    listen       80;
    server_name  localhost;

    # 前端静态资源
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html;  # SPA 路由支持
    }

    # API 反向代理到后端
    location /api/ {
        proxy_pass http://mall-admin:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
    }

    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
}
```

**创建前端 Dockerfile（多阶段构建）：**
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

### 2.3 构建和部署前端

**构建前端镜像：**
```bash
cd /code/mall-admin-web
docker build -t mall/mall-admin-web:latest .
```

**构建统计：**
- 构建时间：约 55 秒
- 最终镜像大小：58.2 MB（从 1.27GB 优化而来）
- 使用 npm 镜像加速下载依赖

**启动前端容器：**
```bash
docker run -d \
  --name mall-admin-web \
  --network docker_default \
  -p 80:80 \
  mall/mall-admin-web:latest
```

**验证前端服务：**
```bash
curl http://localhost:80
docker logs mall-admin-web
```

**结果：**
- 前端成功运行在端口 80
- 可以通过浏览器访问管理后台
- Nginx 反向代理正常工作，前后端通信正常

---

## 第三阶段：首次代码定制

### 3.1 修改管理员密码

**需求：** 将管理员密码从 `macro123` 修改为 `871204`

**操作步骤：**

1. **查询管理员 ID：**
```bash
docker exec -it mysql mysql -uroot -proot -D mall -e "SELECT id, username FROM ums_admin WHERE username='admin';"
```

结果：管理员 ID 为 3

2. **生成密码哈希：**
使用 BCrypt 算法（Spring Security 默认）

3. **通过 API 修改密码：**
```bash
# 先登录获取 token
curl -X POST http://localhost:8080/admin/login \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","password":"macro123"}'

# 使用 token 修改密码
curl -X POST http://localhost:8080/admin/updatePassword \
  -H "Authorization: Bearer <token>" \
  -H "Content-Type: application/json" \
  -d '{"username":"admin","oldPassword":"macro123","newPassword":"871204"}'
```

**结果：**
- 密码修改成功
- 可以使用新密码 `871204` 登录

### 3.2 添加首页消息框

**需求：** 在管理后台首页添加消息框显示 "2026 Qoder CLI"

**修改文件：** `/code/mall-admin-web/src/views/home/index.vue`

**修改内容：**
```vue
<template>
  <div class="app-container">
    <!-- 新增消息框 -->
    <el-alert
      title="2026 Qoder CLI"
      type="info"
      center
      :closable="false"
      show-icon>
    </el-alert>
    
    <!-- 原有内容 -->
    <div class="address-layout">
      <!-- ... -->
    </div>
  </div>
</template>
```

**重新构建和部署：**
```bash
cd /code/mall-admin-web
docker build -t mall/mall-admin-web:latest .
docker stop mall-admin-web && docker rm mall-admin-web
docker run -d --name mall-admin-web --network docker_default -p 80:80 mall/mall-admin-web:latest
```

**结果：**
- 首页成功显示消息框 "2026 Qoder CLI"
- 消息框使用 Element UI 的 Alert 组件
- 样式：蓝色信息提示，居中显示，带图标

---

## 第四阶段：提交代码到 GitHub

### 4.1 配置 Git 用户信息

```bash
cd /code/mall-admin-web
git config user.name "lvzhaobo"
git config user.email "lvzhaobo@users.noreply.github.com"
```

### 4.2 创建首次提交

**添加修改的文件：**
```bash
git add config/prod.env.js \
        src/views/home/index.vue \
        Dockerfile \
        nginx.conf
```

**查看修改内容：**
```bash
git status
git diff --cached
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

**提交信息：**
- 提交 ID：`a82ccc6`
- 包含 4 个文件变更
- 提交者：lvzhaobo

### 4.3 修改远程仓库并推送

**更改远程仓库地址：**
```bash
git remote set-url origin https://github.com/lvzhaobo/mall.git
```

**创建并推送新分支：**
```bash
git checkout -b mall-admin-web-deployment
git push origin mall-admin-web-deployment
```

**推送结果：**
- 创建分支：`mall-admin-web-deployment`
- 推送成功到 GitHub
- GitHub 提示可以创建 Pull Request

---

## 第五阶段：合并到 master 分支

### 5.1 发现问题

**问题：** 远程 master 分支已包含其他内容
- 后端 Java 项目代码
- AI 相关文档（ai-coding 目录）
- 与 mall-admin-web 的历史完全不相关

### 5.2 合并策略

**拉取远程 master：**
```bash
git fetch origin master:origin-master
git log origin-master --oneline -5
```

**远程 master 提交历史：**
```
11bdb23 update knowledge
18738f3 ai-teams and knowledge
af93f1f step-3-1 erd
1bbbb2d step-5-ops
9283b65 todo
```

**创建合并分支：**
```bash
git checkout -b merge-to-master origin-master
```

**执行合并（允许不相关的历史）：**
```bash
git merge mall-admin-web-deployment --allow-unrelated-histories --no-edit
```

### 5.3 解决冲突

**冲突文件：**
- `.gitignore`
- `LICENSE`
- `README.md`

**解决方案：** 保留 mall-admin-web 的版本
```bash
git checkout --theirs .gitignore LICENSE README.md
git add .gitignore LICENSE README.md
```

**完成合并：**
```bash
git commit -m "Merge mall-admin-web deployment files into master

- Add Docker deployment support (Dockerfile + nginx.conf)
- Add complete mall-admin-web Vue.js application
- Update configuration for production deployment
- Add custom welcome message to admin dashboard

🤖 Generated with [Qoder](https://qoder.com)"
```

**推送到远程：**
```bash
git push origin merge-to-master:master
```

**合并结果：**
- 提交 ID：`787be6f`
- 成功合并到远程 master
- GitHub 仓库包含了前端、后端和文档

---

## 第六阶段：GitHub 远程更新（第一次）

### 6.1 远程修改

**时间点：** 合并到 master 后不久

**修改内容：**
- 文件：`src/views/home/index.vue`
- 修改者：lvzhaobo（在 GitHub 网页端修改）
- 修改内容：`"2026 Qoder CLI"` → `"20260114 Qoder CLI"`

**提交信息：**
```
commit 8765733
Author: lvzhaobo <lvzhaobo@gmail.com>
Date:   Wed Jan 14 20:XX:XX 2026 +0800

    Update home page title
```

### 6.2 本地拉取更新

**用户请求：** "GitHub中有代码更新，重新pull拉取代码来更新部署"

**操作步骤：**

1. **检查当前状态：**
```bash
cd /code/mall
git status
git remote -v
```

2. **拉取更新：**
```bash
git pull origin master
```

**拉取结果：**
```
Updating 11bdb23..8765733
Fast-forward
 .babelrc                                           |    12 +
 .editorconfig                                      |     9 +
 .gitignore                                         |    23 +-
 ... (237 个文件更新)
 src/views/home/index.vue                           |     2 +-
 237 files changed, 34245 insertions(+), 188 deletions(-)
```

### 6.3 重新构建和部署

**构建新镜像：**
```bash
cd /code/mall
docker build -t mall/mall-admin-web:latest -f Dockerfile .
```

**构建统计：**
- 构建时间：约 55 秒
- 使用缓存加速构建（步骤 1-5）
- 重新复制源代码和构建（步骤 6-7）

**重启容器：**
```bash
docker stop mall-admin-web
docker rm mall-admin-web
docker run -d --name mall-admin-web \
  --network docker_default \
  -p 80:80 \
  mall/mall-admin-web:latest
```

**验证：**
```bash
docker ps --filter name=mall-admin-web
```

**结果：**
- 前端成功更新
- 首页显示：**"20260114 Qoder CLI"**（包含日期）

---

## 第七阶段：创建部署文档

### 7.1 创建文档

**用户请求：** "把这些内容保存到mall/ai-coding中新的Markdown文件中"

**创建文件：** `/code/mall/ai-coding/deployment-history.md`

**文档内容：**
- 完整的部署流程
- Git 操作详解
- Docker 配置说明
- 问题解决方案
- 访问信息和经验总结

**文档大小：** 355 行

### 7.2 提交文档到 GitHub

**配置 Git：**
```bash
cd /code/mall
git config user.name "lvzhaobo"
git config user.email "lvzhaobo@users.noreply.github.com"
```

**添加并提交：**
```bash
git add ai-coding/deployment-history.md
git commit -m "Add deployment history documentation

- Document complete deployment process from initial setup to updates
- Include Git operations timeline and commit history
- Add Docker configuration and deployment commands
- Record infrastructure setup (MySQL, Redis, backend, frontend)

🤖 Generated with [Qoder](https://qoder.com)"
```

**推送到 GitHub：**
```bash
git push origin master
```

**提交结果：**
- 提交 ID：`9e8ac43`
- 文件：1 个新增
- 行数：355 行插入

---

## 第八阶段：持续更新（第二次）

### 8.1 GitHub 远程更新

**用户请求：** "继续更新"

**拉取最新代码：**
```bash
cd /code/mall
git pull origin master
```

**更新内容：**
```
Updating 9e8ac43..9b413f0
Fast-forward
 src/views/home/index.vue | 2 +-
 1 file changed, 1 insertion(+), 1 deletion(-)
```

**查看详细变更：**
```bash
git diff 9e8ac43 9b413f0
```

**变更内容：**
```diff
-      title="20260114 Qoder CLI"
+      title="20260114 20:28:00 Qoder CLI"
```

**新提交信息：**
```
commit 9b413f0
Author: lvzhaobo <lvzhaobo@gmail.com>
Date:   Wed Jan 14 20:30:53 2026 +0800

    index.vue
```

### 8.2 重新构建和部署

**构建新镜像：**
```bash
cd /code/mall
docker build -t mall/mall-admin-web:latest -f Dockerfile .
```

**构建统计：**
- 构建时间：约 56 秒
- 利用缓存优化构建速度
- webpack 哈希值：`f1a06f0af9a038aaa318`

**重启容器：**
```bash
docker stop mall-admin-web
docker rm mall-admin-web
docker run -d --name mall-admin-web \
  --network docker_default \
  -p 80:80 \
  mall/mall-admin-web:latest
```

**新容器信息：**
- 容器 ID：`d07bae9d4e7e`
- 状态：运行正常
- 端口映射：80:80

**验证结果：**
- 首页显示：**"20260114 20:28:00 Qoder CLI"**（添加了时间戳）
- 所有服务运行正常

---

## 完整时间线总结

### Git 提交历史

| 提交 ID | 提交信息 | 修改内容 | 时间 |
|---------|---------|---------|------|
| `a82ccc6` | Add Docker deployment support | 添加 Docker 配置和首页消息框 | 初始 |
| `787be6f` | Merge mall-admin-web deployment | 合并前端代码到 master | 稍后 |
| `8765733` | Update home page title | 添加日期到消息框 | 20:XX |
| `9e8ac43` | Add deployment history documentation | 创建部署文档 | 20:3X |
| `9b413f0` | index.vue | 添加时间戳 | 20:30 |

### 消息框内容演变

| 版本 | 内容 | 提交/时间点 |
|------|------|------------|
| 1.0 | `"2026 Qoder CLI"` | 本地初始修改（a82ccc6） |
| 2.0 | `"20260114 Qoder CLI"` | GitHub 更新（8765733） |
| 3.0 | `"20260114 20:28:00 Qoder CLI"` | GitHub 更新（9b413f0） |

### Docker 镜像构建历史

| 构建时间 | 原因 | 镜像标签 | 容器 ID |
|---------|------|---------|---------|
| 首次 | 初始部署 | mall/mall-admin-web:latest | 8adf0841 |
| 第二次 | GitHub 更新（添加日期） | mall/mall-admin-web:latest | cd03e002 |
| 第三次 | GitHub 更新（添加时间戳） | mall/mall-admin-web:latest | d07bae9d |

---

## 当前系统状态

### 容器运行状态

```bash
docker ps --filter name=mall
```

**输出：**
```
CONTAINER ID   IMAGE                          COMMAND                  CREATED         STATUS         PORTS
d07bae9d4e7e   mall/mall-admin-web:latest     "/docker-entrypoint.…"   7 seconds ago   Up 6 seconds   0.0.0.0:80->80/tcp
0794a363a2ab   mall/mall-admin:1.0-SNAPSHOT   "java -jar -Dspring.…"   2 hours ago     Up 2 hours     0.0.0.0:8080->8080/tcp
a0577c2d96f4   mysql:5.7                      "docker-entrypoint.s…"   2+ hours ago    Up 2+ hours    0.0.0.0:3306->3306/tcp
65628f17ace5   redis:7                        "docker-entrypoint.s…"   2+ hours ago    Up 2+ hours    0.0.0.0:6379->6379/tcp
```

### 服务访问信息

| 服务 | 地址 | 端口 | 状态 |
|------|------|------|------|
| 管理后台 | http://服务器IP | 80 | ✅ 运行中 |
| 后端 API | http://服务器IP:8080 | 8080 | ✅ 运行中 |
| Swagger 文档 | http://服务器IP:8080/swagger-ui.html | 8080 | ✅ 可访问 |
| MySQL | mysql:3306 | 3306 | ✅ 运行中 |
| Redis | redis:6379 | 6379 | ✅ 运行中 |

### 登录信息

- **用户名：** admin
- **密码：** 871204（已修改）
- **首页显示：** "20260114 20:28:00 Qoder CLI"

### 文件路径

| 类型 | 路径 | 说明 |
|------|------|------|
| 后端项目 | `/code/mall/` | Spring Boot 项目 |
| 前端源码 | `/code/mall/src/` | Vue.js 组件 |
| 首页组件 | `/code/mall/src/views/home/index.vue` | 包含消息框 |
| Docker 配置 | `/code/mall/Dockerfile` | 多阶段构建 |
| Nginx 配置 | `/code/mall/nginx.conf` | 反向代理配置 |
| 部署文档 | `/code/mall/ai-coding/deployment-history.md` | 初版文档 |
| 完整历史 | `/code/mall/ai-coding/complete-deployment-history.md` | 本文档 |

---

## 技术架构详解

### 前端架构

**技术栈：**
- Vue.js 2.x
- Element UI（UI 组件库）
- Vue Router（路由管理）
- Vuex（状态管理）
- Axios（HTTP 客户端）

**构建工具：**
- Webpack 3.12.0
- Babel（ES6+ 转译）
- Node-sass（样式编译）

**生产环境优化：**
- 代码压缩和混淆
- Tree shaking
- 资源哈希命名（缓存优化）
- Gzip 压缩
- CDN 加速（Element UI、Vue 等）

### 后端架构

**技术栈：**
- Spring Boot 2.7.5
- Spring Security（认证授权）
- JWT（令牌认证）
- MyBatis（ORM 框架）
- Swagger（API 文档）
- Redis（缓存）
- MySQL（数据库）

**核心功能模块：**
- 商品管理（PMS）
- 订单管理（OMS）
- 营销管理（SMS）
- 用户管理（UMS）
- 内容管理（CMS）

### 网络架构

**Docker 网络：**
```
docker_default 网络
├── mysql (172.18.0.2)
├── redis (172.18.0.3)
├── mall-admin (172.18.0.4) - 端口 8080
└── mall-admin-web (172.18.0.5) - 端口 80
```

**请求流程：**
```
用户浏览器
    ↓
http://服务器IP:80 (Nginx)
    ↓
/api/* → 反向代理 → http://mall-admin:8080
    ↓
Spring Boot 后端
    ↓
MySQL / Redis
```

---

## 问题与解决方案

### 1. 主机环境缺少工具

**问题：**
- 缺少 docker、cat、grep、ls 等基本命令
- 无法直接执行构建和部署

**解决方案：**
- 使用 `/usr/bin/docker` 完整路径
- 使用 Docker 容器执行构建任务
- 使用 busybox 容器执行文件操作

### 2. 容器网络通信问题

**问题：**
- mall-admin 无法解析 `db` 主机名
- 容器间无法通信

**解决方案：**
- 使用 Docker Compose 创建的 `docker_default` 网络
- 配置容器连接到同一网络
- 使用容器名作为主机名（如 `mysql`、`redis`）

### 3. Git 不相关历史合并

**问题：**
- mall-admin-web 和远程 master 历史不相关
- 普通 merge 失败

**解决方案：**
- 使用 `--allow-unrelated-histories` 参数
- 手动解决冲突文件
- 保留 mall-admin-web 的关键文件

### 4. 前端 API 跨域问题

**问题：**
- 前端直接访问后端 API 会遇到 CORS 错误
- 硬编码的 API 地址不灵活

**解决方案：**
- Nginx 反向代理，将 `/api/*` 转发到后端
- 修改前端配置使用相对路径
- 添加必要的 CORS 响应头

### 5. Docker 镜像过大

**问题：**
- 单阶段构建导致镜像包含构建工具和依赖
- 镜像大小超过 1GB

**解决方案：**
- 使用多阶段构建
- 构建阶段：Node.js 14（构建前端）
- 运行阶段：Nginx Alpine（运行服务）
- 最终镜像：58.2 MB

---

## 经验总结

### 1. Docker 容器化部署优势

✅ **环境一致性**
- 开发、测试、生产环境完全一致
- 避免"在我机器上可以运行"的问题

✅ **资源隔离**
- 每个服务独立运行
- 互不干扰，易于管理

✅ **快速部署**
- 一键构建和启动
- 支持版本回滚

✅ **易于扩展**
- 可以轻松增加容器数量
- 支持负载均衡

### 2. 前后端分离最佳实践

✅ **API 网关模式**
- Nginx 作为统一入口
- 反向代理解决跨域问题
- 统一处理认证和限流

✅ **独立部署**
- 前后端可以独立更新
- 降低发布风险
- 提高开发效率

✅ **清晰的接口规范**
- 使用 Swagger 生成 API 文档
- 前后端约定接口格式
- 减少沟通成本

### 3. Git 协作流程

✅ **分支管理**
- 功能开发使用独立分支
- 测试通过后合并到 master
- 保持 master 分支稳定

✅ **提交规范**
- 清晰的提交信息
- 描述修改内容和原因
- 便于代码审查和回溯

✅ **文档维护**
- 及时更新部署文档
- 记录重要的配置变更
- 方便团队成员查阅

### 4. 持续集成/部署（CI/CD）

💡 **当前流程**
1. 代码提交到 GitHub
2. 本地拉取最新代码
3. Docker 构建新镜像
4. 重启容器应用更新

💡 **改进方向**
- 使用 GitHub Actions 自动构建
- 自动推送镜像到 Docker Hub
- Webhook 触发自动部署
- 自动化测试和验证

---

## 常用命令参考

### Docker 操作

**查看运行的容器：**
```bash
docker ps
docker ps -a  # 包括已停止的容器
docker ps --filter name=mall  # 过滤特定名称
```

**查看容器日志：**
```bash
docker logs mall-admin-web
docker logs -f mall-admin  # 实时查看
docker logs --tail 100 mall-admin  # 查看最后100行
```

**进入容器：**
```bash
docker exec -it mall-admin-web sh
docker exec -it mysql mysql -uroot -proot
```

**重启容器：**
```bash
docker restart mall-admin-web
```

**停止和删除容器：**
```bash
docker stop mall-admin-web
docker rm mall-admin-web
```

**查看镜像：**
```bash
docker images
docker images mall/*
```

**删除镜像：**
```bash
docker rmi mall/mall-admin-web:latest
```

### Git 操作

**查看状态和历史：**
```bash
git status
git log --oneline -10
git log --graph --oneline --all
```

**拉取和推送：**
```bash
git pull origin master
git push origin master
```

**分支操作：**
```bash
git branch  # 查看本地分支
git branch -a  # 查看所有分支
git checkout -b new-branch  # 创建并切换分支
```

**查看差异：**
```bash
git diff
git diff HEAD~1 HEAD  # 比较最近两次提交
git diff branch1 branch2  # 比较两个分支
```

### 数据库操作

**连接 MySQL：**
```bash
docker exec -it mysql mysql -uroot -proot -D mall
```

**常用 SQL：**
```sql
-- 查看所有表
SHOW TABLES;

-- 查看表结构
DESC ums_admin;

-- 查询管理员信息
SELECT * FROM ums_admin WHERE username='admin';

-- 导出数据
mysqldump -uroot -proot mall > mall_backup.sql
```

**连接 Redis：**
```bash
docker exec -it redis redis-cli
```

**Redis 命令：**
```redis
# 查看所有键
KEYS *

# 查看键值
GET key_name

# 查看键类型
TYPE key_name

# 清空数据库
FLUSHDB
```

---

## 附录：配置文件完整内容

### A. Dockerfile（前端）

```dockerfile
# 第一阶段：构建阶段
FROM node:14 AS builder

WORKDIR /app

# 复制 package 文件
COPY package*.json ./

# 设置 npm 镜像加速
RUN npm config set registry https://registry.npmmirror.com

# 安装依赖
RUN npm install

# 复制源代码
COPY . .

# 构建生产环境文件
RUN npm run build

# 第二阶段：nginx 运行阶段
FROM nginx:1.22-alpine

# 复制构建产物到 nginx 目录
COPY --from=builder /app/dist /usr/share/nginx/html

# 复制 nginx 配置
COPY nginx.conf /etc/nginx/conf.d/default.conf

EXPOSE 80

CMD ["nginx", "-g", "daemon off;"]
```

### B. nginx.conf（完整版）

```nginx
server {
    listen       80;
    server_name  localhost;

    # Gzip 压缩配置
    gzip on;
    gzip_vary on;
    gzip_min_length 1024;
    gzip_proxied expired no-cache no-store private auth;
    gzip_types text/plain text/css text/xml text/javascript application/x-javascript application/xml+rss application/json;
    gzip_disable "MSIE [1-6]\.";

    # 前端静态资源
    location / {
        root   /usr/share/nginx/html;
        index  index.html index.htm;
        try_files $uri $uri/ /index.html;
        
        # 缓存配置
        location ~* \.(js|css|png|jpg|jpeg|gif|ico|svg|woff|woff2|ttf|eot)$ {
            expires 30d;
            add_header Cache-Control "public, immutable";
        }
    }

    # 代理后端 API 请求
    location /api/ {
        proxy_pass http://mall-admin:8080/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header X-Forwarded-Proto $scheme;
        
        # 超时配置
        proxy_connect_timeout 60s;
        proxy_send_timeout 60s;
        proxy_read_timeout 60s;
        
        # 缓冲区配置
        proxy_buffer_size 64k;
        proxy_buffers 4 64k;
        proxy_busy_buffers_size 128k;
    }

    # 错误页面
    error_page   500 502 503 504  /50x.html;
    location = /50x.html {
        root   /usr/share/nginx/html;
    }
    
    # 健康检查
    location /health {
        access_log off;
        return 200 "healthy\n";
        add_header Content-Type text/plain;
    }
}
```

### C. config/prod.env.js

```javascript
'use strict'
module.exports = {
  NODE_ENV: '"production"',
  BASE_API: '"/api"'
}
```

### D. docker-compose.yml（示例）

```yaml
version: '3.8'

services:
  mysql:
    image: mysql:5.7
    container_name: mysql
    ports:
      - "3306:3306"
    environment:
      MYSQL_ROOT_PASSWORD: root
      MYSQL_DATABASE: mall
    volumes:
      - mysql-data:/var/lib/mysql
      - ./document/sql:/docker-entrypoint-initdb.d
    networks:
      - docker_default

  redis:
    image: redis:7
    container_name: redis
    ports:
      - "6379:6379"
    networks:
      - docker_default

  mall-admin:
    image: mall/mall-admin:1.0-SNAPSHOT
    container_name: mall-admin
    ports:
      - "8080:8080"
    environment:
      SPRING_DATASOURCE_URL: jdbc:mysql://mysql:3306/mall?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Shanghai
      SPRING_DATASOURCE_USERNAME: root
      SPRING_DATASOURCE_PASSWORD: root
      SPRING_REDIS_HOST: redis
    depends_on:
      - mysql
      - redis
    networks:
      - docker_default

  mall-admin-web:
    image: mall/mall-admin-web:latest
    container_name: mall-admin-web
    ports:
      - "80:80"
    depends_on:
      - mall-admin
    networks:
      - docker_default

networks:
  docker_default:
    driver: bridge

volumes:
  mysql-data:
```

---

## 结语

本文档详细记录了 mall 项目从零开始到完整部署的全过程，包括：

- ✅ 环境搭建（MySQL、Redis、后端、前端）
- ✅ Docker 容器化部署
- ✅ 前后端分离架构
- ✅ Git 版本控制和协作
- ✅ 持续更新和迭代
- ✅ 问题排查和解决

通过这个过程，我们建立了一套完整的开发和部署流程，为后续的功能开发和系统维护打下了坚实的基础。

**项目状态：** 正常运行  
**最后更新：** 2026-01-14 20:39  
**首页显示：** "20260114 20:28:00 Qoder CLI"  
**文档维护：** Qoder CLI

---

**相关文档：**
- [deployment-history.md](./deployment-history.md) - 初版部署文档
- [GitHub 仓库](https://github.com/lvzhaobo/mall.git) - 项目代码

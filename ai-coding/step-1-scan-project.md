## 提示词

### 背景与目标
我是一名开发人员,刚接手这个 mall 电商项目,需要快速掌握项目架构和技术栈,为后续二次开发做准备。

### 扫描任务
请按以下步骤对项目进行全面扫描:

**第一步:项目整体扫描**(保存到本文档)
- 项目基本信息:项目名称、类型、主要功能定位
- 模块结构:列出所有子模块及其职责
- 技术栈:框架版本、核心依赖、中间件
- 构建工具:Maven/Gradle 配置概览
- 配置文件:关键配置项说明(数据库、Redis、端口等)
- 项目依赖关系:模块间的依赖关系图

**第二步:模块详细扫描**(分别保存到独立 Markdown 文档)
针对每个模块(mall-admin、mall-portal、mall-search 等)生成独立文档,包含:
- 模块功能定位与职责边界
- 包结构与分层架构
- 核心类与接口清单
- 关键业务流程说明
- 对外 API 接口列表
- 数据库表关联关系
- 依赖的公共模块

### 输出要求
- 使用 Markdown 表格呈现结构化数据
- 保持内容简洁专业,便于快速查阅
- 突出关键信息,避免过度细节
- 生成的文档需可直接保存使用

## 代码扫描结果

### 一、项目基本信息

| 项目信息 | 内容 |
|---------|------|
| **项目名称** | mall (企业级电商系统) |
| **项目类型** | Spring Boot 单体应用 + 模块化架构 |
| **核心定位** | 完整的电商系统，包含前台商城和后台管理 |
| **版本** | 1.0-SNAPSHOT |
| **GroupId** | com.macro.mall |
| **开发语言** | Java 1.8 |
| **构建工具** | Maven |
| **容器化** | Docker + Docker Compose |

**功能概述**：
- 前台商城：首页门户、商品推荐、商品搜索、商品展示、购物车、订单流程、会员中心、客户服务
- 后台管理：商品管理、订单管理、会员管理、促销管理、运营管理、内容管理、统计报表、权限管理

---

### 二、模块结构

| 模块名称 | 职责定位 | 打包类型 | 依赖关系 |
|---------|---------|---------|----------|
| **mall-common** | 工具类及通用代码 | jar | 无依赖，被其他模块依赖 |
| **mall-mbg** | MyBatis Generator 生成的数据层代码 | jar | 依赖 mall-common |
| **mall-security** | Spring Security 安全框架封装 | jar | 依赖 mall-common |
| **mall-admin** | 后台管理系统接口 | jar (可运行) | 依赖 mall-mbg, mall-security |
| **mall-portal** | 前台商城系统接口 | jar (可运行) | 依赖 mall-mbg, mall-security |
| **mall-search** | Elasticsearch 商品搜索服务 | jar (可运行) | 依赖 mall-mbg (排除 Redis) |
| **mall-demo** | 框架测试代码 | jar | 依赖 mall-common |

**模块依赖关系图**：
```
┌─────────────┐
│ mall-common │  ←── 基础公共模块
└──────┬──────┘
       │
       ├───→ ┌──────────────┐
       │     │ mall-security│  ←── 安全模块
       │     └──────┬───────┘
       │            │
       ├───→ ┌──────┴──────┐
       │     │  mall-mbg   │  ←── 数据层模块
       │     └──────┬──────┘
       │            │
       │     ┌──────┴──────────────┬────────────┐
       │     ↓                     ↓            ↓
       │ ┌───────────┐      ┌─────────────┐ ┌──────────────┐
       │ │mall-admin │      │mall-portal  │ │ mall-search  │
       │ └───────────┘      └─────────────┘ └──────────────┘
       │      ↑                    ↑               ↑
       └──────┴────────────────────┴───────────────┘
          可运行的业务模块（各自独立部署）
```

---

### 三、技术栈

#### 3.1 核心框架

| 技术 | 版本 | 用途 |
|------|------|------|
| **Spring Boot** | 2.7.5 | Web 应用开发框架 |
| **Spring Security** | 2.7.5 | 认证和授权框架 |
| **MyBatis** | 3.5.10 | ORM 持久层框架 |
| **MyBatis Generator** | 1.4.1 | 数据层代码生成器 |

#### 3.2 数据存储

| 中间件 | 版本 | 用途 |
|--------|------|------|
| **MySQL** | 8.0.29 | 关系型数据库 |
| **Redis** | 7.0 | 缓存、Session 存储 |
| **MongoDB** | 5.0 | NoSQL 文档数据库 |
| **Elasticsearch** | 7.17.3 | 商品搜索引擎 |

#### 3.3 中间件与工具

| 组件 | 版本 | 用途 |
|------|------|------|
| **RabbitMQ** | 3.10.5 | 消息队列 |
| **Druid** | 1.2.14 | 数据库连接池 |
| **JWT** | 0.9.1 | Token 认证 |
| **Swagger** | 3.0.0 | API 文档生成 |
| **PageHelper** | 5.3.2 | MyBatis 分页插件 |
| **Hutool** | 5.8.9 | Java 工具类库 |
| **Lombok** | - | Java 语法增强 |
| **Logstash** | 7.17.3 | 日志收集 |
| **Kibana** | 7.17.3 | 日志可视化 |
| **MinIO** | 8.4.5 | 对象存储 |
| **Aliyun OSS** | 2.5.0 | 阿里云对象存储 |
| **Docker** | - | 容器化部署 |
| **Jenkins** | - | CI/CD 自动化部署 |

#### 3.4 前端技术

| 技术 | 用途 |
|------|------|
| **Vue** | 前端框架 |
| **Element UI** | 后台管理 UI 组件库 |
| **uni-app** | 移动端跨平台开发 |

---

### 四、构建工具配置

#### 4.1 Maven 配置概览

| 配置项 | 内容 |
|--------|------|
| **父项目** | spring-boot-starter-parent 2.7.5 |
| **打包方式** | pom (父项目), jar (子模块) |
| **编码** | UTF-8 |
| **跳过测试** | true |
| **镜像仓库** | 阿里云 Maven 镜像 |

#### 4.2 公共依赖

所有模块默认包含：
- `spring-boot-starter-actuator` (监控)
- `spring-boot-starter-aop` (切面编程)
- `spring-boot-starter-test` (测试)
- `hutool-all` (工具类)
- `lombok` (代码简化)

#### 4.3 Docker 构建配置

- 插件：`docker-maven-plugin` 0.40.2
- 基础镜像：`openjdk:8`
- 启动参数：`-Dspring.profiles.active=prod`
- 自动构建：`mvn package` 时触发

---

### 五、关键配置说明

#### 5.1 数据库配置 (application-dev.yml)

```yaml
MySQL:
  地址: localhost:3306
  数据库: mall
  用户名: root
  连接池: Druid (初始5, 最小10, 最大20)
```

#### 5.2 Redis 配置

```yaml
Redis:
  地址: localhost:6379
  数据库: 0
  密码: 空
  超时: 300ms
  业务缓存:
    - ums:admin (管理员信息)
    - ums:resourceList (资源权限列表)
    - 过期时间: 86400秒 (24小时)
```

#### 5.3 JWT 认证配置

```yaml
JWT:
  请求头: Authorization
  密钥: mall-admin-secret
  过期时间: 604800秒 (7天)
  Token前缀: Bearer
```

#### 5.4 对象存储配置

- **MinIO**: localhost:9000, 存储桶 mall
- **阿里云 OSS**: oss-cn-shenzhen, 路径前缀 mall/images/

#### 5.5 安全白名单路径

```
/swagger-ui/**, /admin/login, /admin/register, 
/actuator/**, /druid/**, /minio/upload
```

---

### 六、项目依赖关系

#### 6.1 核心依赖链

```
mall-common (基础)
    ↓
mall-mbg (数据层)  +  mall-security (安全层)
    ↓
mall-admin / mall-portal / mall-search (业务层)
```

#### 6.2 特殊依赖说明

| 模块 | 特殊依赖 | 说明 |
|------|----------|------|
| **mall-admin** | aliyun-oss, minio | 支持多种对象存储 |
| **mall-portal** | mongodb, amqp, alipay-sdk | 订单存储、消息队列、支付集成 |
| **mall-search** | elasticsearch | 排除了 Redis 依赖，使用 ES 独立存储 |
| **mall-security** | spring-security, jjwt | JWT + Spring Security 认证 |

---

### 七、目录结构关键信息

#### 7.1 mall-admin 控制器 (31个)

主要分类：
- **商品管理 (Pms)**: 品牌、分类、属性、SKU 等
- **订单管理 (Oms)**: 订单、退货、物流地址等
- **营销管理 (Sms)**: 优惠券、秒杀、首页推荐等
- **用户管理 (Ums)**: 管理员、角色、权限、菜单等
- **内容管理 (Cms)**: 专题、优选区域等
- **系统功能**: 文件上传 (OSS/MinIO)

#### 7.2 mall-portal 目录

包含 8 个子目录：
- component (组件)
- config (配置)
- controller (13个控制器)
- dao (5个 DAO)
- domain (19个领域对象)
- repository (3个仓储)
- service (16个服务)
- util (工具类)

#### 7.3 mall-search 目录

包含 6 个子目录：
- config (2个配置类)
- controller (1个控制器)
- dao (1个 DAO)
- domain (3个领域模型)
- repository (1个 ES 仓储)
- service (2个搜索服务)

---

### 八、部署与运维

| 部署方式 | 支持情况 |
|---------|----------|
| **本地开发** | ✅ Windows/Linux + IDEA |
| **Docker 单容器** | ✅ 支持 Maven 插件构建镜像 |
| **Docker Compose** | ✅ 一键部署所有服务 |
| **Jenkins CI/CD** | ✅ 自动化构建部署 |

**监控与日志**：
- Actuator 监控端点
- Druid 数据源监控 (/druid/*)
- Logstash + Kibana 日志收集分析

---

### 九、快速启动最小依赖

如果只启动 **mall-admin** 进行后台管理开发：

**必需环境**：
- ✅ MySQL 5.7+ (mall 数据库)
- ✅ Redis 7.0

**可选环境**：
- MinIO/OSS (文件上传功能)
- MongoDB (订单相关功能)
- Elasticsearch (商品搜索功能)
- RabbitMQ (异步消息功能)

**启动命令**：
```bash
cd mall-admin
mvn spring-boot:run -Dspring.profiles.active=dev
```

**访问地址**：
- Swagger 文档: http://localhost:8080/swagger-ui/
- Druid 监控: http://localhost:8080/druid/ (druid/druid)

---

### 十、后续详细扫描计划

**第二步将分别生成以下模块的详细文档**：

1. `mall-admin-detail.md` - 后台管理系统详细扫描
2. `mall-portal-detail.md` - 前台商城系统详细扫描
3. `mall-search-detail.md` - 搜索服务详细扫描
4. `mall-common-detail.md` - 公共模块详细扫描
5. `mall-security-detail.md` - 安全模块详细扫描
6. `mall-mbg-detail.md` - 数据层代码生成详细扫描
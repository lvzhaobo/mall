## 提示词

### 背景与目标
1. 我是一名开发人员,刚接手这个 mall 电商项目,需要快速掌握项目架构和技术栈,为后续二次开发做准备。
2. 前面已经扫描了整个项目、单个模块、价格体系的调用链路，现在需要基于价格体系进行二次开发。
3. 如何发布到阿里云中，选择合适的产品，采用Docker和非Docker两种方式

## 运维方案

### 文档信息

| 项目 | 内容 |
|------|------|
| **文档名称** | mall电商系统 - 阿里云部署运维方案 |
| **文档版本** | v1.0 |
| **创建时间** | 2026-01-02 |
| **适用场景** | 价格体系优化项目上线 |
| **关联文档** | [PRD修正版](./step-4-2-prd-corrected.md), [Roadmap](./step-4-5-roadmap.md), [Todo清单](./todo.md) |

---

## 一、部署方案总览

### 1.1 架构选择

本文档提供**两种部署方案**,适用不同场景:

| 方案 | 适用场景 | 优点 | 缺点 | 成本 |
|------|---------|------|------|------|
| **方案一: Docker容器化** | 测试环境、小规模生产 | 部署快速、资源利用率高、易迁移 | 需要Docker知识 | 低 |
| **方案二: 传统部署** | 大规模生产、高性能要求 | 性能好、稳定性高、运维熟悉 | 部署繁琐、资源浪费 | 中 |

**推荐**: 测试环境使用Docker,生产环境根据规模选择。

---

### 1.2 阿里云产品选型

#### 1.2.1 计算资源

| 组件 | 阿里云产品 | 规格推荐 | 数量 | 月成本 | 说明 |
|------|-----------|---------|-----|--------|------|
| **应用服务器** | ECS通用型g7 | 4C8G | 2台 | ¥600/台 | mall-portal, mall-admin |
| **负载均衡** | SLB标准版 | 按量付费 | 1个 | ¥50 | 流量分发 |
| **容器服务** | ACK托管版(可选) | 3节点 | 1套 | ¥800 | Docker方案使用 |

#### 1.2.2 数据库

| 组件 | 阿里云产品 | 规格推荐 | 数量 | 月成本 | 说明 |
|------|-----------|---------|-----|--------|------|
| **MySQL** | RDS MySQL 5.7 | 4C8G 100GB | 1个 | ¥600 | 主数据库 |
| **Redis** | Redis社区版 | 1G标准版 | 1个 | ¥150 | 缓存 |
| **MongoDB** | MongoDB副本集 | 2C4G 20GB | 1套 | ¥400 | 浏览记录 |
| **Elasticsearch** | ES 7.17 | 2C4G 40GB | 1个 | ¥500 | 商品搜索 |

#### 1.2.3 中间件与存储

| 组件 | 阿里云产品 | 规格推荐 | 数量 | 月成本 | 说明 |
|------|-----------|---------|-----|--------|------|
| **消息队列** | RocketMQ标准版 | 按量付费 | 1个 | ¥200 | 替代RabbitMQ |
| **对象存储** | OSS标准型 | 按量付费 | 1个 | ¥50 | 图片存储 |
| **CDN** | 全站加速DCDN | 按量付费 | 1个 | ¥200 | 静态资源加速 |

#### 1.2.4 监控与安全

| 组件 | 阿里云产品 | 规格推荐 | 数量 | 月成本 | 说明 |
|------|-----------|---------|-----|--------|------|
| **监控告警** | 云监控基础版 | 免费版 | 1个 | ¥0 | 系统监控 |
| **日志服务** | SLS日志服务 | 按量付费 | 1个 | ¥100 | 日志聚合 |
| **Web防火墙** | WAF基础版 | 按量付费 | 1个 | ¥300 | 安全防护 |
| **SSL证书** | 免费版DV证书 | 免费 | 1个 | ¥0 | HTTPS支持 |

#### 1.2.5 成本汇总

| 场景 | 月成本 | 年成本 | 说明 |
|------|-------|--------|------|
| **测试环境** | ¥1,200 | ¥14,400 | 最小化配置 |
| **生产环境(小规模)** | ¥3,950 | ¥47,400 | 完整配置 |
| **生产环境(中等规模)** | ¥8,000+ | ¥96,000+ | 含高可用、弹性伸缩 |

---

## 二、方案一: Docker容器化部署

### 2.1 方案概述

**适用场景**: 测试环境、快速迭代、小规模生产

**技术栈**:
- Docker Engine 20.10+
- Docker Compose 1.29+
- 阿里云 ACR (容器镜像服务)
- 可选: 阿里云 ACK (容器服务Kubernetes)

**优点**:
- ✅ 部署快速 (10分钟完成)
- ✅ 环境一致性强
- ✅ 资源利用率高 (单台ECS可部署多个服务)
- ✅ 易于回滚和扩容

**缺点**:
- ❌ 需要学习Docker知识
- ❌ 单机性能略低于物理部署
- ❌ 复杂故障排查难度增加

---

### 2.2 环境准备

#### 2.2.1 购买阿里云ECS

**规格选择**:
```yaml
实例规格: ecs.g7.xlarge (4C8G)
操作系统: CentOS 7.9 或 Ubuntu 20.04
系统盘: 40GB ESSD云盘
数据盘: 100GB ESSD云盘 (挂载到/data)
网络: VPC专有网络
公网带宽: 5Mbps (按量付费)
安全组: 开放 22, 80, 443, 8080-8085 端口
```

#### 2.2.2 安装Docker

**CentOS 7 安装脚本**:
```bash
#!/bin/bash
# 卸载旧版本
sudo yum remove -y docker docker-client docker-client-latest \
  docker-common docker-latest docker-latest-logrotate \
  docker-logrotate docker-engine

# 安装依赖
sudo yum install -y yum-utils device-mapper-persistent-data lvm2

# 添加阿里云Docker仓库
sudo yum-config-manager --add-repo \
  https://mirrors.aliyun.com/docker-ce/linux/centos/docker-ce.repo

# 安装Docker CE
sudo yum install -y docker-ce docker-ce-cli containerd.io

# 启动Docker
sudo systemctl start docker
sudo systemctl enable docker

# 配置阿里云镜像加速
sudo mkdir -p /etc/docker
sudo tee /etc/docker/daemon.json <<-'EOF'
{
  "registry-mirrors": ["https://xxxxx.mirror.aliyuncs.com"],
  "log-driver": "json-file",
  "log-opts": {
    "max-size": "100m",
    "max-file": "3"
  },
  "storage-driver": "overlay2"
}
EOF

sudo systemctl daemon-reload
sudo systemctl restart docker

# 验证安装
docker --version
docker run hello-world
```

#### 2.2.3 安装Docker Compose

```bash
# 下载Docker Compose
sudo curl -L "https://github.com/docker/compose/releases/download/v2.15.1/docker-compose-$(uname -s)-$(uname -m)" \
  -o /usr/local/bin/docker-compose

# 添加执行权限
sudo chmod +x /usr/local/bin/docker-compose

# 验证安装
docker-compose --version
```

---

### 2.3 配置阿里云容器镜像服务 (ACR)

#### 2.3.1 创建命名空间和镜像仓库

**登录阿里云控制台**:
1. 进入容器镜像服务 ACR → 个人版
2. 创建命名空间: `mall-prod`
3. 创建镜像仓库:
   - `mall-prod/mall-portal`
   - `mall-prod/mall-admin`
   - `mall-prod/mall-search`

**配置访问凭证**:
```bash
# 登录阿里云镜像仓库
docker login --username=your_username registry.cn-hangzhou.aliyuncs.com
# 输入密码
```

---

### 2.4 构建并推送镜像

#### 2.4.1 本地构建方式

```bash
# 进入项目根目录
cd /path/to/mall

# 构建所有模块
mvn clean package -DskipTests

# 构建并推送Docker镜像
mvn docker:build docker:push -pl mall-portal
mvn docker:build docker:push -pl mall-admin
mvn docker:build docker:push -pl mall-search

# 验证镜像已推送
docker images | grep mall
```

---

### 2.5 部署基础设施 (MySQL, Redis, MongoDB)

#### 2.5.1 创建部署目录

```bash
# 创建目录结构
sudo mkdir -p /data/mall/{mysql,redis,mongodb,elasticsearch,nginx,app}
sudo mkdir -p /data/mall/mysql/{data,conf,log}
sudo mkdir -p /data/mall/redis/data
sudo mkdir -p /data/mall/mongodb/db
sudo mkdir -p /data/mall/elasticsearch/{data,plugins}
sudo mkdir -p /data/mall/nginx/{conf,html,logs}

# 设置权限
sudo chmod -R 777 /data/mall
```

#### 2.5.2 创建 `docker-compose-infra.yml`

**文件路径**: `/data/mall/docker-compose-infra.yml`

```yaml
version: '3.8'
services:
  # MySQL 5.7
  mysql:
    image: mysql:5.7
    container_name: mall-mysql
    restart: always
    command: mysqld --character-set-server=utf8mb4 --collation-server=utf8mb4_unicode_ci
    environment:
      MYSQL_ROOT_PASSWORD: YourSecurePassword123!
      TZ: Asia/Shanghai
    ports:
      - "3306:3306"
    volumes:
      - /data/mall/mysql/data:/var/lib/mysql
      - /data/mall/mysql/conf:/etc/mysql/conf.d
      - /data/mall/mysql/log:/var/log/mysql
    networks:
      - mall-network

  # Redis 7.0
  redis:
    image: redis:7.0-alpine
    container_name: mall-redis
    restart: always
    command: redis-server --appendonly yes --requirepass YourRedisPassword123!
    ports:
      - "6379:6379"
    volumes:
      - /data/mall/redis/data:/data
    networks:
      - mall-network

  # MongoDB 4.4
  mongodb:
    image: mongo:4.4
    container_name: mall-mongodb
    restart: always
    environment:
      MONGO_INITDB_ROOT_USERNAME: admin
      MONGO_INITDB_ROOT_PASSWORD: YourMongoPassword123!
      TZ: Asia/Shanghai
    ports:
      - "27017:27017"
    volumes:
      - /data/mall/mongodb/db:/data/db
    networks:
      - mall-network

  # Elasticsearch 7.17
  elasticsearch:
    image: elasticsearch:7.17.3
    container_name: mall-elasticsearch
    restart: always
    environment:
      - discovery.type=single-node
      - ES_JAVA_OPTS=-Xms512m -Xmx1024m
      - TZ=Asia/Shanghai
    ports:
      - "9200:9200"
      - "9300:9300"
    volumes:
      - /data/mall/elasticsearch/data:/usr/share/elasticsearch/data
      - /data/mall/elasticsearch/plugins:/usr/share/elasticsearch/plugins
    networks:
      - mall-network

networks:
  mall-network:
    driver: bridge
```

#### 2.5.3 启动基础设施

```bash
# 启动所有基础设施
cd /data/mall
docker-compose -f docker-compose-infra.yml up -d

# 查看运行状态
docker-compose -f docker-compose-infra.yml ps

# 查看日志
docker-compose -f docker-compose-infra.yml logs -f mysql

# 验证服务
docker exec -it mall-mysql mysql -uroot -p  # 测试MySQL
docker exec -it mall-redis redis-cli -a YourRedisPassword123!  # 测试Redis
```

---

### 2.6 初始化数据库

#### 2.6.1 导入SQL脚本

```bash
# 上传SQL文件到ECS
scp document/sql/mall.sql root@your-ecs-ip:/data/mall/mysql/

# 导入数据库
docker exec -i mall-mysql mysql -uroot -pYourSecurePassword123! <<EOF
CREATE DATABASE IF NOT EXISTS mall DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;
USE mall;
SOURCE /var/lib/mysql/mall.sql;
EOF

# 验证导入
docker exec -it mall-mysql mysql -uroot -pYourSecurePassword123! -e "USE mall; SHOW TABLES;"
```

#### 2.6.2 执行数据库优化 (添加索引)

**根据PRD需求,添加性能优化索引**:

```sql
-- 连接数据库
docker exec -it mall-mysql mysql -uroot -pYourSecurePassword123! mall

-- 添加索引
ALTER TABLE pms_product_ladder ADD INDEX idx_product_id (product_id);
ALTER TABLE pms_product_full_reduction ADD INDEX idx_product_id (product_id);
ALTER TABLE pms_sku_stock ADD INDEX idx_product_id (product_id);

-- 验证索引
SHOW INDEX FROM pms_product_ladder;
SHOW INDEX FROM pms_product_full_reduction;
SHOW INDEX FROM pms_sku_stock;
```

---

## 三、方案二: 传统部署 (非Docker)

### 3.1 方案概述

**适用场景**: 大规模生产环境、对性能要求极高的场景

**优点**:
- ✅ 性能最优 (无容器开销)
- ✅ 运维团队熟悉
- ✅ 故障排查简单
- ✅ 资源隔离性好

**缺点**:
- ❌ 部署流程繁琐
- ❌ 环境一致性难保证
- ❌ 扩容慢 (需要重新配置)

---

### 3.2 环境准备

#### 3.2.1 购买阿里云资源

**ECS规格**:
```yaml
应用服务器:
  - 实例规格: ecs.g7.xlarge (4C8G) × 2台
  - 操作系统: CentOS 7.9
  - 系统盘: 40GB
  - 数据盘: 100GB (挂载到/data)

MySQL:
  - 使用阿里云RDS MySQL 5.7
  - 规格: rds.mysql.s3.large (4C8G)
  - 存储: 100GB SSD

Redis:
  - 使用阿里云Redis社区版
  - 规格: 1GB标准版

MongoDB:
  - 使用阿里云MongoDB副本集
  - 规格: 2C4G
```

#### 3.2.2 安装JDK 1.8

```bash
#!/bin/bash
# 安装OpenJDK 1.8
sudo yum install -y java-1.8.0-openjdk java-1.8.0-openjdk-devel

# 配置环境变量
cat >> ~/.bashrc <<'EOF'
export JAVA_HOME=/usr/lib/jvm/java-1.8.0-openjdk
export PATH=$JAVA_HOME/bin:$PATH
EOF

source ~/.bashrc

# 验证安装
java -version
javac -version
```

---

### 3.3 配置阿里云托管数据库

#### 3.3.1 购买RDS MySQL

**控制台操作**:
1. 登录阿里云控制台 → RDS MySQL
2. 创建实例: 选择MySQL 5.7高可用版
3. 规格: rds.mysql.s3.large (4C8G)
4. 存储: 100GB SSD云盘
5. 网络: 与ECS在同一VPC
6. 白名单: 添加ECS内网IP

**初始化数据库**:
```bash
# 连接RDS
mysql -h rm-xxxxx.mysql.rds.aliyuncs.com -u root -p

# 创建数据库
CREATE DATABASE mall DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_unicode_ci;

# 创建业务用户
CREATE USER 'mall_app'@'%' IDENTIFIED BY 'YourAppPassword123!';
GRANT ALL PRIVILEGES ON mall.* TO 'mall_app'@'%';
FLUSH PRIVILEGES;

# 导入数据
mysql -h rm-xxxxx.mysql.rds.aliyuncs.com -u mall_app -p mall < /path/to/mall.sql

# 添加性能优化索引
USE mall;
ALTER TABLE pms_product_ladder ADD INDEX idx_product_id (product_id);
ALTER TABLE pms_product_full_reduction ADD INDEX idx_product_id (product_id);
ALTER TABLE pms_sku_stock ADD INDEX idx_product_id (product_id);
```

---

## 四、监控与告警

### 4.1 配置阿里云云监控

#### 4.1.1 关键指标监控

| 资源类型 | 监控指标 | 阈值 | 告警级别 |
|---------|---------|------|----------|
| **ECS** | CPU使用率 | > 80% | 警告 |
| | 内存使用率 | > 85% | 警告 |
| | 磁盘使用率 | > 80% | 严重 |
| | 网络流入带宽 | > 80Mbps | 警告 |
| **RDS MySQL** | CPU使用率 | > 70% | 警告 |
| | IOPS使用率 | > 80% | 警告 |
| | 连接数使用率 | > 80% | 严重 |
| | 慢查询数 | > 10/分钟 | 警告 |
| **Redis** | 内存使用率 | > 80% | 警告 |
| | 连接数使用率 | > 80% | 警告 |
| | QPS | > 10000 | 信息 |
| **SLB** | 活跃连接数 | > 5000 | 警告 |
| | 新建连接数 | > 1000/s | 警告 |
| | 后端异常ECS | ≥ 1 | 严重 |

---

## 五、灰度发布流程

### 5.1 灰度发布步骤

**Step 1: 准备新版本镜像**
```bash
# 构建新版本
mvn clean package -DskipTests
docker build -t registry.cn-hangzhou.aliyuncs.com/mall-prod/mall-portal:1.1-SNAPSHOT .
docker push registry.cn-hangzhou.aliyuncs.com/mall-prod/mall-portal:1.1-SNAPSHOT
```

**Step 2: 部署灰度实例**
```bash
# 启动灰度容器
docker run -d --name mall-portal-gray \
  --network mall-network \
  -p 8086:8085 \
  -v /data/mall/app/mall-portal-gray/logs:/var/logs \
  -e SPRING_PROFILES_ACTIVE=prod \
  registry.cn-hangzhou.aliyuncs.com/mall-prod/mall-portal:1.1-SNAPSHOT
```

**Step 3: 配置Nginx按比例分流**
```nginx
upstream mall_portal {
    server localhost:8085 weight=95;  # 老版本 95%
    server localhost:8086 weight=5;   # 灰度版本 5%
}
```

**Step 4: 逐步放量**
```bash
# 观察2天后,调整为20%
# weight=80 (老) + weight=20 (新)

# 再观察1天,调整为50%
# weight=50 (老) + weight=50 (新)

# 确认无问题后,全量切换
# weight=0 (老) + weight=100 (新)

# 停止老版本
docker stop mall-portal
docker rm mall-portal
```

---

## 六、运维指南

### 6.1 日常运维操作

#### 6.1.1 查看应用日志

**Docker方式**:
```bash
# 实时查看日志
docker logs -f mall-portal

# 查看最近100行
docker logs --tail 100 mall-portal

# 查看指定时间段日志
docker logs --since "2026-01-02T10:00:00" --until "2026-01-02T11:00:00" mall-portal
```

**传统方式**:
```bash
# 实时查看日志
tail -f /data/mall/portal/logs/app.log

# 查看systemd日志
journalctl -u mall-portal -f

# 搜索错误日志
grep -i "error" /data/mall/portal/logs/app.log
```

#### 6.1.2 应用重启

**Docker方式**:
```bash
# 重启应用
docker restart mall-portal

# 重启所有应用
docker-compose -f docker-compose-app.yml restart
```

**传统方式**:
```bash
# 重启应用
sudo systemctl restart mall-portal

# 查看状态
sudo systemctl status mall-portal
```

#### 6.1.3 数据库备份

```bash
# 备份RDS MySQL
mysqldump -h rm-xxxxx.mysql.rds.aliyuncs.com \
  -u mall_app -p mall > mall_backup_$(date +%Y%m%d).sql

# 压缩备份
gzip mall_backup_$(date +%Y%m%d).sql

# 上传到OSS
ossutil cp mall_backup_$(date +%Y%m%d).sql.gz oss://mall-backup/
```

---

## 七、故障处理

### 7.1 常见故障及解决方案

| 故障现象 | 可能原因 | 排查步骤 | 解决方案 |
|---------|---------|---------|----------|
| **应用无法启动** | 端口被占用 | `netstat -tuln \| grep 8085` | 停止占用进程或更换端口 |
| | 配置文件错误 | 检查application-prod.yml | 修正配置后重启 |
| | 数据库连接失败 | `telnet rds-host 3306` | 检查白名单、密码、网络 |
| **响应超时** | 数据库慢查询 | 查看RDS慢日志 | 优化SQL、添加索引 |
| | 缓存失效 | 检查Redis连接 | 重启Redis或检查密码 |
| | GC频繁 | 查看JVM GC日志 | 调整堆内存参数 |
| **内存溢出 OOM** | 堆内存不足 | `jmap -heap <pid>` | 增加-Xmx参数 |
| | 内存泄漏 | `jmap -histo <pid>` | 使用MAT工具分析 |
| **CPU使用率高** | 死循环 | `jstack <pid>` | 分析线程栈,修复代码 |
| | GC频繁 | 查看GC日志 | 优化代码,调整参数 |

---

## 八、安全加固

### 8.1 网络安全

**ECS安全组规则**:

| 方向 | 协议 | 端口 | 源地址 | 说明 |
|------|------|------|--------|------|
| 入方向 | TCP | 22 | 运维IP白名单 | SSH登录 |
| 入方向 | TCP | 80 | 0.0.0.0/0 | HTTP |
| 入方向 | TCP | 443 | 0.0.0.0/0 | HTTPS |
| 入方向 | TCP | 8080-8085 | SLB内网IP | 应用端口 |
| 出方向 | ALL | ALL | 0.0.0.0/0 | 允许所有出方向 |

**RDS安全组规则**:
```yaml
白名单:
  - ECS-1内网IP: 172.16.0.10
  - ECS-2内网IP: 172.16.0.11
  - 办公网IP: xxx.xxx.xxx.xxx/24
```

### 8.2 应用安全

**配置HTTPS**:
```nginx
server {
    listen 443 ssl http2;
    server_name mall.example.com;
    
    ssl_certificate /etc/nginx/ssl/mall.crt;
    ssl_certificate_key /etc/nginx/ssl/mall.key;
    ssl_protocols TLSv1.2 TLSv1.3;
    ssl_ciphers HIGH:!aNULL:!MD5;
}
```

---

## 九、总结

### 9.1 方案对比

| 维度 | Docker方案 | 传统方案 | 推荐 |
|------|-----------|----------|------|
| **部署速度** | ⭐⭐⭐⭐⭐ | ⭐⭐ | Docker |
| **运维难度** | ⭐⭐⭐ | ⭐⭐⭐⭐ | 传统 |
| **性能** | ⭐⭐⭐⭐ | ⭐⭐⭐⭐⭐ | 传统 |
| **成本** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | Docker |
| **可扩展性** | ⭐⭐⭐⭐⭐ | ⭐⭐⭐ | Docker |

### 9.2 最佳实践建议

1. **测试环境**: 使用Docker方案,快速部署和迭代
2. **生产环境**: 根据规模选择
   - 小规模 (QPS < 1000): Docker方案
   - 大规模 (QPS > 5000): 传统方案 + 阿里云ACK
3. **数据库**: 优先使用阿里云RDS,避免自建
4. **监控**: 必须配置完整的监控告警体系
5. **灰度发布**: 严格按照 5% → 20% → 50% → 100% 流程

---

**文档创建**: 2026-01-02  
**文档状态**: ✅ 完成  
**下次更新**: 上线后根据实际情况优化
# 部署运维规范

## 1. 环境管理

### 1.1 环境分类
| 环境 | 用途 | 数据 | 访问权限 |
|------|------|------|----------|
| dev | 开发环境 | Mock数据 | 开发人员 |
| test | 测试环境 | 测试数据 | 开发+测试 |
| uat | 预发布环境 | 脱敏生产数据 | 开发+测试+产品 |
| prod | 生产环境 | 真实数据 | 运维+DBA |

### 1.2 配置隔离
```yaml
# application.yml
spring:
  profiles:
    active: ${ENV:dev}  # 环境变量控制

---
# application-dev.yml
spring:
  datasource:
    url: jdbc:mysql://dev-db:3306/mall
    username: dev_user
    password: ${DB_PASSWORD}

redis:
  host: dev-redis
  port: 6379

---
# application-prod.yml
spring:
  datasource:
    url: jdbc:mysql://prod-db:3306/mall
    username: prod_user
    password: ${DB_PASSWORD}

redis:
  host: prod-redis
  port: 6379
```

## 2. 容器化部署

### 2.1 Dockerfile
```dockerfile
FROM openjdk:17-jdk-slim

# 设置工作目录
WORKDIR /app

# 复制jar包
COPY target/mall-service-*.jar app.jar

# 设置时区
ENV TZ=Asia/Shanghai
RUN ln -snf /usr/share/zoneinfo/$TZ /etc/localtime && echo $TZ > /etc/timezone

# JVM参数
ENV JAVA_OPTS="-Xms2g -Xmx2g -XX:+UseG1GC -XX:MaxGCPauseMillis=200"

# 健康检查
HEALTHCHECK --interval=30s --timeout=3s --retries=3 \
  CMD curl -f http://localhost:8080/actuator/health || exit 1

# 暴露端口
EXPOSE 8080

# 启动命令
ENTRYPOINT ["sh", "-c", "java $JAVA_OPTS -jar app.jar"]
```

### 2.2 docker-compose.yml
```yaml
version: '3.8'

services:
  mall-service:
    image: mall-service:1.0.0
    container_name: mall-service
    restart: always
    ports:
      - "8080:8080"
    environment:
      - SPRING_PROFILES_ACTIVE=prod
      - DB_PASSWORD=${DB_PASSWORD}
      - REDIS_PASSWORD=${REDIS_PASSWORD}
    volumes:
      - ./logs:/app/logs
    networks:
      - mall-network
    depends_on:
      - mysql
      - redis
    deploy:
      resources:
        limits:
          cpus: '2'
          memory: 4G
        reservations:
          cpus: '1'
          memory: 2G

  mysql:
    image: mysql:8.0
    container_name: mall-mysql
    restart: always
    environment:
      - MYSQL_ROOT_PASSWORD=${MYSQL_ROOT_PASSWORD}
      - MYSQL_DATABASE=mall
    volumes:
      - mysql-data:/var/lib/mysql
    networks:
      - mall-network

  redis:
    image: redis:7.0
    container_name: mall-redis
    restart: always
    command: redis-server --requirepass ${REDIS_PASSWORD}
    volumes:
      - redis-data:/data
    networks:
      - mall-network

networks:
  mall-network:
    driver: bridge

volumes:
  mysql-data:
  redis-data:
```

## 3. Kubernetes部署

### 3.1 Deployment
```yaml
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mall-service
  namespace: production
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mall-service
  template:
    metadata:
      labels:
        app: mall-service
        version: v1.0.0
    spec:
      containers:
      - name: mall-service
        image: registry.example.com/mall-service:1.0.0
        ports:
        - containerPort: 8080
          name: http
        env:
        - name: SPRING_PROFILES_ACTIVE
          value: "prod"
        - name: DB_PASSWORD
          valueFrom:
            secretKeyRef:
              name: mall-secret
              key: db-password
        resources:
          requests:
            memory: "2Gi"
            cpu: "1000m"
          limits:
            memory: "4Gi"
            cpu: "2000m"
        livenessProbe:
          httpGet:
            path: /actuator/health/liveness
            port: 8080
          initialDelaySeconds: 60
          periodSeconds: 10
          timeoutSeconds: 3
        readinessProbe:
          httpGet:
            path: /actuator/health/readiness
            port: 8080
          initialDelaySeconds: 30
          periodSeconds: 5
          timeoutSeconds: 3
```

### 3.2 Service
```yaml
apiVersion: v1
kind: Service
metadata:
  name: mall-service
  namespace: production
spec:
  selector:
    app: mall-service
  ports:
  - protocol: TCP
    port: 80
    targetPort: 8080
  type: ClusterIP
```

### 3.3 Ingress
```yaml
apiVersion: networking.k8s.io/v1
kind: Ingress
metadata:
  name: mall-ingress
  namespace: production
  annotations:
    nginx.ingress.kubernetes.io/rate-limit: "100"
    nginx.ingress.kubernetes.io/ssl-redirect: "true"
spec:
  ingressClassName: nginx
  tls:
  - hosts:
    - api.example.com
    secretName: tls-secret
  rules:
  - host: api.example.com
    http:
      paths:
      - path: /api/v1
        pathType: Prefix
        backend:
          service:
            name: mall-service
            port:
              number: 80
```

## 4. CI/CD流水线

### 4.1 GitLab CI
```yaml
# .gitlab-ci.yml
stages:
  - build
  - test
  - package
  - deploy

variables:
  MAVEN_OPTS: "-Dmaven.repo.local=.m2/repository"
  IMAGE_NAME: "registry.example.com/mall-service"

# 编译
build:
  stage: build
  image: maven:3.8-openjdk-17
  script:
    - mvn clean compile
  cache:
    paths:
      - .m2/repository
  only:
    - branches

# 单元测试
test:
  stage: test
  image: maven:3.8-openjdk-17
  script:
    - mvn test
    - mvn jacoco:report
  coverage: '/Total.*?([0-9]{1,3})%/'
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml
  only:
    - branches

# 打包Docker镜像
package:
  stage: package
  image: docker:20.10
  services:
    - docker:20.10-dind
  script:
    - mvn clean package -DskipTests
    - docker build -t $IMAGE_NAME:$CI_COMMIT_SHORT_SHA .
    - docker tag $IMAGE_NAME:$CI_COMMIT_SHORT_SHA $IMAGE_NAME:latest
    - docker login -u $CI_REGISTRY_USER -p $CI_REGISTRY_PASSWORD registry.example.com
    - docker push $IMAGE_NAME:$CI_COMMIT_SHORT_SHA
    - docker push $IMAGE_NAME:latest
  only:
    - main

# 部署到K8s
deploy_prod:
  stage: deploy
  image: bitnami/kubectl:latest
  script:
    - kubectl config use-context production
    - kubectl set image deployment/mall-service mall-service=$IMAGE_NAME:$CI_COMMIT_SHORT_SHA -n production
    - kubectl rollout status deployment/mall-service -n production
  only:
    - main
  when: manual  # 手动触发生产部署
```

### 4.2 部署策略

#### 蓝绿部署
```yaml
# 蓝环境（当前生产）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mall-service-blue
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mall-service
      version: blue

---
# 绿环境（新版本）
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mall-service-green
spec:
  replicas: 3
  selector:
    matchLabels:
      app: mall-service
      version: green

---
# Service切换
apiVersion: v1
kind: Service
metadata:
  name: mall-service
spec:
  selector:
    app: mall-service
    version: blue  # 切换到green完成发布
```

#### 金丝雀发布
```yaml
# 90%流量到稳定版本
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mall-service-stable
spec:
  replicas: 9  # 90%

---
# 10%流量到金丝雀版本
apiVersion: apps/v1
kind: Deployment
metadata:
  name: mall-service-canary
spec:
  replicas: 1  # 10%
```

## 5. 配置管理

### 5.1 ConfigMap
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: mall-config
  namespace: production
data:
  application.yml: |
    server:
      port: 8080
    spring:
      datasource:
        url: jdbc:mysql://mysql-service:3306/mall
    logging:
      level:
        root: INFO
```

### 5.2 Secret
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: mall-secret
  namespace: production
type: Opaque
data:
  db-password: cGFzc3dvcmQxMjM=  # base64编码
  redis-password: cmVkaXMxMjM=
```

## 6. 监控告警

### 6.1 健康检查
```java
// Spring Boot Actuator
@Component
public class CustomHealthIndicator implements HealthIndicator {
    
    @Override
    public Health health() {
        // 检查数据库连接
        if (!checkDatabase()) {
            return Health.down()
                .withDetail("database", "connection failed")
                .build();
        }
        
        // 检查Redis连接
        if (!checkRedis()) {
            return Health.down()
                .withDetail("redis", "connection failed")
                .build();
        }
        
        return Health.up().build();
    }
}
```

### 6.2 Prometheus监控
```yaml
# application.yml
management:
  endpoints:
    web:
      exposure:
        include: prometheus,health,info
  metrics:
    export:
      prometheus:
        enabled: true
```

```yaml
# prometheus.yml
scrape_configs:
  - job_name: 'mall-service'
    metrics_path: '/actuator/prometheus'
    static_configs:
      - targets: ['mall-service:8080']
```

### 6.3 告警规则
```yaml
# alert-rules.yml
groups:
  - name: mall-service
    interval: 30s
    rules:
      - alert: HighErrorRate
        expr: rate(http_server_requests_seconds_count{status=~"5.."}[5m]) > 0.05
        for: 5m
        labels:
          severity: critical
        annotations:
          summary: "服务错误率过高"
          description: "{{ $labels.instance }} 错误率超过5%"
      
      - alert: HighResponseTime
        expr: http_server_requests_seconds{quantile="0.95"} > 1
        for: 5m
        labels:
          severity: warning
        annotations:
          summary: "接口响应时间过长"
          description: "{{ $labels.uri }} P95响应时间超过1秒"
```

## 7. 日志管理

### 7.1 日志收集
```yaml
# Filebeat配置
filebeat.inputs:
  - type: log
    enabled: true
    paths:
      - /app/logs/*.log
    fields:
      service: mall-service
      env: production

output.elasticsearch:
  hosts: ["elasticsearch:9200"]
  index: "mall-service-%{+yyyy.MM.dd}"
```

### 7.2 日志查询
```bash
# Kibana查询DSL
GET mall-service-*/_search
{
  "query": {
    "bool": {
      "must": [
        {"match": {"level": "ERROR"}},
        {"range": {"@timestamp": {"gte": "now-1h"}}}
      ]
    }
  }
}
```

## 8. 数据备份

### 8.1 MySQL备份
```bash
#!/bin/bash
# backup-mysql.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/mysql"
DB_NAME="mall"

# 全量备份
mysqldump -h mysql-service -u root -p$DB_PASSWORD \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  $DB_NAME | gzip > $BACKUP_DIR/mall_$DATE.sql.gz

# 删除7天前的备份
find $BACKUP_DIR -name "mall_*.sql.gz" -mtime +7 -delete

# 上传到OSS
ossutil cp $BACKUP_DIR/mall_$DATE.sql.gz oss://backup-bucket/mysql/
```

### 8.2 定时任务
```yaml
# CronJob
apiVersion: batch/v1
kind: CronJob
metadata:
  name: mysql-backup
  namespace: production
spec:
  schedule: "0 2 * * *"  # 每天凌晨2点
  jobTemplate:
    spec:
      template:
        spec:
          containers:
          - name: backup
            image: mysql:8.0
            command:
            - /bin/sh
            - -c
            - /backup/backup-mysql.sh
          restartPolicy: OnFailure
```

## 9. 应急预案

### 9.1 服务回滚
```bash
# K8s回滚
kubectl rollout undo deployment/mall-service -n production

# 回滚到指定版本
kubectl rollout undo deployment/mall-service --to-revision=3 -n production

# 查看回滚状态
kubectl rollout status deployment/mall-service -n production
```

### 9.2 流量切换
```bash
# 关闭故障实例
kubectl scale deployment/mall-service --replicas=0 -n production

# 切换到备用集群
kubectl config use-context backup-cluster
kubectl scale deployment/mall-service --replicas=5 -n production
```

## 10. 禁止事项

### 10.1 部署禁忌
- ❌ 禁止在高峰期（10-22点）部署生产环境
- ❌ 禁止跳过预发布环境直接上生产
- ❌ 禁止未经测试的代码上线
- ❌ 禁止手动修改生产环境配置（使用配置中心）
- ❌ 禁止在生产环境执行DDL（需DBA审批）

### 10.2 权限禁忌
- ❌ 禁止开发人员直接访问生产数据库
- ❌ 禁止在代码中硬编码生产环境配置
- ❌ 禁止使用root用户运行应用

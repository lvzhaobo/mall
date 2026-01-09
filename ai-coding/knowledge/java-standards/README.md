# Java开发规范体系

## 📚 规范体系概览

本目录包含完整的企业级Java开发规范体系，适用于大型企业的多层级、多团队协作场景。

## 📁 规范分类

### 1. 基础规范（必读）
- [Java编码规范](./01-coding-standards.md) - 命名、格式、注释等基础编码规范
- [代码质量规范](./02-code-quality.md) - 代码质量要求、复杂度控制、可读性
- [异常处理规范](./03-exception-handling.md) - 异常捕获、处理、自定义异常

### 2. 架构规范
- [分层架构规范](./04-layered-architecture.md) - Controller、Service、DAO层设计规范
- [微服务开发规范](./05-microservice-standards.md) - 微服务拆分、通信、治理规范
- [API设计规范](./06-api-design.md) - RESTful API设计、版本管理、文档规范

### 3. 框架规范
- [Spring Boot开发规范](./07-springboot-standards.md) - Spring Boot项目结构、配置、最佳实践
- [MyBatis开发规范](./08-mybatis-standards.md) - Mapper、SQL编写、性能优化
- [Spring Cloud规范](./09-springcloud-standards.md) - 服务注册、配置中心、网关规范

### 4. 数据库规范
- [数据库设计规范](./10-database-design.md) - 表设计、索引、分库分表
- [SQL编写规范](./11-sql-standards.md) - SQL语句编写、优化、安全
- [Redis使用规范](./12-redis-standards.md) - 缓存策略、Key设计、性能优化

### 5. 安全规范
- [应用安全规范](./13-security-standards.md) - 认证、授权、加密、防攻击
- [数据安全规范](./14-data-security.md) - 敏感数据处理、日志脱敏、审计
- [接口安全规范](./15-api-security.md) - 接口鉴权、限流、防刷

### 6. 测试规范
- [单元测试规范](./16-unit-test.md) - 测试编写、覆盖率、Mock使用
- [集成测试规范](./17-integration-test.md) - 集成测试策略、环境准备
- [性能测试规范](./18-performance-test.md) - 压测标准、性能指标

### 7. 运维规范
- [日志规范](./19-logging-standards.md) - 日志级别、格式、输出、监控
- [监控告警规范](./20-monitoring-standards.md) - 监控指标、告警阈值、应急响应
- [部署发布规范](./21-deployment-standards.md) - 发布流程、回滚策略、灰度发布

### 8. 团队协作规范
- [Git使用规范](./22-git-standards.md) - 分支管理、提交规范、Code Review
- [文档编写规范](./23-documentation.md) - 接口文档、设计文档、变更记录
- [Code Review规范](./24-code-review.md) - 审查流程、检查清单、最佳实践

## 🎯 适用场景

### 小型项目（5人以下）
**必读规范：**
- Java编码规范
- 代码质量规范
- Spring Boot开发规范
- 日志规范
- Git使用规范

### 中型项目（5-20人）
**必读规范：**
- 基础规范（全部）
- 分层架构规范
- API设计规范
- 数据库设计规范
- 单元测试规范
- 团队协作规范（全部）

### 大型项目（20人以上）
**必读规范：**
- 全部规范

## 📖 学习路径

### 新员工（第1周）
1. Java编码规范
2. 代码质量规范
3. Spring Boot开发规范
4. Git使用规范

### 初级开发（第2-4周）
1. 异常处理规范
2. 分层架构规范
3. API设计规范
4. 数据库设计规范
5. 日志规范

### 中级开发（1-3个月）
1. 微服务开发规范
2. MyBatis开发规范
3. Redis使用规范
4. 应用安全规范
5. 单元测试规范

### 高级开发（3-6个月）
1. Spring Cloud规范
2. 数据安全规范
3. 性能测试规范
4. 监控告警规范
5. Code Review规范

## 🔍 快速查找

### 按问题场景查找

| 问题 | 参考规范 |
|------|---------|
| 类名、方法名怎么命名？ | Java编码规范 |
| 如何处理异常？ | 异常处理规范 |
| Controller怎么写？ | 分层架构规范 |
| API接口如何设计？ | API设计规范 |
| 数据库表如何设计？ | 数据库设计规范 |
| SQL语句如何优化？ | SQL编写规范 |
| 如何使用Redis？ | Redis使用规范 |
| 敏感信息如何加密？ | 数据安全规范 |
| 如何写单元测试？ | 单元测试规范 |
| 日志如何打印？ | 日志规范 |
| Git分支如何管理？ | Git使用规范 |

## ⚡ 常用速查

### 命名速查
```java
// 类名：大驼峰
public class UserService {}

// 方法名：小驼峰
public void getUserById() {}

// 常量：全大写下划线
public static final int MAX_COUNT = 100;

// 变量：小驼峰
String userName = "张三";
```

### 注释速查
```java
/**
 * 用户服务类
 * 
 * @author 张三
 * @date 2024-01-07
 */
public class UserService {
    
    /**
     * 根据ID查询用户
     * 
     * @param userId 用户ID
     * @return 用户信息
     */
    public User getUserById(Long userId) {
        // TODO: 实现查询逻辑
        return null;
    }
}
```

### 异常处理速查
```java
// ✅ 正确
try {
    // 业务逻辑
} catch (SpecificException e) {
    log.error("业务处理失败", e);
    throw new BusinessException("业务处理失败", e);
}

// ❌ 错误
try {
    // 业务逻辑
} catch (Exception e) {
    // 空捕获
}
```

## 📊 规范检查清单

### 代码提交前检查
- [ ] 命名符合规范
- [ ] 代码格式化完成
- [ ] 注释完整清晰
- [ ] 无魔法数字
- [ ] 异常处理正确
- [ ] 日志记录完整
- [ ] 单元测试通过
- [ ] 代码审查通过

### Code Review检查
- [ ] 业务逻辑正确
- [ ] 代码可读性良好
- [ ] 性能符合要求
- [ ] 安全问题已处理
- [ ] 异常处理完善
- [ ] 资源正确释放
- [ ] 日志级别合理
- [ ] 测试覆盖充分

## 📞 技术支持

- 📧 技术规范团队：dev-standards@company.com
- 💬 钉钉群：Java开发规范交流群
- 📖 Wiki：http://wiki.company.com/java-standards

## 🔄 更新记录

| 版本 | 日期 | 说明 |
|------|------|------|
| 1.0.0 | 2024-01-07 | 初始版本，包含24个规范文档 |

---

**注意：** 本规范体系会持续更新，请定期查看最新版本。

# 日志规范

## 1. 日志框架选择

### 1.1 推荐组合
```
SLF4J (门面) + Logback (实现)
```

### 1.2 依赖配置
```xml
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-logging</artifactId>
</dependency>
```

---

## 2. 日志级别规范

### 2.1 日志级别定义

| 级别 | 使用场景 | 示例 |
|------|---------|------|
| ERROR | 系统错误、异常情况 | 数据库连接失败、第三方接口调用失败 |
| WARN | 警告信息、可恢复的异常 | 参数校验失败、业务规则不满足 |
| INFO | 重要业务流程、关键操作 | 用户登录、订单创建、支付成功 |
| DEBUG | 详细的调试信息 | 方法参数、中间结果、SQL语句 |
| TRACE | 最详细的追踪信息 | 框架内部细节、完整调用链路 |

### 2.2 日志级别选择

```java
@Service
@Slf4j
public class OrderService {
    
    public Long createOrder(Order order) {
        // INFO - 重要业务操作
        log.info("开始创建订单，userId: {}, productId: {}", 
            order.getUserId(), order.getProductId());
        
        // DEBUG - 调试信息
        log.debug("订单详细信息: {}", order);
        
        // 业务校验失败 - WARN
        if (order.getQuantity() > 100) {
            log.warn("订单数量超过限制，userId: {}, quantity: {}", 
                order.getUserId(), order.getQuantity());
            throw new BusinessException("订单数量不能超过100");
        }
        
        try {
            // 保存订单
            orderMapper.insert(order);
            
            // INFO - 重要操作成功
            log.info("订单创建成功，orderId: {}, orderNo: {}", 
                order.getId(), order.getOrderNo());
            
            return order.getId();
            
        } catch (Exception e) {
            // ERROR - 系统异常
            log.error("创建订单失败，userId: {}, order: {}", 
                order.getUserId(), order, e);
            throw new SystemException("创建订单失败", e);
        }
    }
}
```

---

## 3. 日志内容规范

### 3.1 日志格式

#### 标准格式
```
[时间] [级别] [线程] [类名] - 消息内容
```

#### 配置示例
```xml
<!-- logback-spring.xml -->
<configuration>
    <property name="LOG_PATTERN" 
              value="%d{yyyy-MM-dd HH:mm:ss.SSS} [%level] [%thread] [%logger{50}] - %msg%n"/>
    
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>${LOG_PATTERN}</pattern>
        </encoder>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
    </root>
</configuration>
```

### 3.2 日志内容要求

#### ✅ 正确示例
```java
// 1. 包含关键业务参数
log.info("用户登录成功，userId: {}, username: {}, ip: {}", 
    userId, username, ipAddress);

// 2. 使用占位符（性能更好）
log.info("订单创建成功，orderId: {}", orderId);

// 3. 记录异常堆栈
log.error("支付失败，orderNo: {}", orderNo, exception);

// 4. 业务流程开始和结束
log.info("开始处理退款，orderId: {}", orderId);
// ... 业务逻辑
log.info("退款处理完成，orderId: {}, refundAmount: {}", orderId, amount);
```

#### ❌ 错误示例
```java
// 1. 使用字符串拼接（性能差）
log.info("用户登录: " + username);  // 不推荐

// 2. 日志信息不明确
log.error("失败");  // 没有说明什么失败

// 3. 没有记录关键参数
log.info("订单创建成功");  // 缺少orderId等关键信息

// 4. 没有记录异常堆栈
log.error("支付失败: " + e.getMessage());  // 缺少堆栈信息
```

---

## 4. 敏感信息脱敏

### 4.1 必须脱敏的信息

| 信息类型 | 脱敏规则 | 示例 |
|---------|---------|------|
| 密码 | 不记录 | 不记录任何密码信息 |
| 身份证号 | 显示前6位后4位 | 110101******1234 |
| 手机号 | 显示前3位后4位 | 138****5678 |
| 银行卡号 | 仅显示后4位 | ****1234 |
| 姓名 | 仅显示姓氏 | 张** |
| 邮箱 | 部分隐藏 | ab***@example.com |

### 4.2 脱敏工具类

```java
/**
 * 日志脱敏工具类
 */
public class MaskUtils {
    
    /**
     * 脱敏手机号
     * 138****5678
     */
    public static String maskMobile(String mobile) {
        if (mobile == null || mobile.length() != 11) {
            return mobile;
        }
        return mobile.substring(0, 3) + "****" + mobile.substring(7);
    }
    
    /**
     * 脱敏身份证号
     * 110101******1234
     */
    public static String maskIdCard(String idCard) {
        if (idCard == null || idCard.length() < 10) {
            return idCard;
        }
        return idCard.substring(0, 6) + "******" + 
               idCard.substring(idCard.length() - 4);
    }
    
    /**
     * 脱敏银行卡号
     * ****1234
     */
    public static String maskBankCard(String bankCard) {
        if (bankCard == null || bankCard.length() < 4) {
            return bankCard;
        }
        return "****" + bankCard.substring(bankCard.length() - 4);
    }
    
    /**
     * 脱敏姓名
     * 张**
     */
    public static String maskName(String name) {
        if (name == null || name.length() == 0) {
            return name;
        }
        return name.charAt(0) + "**";
    }
    
    /**
     * 脱敏邮箱
     * ab***@example.com
     */
    public static String maskEmail(String email) {
        if (email == null || !email.contains("@")) {
            return email;
        }
        String[] parts = email.split("@");
        String prefix = parts[0];
        if (prefix.length() <= 2) {
            return "***@" + parts[1];
        }
        return prefix.substring(0, 2) + "***@" + parts[1];
    }
}
```

### 4.3 使用示例

```java
@Service
@Slf4j
public class UserService {
    
    public void register(User user) {
        // ✅ 正确 - 敏感信息脱敏
        log.info("用户注册，username: {}, mobile: {}, idCard: {}", 
            user.getUsername(),
            MaskUtils.maskMobile(user.getMobile()),
            MaskUtils.maskIdCard(user.getIdCard()));
        
        // ❌ 错误 - 未脱敏
        log.info("用户注册，user: {}", user);  // 可能包含完整敏感信息
    }
}
```

---

## 5. 日志输出规范

### 5.1 控制台输出

#### 开发环境
```yaml
# application-dev.yml
logging:
  level:
    root: INFO
    com.company.project: DEBUG
  pattern:
    console: "%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n"
```

### 5.2 文件输出

#### 生产环境
```yaml
# application-prod.yml
logging:
  level:
    root: INFO
    com.company.project: INFO
  file:
    name: /var/log/app/application.log
    max-size: 100MB
    max-history: 30
```

### 5.3 分类输出

```xml
<!-- logback-spring.xml -->
<configuration>
    <!-- 控制台输出 -->
    <appender name="CONSOLE" class="ch.qos.logback.core.ConsoleAppender">
        <encoder>
            <pattern>%d{HH:mm:ss.SSS} [%thread] %-5level %logger{36} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- 文件输出 - 所有日志 -->
    <appender name="FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/var/log/app/application.log</file>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>/var/log/app/application.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>30</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- 文件输出 - 错误日志 -->
    <appender name="ERROR_FILE" class="ch.qos.logback.core.rolling.RollingFileAppender">
        <file>/var/log/app/error.log</file>
        <filter class="ch.qos.logback.classic.filter.LevelFilter">
            <level>ERROR</level>
            <onMatch>ACCEPT</onMatch>
            <onMismatch>DENY</onMismatch>
        </filter>
        <rollingPolicy class="ch.qos.logback.core.rolling.TimeBasedRollingPolicy">
            <fileNamePattern>/var/log/app/error.%d{yyyy-MM-dd}.log</fileNamePattern>
            <maxHistory>90</maxHistory>
        </rollingPolicy>
        <encoder>
            <pattern>%d{yyyy-MM-dd HH:mm:ss.SSS} [%thread] %-5level %logger{50} - %msg%n</pattern>
        </encoder>
    </appender>
    
    <!-- 项目日志 -->
    <logger name="com.company.project" level="INFO" additivity="false">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
        <appender-ref ref="ERROR_FILE"/>
    </logger>
    
    <!-- 根日志 -->
    <root level="INFO">
        <appender-ref ref="CONSOLE"/>
        <appender-ref ref="FILE"/>
    </root>
</configuration>
```

---

## 6. 特殊场景日志

### 6.1 SQL 日志

```yaml
# application.yml
mybatis-plus:
  configuration:
    log-impl: org.apache.ibatis.logging.stdout.StdOutImpl  # 开发环境

logging:
  level:
    com.company.project.mapper: DEBUG  # 生产环境
```

### 6.2 HTTP 请求日志

```java
/**
 * HTTP请求日志拦截器
 */
@Slf4j
@Component
public class RequestLoggingInterceptor implements HandlerInterceptor {
    
    @Override
    public boolean preHandle(HttpServletRequest request, 
                            HttpServletResponse response, 
                            Object handler) {
        String requestId = UUID.randomUUID().toString();
        request.setAttribute("requestId", requestId);
        
        log.info("HTTP请求开始，requestId: {}, method: {}, uri: {}, params: {}", 
            requestId,
            request.getMethod(),
            request.getRequestURI(),
            request.getParameterMap());
        
        return true;
    }
    
    @Override
    public void afterCompletion(HttpServletRequest request, 
                               HttpServletResponse response, 
                               Object handler, 
                               Exception ex) {
        String requestId = (String) request.getAttribute("requestId");
        
        log.info("HTTP请求结束，requestId: {}, status: {}, duration: {}ms", 
            requestId,
            response.getStatus(),
            System.currentTimeMillis() - (Long) request.getAttribute("startTime"));
    }
}
```

### 6.3 定时任务日志

```java
@Component
@Slf4j
public class DataSyncJob {
    
    @Scheduled(cron = "0 0 2 * * ?")
    public void syncData() {
        long startTime = System.currentTimeMillis();
        
        log.info("开始执行数据同步任务");
        
        try {
            // 执行同步逻辑
            int count = performSync();
            
            log.info("数据同步任务执行成功，同步数量: {}, 耗时: {}ms", 
                count, System.currentTimeMillis() - startTime);
            
        } catch (Exception e) {
            log.error("数据同步任务执行失败，耗时: {}ms", 
                System.currentTimeMillis() - startTime, e);
        }
    }
}
```

---

## 7. 性能优化

### 7.1 避免频繁日志

#### ❌ 错误示例
```java
// 在循环中打印每条记录
for (User user : users) {
    log.info("处理用户: {}", user.getId());  // 不推荐
    processUser(user);
}
```

#### ✅ 正确示例
```java
// 只记录批次信息
log.info("开始批量处理用户，数量: {}", users.size());
for (User user : users) {
    processUser(user);
}
log.info("批量处理完成，成功: {}, 失败: {}", successCount, failCount);
```

### 7.2 使用异步日志

```xml
<!-- logback-spring.xml -->
<configuration>
    <!-- 异步输出 -->
    <appender name="ASYNC_FILE" class="ch.qos.logback.classic.AsyncAppender">
        <discardingThreshold>0</discardingThreshold>
        <queueSize>512</queueSize>
        <appender-ref ref="FILE"/>
    </appender>
    
    <root level="INFO">
        <appender-ref ref="ASYNC_FILE"/>
    </root>
</configuration>
```

---

## 8. 监控告警

### 8.1 错误日志监控

```java
/**
 * 日志监控（集成到监控系统）
 */
@Slf4j
@Component
public class LogMonitor {
    
    @Resource
    private AlertService alertService;
    
    /**
     * 监控ERROR级别日志
     */
    public void monitorError(String message, Throwable throwable) {
        // 记录日志
        log.error(message, throwable);
        
        // 发送告警
        if (isCriticalError(throwable)) {
            alertService.sendAlert(
                "系统错误告警",
                message,
                AlertLevel.HIGH
            );
        }
    }
}
```

---

## 9. 检查清单

### 日志记录检查
- [ ] 使用正确的日志级别
- [ ] 包含关键业务参数
- [ ] 使用占位符而非字符串拼接
- [ ] 记录异常堆栈信息
- [ ] 敏感信息已脱敏
- [ ] 避免在循环中频繁打印
- [ ] 重要业务有开始和结束日志

### 日志配置检查
- [ ] 开发环境配置DEBUG级别
- [ ] 生产环境配置INFO级别
- [ ] 配置日志文件滚动策略
- [ ] 错误日志单独输出
- [ ] 配置合理的保留天数

---

## 10. 最佳实践

### DO（应该做）
✅ 使用SLF4J门面  
✅ 关键业务记录日志  
✅ 敏感信息脱敏  
✅ 使用占位符  
✅ 记录异常堆栈  
✅ 配置日志滚动  

### DON'T（不应该做）
❌ 滥用日志级别  
❌ 记录完整敏感信息  
❌ 循环中频繁打印  
❌ 使用System.out  
❌ 忘记记录关键参数  
❌ 日志信息不明确  

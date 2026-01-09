# 异常处理规范

## 1. 异常分类

### 1.1 异常体系

```
Throwable
├── Error (系统错误，不应捕获)
│   ├── OutOfMemoryError
│   └── StackOverflowError
└── Exception
    ├── RuntimeException (运行时异常，不强制捕获)
    │   ├── NullPointerException
    │   ├── IllegalArgumentException
    │   └── IndexOutOfBoundsException
    └── Checked Exception (受检异常，必须处理)
        ├── IOException
        ├── SQLException
        └── ClassNotFoundException
```

### 1.2 自定义异常分类

```java
/**
 * 业务异常基类
 */
public class BusinessException extends RuntimeException {
    private String code;
    private String message;
    
    public BusinessException(String message) {
        super(message);
        this.message = message;
    }
    
    public BusinessException(String code, String message) {
        super(message);
        this.code = code;
        this.message = message;
    }
}

/**
 * 参数校验异常
 */
public class ValidationException extends BusinessException {
    public ValidationException(String message) {
        super("PARAM_ERROR", message);
    }
}

/**
 * 数据不存在异常
 */
public class DataNotFoundException extends BusinessException {
    public DataNotFoundException(String message) {
        super("DATA_NOT_FOUND", message);
    }
}

/**
 * 系统异常
 */
public class SystemException extends RuntimeException {
    public SystemException(String message, Throwable cause) {
        super(message, cause);
    }
}
```

---

## 2. 异常处理原则

### 2.1 不要捕获 Error

#### ❌ 错误示例
```java
try {
    // 业务代码
} catch (Throwable t) {  // 不要捕获 Throwable
    log.error("错误", t);
}

try {
    // 业务代码
} catch (Error e) {  // 不要捕获 Error
    log.error("错误", e);
}
```

#### ✅ 正确示例
```java
try {
    // 业务代码
} catch (Exception e) {  // 只捕获 Exception
    log.error("错误", e);
}
```

---

### 2.2 不要吞掉异常

#### ❌ 错误示例
```java
// 示例1：空 catch 块
try {
    userService.deleteUser(userId);
} catch (Exception e) {
    // 什么都不做 - 吞掉异常
}

// 示例2：只打印不抛出
try {
    orderService.createOrder(order);
} catch (Exception e) {
    e.printStackTrace();  // 仅打印到控制台
}
```

#### ✅ 正确示例
```java
// 示例1：记录日志并抛出
try {
    userService.deleteUser(userId);
} catch (Exception e) {
    log.error("删除用户失败，userId: {}", userId, e);
    throw new BusinessException("删除用户失败", e);
}

// 示例2：转换为业务异常
try {
    orderService.createOrder(order);
} catch (SQLException e) {
    log.error("创建订单失败，订单信息: {}", order, e);
    throw new SystemException("系统异常，请稍后重试", e);
}
```

---

### 2.3 捕获具体异常

#### ❌ 错误示例
```java
try {
    // 文件操作
    Files.readAllBytes(path);
} catch (Exception e) {  // 过于宽泛
    log.error("文件读取失败", e);
}
```

#### ✅ 正确示例
```java
try {
    Files.readAllBytes(path);
} catch (IOException e) {  // 捕获具体异常
    log.error("文件读取失败，path: {}", path, e);
    throw new SystemException("文件读取失败", e);
} catch (SecurityException e) {
    log.error("文件访问权限不足，path: {}", path, e);
    throw new SystemException("文件访问权限不足", e);
}
```

---

### 2.4 finally 块资源释放

#### ❌ 错误示例
```java
InputStream is = null;
try {
    is = new FileInputStream(file);
    // 处理文件
} catch (IOException e) {
    log.error("文件处理失败", e);
} finally {
    is.close();  // 可能抛出 IOException
}
```

#### ✅ 正确示例
```java
// 方式1：try-with-resources（推荐）
try (InputStream is = new FileInputStream(file)) {
    // 处理文件
} catch (IOException e) {
    log.error("文件处理失败", e);
    throw new SystemException("文件处理失败", e);
}

// 方式2：finally 中安全关闭
InputStream is = null;
try {
    is = new FileInputStream(file);
    // 处理文件
} catch (IOException e) {
    log.error("文件处理失败", e);
    throw new SystemException("文件处理失败", e);
} finally {
    if (is != null) {
        try {
            is.close();
        } catch (IOException e) {
            log.error("关闭文件流失败", e);
        }
    }
}
```

---

## 3. 业务层异常处理

### 3.1 Service 层

#### 规则
- 参数校验失败抛出 `ValidationException`
- 数据不存在抛出 `DataNotFoundException`
- 业务规则不满足抛出 `BusinessException`
- 系统异常转换为 `SystemException`

#### 示例
```java
@Service
@Slf4j
public class OrderService {
    
    @Resource
    private OrderMapper orderMapper;
    
    @Resource
    private InventoryService inventoryService;
    
    /**
     * 创建订单
     */
    @Transactional(rollbackFor = Exception.class)
    public Long createOrder(Order order) {
        // 1. 参数校验
        validateOrder(order);
        
        // 2. 检查库存
        boolean hasStock = inventoryService.checkStock(
            order.getProductId(), 
            order.getQuantity()
        );
        if (!hasStock) {
            throw new BusinessException("商品库存不足");
        }
        
        // 3. 保存订单
        try {
            orderMapper.insert(order);
        } catch (Exception e) {
            log.error("保存订单失败，order: {}", order, e);
            throw new SystemException("系统异常，请稍后重试", e);
        }
        
        return order.getId();
    }
    
    /**
     * 校验订单
     */
    private void validateOrder(Order order) {
        if (order == null) {
            throw new ValidationException("订单信息不能为空");
        }
        if (order.getUserId() == null) {
            throw new ValidationException("用户ID不能为空");
        }
        if (order.getProductId() == null) {
            throw new ValidationException("商品ID不能为空");
        }
        if (order.getQuantity() == null || order.getQuantity() <= 0) {
            throw new ValidationException("商品数量必须大于0");
        }
    }
}
```

---

### 3.2 Controller 层

#### 规则
- Controller 不处理异常，交给全局异常处理器
- 只需要调用 Service 方法，让异常向上抛出

#### 示例
```java
@RestController
@RequestMapping("/api/v1/orders")
@Slf4j
public class OrderController {
    
    @Resource
    private OrderService orderService;
    
    /**
     * 创建订单
     */
    @PostMapping
    public Result<Long> createOrder(@RequestBody Order order) {
        // 直接调用 Service，不捕获异常
        Long orderId = orderService.createOrder(order);
        return Result.success(orderId);
    }
}
```

---

## 4. 全局异常处理

### 4.1 统一异常处理器

```java
/**
 * 全局异常处理器
 */
@RestControllerAdvice
@Slf4j
public class GlobalExceptionHandler {
    
    /**
     * 业务异常
     */
    @ExceptionHandler(BusinessException.class)
    public Result<Void> handleBusinessException(BusinessException e) {
        log.warn("业务异常: {}", e.getMessage());
        return Result.fail(e.getCode(), e.getMessage());
    }
    
    /**
     * 参数校验异常
     */
    @ExceptionHandler(ValidationException.class)
    public Result<Void> handleValidationException(ValidationException e) {
        log.warn("参数校验失败: {}", e.getMessage());
        return Result.fail("PARAM_ERROR", e.getMessage());
    }
    
    /**
     * 数据不存在异常
     */
    @ExceptionHandler(DataNotFoundException.class)
    public Result<Void> handleDataNotFoundException(DataNotFoundException e) {
        log.warn("数据不存在: {}", e.getMessage());
        return Result.fail("DATA_NOT_FOUND", e.getMessage());
    }
    
    /**
     * 参数绑定异常
     */
    @ExceptionHandler(MethodArgumentNotValidException.class)
    public Result<Void> handleMethodArgumentNotValidException(MethodArgumentNotValidException e) {
        BindingResult bindingResult = e.getBindingResult();
        String message = bindingResult.getFieldErrors().stream()
            .map(FieldError::getDefaultMessage)
            .collect(Collectors.joining("; "));
        log.warn("参数绑定异常: {}", message);
        return Result.fail("PARAM_ERROR", message);
    }
    
    /**
     * 系统异常
     */
    @ExceptionHandler(SystemException.class)
    public Result<Void> handleSystemException(SystemException e) {
        log.error("系统异常", e);
        return Result.fail("SYSTEM_ERROR", "系统异常，请稍后重试");
    }
    
    /**
     * 未知异常
     */
    @ExceptionHandler(Exception.class)
    public Result<Void> handleException(Exception e) {
        log.error("未知异常", e);
        return Result.fail("SYSTEM_ERROR", "系统异常，请稍后重试");
    }
}
```

---

### 4.2 统一响应格式

```java
/**
 * 统一响应格式
 */
@Data
public class Result<T> {
    
    /**
     * 响应码
     */
    private String code;
    
    /**
     * 响应消息
     */
    private String message;
    
    /**
     * 响应数据
     */
    private T data;
    
    /**
     * 时间戳
     */
    private Long timestamp;
    
    public static <T> Result<T> success(T data) {
        Result<T> result = new Result<>();
        result.setCode("SUCCESS");
        result.setMessage("操作成功");
        result.setData(data);
        result.setTimestamp(System.currentTimeMillis());
        return result;
    }
    
    public static <T> Result<T> fail(String code, String message) {
        Result<T> result = new Result<>();
        result.setCode(code);
        result.setMessage(message);
        result.setTimestamp(System.currentTimeMillis());
        return result;
    }
}
```

---

## 5. 异常日志规范

### 5.1 日志级别选择

| 异常类型 | 日志级别 | 说明 |
|---------|---------|------|
| ValidationException | WARN | 参数校验失败，用户输入错误 |
| BusinessException | WARN | 业务规则不满足，正常业务场景 |
| DataNotFoundException | WARN | 数据不存在，正常业务场景 |
| SystemException | ERROR | 系统异常，需要排查 |
| Exception | ERROR | 未知异常，需要排查 |

### 5.2 日志内容规范

#### ✅ 正确示例
```java
try {
    orderService.createOrder(order);
} catch (BusinessException e) {
    // 记录关键业务信息
    log.warn("创建订单失败，userId: {}, productId: {}, 原因: {}", 
        order.getUserId(), order.getProductId(), e.getMessage());
    throw e;
}

try {
    userMapper.selectById(userId);
} catch (Exception e) {
    // 记录完整堆栈信息
    log.error("查询用户失败，userId: {}", userId, e);
    throw new SystemException("查询用户失败", e);
}
```

#### ❌ 错误示例
```java
// 示例1：日志信息不足
log.error("失败");  // 没有说明什么失败

// 示例2：没有记录堆栈
log.error("查询用户失败: " + e.getMessage());  // 缺少堆栈信息

// 示例3：重复记录
try {
    userService.deleteUser(userId);
} catch (Exception e) {
    log.error("删除失败", e);  // Controller 记录一次
    throw e;  // Service 又记录一次，导致重复
}
```

---

## 6. 特殊场景处理

### 6.1 事务回滚

#### 规则
- `@Transactional` 默认只对 RuntimeException 回滚
- 必须显式指定 `rollbackFor = Exception.class`

#### 示例
```java
// ✅ 正确
@Transactional(rollbackFor = Exception.class)
public void createOrder(Order order) {
    // 业务代码
}

// ❌ 错误 - 受检异常不回滚
@Transactional
public void createOrder(Order order) throws IOException {
    // 业务代码
    // IOException 不会触发回滚
}
```

---

### 6.2 异步方法异常

#### 规则
- 异步方法的异常不会传播到调用方
- 必须在异步方法内部捕获并处理

#### 示例
```java
@Service
@Slf4j
public class NotificationService {
    
    /**
     * 发送通知（异步）
     */
    @Async
    public void sendNotification(String userId, String message) {
        try {
            // 发送通知逻辑
            sendSms(userId, message);
        } catch (Exception e) {
            // 必须在这里处理异常
            log.error("发送通知失败，userId: {}, message: {}", userId, message, e);
            // 可以记录到数据库或重试队列
        }
    }
}
```

---

## 7. 检查清单

### 异常处理检查
- [ ] 不捕获 Error 和 Throwable
- [ ] 不吞掉异常（空 catch 块）
- [ ] 捕获具体异常而非 Exception
- [ ] 记录异常日志（包含堆栈）
- [ ] 异常信息包含关键业务参数
- [ ] finally 块正确释放资源
- [ ] 使用 try-with-resources
- [ ] 事务方法指定 rollbackFor
- [ ] 异步方法内部处理异常

### 自定义异常检查
- [ ] 继承合适的基类
- [ ] 包含错误码和错误信息
- [ ] 提供多个构造方法
- [ ] 在全局异常处理器中处理

---

## 8. 最佳实践总结

### DO（应该做）
✅ 使用自定义业务异常  
✅ 记录异常日志  
✅ 捕获具体异常  
✅ 使用 try-with-resources  
✅ 全局异常处理器统一处理  
✅ 异常信息对用户友好  

### DON'T（不应该做）
❌ 捕获 Error 和 Throwable  
❌ 空 catch 块吞掉异常  
❌ 只打印异常不抛出  
❌ 返回 null 代替抛异常  
❌ 过度使用异常控制流程  
❌ 在循环中使用异常  

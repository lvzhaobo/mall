# 分层架构规范

## 1. 分层架构概述

### 1.1 标准分层结构

```
┌─────────────────────────────────────┐
│      Controller 层（控制器层）        │  处理HTTP请求/响应
├─────────────────────────────────────┤
│      Service 层（业务逻辑层）         │  业务逻辑处理
├─────────────────────────────────────┤
│      Mapper/DAO 层（数据访问层）      │  数据库操作
├─────────────────────────────────────┤
│      Entity 层（实体层）              │  数据模型
└─────────────────────────────────────┘
```

### 1.2 各层职责

| 层次 | 职责 | 禁止事项 |
|------|------|---------|
| Controller | 接收请求、参数校验、调用Service、返回响应 | 不包含业务逻辑、不直接操作数据库 |
| Service | 业务逻辑处理、事务控制、调用Mapper | 不处理HTTP请求响应 |
| Mapper | 数据库CRUD操作 | 不包含业务逻辑 |
| Entity | 数据模型定义 | 不包含业务逻辑 |

---

## 2. Controller 层规范

### 2.1 基本规范

#### 规则
- 使用 `@RestController` 注解
- 统一使用 `/api/v{version}` 开头
- 方法使用 RESTful 风格
- 返回统一的 Result 格式
- 使用 `@Validated` 进行参数校验

#### 标准模板
```java
@RestController
@RequestMapping("/api/v1/users")
@Slf4j
@Validated
public class UserController {
    
    @Resource
    private UserService userService;
    
    /**
     * 创建用户
     */
    @PostMapping
    public Result<Long> createUser(@RequestBody @Valid UserDTO userDTO) {
        Long userId = userService.createUser(userDTO);
        return Result.success(userId);
    }
    
    /**
     * 查询用户详情
     */
    @GetMapping("/{id}")
    public Result<UserVO> getUserById(@PathVariable Long id) {
        UserVO user = userService.getUserById(id);
        return Result.success(user);
    }
    
    /**
     * 查询用户列表
     */
    @GetMapping
    public Result<List<UserVO>> listUsers(
        @RequestParam(required = false) String keyword,
        @RequestParam(defaultValue = "1") Integer pageNum,
        @RequestParam(defaultValue = "10") Integer pageSize) {
        List<UserVO> users = userService.listUsers(keyword, pageNum, pageSize);
        return Result.success(users);
    }
    
    /**
     * 更新用户
     */
    @PutMapping("/{id}")
    public Result<Void> updateUser(
        @PathVariable Long id,
        @RequestBody @Valid UserDTO userDTO) {
        userService.updateUser(id, userDTO);
        return Result.success();
    }
    
    /**
     * 删除用户
     */
    @DeleteMapping("/{id}")
    public Result<Void> deleteUser(@PathVariable Long id) {
        userService.deleteUser(id);
        return Result.success();
    }
}
```

### 2.2 RESTful API 设计

| 操作 | HTTP方法 | URL | 说明 |
|------|---------|-----|------|
| 创建 | POST | /api/v1/users | 创建用户 |
| 查询单个 | GET | /api/v1/users/{id} | 查询用户详情 |
| 查询列表 | GET | /api/v1/users | 查询用户列表 |
| 更新 | PUT | /api/v1/users/{id} | 更新用户 |
| 删除 | DELETE | /api/v1/users/{id} | 删除用户 |

### 2.3 参数校验

```java
@RestController
@RequestMapping("/api/v1/users")
public class UserController {
    
    @PostMapping
    public Result<Long> createUser(@RequestBody @Valid UserDTO userDTO) {
        // @Valid 自动触发参数校验
        return Result.success(userService.createUser(userDTO));
    }
}

/**
 * UserDTO
 */
@Data
public class UserDTO {
    
    @NotBlank(message = "用户名不能为空")
    @Size(min = 2, max = 20, message = "用户名长度必须在2-20之间")
    private String username;
    
    @NotBlank(message = "密码不能为空")
    @Pattern(regexp = "^(?=.*[0-9])(?=.*[a-z])(?=.*[A-Z]).{8,}$", 
             message = "密码必须包含大小写字母和数字，长度至少8位")
    private String password;
    
    @NotBlank(message = "邮箱不能为空")
    @Email(message = "邮箱格式不正确")
    private String email;
    
    @NotNull(message = "年龄不能为空")
    @Min(value = 18, message = "年龄必须大于等于18")
    @Max(value = 100, message = "年龄必须小于等于100")
    private Integer age;
}
```

---

## 3. Service 层规范

### 3.1 基本规范

#### 规则
- 接口与实现分离
- 使用 `@Service` 注解
- 使用 `@Transactional` 控制事务
- 一个Service对应一个业务领域

#### 标准模板
```java
/**
 * 用户服务接口
 */
public interface UserService {
    
    /**
     * 创建用户
     */
    Long createUser(UserDTO userDTO);
    
    /**
     * 查询用户详情
     */
    UserVO getUserById(Long userId);
    
    /**
     * 查询用户列表
     */
    List<UserVO> listUsers(String keyword, Integer pageNum, Integer pageSize);
    
    /**
     * 更新用户
     */
    void updateUser(Long userId, UserDTO userDTO);
    
    /**
     * 删除用户
     */
    void deleteUser(Long userId);
}

/**
 * 用户服务实现
 */
@Service
@Slf4j
public class UserServiceImpl implements UserService {
    
    @Resource
    private UserMapper userMapper;
    
    @Resource
    private RedisTemplate<String, Object> redisTemplate;
    
    @Override
    @Transactional(rollbackFor = Exception.class)
    public Long createUser(UserDTO userDTO) {
        // 1. 参数校验
        validateUser(userDTO);
        
        // 2. 检查用户名是否已存在
        User existUser = userMapper.selectByUsername(userDTO.getUsername());
        if (existUser != null) {
            throw new BusinessException("用户名已存在");
        }
        
        // 3. 转换DTO为Entity
        User user = convertToEntity(userDTO);
        
        // 4. 加密密码
        user.setPassword(passwordEncoder.encode(userDTO.getPassword()));
        
        // 5. 保存用户
        userMapper.insert(user);
        
        // 6. 清除缓存
        redisTemplate.delete("user:list");
        
        log.info("用户创建成功，userId: {}, username: {}", user.getId(), user.getUsername());
        return user.getId();
    }
    
    @Override
    public UserVO getUserById(Long userId) {
        // 1. 参数校验
        if (userId == null) {
            throw new ValidationException("用户ID不能为空");
        }
        
        // 2. 先查缓存
        String cacheKey = "user:" + userId;
        UserVO cachedUser = (UserVO) redisTemplate.opsForValue().get(cacheKey);
        if (cachedUser != null) {
            return cachedUser;
        }
        
        // 3. 查询数据库
        User user = userMapper.selectById(userId);
        if (user == null) {
            throw new DataNotFoundException("用户不存在");
        }
        
        // 4. 转换为VO
        UserVO userVO = convertToVO(user);
        
        // 5. 写入缓存
        redisTemplate.opsForValue().set(cacheKey, userVO, 1, TimeUnit.HOURS);
        
        return userVO;
    }
    
    @Override
    @Transactional(rollbackFor = Exception.class)
    public void deleteUser(Long userId) {
        // 1. 检查用户是否存在
        User user = userMapper.selectById(userId);
        if (user == null) {
            throw new DataNotFoundException("用户不存在");
        }
        
        // 2. 删除用户
        userMapper.deleteById(userId);
        
        // 3. 清除缓存
        redisTemplate.delete("user:" + userId);
        redisTemplate.delete("user:list");
        
        log.info("用户删除成功，userId: {}", userId);
    }
}
```

### 3.2 事务管理

#### 事务传播行为
```java
// REQUIRED（默认）：支持当前事务，如果不存在则创建新事务
@Transactional(propagation = Propagation.REQUIRED)
public void method1() {
    // 事务操作
}

// REQUIRES_NEW：创建新事务，如果当前存在事务则挂起
@Transactional(propagation = Propagation.REQUIRES_NEW)
public void method2() {
    // 独立事务操作
}

// NOT_SUPPORTED：以非事务方式执行，如果当前存在事务则挂起
@Transactional(propagation = Propagation.NOT_SUPPORTED)
public void method3() {
    // 非事务操作（查询、日志等）
}
```

#### 事务回滚规则
```java
// ✅ 正确：指定所有异常都回滚
@Transactional(rollbackFor = Exception.class)
public void createOrder(Order order) {
    // 业务操作
}

// ❌ 错误：默认只对RuntimeException回滚
@Transactional
public void createOrder(Order order) throws IOException {
    // IOException不会触发回滚
}
```

---

## 4. Mapper 层规范

### 4.1 基本规范

#### 规则
- 使用 `@Mapper` 注解
- 继承 `BaseMapper<T>`
- 方法命名清晰明确
- 复杂SQL使用XML

#### 标准模板
```java
/**
 * 用户Mapper
 */
@Mapper
public interface UserMapper extends BaseMapper<User> {
    
    /**
     * 根据用户名查询用户
     */
    User selectByUsername(@Param("username") String username);
    
    /**
     * 查询用户列表
     */
    List<User> selectUsers(@Param("keyword") String keyword);
    
    /**
     * 批量插入用户
     */
    int batchInsert(@Param("users") List<User> users);
}
```

### 4.2 XML 配置
```xml
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE mapper PUBLIC "-//mybatis.org//DTD Mapper 3.0//EN" 
    "http://mybatis.org/dtd/mybatis-3-mapper.dtd">
<mapper namespace="com.company.project.mapper.UserMapper">
    
    <!-- 根据用户名查询 -->
    <select id="selectByUsername" resultType="com.company.project.entity.User">
        SELECT * FROM t_user
        WHERE username = #{username}
        AND deleted = 0
    </select>
    
    <!-- 查询用户列表 -->
    <select id="selectUsers" resultType="com.company.project.entity.User">
        SELECT * FROM t_user
        WHERE deleted = 0
        <if test="keyword != null and keyword != ''">
            AND (username LIKE CONCAT('%', #{keyword}, '%')
                 OR email LIKE CONCAT('%', #{keyword}, '%'))
        </if>
        ORDER BY create_time DESC
    </select>
    
    <!-- 批量插入 -->
    <insert id="batchInsert">
        INSERT INTO t_user (username, password, email, create_time)
        VALUES
        <foreach collection="users" item="user" separator=",">
            (#{user.username}, #{user.password}, #{user.email}, NOW())
        </foreach>
    </insert>
</mapper>
```

---

## 5. 数据模型规范

### 5.1 Entity（实体类）

```java
/**
 * 用户实体
 */
@Data
@TableName("t_user")
public class User extends BaseEntity {
    
    /**
     * 用户ID
     */
    @TableId(type = IdType.AUTO)
    private Long id;
    
    /**
     * 用户名
     */
    private String username;
    
    /**
     * 密码
     */
    private String password;
    
    /**
     * 邮箱
     */
    private String email;
    
    /**
     * 年龄
     */
    private Integer age;
    
    /**
     * 状态：0-禁用，1-启用
     */
    private Integer status;
}
```

### 5.2 DTO（数据传输对象）

```java
/**
 * 用户DTO（用于接收前端请求）
 */
@Data
public class UserDTO {
    
    @NotBlank(message = "用户名不能为空")
    private String username;
    
    @NotBlank(message = "密码不能为空")
    private String password;
    
    @Email(message = "邮箱格式不正确")
    private String email;
    
    @Min(value = 18, message = "年龄不能小于18")
    private Integer age;
}
```

### 5.3 VO（视图对象）

```java
/**
 * 用户VO（用于返回给前端）
 */
@Data
public class UserVO {
    
    /**
     * 用户ID
     */
    private Long id;
    
    /**
     * 用户名
     */
    private String username;
    
    /**
     * 邮箱（脱敏）
     */
    private String email;
    
    /**
     * 年龄
     */
    private Integer age;
    
    /**
     * 创建时间
     */
    private LocalDateTime createTime;
    
    // 注意：不返回密码等敏感信息
}
```

---

## 6. 层间调用规范

### 6.1 调用原则

```
Controller → Service → Mapper → Database
     ↓          ↓
    VO        Entity
```

#### 规则
- Controller 只能调用 Service
- Service 可以调用 Mapper 和其他 Service
- 禁止跨层调用（Controller直接调用Mapper）
- 禁止反向依赖（Mapper调用Service）

### 6.2 数据转换

```java
@Service
public class UserServiceImpl implements UserService {
    
    /**
     * DTO转Entity
     */
    private User convertToEntity(UserDTO dto) {
        User user = new User();
        BeanUtils.copyProperties(dto, user);
        return user;
    }
    
    /**
     * Entity转VO
     */
    private UserVO convertToVO(User entity) {
        UserVO vo = new UserVO();
        BeanUtils.copyProperties(entity, vo);
        // 邮箱脱敏
        vo.setEmail(maskEmail(entity.getEmail()));
        return vo;
    }
}
```

---

## 7. 检查清单

### Controller 层检查
- [ ] 使用 @RestController 注解
- [ ] URL 统一使用 /api/v{version} 开头
- [ ] 使用 RESTful 风格
- [ ] 参数使用 @Valid 校验
- [ ] 返回统一 Result 格式
- [ ] 不包含业务逻辑
- [ ] 不直接调用 Mapper

### Service 层检查
- [ ] 接口与实现分离
- [ ] 使用 @Service 注解
- [ ] 事务方法添加 @Transactional
- [ ] rollbackFor = Exception.class
- [ ] 参数校验完整
- [ ] 异常处理正确
- [ ] 日志记录完整

### Mapper 层检查
- [ ] 使用 @Mapper 注解
- [ ] 继承 BaseMapper
- [ ] 方法命名清晰
- [ ] SQL语句优化
- [ ] 索引使用正确
- [ ] 避免N+1查询

---

## 8. 最佳实践

### DO（应该做）
✅ 接口与实现分离  
✅ 统一使用Result返回  
✅ 参数校验使用注解  
✅ 事务明确指定回滚规则  
✅ Service方法粒度适中  
✅ 数据转换使用工具方法  

### DON'T（不应该做）
❌ Controller包含业务逻辑  
❌ 跨层调用  
❌ Service方法过于臃肿  
❌ 忘记清除缓存  
❌ 返回Entity给前端  
❌ 密码等敏感信息明文传输  

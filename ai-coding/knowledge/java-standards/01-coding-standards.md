# Java编码规范

## 1. 命名规范

### 1.1 类名规范

#### 规则
- 使用**大驼峰命名法**（UpperCamelCase）
- 类名必须是名词或名词短语
- 避免使用缩写，除非是广为人知的缩写（如DTO、VO、DAO）

#### 示例
```java
// ✅ 正确
public class User {}
public class UserService {}
public class OrderController {}
public class PaymentDTO {}

// ❌ 错误
public class user {}              // 小写开头
public class UserSer {}           // 不清晰的缩写
public class user_service {}      // 使用下划线
public class IUserService {}      // 不要使用I前缀
```

#### 特殊类型命名
| 类型 | 命名格式 | 示例 |
|------|---------|------|
| 实体类 | {业务名} | User, Order, Product |
| DTO | {业务名}DTO | UserDTO, OrderDTO |
| VO | {业务名}VO | UserVO, OrderVO |
| Service | {业务名}Service | UserService, OrderService |
| Controller | {业务名}Controller | UserController |
| Mapper/DAO | {业务名}Mapper | UserMapper, OrderMapper |
| 工具类 | {功能}Util 或 {功能}Utils | StringUtils, DateUtil |
| 常量类 | {业务}Constant 或 {业务}Constants | SystemConstants |

---

### 1.2 方法名规范

#### 规则
- 使用**小驼峰命名法**（lowerCamelCase）
- 方法名必须是动词或动词短语
- 布尔类型的方法使用 is/has/can 开头

#### 示例
```java
// ✅ 正确
public void saveUser() {}
public User getUserById(Long id) {}
public List<User> listUsers() {}
public boolean isValid() {}
public boolean hasPermission() {}
public boolean canDelete() {}

// ❌ 错误
public void SaveUser() {}         // 大写开头
public void save_user() {}        // 使用下划线
public void get() {}              // 不明确
public boolean valid() {}         // 布尔方法应该有is前缀
```

#### 常用方法命名约定
| 操作 | 命名格式 | 示例 |
|------|---------|------|
| 查询单个 | get{Entity}By{Condition} | getUserById, getOrderByNo |
| 查询列表 | list{Entity} | listUsers, listOrders |
| 分页查询 | page{Entity} | pageUsers, pageOrders |
| 新增 | save{Entity} / add{Entity} | saveUser, addOrder |
| 修改 | update{Entity} | updateUser, updateOrder |
| 删除 | delete{Entity} / remove{Entity} | deleteUser, removeOrder |
| 统计 | count{Entity} | countUsers, countOrders |
| 判断存在 | exists{Entity} | existsUser, existsOrder |

---

### 1.3 变量名规范

#### 规则
- 使用**小驼峰命名法**（lowerCamelCase）
- 变量名必须是名词或名词短语
- 避免使用单字母变量名（循环变量除外）

#### 示例
```java
// ✅ 正确
String userName = "张三";
int userAge = 25;
List<User> userList = new ArrayList<>();
boolean isValid = true;

// ❌ 错误
String UserName = "张三";         // 大写开头
String user_name = "张三";        // 使用下划线
String n = "张三";                // 单字母不明确
String s = "张三";                // 不明确的缩写
```

#### 集合类型命名
```java
// ✅ 推荐
List<User> userList = new ArrayList<>();
Set<String> userIdSet = new HashSet<>();
Map<Long, User> userMap = new HashMap<>();

// ⚠️ 可接受但不推荐
List<User> users = new ArrayList<>();
Set<String> userIds = new HashSet<>();
```

---

### 1.4 常量名规范

#### 规则
- 使用**全大写字母**，单词间用下划线分隔
- 常量必须使用 `static final` 修饰

#### 示例
```java
// ✅ 正确
public static final int MAX_COUNT = 100;
public static final String DEFAULT_ENCODING = "UTF-8";
public static final long TIMEOUT_SECONDS = 30L;

// ❌ 错误
public static final int maxCount = 100;           // 小驼峰
public static final String DefaultEncoding = "UTF-8";  // 大驼峰
public static final long timeout_Seconds = 30L;   // 混合命名
```

---

### 1.5 包名规范

#### 规则
- 使用**全小写字母**
- 使用点号分隔，反映模块层次
- 包名使用单数形式

#### 示例
```java
// ✅ 正确
com.company.project.module
com.company.project.controller
com.company.project.service
com.company.project.mapper
com.company.project.entity
com.company.project.dto
com.company.project.util

// ❌ 错误
com.company.Project           // 大写
com.company.project.Controller // 大写
com.company.project.utils     // 复数形式
```

#### 标准包结构
```
com.company.project
├── controller      # 控制器层
├── service         # 服务层
│   └── impl       # 服务实现
├── mapper          # 数据访问层
├── entity          # 实体类
├── dto             # 数据传输对象
├── vo              # 视图对象
├── config          # 配置类
├── constant        # 常量类
├── enums           # 枚举类
├── exception       # 自定义异常
├── util            # 工具类
└── common          # 通用类
```

---

## 2. 代码格式规范

### 2.1 缩进与空格

#### 规则
- 使用 **4个空格** 缩进，禁止使用Tab
- 运算符两侧添加空格
- 逗号后面添加空格
- 关键字与括号之间添加空格

#### 示例
```java
// ✅ 正确
public class User {
    private String name;
    
    public void save(User user) {
        if (user != null) {
            int result = 1 + 2;
            String msg = "Hello, World";
        }
    }
}

// ❌ 错误
public class User{                    // 缺少空格
private String name;                  // 缺少缩进
public void save(User user){          // 缺少空格
if(user!=null){                       // 缺少空格
int result=1+2;                       // 缺少空格
}}}
```

---

### 2.2 换行规范

#### 规则
- 每行代码不超过 **120个字符**
- 运算符换行时，运算符在新行的行首
- 方法调用链换行时，点号在新行的行首

#### 示例
```java
// ✅ 正确 - 运算符换行
String message = "这是一个很长的字符串"
    + "需要分多行显示"
    + "以提高可读性";

// ✅ 正确 - 方法链换行
List<String> result = list.stream()
    .filter(item -> item.length() > 5)
    .map(String::toUpperCase)
    .collect(Collectors.toList());

// ❌ 错误 - 运算符在行尾
String message = "这是一个很长的字符串" +
    "需要分多行显示";
```

---

### 2.3 空行规范

#### 规则
- 不同逻辑块之间添加空行
- 字段与方法之间添加空行
- 方法与方法之间添加空行

#### 示例
```java
// ✅ 正确
public class UserService {
    
    private UserMapper userMapper;
    
    public User getUserById(Long userId) {
        // 参数校验
        if (userId == null) {
            throw new IllegalArgumentException("用户ID不能为空");
        }
        
        // 查询用户
        User user = userMapper.selectById(userId);
        
        // 返回结果
        return user;
    }
    
    public void saveUser(User user) {
        // 实现逻辑
    }
}
```

---

## 3. 注释规范

### 3.1 类注释

#### 规则
- 所有类必须添加类注释
- 使用 `/** */` 格式
- 包含类的功能说明、作者、创建日期

#### 示例
```java
/**
 * 用户服务类
 * 提供用户的增删改查功能
 * 
 * @author 张三
 * @date 2024-01-07
 */
public class UserService {
    // 实现代码
}
```

---

### 3.2 方法注释

#### 规则
- 公共方法必须添加方法注释
- 私有方法可选，但复杂逻辑建议添加
- 说明方法功能、参数、返回值、异常

#### 示例
```java
/**
 * 根据用户ID查询用户信息
 * 
 * @param userId 用户ID，不能为空
 * @return 用户信息，如果不存在返回null
 * @throws IllegalArgumentException 当userId为空时抛出
 */
public User getUserById(Long userId) {
    if (userId == null) {
        throw new IllegalArgumentException("用户ID不能为空");
    }
    return userMapper.selectById(userId);
}
```

---

### 3.3 代码注释

#### 规则
- 关键业务逻辑添加注释
- 复杂算法添加注释
- 临时解决方案使用 TODO/FIXME 标记

#### 示例
```java
public void processOrder(Order order) {
    // 1. 校验订单信息
    validateOrder(order);
    
    // 2. 计算订单金额
    BigDecimal totalAmount = calculateAmount(order);
    
    // 3. 扣减库存
    inventoryService.deductStock(order.getProductId(), order.getQuantity());
    
    // TODO: 后续需要添加积分计算逻辑
    
    // 4. 保存订单
    orderMapper.insert(order);
}
```

---

## 4. 编码最佳实践

### 4.1 避免魔法数字

#### 规则
- 将数字常量定义为有意义的常量
- 使用枚举代替魔法数字

#### 示例
```java
// ❌ 错误 - 使用魔法数字
if (user.getStatus() == 1) {
    // 激活状态
}

// ✅ 正确 - 使用常量
public static final int USER_STATUS_ACTIVE = 1;
if (user.getStatus() == USER_STATUS_ACTIVE) {
    // 激活状态
}

// ✅ 更好 - 使用枚举
public enum UserStatus {
    ACTIVE(1, "激活"),
    INACTIVE(0, "未激活");
    
    private final int code;
    private final String desc;
}

if (user.getStatus() == UserStatus.ACTIVE.getCode()) {
    // 激活状态
}
```

---

### 4.2 字符串拼接

#### 规则
- 少量拼接使用 `+`
- 循环中拼接使用 `StringBuilder`
- 复杂拼接使用 `String.format()` 或 `MessageFormat`

#### 示例
```java
// ✅ 正确 - 少量拼接
String message = "Hello, " + userName;

// ✅ 正确 - 循环中拼接
StringBuilder sb = new StringBuilder();
for (String item : list) {
    sb.append(item).append(",");
}

// ✅ 正确 - 格式化拼接
String message = String.format("用户%s，年龄%d", userName, age);
```

---

### 4.3 集合初始化

#### 规则
- 已知容量时，指定初始容量
- 避免空指针，初始化为空集合

#### 示例
```java
// ✅ 正确 - 指定容量
List<User> userList = new ArrayList<>(100);
Map<Long, User> userMap = new HashMap<>(64);

// ✅ 正确 - 避免空指针
public List<User> listUsers() {
    // 返回空集合而不是null
    return Collections.emptyList();
}

// ❌ 错误
public List<User> listUsers() {
    return null;  // 可能导致空指针
}
```

---

## 5. 检查清单

### 代码提交前检查
- [ ] 类名、方法名、变量名符合命名规范
- [ ] 代码格式化完成（4个空格缩进）
- [ ] 每行不超过120字符
- [ ] 类注释、方法注释完整
- [ ] 无魔法数字
- [ ] 字符串拼接合理
- [ ] 集合初始化正确
- [ ] 无明显的代码坏味道

---

## 6. IDE配置

### IntelliJ IDEA
```
Settings → Editor → Code Style → Java
- Tab size: 4
- Indent: 4
- Continuation indent: 8
- Hard wrap at: 120

Settings → Editor → Inspections
- 启用所有Java代码检查
```

### Eclipse
```
Window → Preferences → Java → Code Style → Formatter
- Indentation: Tab policy = Spaces only
- Tab size: 4
- Line wrapping: Maximum line width = 120
```

---

## 附录：快速参考

### 命名速查表
| 类型 | 命名规则 | 示例 |
|------|---------|------|
| 类名 | 大驼峰 | UserService |
| 方法名 | 小驼峰 | getUserById |
| 变量名 | 小驼峰 | userName |
| 常量名 | 全大写下划线 | MAX_COUNT |
| 包名 | 全小写 | com.company.project |

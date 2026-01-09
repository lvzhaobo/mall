# 接口设计规范

## 1. RESTful设计

### 1.1 资源命名
- **使用名词**：`/users`、`/orders`、`/products`
- **复数形式**：`/users/{id}` 而非 `/user/{id}`
- **层级关系**：`/users/{userId}/orders` 表示用户的订单

### 1.2 HTTP方法
| 方法 | 用途 | 示例 |
|------|------|------|
| GET | 查询资源 | `GET /users/{id}` |
| POST | 创建资源 | `POST /users` |
| PUT | 完整更新 | `PUT /users/{id}` |
| PATCH | 部分更新 | `PATCH /users/{id}` |
| DELETE | 删除资源 | `DELETE /users/{id}` |

### 1.3 URL设计
```
✅ 正确示例：
GET    /api/v1/users?page=1&size=20          # 分页查询用户
GET    /api/v1/users/{id}                    # 查询单个用户
POST   /api/v1/users                         # 创建用户
PUT    /api/v1/users/{id}                    # 更新用户
DELETE /api/v1/users/{id}                    # 删除用户
GET    /api/v1/users/{userId}/orders         # 查询用户订单

❌ 错误示例：
GET    /api/v1/getUserById?id=1              # 不使用动词
POST   /api/v1/user/create                   # 不使用create后缀
GET    /api/v1/users/1/orders/2/items/3      # 层级过深
```

## 2. 统一响应格式

### 2.1 成功响应
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "id": 1,
    "username": "zhangsan",
    "mobile": "13812345678"
  },
  "timestamp": 1704038400000
}
```

### 2.2 分页响应
```json
{
  "code": 200,
  "message": "success",
  "data": {
    "list": [
      {"id": 1, "title": "商品1"},
      {"id": 2, "title": "商品2"}
    ],
    "total": 100,
    "page": 1,
    "size": 20,
    "pages": 5
  },
  "timestamp": 1704038400000
}
```

### 2.3 错误响应
```json
{
  "code": 40001,
  "message": "用户名或密码错误",
  "data": null,
  "timestamp": 1704038400000,
  "trace_id": "a1b2c3d4-e5f6-7890-abcd-ef1234567890"
}
```

## 3. 状态码规范

### 3.1 HTTP状态码
| 状态码 | 说明 | 使用场景 |
|--------|------|----------|
| 200 | 成功 | 请求成功 |
| 201 | 已创建 | POST创建资源成功 |
| 204 | 无内容 | DELETE删除成功 |
| 400 | 请求错误 | 参数校验失败 |
| 401 | 未授权 | 未登录或Token失效 |
| 403 | 禁止访问 | 无权限 |
| 404 | 未找到 | 资源不存在 |
| 429 | 请求过多 | 触发限流 |
| 500 | 服务器错误 | 系统异常 |
| 503 | 服务不可用 | 服务熔断或降级 |

### 3.2 业务状态码
```java
// 成功
20000 - 操作成功

// 客户端错误 4xxxx
40001 - 用户名或密码错误
40002 - 验证码错误
40003 - 手机号格式不正确
40004 - 参数校验失败
40101 - 未登录
40102 - Token已失效
40301 - 无权限访问
40401 - 用户不存在
40402 - 订单不存在

// 服务端错误 5xxxx
50001 - 系统异常
50002 - 数据库异常
50003 - 缓存异常
50004 - 第三方服务异常
```

## 4. 参数设计

### 4.1 查询参数
```java
// ✅ 正确：语义化参数
@GetMapping("/users")
public Result<PageData<UserVO>> listUsers(
    @RequestParam(defaultValue = "1") Integer page,
    @RequestParam(defaultValue = "20") Integer size,
    @RequestParam(required = false) String keyword,
    @RequestParam(required = false) Integer status
) {
    // ...
}

// 请求示例：
// GET /api/v1/users?page=1&size=20&keyword=zhang&status=1
```

### 4.2 路径参数
```java
// ✅ 正确：资源ID使用路径参数
@GetMapping("/users/{id}")
public Result<UserVO> getUser(@PathVariable Long id) {
    // ...
}

// 请求示例：
// GET /api/v1/users/123
```

### 4.3 请求体参数
```java
// ✅ 正确：复杂对象使用请求体
@PostMapping("/users")
public Result<Long> createUser(@RequestBody @Validated UserCreateDTO dto) {
    // ...
}

// 请求体示例：
{
  "username": "zhangsan",
  "mobile": "13812345678",
  "email": "zhangsan@example.com"
}
```

## 5. 接口版本管理

### 5.1 URL版本
```java
// v1版本
@RestController
@RequestMapping("/api/v1/users")
public class UserControllerV1 {
    // ...
}

// v2版本（不兼容变更）
@RestController
@RequestMapping("/api/v2/users")
public class UserControllerV2 {
    // ...
}
```

### 5.2 版本演进
- **兼容性变更**：保持v1不变，新增字段或可选参数
- **不兼容变更**：创建v2版本，v1继续维护3个月后下线
- **废弃通知**：响应头标记：`Deprecated: true`

## 6. 接口文档

### 6.1 Swagger注解
```java
@Api(tags = "用户管理")
@RestController
@RequestMapping("/api/v1/users")
public class UserController {

    @ApiOperation("查询用户详情")
    @ApiImplicitParam(name = "id", value = "用户ID", required = true)
    @GetMapping("/{id}")
    public Result<UserVO> getUser(@PathVariable Long id) {
        // ...
    }

    @ApiOperation("创建用户")
    @PostMapping
    public Result<Long> createUser(@RequestBody @Validated UserCreateDTO dto) {
        // ...
    }
}
```

### 6.2 DTO注释
```java
@Data
@ApiModel("用户创建DTO")
public class UserCreateDTO {
    
    @ApiModelProperty(value = "用户名", required = true, example = "zhangsan")
    @NotBlank(message = "用户名不能为空")
    private String username;
    
    @ApiModelProperty(value = "手机号", required = true, example = "13812345678")
    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
    private String mobile;
    
    @ApiModelProperty(value = "邮箱", example = "zhangsan@example.com")
    @Email(message = "邮箱格式不正确")
    private String email;
}
```

## 7. 接口安全

### 7.1 参数校验
```java
@PostMapping("/users")
public Result<Long> createUser(@RequestBody @Validated UserCreateDTO dto) {
    // @Validated触发JSR303校验
    // ...
}

// DTO中使用校验注解
@NotBlank(message = "用户名不能为空")
@Size(min = 3, max = 20, message = "用户名长度3-20字符")
private String username;

@NotNull(message = "年龄不能为空")
@Min(value = 1, message = "年龄最小为1")
@Max(value = 150, message = "年龄最大为150")
private Integer age;
```

### 7.2 权限控制
```java
@RequiresPermissions("user:update")
@PutMapping("/{id}")
public Result<Void> updateUser(
    @PathVariable Long id,
    @RequestBody UserUpdateDTO dto
) {
    // 校验当前用户是否有权限修改目标用户
    // ...
}
```

### 7.3 防重复提交
```java
@PreventRepeatSubmit(interval = 5000) // 5秒内不允许重复提交
@PostMapping("/orders")
public Result<Long> createOrder(@RequestBody OrderCreateDTO dto) {
    // ...
}
```

## 8. 接口性能

### 8.1 分页查询
```java
// ✅ 正确：强制分页
@GetMapping("/orders")
public Result<PageData<OrderVO>> listOrders(
    @RequestParam(defaultValue = "1") Integer page,
    @RequestParam(defaultValue = "20") Integer size
) {
    // size最大值限制为100
    if (size > 100) {
        size = 100;
    }
    // ...
}
```

### 8.2 批量操作
```java
// ✅ 正确：限制批量数量
@PostMapping("/users/batch")
public Result<Void> batchCreateUsers(@RequestBody List<UserCreateDTO> users) {
    if (users.size() > 100) {
        throw new BusinessException("批量创建最多100条");
    }
    // ...
}
```

### 8.3 异步处理
```java
// 长时间任务使用异步
@PostMapping("/reports/export")
public Result<String> exportReport(@RequestBody ReportQueryDTO dto) {
    // 返回任务ID
    String taskId = reportService.asyncExport(dto);
    return Result.success(taskId);
}

// 轮询查询任务状态
@GetMapping("/reports/export/{taskId}")
public Result<ExportTaskVO> getExportTask(@PathVariable String taskId) {
    // 返回：processing、success、failed
    return Result.success(taskService.getTask(taskId));
}
```

## 9. 接口测试

### 9.1 单元测试
```java
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void testCreateUser() throws Exception {
        String requestBody = """
            {
              "username": "zhangsan",
              "mobile": "13812345678"
            }
            """;
        
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200));
    }
}
```

## 10. 禁止事项

### 10.1 接口设计禁忌
- ❌ 禁止在URL中使用动词：`/getUser`、`/createOrder`
- ❌ 禁止返回敏感信息：密码、完整身份证号
- ❌ 禁止无限制查询：必须分页或限制数量
- ❌ 禁止接口无鉴权：公开接口除外
- ❌ 禁止修改已发布接口：使用版本升级

### 10.2 响应格式禁忌
- ❌ 禁止多种响应格式混用
- ❌ 禁止返回HTML或纯文本（除文件下载）
- ❌ 禁止成功时code不等于200

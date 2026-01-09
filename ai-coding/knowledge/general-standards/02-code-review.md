# 代码评审规范

## 1. 评审原则

### 1.1 核心原则
- **必须评审**：所有代码合并前必须经过评审
- **及时反馈**：评审意见24小时内反馈
- **建设性**：指出问题并提供改进建议
- **相互尊重**：对事不对人，友好沟通

### 1.2 评审目标
- 发现代码缺陷
- 确保代码质量
- 知识分享与传播
- 统一编码风格

## 2. 评审流程

### 2.1 提交评审
```bash
# 1. 创建功能分支
git checkout -b feature/user-login

# 2. 完成开发并自测
# 3. 提交代码
git add .
git commit -m "feat: 实现用户登录功能"

# 4. 推送到远程
git push origin feature/user-login

# 5. 创建Pull Request/Merge Request
```

### 2.2 评审检查项
#### 代码规范
- ✓ 命名是否规范（见名知意）
- ✓ 注释是否充分（复杂逻辑必须注释）
- ✓ 代码格式是否统一（IDE自动格式化）
- ✓ 是否有魔法数字（抽取为常量）

#### 设计质量
- ✓ 是否符合设计原则（SOLID）
- ✓ 是否有代码重复（DRY原则）
- ✓ 方法是否过长（> 50行需拆分）
- ✓ 类是否职责单一

#### 功能实现
- ✓ 是否实现所有需求
- ✓ 边界条件是否处理
- ✓ 异常是否正确处理
- ✓ 是否有单元测试

#### 安全性
- ✓ 是否有SQL注入风险
- ✓ 敏感数据是否加密
- ✓ 权限是否校验
- ✓ 日志是否脱敏

#### 性能
- ✓ 是否有N+1查询
- ✓ 循环内是否有数据库操作
- ✓ 是否有缓存机制
- ✓ 大数据量是否分页

### 2.3 评审反馈
```markdown
## 评审意见

### 必须修改（Blocker）
- [ ] UserService.java:45 - SQL注入风险，使用PreparedStatement
- [ ] OrderController.java:23 - 缺少权限校验

### 建议修改（Major）
- [ ] ProductService.java:67 - 方法过长，建议拆分
- [ ] UserMapper.xml:12 - 避免SELECT *，明确字段

### 可选修改（Minor）
- [ ] OrderDTO.java:8 - 注释可以更详细
- [ ] 代码格式未统一，建议格式化

### 优点
- ✅ 单元测试覆盖完整
- ✅ 异常处理规范
```

### 2.4 修改与合并
1. **开发者修改**：根据评审意见修改代码
2. **重新提交**：推送修改后的代码
3. **确认通过**：评审人确认修改
4. **合并代码**：合并到主分支

## 3. 评审标准

### 3.1 代码示例（错误 vs 正确）

#### 示例1：命名规范
```java
// ❌ 错误：命名不规范
public List<User> getUserList(int a, int b) {
    // ...
}

// ✅ 正确：命名清晰
public List<User> listUsers(int pageNum, int pageSize) {
    // ...
}
```

#### 示例2：魔法数字
```java
// ❌ 错误：魔法数字
if (user.getStatus() == 1) {
    // 已激活
}

// ✅ 正确：使用常量
public enum UserStatus {
    INACTIVE(0, "未激活"),
    ACTIVE(1, "已激活");
}

if (user.getStatus() == UserStatus.ACTIVE.getCode()) {
    // 已激活
}
```

#### 示例3：异常处理
```java
// ❌ 错误：吞掉异常
try {
    userService.createUser(user);
} catch (Exception e) {
    e.printStackTrace();
}

// ✅ 正确：记录日志并抛出业务异常
try {
    userService.createUser(user);
} catch (DuplicateKeyException e) {
    log.error("用户已存在, mobile={}", user.getMobile(), e);
    throw new BusinessException("手机号已注册");
}
```

#### 示例4：SQL注入
```java
// ❌ 错误：SQL注入风险
String sql = "SELECT * FROM user WHERE username = '" + username + "'";
jdbcTemplate.query(sql);

// ✅ 正确：使用预编译
String sql = "SELECT * FROM user WHERE username = ?";
jdbcTemplate.query(sql, username);
```

#### 示例5：性能问题
```java
// ❌ 错误：N+1查询
List<Order> orders = orderMapper.selectAll();
for (Order order : orders) {
    User user = userMapper.selectById(order.getUserId()); // N次查询
    order.setUser(user);
}

// ✅ 正确：批量查询
List<Order> orders = orderMapper.selectAll();
List<Long> userIds = orders.stream()
    .map(Order::getUserId)
    .collect(Collectors.toList());
Map<Long, User> userMap = userMapper.selectByIds(userIds)
    .stream()
    .collect(Collectors.toMap(User::getId, u -> u));
orders.forEach(order -> order.setUser(userMap.get(order.getUserId())));
```

## 4. 评审工具

### 4.1 GitLab/GitHub
- **Pull Request**：提交评审请求
- **Code Review**：在线评审代码
- **Approve/Request Changes**：批准或要求修改

### 4.2 代码扫描工具
- **SonarQube**：代码质量扫描
  - 代码异味检测
  - Bug检测
  - 安全漏洞扫描
  - 覆盖率统计

- **阿里巴巴规约插件**：代码规范检查
  - 命名规范
  - 常量定义
  - 代码格式

### 4.3 自动化检查
```yaml
# .gitlab-ci.yml
code_review:
  stage: review
  script:
    # 代码规范检查
    - mvn checkstyle:check
    
    # 代码质量扫描
    - sonar-scanner \
        -Dsonar.projectKey=mall-service \
        -Dsonar.sources=src/main/java \
        -Dsonar.java.binaries=target/classes
    
    # 安全漏洞扫描
    - mvn dependency-check:check
  rules:
    - if: '$CI_PIPELINE_SOURCE == "merge_request_event"'
```

## 5. 评审分级

### 5.1 严重程度
| 级别 | 说明 | 处理 |
|------|------|------|
| Blocker | 阻塞性问题（安全、功能缺陷） | 必须修改 |
| Major | 严重问题（性能、设计缺陷） | 强烈建议修改 |
| Minor | 轻微问题（代码风格、注释） | 建议修改 |
| Info | 提示信息（优化建议） | 可选修改 |

### 5.2 评审通过标准
- **Blocker = 0**：无阻塞性问题
- **至少1个Approve**：至少一位评审人批准
- **CI通过**：自动化检查全部通过

## 6. 评审最佳实践

### 6.1 评审者
- **及时响应**：24小时内完成评审
- **关注重点**：先看架构设计，再看细节
- **提供建议**：不仅指出问题，还给出方案
- **正向激励**：认可优秀代码

### 6.2 提交者
- **小步提交**：每次改动不超过500行
- **清晰描述**：PR描述清楚改动内容和原因
- **自我检查**：提交前先自己review一遍
- **及时修改**：收到反馈后尽快修改

### 6.3 团队
- **制定标准**：统一代码规范和评审标准
- **定期总结**：分享评审中发现的问题
- **持续改进**：根据反馈优化评审流程

## 7. 评审指标

### 7.1 效率指标
- **评审响应时间**：创建PR到首次反馈的时间
  - 目标：< 24小时
  
- **评审完成时间**：创建PR到合并的时间
  - 目标：< 3天

### 7.2 质量指标
- **缺陷发现率**：评审发现的Bug数 / 总Bug数
  - 目标：> 60%
  
- **一次通过率**：无需修改直接通过的PR比例
  - 目标：< 20%（说明评审有效）

## 8. 特殊场景

### 8.1 紧急修复
- **简化流程**：可先合并后补充评审
- **必须评审**：修复后24小时内补充评审
- **总结改进**：分析问题根因，避免再犯

### 8.2 大型重构
- **提前设计**：先评审设计方案
- **分阶段提交**：按模块拆分，逐步合并
- **充分测试**：回归测试必须充分

### 8.3 新人代码
- **耐心指导**：详细解释问题原因
- **结对编程**：复杂功能可结对开发
- **定期反馈**：每周总结常见问题

## 9. 评审模板

### 9.1 PR描述模板
```markdown
## 变更类型
- [ ] 新功能
- [ ] Bug修复
- [ ] 重构
- [ ] 性能优化
- [ ] 文档更新

## 变更说明
### 背景
xxx需求，需要实现用户登录功能

### 改动内容
1. 新增UserController登录接口
2. 实现JWT Token生成与验证
3. 添加单元测试

### 影响范围
- 新增接口，无影响现有功能
- 新增Redis依赖

## 测试
- [x] 单元测试通过
- [x] 集成测试通过
- [x] 本地功能验证
- [ ] 预发布环境验证

## 截图（如有UI变更）
[截图]

## 相关链接
- 需求文档：http://wiki/xxx
- 技术方案：http://wiki/yyy
```

### 9.2 评审反馈模板
```markdown
## 整体评价
代码实现功能完整，单元测试覆盖充分。有几处需要修改的地方。

## 详细意见

### ⛔ 必须修改
- UserController.java:23
  ```java
  // 问题：未校验权限
  @PostMapping("/delete")
  public Result deleteUser(@RequestParam Long userId) {
  
  // 建议：添加权限校验
  @RequiresPermissions("user:delete")
  @PostMapping("/delete")
  public Result deleteUser(@RequestParam Long userId) {
  ```

### ⚠️ 建议修改
- UserService.java:45
  - 问题：方法过长（80行）
  - 建议：拆分为多个私有方法

### 💡 可选优化
- UserDTO.java
  - 建议：添加字段注释

### ✅ 做得好的地方
- 单元测试覆盖完整
- 异常处理规范
- 代码结构清晰
```

## 10. 禁止事项

### 10.1 评审禁忌
- ❌ 禁止不评审直接合并
- ❌ 禁止流于形式走过场
- ❌ 禁止攻击性评论（"写的什么垃圾代码"）
- ❌ 禁止评审自己的代码
- ❌ 禁止忽视评审意见强行合并

### 10.2 提交禁忌
- ❌ 禁止提交未完成的代码
- ❌ 禁止提交未测试的代码
- ❌ 禁止提交包含临时调试代码
- ❌ 禁止PR过大（> 1000行拆分）
- ❌ 禁止PR描述不清晰

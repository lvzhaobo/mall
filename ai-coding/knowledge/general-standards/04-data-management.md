# 数据管理规范

## 1. 数据分类

### 1.1 数据敏感级别
| 级别 | 说明 | 示例 | 保护措施 |
|------|------|------|----------|
| 高敏感 | 核心机密数据 | 密码、身份证号、银行卡号 | 加密存储+脱敏展示+审计日志 |
| 中敏感 | 个人隐私数据 | 手机号、姓名、地址 | 脱敏展示+访问控制 |
| 低敏感 | 一般业务数据 | 订单ID、商品名称 | 访问控制 |
| 公开 | 可公开数据 | 商品分类、公告 | 无特殊限制 |

## 2. 数据存储

### 2.1 敏感数据加密
```java
// 密码加密
@Service
public class UserService {
    
    @Autowired
    private BCryptPasswordEncoder passwordEncoder;
    
    public void createUser(UserDTO dto) {
        User user = new User();
        // 密码加密存储
        user.setPassword(passwordEncoder.encode(dto.getPassword()));
        userMapper.insert(user);
    }
}

// 身份证号加密
public class IdCardEncryptor {
    
    private static final String ALGORITHM = "AES";
    private static final String KEY = "your-secret-key"; // 使用密钥管理系统
    
    public static String encrypt(String idCard) {
        // AES加密
        return AesUtil.encrypt(idCard, KEY);
    }
    
    public static String decrypt(String encrypted) {
        return AesUtil.decrypt(encrypted, KEY);
    }
}
```

### 2.2 数据脱敏
```java
// 手机号脱敏
public class MaskUtil {
    
    public static String maskMobile(String mobile) {
        if (StringUtils.isEmpty(mobile) || mobile.length() != 11) {
            return mobile;
        }
        return mobile.replaceAll("(\\d{3})\\d{4}(\\d{4})", "$1****$2");
        // 13812345678 → 138****5678
    }
    
    public static String maskIdCard(String idCard) {
        if (StringUtils.isEmpty(idCard) || idCard.length() != 18) {
            return idCard;
        }
        return idCard.replaceAll("(\\d{6})\\d{8}(\\d{4})", "$1********$2");
        // 110101199001011234 → 110101********1234
    }
    
    public static String maskBankCard(String bankCard) {
        if (StringUtils.isEmpty(bankCard) || bankCard.length() < 16) {
            return bankCard;
        }
        return bankCard.replaceAll("(\\d{4})\\d+(\\d{4})", "$1 **** **** $2");
        // 6222021234567890 → 6222 **** **** 7890
    }
}
```

### 2.3 数据保留期限
| 数据类型 | 保留期限 | 归档策略 |
|----------|----------|----------|
| 订单数据 | 3年 | 3个月后归档冷表 |
| 日志数据 | 6个月 | 按月归档压缩 |
| 审计日志 | 3年 | 按月归档 |
| 临时数据 | 7天 | 定时清理 |

## 3. 数据访问

### 3.1 权限控制
```java
// 数据权限注解
@DataPermission(type = DataPermissionType.USER_DATA)
@GetMapping("/orders")
public Result<List<Order>> listOrders() {
    // 只能查询当前用户的订单
    Long userId = SecurityUtil.getCurrentUserId();
    return Result.success(orderService.listByUserId(userId));
}

// 数据权限拦截器
@Aspect
@Component
public class DataPermissionAspect {
    
    @Around("@annotation(dataPermission)")
    public Object around(ProceedingJoinPoint point, DataPermission dataPermission) {
        // 注入数据权限SQL条件
        if (dataPermission.type() == DataPermissionType.USER_DATA) {
            Long userId = SecurityUtil.getCurrentUserId();
            // 添加 WHERE user_id = #{userId} 条件
            DataPermissionHelper.setUserId(userId);
        }
        return point.proceed();
    }
}
```

### 3.2 访问审计
```java
// 敏感操作审计
@AuditLog(operation = "查询用户信息", sensitive = true)
@GetMapping("/users/{id}")
public Result<UserVO> getUser(@PathVariable Long id) {
    return Result.success(userService.getById(id));
}

// 审计日志切面
@Aspect
@Component
public class AuditLogAspect {
    
    @AfterReturning("@annotation(auditLog)")
    public void after(JoinPoint point, AuditLog auditLog) {
        // 记录审计日志
        AuditLogEntity log = new AuditLogEntity();
        log.setUserId(SecurityUtil.getCurrentUserId());
        log.setOperation(auditLog.operation());
        log.setParams(JSON.toJSONString(point.getArgs()));
        log.setIp(RequestUtil.getClientIp());
        log.setCreateTime(new Date());
        auditLogMapper.insert(log);
    }
}
```

## 4. 数据传输

### 4.1 API响应脱敏
```java
@Data
public class UserVO {
    
    private Long id;
    
    private String username;
    
    @JsonSerialize(using = MobileSerializer.class)
    private String mobile;  // 自动脱敏
    
    @JsonIgnore
    private String password;  // 不返回
}

// 手机号序列化器
public class MobileSerializer extends JsonSerializer<String> {
    
    @Override
    public void serialize(String value, JsonGenerator gen, SerializerProvider serializers) 
        throws IOException {
        gen.writeString(MaskUtil.maskMobile(value));
    }
}
```

### 4.2 传输加密
```yaml
# 强制HTTPS
server:
  port: 8443
  ssl:
    enabled: true
    key-store: classpath:keystore.p12
    key-store-password: your-password
    key-store-type: PKCS12
```

## 5. 数据导出

### 5.1 导出审批
```java
// 大量数据导出需审批
@PostMapping("/export")
@RequiresPermissions("order:export")
public Result<String> exportOrders(@RequestBody ExportDTO dto) {
    // 校验导出数量
    int count = orderService.count(dto);
    if (count > 10000) {
        // 超过1万条需要审批
        return Result.error("数据量过大，请联系管理员");
    }
    
    // 异步导出
    String taskId = exportService.asyncExport(dto);
    return Result.success(taskId);
}
```

### 5.2 导出脱敏
```java
// Excel导出自动脱敏
@ExcelProperty("手机号")
@ColumnWidth(20)
@ContentStyle(dataFormat = "@") // 文本格式
private String mobile;

// 导出前脱敏
List<UserExportVO> exportList = userList.stream()
    .map(user -> {
        UserExportVO vo = new UserExportVO();
        BeanUtils.copyProperties(user, vo);
        vo.setMobile(MaskUtil.maskMobile(user.getMobile()));
        vo.setIdCard(MaskUtil.maskIdCard(user.getIdCard()));
        return vo;
    })
    .collect(Collectors.toList());
```

## 6. 数据备份

### 6.1 备份策略
| 备份类型 | 频率 | 保留期限 | 存储位置 |
|----------|------|----------|----------|
| 全量备份 | 每天凌晨2点 | 30天 | OSS |
| 增量备份 | 每6小时 | 7天 | 本地 + OSS |
| 归档备份 | 每月1号 | 3年 | 冷存储 |

### 6.2 备份脚本
```bash
#!/bin/bash
# mysql-backup.sh

DATE=$(date +%Y%m%d_%H%M%S)
BACKUP_DIR="/backup/mysql"
DB_NAME="mall"

# 全量备份
mysqldump -h $DB_HOST -u $DB_USER -p$DB_PASSWORD \
  --single-transaction \
  --routines \
  --triggers \
  --events \
  --hex-blob \
  $DB_NAME | gzip > $BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz

# 上传到OSS
ossutil cp $BACKUP_DIR/${DB_NAME}_${DATE}.sql.gz \
  oss://backup-bucket/mysql/

# 删除30天前的本地备份
find $BACKUP_DIR -name "*.sql.gz" -mtime +30 -delete
```

### 6.3 恢复演练
```bash
# 每季度进行一次恢复演练

# 1. 下载备份文件
ossutil cp oss://backup-bucket/mysql/mall_20240115.sql.gz ./

# 2. 解压
gunzip mall_20240115.sql.gz

# 3. 恢复到测试环境
mysql -h test-db -u root -p mall_test < mall_20240115.sql

# 4. 验证数据完整性
mysql -h test-db -u root -p -e "
  SELECT COUNT(*) FROM mall_test.ums_user;
  SELECT COUNT(*) FROM mall_test.oms_order;
"
```

## 7. 数据清理

### 7.1 清理策略
```java
// 定时清理过期数据
@Scheduled(cron = "0 0 3 * * ?") // 每天凌晨3点
public void cleanExpiredData() {
    // 清理7天前的临时数据
    Date expireTime = DateUtils.addDays(new Date(), -7);
    tempDataMapper.deleteByExpireTime(expireTime);
    
    log.info("清理过期数据完成");
}

// 软删除
@Override
public void deleteUser(Long userId) {
    // 不物理删除，设置删除标记
    User user = new User();
    user.setId(userId);
    user.setIsDeleted(1);
    user.setDeleteTime(new Date());
    userMapper.updateById(user);
}
```

### 7.2 数据归档
```sql
-- 订单归档（按月归档3个月前的订单）
INSERT INTO oms_order_archive 
SELECT * FROM oms_order 
WHERE create_time < DATE_SUB(NOW(), INTERVAL 3 MONTH)
LIMIT 10000;

-- 删除已归档数据
DELETE FROM oms_order 
WHERE create_time < DATE_SUB(NOW(), INTERVAL 3 MONTH)
  AND id IN (SELECT id FROM oms_order_archive)
LIMIT 10000;
```

## 8. 数据质量

### 8.1 数据校验
```java
// DTO参数校验
@Data
public class UserCreateDTO {
    
    @NotBlank(message = "手机号不能为空")
    @Pattern(regexp = "^1[3-9]\\d{9}$", message = "手机号格式不正确")
    private String mobile;
    
    @NotBlank(message = "身份证号不能为空")
    @Pattern(regexp = "^[1-9]\\d{5}(18|19|20)\\d{2}(0[1-9]|1[0-2])(0[1-9]|[12]\\d|3[01])\\d{3}[\\dXx]$", 
             message = "身份证号格式不正确")
    private String idCard;
    
    @Email(message = "邮箱格式不正确")
    private String email;
}
```

### 8.2 数据一致性检查
```java
// 定期检查数据一致性
@Scheduled(cron = "0 0 4 * * ?") // 每天凌晨4点
public void checkDataConsistency() {
    // 检查订单金额与订单明细总金额是否一致
    List<Order> orders = orderMapper.selectInconsistentOrders();
    
    if (!CollectionUtils.isEmpty(orders)) {
        // 记录告警
        log.error("发现{}笔订单金额不一致", orders.size());
        // 发送告警通知
        alertService.sendAlert("数据一致性异常", orders);
    }
}
```

## 9. 数据合规

### 9.1 个人信息保护
```java
// GDPR/PIPL合规：用户注销删除个人信息
@Override
@Transactional(rollbackFor = Exception.class)
public void deleteUserData(Long userId) {
    // 1. 删除用户基础信息
    userMapper.deleteById(userId);
    
    // 2. 删除用户扩展信息
    userDetailMapper.deleteByUserId(userId);
    
    // 3. 删除用户收货地址
    addressMapper.deleteByUserId(userId);
    
    // 4. 脱敏历史订单中的个人信息
    orderMapper.maskUserInfo(userId);
    
    // 5. 记录删除日志
    logService.recordDataDeletion(userId);
}
```

### 9.2 数据跨境传输
```yaml
# 敏感数据禁止跨境传输
data:
  transfer:
    # 允许的国家/地区
    allowed-countries:
      - CN  # 中国大陆
      - HK  # 香港
      - MO  # 澳门
    # 禁止传输的数据类型
    blocked-data-types:
      - ID_CARD
      - BANK_CARD
      - BIOMETRIC
```

## 10. 禁止事项

### 10.1 数据安全禁忌
- ❌ 禁止明文存储密码
- ❌ 禁止在日志中记录敏感信息
- ❌ 禁止通过接口返回完整敏感信息
- ❌ 禁止未加密传输敏感数据
- ❌ 禁止在代码中硬编码加密密钥

### 10.2 数据访问禁忌
- ❌ 禁止越权访问他人数据
- ❌ 禁止批量导出未脱敏数据
- ❌ 禁止在生产环境直连数据库查询
- ❌ 禁止删除审计日志
- ❌ 禁止未经授权查看敏感数据

### 10.3 数据处理禁忌
- ❌ 禁止在客户端存储敏感数据
- ❌ 禁止将敏感数据写入缓存未加密
- ❌ 禁止通过GET请求传输敏感参数
- ❌ 禁止在URL中暴露敏感信息
- ❌ 禁止未经用户同意收集个人信息

# 数据库规范

## 1. 命名规范

### 1.1 表命名
- **小写+下划线**：`user_order`、`product_info`
- **业务前缀**：`ums_user`（用户）、`pms_product`（商品）、`oms_order`（订单）
- **禁止复数**：使用`user`而非`users`

### 1.2 字段命名
- **见名知意**：`create_time`、`update_time`、`is_deleted`
- **布尔字段**：统一使用`is_`前缀，如`is_enabled`
- **禁止保留字**：避免使用`order`、`user`等MySQL保留字

### 1.3 索引命名
- **普通索引**：`idx_字段名`，如`idx_user_id`
- **唯一索引**：`uk_字段名`，如`uk_mobile`
- **联合索引**：`idx_字段1_字段2`，如`idx_user_id_create_time`

## 2. 表设计

### 2.1 必备字段
每张表必须包含：
```sql
CREATE TABLE `xxx_table` (
  `id` BIGINT UNSIGNED NOT NULL AUTO_INCREMENT COMMENT '主键',
  `create_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP COMMENT '创建时间',
  `update_time` DATETIME NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '更新时间',
  `is_deleted` TINYINT(1) NOT NULL DEFAULT 0 COMMENT '逻辑删除(0-未删除 1-已删除)',
  PRIMARY KEY (`id`)
) ENGINE=InnoDB DEFAULT CHARSET=utf8mb4 COMMENT='表注释';
```

### 2.2 字段设计
- **合适的数据类型**：
  - 金额：`DECIMAL(10,2)` 不使用FLOAT/DOUBLE
  - 状态：`TINYINT` 配合枚举类
  - 长文本：`TEXT` 最多255字符用`VARCHAR`
  
- **NOT NULL + 默认值**：避免NULL值
  ```sql
  `status` TINYINT NOT NULL DEFAULT 0 COMMENT '状态(0-待审核 1-已通过 2-已拒绝)'
  `remark` VARCHAR(500) NOT NULL DEFAULT '' COMMENT '备注'
  ```

### 2.3 字段长度
- 手机号：`VARCHAR(11)`
- 用户名：`VARCHAR(32)`
- 邮箱：`VARCHAR(64)`
- 身份证号：`VARCHAR(18)`
- IP地址：`VARCHAR(15)`

## 3. 索引设计

### 3.1 索引原则
- **区分度高的列优先**：状态字段(0/1)不适合建索引
- **最左前缀原则**：联合索引(a,b,c)可用于a、ab、abc查询
- **覆盖索引优化**：索引包含查询字段，避免回表

### 3.2 索引创建
```sql
-- 单列索引
ALTER TABLE `oms_order` ADD INDEX `idx_user_id` (`user_id`);

-- 联合索引（查询条件：user_id + status）
ALTER TABLE `oms_order` ADD INDEX `idx_user_id_status` (`user_id`, `status`);

-- 唯一索引
ALTER TABLE `ums_user` ADD UNIQUE INDEX `uk_mobile` (`mobile`);
```

### 3.3 索引禁忌
- ❌ 不在WHERE条件或JOIN的列上建索引
- ❌ 单表索引数不超过5个
- ❌ 组合索引字段数不超过5个
- ❌ 更新频繁的列不建索引

## 4. SQL编写

### 4.1 查询优化
```sql
-- ✅ 正确：使用具体字段
SELECT id, username, mobile FROM ums_user WHERE id = 1;

-- ❌ 错误：使用SELECT *
SELECT * FROM ums_user WHERE id = 1;

-- ✅ 正确：使用LIMIT分页
SELECT id, title FROM article ORDER BY create_time DESC LIMIT 0, 20;

-- ❌ 错误：不加LIMIT查询大表
SELECT id, title FROM article ORDER BY create_time DESC;
```

### 4.2 JOIN优化
- **小表驱动大表**：小表在前，大表在后
- **JOIN字段建索引**：关联字段必须有索引
- **避免多表JOIN**：超过3张表考虑分步查询

```sql
-- ✅ 正确：小表驱动
SELECT o.* 
FROM oms_order o
INNER JOIN ums_user u ON o.user_id = u.id
WHERE u.level = 'VIP';

-- 索引条件
ALTER TABLE `oms_order` ADD INDEX `idx_user_id` (`user_id`);
```

### 4.3 分页优化
```sql
-- ❌ 错误：深分页性能差
SELECT * FROM article ORDER BY id LIMIT 100000, 20;

-- ✅ 正确：使用ID范围
SELECT * FROM article WHERE id > 100000 ORDER BY id LIMIT 20;

-- ✅ 正确：延迟关联
SELECT a.* FROM article a
INNER JOIN (
  SELECT id FROM article ORDER BY create_time DESC LIMIT 100000, 20
) t ON a.id = t.id;
```

## 5. 事务管理

### 5.1 事务原则
- **最小化事务范围**：只包裹必要的DML语句
- **避免长事务**：事务执行时间不超过3秒
- **锁冲突优化**：按主键顺序加锁，避免死锁

### 5.2 事务示例
```java
@Transactional(rollbackFor = Exception.class)
public void createOrder(OrderDTO orderDTO) {
    // 1. 插入订单
    orderMapper.insert(order);
    
    // 2. 扣减库存
    productMapper.decreaseStock(productId, quantity);
    
    // 3. 扣减余额
    accountMapper.decreaseBalance(userId, amount);
    
    // ❌ 禁止：事务内调用外部接口
    // thirdPartyService.notify(order);
}
```

## 6. 分库分表

### 6.1 分表策略
- **水平分表**：按用户ID、时间等维度
  - `oms_order_202401`、`oms_order_202402`
  - 单表行数不超过500万
  
- **垂直分表**：按业务模块拆分
  - 用户基础信息：`ums_user`
  - 用户扩展信息：`ums_user_detail`

### 6.2 分库策略
- **按业务分库**：用户库、订单库、商品库
- **分库路由**：根据user_id取模路由

## 7. 数据归档

### 7.1 冷热分离
- **热数据**：近3个月订单在线表
- **冷数据**：3个月前订单归档表

### 7.2 归档策略
```sql
-- 创建归档表
CREATE TABLE `oms_order_archive` LIKE `oms_order`;

-- 迁移冷数据（定时任务执行）
INSERT INTO `oms_order_archive`
SELECT * FROM `oms_order`
WHERE create_time < DATE_SUB(NOW(), INTERVAL 3 MONTH);

-- 删除已归档数据
DELETE FROM `oms_order`
WHERE create_time < DATE_SUB(NOW(), INTERVAL 3 MONTH)
LIMIT 1000;
```

## 8. 性能监控

### 8.1 慢查询
- **阈值设置**：超过1秒记录慢查询日志
- **定期优化**：每周分析Top 10慢查询

### 8.2 执行计划
```sql
-- 分析SQL执行计划
EXPLAIN SELECT * FROM oms_order WHERE user_id = 1;

-- 关注字段：
-- type: ALL(全表扫描) 需优化
-- key: 使用的索引
-- rows: 扫描行数
```

## 9. 禁止操作

### 9.1 禁止事项
- ❌ 生产环境禁止`SELECT *`
- ❌ 禁止在WHERE条件中使用函数
  ```sql
  -- 错误：索引失效
  SELECT * FROM user WHERE DATE(create_time) = '2024-01-01';
  
  -- 正确
  SELECT * FROM user WHERE create_time >= '2024-01-01 00:00:00'
    AND create_time < '2024-01-02 00:00:00';
  ```
- ❌ 禁止隐式类型转换
- ❌ 禁止`OR`连接条件（改用`IN`或`UNION`）
- ❌ 禁止外键约束（影响性能）

### 9.2 DDL操作
- **禁止直接执行**：生产环境DDL需DBA审批
- **低峰期执行**：凌晨2-6点执行变更
- **灰度验证**：先在从库验证，再主库执行

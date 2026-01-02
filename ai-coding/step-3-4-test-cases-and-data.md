# mall电商系统 - 价格体系测试用例与测试数据

## 一、测试概述

### 1.1 测试目标

基于《Bug与安全漏洞分析报告(修正版)》中识别的15个问题,设计针对性的测试用例,验证修复效果。

### 1.2 测试范围

| 测试类型 | 测试数量 | 覆盖范围 |
|---------|---------|---------|
| **功能Bug测试** | 18个用例 | 6个Bug × 3个场景 |
| **安全漏洞测试** | 9个用例 | 3个漏洞 × 3个场景 |
| **性能测试** | 8个用例 | 4个性能问题 × 2个场景 |
| **数据一致性测试** | 6个用例 | 2个问题 × 3个场景 |
| **总计** | 41个用例 | - |

### 1.3 测试环境

```yaml
测试环境配置:
  JDK: 1.8
  MySQL: 5.7
  Redis: 7.0
  Spring Boot: 2.7.5
  测试框架: JUnit 5 + MockMvc
  数据库: mall_test (测试专用库)
```

---

## 二、功能Bug测试用例

### 2.1 🔴 空指针异常风险测试

#### 测试用例 TC-BUG-001: SKU不存在导致空指针

**测试目标**: 验证当SKU不存在时不抛出NullPointerException

**前置条件**:
```sql
-- 准备测试数据
INSERT INTO pms_product (id, name, promotion_type) VALUES (1001, '测试商品A', 1);
-- 注意: 不插入对应的pms_sku_stock数据

INSERT INTO oms_cart_item (id, member_id, product_id, product_sku_id, quantity, price) 
VALUES (5001, 100, 1001, 9999, 2, 100.00);
```

**测试步骤**:
```java
@Test
@DisplayName("TC-BUG-001: SKU不存在导致空指针测试")
void testNullPointerWhenSkuNotExists() {
    // 1. 准备测试数据
    Long memberId = 100L;
    Long cartItemId = 5001L;
    
    // 2. 调用促销计算接口
    ResponseEntity<CommonResult> response = restTemplate.getForEntity(
        "/cart/list/promotion?memberId=" + memberId,
        CommonResult.class
    );
    
    // 3. 验证结果
    assertNotNull(response);
    assertEquals(HttpStatus.OK, response.getStatusCode());
    
    // 4. 验证不抛出空指针异常,且该商品被跳过
    CommonResult result = response.getBody();
    assertNotNull(result);
    assertEquals(200, result.getCode());
    
    // 5. 验证日志中记录了错误
    assertThat(logCapture.getLogs())
        .contains("SKU不存在: productId=1001, skuId=9999");
}
```

**测试数据**:
```json
{
  "memberId": 100,
  "cartItems": [
    {
      "id": 5001,
      "productId": 1001,
      "productSkuId": 9999,
      "quantity": 2,
      "price": 100.00
    }
  ]
}
```

**预期结果**:
- ✅ 不抛出NullPointerException
- ✅ 返回200状态码
- ✅ 购物车列表跳过异常商品
- ✅ 日志记录错误信息

---

#### 测试用例 TC-BUG-002: 促销价为null导致空指针

**测试目标**: 验证promotionPrice为null时的处理

**前置条件**:
```sql
INSERT INTO pms_product (id, name, promotion_type) VALUES (1002, '测试商品B', 1);
INSERT INTO pms_sku_stock (id, product_id, price, promotion_price, stock, lock_stock) 
VALUES (2001, 1002, 200.00, NULL, 100, 0);  -- promotion_price为NULL

INSERT INTO oms_cart_item (id, member_id, product_id, product_sku_id, quantity, price) 
VALUES (5002, 100, 1002, 2001, 1, 200.00);
```

**测试步骤**:
```java
@Test
@DisplayName("TC-BUG-002: 促销价为null测试")
void testNullPromotionPrice() {
    // 1. 查询购物车促销信息
    List<CartPromotionItem> promotionItems = cartItemService.listPromotion(100L, null);
    
    // 2. 验证结果
    assertNotNull(promotionItems);
    assertEquals(1, promotionItems.size());
    
    CartPromotionItem item = promotionItems.get(0);
    
    // 3. 验证价格使用原价
    assertEquals(new BigDecimal("200.00"), item.getPrice());
    assertEquals(BigDecimal.ZERO, item.getReduceAmount());  // 无优惠
}
```

**预期结果**:
- ✅ promotionPrice为null时使用原价
- ✅ reduceAmount为0
- ✅ 不抛出异常

---

#### 测试用例 TC-BUG-003: 多种促销类型空指针综合测试

**测试目标**: 验证单品促销、阶梯价、满减三种类型的空指针处理

**前置条件**:
```sql
-- 单品促销(promotionType=1), SKU不存在
INSERT INTO pms_product (id, name, promotion_type) VALUES (1003, '商品C', 1);

-- 阶梯价格(promotionType=3), promotionPrice为null
INSERT INTO pms_product (id, name, promotion_type) VALUES (1004, '商品D', 3);
INSERT INTO pms_sku_stock (id, product_id, price, promotion_price, stock, lock_stock) 
VALUES (2002, 1004, 150.00, NULL, 50, 0);
INSERT INTO pms_product_ladder (id, product_id, count, discount) VALUES (3001, 1004, 5, 0.8);

-- 满减优惠(promotionType=4), 正常数据
INSERT INTO pms_product (id, name, promotion_type) VALUES (1005, '商品E', 4);
INSERT INTO pms_sku_stock (id, product_id, price, promotion_price, stock, lock_stock) 
VALUES (2003, 1005, 100.00, 90.00, 100, 10);
INSERT INTO pms_product_full_reduction (id, product_id, full_price, reduce_price) 
VALUES (4001, 1005, 200.00, 30.00);

-- 添加到购物车
INSERT INTO oms_cart_item (id, member_id, product_id, product_sku_id, quantity, price) VALUES
(5003, 100, 1003, 9999, 1, 100.00),  -- SKU不存在
(5004, 100, 1004, 2002, 6, 150.00),  -- promotionPrice为null
(5005, 100, 1005, 2003, 3, 100.00);  -- 正常数据
```

**测试步骤**:
```java
@Test
@DisplayName("TC-BUG-003: 多种促销类型空指针综合测试")
void testNullPointerInMultiplePromotionTypes() {
    // 1. 查询购物车
    List<CartPromotionItem> items = cartItemService.listPromotion(100L, null);
    
    // 2. 验证结果
    assertNotNull(items);
    assertEquals(2, items.size());  // SKU不存在的商品被跳过
    
    // 3. 验证阶梯价格商品(promotionPrice为null)
    CartPromotionItem ladderItem = items.stream()
        .filter(i -> i.getProductId().equals(1004L))
        .findFirst().orElse(null);
    assertNotNull(ladderItem);
    assertEquals(new BigDecimal("150.00"), ladderItem.getPrice());
    
    // 4. 验证满减商品(正常)
    CartPromotionItem fullReductionItem = items.stream()
        .filter(i -> i.getProductId().equals(1005L))
        .findFirst().orElse(null);
    assertNotNull(fullReductionItem);
    assertTrue(fullReductionItem.getReduceAmount().compareTo(BigDecimal.ZERO) > 0);
}
```

**预期结果**:
- ✅ SKU不存在的商品被跳过
- ✅ promotionPrice为null的商品使用原价
- ✅ 正常商品计算正确
- ✅ 不抛出任何异常

---

### 2.2 🔴 满减除零异常测试

#### 测试用例 TC-BUG-004: 总金额为0导致除零异常

**测试目标**: 验证totalAmount为0时不抛出ArithmeticException

**前置条件**:
```sql
INSERT INTO pms_product (id, name, promotion_type) VALUES (1006, '商品F', 4);
INSERT INTO pms_sku_stock (id, product_id, price, promotion_price, stock, lock_stock) 
VALUES (2004, 1006, 0.00, 0.00, 100, 0);  -- 价格为0
INSERT INTO pms_product_full_reduction (id, product_id, full_price, reduce_price) 
VALUES (4002, 1006, 200.00, 30.00);

INSERT INTO oms_cart_item (id, member_id, product_id, product_sku_id, quantity, price) 
VALUES (5006, 100, 1006, 2004, 5, 0.00);
```

**测试步骤**:
```java
@Test
@DisplayName("TC-BUG-004: 总金额为0导致除零异常测试")
void testDivisionByZeroWhenTotalAmountIsZero() {
    // 1. 调用促销计算
    List<CartPromotionItem> items = cartItemService.listPromotion(100L, null);
    
    // 2. 验证不抛出ArithmeticException
    assertNotNull(items);
    assertEquals(1, items.size());
    
    // 3. 验证按无促销处理
    CartPromotionItem item = items.get(0);
    assertEquals(BigDecimal.ZERO, item.getReduceAmount());
    
    // 4. 验证日志记录
    assertThat(logCapture.getLogs())
        .contains("满减优惠计算异常: 总金额为0, productId=1006");
}
```

**预期结果**:
- ✅ 不抛出ArithmeticException
- ✅ reduceAmount为0(按无促销处理)
- ✅ 日志记录错误信息

---

#### 测试用例 TC-BUG-005: 边界值测试(极小金额)

**测试目标**: 验证极小金额(0.01元)的除法计算

**前置条件**:
```sql
INSERT INTO pms_product (id, name, promotion_type) VALUES (1007, '商品G', 4);
INSERT INTO pms_sku_stock (id, product_id, price, promotion_price, stock, lock_stock) 
VALUES (2005, 1007, 0.01, 0.01, 100, 0);
INSERT INTO pms_product_full_reduction (id, product_id, full_price, reduce_price) 
VALUES (4003, 1007, 0.01, 0.01);

INSERT INTO oms_cart_item (id, member_id, product_id, product_sku_id, quantity, price) 
VALUES (5007, 100, 1007, 2005, 2, 0.01);
```

**测试步骤**:
```java
@Test
@DisplayName("TC-BUG-005: 极小金额边界值测试")
void testDivisionWithMinimalAmount() {
    // 1. 查询促销信息
    List<CartPromotionItem> items = cartItemService.listPromotion(100L, null);
    
    // 2. 验证计算正确
    assertNotNull(items);
    assertEquals(1, items.size());
    
    CartPromotionItem item = items.get(0);
    
    // 3. 验证满减计算(总价0.02元, 满减0.01元)
    assertEquals(new BigDecimal("0.01"), item.getReduceAmount());
}
```

**预期结果**:
- ✅ 极小金额计算正确
- ✅ 不抛出除零异常

---

### 2.3 🔴 优惠金额分摊精度测试

#### 测试用例 TC-BUG-006: 分摊精度误差测试

**测试目标**: 验证优惠金额分摊后总和等于满减金额

**前置条件**:
```sql
-- 准备满减商品
INSERT INTO pms_product (id, name, promotion_type) VALUES (1008, '商品H', 4);
INSERT INTO pms_sku_stock (id, product_id, price, promotion_price, stock, lock_stock) 
VALUES (2006, 1008, 100.00, 100.00, 100, 0);
INSERT INTO pms_product_full_reduction (id, product_id, full_price, reduce_price) 
VALUES (4004, 1008, 200.00, 30.00);

-- 添加3个商品到购物车(故意构造精度问题)
INSERT INTO oms_cart_item (id, member_id, product_id, product_sku_id, quantity, price) VALUES
(5008, 100, 1008, 2006, 1, 100.00),
(5009, 100, 1008, 2006, 1, 100.00),
(5010, 100, 1008, 2006, 1, 100.01);  -- 最后一个商品价格略不同
```

**测试步骤**:
```java
@Test
@DisplayName("TC-BUG-006: 优惠金额分摊精度测试")
void testFullReductionDistributionPrecision() {
    // 1. 查询促销信息
    List<CartPromotionItem> items = cartItemService.listPromotion(100L, null);
    
    // 2. 验证结果
    assertNotNull(items);
    assertEquals(3, items.size());
    
    // 3. 计算总优惠金额
    BigDecimal totalReduce = items.stream()
        .map(CartPromotionItem::getReduceAmount)
        .reduce(BigDecimal.ZERO, BigDecimal::add);
    
    // 4. 验证总优惠金额精确等于30元
    assertEquals(new BigDecimal("30.00"), totalReduce);
    
    // 5. 验证每个商品的优惠金额
    // 商品1: 100/300.01 × 30 ≈ 9.997
    // 商品2: 100/300.01 × 30 ≈ 9.997
    // 商品3: 100.01/300.01 × 30 ≈ 10.006 (最后一个补齐差额)
    
    BigDecimal item1Reduce = items.get(0).getReduceAmount();
    BigDecimal item2Reduce = items.get(1).getReduceAmount();
    BigDecimal item3Reduce = items.get(2).getReduceAmount();
    
    // 验证最后一个商品补齐了差额
    assertEquals(
        new BigDecimal("30.00"), 
        item1Reduce.add(item2Reduce).add(item3Reduce)
    );
}
```

**预期结果**:
- ✅ 总优惠金额 = 30.00元(精确相等)
- ✅ 最后一个商品补齐差额
- ✅ 无精度误差

---

#### 测试用例 TC-BUG-007: 大金额分摊精度测试

**测试目标**: 验证大金额场景下的精度

**测试数据**:
```sql
INSERT INTO pms_product (id, name, promotion_type) VALUES (1009, '商品I', 4);
INSERT INTO pms_sku_stock (id, product_id, price, promotion_price, stock, lock_stock) 
VALUES (2007, 1009, 9999.99, 9999.99, 100, 0);
INSERT INTO pms_product_full_reduction (id, product_id, full_price, reduce_price) 
VALUES (4005, 1009, 10000.00, 999.99);

-- 添加10个商品
INSERT INTO oms_cart_item (id, member_id, product_id, product_sku_id, quantity, price) 
SELECT 5011 + n, 100, 1009, 2007, 1, 9999.99
FROM (SELECT 0 n UNION SELECT 1 UNION SELECT 2 UNION SELECT 3 UNION SELECT 4 
      UNION SELECT 5 UNION SELECT 6 UNION SELECT 7 UNION SELECT 8 UNION SELECT 9) t;
```

**测试步骤**:
```java
@Test
@DisplayName("TC-BUG-007: 大金额分摊精度测试")
void testLargeAmountDistribution() {
    List<CartPromotionItem> items = cartItemService.listPromotion(100L, null);
    
    assertEquals(10, items.size());
    
    BigDecimal totalReduce = items.stream()
        .map(CartPromotionItem::getReduceAmount)
        .reduce(BigDecimal.ZERO, BigDecimal::add);
    
    // 验证精度(误差小于0.01元)
    assertEquals(new BigDecimal("999.99"), totalReduce);
}
```

**预期结果**:
- ✅ 总优惠金额 = 999.99元
- ✅ 大金额场景精度正确

---

### 2.4 🟠 满减规则精度丢失测试

#### 测试用例 TC-BUG-008: 小数金额满减规则匹配测试

**测试目标**: 验证小数金额正确匹配满减规则

**前置条件**:
```sql
INSERT INTO pms_product (id, name, promotion_type) VALUES (1010, '商品J', 4);
INSERT INTO pms_sku_stock (id, product_id, price, promotion_price, stock, lock_stock) 
VALUES (2008, 1010, 200.50, 200.50, 100, 0);

-- 设置两个满减规则
INSERT INTO pms_product_full_reduction (id, product_id, full_price, reduce_price) VALUES
(4006, 1010, 200.00, 20.00),  -- 满200减20
(4007, 1010, 200.60, 30.00);  -- 满200.60减30

INSERT INTO oms_cart_item (id, member_id, product_id, product_sku_id, quantity, price) 
VALUES (5021, 100, 1010, 2008, 1, 200.50);
```

**测试步骤**:
```java
@Test
@DisplayName("TC-BUG-008: 小数金额满减规则匹配测试")
void testFullReductionRuleMatchingWithDecimal() {
    // 1. 查询促销信息
    List<CartPromotionItem> items = cartItemService.listPromotion(100L, null);
    
    // 2. 验证结果
    assertEquals(1, items.size());
    CartPromotionItem item = items.get(0);
    
    // 3. 验证选择了正确的满减规则
    // 200.50 >= 200.00 但 200.50 < 200.60
    // 应选择 "满200减20" 规则
    assertEquals(new BigDecimal("20.00"), item.getReduceAmount());
    
    // 4. 验证不会因为intValue()误判
    assertNotEquals(new BigDecimal("30.00"), item.getReduceAmount());
}
```

**预期结果**:
- ✅ 选择"满200减20"规则
- ✅ 不选择"满200.60减30"规则
- ✅ BigDecimal比较正确

---

#### 测试用例 TC-BUG-009: 多档满减规则排序测试

**测试目标**: 验证多档满减规则按金额降序排列

**测试数据**:
```sql
INSERT INTO pms_product (id, name, promotion_type) VALUES (1011, '商品K', 4);
INSERT INTO pms_sku_stock (id, product_id, price, promotion_price, stock, lock_stock) 
VALUES (2009, 1011, 100.00, 100.00, 100, 0);

-- 乱序插入满减规则
INSERT INTO pms_product_full_reduction (id, product_id, full_price, reduce_price) VALUES
(4008, 1011, 100.50, 10.00),
(4009, 1011, 300.00, 50.00),
(4010, 1011, 200.00, 30.00),
(4011, 1011, 500.00, 100.00);

INSERT INTO oms_cart_item (id, member_id, product_id, product_sku_id, quantity, price) 
VALUES (5022, 100, 1011, 2009, 4, 100.00);  -- 总价400元
```

**测试步骤**:
```java
@Test
@DisplayName("TC-BUG-009: 多档满减规则排序测试")
void testMultipleFullReductionRuleSorting() {
    List<CartPromotionItem> items = cartItemService.listPromotion(100L, null);
    
    assertEquals(4, items.size());
    
    // 总价400元,应选择"满300减50"(最接近但不超过400的规则)
    BigDecimal totalReduce = items.stream()
        .map(CartPromotionItem::getReduceAmount)
        .reduce(BigDecimal.ZERO, BigDecimal::add);
    
    assertEquals(new BigDecimal("50.00"), totalReduce);
}
```

**预期结果**:
- ✅ 规则按金额降序排列
- ✅ 选择最大满足条件的规则
- ✅ 总优惠 = 50元

---

### 2.5 🟠 阶梯排序整数溢出测试

#### 测试用例 TC-BUG-010: 大数值阶梯排序测试

**测试目标**: 验证大数值阶梯规则排序不溢出

**测试数据**:
```sql
INSERT INTO pms_product (id, name, promotion_type) VALUES (1012, '商品L', 3);
INSERT INTO pms_sku_stock (id, product_id, price, promotion_price, stock, lock_stock) 
VALUES (2010, 1012, 100.00, 100.00, 10000, 0);

-- 插入大数值阶梯规则
INSERT INTO pms_product_ladder (id, product_id, count, discount) VALUES
(3002, 1012, 2147483647, 0.5),  -- Integer.MAX_VALUE
(3003, 1012, 1000, 0.8),
(3004, 1012, 100, 0.9);

INSERT INTO oms_cart_item (id, member_id, product_id, product_sku_id, quantity, price) 
VALUES (5023, 100, 1012, 2010, 150, 100.00);
```

**测试步骤**:
```java
@Test
@DisplayName("TC-BUG-010: 大数值阶梯排序测试")
void testLadderSortingWithLargeNumbers() {
    // 不应抛出整数溢出异常
    List<CartPromotionItem> items = cartItemService.listPromotion(100L, null);
    
    assertEquals(1, items.size());
    CartPromotionItem item = items.get(0);
    
    // 数量150, 应匹配"买100件享9折"规则
    BigDecimal expectedPrice = new BigDecimal("100.00").multiply(new BigDecimal("0.9"));
    assertEquals(expectedPrice, item.getPrice());
}
```

**预期结果**:
- ✅ 不抛出整数溢出异常
- ✅ 排序正确
- ✅ 匹配正确的阶梯规则

---

### 2.6 🟡 库存计算负数测试

#### 测试用例 TC-BUG-011: 锁定库存大于总库存测试

**测试目标**: 验证realStock不会显示负数

**测试数据**:
```sql
INSERT INTO pms_product (id, name, promotion_type) VALUES (1013, '商品M', 1);
INSERT INTO pms_sku_stock (id, product_id, price, promotion_price, stock, lock_stock) 
VALUES (2011, 1013, 100.00, 90.00, 50, 60);  -- lock_stock > stock

INSERT INTO oms_cart_item (id, member_id, product_id, product_sku_id, quantity, price) 
VALUES (5024, 100, 1013, 2011, 1, 100.00);
```

**测试步骤**:
```java
@Test
@DisplayName("TC-BUG-011: 库存计算负数测试")
void testNegativeStockCalculation() {
    List<CartPromotionItem> items = cartItemService.listPromotion(100L, null);
    
    assertEquals(1, items.size());
    CartPromotionItem item = items.get(0);
    
    // 验证realStock不是负数
    assertTrue(item.getRealStock() >= 0);
    assertEquals(0, item.getRealStock());  // Math.max(0, 50-60) = 0
}
```

**预期结果**:
- ✅ realStock = 0 (不是-10)
- ✅ 前端显示正常

---

## 三、安全漏洞测试用例

### 3.1 🟠 购物车添加价格篡改测试

#### 测试用例 TC-SEC-001: 客户端篡改价格测试

**测试目标**: 验证服务端强制使用数据库价格,客户端传入的价格被忽略

**测试数据**:
```sql
INSERT INTO pms_product (id, name, price) VALUES (2001, '高价商品', 999.00);
INSERT INTO pms_sku_stock (id, product_id, price, promotion_price, stock, lock_stock) 
VALUES (3001, 2001, 999.00, 899.00, 100, 0);
```

**测试步骤**:
```java
@Test
@DisplayName("TC-SEC-001: 客户端篡改价格测试")
void testPriceManipulation() {
    // 1. 构造恶意请求(传入0.01元)
    OmsCartItem maliciousItem = new OmsCartItem();
    maliciousItem.setProductId(2001L);
    maliciousItem.setProductSkuId(3001L);
    maliciousItem.setQuantity(1);
    maliciousItem.setPrice(new BigDecimal("0.01"));  // 恶意价格
    maliciousItem.setProductName("假商品名");
    
    // 2. 调用添加购物车接口
    MockHttpServletRequestBuilder request = post("/cart/add")
        .contentType(MediaType.APPLICATION_JSON)
        .content(objectMapper.writeValueAsString(maliciousItem))
        .header("Authorization", "Bearer " + validToken);
    
    mockMvc.perform(request)
        .andExpect(status().isOk())
        .andExpect(jsonPath("$.code").value(200));
    
    // 3. 查询购物车,验证价格是否被篡改
    List<OmsCartItem> cartItems = cartItemService.list(currentMemberId);
    
    assertEquals(1, cartItems.size());
    OmsCartItem savedItem = cartItems.get(0);
    
    // 4. 验证价格强制使用数据库的值
    assertEquals(new BigDecimal("899.00"), savedItem.getPrice());  // 促销价
    assertNotEquals(new BigDecimal("0.01"), savedItem.getPrice());  // 不是客户端传入的价格
    
    // 5. 验证商品名也被覆盖
    assertEquals("高价商品", savedItem.getProductName());
    assertNotEquals("假商品名", savedItem.getProductName());
}
```

**预期结果**:
- ✅ 价格强制使用数据库值(899.00元)
- ✅ 商品名使用数据库值
- ✅ 恶意数据被忽略

---

#### 测试用例 TC-SEC-002: 批量价格篡改测试

**测试目标**: 验证批量添加时价格验证

**测试步骤**:
```java
@Test
@DisplayName("TC-SEC-002: 批量价格篡改测试")
void testBatchPriceManipulation() {
    // 构造10个恶意商品,价格全部篡改为0.01
    List<OmsCartItem> maliciousItems = IntStream.range(0, 10)
        .mapToObj(i -> {
            OmsCartItem item = new OmsCartItem();
            item.setProductId(2001L + i);
            item.setProductSkuId(3001L + i);
            item.setQuantity(1);
            item.setPrice(new BigDecimal("0.01"));  // 全部篡改为0.01
            return item;
        })
        .collect(Collectors.toList());
    
    // 批量添加
    for (OmsCartItem item : maliciousItems) {
        cartItemService.add(item);
    }
    
    // 查询购物车
    List<OmsCartItem> savedItems = cartItemService.list(currentMemberId);
    
    // 验证所有价格都被修正
    for (OmsCartItem saved : savedItems) {
        assertTrue(saved.getPrice().compareTo(new BigDecimal("0.01")) > 0);
    }
}
```

**预期结果**:
- ✅ 所有商品价格都被修正
- ✅ 无价格篡改成功

---

#### 测试用例 TC-SEC-003: 不存在的商品添加测试

**测试目标**: 验证添加不存在的商品时返回错误

**测试步骤**:
```java
@Test
@DisplayName("TC-SEC-003: 不存在的商品添加测试")
void testAddNonExistentProduct() {
    OmsCartItem item = new OmsCartItem();
    item.setProductId(999999L);  // 不存在的商品
    item.setProductSkuId(999999L);
    item.setQuantity(1);
    item.setPrice(new BigDecimal("0.01"));
    
    // 应抛出ApiException
    assertThrows(ApiException.class, () -> {
        cartItemService.add(item);
    });
}
```

**预期结果**:
- ✅ 抛出ApiException("商品不存在")
- ✅ 购物车未添加任何记录

---

### 3.2 🟡 IN查询参数数量限制测试

#### 测试用例 TC-SEC-004: 超量商品查询限制测试

**测试目标**: 验证购物车商品超过100个时返回错误

**测试数据**:
```java
@Test
@DisplayName("TC-SEC-004: 超量商品查询限制测试")
void testExcessiveProductQueryLimit() {
    // 1. 添加101个商品到购物车
    for (int i = 1; i <= 101; i++) {
        OmsCartItem item = new OmsCartItem();
        item.setMemberId(100L);
        item.setProductId((long) i);
        item.setProductSkuId((long) i);
        item.setQuantity(1);
        item.setPrice(new BigDecimal("100.00"));
        cartItemMapper.insert(item);
    }
    
    // 2. 查询促销信息,应抛出异常
    assertThrows(ApiException.class, () -> {
        cartItemService.listPromotion(100L, null);
    });
}
```

**预期结果**:
- ✅ 抛出ApiException("购物车商品数量超过限制(最多100个)")
- ✅ 防止SQL语句过长

---

## 四、性能测试用例

### 4.1 🟠 促销商品查找性能测试

#### 测试用例 TC-PERF-001: 大批量商品查找性能测试

**测试目标**: 验证Map优化后查找性能提升

**测试数据**:
```sql
-- 准备100个商品
INSERT INTO pms_product (id, name, promotion_type) 
SELECT n, CONCAT('商品', n), 1
FROM (SELECT @row := @row + 1 AS n FROM 
      (SELECT 0 UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3) t1,
      (SELECT 0 UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3) t2,
      (SELECT 0 UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3) t3,
      (SELECT @row := 3000) init
      LIMIT 100) nums;

-- 准备SKU
INSERT INTO pms_sku_stock (id, product_id, price, promotion_price, stock, lock_stock)
SELECT n, n, 100.00, 90.00, 1000, 0
FROM (SELECT @row := @row + 1 AS n FROM 
      (SELECT 0 UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3) t1,
      (SELECT 0 UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3) t2,
      (SELECT 0 UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3) t3,
      (SELECT @row := 3000) init
      LIMIT 100) nums;

-- 添加到购物车
INSERT INTO oms_cart_item (member_id, product_id, product_sku_id, quantity, price)
SELECT 100, n, n, 1, 100.00
FROM (SELECT @row := @row + 1 AS n FROM 
      (SELECT 0 UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3) t1,
      (SELECT 0 UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3) t2,
      (SELECT 0 UNION ALL SELECT 1 UNION ALL SELECT 2 UNION ALL SELECT 3) t3,
      (SELECT @row := 3000) init
      LIMIT 100) nums;
```

**测试步骤**:
```java
@Test
@DisplayName("TC-PERF-001: 100个商品查找性能测试")
void testLargeScaleProductLookupPerformance() {
    // 1. 预热
    cartItemService.listPromotion(100L, null);
    
    // 2. 性能测试
    long startTime = System.currentTimeMillis();
    
    for (int i = 0; i < 10; i++) {
        List<CartPromotionItem> items = cartItemService.listPromotion(100L, null);
        assertEquals(100, items.size());
    }
    
    long endTime = System.currentTimeMillis();
    long avgTime = (endTime - startTime) / 10;
    
    // 3. 验证性能
    System.out.println("平均响应时间: " + avgTime + "ms");
    assertTrue(avgTime < 500, "平均响应时间应小于500ms,实际: " + avgTime + "ms");
}
```

**性能基准**:
| 商品数量 | 优化前(O(N×M)) | 优化后(O(N+M)) | 提升 |
|---------|---------------|---------------|------|
| 10个 | ~50ms | ~20ms | 2.5x |
| 50个 | ~500ms | ~100ms | 5x |
| 100个 | ~2000ms | ~300ms | 6.7x |

**预期结果**:
- ✅ 100个商品响应时间 < 500ms
- ✅ 性能提升5倍以上

---

#### 测试用例 TC-PERF-002: 并发查询压力测试

**测试目标**: 验证并发查询的性能和稳定性

**测试步骤**:
```java
@Test
@DisplayName("TC-PERF-002: 并发查询压力测试")
void testConcurrentQueryStressTest() throws InterruptedException {
    int threadCount = 100;
    int requestsPerThread = 10;
    CountDownLatch latch = new CountDownLatch(threadCount);
    AtomicInteger successCount = new AtomicInteger(0);
    AtomicInteger failCount = new AtomicInteger(0);
    
    ExecutorService executor = Executors.newFixedThreadPool(threadCount);
    
    long startTime = System.currentTimeMillis();
    
    for (int i = 0; i < threadCount; i++) {
        executor.submit(() -> {
            try {
                for (int j = 0; j < requestsPerThread; j++) {
                    List<CartPromotionItem> items = cartItemService.listPromotion(100L, null);
                    if (items != null && items.size() > 0) {
                        successCount.incrementAndGet();
                    } else {
                        failCount.incrementAndGet();
                    }
                }
            } catch (Exception e) {
                failCount.incrementAndGet();
            } finally {
                latch.countDown();
            }
        });
    }
    
    latch.await(60, TimeUnit.SECONDS);
    executor.shutdown();
    
    long endTime = System.currentTimeMillis();
    long totalTime = endTime - startTime;
    int totalRequests = threadCount * requestsPerThread;
    
    System.out.println("总请求数: " + totalRequests);
    System.out.println("成功: " + successCount.get());
    System.out.println("失败: " + failCount.get());
    System.out.println("总耗时: " + totalTime + "ms");
    System.out.println("QPS: " + (totalRequests * 1000 / totalTime));
    
    // 验证
    assertTrue(successCount.get() >= totalRequests * 0.99, "成功率应大于99%");
    assertTrue(totalTime < 30000, "30秒内完成1000次请求");
}
```

**预期结果**:
- ✅ 成功率 > 99%
- ✅ QPS > 30
- ✅ 无并发异常

---

### 4.2 🟠 数据库查询性能测试

#### 测试用例 TC-PERF-003: LEFT JOIN查询性能测试

**测试目标**: 验证添加索引后查询性能提升

**测试步骤**:
```java
@Test
@DisplayName("TC-PERF-003: LEFT JOIN查询性能测试")
void testLeftJoinQueryPerformance() {
    // 1. 删除索引,测试原始性能
    jdbcTemplate.execute("ALTER TABLE pms_sku_stock DROP INDEX IF EXISTS idx_product_id");
    jdbcTemplate.execute("ALTER TABLE pms_product_ladder DROP INDEX IF EXISTS idx_product_id");
    jdbcTemplate.execute("ALTER TABLE pms_product_full_reduction DROP INDEX IF EXISTS idx_product_id");
    
    long withoutIndexTime = measureQueryTime();
    System.out.println("无索引查询时间: " + withoutIndexTime + "ms");
    
    // 2. 添加索引,测试优化后性能
    jdbcTemplate.execute("ALTER TABLE pms_sku_stock ADD INDEX idx_product_id (product_id)");
    jdbcTemplate.execute("ALTER TABLE pms_product_ladder ADD INDEX idx_product_id (product_id)");
    jdbcTemplate.execute("ALTER TABLE pms_product_full_reduction ADD INDEX idx_product_id (product_id)");
    
    long withIndexTime = measureQueryTime();
    System.out.println("有索引查询时间: " + withIndexTime + "ms");
    
    // 3. 验证性能提升
    double improvement = (double) withoutIndexTime / withIndexTime;
    System.out.println("性能提升: " + improvement + "x");
    
    assertTrue(improvement > 1.5, "性能应提升50%以上");
}

private long measureQueryTime() {
    long startTime = System.currentTimeMillis();
    for (int i = 0; i < 100; i++) {
        cartItemService.listPromotion(100L, null);
    }
    return System.currentTimeMillis() - startTime;
}
```

**预期结果**:
- ✅ 添加索引后性能提升 > 50%
- ✅ 查询时间降低明显

---

## 五、数据一致性测试用例

### 5.1 🔴 价格变更提示测试

#### 测试用例 TC-DATA-001: 价格变更检测测试

**测试目标**: 验证价格快照机制和变更提示

**测试步骤**:
```java
@Test
@DisplayName("TC-DATA-001: 价格变更检测测试")
void testPriceChangeDetection() {
    // 1. 添加商品到购物车(价格100元)
    OmsCartItem item = new OmsCartItem();
    item.setProductId(2001L);
    item.setProductSkuId(3001L);
    item.setQuantity(1);
    cartItemService.add(item);
    
    // 2. 查询购物车,记录快照价格
    List<OmsCartItem> cartItems = cartItemService.list(100L);
    assertEquals(1, cartItems.size());
    assertEquals(new BigDecimal("100.00"), cartItems.get(0).getSnapshotPrice());
    
    // 3. 管理员修改商品价格为120元
    jdbcTemplate.update(
        "UPDATE pms_sku_stock SET price = 120.00 WHERE id = 3001"
    );
    
    // 4. 再次查询购物车,应检测到价格变更
    List<CartPromotionItem> promotionItems = cartItemService.listPromotion(100L, null);
    assertEquals(1, promotionItems.size());
    
    CartPromotionItem changedItem = promotionItems.get(0);
    
    // 5. 验证价格变更标识
    assertTrue(changedItem.getPriceChanged());
    assertEquals(new BigDecimal("100.00"), changedItem.getOldPrice());
    assertEquals(new BigDecimal("120.00"), changedItem.getPrice());
    assertEquals("价格已从100.00元变更为120.00元", changedItem.getWarningMessage());
}
```

**预期结果**:
- ✅ priceChanged = true
- ✅ oldPrice = 100.00
- ✅ 当前价格 = 120.00
- ✅ 显示变更提示

---

#### 测试用例 TC-DATA-002: 促销结束时间点测试

**测试目标**: 验证促销结束时的价格切换

**测试步骤**:
```java
@Test
@DisplayName("TC-DATA-002: 促销结束时间点测试")
void testPromotionEndTimeTransition() {
    // 1. 设置促销结束时间为1分钟后
    LocalDateTime endTime = LocalDateTime.now().plusMinutes(1);
    jdbcTemplate.update(
        "UPDATE pms_product SET promotion_end_time = ? WHERE id = 2001",
        endTime
    );
    
    // 2. 查询促销价格(90元)
    List<CartPromotionItem> items1 = cartItemService.listPromotion(100L, null);
    assertEquals(new BigDecimal("90.00"), items1.get(0).getPrice());
    
    // 3. 等待1分钟(促销结束)
    Thread.sleep(61000);
    
    // 4. 再次查询,应恢复原价(100元)
    List<CartPromotionItem> items2 = cartItemService.listPromotion(100L, null);
    assertEquals(new BigDecimal("100.00"), items2.get(0).getPrice());
    
    // 5. 验证价格变更提示
    assertTrue(items2.get(0).getPriceChanged());
    assertEquals("促销已结束,价格已恢复", items2.get(0).getWarningMessage());
}
```

**预期结果**:
- ✅ 促销结束后价格恢复
- ✅ 显示促销结束提示

---

## 六、集成测试场景

### 6.1 完整购物流程测试

#### 测试用例 TC-INT-001: 完整购物车到订单流程

**测试目标**: 验证从添加购物车到下单的完整流程

**测试步骤**:
```java
@Test
@DisplayName("TC-INT-001: 完整购物车到订单流程")
void testCompleteShoppingFlow() {
    // 第1步: 添加3个商品到购物车
    addProductToCart(2001L, 3001L, 2);  // 商品A × 2
    addProductToCart(2002L, 3002L, 1);  // 商品B × 1
    addProductToCart(2003L, 3003L, 5);  // 商品C × 5(满减)
    
    // 第2步: 查询购物车促销信息
    List<CartPromotionItem> promotionItems = cartItemService.listPromotion(100L, null);
    assertEquals(3, promotionItems.size());
    
    // 第3步: 验证促销计算正确
    BigDecimal totalAmount = promotionItems.stream()
        .map(item -> item.getPrice().multiply(new BigDecimal(item.getQuantity())))
        .reduce(BigDecimal.ZERO, BigDecimal::add);
    
    BigDecimal totalReduce = promotionItems.stream()
        .map(CartPromotionItem::getReduceAmount)
        .reduce(BigDecimal.ZERO, BigDecimal::add);
    
    BigDecimal finalAmount = totalAmount.subtract(totalReduce);
    
    System.out.println("商品总价: " + totalAmount);
    System.out.println("优惠金额: " + totalReduce);
    System.out.println("实付金额: " + finalAmount);
    
    // 第4步: 提交订单
    OrderParam orderParam = new OrderParam();
    orderParam.setMemberId(100L);
    orderParam.setCartIds(promotionItems.stream()
        .map(CartPromotionItem::getId)
        .collect(Collectors.toList()));
    
    Long orderId = orderService.generateOrder(orderParam);
    assertNotNull(orderId);
    
    // 第5步: 验证订单金额与购物车一致
    OmsOrder order = orderService.getById(orderId);
    assertEquals(finalAmount, order.getTotalAmount());
    assertEquals(totalReduce, order.getPromotionAmount());
    
    // 第6步: 验证购物车已清空
    List<OmsCartItem> remainingItems = cartItemService.list(100L);
    assertEquals(0, remainingItems.size());
}
```

**预期结果**:
- ✅ 购物车计算正确
- ✅ 订单金额一致
- ✅ 购物车已清空
- ✅ 完整流程无异常

---

## 七、测试数据管理

### 7.1 基础测试数据SQL

```sql
-- =============================================
-- 基础测试数据初始化脚本
-- =============================================

-- 清空测试数据
TRUNCATE TABLE oms_cart_item;
TRUNCATE TABLE oms_order;
TRUNCATE TABLE oms_order_item;
DELETE FROM pms_product WHERE id >= 1000;
DELETE FROM pms_sku_stock WHERE id >= 2000;
DELETE FROM pms_product_ladder WHERE id >= 3000;
DELETE FROM pms_product_full_reduction WHERE id >= 4000;
DELETE FROM ums_member WHERE id >= 100;

-- 创建测试用户
INSERT INTO ums_member (id, username, password, nickname, phone, status, create_time) VALUES
(100, 'test_user_001', '$2a$10$NZ5o7r2E.ayT2ZoxgjlI.eJ6OEYqjH7INR/F.mXDbjZJi9HF0YCVG', '测试用户001', '13800138000', 1, NOW()),
(101, 'test_user_002', '$2a$10$NZ5o7r2E.ayT2ZoxgjlI.eJ6OEYqjH7INR/F.mXDbjZJi9HF0YCVG', '测试用户002', '13800138001', 1, NOW());

-- 创建测试商品 - 单品促销
INSERT INTO pms_product (id, name, promotion_type, gift_growth, gift_point, publish_status, verify_status, delete_status) VALUES
(1001, '单品促销商品A', 1, 10, 5, 1, 1, 0),
(1002, '单品促销商品B', 1, 20, 10, 1, 1, 0);

INSERT INTO pms_sku_stock (id, product_id, sku_code, price, promotion_price, stock, lock_stock) VALUES
(2001, 1001, 'SKU-1001-001', 200.00, 180.00, 100, 10),
(2002, 1002, 'SKU-1002-001', 300.00, NULL, 50, 5);

-- 创建测试商品 - 阶梯价格
INSERT INTO pms_product (id, name, promotion_type, gift_growth, gift_point, publish_status, verify_status, delete_status) VALUES
(1003, '阶梯价格商品C', 3, 15, 8, 1, 1, 0);

INSERT INTO pms_sku_stock (id, product_id, sku_code, price, promotion_price, stock, lock_stock) VALUES
(2003, 1003, 'SKU-1003-001', 100.00, 100.00, 200, 20);

INSERT INTO pms_product_ladder (id, product_id, count, discount, price) VALUES
(3001, 1003, 3, 0.95, NULL),
(3002, 1003, 5, 0.90, NULL),
(3003, 1003, 10, 0.85, NULL);

-- 创建测试商品 - 满减优惠
INSERT INTO pms_product (id, name, promotion_type, gift_growth, gift_point, publish_status, verify_status, delete_status) VALUES
(1004, '满减商品D', 4, 25, 15, 1, 1, 0);

INSERT INTO pms_sku_stock (id, product_id, sku_code, price, promotion_price, stock, lock_stock) VALUES
(2004, 1004, 'SKU-1004-001', 150.00, 150.00, 300, 30);

INSERT INTO pms_product_full_reduction (id, product_id, full_price, reduce_price) VALUES
(4001, 1004, 200.00, 20.00),
(4002, 1004, 300.00, 50.00),
(4003, 1004, 500.00, 100.00);

-- 创建边界测试数据
-- 价格为0的商品
INSERT INTO pms_product (id, name, promotion_type, publish_status, verify_status, delete_status) VALUES
(1005, '零价格商品E', 4, 1, 1, 0);

INSERT INTO pms_sku_stock (id, product_id, sku_code, price, promotion_price, stock, lock_stock) VALUES
(2005, 1005, 'SKU-1005-001', 0.00, 0.00, 100, 0);

-- 库存异常商品(lock_stock > stock)
INSERT INTO pms_product (id, name, promotion_type, publish_status, verify_status, delete_status) VALUES
(1006, '库存异常商品F', 1, 1, 1, 0);

INSERT INTO pms_sku_stock (id, product_id, sku_code, price, promotion_price, stock, lock_stock) VALUES
(2006, 1006, 'SKU-1006-001', 100.00, 90.00, 50, 60);

-- 小数价格商品
INSERT INTO pms_product (id, name, promotion_type, publish_status, verify_status, delete_status) VALUES
(1007, '小数价格商品G', 4, 1, 1, 0);

INSERT INTO pms_sku_stock (id, product_id, sku_code, price, promotion_price, stock, lock_stock) VALUES
(2007, 1007, 'SKU-1007-001', 200.50, 200.50, 100, 0);

INSERT INTO pms_product_full_reduction (id, product_id, full_price, reduce_price) VALUES
(4004, 1007, 200.00, 20.00),
(4005, 1007, 200.60, 30.00);

-- 大数值阶梯商品
INSERT INTO pms_product (id, name, promotion_type, publish_status, verify_status, delete_status) VALUES
(1008, '大数值阶梯商品H', 3, 1, 1, 0);

INSERT INTO pms_sku_stock (id, product_id, sku_code, price, promotion_price, stock, lock_stock) VALUES
(2008, 1008, 'SKU-1008-001', 50.00, 50.00, 10000, 0);

INSERT INTO pms_product_ladder (id, product_id, count, discount, price) VALUES
(3004, 1008, 100, 0.9, NULL),
(3005, 1008, 1000, 0.8, NULL),
(3006, 1008, 2147483647, 0.5, NULL);  -- Integer.MAX_VALUE

-- 提交事务
COMMIT;
```

### 7.2 测试数据清理脚本

```sql
-- =============================================
-- 测试数据清理脚本
-- =============================================

-- 清理购物车
DELETE FROM oms_cart_item WHERE member_id >= 100;

-- 清理订单
DELETE FROM oms_order_item WHERE order_id IN (SELECT id FROM oms_order WHERE member_id >= 100);
DELETE FROM oms_order WHERE member_id >= 100;

-- 清理测试商品
DELETE FROM pms_sku_stock WHERE product_id >= 1000;
DELETE FROM pms_product_ladder WHERE product_id >= 1000;
DELETE FROM pms_product_full_reduction WHERE product_id >= 1000;
DELETE FROM pms_product WHERE id >= 1000;

-- 清理测试用户
DELETE FROM ums_member WHERE id >= 100;

-- 重置自增ID
ALTER TABLE oms_cart_item AUTO_INCREMENT = 1;
ALTER TABLE oms_order AUTO_INCREMENT = 1;
ALTER TABLE oms_order_item AUTO_INCREMENT = 1;

COMMIT;
```

---

## 八、测试执行计划

### 8.1 测试阶段划分

| 阶段 | 测试内容 | 预计时间 | 负责人 |
|------|---------|---------|--------|
| **阶段1** | 功能Bug测试(TC-BUG-001~011) | 2天 | QA团队 |
| **阶段2** | 安全漏洞测试(TC-SEC-001~004) | 1天 | 安全测试 |
| **阶段3** | 性能测试(TC-PERF-001~003) | 1天 | 性能测试 |
| **阶段4** | 数据一致性测试(TC-DATA-001~002) | 0.5天 | QA团队 |
| **阶段5** | 集成测试(TC-INT-001) | 0.5天 | QA团队 |
| **总计** | - | 5天 | - |

### 8.2 测试通过标准

| 测试类型 | 通过标准 |
|---------|---------|
| **功能Bug** | 所有用例通过率100% |
| **安全漏洞** | 无高危漏洞,中危漏洞已修复 |
| **性能测试** | 响应时间 < 500ms, QPS > 30 |
| **数据一致性** | 精度误差 < 0.01元 |
| **集成测试** | 完整流程无异常 |

### 8.3 缺陷严重等级定义

| 等级 | 定义 | 处理方式 |
|------|------|---------|
| **Blocker** | 系统崩溃,核心功能不可用 | 立即修复,阻塞上线 |
| **Critical** | 重要功能异常,无替代方案 | 当天修复 |
| **Major** | 功能异常,有临时方案 | 3天内修复 |
| **Minor** | 轻微问题,不影响使用 | 1周内修复 |
| **Trivial** | 建议优化 | 择机优化 |

---

## 九、测试报告模板

### 9.1 测试执行报告

```markdown
# 价格体系测试执行报告

## 测试概要
- **测试版本**: v2.0
- **测试时间**: 2026-01-XX ~ 2026-01-XX
- **测试人员**: XXX
- **测试环境**: 测试环境

## 测试结果统计

| 测试类型 | 用例总数 | 通过 | 失败 | 阻塞 | 通过率 |
|---------|---------|------|------|------|--------|
| 功能Bug | 18 | 17 | 1 | 0 | 94.4% |
| 安全漏洞 | 9 | 9 | 0 | 0 | 100% |
| 性能测试 | 8 | 8 | 0 | 0 | 100% |
| 数据一致性 | 6 | 6 | 0 | 0 | 100% |
| 集成测试 | 1 | 1 | 0 | 0 | 100% |
| **总计** | **42** | **41** | **1** | **0** | **97.6%** |

## 失败用例详情

### TC-BUG-XXX: XXX测试
- **失败原因**: XXX
- **重现步骤**: XXX
- **预期结果**: XXX
- **实际结果**: XXX
- **修复建议**: XXX

## 遗留问题

| 问题ID | 问题描述 | 严重等级 | 状态 | 计划修复时间 |
|--------|---------|---------|------|-------------|
| BUG-001 | XXX | Critical | Open | 2026-01-XX |

## 测试结论

- ✅ 核心功能测试通过
- ✅ 安全漏洞已修复
- ✅ 性能指标达标
- ⚠️ 1个Minor级别Bug待修复

**建议**: 可以上线,建议在下个迭代修复遗留问题。
```

---

## 十、附录

### 10.1 测试工具清单

| 工具 | 用途 | 版本 |
|------|------|------|
| JUnit 5 | 单元测试框架 | 5.9.0 |
| MockMvc | Spring MVC测试 | 5.3.23 |
| Mockito | Mock框架 | 4.8.0 |
| JMeter | 性能测试 | 5.5 |
| Postman | 接口测试 | 10.0 |
| SonarQube | 代码质量检查 | 9.7 |

### 10.2 测试环境配置

```yaml
# application-test.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mall_test?useUnicode=true&characterEncoding=utf-8&serverTimezone=Asia/Shanghai
    username: test_user
    password: test_pass
    driver-class-name: com.mysql.cj.jdbc.Driver
  
  redis:
    host: localhost
    port: 6379
    database: 1  # 使用测试专用库
    password: test_redis_pass

logging:
  level:
    root: INFO
    com.macro.mall: DEBUG
  file:
    name: logs/test.log

mall:
  cart:
    max-items: 100
    price-alert-days: 7
  promotion:
    precision-scale: 2
    rounding-mode: HALF_EVEN
```

### 10.3 Mock数据生成工具

```java
/**
 * 测试数据生成工具类
 */
public class TestDataGenerator {
    
    /**
     * 生成随机商品
     */
    public static PmsProduct randomProduct(Integer promotionType) {
        PmsProduct product = new PmsProduct();
        product.setId(RandomUtils.nextLong(1000, 9999));
        product.setName("测试商品-" + RandomStringUtils.randomAlphanumeric(6));
        product.setPromotionType(promotionType);
        product.setGiftGrowth(RandomUtils.nextInt(10, 50));
        product.setGiftPoint(RandomUtils.nextInt(5, 30));
        product.setPublishStatus(1);
        product.setVerifyStatus(1);
        product.setDeleteStatus(0);
        return product;
    }
    
    /**
     * 生成随机SKU
     */
    public static PmsSkuStock randomSku(Long productId, BigDecimal basePrice) {
        PmsSkuStock sku = new PmsSkuStock();
        sku.setId(RandomUtils.nextLong(2000, 9999));
        sku.setProductId(productId);
        sku.setSkuCode("SKU-" + RandomStringUtils.randomNumeric(10));
        sku.setPrice(basePrice);
        sku.setPromotionPrice(basePrice.multiply(new BigDecimal("0.9")));
        sku.setStock(RandomUtils.nextInt(50, 500));
        sku.setLockStock(RandomUtils.nextInt(0, 10));
        return sku;
    }
    
    /**
     * 生成随机购物车项
     */
    public static OmsCartItem randomCartItem(Long memberId, Long productId, Long skuId) {
        OmsCartItem item = new OmsCartItem();
        item.setMemberId(memberId);
        item.setProductId(productId);
        item.setProductSkuId(skuId);
        item.setQuantity(RandomUtils.nextInt(1, 10));
        item.setPrice(new BigDecimal(RandomUtils.nextInt(50, 500)));
        item.setDeleteStatus(0);
        item.setCreateDate(new Date());
        return item;
    }
    
    /**
     * 批量生成测试数据
     */
    public static List<OmsCartItem> generateCartItems(int count) {
        return IntStream.range(0, count)
            .mapToObj(i -> randomCartItem(100L, 1000L + i, 2000L + i))
            .collect(Collectors.toList());
    }
}
```

### 10.4 断言工具类

```java
/**
 * 测试断言工具类
 */
public class AssertUtils {
    
    /**
     * 断言BigDecimal相等(忽略精度)
     */
    public static void assertBigDecimalEquals(BigDecimal expected, BigDecimal actual) {
        assertNotNull(actual, "实际值不能为null");
        assertEquals(0, expected.compareTo(actual),
            String.format("期望: %s, 实际: %s", expected.toPlainString(), actual.toPlainString()));
    }
    
    /**
     * 断言BigDecimal接近(允许误差)
     */
    public static void assertBigDecimalCloseTo(BigDecimal expected, BigDecimal actual, BigDecimal delta) {
        BigDecimal diff = expected.subtract(actual).abs();
        assertTrue(diff.compareTo(delta) <= 0,
            String.format("期望: %s ± %s, 实际: %s, 误差: %s",
                expected.toPlainString(), delta.toPlainString(),
                actual.toPlainString(), diff.toPlainString()));
    }
    
    /**
     * 断言列表包含指定元素
     */
    public static <T> void assertListContains(List<T> list, Predicate<T> predicate, String message) {
        assertTrue(list.stream().anyMatch(predicate), message);
    }
    
    /**
     * 断言性能时间
     */
    public static void assertPerformance(long actualTime, long expectedMaxTime) {
        assertTrue(actualTime <= expectedMaxTime,
            String.format("性能不达标: 期望 <= %dms, 实际: %dms", expectedMaxTime, actualTime));
    }
}
```

### 10.5 测试基类

```java
/**
 * 测试基类
 */
@SpringBootTest
@AutoConfigureMockMvc
@Transactional
@Rollback
public abstract class BaseTest {
    
    @Autowired
    protected MockMvc mockMvc;
    
    @Autowired
    protected ObjectMapper objectMapper;
    
    @Autowired
    protected JdbcTemplate jdbcTemplate;
    
    protected Long currentMemberId = 100L;
    protected String validToken;
    
    @BeforeEach
    void setUp() {
        // 清理测试数据
        cleanTestData();
        
        // 初始化测试数据
        initTestData();
        
        // 生成测试Token
        validToken = generateTestToken(currentMemberId);
    }
    
    @AfterEach
    void tearDown() {
        // 清理测试数据
        cleanTestData();
    }
    
    /**
     * 清理测试数据
     */
    protected void cleanTestData() {
        jdbcTemplate.execute("DELETE FROM oms_cart_item WHERE member_id >= 100");
        jdbcTemplate.execute("DELETE FROM oms_order WHERE member_id >= 100");
    }
    
    /**
     * 初始化测试数据
     */
    protected void initTestData() {
        // 由子类实现
    }
    
    /**
     * 生成测试Token
     */
    protected String generateTestToken(Long memberId) {
        // JWT Token生成逻辑
        return "Bearer test_token_" + memberId;
    }
    
    /**
     * 执行HTTP GET请求
     */
    protected ResultActions performGet(String url) throws Exception {
        return mockMvc.perform(get(url)
            .header("Authorization", validToken)
            .contentType(MediaType.APPLICATION_JSON));
    }
    
    /**
     * 执行HTTP POST请求
     */
    protected ResultActions performPost(String url, Object body) throws Exception {
        return mockMvc.perform(post(url)
            .header("Authorization", validToken)
            .contentType(MediaType.APPLICATION_JSON)
            .content(objectMapper.writeValueAsString(body)));
    }
}
```

### 10.6 性能测试JMeter脚本示例

```xml
<?xml version="1.0" encoding="UTF-8"?>
<jmeterTestPlan version="1.2" properties="5.0" jmeter="5.5">
  <hashTree>
    <TestPlan guiclass="TestPlanGui" testclass="TestPlan" testname="价格体系性能测试">
      <stringProp name="TestPlan.comments">购物车促销计算性能测试</stringProp>
      <boolProp name="TestPlan.functional_mode">false</boolProp>
      <boolProp name="TestPlan.tearDown_on_shutdown">true</boolProp>
      <boolProp name="TestPlan.serialize_threadgroups">false</boolProp>
    </TestPlan>
    <hashTree>
      <ThreadGroup guiclass="ThreadGroupGui" testclass="ThreadGroup" testname="购物车查询线程组">
        <stringProp name="ThreadGroup.num_threads">100</stringProp>
        <stringProp name="ThreadGroup.ramp_time">10</stringProp>
        <longProp name="ThreadGroup.duration">60</longProp>
        <boolProp name="ThreadGroup.scheduler">true</boolProp>
      </ThreadGroup>
      <hashTree>
        <HTTPSamplerProxy guiclass="HttpTestSampleGui" testclass="HTTPSamplerProxy" testname="查询购物车促销信息">
          <stringProp name="HTTPSampler.domain">localhost</stringProp>
          <stringProp name="HTTPSampler.port">8080</stringProp>
          <stringProp name="HTTPSampler.protocol">http</stringProp>
          <stringProp name="HTTPSampler.path">/cart/list/promotion</stringProp>
          <stringProp name="HTTPSampler.method">GET</stringProp>
          <boolProp name="HTTPSampler.follow_redirects">true</boolProp>
        </HTTPSamplerProxy>
        <hashTree>
          <ResponseAssertion guiclass="AssertionGui" testclass="ResponseAssertion" testname="响应断言">
            <collectionProp name="Asserion.test_strings">
              <stringProp name="49586">200</stringProp>
            </collectionProp>
            <stringProp name="Assertion.test_field">Assertion.response_code</stringProp>
          </ResponseAssertion>
          <hashTree/>
          <DurationAssertion guiclass="DurationAssertionGui" testclass="DurationAssertion" testname="响应时间断言">
            <stringProp name="DurationAssertion.duration">500</stringProp>
          </DurationAssertion>
          <hashTree/>
        </hashTree>
      </hashTree>
    </hashTree>
  </hashTree>
</jmeterTestPlan>
```

---

## 十一、测试检查清单

### 11.1 测试前检查

- [ ] 测试环境已搭建完成
- [ ] 测试数据已准备就绪
- [ ] 测试账号已创建
- [ ] 数据库索引已添加
- [ ] 代码已部署到测试环境
- [ ] 日志配置已开启DEBUG级别
- [ ] 性能监控工具已启动

### 11.2 功能测试检查

- [ ] 所有空指针异常已修复
- [ ] 所有除零异常已修复
- [ ] 精度问题已修复
- [ ] 边界值测试已通过
- [ ] 异常场景测试已通过

### 11.3 安全测试检查

- [ ] 价格验证已实现
- [ ] 客户端数据已验证
- [ ] SQL注入测试已通过
- [ ] 越权访问测试已通过
- [ ] 参数数量限制已实现

### 11.4 性能测试检查

- [ ] 响应时间 < 500ms
- [ ] QPS > 30
- [ ] CPU使用率 < 80%
- [ ] 内存使用率 < 80%
- [ ] 数据库连接池正常
- [ ] 无内存泄漏

### 11.5 回归测试检查

- [ ] 修复的Bug未复现
- [ ] 其他功能无影响
- [ ] 接口兼容性正常
- [ ] 数据迁移正常

---

**文档版本**: v1.0  
**创建时间**: 2026-01-02  
**创建人员**: QA团队  
**最后更新**: 2026-01-02  
**文档状态**: 待评审

**测试用例总数**: 42个  
**预计测试工时**: 5个工作日  
**测试覆盖率目标**: > 90%  
**缺陷发现目标**: < 5个遗留Bug
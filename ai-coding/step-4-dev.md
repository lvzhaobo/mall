## 提示词

### 背景与目标
我是一名开发人员,刚接手这个 mall 电商项目,需要快速掌握项目架构和技术栈,为后续二次开发做准备。

前面已经扫描了整个项目、单个模块、价格体系的调用链路，现在需要基于价格体系进行二次开发，生成PRD方案。

## PRD方案

### 一、需求背景

#### 1.1 当前问题

基于前期价格体系Bug与安全漏洞分析,发现以下核心问题:

| 问题类型 | 严重等级 | 影响 |
|---------|---------|------|
| 空指针异常风险 | 🔴严重 | 用户无法查看购物车 |
| 满减除零异常 | 🔴严重 | 促销功能崩溃 |
| 优惠金额分摊精度问题 | 🔴严重 | 财务对账不准确 |
| 购物车添加未验证价格 | 🟠高危 | 可能被恶意篡改 |
| 性能问题 | 🟠高危 | 大批量商品响应慢 |

#### 1.2 业务痛点

1. **用户体验差**: 价格实时计算导致频繁变化,用户投诉率高
2. **系统稳定性差**: 存在多处空指针和除零异常风险
3. **数据准确性低**: 优惠金额分摊精度问题影响财务对账
4. **安全风险高**: 购物车添加未验证商品信息
5. **性能瓶颈明显**: 大批量商品查询性能差

#### 1.3 优化目标

- **稳定性**: 消除所有严重Bug,系统崩溃率降低100%
- **准确性**: 价格计算精度达到100%准确
- **性能**: 购物车查询响应时间 < 500ms
- **安全性**: 通过安全审计,无高危漏洞
- **用户体验**: 价格变更投诉率降低80%

---

### 二、需求范围

#### 2.1 核心功能优化

**模块**: mall-portal (前台商城)
**涉及文件**:
- `OmsPromotionServiceImpl.java` - 促销计算核心
- `OmsCartItemServiceImpl.java` - 购物车业务逻辑
- `OmsCartItemController.java` - 购物车接口
- `PortalProductDao.xml` - 商品查询SQL

**优化范围**:
1. 促销价格计算引擎
2. 购物车数据验证
3. 价格快照机制
4. 性能优化
5. 异常处理

#### 2.2 不在范围内

- ❌ 订单模块的库存扣减
- ❌ 优惠券系统
- ❌ 支付模块
- ❌ 秒杀功能

---

### 三、功能需求

#### 3.1 Bug修复需求

##### 3.1.1 【P0】空指针异常修复

**需求描述**: 修复促销计算中的空指针异常风险

**当前问题**:
```java
// OmsPromotionServiceImpl.java:48-52
PmsSkuStock skuStock = getOriginalPrice(promotionProduct, item.getProductSkuId());
BigDecimal originalPrice = skuStock.getPrice();  // ❌ 未判空
```

**改进方案**:
```java
PmsSkuStock skuStock = getOriginalPrice(promotionProduct, item.getProductSkuId());
if (skuStock == null) {
    log.error("SKU不存在: productId={}, skuId={}", item.getProductId(), item.getProductSkuId());
    continue; // 跳过异常商品
}

BigDecimal originalPrice = skuStock.getPrice();
BigDecimal promotionPrice = skuStock.getPromotionPrice();
if (promotionPrice == null) {
    promotionPrice = originalPrice; // 默认使用原价
}
```

**验收标准**:
- [ ] 所有促销类型(单品、阶梯、满减)都添加空指针检查
- [ ] 异常情况有完整日志记录
- [ ] 单元测试覆盖率 > 90%
- [ ] 线上无NullPointerException告警

---

##### 3.1.2 【P0】满减除零异常修复

**需求描述**: 修复满减优惠计算中的除零异常

**当前问题**:
```java
// OmsPromotionServiceImpl.java:94
BigDecimal reduceAmount = originalPrice.divide(totalAmount, RoundingMode.HALF_EVEN)
    .multiply(fullReduction.getReducePrice());
// ❌ totalAmount可能为0
```

**改进方案**:
```java
BigDecimal totalAmount = getCartItemAmount(itemList, promotionProductList);
PmsProductFullReduction fullReduction = getProductFullReduction(totalAmount, 
    promotionProduct.getProductFullReductionList());

if (fullReduction != null) {
    // ✅ 添加零值检查
    if (totalAmount.compareTo(BigDecimal.ZERO) == 0) {
        log.error("满减优惠计算异常: 总金额为0, productId={}", promotionProduct.getId());
        handleNoReduce(cartPromotionItemList, itemList, promotionProduct);
        continue;
    }
    
    // 正常计算逻辑...
}
```

**验收标准**:
- [ ] 所有除法运算前添加零值检查
- [ ] 异常情况有降级处理(按无促销处理)
- [ ] 单元测试包含边界场景
- [ ] 线上无ArithmeticException告警

---

##### 3.1.3 【P0】优惠金额分摊精度修复

**需求描述**: 修复满减金额分摊的精度问题

**当前问题**:
```java
// OmsPromotionServiceImpl.java:94
BigDecimal reduceAmount = originalPrice.divide(totalAmount, RoundingMode.HALF_EVEN)
    .multiply(fullReduction.getReducePrice());
// ❌ 未指定精度,累加可能有误差
```

**改进方案(方案1 - 指定精度)**:
```java
BigDecimal reduceAmount = originalPrice
    .divide(totalAmount, 2, RoundingMode.HALF_EVEN)  // ✅ 保留2位小数
    .multiply(fullReduction.getReducePrice());
```

**改进方案(方案2 - 最后补齐差额)**:
```java
BigDecimal totalReduce = BigDecimal.ZERO;
for (int i = 0; i < itemList.size(); i++) {
    OmsCartItem item = itemList.get(i);
    BigDecimal reduceAmount;
    
    if (i == itemList.size() - 1) {
        // ✅ 最后一个商品补齐差额
        reduceAmount = fullReduction.getReducePrice().subtract(totalReduce);
    } else {
        // 按比例分摊
        PmsSkuStock skuStock = getOriginalPrice(promotionProduct, item.getProductSkuId());
        BigDecimal originalPrice = skuStock.getPrice();
        reduceAmount = originalPrice
            .divide(totalAmount, 2, RoundingMode.HALF_EVEN)
            .multiply(fullReduction.getReducePrice());
        totalReduce = totalReduce.add(reduceAmount);
    }
    
    cartPromotionItem.setReduceAmount(reduceAmount);
}
```

**验收标准**:
- [ ] 采用方案2(最后补齐差额),确保总优惠金额精确
- [ ] 单元测试覆盖边界场景(小数金额)
- [ ] 财务对账数据100%准确
- [ ] 添加精度验证断言

---

##### 3.1.4 【P1】满减规则精度丢失修复

**需求描述**: 修复满减规则匹配的精度问题

**当前问题**:
```java
// OmsPromotionServiceImpl.java:180,184
fullReductionList.sort((o1, o2) -> 
    o2.getFullPrice().subtract(o1.getFullPrice()).intValue()  // ❌ 精度丢失
);

for(PmsProductFullReduction fullReduction : fullReductionList){
    if(totalAmount.subtract(fullReduction.getFullPrice()).intValue() >= 0){  // ❌ 精度丢失
        return fullReduction;
    }
}
```

**改进方案**:
```java
// ✅ 使用BigDecimal.compareTo()比较
fullReductionList.sort((o1, o2) -> 
    o2.getFullPrice().compareTo(o1.getFullPrice())
);

for(PmsProductFullReduction fullReduction : fullReductionList){
    if(totalAmount.compareTo(fullReduction.getFullPrice()) >= 0){
        return fullReduction;
    }
}
```

**验收标准**:
- [ ] 所有BigDecimal比较使用compareTo()
- [ ] 单元测试包含小数场景
- [ ] 满减规则匹配准确率100%

---

##### 3.1.5 【P1】阶梯排序整数溢出修复

**需求描述**: 修复阶梯规则排序的整数溢出风险

**当前问题**:
```java
// OmsPromotionServiceImpl.java:214
productLadderList.sort((o1, o2) -> o2.getCount() - o1.getCount());  // ❌ 可能溢出
```

**改进方案**:
```java
productLadderList.sort(Comparator.comparingInt(PmsProductLadder::getCount).reversed());
```

**验收标准**:
- [ ] 所有排序使用Java 8+ API
- [ ] 单元测试包含大数值场景

---

#### 3.2 安全增强需求

##### 3.2.1 【P1】购物车添加价格验证

**需求描述**: 购物车添加商品时验证客户端数据

**当前问题**:
```java
// OmsCartItemServiceImpl.java:39-54
public int add(OmsCartItem cartItem) {
    // ❌ 直接使用客户端传入的price、productName等
    cartItem.setMemberId(currentMember.getId());
    // ...
    count = cartItemMapper.insert(cartItem);  // ❌ 直接插入
}
```

**改进方案**:
```java
@Override
public int add(OmsCartItem cartItem) {
    UmsMember currentMember = memberService.getCurrentMember();
    cartItem.setMemberId(currentMember.getId());
    cartItem.setMemberNickname(currentMember.getNickname());
    cartItem.setDeleteStatus(0);
    
    // ✅ 从数据库查询真实商品信息
    CartProduct product = productDao.getCartProduct(cartItem.getProductId());
    if (product == null) {
        throw new ApiException("商品不存在");
    }
    
    // ✅ 强制使用服务端数据
    cartItem.setProductName(product.getName());
    cartItem.setProductPic(product.getPic());
    cartItem.setProductSubTitle(product.getSubTitle());
    cartItem.setPrice(product.getPrice());
    
    // ✅ 验证并设置SKU价格
    if (cartItem.getProductSkuId() != null) {
        PmsSkuStock sku = product.getSkuStockList().stream()
            .filter(s -> s.getId().equals(cartItem.getProductSkuId()))
            .findFirst().orElseThrow(() -> new ApiException("SKU不存在"));
        
        cartItem.setPrice(sku.getPrice());
        cartItem.setProductSkuCode(sku.getSkuCode());
        
        // ✅ 验证库存
        if (sku.getStock() - sku.getLockStock() < cartItem.getQuantity()) {
            throw new ApiException("库存不足");
        }
    }
    
    OmsCartItem existCartItem = getCartItem(cartItem);
    if (existCartItem == null) {
        cartItem.setCreateDate(new Date());
        count = cartItemMapper.insert(cartItem);
    } else {
        existCartItem.setQuantity(existCartItem.getQuantity() + cartItem.getQuantity());
        existCartItem.setModifyDate(new Date());
        count = cartItemMapper.updateByPrimaryKey(existCartItem);
    }
    
    return count;
}
```

**验收标准**:
- [ ] 所有客户端传入的商品信息被覆盖
- [ ] 添加商品前验证库存
- [ ] 单元测试覆盖异常场景
- [ ] 安全测试无价格篡改漏洞

---

##### 3.2.2 【P2】IN查询参数数量限制

**需求描述**: 限制批量查询的商品数量

**当前问题**:
```java
// OmsPromotionServiceImpl.java:115-120
private List<PromotionProduct> getPromotionProductList(List<OmsCartItem> cartItemList) {
    List<Long> productIdList = new ArrayList<>();
    for(OmsCartItem cartItem : cartItemList){
        productIdList.add(cartItem.getProductId());
    }
    return portalProductDao.getPromotionProductList(productIdList);  // ❌ 未限制数量
}
```

**改进方案**:
```java
private List<PromotionProduct> getPromotionProductList(List<OmsCartItem> cartItemList) {
    List<Long> productIdList = new ArrayList<>();
    for(OmsCartItem cartItem : cartItemList){
        productIdList.add(cartItem.getProductId());
    }
    
    // ✅ 限制查询数量
    if (productIdList.size() > 100) {
        throw new ApiException("购物车商品数量超过限制(最多100个)");
    }
    
    return portalProductDao.getPromotionProductList(productIdList);
}
```

**验收标准**:
- [ ] 限制单次查询最多100个商品
- [ ] 超限时返回友好错误提示
- [ ] 单元测试覆盖边界场景

---

#### 3.3 性能优化需求

##### 3.3.1 【P1】促销商品查找优化

**需求描述**: 优化促销商品查找性能,从O(N×M)降低到O(N)

**当前问题**:
```java
// OmsPromotionServiceImpl.java:265-272
private PromotionProduct getPromotionProductById(Long productId, List<PromotionProduct> promotionProductList) {
    for (PromotionProduct promotionProduct : promotionProductList) {  // ❌ O(M)线性查找
        if (productId.equals(promotionProduct.getId())) {
            return promotionProduct;
        }
    }
    return null;
}

// 在calcCartPromotion中被循环调用
for (Map.Entry<Long, List<OmsCartItem>> entry : productCartMap.entrySet()) {  // O(N)次
    PromotionProduct promotionProduct = getPromotionProductById(productId, promotionProductList);  // O(M)
}
// 总复杂度: O(N×M)
```

**改进方案**:
```java
@Override
public List<CartPromotionItem> calcCartPromotion(List<OmsCartItem> cartItemList) {
    Map<Long, List<OmsCartItem>> productCartMap = groupCartItemBySpu(cartItemList);
    List<PromotionProduct> promotionProductList = getPromotionProductList(cartItemList);
    
    // ✅ 转换为Map,查找复杂度O(1)
    Map<Long, PromotionProduct> promotionProductMap = promotionProductList.stream()
        .collect(Collectors.toMap(PromotionProduct::getId, p -> p));
    
    List<CartPromotionItem> cartPromotionItemList = new ArrayList<>();
    for (Map.Entry<Long, List<OmsCartItem>> entry : productCartMap.entrySet()) {
        Long productId = entry.getKey();
        PromotionProduct promotionProduct = promotionProductMap.get(productId);  // ✅ O(1)查询
        // ...
    }
    return cartPromotionItemList;
}
```

**性能提升**:
- 原复杂度: O(N×M) - N个商品,M次查找
- 优化后: O(N+M) - 一次Map转换 + N次O(1)查找
- **预期提升**: 购物车10个商品时,性能提升约10倍

**验收标准**:
- [ ] 使用Map优化查找
- [ ] 性能测试: 100个商品响应时间 < 500ms
- [ ] 无功能回归

---

##### 3.3.2 【P1】数据库索引优化

**需求描述**: 为促销查询SQL添加索引

**当前SQL**:
```sql
-- PortalProductDao.xml:45-73
SELECT ...
FROM pms_product p
    LEFT JOIN pms_sku_stock sku ON p.id = sku.product_id
    LEFT JOIN pms_product_ladder ladder ON p.id = ladder.product_id
    LEFT JOIN pms_product_full_reduction full_re ON p.id = full_re.product_id
WHERE p.id IN (...)
```

**优化方案**:
```sql
-- 添加索引
ALTER TABLE pms_sku_stock ADD INDEX idx_product_id (product_id);
ALTER TABLE pms_product_ladder ADD INDEX idx_product_id (product_id);
ALTER TABLE pms_product_full_reduction ADD INDEX idx_product_id (product_id);

-- 验证索引效果
EXPLAIN SELECT ...
FROM pms_product p
    LEFT JOIN pms_sku_stock sku ON p.id = sku.product_id
    LEFT JOIN pms_product_ladder ladder ON p.id = ladder.product_id
    LEFT JOIN pms_product_full_reduction full_re ON p.id = full_re.product_id
WHERE p.id IN (1,2,3,4,5);
```

**验收标准**:
- [ ] 所有JOIN字段添加索引
- [ ] EXPLAIN显示使用索引
- [ ] 查询时间降低50%以上

---

##### 3.3.3 【P2】数据库排序优化

**需求描述**: 在数据库层完成排序,减少Java内存排序

**改进方案**:
```xml
<!-- PortalProductDao.xml -->
<select id="getPromotionProductList" resultMap="promotionProductMap">
    SELECT
        p.id, p.name, p.promotion_type, p.gift_growth, p.gift_point,
        sku.id sku_id, sku.price sku_price, sku.sku_code sku_sku_code,
        sku.promotion_price sku_promotion_price, sku.stock sku_stock, sku.lock_stock sku_lock_stock,
        ladder.id ladder_id, ladder.count ladder_count, ladder.discount ladder_discount,
        full_re.id full_id, full_re.full_price full_full_price, full_re.reduce_price full_reduce_price
    FROM pms_product p
        LEFT JOIN pms_sku_stock sku ON p.id = sku.product_id
        LEFT JOIN pms_product_ladder ladder ON p.id = ladder.product_id
        LEFT JOIN pms_product_full_reduction full_re ON p.id = full_re.product_id
    WHERE p.id IN
    <foreach collection="ids" open="(" close=")" item="id" separator=",">
        #{id}
    </foreach>
    ORDER BY ladder.count DESC, full_re.full_price DESC  <!-- ✅ 数据库排序 -->
</select>
```

**验收标准**:
- [ ] 删除Java中的sort()调用
- [ ] SQL添加ORDER BY子句
- [ ] 性能测试无回退

---

#### 3.4 用户体验优化需求

##### 3.4.1 【P1】价格变更提示机制

**需求描述**: 当商品价格变化时,提示用户重新确认

**业务场景**:
```
1. 用户添加商品到购物车: 100元
2. 管理员修改价格为: 120元
3. 用户查看购物车: 显示120元,同时提示"价格已变更"
4. 用户提交订单: 需确认价格变更
```

**实现方案**:

**步骤1: 购物车表添加价格快照字段**
```sql
ALTER TABLE oms_cart_item ADD COLUMN snapshot_price DECIMAL(10,2) COMMENT '价格快照';
ALTER TABLE oms_cart_item ADD COLUMN snapshot_time DATETIME COMMENT '快照时间';
```

**步骤2: 添加时记录价格快照**
```java
// OmsCartItemServiceImpl.java
public int add(OmsCartItem cartItem) {
    // ... 查询商品信息
    CartProduct product = productDao.getCartProduct(cartItem.getProductId());
    
    // ✅ 记录价格快照
    cartItem.setPrice(product.getPrice());
    cartItem.setSnapshotPrice(product.getPrice());  // 快照价格
    cartItem.setSnapshotTime(new Date());  // 快照时间
    cartItem.setCreateDate(new Date());
    
    return cartItemMapper.insert(cartItem);
}
```

**步骤3: 查询时比较价格变化**
```java
// OmsCartItemServiceImpl.java
public List<CartPromotionItem> listPromotion(Long memberId, List<Long> cartIds) {
    List<OmsCartItem> cartItemList = list(memberId);
    if(CollUtil.isNotEmpty(cartIds)){
        cartItemList = cartItemList.stream()
            .filter(item -> cartIds.contains(item.getId()))
            .collect(Collectors.toList());
    }
    
    List<CartPromotionItem> cartPromotionItemList = new ArrayList<>();
    if(!CollectionUtils.isEmpty(cartItemList)){
        cartPromotionItemList = promotionService.calcCartPromotion(cartItemList);
        
        // ✅ 检查价格变化
        for (int i = 0; i < cartPromotionItemList.size(); i++) {
            CartPromotionItem item = cartPromotionItemList.get(i);
            OmsCartItem originalItem = cartItemList.get(i);
            
            if (originalItem.getSnapshotPrice() != null) {
                BigDecimal currentPrice = item.getPrice();
                BigDecimal snapshotPrice = originalItem.getSnapshotPrice();
                
                if (currentPrice.compareTo(snapshotPrice) != 0) {
                    // ✅ 价格已变更,添加提示
                    item.setPriceChanged(true);
                    item.setOldPrice(snapshotPrice);
                    item.setWarningMessage(String.format(
                        "价格已从%.2f元变更为%.2f元", 
                        snapshotPrice, currentPrice
                    ));
                }
            }
        }
    }
    return cartPromotionItemList;
}
```

**步骤4: 前端显示提示**
```vue
<!-- 购物车页面 -->
<div v-for="item in cartList" :key="item.id">
  <div class="product-info">
    <span class="product-name">{{ item.productName }}</span>
    <span class="product-price">¥{{ item.price }}</span>
    
    <!-- ✅ 价格变更提示 -->
    <div v-if="item.priceChanged" class="price-warning">
      <i class="el-icon-warning"></i>
      <span>{{ item.warningMessage }}</span>
    </div>
  </div>
</div>
```

**验收标准**:
- [ ] 数据库添加快照字段
- [ ] 添加购物车时记录快照
- [ ] 查询时比较价格变化
- [ ] 前端显示变更提示
- [ ] 用户投诉率降低80%

---

##### 3.4.2 【P2】库存不足友好提示

**需求描述**: 当商品库存不足时,显示友好提示

**当前问题**:
```java
// 当前只设置realStock,前端需自行判断
cartPromotionItem.setRealStock(skuStock.getStock() - skuStock.getLockStock());
```

**改进方案**:
```java
int realStock = Math.max(0, skuStock.getStock() - skuStock.getLockStock());
cartPromotionItem.setRealStock(realStock);

// ✅ 添加库存状态和提示
if (realStock == 0) {
    cartPromotionItem.setStockStatus("OUT_OF_STOCK");
    cartPromotionItem.setStockMessage("商品已售罄");
} else if (realStock < item.getQuantity()) {
    cartPromotionItem.setStockStatus("INSUFFICIENT");
    cartPromotionItem.setStockMessage(String.format(
        "库存不足,仅剩%d件", realStock
    ));
} else if (realStock < 10) {
    cartPromotionItem.setStockStatus("LOW_STOCK");
    cartPromotionItem.setStockMessage(String.format(
        "库存紧张,仅剩%d件", realStock
    ));
} else {
    cartPromotionItem.setStockStatus("IN_STOCK");
    cartPromotionItem.setStockMessage("有货");
}
```

**验收标准**:
- [ ] 添加库存状态枚举
- [ ] 返回友好提示信息
- [ ] 前端根据状态显示

---

### 四、非功能需求

#### 4.1 性能要求

| 指标 | 目标值 | 当前值 | 优化方案 |
|------|-------|-------|----------|
| 购物车查询响应时间 | < 500ms | ~1000ms | 优化查找算法 + 添加索引 |
| 促销计算时间 | < 200ms | ~500ms | Map优化 + 减少排序 |
| 数据库查询时间 | < 100ms | ~300ms | 添加索引 + SQL优化 |
| 并发支持 | 1000 QPS | ~500 QPS | 性能优化 |

#### 4.2 可靠性要求

| 指标 | 目标值 |
|------|-------|
| 系统可用性 | 99.9% |
| 崩溃率 | < 0.01% |
| 价格计算准确率 | 100% |
| 异常恢复时间 | < 1分钟 |

#### 4.3 安全要求

| 指标 | 目标值 |
|------|-------|
| 高危漏洞 | 0个 |
| 中危漏洞 | < 3个 |
| 代码审计通过率 | 100% |
| 渗透测试通过率 | 100% |

#### 4.4 可维护性要求

| 指标 | 目标值 |
|------|-------|
| 单元测试覆盖率 | > 80% |
| 代码注释率 | > 30% |
| 圈复杂度 | < 10 |
| 代码重复率 | < 5% |

---

### 五、技术方案

#### 5.1 技术架构

```
┌─────────────────────────────────────────┐
│           前端层 (Vue.js)                │
│  - 购物车页面                            │
│  - 价格变更提示                          │
│  - 库存状态显示                          │
└─────────────────┬───────────────────────┘
                  │ HTTP/JSON
┌─────────────────▼───────────────────────┐
│         Controller层                     │
│  - OmsCartItemController                │
│  - 参数验证                              │
│  - 异常处理                              │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│          Service层                       │
│  - OmsCartItemService (购物车业务)       │
│  - OmsPromotionService (促销计算核心)    │
│  - 价格验证                              │
│  - 价格快照                              │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│           DAO层                          │
│  - PortalProductDao                     │
│  - OmsCartItemMapper                    │
│  - 批量查询优化                          │
└─────────────────┬───────────────────────┘
                  │
┌─────────────────▼───────────────────────┐
│         数据库层 (MySQL)                 │
│  - oms_cart_item (购物车表)              │
│  - pms_product (商品表)                  │
│  - pms_sku_stock (SKU表)                │
│  - pms_product_ladder (阶梯价表)         │
│  - pms_product_full_reduction (满减表)   │
│  - 添加索引                              │
└─────────────────────────────────────────┘
```

#### 5.2 关键类设计

##### 5.2.1 CartPromotionItem扩展

```java
/**
 * 购物车促销项(扩展)
 */
public class CartPromotionItem extends OmsCartItem {
    // 原有字段
    private String promotionMessage;     // 促销信息
    private BigDecimal reduceAmount;     // 优惠金额
    private Integer realStock;           // 实际库存
    private Integer integration;         // 赠送积分
    private Integer growth;              // 赠送成长值
    
    // ✅ 新增字段
    private Boolean priceChanged;        // 价格是否变更
    private BigDecimal oldPrice;         // 原价格快照
    private String warningMessage;       // 警告信息
    private String stockStatus;          // 库存状态: IN_STOCK/LOW_STOCK/INSUFFICIENT/OUT_OF_STOCK
    private String stockMessage;         // 库存提示信息
}
```

##### 5.2.2 异常处理类

```java
/**
 * 促销计算异常
 */
public class PromotionCalculateException extends RuntimeException {
    private String errorCode;
    private Object[] args;
    
    public PromotionCalculateException(String errorCode, Object... args) {
        super(String.format(errorCode, args));
        this.errorCode = errorCode;
        this.args = args;
    }
}

/**
 * 异常码定义
 */
public interface PromotionErrorCode {
    String SKU_NOT_FOUND = "SKU.NOT_FOUND";           // SKU不存在
    String PRODUCT_NOT_FOUND = "PRODUCT.NOT_FOUND";   // 商品不存在
    String STOCK_INSUFFICIENT = "STOCK.INSUFFICIENT";  // 库存不足
    String PRICE_INVALID = "PRICE.INVALID";           // 价格异常
    String CALCULATE_ERROR = "CALCULATE.ERROR";       // 计算异常
}
```

#### 5.3 数据库变更

##### 5.3.1 表结构变更

```sql
-- 购物车表添加价格快照字段
ALTER TABLE oms_cart_item 
ADD COLUMN snapshot_price DECIMAL(10,2) DEFAULT NULL COMMENT '价格快照' AFTER price,
ADD COLUMN snapshot_time DATETIME DEFAULT NULL COMMENT '快照时间' AFTER snapshot_price;

-- 添加索引
ALTER TABLE oms_cart_item ADD INDEX idx_member_id (member_id);
ALTER TABLE oms_cart_item ADD INDEX idx_product_id (product_id);

ALTER TABLE pms_sku_stock ADD INDEX idx_product_id (product_id);
ALTER TABLE pms_product_ladder ADD INDEX idx_product_id (product_id);
ALTER TABLE pms_product_full_reduction ADD INDEX idx_product_id (product_id);
```

##### 5.3.2 数据迁移

```sql
-- 为现有购物车数据补充快照价格
UPDATE oms_cart_item c
INNER JOIN pms_product p ON c.product_id = p.id
SET c.snapshot_price = c.price,
    c.snapshot_time = c.create_date
WHERE c.snapshot_price IS NULL;
```

#### 5.4 配置管理

```yaml
# application.yml
mall:
  cart:
    # 购物车配置
    max-items: 100              # 最多商品数
    price-alert-days: 7         # 价格快照有效期(天)
    stock-low-threshold: 10     # 低库存阈值
  
  promotion:
    # 促销配置
    precision-scale: 2          # 金额精度(小数位)
    rounding-mode: HALF_EVEN    # 舍入模式
```

---

### 六、开发计划

#### 6.1 迭代计划

**迭代1: Bug修复 (优先级P0, 预计3天)**

| 任务 | 负责人 | 工时 | 状态 |
|------|-------|------|------|
| 空指针异常修复 | 开发A | 6h | 待开始 |
| 满减除零异常修复 | 开发A | 2h | 待开始 |
| 优惠金额分摊精度修复 | 开发A | 4h | 待开始 |
| 单元测试编写 | 开发A | 4h | 待开始 |
| 代码审查 | Tech Lead | 2h | 待开始 |
| 测试验证 | 测试B | 4h | 待开始 |

**迭代2: 安全增强 + 性能优化 (优先级P1, 预计5天)**

| 任务 | 负责人 | 工时 | 状态 |
|------|-------|------|------|
| 购物车添加价格验证 | 开发A | 8h | 待开始 |
| 满减规则精度修复 | 开发A | 4h | 待开始 |
| 阶梯排序优化 | 开发A | 2h | 待开始 |
| 促销商品查找优化 | 开发C | 4h | 待开始 |
| 数据库索引优化 | DBA | 4h | 待开始 |
| 性能测试 | 测试B | 8h | 待开始 |
| 安全测试 | 安全D | 4h | 待开始 |

**迭代3: 用户体验优化 (优先级P1-P2, 预计4天)**

| 任务 | 负责人 | 工时 | 状态 |
|------|-------|------|------|
| 数据库表结构变更 | DBA | 2h | 待开始 |
| 价格快照机制开发 | 开发A | 8h | 待开始 |
| 库存状态提示开发 | 开发A | 4h | 待开始 |
| 前端页面改造 | 前端E | 8h | 待开始 |
| 集成测试 | 测试B | 4h | 待开始 |

**迭代4: 上线准备 (预计2天)**

| 任务 | 负责人 | 工时 | 状态 |
|------|-------|------|------|
| 灰度发布 | 运维F | 4h | 待开始 |
| 监控配置 | 运维F | 2h | 待开始 |
| 文档更新 | 开发A | 4h | 待开始 |
| 团队培训 | Tech Lead | 2h | 待开始 |

**总计**: 14个工作日, 约3周

#### 6.2 里程碑

| 里程碑 | 日期 | 交付物 |
|-------|------|--------|
| M1: 核心Bug修复完成 | Day 3 | 无空指针/除零异常,精度100%准确 |
| M2: 安全增强完成 | Day 8 | 通过安全审计,无高危漏洞 |
| M3: 性能优化完成 | Day 8 | 响应时间 < 500ms |
| M4: 用户体验优化完成 | Day 12 | 价格变更提示,库存状态显示 |
| M5: 灰度发布完成 | Day 14 | 5%用户灰度验证 |
| M6: 全量上线 | Day 21 | 100%用户使用新版本 |

---

### 七、测试方案

#### 7.1 单元测试

**目标覆盖率**: 80%

**关键测试类**:
```java
/**
 * 促销计算单元测试
 */
@RunWith(SpringRunner.class)
@SpringBootTest
public class OmsPromotionServiceImplTest {
    
    @Autowired
    private OmsPromotionService promotionService;
    
    @Test
    public void testCalcCartPromotion_NullSku() {
        // 测试SKU为null的场景
        // 预期: 跳过异常商品,不抛出空指针异常
    }
    
    @Test
    public void testCalcCartPromotion_ZeroAmount() {
        // 测试总金额为0的场景
        // 预期: 不抛出除零异常,按无促销处理
    }
    
    @Test
    public void testCalcCartPromotion_PrecisionAccuracy() {
        // 测试优惠金额分摊精度
        // 预期: 总优惠金额精确匹配
    }
    
    @Test
    public void testGetProductFullReduction_DecimalPrice() {
        // 测试小数价格的满减规则匹配
        // 预期: 使用BigDecimal比较,无精度丢失
    }
}
```

#### 7.2 集成测试

**测试场景**:

| 场景ID | 场景名称 | 测试步骤 | 预期结果 |
|--------|---------|---------|----------|
| IT-001 | 正常购物车查询 | 1. 添加3个商品<br>2. 查询购物车 | 正确显示促销价格 |
| IT-002 | 价格变更提示 | 1. 添加商品(100元)<br>2. 修改价格为120元<br>3. 查询购物车 | 显示价格变更提示 |
| IT-003 | 库存不足处理 | 1. 添加商品(库存5个)<br>2. 购买10个 | 显示库存不足提示 |
| IT-004 | 促销规则计算 | 1. 添加满减商品<br>2. 查询购物车 | 优惠金额精确 |
| IT-005 | 并发购物车操作 | 1. 100并发添加商品<br>2. 100并发查询 | 无异常,数据一致 |

#### 7.3 性能测试

**测试指标**:

| 接口 | 并发数 | 响应时间(95th) | TPS | 错误率 |
|------|-------|---------------|-----|--------|
| /cart/add | 500 | < 200ms | > 2000 | < 0.1% |
| /cart/list/promotion | 1000 | < 500ms | > 1500 | < 0.1% |
| /cart/update/quantity | 500 | < 100ms | > 3000 | < 0.1% |

**测试场景**:
- 压力测试: 持续1小时,TPS稳定
- 负载测试: 逐步增加并发,找到系统瓶颈
- 浸泡测试: 持续24小时,无内存泄漏

#### 7.4 安全测试

**测试项**:

| 测试项 | 测试方法 | 预期结果 |
|-------|---------|----------|
| 价格篡改 | 修改请求参数中的price | 被服务端覆盖 |
| SQL注入 | 在productId中注入SQL | 参数化查询,无注入 |
| 越权访问 | 修改其他用户的购物车 | 验证失败 |
| XSS攻击 | 在商品名称中注入脚本 | 被转义 |
| CSRF攻击 | 跨站请求伪造 | Token验证失败 |

---

### 八、风险管理

#### 8.1 技术风险

| 风险 | 概率 | 影响 | 应对措施 |
|------|------|------|----------|
| 数据库迁移失败 | 中 | 高 | 先在测试环境验证,准备回滚脚本 |
| 性能优化效果不达标 | 中 | 中 | 准备Plan B(添加Redis缓存) |
| 第三方依赖升级冲突 | 低 | 中 | 充分测试,准备降级方案 |
| 并发问题导致数据不一致 | 中 | 高 | 增加事务控制,压力测试验证 |

#### 8.2 业务风险

| 风险 | 概率 | 影响 | 应对措施 |
|------|------|------|----------|
| 价格快照影响促销效果 | 高 | 中 | 设置合理的过期时间,提示用户 |
| 用户不理解价格变更提示 | 中 | 低 | 优化文案,增加帮助说明 |
| 库存提示引起恐慌性购买 | 低 | 低 | A/B测试,观察用户行为 |

#### 8.3 上线风险

| 风险 | 概率 | 影响 | 应对措施 |
|------|------|------|----------|
| 灰度发布期间Bug | 中 | 高 | 5%灰度 → 20% → 50% → 100%,逐步放量 |
| 数据库索引影响写入性能 | 低 | 中 | 监控数据库性能,必要时调整 |
| 高峰期流量冲击 | 中 | 高 | 避开业务高峰,准备扩容方案 |

---

### 九、上线方案

#### 9.1 灰度发布策略

**阶段1: 内测(1%用户, 1天)**
- 选择内部员工账号
- 全功能验证
- 收集反馈

**阶段2: 小流量灰度(5%用户, 2天)**
- 随机选择5%用户
- 监控核心指标
- 对比老版本数据

**阶段3: 中流量灰度(20%用户, 3天)**
- 扩大到20%用户
- 验证性能表现
- 确认无重大问题

**阶段4: 大流量灰度(50%用户, 3天)**
- 扩大到50%用户
- 全面性能测试
- 准备全量上线

**阶段5: 全量上线(100%用户)**
- 所有用户使用新版本
- 持续监控1周
- 收集用户反馈

#### 9.2 回滚方案

**触发条件**:
- 错误率 > 1%
- 响应时间 > 2秒
- 用户投诉量激增
- 发现严重Bug

**回滚步骤**:
1. 立即停止灰度,切回老版本(5分钟)
2. 分析问题原因(30分钟)
3. 修复问题后重新灰度(1天后)

**回滚脚本**:
```sql
-- 回滚数据库变更
ALTER TABLE oms_cart_item DROP COLUMN snapshot_price;
ALTER TABLE oms_cart_item DROP COLUMN snapshot_time;

-- 删除新增索引
ALTER TABLE pms_sku_stock DROP INDEX idx_product_id;
ALTER TABLE pms_product_ladder DROP INDEX idx_product_id;
ALTER TABLE pms_product_full_reduction DROP INDEX idx_product_id;
```

#### 9.3 监控指标

**业务指标**:
- 购物车添加成功率
- 促销计算准确率
- 价格变更提示触发率
- 用户投诉率

**技术指标**:
- 接口响应时间(P50/P95/P99)
- 接口错误率
- 数据库查询时间
- JVM内存使用率
- 线程池使用情况

**告警规则**:
```yaml
# Prometheus告警规则
groups:
- name: mall_cart_alerts
  rules:
  - alert: CartApiHighErrorRate
    expr: rate(http_requests_total{status=~"5..",path="/cart/*"}[5m]) > 0.01
    for: 5m
    annotations:
      summary: "购物车接口错误率过高"
      
  - alert: CartApiSlowResponse
    expr: histogram_quantile(0.95, rate(http_request_duration_seconds_bucket{path="/cart/*"}[5m])) > 0.5
    for: 5m
    annotations:
      summary: "购物车接口响应时间过长"
```

---

### 十、验收标准

#### 10.1 功能验收

- [ ] 所有P0级Bug已修复,无空指针/除零异常
- [ ] 所有P1级Bug已修复,精度问题已解决
- [ ] 价格验证功能正常,无安全漏洞
- [ ] 价格变更提示功能正常
- [ ] 库存状态提示功能正常
- [ ] 单元测试覆盖率 > 80%
- [ ] 集成测试全部通过

#### 10.2 性能验收

- [ ] 购物车查询响应时间 < 500ms (95th)
- [ ] 促销计算时间 < 200ms
- [ ] 数据库查询时间 < 100ms
- [ ] 并发1000 QPS无压力
- [ ] 压力测试持续1小时稳定

#### 10.3 安全验收

- [ ] 无高危安全漏洞
- [ ] 通过安全代码审计
- [ ] 通过渗透测试
- [ ] 敏感数据已加密
- [ ] 所有输入已验证

#### 10.4 用户体验验收

- [ ] 价格变更投诉率降低 > 80%
- [ ] 用户满意度 > 90%
- [ ] 页面加载时间无明显变化
- [ ] UI显示友好,无错乱

---

### 十一、附录

#### 11.1 相关文档

- [价格体系调用链路分析](./step-3-1-price-system-chain.md)
- [价格体系调用关系图](./step-3-2-price-system-diagrams.md)
- [Bug与安全漏洞分析(修正版)](./step-3-3-bugs-and-security-analysis-corrected.md)
- [分析过程与错误修正](./step-3-4-analysis-process-and-corrections.md)

#### 11.2 技术栈

- Spring Boot 2.7.5
- MyBatis 3.5.9
- MySQL 5.7
- Redis 7.0
- JDK 1.8

#### 11.3 联系人

| 角色 | 姓名 | 邮箱 | 职责 |
|------|------|------|------|
| 产品经理 | - | - | 需求确认 |
| Tech Lead | - | - | 技术方案 |
| 开发负责人 | - | - | 开发实施 |
| 测试负责人 | - | - | 测试验证 |
| 运维负责人 | - | - | 上线发布 |

---

**文档版本**: v1.0  
**创建时间**: 2026-01-02  
**创建人员**: 产品 + 技术团队  
**最后更新**: 2026-01-02  
**文档状态**: 待评审
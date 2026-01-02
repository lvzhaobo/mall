# mall电商系统 - 价格体系全链路调用分析

## 一、价格体系概述

### 1.1 价格类型

| 价格类型 | 数据库字段 | 优先级 | 说明 |
|---------|-----------|--------|------|
| **秒杀价** | flash_promotion_price | 最高 | 限时秒杀活动价格 |
| **单品促销价** | promotion_price | 高 | 商品单独设置的促销价 |
| **会员价** | member_price | 中 | 不同会员等级的专属价格 |
| **阶梯价** | ladder_discount | 中 | 购买数量达到阶梯时的折扣 |
| **满减价** | full_reduction | 中 | 满足金额条件的减免 |
| **原价** | price | 最低 | 商品基础价格 |

### 1.2 价格计算优先级

```
秒杀价 > 单品促销价 > (会员价/阶梯价/满减价) > 原价
```

---

## 二、前端交互层

### 2.1 用户触发场景

| 场景 | 触发时机 | 前端页面 |
|------|---------|---------|
| **商品详情页** | 查看商品时 | 展示原价、促销价、会员价 |
| **添加购物车** | 点击加入购物车 | 计算商品预估价格 |
| **购物车列表** | 查看购物车 | 展示促销后价格、优惠信息 |
| **确认订单页** | 生成订单前 | 计算总价、优惠券、运费 |
| **订单提交** | 提交订单 | 最终价格锁定 |

### 2.2 前端API调用

```
场景1: 查看购物车
GET /cart/list
   ↓
后端返回: List<CartPromotionItem>
   - 包含促销信息
   - 包含优惠金额
   - 包含促销活动描述

场景2: 生成确认订单
POST /order/generateConfirmOrder
Body: [cartIds]
   ↓
后端返回: ConfirmOrderResult
   - 商品总价
   - 活动优惠
   - 优惠券优惠
   - 运费
   - 应付金额
```

---

## 三、Controller 层

### 3.1 购物车价格计算入口

**文件**: `OmsCartItemController.java`

```java
@RequestMapping("/cart")
public class OmsCartItemController {
    
    // 1. 查看购物车（带促销价格）
    @GetMapping("/list")
    public CommonResult<List<CartPromotionItem>> list() {
        List<OmsCartItem> cartList = cartItemService.list(memberId);
        // 调用促销服务计算价格
        List<CartPromotionItem> items = 
            cartItemService.listPromotion(memberId, null);
        return CommonResult.success(items);
    }
    
    // 2. 添加商品到购物车
    @PostMapping("/add")
    public CommonResult add(@RequestBody OmsCartItem cartItem) {
        // 存储原始信息，价格计算在查询时进行
        int count = cartItemService.add(cartItem);
        return CommonResult.success(count);
    }
}
```

**链路说明**:
- 添加购物车时: 只存储商品ID、SKU、数量，不存储计算后的价格
- 查询购物车时: 实时计算促销价格和优惠信息

---

## 四、Service 层

### 4.1 购物车服务层

**文件**: `OmsCartItemServiceImpl.java`

```java
@Service
public class OmsCartItemServiceImpl {
    @Autowired
    private OmsPromotionService promotionService;
    
    // 获取带促销信息的购物车列表
    public List<CartPromotionItem> listPromotion(
            Long memberId, List<Long> cartIds) {
        // 1. 查询购物车商品
        List<OmsCartItem> cartItemList = list(memberId);
        
        // 2. 过滤指定购物车项
        if(CollUtil.isNotEmpty(cartIds)){
            cartItemList = cartItemList.stream()
                .filter(item->cartIds.contains(item.getId()))
                .collect(Collectors.toList());
        }
        
        // 3. 调用促销服务计算价格 ⭐核心调用
        List<CartPromotionItem> cartPromotionItemList = 
            promotionService.calcCartPromotion(cartItemList);
        
        return cartPromotionItemList;
    }
}
```

### 4.2 促销计算服务层（核心）

**文件**: `OmsPromotionServiceImpl.java`

```java
@Service
public class OmsPromotionServiceImpl {
    @Autowired
    private PortalProductDao portalProductDao;
    
    /**
     * 计算购物车促销价格 - 核心算法
     */
    public List<CartPromotionItem> calcCartPromotion(
            List<OmsCartItem> cartItemList) {
        
        // ========== 步骤1: 商品分组 ==========
        // 按SPU分组（同一商品不同SKU归为一组）
        Map<Long, List<OmsCartItem>> productCartMap = 
            groupCartItemBySpu(cartItemList);
        
        // ========== 步骤2: 查询促销信息 ==========
        // 从数据库查询所有商品的促销信息
        List<PromotionProduct> promotionProductList = 
            getPromotionProductList(cartItemList);
        
        // ========== 步骤3: 按促销类型计算价格 ==========
        List<CartPromotionItem> cartPromotionItemList = new ArrayList<>();
        
        for (Map.Entry<Long, List<OmsCartItem>> entry : productCartMap.entrySet()) {
            Long productId = entry.getKey();
            PromotionProduct promotionProduct = 
                getPromotionProductById(productId, promotionProductList);
            List<OmsCartItem> itemList = entry.getValue();
            
            // 获取促销类型
            Integer promotionType = promotionProduct.getPromotionType();
            
            if (promotionType == 1) {
                // ===== 类型1: 单品促销 =====
                handleSinglePromotion(itemList, promotionProduct);
                
            } else if (promotionType == 3) {
                // ===== 类型3: 阶梯价格 =====
                handleLadderPromotion(itemList, promotionProduct);
                
            } else if (promotionType == 4) {
                // ===== 类型4: 满减优惠 =====
                handleFullReduction(itemList, promotionProduct);
                
            } else {
                // ===== 无促销 =====
                handleNoReduce(itemList, promotionProduct);
            }
        }
        
        return cartPromotionItemList;
    }
}
```

### 4.3 促销类型计算细节

#### 类型1: 单品促销

```java
// 计算逻辑
for (OmsCartItem item : itemList) {
    CartPromotionItem cartPromotionItem = new CartPromotionItem();
    
    // 获取SKU库存信息（包含促销价）
    PmsSkuStock skuStock = getOriginalPrice(promotionProduct, item.getProductSkuId());
    BigDecimal originalPrice = skuStock.getPrice();
    BigDecimal promotionPrice = skuStock.getPromotionPrice();
    
    // 计算优惠金额
    cartPromotionItem.setPrice(originalPrice);
    cartPromotionItem.setReduceAmount(
        originalPrice.subtract(promotionPrice)
    );
    cartPromotionItem.setPromotionMessage("单品促销");
}
```

#### 类型3: 阶梯价格

```java
// 计算总数量
int count = getCartItemCount(itemList);

// 获取满足条件的阶梯价
PmsProductLadder ladder = getProductLadder(count, promotionProduct.getProductLadderList());

if(ladder != null) {
    for (OmsCartItem item : itemList) {
        // 原价 * (1 - 折扣)
        BigDecimal originalPrice = skuStock.getPrice();
        BigDecimal discount = ladder.getDiscount(); // 0.8表示8折
        BigDecimal reduceAmount = originalPrice.subtract(
            discount.multiply(originalPrice)
        );
        
        cartPromotionItem.setReduceAmount(reduceAmount);
        cartPromotionItem.setPromotionMessage(
            "满" + ladder.getCount() + "件，打" + 
            discount.multiply(new BigDecimal(10)) + "折"
        );
    }
}
```

#### 类型4: 满减优惠

```java
// 计算购物车总金额
BigDecimal totalAmount = getCartItemAmount(itemList, promotionProductList);

// 获取满足条件的满减规则
PmsProductFullReduction fullReduction = getProductFullReduction(
    totalAmount, 
    promotionProduct.getProductFullReductionList()
);

if(fullReduction != null) {
    for (OmsCartItem item : itemList) {
        // (商品原价/总价) * 满减金额
        BigDecimal originalPrice = skuStock.getPrice();
        BigDecimal reduceAmount = originalPrice
            .divide(totalAmount, RoundingMode.HALF_EVEN)
            .multiply(fullReduction.getReducePrice());
        
        cartPromotionItem.setReduceAmount(reduceAmount);
        cartPromotionItem.setPromotionMessage(
            "满" + fullReduction.getFullPrice() + "元，" +
            "减" + fullReduction.getReducePrice() + "元"
        );
    }
}
```

---

## 五、DAO 层

### 5.1 促销商品查询

**文件**: `PortalProductDao.java`

```java
public interface PortalProductDao {
    /**
     * 获取促销商品信息列表
     * 包含：商品基本信息、SKU信息、阶梯价、满减信息
     */
    List<PromotionProduct> getPromotionProductList(@Param("ids") List<Long> ids);
}
```

**映射文件**: `PortalProductDao.xml`

```xml
<select id="getPromotionProductList" resultMap="promotionProductMap">
    SELECT
        p.id,
        p.`name`,
        p.promotion_type,          -- 促销类型
        p.gift_growth,             -- 赠送成长值
        p.gift_point,              -- 赠送积分
        sku.id sku_id,
        sku.price sku_price,       -- SKU原价
        sku.promotion_price sku_promotion_price,  -- SKU促销价
        sku.stock sku_stock,
        sku.lock_stock sku_lock_stock,
        ladder.id ladder_id,
        ladder.count ladder_count,         -- 阶梯数量
        ladder.discount ladder_discount,   -- 阶梯折扣
        full_re.id full_id,
        full_re.full_price full_full_price,      -- 满减门槛
        full_re.reduce_price full_reduce_price   -- 满减金额
    FROM
        pms_product p
        LEFT JOIN pms_sku_stock sku ON p.id = sku.product_id
        LEFT JOIN pms_product_ladder ladder ON p.id = ladder.product_id
        LEFT JOIN pms_product_full_reduction full_re ON p.id = full_re.product_id
    WHERE
        p.id IN
        <foreach collection="ids" open="(" close=")" item="id" separator=",">
            #{id}
        </foreach>
</select>
```

---

## 六、数据库层

### 6.1 价格相关表结构

#### 表1: pms_product（商品主表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 商品ID |
| name | varchar | 商品名称 |
| price | decimal(10,2) | **原价** |
| promotion_type | int | 促销类型：0-无，1-单品，3-阶梯，4-满减 |
| gift_point | int | 赠送积分 |
| gift_growth | int | 赠送成长值 |

#### 表2: pms_sku_stock（SKU库存表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | SKU_ID |
| product_id | bigint | 商品ID |
| sku_code | varchar | SKU编码 |
| price | decimal(10,2) | **SKU原价** |
| **promotion_price** | decimal(10,2) | **SKU促销价（单品促销）** ⭐ |
| stock | int | 库存 |
| lock_stock | int | 锁定库存 |

#### 表3: pms_product_ladder（阶梯价格表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| product_id | bigint | 商品ID |
| **count** | int | **满足数量（阶梯数量）** ⭐ |
| **discount** | decimal(10,2) | **折扣（0.8表示8折）** ⭐ |

#### 表4: pms_product_full_reduction（满减表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| product_id | bigint | 商品ID |
| **full_price** | decimal(10,2) | **满减门槛（满XX元）** ⭐ |
| **reduce_price** | decimal(10,2) | **减免金额（减XX元）** ⭐ |

#### 表5: sms_flash_promotion（秒杀活动表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 活动ID |
| title | varchar | 活动名称 |
| start_date | datetime | 开始时间 |
| end_date | datetime | 结束时间 |
| status | int | 状态：0-下线，1-上线 |

#### 表6: sms_flash_promotion_product_relation（秒杀商品关联表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| flash_promotion_id | bigint | 秒杀活动ID |
| flash_promotion_session_id | bigint | 场次ID |
| product_id | bigint | 商品ID |
| **flash_promotion_price** | decimal(10,2) | **秒杀价格** ⭐ |
| flash_promotion_count | int | 秒杀数量 |
| flash_promotion_limit | int | 限购数量 |

#### 表7: pms_member_price（会员价格表）

| 字段 | 类型 | 说明 |
|------|------|------|
| id | bigint | 主键 |
| product_id | bigint | 商品ID |
| member_level_id | bigint | 会员等级ID |
| **member_price** | decimal(10,2) | **会员价格** ⭐ |

---

## 七、完整调用链路图

```
┌─────────────────────────────────────────────────────────────────┐
│                          前端页面                                │
│  - 商品详情页   - 购物车页面   - 确认订单页                      │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     │ HTTP Request
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                     Controller 层                                │
│  OmsCartItemController.list()                                   │
│  ├─ 路径: GET /cart/list                                        │
│  └─ 返回: List<CartPromotionItem>                               │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     │ 调用 Service
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                   Service 层（购物车）                           │
│  OmsCartItemServiceImpl.listPromotion()                         │
│  ├─ 查询购物车商品列表                                          │
│  └─ 调用促销计算服务 ⭐                                         │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     │ 调用促销服务
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│              Service 层（促销计算 - 核心）                       │
│  OmsPromotionServiceImpl.calcCartPromotion()                    │
│  ├─ 步骤1: 商品按SPU分组                                        │
│  ├─ 步骤2: 查询促销信息（调用DAO） ⭐                           │
│  ├─ 步骤3: 根据促销类型计算价格                                 │
│  │    ├─ promotionType=1: 单品促销                              │
│  │    ├─ promotionType=3: 阶梯价格                              │
│  │    ├─ promotionType=4: 满减优惠                              │
│  │    └─ promotionType=0: 无促销                                │
│  └─ 返回: List<CartPromotionItem>（带促销信息和优惠金额）       │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     │ 调用 DAO
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                        DAO 层                                    │
│  PortalProductDao.getPromotionProductList()                     │
│  └─ 执行复杂联表查询 ⭐                                         │
└────────────────────┬────────────────────────────────────────────┘
                     │
                     │ 执行 SQL
                     ▼
┌─────────────────────────────────────────────────────────────────┐
│                      数据库层（MySQL）                           │
│  联表查询（4张表）：                                             │
│  ├─ pms_product            （商品主表）                          │
│  ├─ pms_sku_stock          （SKU + 单品促销价）                 │
│  ├─ pms_product_ladder     （阶梯价规则）                        │
│  └─ pms_product_full_reduction （满减规则）                      │
│                                                                  │
│  返回: PromotionProduct对象                                      │
│  包含: 商品信息 + SKU列表 + 阶梯价列表 + 满减列表                │
└─────────────────────────────────────────────────────────────────┘
```

---

## 八、数据流转示例

### 8.1 单商品价格计算示例

**场景**: 用户购买"小米手机"2台

**步骤1: 查询商品促销信息**

```sql
-- 执行的SQL（简化）
SELECT 
    p.id, p.name, p.promotion_type,
    sku.price sku_price,
    sku.promotion_price sku_promotion_price,
    ladder.count ladder_count,
    ladder.discount ladder_discount
FROM pms_product p
LEFT JOIN pms_sku_stock sku ON p.id = sku.product_id
LEFT JOIN pms_product_ladder ladder ON p.id = ladder.product_id
WHERE p.id = 123
```

**查询结果**:
```json
{
  "id": 123,
  "name": "小米手机",
  "promotion_type": 3,  // 阶梯价格
  "skuStockList": [
    {
      "id": 1001,
      "price": 2999.00,
      "promotion_price": null
    }
  ],
  "productLadderList": [
    {
      "count": 2,
      "discount": 0.95  // 满2件，95折
    },
    {
      "count": 5,
      "discount": 0.90  // 满5件，9折
    }
  ]
}
```

**步骤2: 计算促销价格**

```java
// 购买数量: 2件
int count = 2;

// 匹配阶梯规则
PmsProductLadder ladder = getProductLadder(count, productLadderList);
// 结果: count=2, discount=0.95

// 计算优惠
BigDecimal originalPrice = new BigDecimal("2999.00");
BigDecimal discount = new BigDecimal("0.95");
BigDecimal finalPrice = originalPrice.multiply(discount);
// finalPrice = 2849.05

BigDecimal reduceAmount = originalPrice.subtract(finalPrice);
// reduceAmount = 149.95
```

**步骤3: 返回结果**

```json
{
  "productId": 123,
  "productName": "小米手机",
  "quantity": 2,
  "price": 2999.00,
  "reduceAmount": 149.95,
  "realPrice": 2849.05,
  "promotionMessage": "满2件，打9.5折"
}
```

### 8.2 多商品满减计算示例

**场景**: 购物车有3件商品，满200元减20元

| 商品 | 单价 | 数量 | 小计 |
|------|------|------|------|
| 商品A | 50元 | 1 | 50元 |
| 商品B | 100元 | 1 | 100元 |
| 商品C | 80元 | 1 | 80元 |
| **总计** | - | - | **230元** |

**满减规则**: 满200元减20元

**计算逻辑**:
```java
// 总金额
BigDecimal totalAmount = new BigDecimal("230.00");

// 满减金额
BigDecimal reducePrice = new BigDecimal("20.00");

// 商品A分摊的优惠 = (50 / 230) * 20 = 4.35元
BigDecimal reduceA = new BigDecimal("50")
    .divide(totalAmount, RoundingMode.HALF_EVEN)
    .multiply(reducePrice);

// 商品B分摊的优惠 = (100 / 230) * 20 = 8.70元
BigDecimal reduceB = new BigDecimal("100")
    .divide(totalAmount, RoundingMode.HALF_EVEN)
    .multiply(reducePrice);

// 商品C分摊的优惠 = (80 / 230) * 20 = 6.96元
BigDecimal reduceC = new BigDecimal("80")
    .divide(totalAmount, RoundingMode.HALF_EVEN)
    .multiply(reducePrice);
```

**最终结果**:

| 商品 | 原价 | 分摊优惠 | 实付 |
|------|------|---------|------|
| 商品A | 50元 | -4.35元 | 45.65元 |
| 商品B | 100元 | -8.70元 | 91.30元 |
| 商品C | 80元 | -6.96元 | 73.04元 |
| **总计** | 230元 | **-20.01元** | **209.99元** |

---

## 九、优惠券叠加计算

### 9.1 优惠券查询

**文件**: `UmsMemberCouponServiceImpl.java`

```java
// 获取购物车可用优惠券
public List<SmsCouponHistoryDetail> listCart(
        List<CartPromotionItem> cartPromotionItemList, Integer type) {
    
    // 1. 获取会员已领取的优惠券
    List<SmsCouponHistory> allList = getCouponHistoryList(memberId, 0);
    
    // 2. 计算购物车总金额（促销后价格）
    BigDecimal total = calcTotalAmount(cartPromotionItemList);
    
    // 3. 计算指定分类商品总金额
    BigDecimal categoryTotal = calcTotalAmountByCategory(
        cartPromotionItemList, categoryIds
    );
    
    // 4. 计算指定商品总金额
    BigDecimal productTotal = calcTotalAmountByProduct(
        cartPromotionItemList, productIds
    );
    
    // 5. 判断优惠券是否可用
    if(type == 1) {
        // 过滤可用优惠券
        return filterAvailableCoupon(allList, total);
    }
}
```

### 9.2 价格计算流程

```
原价 
  → 促销优惠（单品/阶梯/满减）
    → 促销后价格
      → 优惠券优惠
        → 最终价格
```

---

## 十、性能关键点

### 10.1 数据库查询

**问题点**:
- 4表联查，JOIN操作较多
- 每次查看购物车都要重新计算
- 没有缓存机制

**SQL示例**:
```sql
SELECT p.*, sku.*, ladder.*, full_re.*
FROM pms_product p
LEFT JOIN pms_sku_stock sku ON p.id = sku.product_id
LEFT JOIN pms_product_ladder ladder ON p.id = ladder.product_id
LEFT JOIN pms_product_full_reduction full_re ON p.id = full_re.product_id
WHERE p.id IN (1, 2, 3, ..., N)
```

### 10.2 价格计算逻辑

**特点**:
- ✅ 实时计算，保证价格准确性
- ❌ 每次都要查库和计算，性能开销大
- ❌ 商品多时计算复杂度高

---

## 十一、总结

### 11.1 链路特点

| 特点 | 说明 |
|------|------|
| **实时计算** | 每次查询都重新计算，保证价格准确 |
| **无缓存** | 没有价格缓存机制 |
| **联表查询** | 4张表联查，SQL较复杂 |
| **优先级清晰** | 价格计算优先级明确 |
| **支持叠加** | 促销+优惠券可叠加 |

### 11.2 涉及的核心类

| 层次 | 类名 | 职责 |
|------|------|------|
| Controller | OmsCartItemController | 购物车接口 |
| Service | OmsCartItemServiceImpl | 购物车业务 |
| Service | OmsPromotionServiceImpl | **促销计算核心** ⭐ |
| DAO | PortalProductDao | 促销商品查询 |
| Domain | PromotionProduct | 促销商品VO |
| Domain | CartPromotionItem | 购物车促销VO |

### 11.3 涉及的数据库表

| 表名 | 作用 |
|------|------|
| pms_product | 商品主表（原价、促销类型） |
| pms_sku_stock | SKU表（SKU价格、促销价） |
| pms_product_ladder | 阶梯价格规则 |
| pms_product_full_reduction | 满减规则 |
| sms_flash_promotion | 秒杀活动 |
| pms_member_price | 会员价格 |
| oms_cart_item | 购物车数据 |

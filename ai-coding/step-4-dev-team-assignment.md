# mall电商系统 - 价格体系优化 人员安排 (原始PRD版)

## 文档信息

| 项目 | 内容 |
|------|------|
| **文档名称** | 价格体系优化项目人员安排 |
| **文档版本** | v1.0 |
| **创建时间** | 2026-01-02 |
| **项目周期** | 14个工作日 (3周) |
| **关联文档** | [原始PRD](./step-4-dev.md), [Roadmap](./step-4-dev-roadmap.md) |

---

## 一、团队组成总览

### 1.1 团队人员配置

| 角色 | 姓名 | 投入时间 | 工作日 | 主要职责 | 关键技能 |
|------|------|---------|--------|---------|---------|
| **产品经理** | 产品张 | 10% | 1.4天 | 需求确认、产品验收 | 需求分析 |
| **Tech Lead** | 技术李 | 30% | 4.2天 | 技术方案设计、代码审查 | 架构设计 |
| **后端开发A** | 开发王 | 100% | 14天 | 核心开发、单元测试 | Java、Spring Boot |
| **后端开发C** | 开发刘 | 50% | 7天 | 性能优化、Map算法 | 算法、性能调优 |
| **前端开发E** | 前端吴 | 50% | 7天 | 前端页面改造 | Vue.js、Element UI |
| **QA测试** | 测试陈 | 80% | 11.2天 | 集成测试、性能测试 | JMeter、测试设计 |
| **安全工程师** | 安全赵 | 20% | 2.8天 | 安全测试、渗透测试 | 安全测试 |
| **DBA** | DBA孙 | 20% | 2.8天 | 数据库优化、索引设计 | MySQL优化 |
| **运维工程师** | 运维周 | 30% | 4.2天 | 灰度发布、监控配置 | Docker、监控 |

### 1.2 人力资源甘特图

```
角色          Week 1    Week 2    Week 3
产品经理      ██░░░░    ░░░░░░    ░░░░░░  (10%)
Tech Lead     ████░░    ██░░░░    ░░░░██  (30%)
开发A         ██████    ██████    ██████  (100%)
开发C         ░░░░░░    ░░░███    ░░░░░░  (50%)
前端E         ░░░░░░    ░░░░░░    ░░██░░  (50%)
测试B         ░░░░██    ░░░░██    ██████  (80%)
安全D         ░░░░░░    ░░░░██    ░░░░░░  (20%)
DBA           ░░░░░░    ░░██░░    ██░░░░  (20%)
运维F         ░░░░░░    ░░░░░░    ░░░░██  (30%)

图例: ██ = 工作时间  ░░ = 非工作时间
```

### 1.3 人力成本估算

| 角色 | 日均成本 | 投入天数 | 总成本 | 占比 |
|------|---------|---------|--------|-----|
| 产品经理 | ¥1,200 | 1.4天 | ¥1,680 | 2.6% |
| Tech Lead | ¥2,000 | 4.2天 | ¥8,400 | 13.1% |
| 后端开发A | ¥1,500 | 14天 | ¥21,000 | 32.8% |
| 后端开发C | ¥1,500 | 7天 | ¥10,500 | 16.4% |
| 前端开发E | ¥1,200 | 7天 | ¥8,400 | 13.1% |
| QA测试 | ¥1,000 | 11.2天 | ¥11,200 | 17.5% |
| 安全工程师 | ¥1,800 | 2.8天 | ¥5,040 | 7.9% |
| DBA | ¥1,800 | 2.8天 | ¥5,040 | 7.9% |
| 运维工程师 | ¥1,200 | 4.2天 | ¥5,040 | 7.9% |
| **总计** | - | **54.6人天** | **¥76,300** | **100%** |

---

## 二、分角色详细工作安排

### 👤 产品经理 (产品张)

**投入时间**: 10% (约1.4个工作日)

#### 工作内容

| 阶段 | 时间 | 任务 | 产出 | 优先级 |
|------|------|------|------|-------|
| **需求确认** | Day 1 (2h) | 参加Kick-off会议<br>确认需求优先级<br>确认价格快照是否实施 | 会议纪要 | P0 |
| **过程跟进** | Week 1-3 (3h) | 解答需求疑问<br>协调资源 | - | P1 |
| **产品验收** | Day 14 (3h) | 抽查关键场景<br>确认用户体验 | 验收报告 | P0 |

#### 关键决策

1. **Day 1**: 确认价格快照是否实施 (建议: 实施,降低用户投诉率)
2. **Day 5**: P0级Bug验收
3. **Day 14**: 最终产品验收

#### 联系方式

- **邮箱**: product.zhang@mall.com
- **钉钉**: @产品张
- **电话**: 139-XXXX-XXXX

---

### 👤 Tech Lead (技术李)

**投入时间**: 30% (约4.2个工作日)

#### 工作内容

| 阶段 | 时间 | 任务 | 产出 | 优先级 |
|------|------|------|------|-------|
| **技术方案设计** | Day 1 (4h) | 设计Bug修复方案<br>设计精度算法<br>设计性能优化方案 | 技术设计文档 | P0 |
| **方案评审** | Day 1 (2h) | 主持技术方案评审<br>确认技术可行性 | 评审纪要 | P0 |
| **代码审查** | Day 3 (2h) | 审查P0级代码<br>确认单元测试 | 代码审查报告 | P0 |
| **技术支持** | Week 1-3 (2天) | 解答技术问题<br>技术难点攻关 | 技术支持记录 | P1 |
| **团队培训** | Day 14 (2h) | 编写培训材料<br>讲解技术方案 | 培训材料 | P1 |

#### 关键技术决策

| 决策点 | 时间 | 决策内容 | 理由 |
|-------|------|---------|------|
| **优惠分摊算法** | Day 1 | 采用"最后补齐差额"算法 | 确保精度100%准确 |
| **BigDecimal精度** | Day 1 | 统一保留2位小数 | 符合金额规范 |
| **Map优化方案** | Day 1 | `Map<Long, PromotionProduct>` | O(N×M) → O(N+M) |
| **价格快照设计** | Day 1 | 添加snapshot_price字段 | 最小化表结构变更 |

#### 联系方式

- **邮箱**: techlead.li@mall.com
- **钉钉**: @技术李
- **电话**: 138-XXXX-XXXX

---

### 👤 后端开发A (开发王) - 核心开发

**投入时间**: 100% (14个工作日)

#### 迭代1: Bug修复 (Day 1-3)

| 时间 | 任务 | 文件 | 代码行数 | 单元测试 |
|------|------|------|---------|---------|
| **Day 1下午** | 环境准备 + 代码开发启动 | - | - | - |
| **Day 2上午** | P0-1: 空指针异常修复 | OmsPromotionServiceImpl.java | ~50行 | 6个测试用例 |
| **Day 2下午** | P0-2: 满减除零异常修复 | OmsPromotionServiceImpl.java | ~20行 | 4个测试用例 |
| **Day 2下午** | P0-3: 精度修复(启动) | OmsPromotionServiceImpl.java | ~30行 | - |
| **Day 3上午** | P0-3: 精度修复(完成) | OmsPromotionServiceImpl.java | ~40行 | 8个测试用例 |
| **Day 3下午** | 代码修改(审查意见) | 全部文件 | ~20行 | 修改测试 |

**小计**:
- 代码修改: ~160行
- 单元测试: ~18个用例
- 测试覆盖率: > 80%

**关键代码示例**:

```java
// 1. 空指针异常修复
PmsSkuStock skuStock = getOriginalPrice(promotionProduct, item.getProductSkuId());
if (skuStock == null) {
    log.error("SKU不存在: productId={}, skuId={}", item.getProductId(), item.getProductSkuId());
    continue;
}

// 2. 除零异常修复
if (totalAmount.compareTo(BigDecimal.ZERO) == 0) {
    log.error("满减优惠计算异常: 总金额为0, productId={}", promotionProduct.getId());
    handleNoReduce(cartPromotionItemList, itemList, promotionProduct);
    continue;
}

// 3. 最后补齐差额算法
BigDecimal totalReduce = BigDecimal.ZERO;
for (int i = 0; i < itemList.size(); i++) {
    BigDecimal reduceAmount;
    if (i == itemList.size() - 1) {
        reduceAmount = fullReduction.getReducePrice().subtract(totalReduce);
    } else {
        reduceAmount = originalPrice
            .divide(totalAmount, 2, RoundingMode.HALF_EVEN)
            .multiply(fullReduction.getReducePrice());
        totalReduce = totalReduce.add(reduceAmount);
    }
    cartPromotionItem.setReduceAmount(reduceAmount);
}
```

---

#### 迭代2: 安全增强 + 性能优化 (Day 4-8)

| 时间 | 任务 | 文件 | 代码行数 | 单元测试 |
|------|------|------|---------|---------|
| **Day 4上午** | P1-1: 满减规则精度修复 | OmsPromotionServiceImpl.java | ~15行 | 3个测试用例 |
| **Day 4下午** | P1-2: 阶梯排序整数溢出修复 | OmsPromotionServiceImpl.java | ~10行 | 2个测试用例 |
| **Day 5** | 集成测试支持 + Bug修复 | 全部文件 | ~20行 | - |
| **Day 6** | P1-3: 购物车添加价格验证 | OmsCartItemServiceImpl.java | ~80行 | 10个测试用例 |

**小计**:
- 代码修改: ~125行
- 单元测试: ~15个用例

**关键代码示例**:

```java
// 1. 满减规则精度修复
fullReductionList.sort((o1, o2) -> 
    o2.getFullPrice().compareTo(o1.getFullPrice())  // ✅ 使用compareTo()
);

for(PmsProductFullReduction fullReduction : fullReductionList){
    if(totalAmount.compareTo(fullReduction.getFullPrice()) >= 0){  // ✅ 精度正确
        return fullReduction;
    }
}

// 2. 阶梯排序优化
productLadderList.sort(
    Comparator.comparingInt(PmsProductLadder::getCount).reversed()
);

// 3. 购物车添加价格验证
public int add(OmsCartItem cartItem) {
    // 从数据库查询真实商品信息
    CartProduct product = productDao.getCartProduct(cartItem.getProductId());
    if (product == null) {
        throw new ApiException("商品不存在");
    }
    
    // 强制使用服务端数据
    cartItem.setProductName(product.getName());
    cartItem.setProductPic(product.getPic());
    cartItem.setPrice(product.getPrice());
    
    // 验证SKU
    if (cartItem.getProductSkuId() != null) {
        PmsSkuStock sku = product.getSkuStockList().stream()
            .filter(s -> s.getId().equals(cartItem.getProductSkuId()))
            .findFirst().orElseThrow(() -> new ApiException("SKU不存在"));
        
        cartItem.setPrice(sku.getPrice());
        
        // 验证库存
        if (sku.getStock() - sku.getLockStock() < cartItem.getQuantity()) {
            throw new ApiException("库存不足");
        }
    }
    
    // ... 插入逻辑
}
```

---

#### 迭代3: 用户体验优化 (Day 9-12)

| 时间 | 任务 | 文件 | 代码行数 | 单元测试 |
|------|------|------|---------|---------|
| **Day 9下午** | P1-6: 价格快照机制(启动) | OmsCartItemServiceImpl.java | ~40行 | - |
| **Day 10上午** | P1-6: 价格快照机制(完成) | OmsCartItemServiceImpl.java | ~50行 | 8个测试用例 |
| **Day 10下午** | P2-1: 库存状态友好提示 | OmsPromotionServiceImpl.java | ~30行 | 4个测试用例 |
| **Day 11下午** | P2-2: IN查询参数数量限制 | OmsPromotionServiceImpl.java | ~10行 | 2个测试用例 |
| **Day 12上午** | P2-3: 数据库排序优化 | PortalProductDao.xml | ~5行 | - |

**小计**:
- 代码修改: ~135行
- 单元测试: ~14个用例

**关键代码示例**:

```java
// 1. 价格快照 - 添加时记录
public int add(OmsCartItem cartItem) {
    CartProduct product = productDao.getCartProduct(cartItem.getProductId());
    
    // 记录价格快照
    cartItem.setPrice(product.getPrice());
    cartItem.setSnapshotPrice(product.getPrice());
    cartItem.setSnapshotTime(new Date());
    cartItem.setCreateDate(new Date());
    
    return cartItemMapper.insert(cartItem);
}

// 2. 价格快照 - 查询时比较
public List<CartPromotionItem> listPromotion(Long memberId, List<Long> cartIds) {
    List<OmsCartItem> cartItemList = list(memberId);
    List<CartPromotionItem> cartPromotionItemList = promotionService.calcCartPromotion(cartItemList);
    
    // 检查价格变化
    for (int i = 0; i < cartPromotionItemList.size(); i++) {
        CartPromotionItem item = cartPromotionItemList.get(i);
        OmsCartItem originalItem = cartItemList.get(i);
        
        if (originalItem.getSnapshotPrice() != null) {
            BigDecimal currentPrice = item.getPrice();
            BigDecimal snapshotPrice = originalItem.getSnapshotPrice();
            
            if (currentPrice.compareTo(snapshotPrice) != 0) {
                item.setPriceChanged(true);
                item.setOldPrice(snapshotPrice);
                item.setWarningMessage(String.format(
                    "价格已从%.2f元变更为%.2f元", 
                    snapshotPrice, currentPrice
                ));
            }
        }
    }
    return cartPromotionItemList;
}

// 3. 库存状态提示
int realStock = Math.max(0, skuStock.getStock() - skuStock.getLockStock());
cartPromotionItem.setRealStock(realStock);

if (realStock == 0) {
    cartPromotionItem.setStockStatus("OUT_OF_STOCK");
    cartPromotionItem.setStockMessage("商品已售罄");
} else if (realStock < item.getQuantity()) {
    cartPromotionItem.setStockStatus("INSUFFICIENT");
    cartPromotionItem.setStockMessage(String.format("库存不足,仅剩%d件", realStock));
} else if (realStock < 10) {
    cartPromotionItem.setStockStatus("LOW_STOCK");
    cartPromotionItem.setStockMessage(String.format("库存紧张,仅剩%d件", realStock));
} else {
    cartPromotionItem.setStockStatus("IN_STOCK");
    cartPromotionItem.setStockMessage("有货");
}

// 4. IN查询参数数量限制
private List<PromotionProduct> getPromotionProductList(List<OmsCartItem> cartItemList) {
    List<Long> productIdList = new ArrayList<>();
    for(OmsCartItem cartItem : cartItemList){
        productIdList.add(cartItem.getProductId());
    }
    
    // 限制查询数量
    if (productIdList.size() > 100) {
        throw new ApiException("购物车商品数量超过限制(最多100个)");
    }
    
    return portalProductDao.getPromotionProductList(productIdList);
}
```

---

#### 迭代4: 上线准备 (Day 13)

| 时间 | 任务 | 产出 |
|------|------|------|
| **Day 13下午** | 文档更新 | 技术文档、接口文档 |

---

#### 总工作量统计

| 阶段 | 代码修改 | 单元测试 | 文档 | 总工时 |
|------|---------|---------|------|--------|
| 迭代1 | ~160行 | ~18个用例 | - | 3天 |
| 迭代2 | ~125行 | ~15个用例 | - | 3天 |
| 迭代3 | ~135行 | ~14个用例 | - | 4天 |
| 迭代4 | - | - | 技术文档 | 0.5天 |
| **总计** | **~420行** | **~47个用例** | **3份文档** | **14天** |

#### 关键产出

1. **代码文件**:
   - OmsPromotionServiceImpl.java (修改~250行)
   - OmsCartItemServiceImpl.java (修改~170行)
   - PortalProductDao.xml (修改~5行)

2. **单元测试**:
   - ~47个测试用例
   - 覆盖率目标: > 80%

3. **技术文档**:
   - 技术方案文档
   - 接口文档
   - 上线文档

#### 联系方式

- **邮箱**: developer.wang@mall.com
- **钉钉**: @开发王
- **电话**: 137-XXXX-XXXX

---

### 👤 后端开发C (开发刘) - 性能优化

**投入时间**: 50% (约7个工作日)

#### 工作内容

| 时间 | 任务 | 产出 | 优先级 |
|------|------|------|-------|
| **Day 7** (1天) | P1-4: 促销商品查找优化<br>List → Map转换<br>O(N×M) → O(N+M)优化 | 优化代码+测试 | P1 |
| **Week 2-3** (兼职) | 性能调优支持<br>算法优化建议 | 技术支持 | P1 |

#### 关键优化代码

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
        PromotionProduct promotionProduct = promotionProductMap.get(productId);  // O(1)查询
        
        // ... 促销计算逻辑
    }
    return cartPromotionItemList;
}
```

#### 性能提升预期

| 商品数量 | 优化前 | 优化后 | 提升 |
|---------|-------|-------|-----|
| 10个 | ~100ms | ~50ms | 50% |
| 50个 | ~800ms | ~200ms | 75% |
| 100个 | ~2000ms | ~400ms | 80% |

#### 联系方式

- **邮箱**: developer.liu@mall.com
- **钉钉**: @开发刘
- **电话**: 136-XXXX-XXXX

---

### 👤 前端开发E (前端吴)

**投入时间**: 50% (约7个工作日)

#### 工作内容

| 时间 | 任务 | 产出 | 优先级 |
|------|------|------|-------|
| **Day 11** (1天) | 价格变更提示页面<br>库存状态提示页面<br>接口对接 | 前端代码 | P1 |
| **Day 12上午** (0.5天) | 前后端联调 | 联调完成 | P0 |
| **Week 2-3** (兼职) | 前端问题修复 | Bug修复 | P1 |

#### 前端页面改造

**1. 价格变更提示**:
```vue
<template>
  <div class="cart-item">
    <div class="product-info">
      <span class="product-name">{{ item.productName }}</span>
      <span class="product-price">¥{{ item.price }}</span>
      
      <!-- 价格变更提示 -->
      <div v-if="item.priceChanged" class="price-warning">
        <i class="el-icon-warning"></i>
        <span>{{ item.warningMessage }}</span>
        <span class="old-price">原价: ¥{{ item.oldPrice }}</span>
      </div>
    </div>
  </div>
</template>

<style>
.price-warning {
  color: #E6A23C;
  background: #FDF6EC;
  padding: 8px;
  border-radius: 4px;
  margin-top: 8px;
}
</style>
```

**2. 库存状态提示**:
```vue
<template>
  <div class="stock-info" :class="getStockClass(item.stockStatus)">
    <i :class="getStockIcon(item.stockStatus)"></i>
    <span>{{ item.stockMessage }}</span>
  </div>
</template>

<script>
export default {
  methods: {
    getStockClass(status) {
      return {
        'OUT_OF_STOCK': 'stock-out',
        'INSUFFICIENT': 'stock-low',
        'LOW_STOCK': 'stock-warning',
        'IN_STOCK': 'stock-ok'
      }[status];
    },
    getStockIcon(status) {
      return {
        'OUT_OF_STOCK': 'el-icon-close',
        'INSUFFICIENT': 'el-icon-warning',
        'LOW_STOCK': 'el-icon-warning',
        'IN_STOCK': 'el-icon-check'
      }[status];
    }
  }
}
</script>

<style>
.stock-out { color: #F56C6C; }
.stock-low { color: #E6A23C; }
.stock-warning { color: #E6A23C; }
.stock-ok { color: #67C23A; }
</style>
```

#### 联系方式

- **邮箱**: frontend.wu@mall.com
- **钉钉**: @前端吴
- **电话**: 135-XXXX-XXXX

---

### 👤 QA测试 (测试陈)

**投入时间**: 80% (约11.2个工作日)

#### 测试计划

| 阶段 | 时间 | 测试内容 | 测试用例数 | 产出 |
|------|------|---------|----------|------|
| **迭代1测试** | Day 5 (0.5天) | 5个Bug修复测试 | ~20个用例 | 测试报告 |
| **迭代2测试** | Day 8上午 (0.5天) | 性能测试 | ~15个用例 | 性能报告 |
| **迭代3测试** | Day 12下午 (0.5天) | 体验优化测试 | ~15个用例 | 测试报告 |
| **回归测试** | Day 13上午 (0.5天) | 完整回归测试 | ~50个用例 | 回归测试报告 |
| **日常测试** | Week 1-3 (9天) | Bug验证、冒烟测试 | - | 测试记录 |

---

#### 迭代1: Bug修复测试 (Day 5)

**测试用例清单**:

| 测试项 | 用例数 | 测试方法 | 预期结果 |
|-------|-------|---------|---------|
| 空指针异常 | 6个 | 单元测试 + 集成测试 | 无NullPointerException |
| 除零异常 | 4个 | 边界值测试 | 无ArithmeticException |
| 精度问题 | 8个 | 财务对账测试 | 100%准确 |
| 回归测试 | 5个 | 功能测试 | 无功能回归 |

---

#### 迭代2: 性能测试 (Day 8)

**性能测试目标**:

| 指标 | 当前值 | 目标值 | 测试方法 |
|------|-------|--------|---------|
| 购物车查询(10商品) | ~100ms | < 200ms (95th) | JMeter 500并发 |
| 购物车查询(50商品) | ~800ms | < 500ms (95th) | JMeter 1000并发 |
| 购物车添加 | ~150ms | < 200ms (95th) | JMeter 500并发 |
| QPS | ~500 | > 1000 | JMeter压力测试 |

**JMeter测试脚本**:
```xml
<TestPlan>
  <ThreadGroup numThreads="1000" rampUp="10" duration="300">
    <HTTPSamplerProxy>
      <stringProp name="path">/cart/list/promotion</stringProp>
      <stringProp name="method">GET</stringProp>
    </HTTPSamplerProxy>
  </ThreadGroup>
</TestPlan>
```

---

#### 迭代3: 体验优化测试 (Day 12)

**测试场景**:

| 场景ID | 场景名称 | 测试步骤 | 预期结果 |
|--------|---------|---------|----------|
| TC-001 | 价格变更提示 | 1. 添加商品(100元)<br>2. 修改价格为120元<br>3. 查询购物车 | 显示价格变更提示 |
| TC-002 | 库存不足提示 | 1. 添加商品(库存5个)<br>2. 购买10个 | 显示"库存不足,仅剩5件" |
| TC-003 | 库存紧张提示 | 1. 添加商品(库存8个)<br>2. 查询购物车 | 显示"库存紧张,仅剩8件" |

---

#### 回归测试 (Day 13)

**测试清单**:

| 测试模块 | 用例数 | 测试方法 | 预期结果 |
|---------|-------|---------|---------|
| 购物车查询 | 15个 | 功能测试 | 全部通过 |
| 购物车添加 | 10个 | 功能测试 | 全部通过 |
| 购物车修改 | 8个 | 功能测试 | 全部通过 |
| 促销计算 | 12个 | 功能测试 | 全部通过 |
| 性能回归 | 5个 | 性能测试 | 无性能下降 |

#### 联系方式

- **邮箱**: qa.chen@mall.com
- **钉钉**: @测试陈
- **电话**: 134-XXXX-XXXX

---

### 👤 安全工程师 (安全赵)

**投入时间**: 20% (约2.8个工作日)

#### 安全测试计划

| 时间 | 测试内容 | 测试方法 | 预期结果 |
|------|---------|---------|---------|
| **Day 8下午** (0.5天) | **安全测试** | | |
| | 价格篡改测试 | 手动测试 | 无法篡改价格 |
| | SQL注入测试 | SQLMap扫描 | 无SQL注入漏洞 |
| | XSS攻击测试 | 手动测试 | 无XSS漏洞 |
| | 越权访问测试 | 手动测试 | 无越权问题 |
| **Week 2-3** (兼职) | **安全支持** | | |
| | 代码审查 | SonarQube | 无Critical问题 |

#### 价格篡改测试用例

| 测试场景 | 篡改方式 | 预期结果 | 实际结果 |
|---------|---------|---------|---------|
| 添加购物车 | 客户端传入0.01元 | 实际保存真实价格 | ⏳ 待测试 |
| 添加购物车 | 客户端传入9999.99元 | 实际保存真实价格 | ⏳ 待测试 |
| 修改商品名称 | 客户端传入恶意名称 | 实际保存真实名称 | ⏳ 待测试 |

#### 联系方式

- **邮箱**: security.zhao@mall.com
- **钉钉**: @安全赵
- **电话**: 133-XXXX-XXXX

---

### 👤 DBA (DBA孙)

**投入时间**: 20% (约2.8个工作日)

#### 数据库优化计划

| 时间 | 任务 | 产出 | 风险 |
|------|------|------|------|
| **Day 7** (0.5天) | 索引方案评审<br>添加3个索引 | 索引脚本 | 中 |
| **Day 9** (0.5天) | 表结构变更<br>数据迁移 | 变更完成 | 中 |
| **Week 2-3** (兼职) | 性能监控<br>慢查询优化 | 监控报告 | 低 |

#### 索引变更清单

```sql
-- Day 7: 添加索引
ALTER TABLE pms_sku_stock ADD INDEX idx_product_id (product_id);
ALTER TABLE pms_product_ladder ADD INDEX idx_product_id (product_id);
ALTER TABLE pms_product_full_reduction ADD INDEX idx_product_id (product_id);

-- 验证索引效果
EXPLAIN SELECT ...;
```

#### 表结构变更 (Day 9)

```sql
-- 添加价格快照字段
ALTER TABLE oms_cart_item 
ADD COLUMN snapshot_price DECIMAL(10,2) DEFAULT NULL COMMENT '价格快照' AFTER price,
ADD COLUMN snapshot_time DATETIME DEFAULT NULL COMMENT '快照时间' AFTER snapshot_price;

-- 数据迁移
UPDATE oms_cart_item c
INNER JOIN pms_product p ON c.product_id = p.id
SET c.snapshot_price = c.price,
    c.snapshot_time = c.create_date
WHERE c.snapshot_price IS NULL;
```

#### 回滚脚本

```sql
-- 回滚表结构变更
ALTER TABLE oms_cart_item DROP COLUMN snapshot_price;
ALTER TABLE oms_cart_item DROP COLUMN snapshot_time;

-- 删除索引
ALTER TABLE pms_sku_stock DROP INDEX idx_product_id;
ALTER TABLE pms_product_ladder DROP INDEX idx_product_id;
ALTER TABLE pms_product_full_reduction DROP INDEX idx_product_id;
```

#### 联系方式

- **邮箱**: dba.sun@mall.com
- **钉钉**: @DBA孙
- **电话**: 132-XXXX-XXXX

---

### 👤 运维工程师 (运维周)

**投入时间**: 30% (约4.2个工作日)

#### 运维支持计划

| 时间 | 任务 | 产出 | 优先级 |
|------|------|------|-------|
| **Day 1** (0.5天) | 搭建测试环境 | 测试环境 | P0 |
| **Day 13** (1天) | 灰度发布方案<br>监控配置 | 灰度方案 | P0 |
| **Day 14** (1天) | 灰度发布执行<br>持续监控 | 生产环境 | P0 |
| **Week 3** (兼职) | 灰度观察<br>应急处理 | 监控报告 | P0 |

#### 灰度发布方案

**灰度策略**: 1% → 5% → 20% → 50% → 100%

| 阶段 | 流量比例 | 观察期 | 回滚条件 |
|------|---------|--------|---------|
| 内测 | 1% | 1天 | 发现严重Bug |
| 小流量 | 5% | 2天 | 错误率 > 0.1% |
| 中流量 | 20% | 3天 | 响应时间 > 1s |
| 大流量 | 50% | 3天 | 用户投诉增加 |
| 全量 | 100% | 持续监控 | - |

#### 监控配置

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

#### 联系方式

- **邮箱**: devops.zhou@mall.com
- **钉钉**: @运维周
- **电话**: 131-XXXX-XXXX

---

## 三、协作与沟通

### 3.1 会议安排

| 会议名称 | 频率 | 时间 | 参与人 | 主持人 |
|---------|------|------|-------|--------|
| **Kick-off会议** | 1次 (Day 1) | 09:00-10:00 | 全员 | 产品张 |
| **技术方案评审** | 1次 (Day 1) | 14:00-16:00 | 技术团队 | 技术李 |
| **每日站会** | 每天 | 09:30-09:45 | 开发+测试 | 技术李 |
| **里程碑评审** | 3次 (Day 5, 8, 14) | 下午 | 全员 | 产品张 |

### 3.2 沟通渠道

| 渠道 | 用途 | 响应时间 |
|------|------|---------|
| **钉钉群** | 日常沟通、问题咨询 | 30分钟内 |
| **邮件** | 正式通知、文档分享 | 4小时内 |
| **电话** | 紧急问题 | 立即响应 |
| **JIRA** | 任务管理、Bug跟踪 | 每日查看 |

---

## 四、绩效考核

### 4.1 团队目标

| 目标 | 权重 | 评价标准 |
|------|-----|---------|
| **按时交付** | 30% | Day 14完成灰度发布 |
| **质量达标** | 30% | 无P0/P1级遗留Bug |
| **性能达标** | 20% | 响应时间 < 500ms |
| **用户满意** | 20% | 投诉率降低80% |

### 4.2 个人考核

| 角色 | 考核指标 | 权重 |
|------|---------|-----|
| **开发A/C** | 代码质量、单元测试覆盖率、Bug修复及时性 | 100% |
| **前端E** | 页面质量、用户体验、联调配合度 | 100% |
| **测试B** | 测试覆盖率、Bug发现率、测试及时性 | 100% |
| **安全D** | 安全问题发现率、安全测试完整性 | 100% |
| **DBA** | 索引优化效果、数据库稳定性 | 100% |
| **运维F** | 灰度发布成功率、监控及时性 | 100% |

---

## 五、版本信息

| 项目 | 内容 |
|------|------|
| **文档版本** | v1.0 |
| **创建时间** | 2026-01-02 |
| **基于PRD** | step-4-dev.md (原始版本) |
| **项目周期** | 14天 (3周) |
| **人力成本** | ¥76,300 |
| **文档状态** | ✅ 待评审 |

---

## 附录: 快速联系表

| 姓名 | 角色 | 钉钉 | 电话 | 邮箱 |
|------|------|------|------|------|
| 产品张 | 产品经理 | @产品张 | 139-XXXX-XXXX | product.zhang@mall.com |
| 技术李 | Tech Lead | @技术李 | 138-XXXX-XXXX | techlead.li@mall.com |
| 开发王 | 后端开发A | @开发王 | 137-XXXX-XXXX | developer.wang@mall.com |
| 开发刘 | 后端开发C | @开发刘 | 136-XXXX-XXXX | developer.liu@mall.com |
| 前端吴 | 前端开发E | @前端吴 | 135-XXXX-XXXX | frontend.wu@mall.com |
| 测试陈 | QA测试 | @测试陈 | 134-XXXX-XXXX | qa.chen@mall.com |
| 安全赵 | 安全工程师 | @安全赵 | 133-XXXX-XXXX | security.zhao@mall.com |
| DBA孙 | DBA | @DBA孙 | 132-XXXX-XXXX | dba.sun@mall.com |
| 运维周 | 运维工程师 | @运维周 | 131-XXXX-XXXX | devops.zhou@mall.com |

---

**文档创建**: 2026-01-02  
**审批状态**: ✅ 待评审  
**下次更新**: 项目启动后每周更新

# mall电商系统 - 价格体系调用关系图

## 一、时序图（购物车价格计算完整流程）

### 1.1 购物车查询价格计算时序图

```mermaid
sequenceDiagram
    actor 用户
    participant 前端
    participant Controller as OmsCartItemController
    participant CartService as OmsCartItemService
    participant PromotionService as OmsPromotionService
    participant DAO as PortalProductDao
    participant DB as MySQL数据库

    用户->>前端: 1. 查看购物车
    前端->>Controller: 2. GET /cart/list
    
    Controller->>CartService: 3. listPromotion(memberId, null)
    
    Note over CartService: 查询购物车商品
    CartService->>DB: 4. SELECT * FROM oms_cart_item<br/>WHERE member_id = ?
    DB-->>CartService: 5. 返回购物车商品列表
    
    CartService->>PromotionService: 6. calcCartPromotion(cartItemList)
    
    Note over PromotionService: 步骤1: 按SPU分组
    PromotionService->>PromotionService: 7. groupCartItemBySpu()
    
    Note over PromotionService: 步骤2: 查询促销信息
    PromotionService->>DAO: 8. getPromotionProductList(productIds)
    DAO->>DB: 9. LEFT JOIN 4表查询<br/>pms_product + sku_stock<br/>+ ladder + full_reduction
    DB-->>DAO: 10. 返回促销商品信息
    DAO-->>PromotionService: 11. List<PromotionProduct>
    
    Note over PromotionService: 步骤3: 计算促销价格
    loop 遍历每个商品
        alt 促销类型=1 单品促销
            PromotionService->>PromotionService: 12a. 原价 - 促销价
        else 促销类型=3 阶梯价格
            PromotionService->>PromotionService: 12b. 原价 × 折扣
        else 促销类型=4 满减优惠
            PromotionService->>PromotionService: 12c. 按比例分摊满减金额
        else 无促销
            PromotionService->>PromotionService: 12d. 优惠金额=0
        end
    end
    
    PromotionService-->>CartService: 13. List<CartPromotionItem><br/>(含促销信息和优惠金额)
    CartService-->>Controller: 14. 返回计算结果
    Controller-->>前端: 15. JSON响应
    前端->>用户: 16. 显示购物车<br/>展示促销价格和优惠信息
```

---

## 二、架构图（价格体系分层架构）

### 2.1 价格计算系统分层架构图

```mermaid
graph TB
    subgraph 表现层
        A1[商品详情页]
        A2[购物车页面]
        A3[确认订单页]
    end
    
    subgraph Controller层
        B1[OmsCartItemController<br/>购物车接口]
        B2[OmsPortalOrderController<br/>订单接口]
    end
    
    subgraph Service层
        C1[OmsCartItemService<br/>购物车服务]
        C2[OmsPromotionService<br/>促销计算核心⭐]
        C3[UmsMemberCouponService<br/>优惠券服务]
    end
    
    subgraph DAO层
        D1[PortalProductDao<br/>促销商品查询]
        D2[OmsCartItemMapper<br/>购物车数据]
    end
    
    subgraph 数据库层
        E1[(pms_product<br/>商品主表)]
        E2[(pms_sku_stock<br/>SKU+促销价)]
        E3[(pms_product_ladder<br/>阶梯价)]
        E4[(pms_product_full_reduction<br/>满减)]
        E5[(oms_cart_item<br/>购物车)]
        E6[(sms_coupon<br/>优惠券)]
    end
    
    A1 --> B1
    A2 --> B1
    A3 --> B2
    
    B1 --> C1
    B2 --> C1
    B2 --> C3
    
    C1 --> C2
    C1 --> D2
    C2 --> D1
    C3 --> D1
    
    D1 --> E1
    D1 --> E2
    D1 --> E3
    D1 --> E4
    D2 --> E5
    C3 --> E6
    
    style C2 fill:#ff9999
    style D1 fill:#ffcc99
```

---

## 三、流程图（促销价格计算核心逻辑）

### 3.1 促销计算主流程

```mermaid
flowchart TD
    Start([开始: calcCartPromotion]) --> Step1[步骤1: 按SPU分组<br/>groupCartItemBySpu]
    
    Step1 --> Step2[步骤2: 查询促销信息<br/>getPromotionProductList]
    
    Step2 --> Step3{步骤3: 遍历每个商品}
    
    Step3 --> GetType[获取促销类型<br/>promotionType]
    
    GetType --> CheckType{判断促销类型}
    
    CheckType -->|type=1| Type1[单品促销<br/>handleSinglePromotion]
    CheckType -->|type=3| Type3[阶梯价格<br/>handleLadderPromotion]
    CheckType -->|type=4| Type4[满减优惠<br/>handleFullReduction]
    CheckType -->|type=0| Type0[无促销<br/>handleNoReduce]
    
    Type1 --> Calc1[计算: 原价 - 促销价]
    Type3 --> Calc3[计算: 原价 × 折扣]
    Type4 --> Calc4[计算: 按比例分摊满减]
    Type0 --> Calc0[优惠金额 = 0]
    
    Calc1 --> AddResult[添加到结果列表<br/>CartPromotionItem]
    Calc3 --> AddResult
    Calc4 --> AddResult
    Calc0 --> AddResult
    
    AddResult --> HasMore{还有商品?}
    HasMore -->|是| Step3
    HasMore -->|否| End([返回计算结果])
    
    style GetType fill:#e1f5ff
    style CheckType fill:#fff4e1
    style Type1 fill:#ffe1e1
    style Type3 fill:#e1ffe1
    style Type4 fill:#f0e1ff
    style AddResult fill:#ffffe1
```

### 3.2 单品促销计算详细流程

```mermaid
flowchart TD
    Start([单品促销计算]) --> Loop{遍历购物车项}
    
    Loop -->|每一项| GetSku[获取SKU信息<br/>getOriginalPrice]
    
    GetSku --> GetPrice[获取原价<br/>skuStock.getPrice]
    GetPrice --> GetPromo[获取促销价<br/>skuStock.getPromotionPrice]
    
    GetPromo --> CalcReduce[计算优惠金额<br/>reduceAmount = <br/>originalPrice - promotionPrice]
    
    CalcReduce --> SetFields[设置字段]
    
    SetFields --> Field1[setPrice: 原价]
    Field1 --> Field2[setReduceAmount: 优惠金额]
    Field2 --> Field3[setPromotionMessage: 单品促销]
    Field3 --> Field4[setRealStock: 可用库存]
    Field4 --> Field5[setIntegration: 赠送积分]
    Field5 --> Field6[setGrowth: 赠送成长值]
    
    Field6 --> AddItem[添加到结果列表]
    
    AddItem --> Loop
    Loop -->|完成| End([返回结果])
    
    style CalcReduce fill:#ff9999
    style SetFields fill:#99ccff
```

### 3.3 阶梯价格计算详细流程

```mermaid
flowchart TD
    Start([阶梯价格计算]) --> CalcCount[计算购买总数量<br/>getCartItemCount]
    
    CalcCount --> GetLadder[获取满足条件的阶梯<br/>getProductLadder]
    
    GetLadder --> CheckLadder{阶梯规则存在?}
    
    CheckLadder -->|否| NoPromo[按无促销处理<br/>handleNoReduce]
    NoPromo --> End
    
    CheckLadder -->|是| Loop{遍历购物车项}
    
    Loop -->|每一项| GetSku[获取SKU原价<br/>skuStock.getPrice]
    
    GetSku --> GetDiscount[获取折扣<br/>ladder.getDiscount]
    
    GetDiscount --> CalcReduce[计算优惠金额<br/>reduceAmount = <br/>originalPrice - <br/>discount × originalPrice]
    
    CalcReduce --> BuildMsg[构建促销消息<br/>满X件,打Y折]
    
    BuildMsg --> SetFields[设置所有字段]
    SetFields --> AddItem[添加到结果列表]
    
    AddItem --> Loop
    Loop -->|完成| End([返回结果])
    
    style GetLadder fill:#ffcc99
    style CalcReduce fill:#ff9999
```

### 3.4 满减优惠计算详细流程

```mermaid
flowchart TD
    Start([满减优惠计算]) --> CalcTotal[计算购物车总金额<br/>getCartItemAmount]
    
    CalcTotal --> GetReduction[获取满足条件的满减规则<br/>getProductFullReduction]
    
    GetReduction --> CheckReduction{满减规则存在?}
    
    CheckReduction -->|否| NoPromo[按无促销处理<br/>handleNoReduce]
    NoPromo --> End
    
    CheckReduction -->|是| Loop{遍历购物车项}
    
    Loop -->|每一项| GetSku[获取SKU原价<br/>skuStock.getPrice]
    
    GetSku --> CalcRatio[计算分摊比例<br/>ratio = originalPrice / totalAmount]
    
    CalcRatio --> CalcReduce[计算优惠金额<br/>reduceAmount = <br/>ratio × reducePrice]
    
    CalcReduce --> BuildMsg[构建促销消息<br/>满X元,减Y元]
    
    BuildMsg --> SetFields[设置所有字段]
    SetFields --> AddItem[添加到结果列表]
    
    AddItem --> Loop
    Loop -->|完成| End([返回结果])
    
    style CalcRatio fill:#ffcc99
    style CalcReduce fill:#ff9999
    style BuildMsg fill:#99ff99
```

---

## 四、数据流图（价格数据流转）

### 4.1 价格数据流转图

```mermaid
flowchart LR
    subgraph 数据库表
        T1[(pms_product<br/>原价+促销类型)]
        T2[(pms_sku_stock<br/>SKU价格)]
        T3[(pms_product_ladder<br/>阶梯规则)]
        T4[(pms_product_full_reduction<br/>满减规则)]
    end
    
    subgraph DAO层查询
        Query[4表LEFT JOIN<br/>PortalProductDao]
    end
    
    subgraph Domain对象
        VO1[PromotionProduct<br/>促销商品VO]
    end
    
    subgraph Service层计算
        Calc[促销价格计算<br/>OmsPromotionService]
    end
    
    subgraph 结果对象
        Result[CartPromotionItem<br/>购物车促销项]
    end
    
    subgraph 响应数据
        Response[JSON响应<br/>含价格+优惠信息]
    end
    
    T1 --> Query
    T2 --> Query
    T3 --> Query
    T4 --> Query
    
    Query --> VO1
    VO1 --> Calc
    
    Calc -->|单品促销| Result
    Calc -->|阶梯价格| Result
    Calc -->|满减优惠| Result
    Calc -->|无促销| Result
    
    Result --> Response
    
    style Query fill:#ffcc99
    style Calc fill:#ff9999
    style Result fill:#99ff99
```

### 4.2 价格计算数据转换图

```mermaid
flowchart TB
    subgraph 输入数据
        Input1[购物车商品<br/>OmsCartItem]
        Input2[商品ID列表<br/>List&lt;Long&gt;]
    end
    
    subgraph 查询结果
        Data1[商品基础信息<br/>PmsProduct]
        Data2[SKU信息<br/>PmsSkuStock]
        Data3[阶梯规则<br/>PmsProductLadder]
        Data4[满减规则<br/>PmsProductFullReduction]
    end
    
    subgraph 中间对象
        VO[PromotionProduct<br/>组装后的促销商品]
    end
    
    subgraph 计算过程
        Calc1[提取原价]
        Calc2[提取促销价/折扣]
        Calc3[计算优惠金额]
    end
    
    subgraph 输出结果
        Output[CartPromotionItem<br/>带促销信息的购物车项]
    end
    
    Input1 --> Input2
    Input2 --> Data1
    Input2 --> Data2
    Input2 --> Data3
    Input2 --> Data4
    
    Data1 --> VO
    Data2 --> VO
    Data3 --> VO
    Data4 --> VO
    
    VO --> Calc1
    VO --> Calc2
    Calc1 --> Calc3
    Calc2 --> Calc3
    
    Calc3 --> Output
    Input1 --> Output
    
    Output -->|字段| F1[productId<br/>productName<br/>quantity]
    Output -->|字段| F2[price: 原价<br/>reduceAmount: 优惠金额<br/>realPrice: 实付价]
    Output -->|字段| F3[promotionMessage: 促销描述<br/>integration: 积分<br/>growth: 成长值]
    
    style VO fill:#ffcc99
    style Calc3 fill:#ff9999
    style Output fill:#99ff99
```

---

## 五、类图（价格体系核心类关系）

### 5.1 价格计算核心类图

```mermaid
classDiagram
    class OmsCartItemController {
        +list() CommonResult
        +add(cartItem) CommonResult
        -cartItemService: OmsCartItemService
    }
    
    class OmsCartItemService {
        <<interface>>
        +listPromotion(memberId, cartIds) List~CartPromotionItem~
        +add(cartItem) int
        +list(memberId) List~OmsCartItem~
    }
    
    class OmsCartItemServiceImpl {
        -promotionService: OmsPromotionService
        -cartItemMapper: OmsCartItemMapper
        +listPromotion(memberId, cartIds) List~CartPromotionItem~
    }
    
    class OmsPromotionService {
        <<interface>>
        +calcCartPromotion(cartItemList) List~CartPromotionItem~
    }
    
    class OmsPromotionServiceImpl {
        -portalProductDao: PortalProductDao
        +calcCartPromotion(cartItemList) List~CartPromotionItem~
        -groupCartItemBySpu() Map
        -getPromotionProductList() List~PromotionProduct~
        -handleSinglePromotion() void
        -handleLadderPromotion() void
        -handleFullReduction() void
        -handleNoReduce() void
    }
    
    class PortalProductDao {
        <<interface>>
        +getPromotionProductList(ids) List~PromotionProduct~
        +getCartProduct(id) CartProduct
    }
    
    class OmsCartItem {
        +id: Long
        +memberId: Long
        +productId: Long
        +productSkuId: Long
        +quantity: Integer
        +price: BigDecimal
    }
    
    class CartPromotionItem {
        +promotionMessage: String
        +reduceAmount: BigDecimal
        +realStock: Integer
        +integration: Integer
        +growth: Integer
    }
    
    class PromotionProduct {
        +skuStockList: List~PmsSkuStock~
        +productLadderList: List~PmsProductLadder~
        +productFullReductionList: List~PmsProductFullReduction~
    }
    
    class PmsProduct {
        +id: Long
        +name: String
        +price: BigDecimal
        +promotionType: Integer
    }
    
    class PmsSkuStock {
        +id: Long
        +price: BigDecimal
        +promotionPrice: BigDecimal
        +stock: Integer
    }
    
    class PmsProductLadder {
        +id: Long
        +count: Integer
        +discount: BigDecimal
    }
    
    class PmsProductFullReduction {
        +id: Long
        +fullPrice: BigDecimal
        +reducePrice: BigDecimal
    }
    
    OmsCartItemController --> OmsCartItemService
    OmsCartItemService <|.. OmsCartItemServiceImpl
    OmsCartItemServiceImpl --> OmsPromotionService
    OmsPromotionService <|.. OmsPromotionServiceImpl
    OmsPromotionServiceImpl --> PortalProductDao
    
    OmsCartItemServiceImpl ..> OmsCartItem
    OmsPromotionServiceImpl ..> OmsCartItem
    OmsPromotionServiceImpl ..> CartPromotionItem
    OmsPromotionServiceImpl ..> PromotionProduct
    
    CartPromotionItem --|> OmsCartItem
    PromotionProduct --|> PmsProduct
    PromotionProduct --> PmsSkuStock
    PromotionProduct --> PmsProductLadder
    PromotionProduct --> PmsProductFullReduction
```

---

## 六、ER图（价格相关数据库表关系）

### 6.1 价格体系数据库ER图

```mermaid
erDiagram
    PMS_PRODUCT ||--o{ PMS_SKU_STOCK : has
    PMS_PRODUCT ||--o{ PMS_PRODUCT_LADDER : has
    PMS_PRODUCT ||--o{ PMS_PRODUCT_FULL_REDUCTION : has
    PMS_PRODUCT ||--o{ PMS_MEMBER_PRICE : has
    PMS_PRODUCT ||--o{ SMS_FLASH_PROMOTION_PRODUCT_RELATION : has
    SMS_FLASH_PROMOTION ||--o{ SMS_FLASH_PROMOTION_PRODUCT_RELATION : contains
    SMS_FLASH_PROMOTION_SESSION ||--o{ SMS_FLASH_PROMOTION_PRODUCT_RELATION : contains
    
    PMS_PRODUCT {
        bigint id PK
        varchar name
        decimal price "原价"
        int promotion_type "促销类型"
        int gift_point "赠送积分"
        int gift_growth "赠送成长值"
    }
    
    PMS_SKU_STOCK {
        bigint id PK
        bigint product_id FK
        varchar sku_code
        decimal price "SKU原价"
        decimal promotion_price "单品促销价⭐"
        int stock "库存"
        int lock_stock "锁定库存"
    }
    
    PMS_PRODUCT_LADDER {
        bigint id PK
        bigint product_id FK
        int count "阶梯数量⭐"
        decimal discount "折扣⭐"
    }
    
    PMS_PRODUCT_FULL_REDUCTION {
        bigint id PK
        bigint product_id FK
        decimal full_price "满减门槛⭐"
        decimal reduce_price "减免金额⭐"
    }
    
    PMS_MEMBER_PRICE {
        bigint id PK
        bigint product_id FK
        bigint member_level_id FK
        decimal member_price "会员价格⭐"
    }
    
    SMS_FLASH_PROMOTION {
        bigint id PK
        varchar title
        datetime start_date
        datetime end_date
        int status
    }
    
    SMS_FLASH_PROMOTION_SESSION {
        bigint id PK
        varchar name
        time start_time
        time end_time
    }
    
    SMS_FLASH_PROMOTION_PRODUCT_RELATION {
        bigint id PK
        bigint flash_promotion_id FK
        bigint flash_promotion_session_id FK
        bigint product_id FK
        decimal flash_promotion_price "秒杀价格⭐"
        int flash_promotion_count
        int flash_promotion_limit
    }
```

---

## 七、状态图（促销类型状态转换）

### 7.1 促销类型判断状态图

```mermaid
stateDiagram-v2
    [*] --> 查询商品
    查询商品 --> 获取促销类型
    
    获取促销类型 --> 判断促销类型
    
    判断促销类型 --> 单品促销: promotionType = 1
    判断促销类型 --> 阶梯价格: promotionType = 3
    判断促销类型 --> 满减优惠: promotionType = 4
    判断促销类型 --> 无促销: promotionType = 0
    
    单品促销 --> 计算中: 原价 - 促销价
    阶梯价格 --> 计算中: 原价 × 折扣
    满减优惠 --> 计算中: 按比例分摊
    无促销 --> 计算中: 优惠金额=0
    
    计算中 --> 设置结果
    设置结果 --> 添加到列表
    添加到列表 --> 是否还有商品
    
    是否还有商品 --> 查询商品: 有
    是否还有商品 --> [*]: 无,返回结果
```

---

## 八、关键逻辑说明

### 8.1 时序图说明

**时序图展示了完整的调用时间顺序:**
1. 从用户触发到前端请求
2. Controller层接收请求
3. Service层业务处理
4. DAO层数据库查询
5. 价格计算核心逻辑
6. 结果返回到前端

**关键点:**
- 采用LEFT JOIN 4表联查,一次获取所有促销信息
- 价格计算在内存中完成,避免多次数据库交互
- 支持4种促销类型的分支处理

### 8.2 架构图说明

**架构图展示了分层设计:**
- 表现层: 多个页面触发价格计算
- Controller层: 统一入口
- Service层: 核心业务逻辑(OmsPromotionService是关键)
- DAO层: 数据访问
- 数据库层: 7张价格相关表

**设计特点:**
- 清晰的分层架构
- 促销计算服务独立,便于复用
- 数据库表职责明确

### 8.3 流程图说明

**主流程图:**
- 展示了3步核心逻辑: 分组→查询→计算
- 清晰的促销类型分支判断
- 循环处理每个商品

**详细流程图:**
- 单品促销: 直接取促销价差值
- 阶梯价格: 先匹配阶梯规则,再计算折扣
- 满减优惠: 先计算总额,再按比例分摊

### 8.4 数据流图说明

**数据流转过程:**
```
原始数据(购物车) 
  → 数据库查询(4表JOIN) 
    → 中间对象(PromotionProduct) 
      → 价格计算(促销逻辑) 
        → 结果对象(CartPromotionItem) 
          → JSON响应
```

### 8.5 类图说明

**核心类职责:**
- `OmsPromotionServiceImpl`: 促销计算核心,包含4种促销算法
- `PortalProductDao`: 数据查询,4表联查
- `CartPromotionItem`: 结果封装,继承OmsCartItem并扩展促销字段
- `PromotionProduct`: 促销商品VO,聚合SKU、阶梯、满减信息

### 8.6 ER图说明

**表关系说明:**
- `pms_product`: 1对多关系连接所有价格相关表
- `pms_sku_stock`: 存储单品促销价
- `pms_product_ladder`: 存储阶梯价规则
- `pms_product_full_reduction`: 存储满减规则
- `sms_flash_promotion_product_relation`: 关联秒杀活动和商品

**关键字段:**
- ⭐标记的字段是价格计算的核心数据

---

## 九、性能分析图

### 9.1 性能瓶颈分析

```mermaid
flowchart TD
    subgraph 性能瓶颈点
        P1[🔴 4表LEFT JOIN<br/>查询复杂度高]
        P2[🔴 无缓存机制<br/>每次都查库]
        P3[🟡 内存计算<br/>商品多时开销大]
        P4[🟡 循环遍历<br/>时间复杂度O&#40;n&#41;]
    end
    
    subgraph 优化方案
        S1[✅ 添加Redis缓存<br/>缓存促销信息]
        S2[✅ SQL优化<br/>添加索引]
        S3[✅ 批量计算<br/>减少循环次数]
        S4[✅ 异步计算<br/>大批量时异步处理]
    end
    
    P1 --> S1
    P1 --> S2
    P2 --> S1
    P3 --> S3
    P4 --> S3
    P4 --> S4
    
    style P1 fill:#ff9999
    style P2 fill:#ff9999
    style P3 fill:#ffcc99
    style P4 fill:#ffcc99
    style S1 fill:#99ff99
    style S2 fill:#99ff99
    style S3 fill:#99ff99
    style S4 fill:#99ff99
```

---

## 十、总结

### 10.1 图表类型总结

| 图表类型 | 用途 | 关键信息 |
|---------|------|----------|
| **时序图** | 展示完整调用时序 | 从用户操作到数据库查询的完整流程 |
| **架构图** | 展示系统分层结构 | 5层架构,职责清晰 |
| **流程图** | 展示核心计算逻辑 | 4种促销类型的计算算法 |
| **数据流图** | 展示数据转换过程 | 从数据库到JSON响应的数据流转 |
| **类图** | 展示类关系 | 核心类的继承、组合、依赖关系 |
| **ER图** | 展示数据库表关系 | 7张表的1对多关系 |
| **状态图** | 展示状态转换 | 促销类型判断和计算状态 |

### 10.2 关键发现

通过图表分析发现:

1. **调用链路清晰**: Controller → Service → DAO → DB,典型的分层架构
2. **核心在Service层**: `OmsPromotionServiceImpl.calcCartPromotion()`是价格计算的核心
3. **数据库查询复杂**: 4表LEFT JOIN,是性能瓶颈点
4. **无缓存设计**: 每次都实时计算,保证准确性但影响性能
5. **算法清晰**: 4种促销类型的计算逻辑各自独立,便于维护

### 10.3 优化建议

基于图表分析的优化建议:

1. **添加Redis缓存**: 缓存促销信息,减少数据库查询
2. **SQL优化**: 为联表字段添加索引
3. **批量处理**: 减少循环次数
4. **异步计算**: 大批量订单时采用异步计算
5. **读写分离**: 查询走从库,减轻主库压力

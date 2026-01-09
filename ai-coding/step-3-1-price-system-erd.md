## mall 电商系统价格体系 - 数据库实体关系图

### 1. 实体概览

本图基于价格体系扫描结果，涵盖以下与价格/促销强相关的表：

- **pms_product**：商品主表，定义商品原价与促销类型。
- **pms_sku_stock**：SKU 库存与单品促销价。
- **pms_product_ladder**：按数量阶梯打折规则。
- **pms_product_full_reduction**：满减规则（满多少减多少）。
- **sms_flash_promotion**：秒杀活动主表。
- **sms_flash_promotion_product_relation**：秒杀活动与商品的关联及秒杀价。
- **pms_member_price**：不同会员等级的会员价。
- **oms_cart_item**：购物车条目，引用商品与 SKU。

### 2. ER 图（Mermaid）

```mermaid
erDiagram

  PMS_PRODUCT {
    bigint id PK "商品ID"
    varchar name "商品名称"
    decimal price "原价"
    int promotion_type "促销类型：0-无，1-单品，3-阶梯，4-满减"
    int gift_point "赠送积分"
    int gift_growth "赠送成长值"
  }

  PMS_SKU_STOCK {
    bigint id PK "SKU_ID"
    bigint product_id FK "商品ID"
    varchar sku_code "SKU编码"
    decimal price "SKU原价"
    decimal promotion_price "SKU促销价（单品促销）"
    int stock "库存"
    int lock_stock "锁定库存"
  }

  PMS_PRODUCT_LADDER {
    bigint id PK "主键"
    bigint product_id FK "商品ID"
    int count "满足数量（阶梯数量）"
    decimal discount "折扣（0.8 表示 8 折）"
  }

  PMS_PRODUCT_FULL_REDUCTION {
    bigint id PK "主键"
    bigint product_id FK "商品ID"
    decimal full_price "满减门槛（满XX元）"
    decimal reduce_price "减免金额（减XX元）"
  }

  SMS_FLASH_PROMOTION {
    bigint id PK "活动ID"
    varchar title "活动名称"
    datetime start_date "开始时间"
    datetime end_date "结束时间"
    int status "状态：0-下线，1-上线"
  }

  SMS_FLASH_PROMOTION_PRODUCT_RELATION {
    bigint id PK "主键"
    bigint flash_promotion_id FK "秒杀活动ID"
    bigint flash_promotion_session_id "场次ID"
    bigint product_id FK "商品ID"
    decimal flash_promotion_price "秒杀价格"
    int flash_promotion_count "秒杀数量"
    int flash_promotion_limit "限购数量"
  }

  PMS_MEMBER_PRICE {
    bigint id PK "主键"
    bigint product_id FK "商品ID"
    bigint member_level_id "会员等级ID"
    decimal member_price "会员价格"
  }

  OMS_CART_ITEM {
    bigint id PK "购物车项ID"
    bigint product_id FK "商品ID"
    bigint product_sku_id FK "SKU_ID"
    int quantity "购买数量"
    decimal price "下单时商品单价（快照）"
  }

  PMS_PRODUCT ||--o{ PMS_SKU_STOCK : "1 对多 SKU"
  PMS_PRODUCT ||--o{ PMS_PRODUCT_LADDER : "1 对多 阶梯价"
  PMS_PRODUCT ||--o{ PMS_PRODUCT_FULL_REDUCTION : "1 对多 满减规则"
  PMS_PRODUCT ||--o{ PMS_MEMBER_PRICE : "1 对多 会员价"
  PMS_PRODUCT ||--o{ SMS_FLASH_PROMOTION_PRODUCT_RELATION : "商品参与多个秒杀"

  SMS_FLASH_PROMOTION ||--o{ SMS_FLASH_PROMOTION_PRODUCT_RELATION : "活动关联多商品"

  PMS_SKU_STOCK ||--o{ OMS_CART_ITEM : "SKU 被购物车引用"
  PMS_PRODUCT ||--o{ OMS_CART_ITEM : "商品被购物车引用"
```

### 3. 价格相关关系小结

- **商品维度（pms_product）**：作为价格体系的核心实体，所有促销规则（阶梯价、满减、会员价、秒杀关联）都通过 `product_id` 与它关联。
- **SKU 维度（pms_sku_stock）**：在商品基础上细化到 SKU，支持同一商品不同规格不同价格/促销价。
- **活动维度**：
  - 阶梯价、满减、会员价属于“商品内置规则”，直接挂在商品上。
  - 秒杀活动通过 `sms_flash_promotion_product_relation` 将活动与商品关联，并提供独立的 `flash_promotion_price`。
- **购物车维度（oms_cart_item）**：通过 `product_id` 和 `product_sku_id` 连接到商品和 SKU，是价格计算链路的入口实体之一。

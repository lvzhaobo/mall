## 提示词

### 背景与目标
我是一名开发人员,刚接手这个 mall 电商项目,需要快速掌握项目架构和技术栈,为后续二次开发做准备。

### 扫描任务
请按以下步骤对项目进行全面扫描:

**第一步:项目整体扫描**(已扫描完成，在step-1-scan-project.md)


**第二步:模块详细扫描**(分别保存到独立 Markdown 文档)
扫描`mall-portal-detail.md`前台商城系统详细扫描，保存在当前文件中

### 输出要求
- 使用 Markdown 表格呈现结构化数据
- 保持内容简洁专业,便于快速查阅
- 突出关键信息,避免过度细节
- 生成的文档需可直接保存使用

## 扫描结果

### 一、模块基本信息

| 项目 | 内容 |
|------|------|
| **模块名称** | mall-portal |
| **功能定位** | 前台商城系统接口服务 |
| **端口** | 8085 |
| **打包方式** | jar (可运行) |
| **启动类** | MallPortalApplication.java |
| **版本** | 1.0-SNAPSHOT |

**职责边界**：
- ✅ 会员登录注册、信息管理
- ✅ 首页内容展示、商品推荐
- ✅ 商品浏览、搜索、详情
- ✅ 购物车管理
- ✅ 订单生成、支付、查询、取消
- ✅ 会员收藏、关注、浏览历史
- ✅ 优惠券领取与使用
- ✅ 支付宝支付集成

---

### 二、依赖关系

#### 2.1 核心依赖

| 依赖模块/组件 | 版本 | 用途 |
|--------------|------|------|
| **mall-mbg** | 1.0-SNAPSHOT | 数据层操作 |
| **mall-security** | 1.0-SNAPSHOT | JWT + Spring Security 认证 |
| **spring-boot-starter-data-mongodb** | 2.7.5 | 订单相关数据存储 |
| **spring-boot-starter-data-redis** | 2.7.5 | 缓存、验证码存储 |
| **spring-boot-starter-amqp** | 2.7.5 | RabbitMQ 消息队列 |
| **alipay-sdk-java** | 4.38.61.ALL | 支付宝支付 SDK |

#### 2.2 依赖链路径

```
mall-common (基础工具)
    ↓
mall-mbg (数据层) + mall-security (安全层)
    ↓
mall-portal (前台业务层)
    │
    ├─ MySQL (业务数据)
    ├─ Redis (缓存 + 验证码)
    ├─ MongoDB (订单详情 + 浏览历史)
    ├─ RabbitMQ (订单取消延迟消息)
    └─ Alipay SDK (支付集成)
```

---

### 三、包结构与分层架构

```
com.macro.mall.portal
├─ controller/        (控制层 - 13个控制器)
├─ service/           (服务层 - 15个服务接口)
│   └─ impl/          (15个服务实现类)
├─ dao/               (数据访问层 - 5个 DAO)
├─ domain/            (领域模型 - 19个实体类)
├─ repository/        (MongoDB 仓储 - 3个)
├─ component/         (组件 - 3个)
├─ config/            (配置 - 9个配置类)
└─ util/              (工具类 - 1个)
```

**分层说明**：
- **Controller**: RESTful API 接口层，处理 HTTP 请求
- **Service**: 业务逻辑层，处理核心业务
- **DAO**: 数据访问层，MyBatis 自定义查询
- **Repository**: MongoDB 数据访问层
- **Domain**: 领域对象、DTO、VO 定义
- **Component**: 公共组件（消息队列、定时任务）
- **Config**: Spring 配置类

---

### 四、核心类与接口清单

#### 4.1 Controller 层 (13个)

| 控制器 | 路径 | 主要功能 | 接口数 |
|---------|------|----------|--------|
| **UmsMemberController** | /sso | 会员注册、登录、信息获取、密码修改 | 6 |
| **HomeController** | /home | 首页内容、分类、推荐、专题 | 6 |
| **PmsPortalProductController** | /product | 商品详情、分类、搜索 | - |
| **PmsPortalBrandController** | /brand | 品牌列表、推荐品牌 | - |
| **OmsCartItemController** | /cart | 购物车增删改查、清空 | - |
| **OmsPortalOrderController** | /order | 订单生成、支付、取消、查询 | 10 |
| **OmsPortalOrderReturnApplyController** | /returnApply | 订单退货申请 | - |
| **UmsMemberReceiveAddressController** | /member/address | 收货地址管理 | - |
| **UmsMemberCouponController** | /member/coupon | 优惠券领取、查询 | - |
| **MemberAttentionController** | /member/attention | 会员关注品牌 | - |
| **MemberProductCollectionController** | /member/productCollection | 会员商品收藏 | - |
| **MemberReadHistoryController** | /member/readHistory | 浏览历史 | - |
| **AlipayController** | /alipay | 支付宝支付、回调 | - |

#### 4.2 Service 层 (15个)

| 服务接口 | 核心功能 |
|---------|----------|
| **UmsMemberService** | 会员注册、登录、获取信息、生成验证码、修改密码 |
| **UmsMemberCacheService** | 会员缓存操作 |
| **HomeService** | 首页内容加载、商品推荐、分类展示 |
| **PmsPortalProductService** | 商品详情查询、搜索 |
| **PmsPortalBrandService** | 品牌列表、详情 |
| **OmsCartItemService** | 购物车增删改查、促销价计算 |
| **OmsPortalOrderService** | 订单生成、支付、取消、超时处理 |
| **OmsPromotionService** | 促销信息计算 |
| **UmsMemberReceiveAddressService** | 收货地址管理 |
| **UmsMemberCouponService** | 优惠券领取、使用 |
| **MemberAttentionService** | 品牌关注管理 |
| **MemberCollectionService** | 商品收藏管理 |
| **MemberReadHistoryService** | 浏览历史管理 |
| **OmsPortalOrderReturnApplyService** | 退货申请 |
| **AlipayService** | 支付宝支付集成 |

#### 4.3 DAO 层 (5个)

| DAO | Mapper XML | 功能 |
|-----|------------|------|
| **HomeDao** | HomeDao.xml | 首页内容联表查询 |
| **PortalOrderDao** | PortalOrderDao.xml | 订单详情查询、超时订单查询 |
| **PortalOrderItemDao** | PortalOrderItemDao.xml | 订单明细插入 |
| **PortalProductDao** | PortalProductDao.xml | 商品详情查询 |
| **SmsCouponHistoryDao** | SmsCouponHistoryDao.xml | 优惠券使用历史查询 |

#### 4.4 MongoDB Repository (3个)

| Repository | 存储内容 |
|------------|----------|
| **MemberReadHistoryRepository** | 会员浏览历史 |
| **MemberProductCollectionRepository** | 会员商品收藏 |
| **MemberBrandAttentionRepository** | 会员品牌关注 |

#### 4.5 Component 组件 (3个)

| 组件 | 功能 |
|------|------|
| **CancelOrderSender** | 发送订单取消延迟消息 |
| **CancelOrderReceiver** | 接收并处理订单取消消息 |
| **OrderTimeOutCancelTask** | 定时扫描超时未支付订单 |

#### 4.6 Config 配置类 (9个)

| 配置类 | 功能 |
|--------|------|
| **AlipayConfig** | 支付宝支付参数配置 |
| **AlipayClientConfig** | 支付宝客户端初始化 |
| **RabbitMqConfig** | RabbitMQ 队列、交换机、绑定配置 |
| **MallSecurityConfig** | Spring Security 安全配置 |
| **GlobalCorsConfig** | 跨域请求配置 |
| **MyBatisConfig** | MyBatis 扫描配置 |
| **SwaggerConfig** | Swagger API 文档配置 |
| **JacksonConfig** | Jackson JSON 序列化配置 |
| **SpringTaskConfig** | Spring 定时任务配置 |

---

### 五、关键业务流程

#### 5.1 会员注册登录流程

```
1. 用户注册
   └─ 获取手机验证码 (Redis 存储, 90秒过期)
   └─ 验证码验证
   └─ 密码加密存储 (BCrypt)
   └─ 创建会员账号

2. 会员登录
   └─ 用户名/密码验证
   └─ 生成 JWT Token (7天有效期)
   └─ 返回 Token + TokenHead
   └─ 会员信息缓存至 Redis

3. Token 刷新
   └─ 检查 Token 有效性
   └─ 生成新 Token
```

#### 5.2 订单生成流程

```
1. 生成确认单
   └─ 根据购物车 ID 查询商品信息
   └─ 计算促销价格 (秒杀、优惠券、满减)
   └─ 查询可用优惠券
   └─ 查询收货地址
   └─ 查询会员积分

2. 提交订单
   └─ 锁库存检查
   └─ 创建订单主表 (oms_order)
   └─ 创建订单明细 (oms_order_item)
      └─ 生成 MongoDB 订单详情快照
   └─ 扣减库存
   └─ 删除购物车项
   └─ 发送延迟消息到 RabbitMQ (30分钟后取消)

3. 订单支付
   └─ 跳转支付宝支付页面
   └─ 支付宝回调 (notifyUrl)
   └─ 验证签名
   └─ 更新订单状态为已支付

4. 订单取消
   └─ 用户主动取消
   └─ 超时自动取消 (RabbitMQ 延迟消息)
   └─ 定时任务扫描取消
   └─ 恢复库存
   └─ 恢复优惠券
   └─ 恢复积分
```

#### 5.3 购物车流程

```
1. 添加商品到购物车
   └─ 检查商品库存
   └─ 检查是否已存在 (更新数量)
   └─ 创建购物车项

2. 查看购物车
   └─ 查询购物车列表
   └─ 计算促销价格
   └─ 展示总价

3. 修改数量/删除
   └─ 更新数据库
```

---

### 六、对外 API 接口列表

#### 6.1 会员相关 API (/sso)

| 接口 | 方法 | 路径 | 描述 |
|------|------|------|------|
| 会员注册 | POST | /sso/register | username, password, telephone, authCode |
| 会员登录 | POST | /sso/login | username, password |
| 获取会员信息 | GET | /sso/info | 需要 Token |
| 获取验证码 | GET | /sso/getAuthCode | telephone |
| 修改密码 | POST | /sso/updatePassword | telephone, password, authCode |
| 刷新 Token | GET | /sso/refreshToken | 需要 Token |

#### 6.2 首页相关 API (/home)

| 接口 | 方法 | 路径 | 描述 |
|------|------|------|------|
| 首页内容 | GET | /home/content | 轮播图、秒杀、新品、热门 |
| 推荐商品 | GET | /home/recommendProductList | pageSize, pageNum |
| 商品分类 | GET | /home/productCateList/{parentId} | - |
| 专题列表 | GET | /home/subjectList | cateId, pageSize, pageNum |
| 人气商品 | GET | /home/hotProductList | pageSize, pageNum |
| 新品推荐 | GET | /home/newProductList | pageSize, pageNum |

#### 6.3 订单相关 API (/order)

| 接口 | 方法 | 路径 | 描述 |
|------|------|------|------|
| 生成确认单 | POST | /order/generateConfirmOrder | cartIds |
| 生成订单 | POST | /order/generateOrder | OrderParam |
| 支付成功回调 | POST | /order/paySuccess | orderId, payType |
| 取消超时订单 | POST | /order/cancelTimeOutOrder | - |
| 取消订单 | POST | /order/cancelOrder | orderId |
| 订单列表 | GET | /order/list | status, pageNum, pageSize |
| 订单详情 | GET | /order/detail/{orderId} | - |
| 用户取消订单 | POST | /order/cancelUserOrder | orderId |
| 确认收货 | POST | /order/confirmReceiveOrder | orderId |
| 删除订单 | POST | /order/deleteOrder | orderId |

#### 6.4 其他 API

- **/cart**: 购物车管理 API
- **/product**: 商品详情、搜索 API
- **/brand**: 品牌相关 API
- **/member/address**: 收货地址 API
- **/member/coupon**: 优惠券 API
- **/member/attention**: 品牌关注 API
- **/member/productCollection**: 商品收藏 API
- **/member/readHistory**: 浏览历史 API
- **/alipay**: 支付宝支付 API

---

### 七、数据库表关联关系

#### 7.1 MySQL 表 (主要使用)

| 表名 | 用途 | 关联表 |
|------|------|--------|
| **ums_member** | 会员信息 | ums_member_level, ums_member_receive_address |
| **pms_product** | 商品信息 | pms_product_category, pms_brand, pms_sku_stock |
| **oms_order** | 订单主表 | oms_order_item, ums_member, ums_member_receive_address |
| **oms_order_item** | 订单明细 | oms_order, pms_product, pms_sku_stock |
| **oms_cart_item** | 购物车 | pms_product, pms_sku_stock, ums_member |
| **sms_coupon** | 优惠券 | sms_coupon_history, sms_coupon_product_relation |
| **sms_flash_promotion** | 秒杀活动 | sms_flash_promotion_session, sms_flash_promotion_product_relation |

#### 7.2 MongoDB 集合

| 集合名 | 用途 | 数据特点 |
|--------|------|----------|
| **memberReadHistory** | 会员浏览历史 | 时间序列数据, 高频写入 |
| **memberProductCollection** | 商品收藏 | 用户UGC数据 |
| **memberBrandAttention** | 品牌关注 | 用户UGC数据 |

**使用 MongoDB 原因**：
- ✅ 浏览历史写入频繁，降低 MySQL 负载
- ✅ 无需复杂查询，适合文档存储
- ✅ 数据结构灵活，方便扩展

#### 7.3 Redis 缓存 Key

| Key | TTL | 用途 |
|-----|-----|------|
| **ums:authCode:{telephone}** | 90秒 | 手机验证码 |
| **ums:member:{memberId}** | 24小时 | 会员信息缓存 |
| **oms:orderId** | - | 订单 ID 生成器 |

---

### 八、依赖的公共模块

#### 8.1 mall-common 模块

使用的公共组件：
- ✅ `CommonResult<T>` - 统一响应包装
- ✅ `CommonPage<T>` - 分页结果封装
- ✅ `IErrorCode` - 错误码接口
- ✅ `ApiException` - 自定义异常
- ✅ `GlobalExceptionHandler` - 全局异常处理
- ✅ `RedisService` - Redis 操作封装
- ✅ `WebLogAspect` - Web 日志切面

#### 8.2 mall-security 模块

使用的安全组件：
- ✅ `JwtAuthenticationTokenFilter` - JWT 认证过滤器
- ✅ `RestAuthenticationEntryPoint` - 未认证处理
- ✅ `RestfulAccessDeniedHandler` - 权限不足处理
- ✅ `JwtTokenUtil` - JWT Token 工具类

#### 8.3 mall-mbg 模块

使用的数据层组件：
- ✅ Mapper 接口 (62个)
- ✅ Model 实体类 (152个)
- ✅ Example 条件查询类

---

### 九、配置说明

#### 9.1 应用配置 (application.yml)

```yaml
应用名: mall-portal
端口: 8085
MyBatis Mapper: classpath:dao/*.xml

JWT 配置:
  tokenHeader: Authorization
  secret: mall-portal-secret
  expiration: 604800 (7天)
  tokenHead: Bearer

安全白名单:
  - Swagger 文档
  - 首页相关 (/home/**)
  - 商品相关 (/product/**, /brand/**)
  - 支付宝回调 (/alipay/**)
```

#### 9.2 开发环境配置 (application-dev.yml)

```yaml
MySQL:
  地址: localhost:3306
  数据库: mall
  用户名/密码: root/root
  Druid 连接池: 5-20

MongoDB:
  地址: localhost:27017
  数据库: mall-port

Redis:
  地址: localhost:6379
  数据库: 0

RabbitMQ:
  地址: localhost:5672
  虚拟主机: /mall
  用户名/密码: mall/mall

支付宝:
  网关: https://openapi-sandbox.dl.alipaydev.com/gateway.do
  appId: 需配置
  公钥私钥: 需配置
```

---

### 十、核心功能特点

#### 10.1 订单超时取消机制

**三种取消方式**：

1. **RabbitMQ 延迟消息** (主要方式)
   - 订单生成后发送 30 分钟延迟消息
   - 到期后消费者检查订单状态
   - 未支付则自动取消

2. **Spring Task 定时任务** (备用方式)
   - 每半小时扫描一次
   - 查找创建超过 30 分钟的未支付订单
   - 批量取消

3. **手动触发** (管理员操作)
   - 调用 `/order/cancelTimeOutOrder` 接口

#### 10.2 促销价格计算

**支持的促销类型**：
- ✅ 限时秒杀价
- ✅ 会员价格
- ✅ 优惠券折扣
- ✅ 满减促销
- ✅ 阶梯价格

**价格计算优先级**：
```
秒杀价 > 会员价 > 原价 + 优惠券 + 满减
```

#### 10.3 浏览历史管理

- 存储到 MongoDB，减少 MySQL 压力
- 每个会员最多保存 10 条浏览记录
- 支持按时间倒序查询

---

### 十一、开发注意事项

#### 11.1 环境依赖

**必需环境**：
- ✅ MySQL 5.7+ (业务数据)
- ✅ Redis 7.0 (缓存 + 验证码)
- ✅ MongoDB 5.0 (订单快照 + 浏览历史)
- ✅ RabbitMQ 3.10.5 (订单取消消息)

**可选环境**：
- 支付宝沙箱环境账号 (支付功能)

#### 11.2 数据初始化

```sql
-- 需要初始化的数据
- cms_subject (专题数据)
- pms_product (商品数据)
- pms_brand (品牌数据)
- sms_home_advertise (首页轮播图)
- sms_home_recommend_product (首页推荐商品)
- sms_flash_promotion (秒杀活动)
```

#### 11.3 启动步骤

```bash
# 1. 启动基础环境
mysql
redis-server
mongod
rabbitmq-server

# 2. 导入数据库
mall.sql

# 3. 启动应用
cd mall-portal
mvn spring-boot:run

# 4. 访问 Swagger 文档
http://localhost:8085/swagger-ui/
```

#### 11.4 常见问题

| 问题 | 解决方案 |
|------|----------|
| MongoDB 连接失败 | 检查 MongoDB 服务是否启动 |
| RabbitMQ 连接失败 | 检查虚拟主机 `/mall` 是否创建 |
| 订单不自动取消 | 检查 RabbitMQ 延迟消息插件 |
| 支付宝回调失败 | 配置内网穿透或使用公网域名 |

---

### 十二、API 文档访问

**Swagger 文档**：
- 本地: http://localhost:8085/swagger-ui/
- 分组: 会员、首页、商品、订单、购物车、支付

**Postman 集合**：
- 文件: `document/postman/mall-portal.postman_collection.json`

---

### 十三、后续优化建议

1. **性能优化**
   - 商品详情缓存至 Redis
   - 首页内容静态化
   - 订单查询增加索引

2. **功能增强**
   - 增加微信支付
   - 支持拼团功能
   - 增加积分兑换

3. **代码质量**
   - 增加单元测试
   - 增加接口文档注释
   - 提取公共方法
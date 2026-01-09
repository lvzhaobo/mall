# 测试规范

## 1. 测试分层

### 1.1 测试金字塔
```
       /\
      /  \    E2E测试（5%）  - UI自动化
     /____\   
    /      \  集成测试（15%） - API测试
   /________\ 
  /          \ 单元测试（80%） - 核心逻辑
 /____________\
```

### 1.2 测试目标
- **单元测试覆盖率**：核心业务逻辑 ≥ 80%
- **接口测试覆盖率**：所有对外API 100%
- **回归测试**：主流程自动化覆盖

## 2. 单元测试

### 2.1 测试框架
```xml
<!-- JUnit 5 + Mockito + AssertJ -->
<dependency>
    <groupId>org.springframework.boot</groupId>
    <artifactId>spring-boot-starter-test</artifactId>
    <scope>test</scope>
</dependency>
```

### 2.2 命名规范
```java
// 测试类命名：被测试类名 + Test
public class UserServiceTest {
    
    // 测试方法命名：test + 方法名 + 场景 + 预期结果
    @Test
    void testCreateUser_WithValidData_Success() {
        // ...
    }
    
    @Test
    void testCreateUser_WithDuplicateMobile_ThrowException() {
        // ...
    }
}
```

### 2.3 测试结构（AAA模式）
```java
@Test
void testCreateUser_WithValidData_Success() {
    // Arrange（准备）
    UserCreateDTO dto = new UserCreateDTO();
    dto.setUsername("zhangsan");
    dto.setMobile("13812345678");
    
    when(userMapper.selectByMobile(anyString())).thenReturn(null);
    when(userMapper.insert(any())).thenReturn(1);
    
    // Act（执行）
    Long userId = userService.createUser(dto);
    
    // Assert（断言）
    assertThat(userId).isNotNull();
    verify(userMapper, times(1)).insert(any());
}
```

### 2.4 Mock使用
```java
@SpringBootTest
class OrderServiceTest {
    
    @InjectMocks
    private OrderServiceImpl orderService;
    
    @Mock
    private OrderMapper orderMapper;
    
    @Mock
    private ProductService productService;
    
    @Test
    void testCreateOrder_WithInsufficientStock_ThrowException() {
        // Mock外部依赖
        when(productService.checkStock(1L, 10))
            .thenReturn(false);
        
        OrderCreateDTO dto = new OrderCreateDTO();
        dto.setProductId(1L);
        dto.setQuantity(10);
        
        // 断言异常
        assertThatThrownBy(() -> orderService.createOrder(dto))
            .isInstanceOf(BusinessException.class)
            .hasMessage("库存不足");
    }
}
```

## 3. 集成测试

### 3.1 Controller测试
```java
@SpringBootTest
@AutoConfigureMockMvc
class UserControllerTest {
    
    @Autowired
    private MockMvc mockMvc;
    
    @Test
    void testCreateUser_Success() throws Exception {
        String requestBody = """
            {
              "username": "zhangsan",
              "mobile": "13812345678",
              "email": "zhangsan@example.com"
            }
            """;
        
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(200))
                .andExpect(jsonPath("$.data").isNumber());
    }
    
    @Test
    void testCreateUser_WithInvalidMobile_ReturnError() throws Exception {
        String requestBody = """
            {
              "username": "zhangsan",
              "mobile": "123456"
            }
            """;
        
        mockMvc.perform(post("/api/v1/users")
                .contentType(MediaType.APPLICATION_JSON)
                .content(requestBody))
                .andExpect(status().isOk())
                .andExpect(jsonPath("$.code").value(40003))
                .andExpect(jsonPath("$.message").value("手机号格式不正确"));
    }
}
```

### 3.2 数据库测试
```java
@SpringBootTest
@Transactional // 测试后自动回滚
class UserMapperTest {
    
    @Autowired
    private UserMapper userMapper;
    
    @Test
    void testInsertUser_Success() {
        User user = new User();
        user.setUsername("testuser");
        user.setMobile("13812345678");
        
        int rows = userMapper.insert(user);
        
        assertThat(rows).isEqualTo(1);
        assertThat(user.getId()).isNotNull();
    }
    
    @Test
    void testSelectByMobile_Found() {
        // 准备测试数据
        User user = new User();
        user.setUsername("testuser");
        user.setMobile("13812345678");
        userMapper.insert(user);
        
        // 执行查询
        User found = userMapper.selectByMobile("13812345678");
        
        assertThat(found).isNotNull();
        assertThat(found.getUsername()).isEqualTo("testuser");
    }
}
```

## 4. 测试数据

### 4.1 测试数据准备
```java
@SpringBootTest
class OrderServiceTest {
    
    @Autowired
    private OrderService orderService;
    
    @BeforeEach
    void setUp() {
        // 每个测试前准备数据
        cleanTestData();
        prepareTestData();
    }
    
    @AfterEach
    void tearDown() {
        // 每个测试后清理数据
        cleanTestData();
    }
    
    private void prepareTestData() {
        // 插入测试用户
        User user = new User();
        user.setId(1L);
        user.setUsername("testuser");
        userMapper.insert(user);
        
        // 插入测试商品
        Product product = new Product();
        product.setId(1L);
        product.setTitle("测试商品");
        product.setStock(100);
        productMapper.insert(product);
    }
}
```

### 4.2 测试数据隔离
```java
// 使用独立的测试数据库
# application-test.yml
spring:
  datasource:
    url: jdbc:mysql://localhost:3306/mall_test?useUnicode=true
    username: root
    password: test123456
```

## 5. 测试场景

### 5.1 正常流程
```java
@Test
void testCreateOrder_WithValidData_Success() {
    // 测试正常下单流程
    OrderCreateDTO dto = buildValidOrderDTO();
    Long orderId = orderService.createOrder(dto);
    assertThat(orderId).isNotNull();
}
```

### 5.2 边界条件
```java
@Test
void testCreateOrder_WithMinQuantity_Success() {
    // 最小购买数量
    dto.setQuantity(1);
    Long orderId = orderService.createOrder(dto);
    assertThat(orderId).isNotNull();
}

@Test
void testCreateOrder_WithMaxQuantity_Success() {
    // 最大购买数量
    dto.setQuantity(999);
    Long orderId = orderService.createOrder(dto);
    assertThat(orderId).isNotNull();
}

@Test
void testCreateOrder_WithZeroQuantity_ThrowException() {
    // 非法数量
    dto.setQuantity(0);
    assertThatThrownBy(() -> orderService.createOrder(dto))
        .isInstanceOf(BusinessException.class);
}
```

### 5.3 异常场景
```java
@Test
void testCreateOrder_WithInsufficientStock_ThrowException() {
    // 库存不足
    when(productService.checkStock(anyLong(), anyInt()))
        .thenReturn(false);
    assertThatThrownBy(() -> orderService.createOrder(dto))
        .isInstanceOf(BusinessException.class)
        .hasMessage("库存不足");
}

@Test
void testCreateOrder_WithInsufficientBalance_ThrowException() {
    // 余额不足
    when(accountService.checkBalance(anyLong(), any()))
        .thenReturn(false);
    assertThatThrownBy(() -> orderService.createOrder(dto))
        .isInstanceOf(BusinessException.class)
        .hasMessage("余额不足");
}
```

### 5.4 并发场景
```java
@Test
void testCreateOrder_Concurrent_NoOversell() throws Exception {
    // 模拟100个并发请求
    int threadCount = 100;
    CountDownLatch latch = new CountDownLatch(threadCount);
    
    ExecutorService executor = Executors.newFixedThreadPool(threadCount);
    AtomicInteger successCount = new AtomicInteger(0);
    
    for (int i = 0; i < threadCount; i++) {
        executor.submit(() -> {
            try {
                orderService.createOrder(dto);
                successCount.incrementAndGet();
            } catch (Exception e) {
                // 预期部分请求失败（库存不足）
            } finally {
                latch.countDown();
            }
        });
    }
    
    latch.await();
    executor.shutdown();
    
    // 验证成功订单数不超过库存
    assertThat(successCount.get()).isLessThanOrEqualTo(100);
}
```

## 6. 性能测试

### 6.1 响应时间
```java
@Test
void testQueryOrders_ResponseTime() {
    long startTime = System.currentTimeMillis();
    
    List<Order> orders = orderService.listOrders(1, 20);
    
    long duration = System.currentTimeMillis() - startTime;
    assertThat(duration).isLessThan(1000); // 1秒内响应
}
```

### 6.2 压力测试
```bash
# JMeter压测脚本
# 测试场景：下单接口
# 并发用户：1000
# 持续时间：60秒
# 预期QPS：≥ 500
# 预期响应时间：P95 < 500ms
```

## 7. 测试报告

### 7.1 覆盖率报告
```xml
<!-- JaCoCo插件 -->
<plugin>
    <groupId>org.jacoco</groupId>
    <artifactId>jacoco-maven-plugin</artifactId>
    <version>0.8.10</version>
    <executions>
        <execution>
            <goals>
                <goal>prepare-agent</goal>
                <goal>report</goal>
            </goals>
        </execution>
    </executions>
</plugin>
```

```bash
# 生成覆盖率报告
mvn clean test jacoco:report

# 报告路径：target/site/jacoco/index.html
```

### 7.2 测试结果
- **通过率**：≥ 95%
- **覆盖率**：核心业务 ≥ 80%
- **执行时间**：单元测试 < 10分钟

## 8. 持续集成

### 8.1 CI流水线
```yaml
# .gitlab-ci.yml
test:
  stage: test
  script:
    - mvn clean test
    - mvn jacoco:report
  coverage: '/Total.*?([0-9]{1,3})%/'
  artifacts:
    reports:
      junit: target/surefire-reports/TEST-*.xml
      coverage_report:
        coverage_format: cobertura
        path: target/site/jacoco/jacoco.xml
```

### 8.2 质量门禁
- 测试通过率 < 95% → 阻止合并
- 覆盖率下降 > 5% → 阻止合并
- 新增代码覆盖率 < 70% → 阻止合并

## 9. 测试最佳实践

### 9.1 原则
- **FIRST原则**：
  - Fast（快速）：单元测试秒级完成
  - Independent（独立）：测试间无依赖
  - Repeatable（可重复）：结果稳定
  - Self-Validating（自我验证）：无需人工判断
  - Timely（及时）：与代码同步编写

### 9.2 建议
- ✅ 每个方法至少1个正常用例 + 1个异常用例
- ✅ 关键业务逻辑优先测试
- ✅ 使用有意义的断言消息
- ✅ 避免测试私有方法（测试公共接口）

## 10. 禁止事项

### 10.1 测试禁忌
- ❌ 禁止依赖测试执行顺序
- ❌ 禁止使用真实外部服务（Mock或测试环境）
- ❌ 禁止在测试中使用Thread.sleep()
- ❌ 禁止测试方法过长（> 50行拆分）
- ❌ 禁止跳过失败的测试（@Disabled需说明原因）

### 10.2 数据禁忌
- ❌ 禁止使用生产数据测试
- ❌ 禁止测试污染数据库（使用事务回滚）
- ❌ 禁止硬编码测试数据（使用工厂方法）

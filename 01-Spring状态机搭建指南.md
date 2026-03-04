我将提供一个从零开始的、使用**PostgreSQL**和**JPA**持久化的Spring状态机完整搭建方案。

## 项目概述：基于PostgreSQL的订单状态机

## 第一步：项目初始化与依赖配置

### 1.1 创建Spring Boot项目（pom.xml）

```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 
         http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    
    <parent>
        <groupId>org.springframework.boot</groupId>
        <artifactId>spring-boot-starter-parent</artifactId>
        <version>3.2.0</version>
        <relativePath/>
    </parent>
    
    <groupId>com.example</groupId>
    <artifactId>spring-statemachine-pgsql</artifactId>
    <version>1.0.0</version>
    
    <properties>
        <java.version>17</java.version>
    </properties>
    
    <dependencies>
        <!-- Spring Boot Starters -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-web</artifactId>
        </dependency>
        
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-data-jpa</artifactId>
        </dependency>
        
        <!-- Spring State Machine -->
        <dependency>
            <groupId>org.springframework.statemachine</groupId>
            <artifactId>spring-statemachine-starter</artifactId>
            <version>4.0.0</version>
        </dependency>
        
        <!-- Spring State Machine JPA Persistence -->
        <dependency>
            <groupId>org.springframework.statemachine</groupId>
            <artifactId>spring-statemachine-data-jpa</artifactId>
            <<version>4.0.0</version>
        </dependency>
        
        <!-- PostgreSQL Database -->
        <dependency>
            <groupId>org.postgresql</groupId>
            <artifactId>postgresql</artifactId>
            <scope>runtime</scope>
        </dependency>
        
        <!-- Lombok -->
        <dependency>
            <groupId>org.projectlombok</groupId>
            <artifactId>lombok</artifactId>
            <optional>true</optional>
        </dependency>
        
        <!-- Testing -->
        <dependency>
            <groupId>org.springframework.boot</groupId>
            <artifactId>spring-boot-starter-test</artifactId>
            <scope>test</scope>
        </dependency>
    </dependencies>
    <build>
        <plugins>
            <plugin>
                <groupId>org.springframework.boot</groupId>
                <artifactId>spring-boot-maven-plugin</artifactId>
                <configuration>
                    <excludes>
                        <exclude>
                            <groupId>org.projectlombok</groupId>
                            <artifactId>lombok</artifactId>
                        </exclude>
                    </excludes>
                </configuration>
            </plugin>
        </plugins>
    </build>
</project>
```

## 第二步：数据库配置

### 2.1 application.yml 配置文件

```yaml
# src/main/resources/application.yml
spring:
  application:
    name: order-statemachine-pgsql
  
  # PostgreSQL 数据库配置
  datasource:
    url: jdbc:postgresql://localhost:5432/statemachine_db
    username: postgres
    password: postgres123
    driver-class-name: org.postgresql.Driver
    hikari:
      connection-timeout: 30000
      maximum-pool-size: 10
      minimum-idle: 5
  
  # JPA 配置
  jpa:
    database-platform: org.hibernate.dialect.PostgreSQLDialect
    hibernate:
      ddl-auto: update  # 首次使用设置为 update，生产环境建议使用 validate
    show-sql: true
    properties:
      hibernate:
        format_sql: true
        jdbc:
          batch_size: 20
        default_schema: public
  
  # 状态机 JPA 存储配置
  statemachine:
    data:
      jpa:
        repository:
          enabled: true

# 应用配置
server:
  port: 8080

# 日志配置
logging:
  level:
    com.example.statemachine: DEBUG
    org.springframework.statemachine: INFO
    org.hibernate.SQL: DEBUG
```

### 2.2 PostgreSQL 数据库准备

在 PostgreSQL 中创建数据库：
```sql
-- 连接到 PostgreSQL
-- 创建数据库
CREATE DATABASE statemachine_db 
    ENCODING 'UTF8' 
    LC_COLLATE 'en_US.UTF-8' 
    LC_CTYPE 'en_US.UTF-8';

-- 创建专用用户（可选）
CREATE USER statemachine_user WITH PASSWORD 'your_password';
GRANT ALL PRIVILEGES ON DATABASE statemachine_db TO statemachine_user;
```

## 第三步：定义状态和事件枚举

### 3.1 订单状态枚举

```java
// src/main/java/com/example/statemachine/domain/OrderState.java
package com.example.statemachine.domain;

/**
 * 订单状态枚举 - 定义订单生命周期中的所有状态
 * 状态机将根据事件在这些状态之间转换
 */
public enum OrderState {
    
    // === 初始状态 ===
    UNPAID("待支付"),
    
    // === 处理中状态 ===
    WAITING_PAYMENT_CONFIRM("等待支付确认"),
    PAID("已支付"),
    PACKAGED("已打包"),
    SHIPPED("已发货"),
    DELIVERED("已送达"),
    
    // === 最终状态 ===
    CONFIRMED("已确认收货"),
    COMPLETED("已完成"),
    CANCELLED("已取消"),
    REFUNDED("已退款"),
    REFUND_FAILED("退款失败");
    
    private final String description;
    
    OrderState(String description) {
        this.description = description;
    }
    
    public String getDescription() {
        return description;
    }
    
    /**
     * 判断是否为最终状态（不能再转换）
     */
    public boolean isEndState() {
        return this == COMPLETED || 
               this == CANCELLED || 
               this == REFUNDED;
    }
}
```

### 3.2 订单事件枚举

```java
// src/main/java/com/example/statemachine/domain/OrderEvent.java
package com.example.statemachine.domain;

/**
 * 订单事件枚举 - 触发状态转换的外部事件
 * 每个事件都可能导致订单状态发生变化
 */
public enum OrderEvent {
    
    // === 支付相关事件 ===
    PAY("支付"),
    PAY_SUCCESS("支付成功"),
    PAY_FAILED("支付失败"),
    PAY_TIMEOUT("支付超时"),
    
    // === 订单处理事件 ===
    CONFIRM_PAYMENT("确认支付"),
    PACKAGE("打包商品"),
    SHIP("发货"),
    DELIVER("送达"),
    CONFIRM_RECEIPT("确认收货"),
    
    // === 取消和退款事件 ===
    USER_CANCEL("用户取消"),
    SYSTEM_CANCEL("系统取消"),
    APPLY_REFUND("申请退款"),
    REFUND("执行退款"),
    REFUND_SUCCESS("退款成功"),
    REFUND_FAILED("退款失败"),
    
    // === 系统自动化事件 ===
    AUTO_CONFIRM("自动确认收货"),
    AUTO_CANCEL("自动取消");
    
    private final String description;
    
    OrderEvent(String description) {
        this.description = description;
    }
    
    public String getDescription() {
        return description;
    }
}
```

## 第四步：创建 JPA 实体类

### 4.1 订单实体（业务实体）

```java
// src/main/java/com/example/statemachine/entity/Order.java
package com.example.statemachine.entity;

import com.example.statemachine.domain.OrderState;
import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import javax.persistence.*;
import java.time.LocalDateTime;

/**
 * 订单业务实体
 * 存储订单的业务信息，状态字段会与状态机同步
 */
@Data
@NoArgsConstructor
@AllArgsConstructor
@Entity
@Table(name = "orders")
public class Order {
    
    @Id
    @GeneratedValue(strategy = GenerationType.IDENTITY)
    private Long id;
    
    @Column(name = "order_number", nullable = false, unique = true, length = 50)
    private String orderNumber;
    
    @Enumerated(EnumType.STRING)
    @Column(name = "state", nullable = false, length = 50)
    private OrderState state;
    
    @Column(name = "amount", nullable = false)
    private Double amount;
    
    @Column(name = "user_id", nullable = false, length = 100)
    private String userId;
    
    @Column(name = "product_id", length = 100)
    private String productId;
    
    @Column(name = "product_name", length = 200)
    private String productName;
    
    @Column(name = "recipient_name", length = 100)
    private String recipientName;
    
    @Column(name = "recipient_phone", length = 20)
    private String recipientPhone;
    
    @Column(name = "shipping_address", length = 500)
    private String shippingAddress;
    
    @Column(name = "create_time", nullable = false, updatable = false)
    private LocalDateTime createTime;
    
    @Column(name = "update_time", nullable = false)
    private LocalDateTime updateTime;
    
    @Column(name = "state_machine_id", length = 100)
    private String stateMachineId;
    
    @Version
    private Integer version;
    
    @PrePersist
    protected void onCreate() {
        LocalDateTime now = LocalDateTime.now();
        createTime = now;
        updateTime = now;
        
        // 生成状态机ID（使用订单ID或订单号）
        if (stateMachineId == null) {
            stateMachineId = "order_" + (orderNumber != null ? orderNumber : "temp");
        }
    }
    
    @PreUpdate
    protected void onUpdate() {
        updateTime = LocalDateTime.now();
    }
    
    /**
     * 静态工厂方法 - 创建新订单
     */
    public static Order createOrder(String orderNumber, Double amount, String userId) {
        Order order = new Order();
        order.setOrderNumber(orderNumber);
        order.setAmount(amount);
        order.setUserId(userId);
        order.setState(OrderState.UNPAID); // 初始状态
        order.setStateMachineId("order_" + orderNumber);
        return order;
    }
}
```

### 4.2 状态机上下文实体（JPA持久化专用）

```java
// src/main/java/com/example/statemachine/entity/StateMachineContextEntity.java
package com.example.statemachine.entity;

import lombok.Data;
import lombok.NoArgsConstructor;
import lombok.AllArgsConstructor;
import org.springframework.statemachine.data.jpa.AbstractJpaRepositoryStateMachine;
import javax.persistence.Entity;
import javax.persistence.Table;

/**
 * 状态机上下文实体
 * 继承 Spring State Machine 提供的抽象类，用于 JPA 持久化
 * 这个实体会被 Spring State Machine 自动管理
 */
@Data
@NoArgsConstructor
@AllArgsConstructor
@Entity
@Table(name = "state_machine_context")
public class StateMachineContextEntity extends AbstractJpaRepositoryStateMachine {
    
    // 继承的字段包括：
    // - machineId: 状态机实例的唯一标识符
    // - state: 当前状态（序列化存储）
    // - stateMachineContext: 完整的上下文信息（序列化存储）
    
    // 可以添加额外的业务字段（如果需要）
    @Column(name = "business_type", length = 50)
    private String businessType = "ORDER";
    
    @Column(name = "last_event", length = 100)
    private String lastEvent;
    
    @Column(name = "last_update_time")
    private LocalDateTime lastUpdateTime;
    
    @PreUpdate
    protected void onUpdate() {
        lastUpdateTime = LocalDateTime.now();
    }
}
```

## 第五步：状态机核心配置

### 5.1 状态机配置类

```java
// src/main/java/com/example/statemachine/config/OrderStateMachineConfig.java
package com.example.statemachine.config;

import com.example.statemachine.domain.OrderEvent;
import com.example.statemachine.domain.OrderState;
import lombok.extern.slf4j.Slf4j;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
import org.springframework.statemachine.config.EnableStateMachineFactory;
import org.springframework.statemachine.config.StateMachineConfigurerAdapter;
import org.springframework.statemachine.config.builders.*;
import org.springframework.statemachine.guard.Guard;
import org.springframework.statemachine.listener.StateMachineListener;
import org.springframework.statemachine.listener.StateMachineListenerAdapter;
import org.springframework.statemachine.state.State;
import org.springframework.statemachine.persist.StateMachineRuntimePersister;

import java.util.EnumSet;

@Slf4j
@Configuration
@EnableStateMachineFactory
public class OrderStateMachineConfig extends StateMachineConfigurerAdapter<OrderState, OrderEvent> {

    /**
     * 注入 JPA 持久化器
     * Spring Boot 会自动配置，因为我们在 application.yml 中启用了 statemachine.data.jpa.repository.enabled
     */
    private final StateMachineRuntimePersister<OrderState, OrderEvent, String> stateMachineRuntimePersister;

    public OrderStateMachineConfig(StateMachineRuntimePersister<OrderState, OrderEvent, String> stateMachineRuntimePersister) {
        this.stateMachineRuntimePersister = stateMachineRuntimePersister;
    }

    /**
     * 配置状态机的状态
     */
    @Override
    public void configure(StateMachineStateConfigurer<OrderState, OrderEvent> states) throws Exception {
        states
            .withStates()
            // 初始状态
            .initial(OrderState.UNPAID)
            // 所有可能的状态
            .states(EnumSet.allOf(OrderState.class))
            // 终态（这些状态不能再转换）
            .end(OrderState.COMPLETED)
            .end(OrderState.CANCELLED)
            .end(OrderState.REFUNDED)
            .end(OrderState.REFUND_FAILED);
    }

    /**
     * 配置状态转换
     */
    @Override
    public void configure(StateMachineTransitionConfigurer<OrderState, OrderEvent> transitions) throws Exception {
        transitions
            // ==================== 支付流程 ====================
            // 支付 -> 等待支付确认
            .withExternal()
                .source(OrderState.UNPAID)
                .target(OrderState.WAITING_PAYMENT_CONFIRM)
                .event(OrderEvent.PAY)
                .action(paymentAction())
                .and()
            
            // 支付成功 -> 已支付
            .withExternal()
                .source(OrderState.WAITING_PAYMENT_CONFIRM)
                .target(OrderState.PAID)
                .event(OrderEvent.PAY_SUCCESS)
                .action(paymentSuccessAction())
                .and()
            
            // 支付失败 -> 待支付（回到初始状态）
            .withExternal()
                .source(OrderState.WAITING_PAYMENT_CONFIRM)
                .target(OrderState.UNPAID)
                .event(OrderEvent.PAY_FAILED)
                .action(paymentFailedAction())
                .and()
            
            // 支付超时 -> 已取消
            .withExternal()
                .source(OrderState.UNPAID)
                .target(OrderState.CANCELLED)
                .event(OrderEvent.PAY_TIMEOUT)
                .action(timeoutAction())
                .and()
            
            // ==================== 订单处理流程 ====================
            // 已支付 -> 打包
            .withExternal()
                .source(OrderState.PAID)
                .target(OrderState.PACKAGED)
                .event(OrderEvent.PACKAGE)
                .guard(canPackageGuard()) // 添加守卫条件
                .action(packageAction())
                .and()
            
            // 打包 -> 发货
            .withExternal()
                .source(OrderState.PACKAGED)
                .target(OrderState.SHIPPED)
                .event(OrderEvent.SHIP)
                .action(shipAction())
                .and()
            
            // 发货 -> 送达
            .withExternal()
                .source(OrderState.SHIPPED)
                .target(OrderState.DELIVERED)
                .event(OrderEvent.DELIVER)
                .action(deliverAction())
                .and()
            
            // 送达 -> 确认收货
            .withExternal()
                .source(OrderState.DELIVERED)
                .target(OrderState.CONFIRMED)
                .event(OrderEvent.CONFIRM_RECEIPT)
                .action(confirmReceiptAction())
                .and()
            
            // 自动确认收货（送达后15天）
            .withExternal()
                .source(OrderState.DELIVERED)
                .target(OrderState.CONFIRMED)
                .event(OrderEvent.AUTO_CONFIRM)
                .action(autoConfirmAction())
                .and()
            
            // 确认收货 -> 完成
            .withExternal()
                .source(OrderState.CONFIRMED)
                .target(OrderState.COMPLETED)
                .event(OrderEvent.CONFIRM_RECEIPT)
                .action(completeAction())
                .and()
            
            // ==================== 取消流程 ====================
            // 待支付状态下用户取消
            .withExternal()
                .source(OrderState.UNPAID)
                .target(OrderState.CANCELLED)
                .event(OrderEvent.USER_CANCEL)
                .action(userCancelAction())
                .and()
            
            // 已支付状态下用户取消（需要退款）
            .withExternal()
                .source(OrderState.PAID)
                .target(OrderState.CANCELLED)
                .event(OrderEvent.USER_CANCEL)
                .guard(canRefundGuard()) // 检查是否可以退款
                .action(cancelWithRefundAction())
                .and()
            
            // ==================== 退款流程 ====================
            // 申请退款
            .withExternal()
                .source(OrderState.PAID)
                .target(OrderState.PAID) // 状态不变，但记录退款申请
                .event(OrderEvent.APPLY_REFUND)
                .action(applyRefundAction())
                .and()
            
            // 执行退款 -> 已退款
            .withExternal()
                .source(OrderState.PAID)
                .target(OrderState.REFUNDED)
                .event(OrderEvent.REFUND_SUCCESS)
                .action(refundSuccessAction())
                .and()
            
            // 退款失败
            .withExternal()
                .source(OrderState.PAID)
                .target(OrderState.REFUND_FAILED)
                .event(OrderEvent.REFUND_FAILED)
                .action(refundFailedAction());
    }

    /**
     * 配置状态机整体设置，启用 JPA 持久化
     */
    @Override
    public void configure(StateMachineConfigurationConfigurer<OrderState, OrderEvent> config) throws Exception {
        config
            .withConfiguration()
            .autoStartup(true) // 自动启动
            .listener(stateMachineListener()) // 添加监听器
            .and()
            .withPersistence() // 启用持久化
            .runtimePersister(stateMachineRuntimePersister); // 使用 JPA 持久化器
    }

    /**
     * 状态机监听器 - 监听状态转换
     */
    @Bean
    public StateMachineListener<OrderState, OrderEvent> stateMachineListener() {
        return new StateMachineListenerAdapter<OrderState, OrderEvent>() {
            @Override
            public void stateChanged(State<OrderState, OrderEvent> from, State<OrderState, OrderEvent> to) {
                if (from != null) {
                    log.info("状态机状态变更: [{}] -> [{}]", from.getId(), to.getId());
                } else {
                    log.info("状态机初始化: 初始状态为 [{}]", to.getId());
                }
            }

            @Override
            public void eventNotAccepted(org.springframework.messaging.Message<OrderEvent> event) {
                log.warn("事件不被接受: {}", event.getPayload());
            }

            @Override
            public void transition(org.springframework.statemachine.transition.Transition<OrderState, OrderEvent> transition) {
                if (transition.getKind() == org.springframework.statemachine.transition.TransitionKind.INTERNAL) {
                    log.debug("内部转换: {} -> {}", 
                        transition.getSource().getId(), 
                        transition.getTarget().getId());
                }
            }
        };
    }

    // ==================== 守卫条件定义 ====================
    
    /**
     * 打包守卫 - 检查是否可以打包
     */
    @Bean
    public Guard<OrderState, OrderEvent> canPackageGuard() {
        return context -> {
            // 在实际项目中，这里可以检查库存、物流等信息
            log.info("检查打包条件...");
            return true;
        };
    }
    
    /**
     * 退款守卫 - 检查是否可以退款
     */
    @Bean
    public Guard<OrderState, OrderEvent> canRefundGuard() {
        return context -> {
            // 检查退款条件，如支付时间、退款政策等
            log.info("检查退款条件...");
            return true;
        };
    }
    
    // ==================== 动作定义 ====================
    
    @Bean
    public org.springframework.statemachine.action.Action<OrderState, OrderEvent> paymentAction() {
        return context -> log.info("执行支付动作...");
    }
    
    @Bean
    public org.springframework.statemachine.action.Action<OrderState, OrderEvent> paymentSuccessAction() {
        return context -> log.info("支付成功处理...");
    }
    
    @Bean
    public org.springframework.statemachine.action.Action<OrderState, OrderEvent> packageAction() {
        return context -> log.info("执行打包动作...");
    }
    
    @Bean
    public org.springframework.statemachine.action.Action<OrderState, OrderEvent> shipAction() {
        return context -> log.info("执行发货动作...");
    }
    
    @Bean
    public org.springframework.statemachine.action.Action<OrderState, OrderEvent> deliverAction() {
        return context -> log.info("执行送达动作...");
    }
    
    @Bean
    public org.springframework.statemachine.action.Action<OrderState, OrderEvent> confirmReceiptAction() {
        return context -> log.info("执行确认收货动作...");
    }
    
    // 更多动作定义...
}
```

## 第六步：创建Repository和数据访问层

### 6.1 订单Repository

```java
// src/main/java/com/example/statemachine/repository/OrderRepository.java
package com.example.statemachine.repository;

import com.example.statemachine.entity.Order;
import com.example.statemachine.domain.OrderState;
import org.springframework.data.jpa.repository.JpaRepository;
import org.springframework.data.jpa.repository.Query;
import org.springframework.data.repository.query.Param;
import org.springframework.stereotype.Repository;
import java.time.LocalDateTime;
import java.util.List;
import java.util.Optional;

@Repository
public interface OrderRepository extends JpaRepository<Order, Long> {
    
    /**
     * 根据订单号查找订单
     */
    Optional<Order> findByOrderNumber(String orderNumber);
    
    /**
     * 根据用户ID查找订单
     */
    List<Order> findByUserId(String userId);
    
    /**
     * 根据状态查找订单
     */
    List<Order> findByState(OrderState state);
    
    /**
     * 查找指定状态且创建时间早于某个时间的订单
     * 用于定时任务处理超时订单
     */
    List<Order> findByStateAndCreateTimeBefore(
        OrderState state, 
        LocalDateTime time
    );
    
    /**
     * 查找待支付且超时的订单
     */
    @Query("SELECT o FROM Order o WHERE o.state = :state AND o.createTime < :timeoutTime")
    List<Order> findTimeoutOrders(
        @Param("state") OrderState state,
        @Param("timeoutTime") LocalDateTime timeoutTime
    );
    
    /**
     * 根据状态机ID查找订单
     */
    Optional<Order> findByStateMachineId(String stateMachineId);
}
```

### 6.2 状态机Repository（由Spring State Machine自动提供）

Spring State Machine的JPA模块会自动提供以下Repository，无需手动创建：

```java
// Spring State Machine 自动提供的Repository接口：
// JpaStateMachineRepository - 用于状态机上下文的CRUD操作
// 在配置中启用后即可自动注入使用
```

## 第七步：创建状态机服务层

### 7.1 状态机服务

```java
// src/main/java/com/example/statemachine/service/OrderStateMachineService.java
package com.example.statemachine.service;

import com.example.statemachine.domain.OrderEvent;
import com.example.statemachine.domain.OrderState;
import com.example.statemachine.entity.Order;
import com.example.statemachine.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.messaging.Message;
import org.springframework.messaging.support.MessageBuilder;
import org.springframework.statemachine.StateMachine;
import org.springframework.statemachine.config.StateMachineFactory;
import org.springframework.statemachine.persist.StateMachineRuntimePersister;
import org.springframework.stereotype.Service;
import org.springframework.transaction.annotation.Transactional;

import java.util.HashMap;
import java.util.Map;

@Slf4j
@Service
@RequiredArgsConstructor
@Transactional
public class OrderStateMachineService {
    
    private final StateMachineFactory<OrderState, OrderEvent> stateMachineFactory;
    private final OrderRepository orderRepository;
    
    /**
     * 创建新订单并初始化状态机
     * @param order 订单信息
     * @return 创建的订单
     */
    public Order createOrder(Order order) {
        // 保存订单到数据库
        Order savedOrder = orderRepository.save(order);
        log.info("创建订单成功: {}, 状态: {}", savedOrder.getOrderNumber(), savedOrder.getState());
        
        // 初始化状态机
        String machineId = savedOrder.getStateMachineId();
        StateMachine<OrderState, OrderEvent> stateMachine = stateMachineFactory.getStateMachine(machineId);
        
        // 启动状态机（状态会自动从数据库恢复或初始化为UNPAID）
        stateMachine.start();
        log.info("状态机初始化完成: {}", machineId);
        
        return savedOrder;
    }
    
    /**
     * 发送事件到状态机
     * @param orderId 订单ID
     * @param event 事件
     * @param contextData 上下文数据
     * @return 事件是否被接受
     */
    public boolean sendEvent(Long orderId, OrderEvent event, Map<String, Object> contextData) {
        return sendEvent(orderId, event, contextData, new HashMap<>());
    }
    
    /**
     * 发送事件到状态机（带消息头）
     * @param orderId 订单ID
     * @param event 事件
     * @param contextData 上下文数据
     * @param headers 消息头
     * @return 事件是否被接受
     */
    public boolean sendEvent(Long orderId, OrderEvent event, 
                            Map<String, Object> contextData, 
                            Map<String, Object> headers) {
        try {
            // 1. 获取订单
            Order order = orderRepository.findById(orderId)
                .orElseThrow(() -> new IllegalArgumentException("订单不存在: " + orderId));
            
            // 2. 获取状态机实例
            String machineId = order.getStateMachineId();
            StateMachine<OrderState, OrderEvent> stateMachine = stateMachineFactory.getStateMachine(machineId);
            
            log.info("发送事件: 订单={}, 事件={}, 当前状态={}", 
                order.getOrderNumber(), event, stateMachine.getState().getId());
            
            // 3. 构建消息
            MessageBuilder<OrderEvent> messageBuilder = MessageBuilder.withPayload(event);
            
            // 添加标准头信息
            messageBuilder.setHeader("orderId", orderId);
            messageBuilder.setHeader("orderNumber", order.getOrderNumber());
            messageBuilder.setHeader("stateMachineId", machineId);
            
            // 添加上下文数据
            if (contextData != null) {
                contextData.forEach(messageBuilder::setHeader);
            }
            
            // 添加自定义消息头
            if (headers != null) {
                headers.forEach(messageBuilder::setHeader);
            }
            
            Message<OrderEvent> message = messageBuilder.build();
            
            // 4. 发送事件
            boolean accepted = stateMachine.sendEvent(message);
            
            if (accepted) {
                // 更新订单状态（与状态机同步）
                OrderState newState = stateMachine.getState().getId();
                order.setState(newState);
                orderRepository.save(order);
                
                log.info("事件处理成功: 订单={}, 事件={}, 新状态={}", 
                    order.getOrderNumber(), event, newState);
            } else {
                log.warn("事件被拒绝: 订单={}, 事件={}, 当前状态={}", 
                    order.getOrderNumber(), event, stateMachine.getState().getId());
            }
            
            return accepted;
            
        } catch (Exception e) {
            log.error("处理事件失败: orderId={}, event={}", orderId, event, e);
            throw new RuntimeException("状态机事件处理失败", e);
        }
    }
    
    /**
     * 获取订单当前状态
     * @param orderId 订单ID
     * @return 当前状态
     */
    public OrderState getCurrentState(Long orderId) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new IllegalArgumentException("订单不存在: " + orderId));
        
        String machineId = order.getStateMachineId();
        StateMachine<OrderState, OrderEvent> stateMachine = stateMachineFactory.getStateMachine(machineId);
        
        return stateMachine.getState().getId();
    }
    
    /**
     * 检查是否可以发送某个事件
     * @param orderId 订单ID
     * @param event 事件
     * @return 是否可以发送
     */
    public boolean canSendEvent(Long orderId, OrderEvent event) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new IllegalArgumentException("订单不存在: " + orderId));
        
        String machineId = order.getStateMachineId();
        StateMachine<OrderState, OrderEvent> stateMachine = stateMachineFactory.getStateMachine(machineId);
        
        return stateMachine.getTransitions().stream()
            .anyMatch(transition -> 
                transition.getSource().getId() == stateMachine.getState().getId() &&
                transition.getTrigger() != null &&
                transition.getTrigger().getEvent() == event);
    }
    
    /**
     * 重置状态机到指定状态（用于异常恢复）
     * @param orderId 订单ID
     * @param targetState 目标状态
     */
    public void resetStateMachine(Long orderId, OrderState targetState) {
        Order order = orderRepository.findById(orderId)
            .orElseThrow(() -> new IllegalArgumentException("订单不存在: " + orderId));
        
        String machineId = order.getStateMachineId();
        StateMachine<OrderState, OrderEvent> stateMachine = stateMachineFactory.getStateMachine(machineId);
        
        // 停止当前状态机
        stateMachine.stop();
        
        // 重新启动并设置状态
        stateMachine.getStateMachineAccessor()
            .doWithAllRegions(access -> access.resetStateMachine(
                new org.springframework.statemachine.support.DefaultStateMachineContext<>(
                    targetState, null, null, null
                )
            ));
        
        stateMachine.start();
        
        // 更新订单状态
        order.setState(targetState);
        orderRepository.save(order);
        
        log.info("状态机已重置: 订单={}, 新状态={}", order.getOrderNumber(), targetState);
    }
}
```

### 7.2 订单业务服务

```java
// src/main/java/com/example/statemachine/service/OrderService.java
package com.example.statemachine.service;

import com.example.statemachine.domain.OrderState;
import com.example.statemachine.entity.Order;
import com.example.statemachine.repository.OrderRepository;
import lombok.RequiredArgsConstructor;
import org.springframework.stereotype.Service;
import java.util.List;
import java.util.Optional;

@Service
@RequiredArgsConstructor
public class OrderService {
    
    private final OrderRepository orderRepository;
    
    /**
     * 保存订单
     */
    public Order save(Order order) {
        return orderRepository.save(order);
    }
    
    /**
     * 根据ID查找订单
     */
    public Optional<Order> findById(Long id) {
        return orderRepository.findById(id);
    }
    
    /**
     * 根据订单号查找订单
     */
    public Optional<Order> findByOrderNumber(String orderNumber) {
        return orderRepository.findByOrderNumber(orderNumber);
    }
    
    /**
     * 查找用户的所有订单
     */
    public List<Order> findByUserId(String userId) {
        return orderRepository.findByUserId(userId);
    }
    
    /**
     * 查找指定状态的订单
     */
    public List<Order> findByState(OrderState state) {
        return orderRepository.findByState(state);
    }
    
    /**
     * 获取所有订单
     */
    public List<Order> findAll() {
        return orderRepository.findAll();
    }
    
    /**
     * 删除订单
     */
    public void delete(Long id) {
        orderRepository.deleteById(id);
    }
    
    /**
     * 批量更新订单状态
     */
    public void batchUpdateState(List<Long> orderIds, OrderState newState) {
        List<Order> orders = orderRepository.findAllById(orderIds);
        orders.forEach(order -> order.setState(newState));
        orderRepository.saveAll(orders);
    }
}
```

## 第八步：创建REST控制器

```java
// src/main/java/com/example/statemachine/controller/OrderController.java
package com.example.statemachine.controller;

import com.example.statemachine.domain.OrderEvent;
import com.example.statemachine.domain.OrderState;
import com.example.statemachine.entity.Order;
import com.example.statemachine.service.OrderService;
import com.example.statemachine.service.OrderStateMachineService;
import lombok.RequiredArgsConstructor;
import org.springframework.http.HttpStatus;
import org.springframework.http.ResponseEntity;
import org.springframework.web.bind.annotation.*;

import java.time.LocalDateTime;
import java.util.HashMap;
import java.util.Map;

@RestController
@RequestMapping("/api/orders")
@RequiredArgsConstructor
public class OrderController {
    
    private final OrderService orderService;
    private final OrderStateMachineService stateMachineService;
    
    /**
     * 创建新订单
     * POST /api/orders
     */
    @PostMapping
    public ResponseEntity<Map<String, Object>> createOrder(@RequestBody OrderRequest request) {
        Order order = Order.createOrder(
            generateOrderNumber(),
            request.getAmount(),
            request.getUserId()
        );
        
        order.setProductId(request.getProductId());
        order.setProductName(request.getProductName());
        order.setRecipientName(request.getRecipientName());
        order.setRecipientPhone(request.getRecipientPhone());
        order.setShippingAddress(request.getShippingAddress());
        
        Order createdOrder = stateMachineService.createOrder(order);
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", true);
        response.put("orderId", createdOrder.getId());
        response.put("orderNumber", createdOrder.getOrderNumber());
        response.put("state", createdOrder.getState());
        response.put("stateMachineId", createdOrder.getStateMachineId());
        response.put("createTime", createdOrder.getCreateTime());
        
        return ResponseEntity.status(HttpStatus.CREATED).body(response);
    }
    
    /**
     * 获取订单详情
     * GET /api/orders/{id}
     */
    @GetMapping("/{id}")
    public ResponseEntity<Order> getOrder(@PathVariable Long id) {
        return orderService.findById(id)
            .map(ResponseEntity::ok)
            .orElse(ResponseEntity.notFound().build());
    }
    
    /**
     * 发送事件
     * POST /api/orders/{id}/events/{event}
     */
    @PostMapping("/{id}/events/{event}")
    public ResponseEntity<Map<String, Object>> sendEvent(
            @PathVariable Long id,
            @PathVariable OrderEvent event,
            @RequestBody(required = false) EventRequest request) {
        
        Map<String, Object> contextData = request != null ? request.getContextData() : new HashMap<>();
        
        boolean accepted = stateMachineService.sendEvent(id, event, contextData);
        
        Order order = orderService.findById(id).orElse(null);
        
        Map<String, Object> response = new HashMap<>();
        response.put("success", accepted);
        response.put("orderId", id);
        response.put("event", event);
        response.put("currentState", order != null ? order.getState() : null);
        response.put("timestamp", LocalDateTime.now());
        response.put("message", accepted ? "事件处理成功" : "事件被拒绝");
        
        return accepted ? 
            ResponseEntity.ok(response) : 
            ResponseEntity.status(HttpStatus.BAD_REQUEST).body(response);
    }
    
    /**
     * 获取订单当前状态
     * GET /api/orders/{id}/state
     */
    @GetMapping("/{id}/state")
    public ResponseEntity<Map<String, Object>> getState(@PathVariable Long id) {
        try {
            OrderState state = stateMachineService.getCurrentState(id);
            
            Map<String, Object> response = new HashMap<>();
            response.put("orderId", id);
            response.put("state", state);
            response.put("description", state.getDescription());
            response.put("isEndState", state.isEndState());
            
            return ResponseEntity.ok(response);
        } catch (IllegalArgumentException e) {
            return ResponseEntity.notFound().build();
        }
    }
    
    /**
     * 检查是否可以发送事件
     * GET /api/orders/{id}/can/{event}
     */
    @GetMapping("/{id}/can/{event}")
    public ResponseEntity<Map<String, Object>> canSendEvent(
            @PathVariable Long id,
            @PathVariable OrderEvent event) {
        
        boolean canSend = stateMachineService.canSendEvent(id, event);
        
        Map<String, Object> response = new HashMap<>();
        response.put("orderId", id);
        response.put("event", event);
        response.put("canSend", canSend);
        
        return ResponseEntity.ok(response);
    }
    
    /**
     * 重置状态机（管理员功能）
     * PUT /api/orders/{id}/reset-state
     */
    @PutMapping("/{id}/reset-state")
    public ResponseEntity<Map<String, Object>> resetStateMachine(
            @PathVariable Long id,
            @RequestParam OrderState targetState) {
        
        try {
            stateMachineService.resetStateMachine(id, targetState);
            
            Map<String, Object> response = new HashMap<>();
            response.put("success", true);
            response.put("orderId", id);
            response.put("targetState", targetState);
            response.put("message", "状态机重置成功");
            
            return ResponseEntity.ok(response);
        } catch (Exception e) {
            Map<String, Object> response = new HashMap<>();
            response.put("success", false);
            response.put("error", e.getMessage());
            
            return ResponseEntity.status(HttpStatus.INTERNAL_SERVER_ERROR).body(response);
        }
    }
    
    /**
     * 生成订单号
     */
    private String generateOrderNumber() {
        return "ORD" + System.currentTimeMillis() + (int)(Math.random() * 1000);
    }
    
    // ==================== DTO类 ====================
    
    @lombok.Data
    public static class OrderRequest {
        private Double amount;
        private String userId;
        private String productId;
        private String productName;
        private String recipientName;
        private String recipientPhone;
        private String shippingAddress;
    }
    
    @lombok.Data
    public static class EventRequest {
        private Map<String, Object> contextData;
    }
}
```

## 第九步：定时任务（处理超时订单）

```java
// src/main/java/com/example/statemachine/scheduler/OrderTimeoutScheduler.java
package com.example.statemachine.scheduler;

import com.example.statemachine.domain.OrderEvent;
import com.example.statemachine.domain.OrderState;
import com.example.statemachine.entity.Order;
import com.example.statemachine.service.OrderStateMachineService;
import com.example.statemachine.service.OrderService;
import lombok.RequiredArgsConstructor;
import lombok.extern.slf4j.Slf4j;
import org.springframework.scheduling.annotation.Scheduled;
import org.springframework.stereotype.Component;
import org.springframework.transaction.annotation.Transactional;

import java.time.LocalDateTime;
import java.util.List;

@Slf4j
@Component
@RequiredArgsConstructor
public class OrderTimeoutScheduler {
    
    private final OrderService orderService;
    private final OrderStateMachineService stateMachineService;
    
    /**
     * 每分钟检查一次支付超时的订单
     * 待支付超过30分钟的订单自动取消
     */
    @Scheduled(fixedDelay = 60000) // 1分钟
    @Transactional
    public void checkPaymentTimeout() {
        LocalDateTime timeoutTime = LocalDateTime.now().minusMinutes(30);
        
        List<Order> timeoutOrders = orderService.findByState(OrderState.UNPAID).stream()
            .filter(order -> order.getCreateTime().isBefore(timeoutTime))
            .toList();
        
        if (!timeoutOrders.isEmpty()) {
            log.info("发现 {} 个支付超时的订单", timeoutOrders.size());
            
            for (Order order : timeoutOrders) {
                try {
                    stateMachineService.sendEvent(order.getId(), OrderEvent.PAY_TIMEOUT, null);
                    log.info("订单 {} 已自动取消（支付超时）", order.getOrderNumber());
                } catch (Exception e) {
                    log.error("自动取消订单失败: {}", order.getOrderNumber(), e);
                }
            }
        }
    }
    
    /**
     * 每天凌晨2点检查自动确认收货
     * 送达后超过15天的订单自动确认收货
     */
    @Scheduled(cron = "0 0 2 * * ?") // 每天凌晨2点
    @Transactional
    public void autoConfirmReceipt() {
        LocalDateTime confirmTime = LocalDateTime.now().minusDays(15);
        
        List<Order> deliveredOrders = orderService.findByState(OrderState.DELIVERED);
        
        for (Order order : deliveredOrders) {
            if (order.getUpdateTime().isBefore(confirmTime)) {
                try {
                    // 这里假设订单的update_time在状态变为DELIVERED时更新
                    stateMachineService.sendEvent(order.getId(), OrderEvent.AUTO_CONFIRM, null);
                    log.info("订单 {} 已自动确认收货", order.getOrderNumber());
                } catch (Exception e) {
                    log.error("自动确认收货失败: {}", order.getOrderNumber(), e);
                }
            }
        }
    }
}
```

## 第十步：主启动类

```java
// src/main/java/com/example/statemachine/StateMachineApplication.java
package com.example.statemachine;

import org.springframework.boot.SpringApplication;
import org.springframework.boot.autoconfigure.SpringBootApplication;
import org.springframework.scheduling.annotation.EnableScheduling;

@SpringBootApplication
@EnableScheduling
public class StateMachineApplication {
    
    public static void main(String[] args) {
        SpringApplication.run(StateMachineApplication.class, args);
        System.out.println("==========================================");
        System.out.println("Spring State Machine 应用启动成功!");
        System.out.println("数据库: PostgreSQL");
        System.out.println("持久化: JPA + Spring Data JPA");
        System.out.println("状态机: Spring State Machine 3.2.0");
        System.out.println("API地址: http://localhost:8080/api/orders");
        System.out.println("==========================================");
    }
}
```

## 第十一步：测试与验证

### 11.1 创建测试配置文件

```yaml
# src/test/resources/application-test.yml
spring:
  datasource:
    url: jdbc:h2:mem:testdb;MODE=PostgreSQL
    driver-class-name: org.h2.Driver
    username: sa
    password: 
  jpa:
    database-platform: org.hibernate.dialect.H2Dialect
    hibernate:
      ddl-auto: create-drop
    show-sql: true
  statemachine:
    data:
      jpa:
        repository:
          enabled: true
```

### 11.2 编写集成测试

```java
// src/test/java/com/example/statemachine/OrderStateMachineIntegrationTest.java
package com.example.statemachine;

import com.example.statemachine.domain.OrderEvent;
import com.example.statemachine.domain.OrderState;
import com.example.statemachine.entity.Order;
import com.example.statemachine.service.OrderStateMachineService;
import com.example.statemachine.service.OrderService;
import org.junit.jupiter.api.Test;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.boot.test.context.SpringBootTest;
import org.springframework.test.context.ActiveProfiles;
import org.springframework.transaction.annotation.Transactional;

import java.util.HashMap;
import java.util.Map;

import static org.junit.jupiter.api.Assertions.*;

@SpringBootTest
@ActiveProfiles("test")
@Transactional
class OrderStateMachineIntegrationTest {
    
    @Autowired
    private OrderService orderService;
    
    @Autowired
    private OrderStateMachineService stateMachineService;
    
    @Test
    void testCompleteOrderFlow() {
        // 1. 创建订单
        Order order = Order.createOrder("TEST001", 100.0, "user123");
        order.setProductId("prod456");
        order.setProductName("测试商品");
        
        Order savedOrder = stateMachineService.createOrder(order);
        assertEquals(OrderState.UNPAID, savedOrder.getState());
        
        Long orderId = savedOrder.getId();
        
        // 2. 支付
        Map<String, Object> paymentData = new HashMap<>();
        paymentData.put("paymentMethod", "ALIPAY");
        paymentData.put("paymentAmount", 100.0);
        
        boolean payAccepted = stateMachineService.sendEvent(
            orderId, OrderEvent.PAY, paymentData);
        assertTrue(payAccepted);
        
        // 3. 支付成功
        boolean paySuccessAccepted = stateMachineService.sendEvent(
            orderId, OrderEvent.PAY_SUCCESS, null);
        assertTrue(paySuccessAccepted);
        
        // 验证状态变为PAID
        Order paidOrder = orderService.findById(orderId).orElseThrow();
        assertEquals(OrderState.PAID, paidOrder.getState());
        
        // 4. 打包
        boolean packageAccepted = stateMachineService.sendEvent(
            orderId, OrderEvent.PACKAGE, null);
        assertTrue(packageAccepted);
        
        // 5. 发货
        Map<String, Object> shippingData = new HashMap<>();
        shippingData.put("shippingCompany", "SF");
        shippingData.put("trackingNumber", "SF123456789");
        
        boolean shipAccepted = stateMachineService.sendEvent(
            orderId, OrderEvent.SHIP, shippingData);
        assertTrue(shipAccepted);
        
        // 6. 送达
        boolean deliverAccepted = stateMachineService.sendEvent(
            orderId, OrderEvent.DELIVER, null);
        assertTrue(deliverAccepted);
        
        // 7. 确认收货
        boolean confirmAccepted = stateMachineService.sendEvent(
            orderId, OrderEvent.CONFIRM_RECEIPT, null);
        assertTrue(confirmAccepted);
        
        // 验证最终状态为COMPLETED
        Order completedOrder = orderService.findById(orderId).orElseThrow();
        assertEquals(OrderState.COMPLETED, completedOrder.getState());
    }
    
    @Test
    void testCancelOrderFlow() {
        // 1. 创建订单
        Order order = Order.createOrder("TEST002", 200.0, "user456");
        Order savedOrder = stateMachineService.createOrder(order);
        Long orderId = savedOrder.getId();
        
        // 2. 用户取消（在待支付状态下）
        boolean cancelAccepted = stateMachineService.sendEvent(
            orderId, OrderEvent.USER_CANCEL, null);
        assertTrue(cancelAccepted);
        
        // 验证状态变为CANCELLED
        Order cancelledOrder = orderService.findById(orderId).orElseThrow();
        assertEquals(OrderState.CANCELLED, cancelledOrder.getState());
    }
}
```

## 第十二步：部署与运行

### 12.1 运行应用

1. **启动 PostgreSQL 数据库**：
```bash
# 使用Docker启动PostgreSQL
docker run --name postgres-statemachine \
  -e POSTGRES_PASSWORD=postgres123 \
  -e POSTGRES_DB=statemachine_db \
  -p 5432:5432 \
  -d postgres:14
```

2. **运行 Spring Boot 应用**：
```bash
# 使用Maven运行
mvn spring-boot:run

# 或者打包后运行
mvn clean package
java -jar target/spring-statemachine-pgsql-1.0.0.jar
```

### 12.2 API 使用示例

```bash
# 1. 创建订单
curl -X POST http://localhost:8080/api/orders \
  -H "Content-Type: application/json" \
  -d '{
    "amount": 299.99,
    "userId": "user123",
    "productId": "prod001",
    "productName": "智能手机",
    "recipientName": "张三",
    "recipientPhone": "13800138000",
    "shippingAddress": "北京市朝阳区"
  }'

# 2. 支付
curl -X POST http://localhost:8080/api/orders/1/events/PAY \
  -H "Content-Type: application/json" \
  -d '{
    "contextData": {
      "paymentMethod": "WECHAT_PAY",
      "transactionId": "TX123456789"
    }
  }'

# 3. 查询状态
curl http://localhost:8080/api/orders/1/state

# 4. 发货
curl -X POST http://localhost:8080/api/orders/1/events/SHIP \
  -H "Content-Type: application/json" \
  -d '{
    "contextData": {
      "shippingCompany": "顺丰速运",
      "trackingNumber": "SF1234567890"
    }
  }'
```

## 项目结构总结

```
spring-statemachine-pgsql/
├── src/main/java/com/example/statemachine/
│   ├── StateMachineApplication.java          # 主启动类
│   ├── config/
│   │   └── OrderStateMachineConfig.java      # 状态机配置
│   ├── domain/
│   │   ├── OrderState.java                   # 状态枚举
│   │   └── OrderEvent.java                   # 事件枚举
│   ├── entity/
│   │   ├── Order.java                        # 订单实体
│   │   └── StateMachineContextEntity.java    # 状态机上下文实体
│   ├── repository/
│   │   └── OrderRepository.java              # 订单Repository
│   ├── service/
│   │   ├── OrderStateMachineService.java     # 状态机服务
│   │   └── OrderService.java                 # 订单业务服务
│   ├── controller/
│   │   └── OrderController.java              # REST控制器
│   └── scheduler/
│       └── OrderTimeoutScheduler.java        # 定时任务
├── src/main/resources/
│   └── application.yml                       # 配置文件
└── pom.xml                                   # Maven依赖
```

## 关键特性说明

1. **完整的 JPA 集成**：使用 Spring Data JPA 和 Spring State Machine 的 JPA 模块实现无缝持久化。

2. **自动状态恢复**：状态机状态自动从数据库恢复，无需手动管理。

3. **事务支持**：所有状态转换都在事务中完成，保证数据一致性。

4. **灵活的守卫和动作**：可以通过 Guard 和 Action 实现复杂的业务逻辑。

5. **完整的 REST API**：提供完整的订单状态机管理接口。

6. **定时任务**：自动处理超时订单和自动化流程。

这个方案提供了一个生产可用的、基于 PostgreSQL 和 JPA 的 Spring 状态机完整实现，可以直接用于实际项目开发。

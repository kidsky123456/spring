---
title: 07-spring状态机案例1
description: 
published: true
date: 2026-01-09T03:04:34.323Z
tags: 
editor: markdown
dateCreated: 2026-01-09T03:04:34.323Z
---

下面将**从零开始**，手把手教你搭建一个基于 **Spring Boot + Spring State Machine** 的状态机项目。我们将以一个 **订单状态流转系统** 为例（如：`待支付 → 已支付 → 已发货 → 已完成`），涵盖：

- 项目创建  
- 依赖引入  
- 状态/事件定义  
- 状态机配置  
- 动作与守卫  
- 控制器调用  
- 启动测试

------

## 🧱 一、环境准备

- JDK ≥ 17（Spring Boot 3.x 要求）
- Maven 或 Gradle
- IDE（IntelliJ IDEA / VS Code 等）

> 本文使用 **Spring Boot 3.2+** + **Maven**

------

## 📦 二、创建 Spring Boot 项目

### 1. 使用 [Spring Initializr](https://start.spring.io/) 创建项目

选择：

- Project: Maven
- Language: Java
- Spring Boot: 3.2.x
- Dependencies:
  - **Spring Web**
  - （可选）Lombok（简化代码）

### 2. 手动添加 Spring State Machine 依赖

在 `pom.xml` 中添加：

```xml
<dependency>
    <groupId>org.springframework.statemachine</groupId>
    <artifactId>spring-statemachine-starter</artifactId>
    <version>3.2.0</version>
</dependency>
```

> ✅ 注意：`3.2.0` 对应 Spring Boot 3.2，若你用 Spring Boot 2.x，请改用 `2.4.0` 并调整包名（`javax` → `jakarta` 不适用）。

------

## 🏷️ 三、定义状态（State）和事件（Event）

我们使用 **枚举** 来定义状态和事件，清晰且类型安全。

### 1. 定义订单状态 `OrderStates.java`

```java
// src/main/java/com/example/statemachine/states/OrderStates.java
package com.example.statemachine.states;

public enum OrderStates {
    UNPAID,      // 待支付
    PAID,        // 已支付
    SHIPPED,     // 已发货
    COMPLETED    // 已完成
}
```

### 2. 定义触发事件 `OrderEvents.java`

```java
// src/main/java/com/example/statemachine/events/OrderEvents.java
package com.example.statemachine.events;

public enum OrderEvents {
    PAY,         // 支付
    SHIP,        // 发货
    COMPLETE     // 完成
}
```

------

## ⚙️ 四、配置状态机（核心步骤）

创建配置类，继承 `EnumStateMachineConfigurerAdapter`，用于声明状态、转移、动作、守卫等。

### 创建 `OrderStateMachineConfig.java`

```java
// src/main/java/com/example/statemachine/config/OrderStateMachineConfig.java
package com.example.statemachine.config;

import com.example.statemachine.events.OrderEvents;
import com.example.statemachine.states.OrderStates;
import org.springframework.context.annotation.Configuration;
import org.springframework.statemachine.config.EnableStateMachine;
import org.springframework.statemachine.config.EnumStateMachineConfigurerAdapter;
import org.springframework.statemachine.config.builders.StateMachineStateConfigurer;
import org.springframework.statemachine.config.builders.StateMachineTransitionConfigurer;
import org.springframework.statemachine.guard.Guard;
import org.springframework.statemachine.listener.StateMachineListenerAdapter;
import org.springframework.statemachine.state.State;
import org.springframework.statemachine.transition.Transition;

import java.util.EnumSet;

@Configuration
@EnableStateMachine
public class OrderStateMachineConfig extends EnumStateMachineConfigurerAdapter<OrderStates, OrderEvents> {

    /**
     * 配置所有状态 + 初始状态
     */
    @Override
    public void configure(StateMachineStateConfigurer<OrderStates, OrderEvents> states) throws Exception {
        states
            .withStates()
            .initial(OrderStates.UNPAID) // 初始状态为 UNPAID
            .states(EnumSet.allOf(OrderStates.class)); // 注册所有状态
    }

    /**
     * 配置状态之间的转移规则
     */
    @Override
    public void configure(StateMachineTransitionConfigurer<OrderStates, OrderEvents> transitions) throws Exception {
        transitions
            // UNPAID --[PAY]--> PAID
            .withExternal()
                .source(OrderStates.UNPAID).target(OrderStates.PAID)
                .event(OrderEvents.PAY)
                .action((context) -> System.out.println("✅ 订单已支付！"))
                .and()
            // PAID --[SHIP]--> SHIPPED
            .withExternal()
                .source(OrderStates.PAID).target(OrderStates.SHIPPED)
                .event(OrderEvents.SHIP)
                .guard(paymentGuard()) // 添加守卫条件（可选）
                .action((context) -> System.out.println("🚚 订单已发货！"))
                .and()
            // SHIPPED --[COMPLETE]--> COMPLETED
            .withExternal()
                .source(OrderStates.SHIPPED).target(OrderStates.COMPLETED)
                .event(OrderEvents.COMPLETE)
                .action((context) -> System.out.println("🎉 订单已完成！"));
    }

    /**
     * 守卫：只有已支付的订单才能发货（演示用途）
     */
    private Guard<OrderStates, OrderEvents> paymentGuard() {
        return context -> {
            System.out.println("🔍 检查是否已支付...");
            // 实际项目中可查询数据库或上下文变量
            return true; // 这里简化为 always allow
        };
    }

    /**
     * 可选：添加监听器，观察状态变化
     */
    @Override
    public void configure(org.springframework.statemachine.config.configuration.StateMachineConfigurationConfigurer<OrderStates, OrderEvents> config) throws Exception {
        config
            .withConfiguration()
            .listener(new StateMachineListenerAdapter<>() {
                @Override
                public void stateChanged(State<OrderStates, OrderEvents> from, State<OrderStates, OrderEvents> to) {
                    System.out.println("🔄 状态变更: " + from.getId() + " → " + to.getId());
                }
            });
    }
}
```

> 🔍 说明：
>
> - `withExternal()` 表示外部转移（离开当前状态）
> - `action` 是转移成功后执行的逻辑
> - `guard` 是转移前的条件判断（返回 false 则拒绝转移）

------

## 🖥️ 五、创建控制器（用于测试）

### 创建 `OrderController.java`

```java
// src/main/java/com/example/statemachine/controller/OrderController.java
package com.example.statemachine.controller;

import com.example.statemachine.events.OrderEvents;
import com.example.statemachine.states.OrderStates;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.statemachine.StateMachine;
import org.springframework.web.bind.annotation.*;

@RestController
@RequestMapping("/order")
public class OrderController {

    @Autowired
    private StateMachine<OrderStates, OrderEvents> stateMachine;

    @PostMapping("/event")
    public String sendEvent(@RequestParam String event) {
        OrderEvents orderEvent = OrderEvents.valueOf(event.toUpperCase());

        // 必须先启动状态机（如果是新实例）
        if (!stateMachine.isRunning()) {
            stateMachine.start();
        }

        // 发送事件
        stateMachine.sendEvent(orderEvent);

        return "当前状态: " + stateMachine.getState().getId();
    }

    @GetMapping("/status")
    public String getStatus() {
        if (!stateMachine.isRunning()) {
            stateMachine.start();
        }
        return "当前状态: " + stateMachine.getState().getId();
    }
}
```

> ⚠️ 注意：默认情况下，Spring State Machine 是 **单例** 的。这意味着所有用户共享同一个状态机实例！
> 在真实业务中，每个订单应有独立的状态机（见“进阶建议”部分）。

------

## ▶️ 六、启动 & 测试

### 1. 启动应用

运行 `main` 方法（Spring Boot Application）

### 2. 使用 curl 或 Postman 测试

```bash
# 查看初始状态
curl http://localhost:8080/order/status
# 返回：当前状态: UNPAID

# 触发 PAY 事件
curl "http://localhost:8080/order/event?event=PAY"
# 输出日志：
#   ✅ 订单已支付！
#   🔄 状态变更: UNPAID → PAID
# 返回：当前状态: PAID

# 触发 SHIP
curl "http://localhost:8080/order/event?event=SHIP"

# 触发 COMPLETE
curl "http://localhost:8080/order/event?event=COMPLETE"
```

------

## 🧩 七、进阶建议（生产级考虑）

### ❗ 问题：当前是单例状态机，无法支持多订单

#### 解决方案：使用 **StateMachineFactory**

1. 修改配置类，移除 `@EnableStateMachine`，改为 `@EnableStateMachineFactory`

```java
@Configuration
@EnableStateMachineFactory
public class OrderStateMachineConfig extends EnumStateMachineConfigurerAdapter<OrderStates, OrderEvents> {
    // ... same as before
}
```

1. 控制器中通过 Factory 创建独立实例

```java
@Autowired
private StateMachineFactory<OrderStates, OrderEvents> stateMachineFactory;

@PostMapping("/order/{orderId}/event")
public String handleEvent(@PathVariable String orderId, @RequestParam String event) {
    StateMachine<OrderStates, OrderEvents> sm = stateMachineFactory.getStateMachine(orderId);
    sm.start(); // 每次需 start（或从持久化恢复）
    sm.sendEvent(OrderEvents.valueOf(event.toUpperCase()));
    return "订单 " + orderId + " 当前状态: " + sm.getState().getId();
}
```

> ✅ 这样每个 `orderId` 对应一个独立状态机实例。

### 💾 持久化状态（可选）

实现 `StateMachinePersist` 接口，将状态存入 Redis/DB，支持重启后恢复。

------

## ✅ 总结：完整步骤回顾

| 步骤 | 内容                                                         |
| ---- | ------------------------------------------------------------ |
| 1    | 创建 Spring Boot 项目（含 Web）                              |
| 2    | 添加 `spring-statemachine-starter` 依赖                      |
| 3    | 定义状态枚举（`OrderStates`）和事件枚举（`OrderEvents`）     |
| 4    | 编写状态机配置类（继承 `EnumStateMachineConfigurerAdapter`） |
| 5    | 配置初始状态、转移规则、动作、守卫                           |
| 6    | 创建 Controller，注入 `StateMachine` 并发送事件              |
| 7    | 启动应用，通过 HTTP 接口测试状态流转                         |
| 8    | （进阶）改用 `StateMachineFactory` 支持多实例                |
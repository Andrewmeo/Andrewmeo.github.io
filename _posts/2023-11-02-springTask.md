---
layout: post
title: SpringTask-订单状态定时处理
subtitle: 
categories: 青藤外卖用户客户端接口开发笔记
author: "Maxlec"
banner:
  image: https://github.com/Andrewmeo/images/blob/master/tales/0200.jpg?raw=true
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: springtask cron order
top: 20

---

## 1 SpringTask

> - 定时任务框架
> - 定时自动执行某段程序代码。

Spring Task 是 Spring 框架提供的任务调度工具。可以按照约定的时间自动执行某个代码逻辑。代码实现比较简单。

应用场景：

1. 快递24小时未领取提醒。
2. 票务系统处理未支付订单。
3. 银行贷款每月还款提醒。

**cron表达式：**

> 规则：分为6或7个域（位），由空格分隔开，每个域代表一个含义。每个域的含义分别是：秒、分、时、日、月、周、年（可选）。
>
> 如：2023年10月29日上午11点，对应的cron表达式为：0 0 29 10 ? 2023。未知周用"?"替代。

cron表达式就是一个字符串，通过cron表达式可以定义任务触发的时间。

cron表达式还有很多用法，不用死记硬背这些表达式。我们可以通过cron表达式在线生成。

**Task使用步骤：**

1. 导入maven坐标spring-context。
2. 在启动类上添加注解@EnableScheduling开启任务调度功能。
3. 自定义定时任务类。

## 2 订单状态

### 2.1 简单需求分析

如以下状态，我们需要通过定时任务触发执行处理这些状态。

1. 用户下单后可能超时未支付，订单一直处于"待支付"状态。
2. 用户收到外卖之后，订单一直处于"派送中"状态。

### 2.2 开发流程

#### 2.2.1 OrderTask

创建定时任务类，添加@Component注解。在任务类中定义处理超时支付订单的方法。在方法上添加@Scheduled注解，指定cron表达式，每分钟触发一次。

```java
/**
 * 定时任务类，定时处理订单状态
 */
@Component
@Slf4j
public class OrderTask {

    @Autowired
    private OrderMapper orderMapper;

    /**
     * 处理超时订单定时任务方法
     */
    @Scheduled(cron = "0 * * * * ?")
    public void processOrderTimeout() {
        log.info("处理超时订单：{}", LocalDateTime.now());

        // 查询超时15分钟的订单：select * from orders where status = ? and order_time < (now - 15)
        LocalDateTime time = LocalDateTime.now().plusMinutes(-15);
        List<Orders> orderList = orderMapper.getByStatusAndOrderTimeLT(Orders.PENDING_PAYMENT, time);

        if (orderList != null && orderList.size() > 0) {
            for (Orders orders : orderList) {
                orders.setStatus(Orders.CANCELLED);
                orders.setCancelReason(MessageConstant.ORDER_TIMEOUT);
                orders.setCancelTime(LocalDateTime.now());

                orderMapper.update(orders);
            }
        }
    }

    /**
     * 处理一直处于派送中的订单
     */
    @Scheduled(cron = "0 0 1 * * ?")
    public void processDeliveryOrder(){
        log.info("定时处理处于派送中的订单：{}", LocalDateTime.now());
        LocalDateTime time = LocalDateTime.now().plusMinutes(-60);
        List<Orders> orderList = orderMapper.getByStatusAndOrderTimeLT(Orders.DELIVERY_IN_PROGRESS, time);

        if (orderList != null && orderList.size() > 0) {
            for (Orders orders : orderList) {
                orders.setStatus(Orders.COMPLETED);

                orderMapper.update(orders);
            }
        }
    }
    
}
```

#### 2.2.2 OrderMapper

```java
    /**
     * 查询状态为待支付且超时15分钟的订单，或上一天处在派送中的订单
     * @param status
     * @param time
     * @return
     */
    @Select("select * from orders where status = #{status} and order_time < #{time}")
    List<Orders> getByStatusAndOrderTimeLT(Integer status, LocalDateTime time);
```

### 2.3 测试


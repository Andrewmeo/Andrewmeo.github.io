---
layout: post
title: 营业状态设置
subtitle: 
categories: 青藤外卖管理客户端接口开发笔记
author: "Maxlec"
banner:
  image: https://github.com/Andrewmeo/images/blob/master/tales/0100.jpg?raw=true
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: 营业状态
top: 11
---

主要的营业状态有：

1. 营业中：客户端用户可以进行点餐和下单。

2. 已打烊：客户端用户不能进行点餐和下单。

在管理端需要设置查询和修改营业状态的接口，而客户端只需设置查询营业状态的接口。基于Redis的字符串存储营业状态数据。

## 1 管理端更新和查询营业状态接口

### 1.1 接口设计

| 名称   | 类型          | 说明                             |
| ------ | ------------- | -------------------------------- |
| status | Integer：0或1 | 必须。1表示营业中，0表示已打烊。 |

status就是数据库中的SHOP_STATUS：

| key         | value |
| ----------- | ----- |
| SHOP_STATUS | 0或1  |

### 1.2 开发流程

#### ShopController

```java
@RestController("adminShopController") // 避免创建bean时发生冲突，为Bean定义name
@RequestMapping("/admin/shop")
@Api(tags = "营业接口")
@Slf4j
public class ShopController {

    public static final String KEY = "SHOP_STATUS";

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 设置营业状态
     * @param status
     * @return
     */
    @PutMapping("/{status}")
    @ApiOperation("设置营业状态")
    public Result setStatus(@PathVariable Integer status){
        log.info("设置营业状态为：{}", status == 1 ? "营业中" : "已打烊");
        redisTemplate.opsForValue().set(KEY, status);
        return Result.success();
    }

    /**
     * 查询营业状态
     * @return
     */
    @GetMapping("/status")
    @ApiOperation("查询营业状态")
    public Result<Integer> queryStatus(){
        Integer status = (Integer) redisTemplate.opsForValue().get(KEY);
        log.info("获取到的营业状态为：{}", status == 1 ? "营业中" : "已打烊");
        return Result.success(status);
    }

}
```

## 2 客户端查询营业状态接口

### 2.1 接口设计

客户端查询接口功能和服务端一样，只是请求路径不一样，接口不一样。查询接口的响应消息的数据：

| 名称 | 类型          | 说明                                 |
| ---- | ------------- | ------------------------------------ |
| code | integer：0或1 | 必须。1表示获取成功，0表示获取失败。 |
| data | integer：0或1 | 必须。1表示营业中，0表示已打烊。     |
| msg  | string        | 非必须。描述信息。                   |

### 2.2 开发流程

#### ShopController

```java
@RestController("userShopController") // 避免创建bean时发生冲突，为Bean定义name
@RequestMapping("/user/shop")
@Api(tags = "营业接口")
@Slf4j
public class ShopController {

    public static final String KEY = "SHOP_STATUS";

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 查询营业状态
     * @return
     */
    @GetMapping("/status")
    @ApiOperation("查询营业状态")
    public Result<Integer> queryStatus(){
        Integer status = (Integer) redisTemplate.opsForValue().get(KEY);
        log.info("获取到的营业状态为：{}", status == 1 ? "营业中" : "已打烊");
        return Result.success(status);
    }

}
```

## 3 测试

![](https://github.com/Andrewmeo/images/blob/main/image-20231016124543699.png?raw=true)
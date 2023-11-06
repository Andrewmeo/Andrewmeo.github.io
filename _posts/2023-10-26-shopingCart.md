---
layout: post
title: 购物车功能模块
subtitle: 
categories: 用户客户端接口开发笔记
author: "Maxlec"
banner:
  image: https://github.com/Andrewmeo/images/blob/main/QQ%E5%9B%BE%E7%89%8720230504160139.jpg?raw=true
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: 购物车
top: 15
---

## 1 添加购物车

### 1.1 接口设计

1. 基本信息：

> 1. Path: /user/shoppingCart/add
> 2. Method: POST

2. 请求参数：

Body:

| 名称       | 类型    | 是否必须 | 默认值 | 备注   | 其他信息          |
| :--------- | :------ | :------- | :----- | :----- | :---------------- |
| dishFlavor | string  | 非必须   |        | 口味   |                   |
| dishId     | integer | 非必须   |        | 菜品id | **format:** int64 |
| setmealId  | integer | 非必须   |        | 套餐id | **format:** int64 |

返回数据：

| 名称 | 类型    | 是否必须 | 默认值 | 备注 | 其他信息          |
| :--- | :------ | :------- | :----- | :--- | :---------------- |
| code | integer | 必须     |        |      | **format:** int32 |
| data | string  | 非必须   |        |      |                   |
| msg  | string  | 非必须   |        |      |                   |

### 1.2 开发流程

#### 1.2.1 user/ShoppingCartController

创建购物车控制器，定义添加购物车的接口方法，接收的请求方式为Post。通过shoppingCartDTO接收数据，DTO中封装了添加购物车菜品id和菜品口味，或者封装了套餐的id。所以在进行添加的逻辑处理时我们需要进行判断。

```java
@RestController
@RequestMapping("/user/shoppingCart")
@Api(tags = "客户端-购物车接口")
@Slf4j
public class ShoppingCartController {

    @Autowired
    private ShoppingCartService shoppingCartService;
    
    /**
     * 添加购物车
     * @param shoppingCartDTO
     * @return
     */
    @PostMapping("/add")
    @ApiOperation("添加购物车")
    public Result addCart(@RequestBody ShoppingCartDTO shoppingCartDTO){

        log.info("添加购物车商品：{}", shoppingCartDTO);
        shoppingCartService.add(shoppingCartDTO);
        return Result.success();
    }
    
}
```

#### 1.2.2 ShoppingCartServiceImpl

在add业务处理方法中，首先将DTO对象拷贝到购物车对象中，购物车对象中包含购物车商品的信息。然后查询购物车表中是否已经存在该购物车商品。如果存在直接对数量进行加1即可。

不存在则需要插入该购物车商品，首先我们需要判断该商品是菜品还是套餐，我们只需要判断id是否为空即可知道是菜品还是套餐。知道是添加菜品还是套餐到购物车后，我们需要为该购物车商品对象赋值然后插入数据库。要进行赋值的数据包括购物车商品的名称、图片、价格、初始数量1、购物车商品添加时间。

```java
    /**
     * 添加购物车
     * @param shoppingCartDTO
     */
    public void add(ShoppingCartDTO shoppingCartDTO) {
        // 通过查询，判断当前加入购物车中的商品是否已经存在
        // 1.拷贝购物商品信息作为查询条件
        ShoppingCart shoppingCart = new ShoppingCart();
        BeanUtils.copyProperties(shoppingCartDTO, shoppingCart);
        // 2.通过ThreadLocal获取用户Id作为查询条件
        Long userId = BaseContext.getCurrentId();
        shoppingCart.setUserId(userId);
        // 3.查询出购物商品信息
        List<ShoppingCart> list = shoppingCartMapper.list(shoppingCart);

        // 如果已经存在，只需要将数量加1
        if(list !=null && list.size() > 0){
            ShoppingCart cart = list.get(0);
            cart.setNumber(cart.getNumber()+1);

            shoppingCartMapper.updateNumberById(cart);
        }else{
            // 如果不存在，需要插入一条购物车数据
            // 1.判断添加购物车的商品是菜品还是套餐
            Long dishId = shoppingCartDTO.getDishId();
            if(dishId != null){
                // 本次添加的商品为菜品
                Dish dish = dishMapper.getById(dishId);
                shoppingCart.setName(dish.getName());
                shoppingCart.setImage(dish.getImage());
                shoppingCart.setAmount(dish.getPrice());
            }else{
                // 本次添加的商品为套餐
                SetMeal setMeal = setMealMapper.getById(shoppingCartDTO.getSetmealId());

                shoppingCart.setName(setMeal.getName());
                shoppingCart.setImage(setMeal.getImage());
                shoppingCart.setAmount(setMeal.getPrice());
            }
            shoppingCart.setNumber(1);
            shoppingCart.setCreateTime(LocalDateTime.now());
            shoppingCartMapper.insert(shoppingCart); // 最终进行商品插入购物车
        }
    }
```

#### 1.2.3 ShopingCartMapper.xml

最后调用mapper中的insert方法插入购物车商品对象。

```java
@Insert("insert into shopping_cart (name, image, user_id, dish_id, setmeal_id, dish_flavor, number, amount, create_time) " +
            "values (#{name}, #{image}, #{userId}, #{dishId}, #{setmealId}, #{dishFlavor}, #{number}, #{amount}, #{createTime})")
    void insert(ShoppingCart shoppingCart);
```

### 1.3 测试

## 2 查询获取购物车

### 2.1 接口设计

1. 基本信息：

> 1. Path: /user/shoppingCart/list
> 2. Method: GET

2. 请求参数：

返回数据

| 名称 | 类型      | 是否必须 | 默认值 | 备注 | 其他信息              |
| :--- | :-------- | :------- | :----- | :--- | :-------------------- |
| code | integer   | 必须     |        |      | **format:** int32     |
| data | object [] | 非必须   |        |      | **item 类型:** object |
| msg  | string    | 非必须   |        |      |                       |

### 2.2 开发流程

#### 2.2.1 user/ShoppingCartController

在获取购物车商品信息的接口中，接收的请求方式为Get，而不用接收参数。因为我们查询用户的购物车商品信息可以通过用户的查询返回，而用户的id可以从token中获取。

```java
    /**
     * 获取购物车商品列表
     * @return
     */
    @GetMapping("/list")
    @ApiOperation("获取购物车商品列表")
    public Result<List<ShoppingCart>> listCart() {

        List<ShoppingCart> list = shoppingCartService.showShoppingCart();

        return Result.success(list);
    }
```

#### 2.2.2 ShoppingCartServiceImpl

进入查询购物车商品信息的业务逻辑处理方法中，我们通过ThreadLocal获取到token解析出来载荷-用户id，通过用户id我们可以查询到多条购物车商品信息，封装到list中进行返回。

```java
/**
 * 获取购物车商品列表
 * @return
 */
public List<ShoppingCart> showShoppingCart() {

    Long currentUserId = BaseContext.getCurrentId();
    ShoppingCart shoppingCart = ShoppingCart.builder().userId(currentUserId).build();
    List<ShoppingCart> list = shoppingCartMapper.list(shoppingCart);

    return list;
}
```

#### 2.2.3 ShoppingCartMapper.xml

```xml
    <select id="list" resultType="com.bree.entity.ShoppingCart">
        select * from shopping_cart
        <where>
            <if test="userId != null">
                and user_id = #{userId}
            </if>
            <if test="setmealId != null">
                and setmeal_id = #{setmealId}
            </if>
            <if test="dishId != null">
                and dish_id = #{dishId}
            </if>
            <if test="dishFlavor != null">
                and dish_flavor = #{dishFlavor}
            </if>
        </where>
    </select>
```



### 2.3 测试

在查询获取购物车商品列表的业务方法中，打上断点，启动程序。再将客户端小程序启动，小程序启动后会自动发生获取购物车商品列表的请求，请求处理到达以下断点。

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231025142148896.png?raw=true)

单步调试，通过ThreadLocal获取到token解析出来载荷-用户id，为4，封装成shoppingCart对象进行查询。查询到该用户购物车中有三类商品。最后进行返回。

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231025142255168.png?raw=true)

## 3 减少购物车商品

### 3.1 接口设计

1. 基本信息：

> 1. Path: /user/shoppingCart/sub
> 2. Method: POST

2. 请求参数：

Headers：

| 参数名称     | 参数值           | 是否必须 | 示例 | 备注 |
| :----------- | :--------------- | :------- | :--- | :--- |
| Content-Type | application/json | 是       |      |      |

Body:

| 名称       | 类型    | 是否必须 | 默认值 | 备注   | 其他信息          |
| :--------- | :------ | :------- | :----- | :----- | :---------------- |
| dishFlavor | string  | 非必须   |        | 口味   |                   |
| dishId     | integer | 非必须   |        | 菜品id | **format:** int64 |
| setmealId  | integer | 非必须   |        | 套餐id | **format:** int64 |

返回数据

| 名称 | 类型    | 是否必须 | 默认值 | 备注 | 其他信息          |
| :--- | :------ | :------- | :----- | :--- | :---------------- |
| code | integer | 必须     |        |      | **format:** int32 |
| data | string  | 非必须   |        |      |                   |
| msg  | string  | 非必须   |        |      |                   |

### 3.2 开发流程

#### 3.2.1 user/ShoppingCartController

和添加购物车商品一样，删除一个购物车商品接收一个购物车DTO对象，因为我们不知道是删除菜品还是删除套餐，所以我们通过DTO对象接收商品id。不过接收的请求方法为Post，

```java
    /**
     * 减少购物车的商品
     * @param shoppingCartDTO
     * @return
     */
    @PostMapping("/sub")
    @ApiOperation("减少购物车商品")
    public Result subCart(@RequestBody ShoppingCartDTO shoppingCartDTO){
        log.info("减少购物车商品：{}", shoppingCartDTO);
        shoppingCartService.subShoppingCart(shoppingCartDTO);
        return Result.success();
    }
```

#### 3.2.2 ShoppingCartServiceImpl

在实现删除购物车一个商品的业务方法中，将DTO对象中的数据拷贝到新的购物车对象中，然后设置获取并设置购物车商品用户的id。然后通过购物车商品对象查出存在数据库中的商品数据，将商品的数量number减一。最后再更新到数据库中。（这里不用担心当number为0的情况，因为在小程序客户端当number减到0时就不能再点击删除了商品了）

```java
    /**
     * 减少购物车商品
     * @param shoppingCartDTO
     */
    public void subShoppingCart(ShoppingCartDTO shoppingCartDTO) {
        // 1.拷贝购物商品信息作为查询条件
        ShoppingCart shoppingCart = new ShoppingCart();
        BeanUtils.copyProperties(shoppingCartDTO, shoppingCart);
        // 2.通过ThreadLocal获取用户Id作为查询条件
        Long userId = BaseContext.getCurrentId();
        shoppingCart.setUserId(userId);
        // 3.查询出购物商品信息
        shoppingCart = shoppingCartMapper.list(shoppingCart).get(0);
        shoppingCart.setNumber(shoppingCart.getNumber()-1);

        shoppingCartMapper.updateNumberById(shoppingCart);
    }
```

### 3.3 测试

同样的，在实现业务实现方法中开始位置打上断点。客户端小程序执行删除购物车商品时，服务端接收到请求数据。继续单步调试。

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231025145503507.png?raw=true)

可以看到，经过购物车商品数量number减一处理后，更新到数据库中的购物车商品数量number为0。

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231025150638020.png?raw=true)

## 4 清空购物车

### 4.1 接口设计

1. 基本信息：

> 1. Path: /user/shoppingCart/clean
> 2. Method: DELETE

### 4.2 开发流程

#### 4.2.1 user/ShopingCartController

定义清空购物车的接口方法，接收的请求方式为Delete。同查询获取购物车一样不用接收，只需要在解析token的载荷中获取到用户id就可以将该用户下的所有购物车商品获取或者删除。

```java
    /**
     * 清空当前登录用户的购物车
     * @return
     */
    @DeleteMapping("/clean")
    @ApiOperation("清空购物车")
    public Result cleanCart(){
        shoppingCartService.cleanShoppingCart();
        return Result.success();
    }
```

#### 4.2.2 ShoppingCartServiceImpl

清空购物车业务实现方法以及SQL语句也很简单。

```java
    /**
     * 清空购物车
     */
    public void cleanShoppingCart() {

        Long currentUserId = BaseContext.getCurrentId();

        shoppingCartMapper.cleanByUserId(currentUserId);
    }
```

#### 4.2.3 ShoppingCartMapper

```java
/**
 * 清空购物车
 * @param currentUserId
 */
@Delete("delete from shopping_cart where user_id = #{currentUserId}")
void cleanByUserId(Long currentUserId);
```

### 4.3 测试

功能简单测试过程就不用写出来了。
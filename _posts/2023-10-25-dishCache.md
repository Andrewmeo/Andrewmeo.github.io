---
layout: post
title: 菜品缓存与清理
subtitle: 
categories: 用户客户端接口开发笔记
author: "Maxlec"
banner:
  image: https://github.com/Andrewmeo/images/blob/master/tales/0259.jpg?raw=true
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: redis redisTemplate springcache
top: 14
---

## 1 **菜品缓存**

在WEB后端应用程序来说，耗时比较大的往往有两个地方：一个是查数据库过程，一个是调用其它服务的API（因为其它服务最终也要去做查数据库等耗时操作）。缓存的原理其实提前准备好数据放在内存的某一个地方，目的其实就是提升查询效率。

### 1.1 缓存商品实现原理

客户端小程序展示的商品数据都是通过查询数据库获得的，如果用户访问量比较大，数据库访问压力随之增大。那么程序响应会变慢，用户体验差。

所以我们可以通过redis来缓存菜品数据，减少后端数据库查询操作。当用户端访问小程序，小程序需要向服务端查询数据进行展示。

当请求到达服务端后首先询问是否存在该请求数据的缓存，如果存在则直接返回，不用再向数据库查询。询问缓存，不存在则需要查询数据库获取数据，然后将数据载入缓存，方便后面的请求调用获取。

数据库中的数据有变更的同时，需要清理缓存中失效的数据。然后在在下次查询的时候在将新的数据载入缓存中。

（我有一个思考，就是如果客户端发送更新或者删除操作，特别是针对更新操作，我们是否可以不清理缓存中失效的数据，而是更新缓存中失效的数据，最后再其更新到数据库中再响应更新成功。

这样的处理数据的方式其实可以用于高访问量的热点数据，其他请求获取时直接从缓存中拿。当然一直在内存中操作意味着该数据是不安全的，所以对于一些重要的数据还是尽量通过数据库操作）

在此我还是保守一点老实清理由于更新删除等操作导致的失效数据。

### 1.2 接口设计

1. 基本信息：

> 1. Path: /user/dish/list
> 2. Method: Get

2. 请求参数：

Query：

| 参数名称   | 是否必须 | 示例 | 备注   |
| :--------- | :------- | :--- | :----- |
| categoryId | 是       | 101  | 分类id |

3. 返回数据

| 名称 | 类型      | 是否必须 | 默认值 | 备注 | 其他信息              |
| :--- | :-------- | :------- | :----- | :--- | :-------------------- |
| code | integer   | 必须     |        |      | **format:** int32     |
| data | object [] | 非必须   |        |      | **item 类型:** object |
| msg  | string    | 非必须   |        |      |                       |

### 1.3 开发流程

我们在管理端开发商店营业状态时已经进行过redis的配置了，包括引入redis依赖，设置redis的属性以及声明配置redis的操作对象redisTemplate。在redis文章中有详细的介绍。

```yaml
redis:
    host: ${ivy.redis.host}
    port: ${ivy.redis.port}
    password: ${ivy.redis.password}
```

#### 1.3.1 RedisConfiguration

```java
@Configuration
@Slf4j
public class RedisConfiguration {

    @Bean
    public RedisTemplate redisTemplate(RedisConnectionFactory redisConnectionFactory){
        log.info("开始创建redisTemplate对象...");
        RedisTemplate redisTemplate = new RedisTemplate();
        // 设置redis连接工厂对象
        redisTemplate.setConnectionFactory(redisConnectionFactory);
        // 设置key的序列化器，默认为JdkSerializationRedisSerializer
        redisTemplate.setKeySerializer(new StringRedisSerializer());

        return redisTemplate;
    }
    
}
```

#### 1.3.2 user/DishController

所以在用户端菜品控制器中我们可以直接注入redisTemplate对象，然对缓存数据进行写入和读取。

```java
@RestController("userDishController")
@RequestMapping("/user/dish")
@Slf4j
@Api(tags = "客户端端-菜品浏览接口")
public class DishController {

    @Autowired
    private DishService dishService;

    @Autowired
    private RedisTemplate redisTemplate;

    /**
     * 根据分类id查询菜品
     * @param categoryId
     * @return
     */
    @GetMapping("/list")
    @ApiOperation("根据分类id查询菜品")
    public Result<List<DishVO>> list(Long categoryId) {

        // 构造redis中的key，规则dish_分类id
        String key = "dish_"+categoryId;
        // 查询redis中是否存在该分类下的菜品数据，每个分类下存在多个菜品
        List<DishVO> list = (List<DishVO>) redisTemplate.opsForValue().get(key);

        if(list != null && list.size() > 0){
            // 如果存在，直接返回，无需查询数据库
            return Result.success(list);
        }

        // 如果不存在，查询数据库，将查询到的数据放入redis中
        list = dishService.listWithFlavor(categoryId);
        redisTemplate.opsForValue().set(key, list);

        return Result.success(list);
    }

}
```

在用户端的菜品控制器中目前只有菜品查询接口方法，在方法中根据分类id查询菜品列表，该查询数据我们可以放入缓存中。我们将缓存数据的key定义为dish_categoryId。当查询某分类下的菜品数据时，会先从redis缓存中获取这些数据，如果存在这些数据会直接将其返回。不用再进行数据库的查询。如果缓存中未插入这些数据，则需要从数据库中查询，然后返回将这些数据插入到redis缓存中再进返回查询数据。

### 1.4 测试

#### 1.4.1 测试数据插入缓存

我们以Debug模式启动应用，在该接口方法中打上断点。

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231023135904177.png?raw=true)

启动客户端小程序，首先自动进行第一栏分类下的数据查询，服务端执行到断点。我们进行单步调试，可以看到，请求要查询的是分类id为16的菜品数据。在reids缓存中查询不到key为dish_16的数据，所以向数据库中查询分类下的菜品数据。

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231023140309922.png?raw=true)

查询到的三条菜品数据，放入list中，然后插入到redis缓存中后进行返回。

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231023141026757.png?raw=true)

打开redis客户端图像界面可以看到已经插入成功了。

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231023141154194.png?raw=true)

#### 1.4.2 测试读取缓存

依然在同一个位置打上断点，然后再次进行相同的请求。可以看到首先从缓存中查询到了该分类下的三条菜品数据，所以可以直接进行返回。

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231023141558208.png?raw=true)

## 2 **清理菜品缓存**

缓存不可能永远存储数据，当数据库中的数据发生变化时，缓存中对应的数据就已经失效了，我们需要将其清除掉。比如管理客户端发送更新、删除或者新增操作时，数据库中的数据内容和大小就发生了变化。

### 2.1 admin/DishController

所以我们在管理端菜品控制器中定义cleanCache方法，方便调用清理掉失效的数据。该方法接收一个pattern字符串，通过redisTemplate查询出匹配的key，然后调用delete进行清理。

```java
/**
 * 清理缓存数据
 * @param pattern
 */
private void cleanCache(String pattern) {
    Set keys = redisTemplate.keys(pattern);
    redisTemplate.delete(keys);
}
```

首先是新增数据，新增一个菜品，该菜品一定属于一个分类下的菜品，所以对应的菜品分类下会多一个菜品数据。所以存在redis缓存中的该分类下的数据就失效了，需要将其清理。（此次变化只影响到了一个分类数据，所以只清理掉该分类下的数据。）

```java
    public Result save(@RequestBody DishDTO dishDTO){
        log.info("新增菜品：{}", dishDTO);
        dishService.save(dishDTO);

        // 清理所在分类的缓存数据
        String key = "dish_"+dishDTO.getCategoryId();
        cleanCache(key);

        return Result.success();
    }
```

然后是删除数据，批量删除菜品。批量删除的菜品可能存在于多个分类下，所以我们通过dish_*将所有分类下的数据全部清理。

（这里其实可以进行一个判断，如果只是删除一个菜品，那么我们可以只清理一个分类下的数据。因为大多数情况是只删除一个菜品数据，只清理一个分类数据的话，其他未影响到的分类数据就可以不用重复查询插入了，提高效率）

```java
    public Result delete(@RequestParam List<Long> ids){
        log.info("批量删除菜品：{}", ids);
        dishService.deleteBatch(ids);

        // 批量删除菜品也要将所有菜品缓存数据清理掉，因为不同的菜品可能在不同的分类下
        cleanCache("dish_*");
        return Result.success();
    }
```

最后是更新数据，更新一个菜品的数据，看似修改一个菜品只影响到了一个分类数据，但是如果该菜品修改的是分类属性，那么就影响到了两个分类下的数据，都发生改变，都失效了。所以直接简单处理，清理掉所有分类下的数据。

```java
    public Result dishUpdate(@RequestBody DishDTO dishDTO){
        log.info("更新菜品：{}", dishDTO);
        dishService.dishUpdate(dishDTO);

        // 将所有菜品缓存数据清理掉，即清理所有dish_开头的key
        // 因为修改菜品时可能会修改到分类，那么该分类下菜品减少，另一个分类下增加，所以要清理两个分类下的菜品数据，但是再去做各种判断的话比较麻烦。
        // （记住清除缓存是为了查询的数据与数据库一致）
        cleanCache("dish_*");
        return Result.success();
    }
```

还有其他对到分类下数据造成改变的操作就不例举了。

## 3 **查询套餐缓存**

### 3.1 需求分析

我们使用SpringCache注解的方式代替通过redisTemplate对缓存数据的原始操作。

SpringCache通过注解的方式能自动将数据插入缓存，从缓存中读取数据，以及对缓存中数据的清理。相当于SpringCache封装了redis的操作，还对数据的变化进行监控然后自动完成数据的插入、读取以及清理。

### 3.2 接口设计

### 3.3 开发流程

#### 3.3.1 user/SetMealController

```java
@RestController("userSetMealController")
@RequestMapping("/user/setmeal")
@Api(tags = "用户端-套餐浏览接口")
public class SetMealController {
    @Autowired
    private SetMealService setMealService;

    /**
     * 条件查询
     *
     * @param categoryId
     * @return
     */
    @GetMapping("/list")
    @ApiOperation("根据分类id查询套餐")
    @Cacheable(cacheNames = "setMealCache", key = "#categoryId") // key：setMealCache::12
    public Result<List<SetMeal>> list(Long categoryId) {
        SetMeal setmeal = new SetMeal();
        setmeal.setCategoryId(categoryId);
        setmeal.setStatus(StatusConstant.ENABLE);

        List<SetMeal> list = setMealService.list(setmeal);
        return Result.success(list);
    }

    /**
     * 根据套餐id查询包含的菜品列表
     * @param id
     * @return
     */
    @GetMapping("/dish/{id}")
    @ApiOperation("根据套餐id查询包含的菜品列表")
    public Result<List<DishItemVO>> dishList(@PathVariable("id") Long id) {
        List<DishItemVO> list = setMealService.getDishItemById(id);
        return Result.success(list);
    }
}
```

#### 3.3.2 admin/SetMealController

```java
@RestController
@RequestMapping("/admin/setmeal")
@Api(tags = "套餐管理相关接口")
@Slf4j
public class SetMealController {

    @Autowired
    private SetMealService setmealService;

    /**
     * 新增套餐
     * @param setMealDTO
     * @return
     */
    @PostMapping
    @ApiOperation("新增套餐")
    @CacheEvict(cacheNames = "setMealCache", key = "#setMealDTO.categoryId") // key：setMealCache::12
    public Result save(@RequestBody SetMealDTO setMealDTO) {
        setmealService.saveWithDish(setMealDTO);
        return Result.success();
    }

    /**
     * 分页查询
     * @param setmealPageQueryDTO
     * @return
     */
    @GetMapping("/page")
    @ApiOperation("分页查询")
    public Result<PageResult> page(SetMealPageQueryDTO setmealPageQueryDTO) {
        PageResult pageResult = setmealService.pageQuery(setmealPageQueryDTO);
        return Result.success(pageResult);
    }

    /**
     * 批量删除套餐
     * @param ids
     * @return
     */
    @DeleteMapping
    @ApiOperation("批量删除套餐")
    @CacheEvict(cacheNames = "setMealCache", allEntries = true) // 删除setMealCache下的所有缓存
    public Result delete(@RequestParam List<Long> ids){
        setmealService.deleteBatch(ids);
        return Result.success();
    }

    /**
     * 根据id查询套餐，用于修改页面回显数据
     *
     * @param id
     * @return
     */
    @GetMapping("/{id}")
    @ApiOperation("根据id查询套餐")
    public Result<SetMealVO> getById(@PathVariable Long id) {
        SetMealVO setmealVO = setmealService.getByIdWithDish(id);
        return Result.success(setmealVO);
    }

    /**
     * 修改套餐
     * @param setMealDTO
     * @return
     */
    @PutMapping
    @ApiOperation("修改套餐")
    @CacheEvict(cacheNames = "setMealCache", allEntries = true) // 删除setMealCache下的所有缓存
    public Result update(@RequestBody SetMealDTO setMealDTO) {
        setmealService.update(setMealDTO);
        return Result.success();
    }

    /**
     * 套餐起售停售
     * @param status
     * @param id
     * @return
     */
    @PostMapping("/status/{status}")
    @ApiOperation("套餐起售停售")
    @CacheEvict(cacheNames = "setMealCache", allEntries = true) // 删除setMealCache下的所有缓存
    public Result startOrStop(@PathVariable Integer status, Long id) {
        setmealService.startOrStop(status, id);
        return Result.success();
    }
} 
```

### 3.4 测试
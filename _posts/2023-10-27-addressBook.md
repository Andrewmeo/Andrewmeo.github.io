---
layout: post
title: 地址薄模块功能
subtitle: 
categories: 用户客户端接口开发笔记
author: "Maxlec"
banner:
  image: https://github.com/Andrewmeo/images/blob/master/tales/0015.jpg?raw=true
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: 地址簿
top: 16
---

## 1 新增地址

### 1.1 接口设计

1. 基本信息：

> 1. Path: /user/addressBook
> 2. Method: POST

2. 请求参数：

Headers：

| 参数名称     | 参数值           | 是否必须 | 示例 | 备注 |
| :----------- | :--------------- | :------- | :--- | :--- |
| Content-Type | application/json | 是       |      |      |

Body:

| 名称         | 类型    | 是否必须 | 默认值 | 备注     | 其他信息          |
| :----------- | :------ | :------- | :----- | :------- | :---------------- |
| cityCode     | string  | 非必须   |        |          |                   |
| cityName     | string  | 非必须   |        |          |                   |
| consignee    | string  | 非必须   |        |          |                   |
| detail       | string  | 必须     |        | 详细地址 |                   |
| districtCode | string  | 非必须   |        |          |                   |
| districtName | string  | 非必须   |        |          |                   |
| id           | integer | 非必须   |        | 主键值   | **format:** int64 |
| isDefault    | integer | 非必须   |        |          | **format:** int32 |
| label        | string  | 非必须   |        |          |                   |
| phone        | string  | 必须     |        | 手机号   |                   |
| provinceCode | string  | 非必须   |        |          |                   |
| provinceName | string  | 非必须   |        |          |                   |
| sex          | string  | 必须     |        |          |                   |
| userId       | integer | 非必须   |        |          | **format:** int64 |

### 1.2 开发流程

#### 1.2.1 user/AddressBookController

首先需要创建地址簿控制器，在控制器中定义新增地址接口，接收的请求方式为Post，使用addressBook对象接收用户传递的地址数据。

```java
@RestController
@RequestMapping("/user/addressBook")
@Api(tags = "客户端小程序地址簿接口")
public class AddressBookController {
    
    @Autowired
    private AddressBookService addressBookService;
    
	/**
     * 新增地址
     *
     * @param addressBook
     * @return
     */
    @PostMapping
    @ApiOperation("新增地址")
    public Result save(@RequestBody AddressBook addressBook) {
        addressBookService.save(addressBook);
        return Result.success();
    }
}
```

#### 1.2.2 AddressBookServiceImpl

新增地址的业务实现方法也比较简单，addressBook对象中包含地址数据，我们还需要添加该用户id标识，以及设置该地址为非默认地址，最后插入到地址表中。

```java
    /**
     * 新增地址
     *
     * @param addressBook
     */
    public void save(AddressBook addressBook) {
        addressBook.setUserId(BaseContext.getCurrentId());
        addressBook.setIsDefault(0);
        addressBookMapper.insert(addressBook);
    }
```

### 1.3 测试

功能简单，测试过程省略。

## 2 编辑地址

功能相关接口：根据id查询地址接口、根据id修改地址。前者是因为需要将要修改的地址查询出来方便用户参考修改。

### 2.1 接口设计

1. 基本信息：

> 1. Path: /user/addressBook
> 2. Method: PUT

2. 请求参数：接收修改后的数据同新增地址一样。

### 2.2 开发流程

#### 2.2.1 user/AddressBookController

在地址簿控制器中继续定义更新地址的接口方法，该方法接收的请求方式为Put，依然是用addressBook接收修改后的数据。所以说相比于新增地址接口，修改地址接口也就接收的请求方法不一样，以及接收的数据中包含地址id。

```java
    @PutMapping
    @ApiOperation("根据id修改地址")
    public Result update(@RequestBody AddressBook addressBook) {
        addressBookService.update(addressBook);
        return Result.success();
    }
```

#### 2.2.2 AddressBookServiceImpl

更新的业务方法没有什么复杂的。

```java
    /**
     * 根据id修改地址
     *
     * @param addressBook
     */
    public void update(AddressBook addressBook) {
        addressBookMapper.update(addressBook);
    }
```

#### 2.2.3 AddressBookMapper.xml

主要写一下动态更新的SQL语句。根据地址id更新，有数据就更新，没有数据就不更新。

```xml
<update id="update" parameterType="addressBook">
    update address_book
    <set>
        <if test="consignee != null">consignee = #{consignee},</if>
        <if test="sex != null">sex = #{sex},</if>
        <if test="phone != null">phone = #{phone},</if>
        <if test="detail != null">detail = #{detail},</if>
        <if test="label != null">label = #{label},</if>
        <if test="isDefault != null">is_default = #{isDefault},</if>
    </set>
    where id = #{id}
</update>
```

### 2.3 测试

功能简单，测试过程省略。

## 3 查询地址

### 3.1 接口设计

1. 基本信息：

> 1. Path: /user/addressBook/{id}
> 2. Method: GET

2. 请求参数：

路径参数：

| 参数名称 | 示例 | 备注   |
| :------- | :--- | :----- |
| id       | 101  | 地址id |

### 3.2 开发流程

#### 3.2.1 user/AddressBookController

在地址簿控制器中继续定义查询地址的接口方法，该方法接收的请求方式为Get。通过路径参数接收查询地址的id，注意需要加上@PathVariable。

```java
    @GetMapping("/{id}")
    @ApiOperation("根据id查询地址")
    public Result<AddressBook> getById(@PathVariable Long id) {
        AddressBook addressBook = addressBookService.getById(id);
        return Result.success(addressBook);
    }
```

业务逻辑处理方法和SQL都比较简单，省略。

### 3.3 测试

功能简单，测试过程省略。

## 4 删除地址

### 4.1 接口设计

1. 基本信息：

> 1. Path: /user/addressBook
> 2. Method: DELETE

2. 请求参数：

Query：

| 参数名称 | 是否必须 | 示例 | 备注   |
| :------- | :------- | :--- | :----- |
| id       | 是       | 101  | 地址id |

### 4.2 开发流程

#### 4.2.1 user/AddressBookController

在地址簿控制器中继续定义删除地址的接口方法，该方法接收的请求方式为Delete。只需要接收地址id，根据地址id删除地址就行。

```java
    @DeleteMapping
    @ApiOperation("根据id删除地址")
    public Result deleteById(Long id) {
        addressBookService.deleteById(id);
        return Result.success();
    }
```

业务逻辑处理方法和SQL都比较简单，省略。

### 4.3 测试

功能简单，测试过程省略。

## 5 获取地址列表

### 5.1 接口设计

1. 基本信息：

> 1. Path: /user/addressBook/list
> 2. Method: GET

2. 请求参数：无。

3. 返回数据：返回的数据为包含AddressBook类型数据的集合对象。

### 5.2 开发流程

#### 5.2.1 user/AddressBookController

在地址簿控制器中继续定义查询获取地址列表的接口方法，该方法接收的请求方式为Get。无接收的参数，因为查询所有地址只需要知道该用户id即可。

```java
    /**
     * 查询当前登录用户的所有地址信息
     * @return
     */
    @GetMapping("/list")
    @ApiOperation("查询当前登录用户的所有地址信息")
    public Result<List<AddressBook>> list() {
        AddressBook addressBook = new AddressBook();
        addressBook.setUserId(BaseContext.getCurrentId());
        List<AddressBook> list = addressBookService.list(addressBook);
        return Result.success(list);
    }
```

业务逻辑处理方法和SQL都比较简单，省略。

### 5.3 测试

功能简单，测试过程省略。

## 6 设置默认地址

### 6.1 接口设计

1. 基本信息：

> 1. Path: /user/addressBook/default
> 2. Method: PUT

2. 请求参数：

Headers：

| 参数名称     | 参数值           | 是否必须 | 示例 | 备注 |
| :----------- | :--------------- | :------- | :--- | :--- |
| Content-Type | application/json | 是       |      |      |

Body:

| 名称 | 类型    | 是否必须 | 默认值 | 备注   | 其他信息          |
| :--- | :------ | :------- | :----- | :----- | :---------------- |
| id   | integer | 必须     |        | 地址id | **format:** int64 |

### 6.2 开发流程

#### 6.2.1 user/AddressBookController

设置默认地址其实也就是针对某个字段更新的过程。

```java
@PutMapping("/default")
@ApiOperation("设置默认地址")
public Result setDefault(@RequestBody AddressBook addressBook) {
    addressBookService.setDefault(addressBook);
    return Result.success();
}
```

#### 6.2.2 AddressBookServiceImpl

在业务处理方法中，我们需要将之前的默认地址改为非默认地址，我们只需要根据用户id将地址表所有地址的is_default设置为0（再去查询哪个是默认地址就麻烦了）。然后我们再将根据当前地址id设置当前地址为默认地址。

```java
@Transactional
public void setDefault(AddressBook addressBook) {
    //1、将当前用户的所有地址修改为非默认地址 update address_book set is_default = ? where user_id = ?
    addressBook.setIsDefault(0);
    addressBook.setUserId(BaseContext.getCurrentId());
    addressBookMapper.updateIsDefaultByUserId(addressBook);

    //2、将当前地址改为默认地址 update address_book set is_default = ? where id = ?
    addressBook.setIsDefault(1);
    addressBookMapper.update(addressBook);
}
```

### 6.3 测试

测试过程省略。

## 7  获取默认地址

获取默认地址的功能在用户下单时调用。

### 7.1 接口设计

### 7.2 开发流程

#### 7.2.1 user/AddressBookController

```java
    @GetMapping("default")
    @ApiOperation("查询默认地址")
    public Result<AddressBook> getDefault() {
        //SQL:select * from address_book where user_id = ? and is_default = 1
        AddressBook addressBook = new AddressBook();
        addressBook.setIsDefault(1);
        addressBook.setUserId(BaseContext.getCurrentId());
        List<AddressBook> list = addressBookService.list(addressBook);

        if (list != null && list.size() == 1) {
            return Result.success(list.get(0));
        }

        return Result.error("没有查询到默认地址");
    }
```

#### 7.2.2 AddressBookServiceImpl

```java
    /**
     * 条件查询
     *
     * @param addressBook
     * @return
     */
    public List<AddressBook> list(AddressBook addressBook) {
        return addressBookMapper.list(addressBook);
    }
```

#### 7.2.3 AddressBookMapper.xml

```xml
    <select id="list" parameterType="AddressBook" resultType="AddressBook">
        select * from address_book
        <where>
            <if test="userId != null">and user_id = #{userId}</if>
            <if test="phone != null">and phone = #{phone}</if>
            <if test="isDefault != null">and is_default = #{isDefault}</if>
        </where>
    </select>
```

### 7.3 测试


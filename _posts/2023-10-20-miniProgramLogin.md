---
layout: post
title: 微信小程序开发准备与用户登录
subtitle: 
categories: 用户客户端接口开发笔记
author: "Maxlec"
banner:
  image: https://github.com/Andrewmeo/images/blob/master/%E7%8E%AF%E5%BD%A2%E7%89%A9%E8%AF%AD/0235.jpg?raw=true
  opacity: 0.618
  background: "#000"
  height: "100vh"
  min_height: "38vh"
  heading_style: "font-size: 4.25em; font-weight: bold; text-decoration: underline"
  subheading_style: "color: gold"
tags: 微信服务接口 httpclient postman
top: 13
---

## **Apache HttpClient入门和使用**

### 介绍

HttpClient是Apache下的子项目，可以用来提供高效、最新的、功能丰富的支持HTTP协议的客户端编程工具包，并且它支持HTTP协议最新的版本和建议。（项目中已经导入了AliyunOss依赖，其中传递了httpclient的依赖）

```xml
<dependency>
    <groupId>org.apache.httpcomponents.client5</groupId>
    <artifactId>httpclient5</artifactId>
    <version>5.2.1</version>
</dependency>
```

工具包的核心API：

1. HttpClient：接口，定义发送http请求的功能。
2. HttpClients：创建HTTP对象
3. CloseableHttpClient：具体实现了HttpClient接口。
4. HttpGet
5. HttpPost

发送请求步骤：

1. 通过实现类创建HttpClient对象发送请求。
2. 创建Http请求对象。
3. 调用HttpClient的execute方法发送请求。

### 入门案例

发送Get请求：

```java
// 通过httpclient发送GET方式的请求
	public void testGET() throws Exception{
        // 1.创建httpclient对象
        CloseableHttpClient httpClient = HttpClients.createDefault();

        // 2.创建请求对象
        HttpGet httpGet = new HttpGet("http://localhost:8070/user/shop/status");

        // 3.发送请求，接受响应结果
        CloseableHttpResponse response = httpClient.execute(httpGet);

        // 获取服务端返回的状态码
        int statusCode = response.getStatusLine().getStatusCode();
        System.out.println("服务端返回的状态码为：" + statusCode);

        // 获取并解析具体响应数据
        HttpEntity entity = response.getEntity();
        String body = EntityUtils.toString(entity);
        System.out.println("服务端返回的数据为：" + body);

        //关闭资源
        response.close();
        httpClient.close();
    }
```

发送Post请求：

```java
// 通过httpclient发送POST方式的请求
	public void testPOST() throws Exception{
        // 创建httpclient对象
        CloseableHttpClient httpClient = HttpClients.createDefault();

        // 1.创建请求对象
        HttpPost httpPost = new HttpPost("http://localhost:8070/admin/employee/login");
        
        //封装具体数据
        JSONObject jsonObject = new JSONObject();
        jsonObject.put("username","admin");
        jsonObject.put("password","123456");
        StringEntity entity = new StringEntity(jsonObject.toString());
        
        //指定请求编码方式
        entity.setContentEncoding("utf-8");
        //数据格式
        entity.setContentType("application/json");
        //加入Post请求中
        httpPost.setEntity(entity);

        // 2.发送请求
        CloseableHttpResponse response = httpClient.execute(httpPost);

        // 3.解析返回结果
        int statusCode = response.getStatusLine().getStatusCode();
        System.out.println("响应码为：" + statusCode);

        HttpEntity entity1 = response.getEntity();
        String body = EntityUtils.toString(entity1);
        System.out.println("响应数据为：" + body);

        //关闭资源
        response.close();
        httpClient.close();
    }
```

## **用户端微信小程序开发**

### 微信小程序介绍

小程序：一种新的开放能力，可以在微信内被便捷地获取和传播，同时具有出色的使用体验。

> 微信小程序官网：https://mp.weixin.qq.com/cgi-bin/wx

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231016205144564.png?raw=true)

开放注册范围有个人、企业、政府、媒体、其他组织。不同组织开放的权限不一样。

接入流程：

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231016205428906.png?raw=true)

### 开发准备工作

1. 注册：在微信公众平台注册小程序。地址：https://mp.weixin.qq.com/wxopen/waregister?action=step1
2. 完善小程序基本信息，名称、头像、介绍。以及服务范围等。
3. 下载开发者工具，参考开发文档进行小程序开发和调试。下载地址：https://developers.weixin.qq.com/doc/

首先进入注册页面，输入信息进行注册。

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231016210239235.png?raw=true)

注册后进入微信公众平台，然后就可以完善小程序基本信息：

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231016210923589.png?raw=true)

完善后下载开发工具进行开发：

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231016211512972.png?raw=true)

安装后进行开发平台创建小程序，其中AppID在微信公众平台上开发管理中：

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231016212613067.png?raw=true)

微信小程序开发工具的开发平台如下：

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231016213132625.png?raw=true)

### 入门案例

小程序目录结构：

**主体部分：**

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231016214444406.png?raw=true)

**page页面：**

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231016214838209.png?raw=true)

**编写页面代码：**

```html
<!--index.wxml-->
<navigation-bar title="WeiXin" back="{{false}}" color="black" background="#FFF"></navigation-bar>
<scroll-view class="scrollarea" scroll-y type="list">
  <view class="container">
    <view>
      {{msg}}
    </view>
    <view>
      <button bind:tap="userLogin" type="primary">同意协议并登录</button>
    </view>
    <view>
      <button bind:tap="getStatus" type="primary">获取营业状态</button>
    </view>
  </view>
</scroll-view>
```

```javascript
// index.js
Page({
  data: {
    msg: '用户信息'
  },
  // 微信用户登录，获取用户授权码
  userLogin(){
    wx.login({
      desc: '用户登录',
      success: (res) => {
        console.log(res.code)
      }
    })
  },
  getStatus(){
    wx.request({
      url: 'http://localhost:8070/user/shop/status',
      method: 'GET',
      success: (res) => {
        console.log(res.data)
      }
    })
  }
})
```

### 发布小程序

当小程序代码开发完毕后，就可以进行发布上线，让客户端用户能访问使用。

首先上传该版本到微信公众平台，未审核的版本称为开发版本：

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231017121000544.png?raw=true)

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231017121300615.png?raw=true)

上传为开发版本后还需要提交审核，通过后才能进行发布上线：

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231017121346227.png?raw=true)

## **用户登录流程开发**

我已经开发好了小程序页面，现在需要进行前后端的联调，首先是进行用户登录。

### 小程序用户登录原理

> 详细的文档说明：[开放能力 / 用户信息 / 小程序登录 (qq.com)](https://developers.weixin.qq.com/miniprogram/dev/framework/open-ability/login.html)

流程：以下是微信官方文档的说明。

> 1. 在小程序客户端中调用 [wx.login()](https://developers.weixin.qq.com/miniprogram/dev/api/open-api/login/wx.login.html) 获取 **临时登录凭证code** ，并回传到开发者服务器。（只能使用一次，再次需要重新生成）
> 2. 调用 [auth.code2Session](https://developers.weixin.qq.com/miniprogram/dev/OpenApiDoc/user-login/code2Session.html) 接口，换取用户唯一标识 **OpenID** 、 用户在微信开放平台账号下的唯一标识**UnionID**（若当前小程序已绑定到微信开放平台账号） 和会话密钥 **session_key**。
>
> 以下是微信服务接口调用地址以及请求方式：
>
> ```text
> GET https://api.weixin.qq.com/sns/jscode2session
> ```
>
> 之后开发者服务器可以根据用户标识来生成自定义登录态，用于后续业务逻辑中前后端交互时识别用户身份。

首先在小程序中调用login()方法获取授权码code，将授权码发给到服务端。然后服务端需要向微信服务接口发送登录凭证，最后返回登录用户的唯一标识openid，用于后续业务逻辑中前后端交互时识别用户身份，如为小程序客户端生成token令牌，当小程序要访问服务端时就会携带该token，服务端进行核验，返回业务数据。

![](https://github.com/Andrewmeo/client_images/blob/main/api-login.2fcc9f35.jpg?raw=true)

### 使用PostMan测试以上流程

步骤：

1. 微信小程序客户端调用wx.login()获取code。
2. PostMan利用code访问微信接口服务，传递小程序appid、密钥以及code进行校验。
3. 微信服务接口校验成功后返回用户唯一标识，以及会话密钥。

编译小程序代码，然后调用wx.login()获取到微信用户的授权码。

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231017133401303.png?raw=true)

打开PostMan，粘贴请求路径，设置请求方法，然后设置Query参数。最后发送请求获取返回数据。

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231017133802599.png?raw=true)

### 需求分析

### 开发流程

首先配置小程序的信息以及生成token的配置：

```yaml
wechat: 
	appid: ${ivy.wechat.appdi}
	secret: ${ivy.wechat.secret}
```

```yaml
ivy: 
	jwt: 
		user-secret-key: abcdefgabcdefgabcdefg
		user-ttl: 7200000
		user-token-name: authentication
```

定义DTO接收客户端发送的用户标识：

```java
/**
 * 小程序客户端用户登录
 */
@Data
public class UserLoginDTO implements Serializable {

    private String code;

}
```

定义VO将用户openid以及token返回：

```java
@Data
@Builder
@NoArgsConstructor
@AllArgsConstructor
public class UserLoginVO implements Serializable {

    private Long id;
    private String openid;
    private String token;

}
```

在面向用户端登录的业务中，主要围绕用户对象进行，需要保存用户信息到数据库，以及返回部分信息，以及访问令牌。

#### user/UserController

定义用户控制器，先写一个接收小程序客户端请求登录时发送来的用户标识，我们经过向微信服务接口校验，保存用户信息后，返回用户openid和jwt令牌：

```java
@RestController
@RequestMapping("/user/user")
@Api(tags = "小程序客户端用户接口")
@Slf4j
public class UserController {

    @Autowired
    private JwtProperties jwtProperties;

    @Autowired
    private UserService userService;

    @PostMapping("/login")
    @ApiOperation("微信登录接口")
    public Result<UserLoginVO> wxLogin(@RequestBody UserLoginDTO userLoginDTO){
        // 微信登录
        User user = userService.wxLogin(userLoginDTO);

        // 生成jwt令牌
        Map<String, Object> claims = new HashMap<>();
        claims.put(JwtClaimsConstant.USER_ID, user.getId());
        String token = JwtUtil.createJWT(jwtProperties.getUserSecretKey(), jwtProperties.getUserTtl(), claims);

        UserLoginVO userLoginVO = UserLoginVO.builder()
                .id(user.getId())
                .openid(user.getOpenid())
                .token(token)
                .build();

        return Result.success(userLoginVO);
    }
    
}
```

#### UserServiceImpl

在微信登录的业务方法中，首先向微信服务接口发送了code校验请求，然后获取到用户openid，然后通过openid查询用户，进行判断该用户是否是新用户，是的话进行插入，不是的话直接返回根据openid查询到的用户：

```java
@Service
@Slf4j
public class UserServiceImpl implements UserService {

    public static final String WX_Login = "https://api.weixin.qq.com/sns/jscode2session";

    @Autowired
    private WeChatProperties weChatProperties;

    @Autowired
    private UserMapper userMapper;

    /**
     * 微信小程序登录
     * @param userLoginDTO
     * @return
     */
    public User wxLogin(UserLoginDTO userLoginDTO) {

        // 调用微信服务接口获取openid
        String openid = getOpenId(userLoginDTO.getCode());

        // 判断openid
        if(openid == null){
            // 抛异常
            throw new LoginFailedException(MessageConstant.LOGIN_FAILED);
        }

        // 是否为新用户
        User user = userMapper.getByOpenId(openid);
        if(user == null){
            user = User.builder()
                    .openid(openid)
                    .createTime(LocalDateTime.now())
                    .build();
            userMapper.insert(user);
        }

        return user;
    }

    private String getOpenId(String code){

        Map<String, String> map = new HashMap<>();
        map.put("appid", weChatProperties.getAppid());
        map.put("secret", weChatProperties.getSecret());
        map.put("js_code", code);
        map.put("grant_type", "authorization_code");
        String json = HttpClientUtil.doGet(WX_Login, map);

        // 解析返回的json数据
        JSONObject jsonObject = JSON.parseObject(json);

        return jsonObject.getString("openid");
    }
    
}
```

### 测试

首先看一下实现微信登录业务实现逻辑，发送HTTP请求远程调用微信服务接口进行校验的过程我们封装到了getOpenId中。通过校验获取到openid后向下执行，判断openid是否为空，然后通过openid获取数据库中的用户，判断是否存在该用户，不存在则为新用户，插入用户。存在则直接返回该用户对象。

```java
/**
 * 微信小程序登录
 * @param userLoginDTO
 * @return
 */
public User wxLogin(UserLoginDTO userLoginDTO) {

    // 调用微信服务接口获取openid
    String openid = getOpenId(userLoginDTO.getCode());

    // 判断openid
    if(openid == null){
        // 抛异常
        throw new LoginFailedException(MessageConstant.LOGIN_FAILED);
    }

    // 是否为新用户
    User user = userMapper.getByOpenId(openid);
    if(user == null){
        user = User.builder()
                .openid(openid)
                .build();
        userMapper.insert(user);
    }

    return user;
}
```

Debug方式启动服务端和redis服务，然后在发送HTTP请求远程调用微信服务接口的地方打断点。

编译小程序，小程序启动后自动发送的第一个请求是获取店铺的营业状态：

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231019101645517.png?raw=true)

当点击允许后，小程序调用wx.login()生成授权码code，然后发送请求传递到微信登录接口。

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231019102013185.png?raw=true)

可以看到在发送请求远程调用的业务方法中已经获取到了用户的授权码：

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231019102215370.png?raw=true)

单步调试可以看到已经将小程序id、小程序密钥、授权码code以及授权方式grant_type封装到了哈希集合中。

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231019102403274.png?raw=true)

继续单步调试，已经成功通过了微信服务接口的校验，其响应了一串json数据，包含的是一长串字节数组，需要通过json解析出来，最后得到了该用户的开放标识openid以及会话密钥。

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231019102809246.png?raw=true)

获取到openid后就是进行用户判断了，可以看到通过openid查询数据库返回用户不为null，说明该用户之前登录过。最后直接返回用户对象就行。

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231019103833272.png?raw=true)

最后在用户登录校验成功后，接下来的一般操作就是生成jwt令牌了，可以看到我们将数据库中的用户id作为载荷生成了token。最后我们将用户id、openid以及token封装到了VO对象中响应给客户端小程序。

![](https://github.com/Andrewmeo/client_images/blob/main/image-20231019104116743.png?raw=true)


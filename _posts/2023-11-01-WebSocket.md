---
layout: post
title: WebSocket开发-来单提醒和客户催单
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
tags: 支付成功回调 websocket
top: 19
---

WebSocket 是基于 TCP 的一种新的网路协议。它实现了浏览器与服务器双工通信—浏览器和服务器只需要完成一次握手，两者之间就可以创建持久性的连接，并且进行双向数据传输。

| 通信协议      | 通信方式                | 说明                                                         |
| ------------- | ----------------------- | ------------------------------------------------------------ |
| HTTP协议      | 基于TCP的单向短连接通信 | 就像开始跟对象聊天一样，你一句ta一句，你不发消息，ta也不会主动发给你。 |
| WebSocket协议 | 基于TCP的双向长连接通信 | 对象不再是一个不主动的服务器了，ta遇到特别的事情ta会主动发消息给我了。 |

HTTP协议与WebSocket协议对比：

1. HTTP是短连接，WebSocket是长连接。
2. HTTP通信是单向的，基于请求响应模式。WebSocket支持双向通信。
3. HTTP和WebSocket底层都是TCP连接。

应用场景：网页聊天、股票报价实时更新推送、视频弹幕推送。

## 1 支付成功来单提醒

### 1.2 简单需求分析

用户下单并且支付成功后，需要第一时间通知商家。通知形式比如弹出提示框，语音播报。

### 1.3 开发流程

1. 通过 WebSocket 实现管理客户端页面与服务端保存长连接状态。
2. 当客户支付后，服务端调用WebSocket相关接口实现服务端向管理客户端推送消息。
3. 管理客户端解析服务端推送的消息，判断是来单提醒还是客户催单，并进行相应的消息提示和语言播报。

约定服务端向管理客户端发送的数据格式未JSON，包括type消息类型、orderId订单id、content消息内容。

#### 1.3.1 PayNotifyController

```java
/**
 * 支付回调相关接口
 */
@RestController
@RequestMapping("/notify")
@Slf4j
public class PayNotifyController {
    @Autowired
    private OrderService orderService;
    @Autowired
    private WeChatProperties weChatProperties;

    /**
     * 支付成功回调，该接口方法由微信接口调用
     * @param request
     */
    @RequestMapping("/paySuccess")
    public void paySuccessNotify(HttpServletRequest request, HttpServletResponse response) throws Exception {
        // 1.读取数据
        String body = readData(request);
        log.info("支付成功回调：{}", body);

        // 2.数据解密
        String plainText = decryptData(body);
        log.info("解密后的文本：{}", plainText);

        // 3.解析为Json对象
        JSONObject jsonObject = JSON.parseObject(plainText);
        String outTradeNo = jsonObject.getString("out_trade_no"); // 商户平台订单号
        String transactionId = jsonObject.getString("transaction_id"); // 微信支付交易号

        log.info("商户平台订单号：{}", outTradeNo);
        log.info("微信支付交易号：{}", transactionId);

        // 4.业务处理，传入支付订单号，修改订单状态、来单提醒
        orderService.paySuccess(outTradeNo);

        // 5.给微信服务接口响应
        responseToWeixin(response);
    }

    /**
     * 读取响应数据
     *
     * @param request
     * @return
     * @throws Exception
     */
    private String readData(HttpServletRequest request) throws Exception {
        BufferedReader reader = request.getReader(); // 获取request对象的缓冲输入流对象，提高字符输入流读数据的性能
        StringBuilder result = new StringBuilder();
        String line = null;
        while ((line = reader.readLine()) != null) { // 读取一行数据返回，读取完毕返回null
            if (result.length() > 0) {
                result.append("\n"); // 一行一行的加入到StringBuilder对象中
            }
            result.append(line);
        }
        return result.toString(); // 转化为字符串返回
    }

    /**
     * 数据解密
     *
     * @param body
     * @return
     * @throws Exception
     */
    private String decryptData(String body) throws Exception {
        JSONObject resultObject = JSON.parseObject(body);
        JSONObject resource = resultObject.getJSONObject("resource");
        String ciphertext = resource.getString("ciphertext");
        String nonce = resource.getString("nonce");
        String associatedData = resource.getString("associated_data");

        AesUtil aesUtil = new AesUtil(weChatProperties.getApiV3Key().getBytes(StandardCharsets.UTF_8));
        //密文解密
        String plainText = aesUtil.decryptToString(associatedData.getBytes(StandardCharsets.UTF_8),
                nonce.getBytes(StandardCharsets.UTF_8),
                ciphertext);

        return plainText;
    }

    /**
     * 给微信响应
     * @param response
     */
    private void responseToWeixin(HttpServletResponse response) throws Exception{
        response.setStatus(200);
        HashMap<Object, Object> map = new HashMap<>();
        map.put("code", "SUCCESS");
        map.put("message", "SUCCESS");
        response.setHeader("Content-type", ContentType.APPLICATION_JSON.toString());
        response.getOutputStream().write(JSONUtils.toJSONString(map).getBytes(StandardCharsets.UTF_8));
        response.flushBuffer();
    }

}
```

#### 1.3.2 OrderServiceImpl

```java
    /**
     * 支付成功，修改订单状态
     * @param outTradeNo
     */
    public void paySuccess(String outTradeNo) {

        // 1.根据订单号查询订单
        Orders ordersDB = orderMapper.getByNumber(outTradeNo);

        // 2.根据订单id修改订单的状态、支付方式、支付状态、结账时间，更新到数据库
        Orders orders = Orders.builder()
                .id(ordersDB.getId())
                .status(Orders.TO_BE_CONFIRMED)
                .payStatus(Orders.PAID)
                .checkoutTime(LocalDateTime.now())
                .build();
        
        orderMapper.update(orders);

        // 3.修改订单状态之后，提醒管理客户端处理来单
        Map map = new HashMap();
        map.put("type", 1); // 1表示来单提醒，2表示客户端催单
        map.put("orderId", ordersDB.getId());
        map.put("content", "订单号：" + outTradeNo);

        String message = JSONObject.toJSONString(map); // 将map对象转为json字符串，message中包含推送消息类型，如推送消息内容，要处理订单的订单号

        webSocketServer.sendToAllClient(message); // 调用websocket的群发方法推送消息
    }
```

#### 1.3.3 WebSocketServer

```java
/**
 * WebSocket服务
 */
@Component
@ServerEndpoint("/ws/{sid}") // WebSocket控制器，当客户端登录时自动发送此请求建立长连接
public class WebSocketServer {

    //存放会话对象
    private static Map<String, Session> sessionMap = new HashMap();

    /**
     * 定义消息推送方法，服务端向客户端主动发送消息message
     *
     * @param message
     */
    public void sendToAllClient(String message) {
        Collection<Session> sessions = sessionMap.values();
        for (Session session : sessions) {
            try {
                //服务器通过当前会话向客户端发送消息
                session.getBasicRemote().sendText(message);
            } catch (Exception e) {
                e.printStackTrace();
            }
        }
    }

}
```

## 2 客户催单

### 2.2 简单需求分析

用户在客户端小程序点击催单按钮后，需要第一时间通知外卖商家。通知方式为弹出提示框，语言播报。

客户催单提醒实现逻辑和下单提醒一样，都是可以通过建立WebSocket连接实现。

### 2.3 接口设计

### 2.4 开发流程

#### 2.4.1 OrderController

```java
    @GetMapping("/reminder/{id}")
    @ApiOperation("客户催单")
    public Result reminder(@PathVariable("id") Long id){
        orderService.reminder(id);

        return Result.success();
    }
```

#### 2.4.2 OrderServiceImpl

```java
/**
 * 客户催单
 * @param id
 */
public void reminder(Long id) {

    // 根据id查询订单
    Orders ordersDB = orderMapper.getById(id);

    // 校验订单是否存在
    if (ordersDB == null) {
        throw new OrderBusinessException(MessageConstant.ORDER_STATUS_ERROR);
    }

    // 调用websocket的群发方法推送消息，message中包含推送消息类型，消息内容就是客户催单
    Map map = new HashMap();
    map.put("type", 2);
    map.put("orderId", id);
    map.put("content", "订单号" + ordersDB.getNumber());

    webSocketServer中的方法.sendToAllClient(JSONObject.toJSONString(map)); // 同样调用WebSocketServer中的方法
}
```

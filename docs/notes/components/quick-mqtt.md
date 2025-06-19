---
title: QUICK-MQTT 使用说明文档
createTime: 2025/06/19 10:56:13
permalink: /components/i6pm8ql8/
---
# QUICK-MQTT 使用说明文档

本模块基于 Spring 封装了 MQTT 客户端，支持：

- ✅ 适用于springboot 2.xx
- ✅ 配置文件默认主题订阅
- ✅ 注解方式动态主题注册
- ✅ 统一发布接口

---
## 依赖
```xml
<dependency>
    <groupId>top.hjcwzx.wrench</groupId>
    <artifactId>wrench-kit-starter-quick-mqtt</artifactId>
    <version>1.0.3</version>
</dependency>
```


## 1、配置文件

### 1.1、 配置格式（`application.yml`）

```yaml
wrench:
  mqtt:
    //协议
    agreement: tcp
    ip: 127.0.0.1
    port: 1883
    //客户端ID:注册唯一码
    client-id: wrench-client
    username: test-user
    password: test-pass
    //策略
    strategy-enum: annotation_callback
    //默认主题列表
    default-topics:
      - wrench/topic/default
      - wrench/topic/device-status
```

### 1.2、消息处理策略（ strategy-enum 的配置）

```java
ANNOTATION_CALLBACK("基于注解回调的策略","directCallbackStrategy"),
EVENT_PUBLISH("基于事件发布的策略","eventPublishStrategy"),
MIXED("基于注解回调与事件发布的策略","mixedStrategy");

若配置基于注解回调的策略(annotation_callback),系统启动时会订阅配置文件中定义的default-topics主题列表以及注解配置的主题(下文中解释), 消费时,不会消费配置文件中定义的default-topics的主题，只会消费注解配置的主题，消费方式使用下文中2.1对应的注解式主题消费
    
若配置基于事件发布的策略event_publish的策略,系统启动时仅会订阅配置文件中default-topics主题列表,不会订阅注解配置的主题(下文中解释),消费时,仅消费配置文件中定义的default-topics的主题，不会消费注解配置的主题，消费方式使用下文中2.2对应的消息事件主题消费
    
若配置基于注解回调与事件发布mixed的策略,系统启动时会订阅配置文件中定义的default-topics主题列表以及注解配置的主题(下文中解释), 消费时,会消费配置文件中定义的default-topics的主题，也会消费注解配置的主题，default-topics主题列表的消费方式需采用下文中2.2的消息事件主题消费，注解配置的主题需使用下文中2.1对应的注解式主题消费
```

### 1.3、 默认配置

```yaml
//策略采用注解式配置
strategy-enum: annotation_callback
//默认主题为空
default-topics
```



## 2、消费消息

### 2.1、注解式主题消费（推荐）

#### 2.1.1. 使用 `@MqttSubscribe` 注解监听主题

```java
//例子
@Component
public class MqttMessageListener {

    @MqttSubscribe(topic = "wrench/topic/command", messageType = MyCommandDTO.class)
    public void handleCommand(MyCommandDTO cmd) {
        // 处理命令
        System.out.println("接收到命令: " + cmd);
    }

    @MqttSubscribe(topic = "wrench/topic/raw")
    public void handleRawMessage(String msg) {
        // 处理字符串消息
        System.out.println("接收到原始消息: " + msg);
    }
}
```

#### 2.1.2. 注解参数说明

| 参数名        | 类型       | 描述                          |
| ------------- | ---------- | ----------------------------- |
| `topic`       | `String`   | MQTT 订阅主题（必填）         |
| `messageType` | `Class<?>` | 消息数据类型，默认是 `String` |

#### 2.1.3. 注意事项

该主题订阅方式只能消费@MqttSubscribe注解定义的主题，无法消费配置文件配置的    默认主题列表default-topics的主题



### 2.2、消息事件主题消费（事件驱动）

#### 2.2.1、使用方式

##### 2.2.1.1、自定义监听器实现抽象类AbstractMqttMessageListener，实现方法

MqttMessageEvent中包括topic以及message

##### 2.2.1.2、将自定义的监听器交给string管理

@Component

####  2.2.2、例子

```java
@Component
public class MsgListener extends AbstractMqttMessageListener {
    @Override
    public void doHandler(MqttMessageEvent event) {
        System.out.println(event.getMessage());
        System.out.println(event.getTopic());
    }
}
```

#### 2.2.3. 注意事项

该主题订阅方式只能消费配置文件中定义的主题，无法消费注解式定义的主题



## 3、发布消息

### 3.1. 使用方式

```java
//1、注入MqttPublisher 消息发布器
@Autowired
private MqttPublisher mqttPublisher;

//2、发布消息
mqttPublisher.publish(topic, message);

topic: 主题
message: 消息
```

### 3.2. 例子

```java
@Autowired
private MqttPublisher mqttPublisher;

public void sendMessage() throws Exception {
    MyPayload payload = new MyPayload("foo", 123);
    mqttPublisher.publish("wrench/topic/command", payload);
}
```



## 4、测试建议

1. 使用 MQTT 工具（如 EMQX）测试 Broker 是否正常
2. 测试配置文件默认主题是否生效
3. 在类中添加注解方法，验证注解式注册是否成功
4. 调用 `mqttPublisher.publish()` 测试消息发送功能



## 5、注解式使用方式

### 5.1、配置文件

```yaml
wrench:
  mqtt:
    //协议
    agreement: tcp
    ip: 127.0.0.1
    port: 1883
    //客户端ID:注册唯一码
    client-id: wrench-client
    username: test-user
    password: test-pass
```

### 5.2、消息消费

```java
@Service
public class MqttAccessMessage {


    @MqttSubscribe(topic = "test",messageType = String.class)
    public void a(String msg){
        System.out.println(msg);
    }


    @MqttSubscribe(topic = "testb",messageType = String.class)
    public void B(String msg){
        System.out.println(msg);
    }

}
```

### 5.3、消息发送

```java
@RestController
@RequestMapping("/mqtt")
public class MqttController {

    @Autowired
    private MqttPublisher<String> mqttPublisher;

    @GetMapping("/publish")
    public String publish(String topic) throws MqttException, JsonProcessingException {
        mqttPublisher.publish(topic, "你还哦");
        return "发布成功";
    }

}
```



## 6、事件驱动使用方式

### 6.1、配置文件

```yaml
wrench:
  mqtt:
    //协议
    agreement: tcp
    ip: 127.0.0.1
    port: 1883
    //客户端ID:注册唯一码
    client-id: wrench-client
    username: test-user
    password: test-pass
    strategy-enum: event_publish
```

### 6.2、消息消费

```java
@Component
public class MsgListener extends AbstractMqttMessageListener {
    @Override
    public void doHandler(MqttMessageEvent event) {
        System.out.println(event.getMessage());
        System.out.println(event.getTopic());
    }
}

```

### 6.3、消息发送

```java
@RestController
@RequestMapping("/mqtt")
public class MqttController {

    @Autowired
    private MqttPublisher<String> mqttPublisher;

    @GetMapping("/publish")
    public String publish(String topic) throws MqttException, JsonProcessingException {
        mqttPublisher.publish(topic, "你还哦");
        return "发布成功";
    }

}
```



## 7、混合使用方式

### 7.1、配置文件

```yaml
wrench:
  mqtt:
    //协议
    agreement: tcp
    ip: 127.0.0.1
    port: 1883
    //客户端ID:注册唯一码
    client-id: wrench-client
    username: test-user
    password: test-pass
    strategy-enum: mixed
    default-topics:
        - abc111
        - mmm000
```

### 7.2、消息消费

```java
//消费配置文件式主题
@Component
public class MsgListener extends AbstractMqttMessageListener {
    @Override
    public void doHandler(MqttMessageEvent event) {
        System.out.println(event.getMessage());
        System.out.println(event.getTopic());
    }
}

```

```java
//消费注解式主题
@Service
public class MqttAccessMessage {


    @MqttSubscribe(topic = "test",messageType = String.class)
    public void a(String msg){
        System.out.println(msg);
    }


    @MqttSubscribe(topic = "testb",messageType = String.class)
    public void B(String msg){
        System.out.println(msg);
    }

}
```

### 7.3、消息发送

```java
@RestController
@RequestMapping("/mqtt")
public class MqttController {

    @Autowired
    private MqttPublisher<String> mqttPublisher;

    @GetMapping("/publish")
    public String publish(String topic) throws MqttException, JsonProcessingException {
        mqttPublisher.publish(topic, "你还哦");
        return "发布成功";
    }

}
```





## 8、如有问题欢迎找小胡哥或查看源码 🙌

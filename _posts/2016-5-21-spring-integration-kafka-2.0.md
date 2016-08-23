---
layout: post
title: spring-integration-kafka 2.0 使用
subtitle: spring-integration-kafka
author: kylo
catalog: true
tags:
  - Kafka
  - spring-integration-kafka
---

Spring 终于在 2016 年的 4 月 9 日发布了支持 kafka 0.9 版本的集成框架 [spring-integration-kafka](!https://github.com/spring-projects/spring-integration-kafka)，虽然还只是个里程碑的版本。新版的集成底层使用自家的 [spring-kafka](!https://github.com/spring-projects/spring-kafka)，使用的是 kafka 0.9 提供的新接口，kafka 0.9 不再区分 High Level API 和 Simple Consumer API。

# 打包编译 spring-integration-kafka

由于 2.0 还没有正式 release，在 <http://mvnrepository.com> 上找不到这个版本的 jar 包，所以只能源代码编译了。<br>
按照 [README](!https://github.com/spring-projects/spring-integration-kafka) 上的说明，使用 gradle 进行 build 和 install。同理，spring-kafka 1.0.0.M2 也需要源码编译。

```bash
./gradlew build
./gradlew install
```

国内 gradle 的下载速度不是很理想。

# Outbound Channel Adapter 和 Message Driven Channel Adapter

Outbound Channel Adapter 和 Message Driven Channel Adapter 分别是 spring-integration-kafka 中对应 kafka 的生产者和消费者的适配器。 Outbound Channel Adapter 通过将发送到这个适配器的 Spring 消息(org.springframework.messaging.Message)，在内部构件成 kafka 的消息，然后使用生产者接口发送给 kafka，而 Message Driven Channel Adapter 则接收并转化 kafka 消费者接收到的消息为 Spring 消息。因此，我们的创建、接受的消息内容都是 Spring 消息。

## Outbound Channel Adapter 配置

参考官方给出的 Spring Boot 的例子，进行 Outbound Channel Adapter 的配置。

```java
@Configuration
public class KafkaProducerConfiguration {

    @Value("${kafka.topic}")
    private String topic;

    @Value("${kafka.messageKey}")
    private String messageKey;

    @Value("${kafka.broker.address}")
    private String brokerAddress;

    @Bean
    @ServiceActivator(inputChannel = "inputToKafka", autoStartup="true")
    public MessageHandler handler() throws Exception {
        KafkaProducerMessageHandler<String, String> handler = new KafkaProducerMessageHandler<>(kafkaTemplate());
        handler.setTopicExpression(new SpelExpressionParser().parseExpression("headers.kafka_topic != null ? headers.kafka_topic : '"+ topic +"'"));
        handler.setMessageKeyExpression(new SpelExpressionParser().parseExpression("headers.kafka_messageKey != null ? headers.kafka_messageKey : ''"+ messageKey +"'"));
        return handler;
    }

    @Bean
    public KafkaTemplate<String, String> kafkaTemplate() {
        return new KafkaTemplate<>(producerFactory());
    }

    @Bean
    public ProducerFactory<String, String> producerFactory() {
        Map<String, Object> props = new HashMap()<>;
        props.put(ProducerConfig.BOOTSTRAP_SERVERS_CONFIG, this.brokerAddress);
        props.put(ProducerConfig.RETRIES_CONFIG, 0);
        props.put(ProducerConfig.BATCH_SIZE_CONFIG, 16384);
        props.put(ProducerConfig.LINGER_MS_CONFIG, 1);
        props.put(ProducerConfig.BUFFER_MEMORY_CONFIG, 33554432);
        props.put(ProducerConfig.KEY_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        props.put(ProducerConfig.VALUE_SERIALIZER_CLASS_CONFIG, StringSerializer.class);
        return new DefaultKafkaProducerFactory<>(props);
    }

}
```

使用 `@Configuration` 进行配置，有一个问题还没找到原因。进行上述配置，在使用 Juint 和 Spring Boot 的时候，容器会把 handler 注册成 inputToKafka 的订阅者，但是使用 web.xml 配置的 Spirng 项目的时候并不会注册。 猜测原因：

1. `@ServiceActivator` 没有生效
2. web.xml 配置问题

目前暂时采用 xml 方式配置 Outbound Channel Adapter。 关于 topic 和 messageKey，spring-integration-kafka 提供了表达式对 Message.headers 的内容进行判断，所以生产者的消息要相对应的结构。 比如设置了 `headers.kafka_topic != null ? headers.kafka_topic : 'si.topic'`，这样设置的话，会判断 Message.headers.kafka_topic 的值，如果为空，则设置 topic 为 si.topic。所以，Message.headers Map 一定要存在 kafka_topic 这个 key，不然会报空异常。

```xml
<?xml version="1.0" encoding="UTF-8"?>
<beans xmlns="http://www.springframework.org/schema/beans"
       xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
       xmlns:int-kafka="http://www.springframework.org/schema/integration/kafka"
       xmlns:int="http://www.springframework.org/schema/integration"
       xsi:schemaLocation="http://www.springframework.org/schema/integration/kafka http://www.springframework.org/schema/integration/kafka/spring-integration-kafka.xsd
       http://www.springframework.org/schema/integration http://www.springframework.org/schema/integration/spring-integration.xsd
       http://www.springframework.org/schema/beans http://www.springframework.org/schema/beans/spring-beans.xsd">

    <int:channel id="inputToKafka"/>


    <int-kafka:outbound-channel-adapter id="kafkaOutboundChannelAdapter"
                                        kafka-template="kafkaTemplate"
                                        auto-startup="true"
                                        channel="inputToKafka"
                                        topic-expression="headers.kafka_topic != null ? headers.kafka_topic : '${kafka.topic}'">
    </int-kafka:outbound-channel-adapter>

    <bean id="kafkaTemplate" class="org.springframework.kafka.core.KafkaTemplate">
        <constructor-arg>
            <bean class="org.springframework.kafka.core.DefaultKafkaProducerFactory">
                <constructor-arg>
                    <map>
                        <entry key="bootstrap.servers" value="${kafka.broker.address}"/>
                        <entry key="retries" value="10"/>
                        <entry key="batch.size" value="16384"/>
                        <entry key="linger.ms" value="1"/>
                        <entry key="buffer.memory" value="33554432"/>
                        <entry key="key.serializer" value="org.apache.kafka.common.serialization.StringSerializer"/>
                        <entry key="value.serializer" value="org.apache.kafka.common.serialization.StringSerializer"/>
                        <entry key="value.serializer" value="org.apache.kafka.common.serialization.StringSerializer"/>
                    </map>
                </constructor-arg>
            </bean>
        </constructor-arg>
    </bean>
</beans>
```

## Message Driven Channel Adapter 配置

消费者采用 `@Configuration` 并没有出现类似生产者的问题。

```java
@Configuration
public class KafkaConsumerConfiguration extends AbstractKafkaConsumerConfiguration {

    @Value("${kafka.broker.address}")
    private String brokerAddress;

    @Bean
    public KafkaMessageListenerContainer<String, String> container() throws Exception {
        Map<String, Object> props = new HashMap()<>;
        props.put(ConsumerConfig.GROUP_ID_CONFIG, "testGroup");
        props.put(ConsumerConfig.BOOTSTRAP_SERVERS_CONFIG, this.brokerAddress);
        props.put(ConsumerConfig.ENABLE_AUTO_COMMIT_CONFIG, true);
        props.put(ConsumerConfig.AUTO_COMMIT_INTERVAL_MS_CONFIG, 100);
        props.put(ConsumerConfig.SESSION_TIMEOUT_MS_CONFIG, 15000);
        props.put(ConsumerConfig.KEY_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);
        props.put(ConsumerConfig.VALUE_DESERIALIZER_CLASS_CONFIG, StringDeserializer.class);

        return new KafkaMessageListenerContainer<>(new DefaultKafkaConsumerFactory<>(props), "test.topic");
    }

    @Bean
    public KafkaMessageDrivenChannelAdapter<String, String>
    adapter(KafkaMessageListenerContainer<String, String> container) {
        KafkaMessageDrivenChannelAdapter<String, String> kafkaMessageDrivenChannelAdapter =
                new KafkaMessageDrivenChannelAdapter<>(container);
        kafkaMessageDrivenChannelAdapter.setOutputChannel(received());
        return kafkaMessageDrivenChannelAdapter;
    }

    @Bean
    public PollableChannel received() {
        return new QueueChannel();
    }
}
```

# 生产者和消费者使用

生产者和消费者的使用都非常简单。

- 生产者只需要将 GenericMessage 对象作为参数，传入 MessageChannel.send(Message<?> message) 方法即可；

  ```java
  MessageChannel inputToKafka = context.getBean("inputToKafka", MessageChannel.class);
  inputToKafka.send(new GenericMessage<>("foo"));
  ```

- 消费者只需要循环接收 PollableChannel.receive() 方法，然后对获得的信息进行业务处理。

  ```java
  PollableChannel fromKafka = context.getBean("received", PollableChannel.class);
  Message<?> received = fromKafka.receive(10000);
  while (received != null) {
    System.out.println(received);
    received = fromKafka.receive(10000);
  }
  ```

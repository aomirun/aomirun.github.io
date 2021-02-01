---
title: "实现NSQ的pub/sub模式，监听者把数据存储到数据库(微服务系列第五天)"
date: 2021-02-01T18:14:10+08:00
draft: true
toc: true
categories: 
  - java
images:
series:
  - hello java
  - 微服务
  - tars
tags: 
  - java
  - docker
  - mq
  - jms
  - tars
  - micro serivce
  - 微服务
---

## 前言
今天的目标是： 实现收到MQ队列消息，转发到NSQ topic上面，由topic消费者把数据保存到数据库中。为啥要这么做呢，因为这样MQ消息服务可以和数据库服务解耦，功能都可以独立部署，互不影响。再则IBM MQ确实不便宜，也比较重，我自己则更偏爱轻量点的服务，所以就再做一个基于大名鼎鼎的nsq的服务。

## docker
### 添加nsq服务
docker-compose.yml
```yml
  nsqlookupd:
    image: nsqio/nsq
    command: /nsqlookupd
    networks:
      tars:
        ipv4_address: 172.25.0.211
    ports:
      - "4160"
      - "4161"
  nsqd:
    image: nsqio/nsq
    command: /nsqd --lookupd-tcp-address=nsqlookupd:4160
    networks:
      tars:
        ipv4_address: 172.25.0.213
    depends_on:
      - nsqlookupd
    ports:
      - "4150"
      - "4151"
  nsqadmin:
    image: nsqio/nsq
    command: /nsqadmin --lookupd-http-address=nsqlookupd:4161
    networks:
      tars:
        ipv4_address: 172.25.0.210
    depends_on:
      - nsqlookupd  
    ports:
      - "4171:4171"
```
### 启动服务
```sh
$ docker-compose up -d
```
## 修改mqserver项目
### pom.xml 添加依赖
```xml
		<dependency>
			<groupId>com.sproutsocial</groupId>
			<artifactId>nsq-j</artifactId>
			<version>1.2</version>
		</dependency>
```
### 添加配置文件
```ini
# nsq config
nsq.nsqlookupd-host=172.25.0.211
nsq.nsqlookupd-slave-host=172.25.0.212
nsq.node=172.25.0.213:4150
```
### 添加配置类
src/main/java/com/example/tarsmqserver/domain/NsqConfig.java
```java
@Component
public class NsqConfig {
    
    @Value("${nsq.nsqlookupd-host}")
    private String nsqlookupd;

    @Value("${nsq.nsqlookupd-slave-host}")
    private String nsqlookupdSlave;

    @Value("${nsq.node}")
    private String node;

    private List<String> host;

    public List<String> getHost() {
        return host;
    }

    public String getNsqlookupd() {
        return nsqlookupd;
    }

    public String getNsqlookupdSlave() {
        return nsqlookupdSlave;
    }

    public String getNode() {
        return node;
    }
}
```
### 修改 mqserver
src/main/java/com/example/tarsmqserver/service/mqserver/ConsumerListener.java
```java
    @Autowired
    private NsqConfig nsqConfig;

    /**
     * 使用JmsListener配置消费者监听的队列
     * 
     * @param receivedMsg 接收到的消息
     */
    @JmsListener(destination = "#{@conf.recvQueueName}")
    public void receiveQueue(String receivedMsg) {
        JMS_LISTENER.info("RECEIVE: {}", receivedMsg);
        Publisher publisher = new Publisher(nsqConfig.getNode()); 

        byte[] data = "Hello nsq".getBytes();
        publisher.publishBuffered("example_topic", data);

    }
```
### 打包/上传/测试/观察
```sh
$ make build-upload
```
随意发了一个消息，返回如下
```log
2021-02-01 19:38:57.002 INFO 3376 --- [ageObjAdapter-1] MQSERVER : send: DEV.QUEUE.1 -> qqqqqqqqqq
2021-02-01 19:38:57.148 INFO 3376 --- [enerContainer-1] JMS_LISTENER : RECEIVE: qqqqqqqqqq
2021-02-01 19:38:57.465 INFO 3376 --- [ nsq-batch-0] com.sproutsocial.nsq.Connection : connected 172.25.0.213:4150 msgTimeout:60000 heartbeatInterval:30000 maxRdyCount:2500
2021-02-01 19:38:57.467 INFO 3376 --- [ nsq-batch-0] com.sproutsocial.nsq.Publisher : publisher connected:172.25.0.213:4150
```
代表已发布成功
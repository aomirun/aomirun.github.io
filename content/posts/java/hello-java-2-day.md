---
title: "Hello Java 2 Day 学习JAVA第2天 实现基于tars springboot ibm mq的微服务"
date: 2021-01-27T12:37:57+08:00
draft: false
toc: true
categories: 
  - java
images:
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
第1天学习了java的项目创建，打包，上传到tars中，收获良多，今天折腾一下项目中需要用到的IBM MQ消息服务,实现基于tars springboot ibm mq的微服务.

## 搭建IBM MQ
同样还是使用docker来创建，添加ibmmq
```sh
$ docker volume create qm1data  # ibm mq 有些权限要求，配置不当有可能启动不了，所以使用卷的方式比较简单
```
docker-compose.yml 中添加内容如下：
```yml
  ibmmq:
    image: ibmcom/mq
    container_name: ibmmq
    volumes:
      - qm1data:/mnt/mqm
    environment:
      LICENSE: "accept"
      MQ_QMGR_NAME: "QM1"
      MQ_APP_PASSWORD: "abcd1234"
    ports:
      - "1414:1414"
      - "9443:9443"
    networks:
      tars:
        ipv4_address: 172.25.0.201

volumes:
  qm1data:
   
```
另外给node一个web服务端口，后续会用上
```yml
  node:
    # image: tarscloud/tars-node:stable
    image: tarscloud/tars-node:latest
    container_name: tars-node
    # restart: always
    ports:
      - "1080:1080"
    networks:
      tars:
        ipv4_address: 172.25.0.5
    volumes:
      - ./node/data:/data/tars:rw
      - /etc/localtime:/etc/localtime
    environment:
      INET: eth0
      WEB_HOST: http://172.25.0.3:3000
    depends_on:
      - framework
```
运行起来
```sh
$ docker-compose up -d

Creating tars-mysql ... done
Creating ibmmq      ... done
Creating tars-framework ... done
Creating tars-node      ... done
```
打开浏览器访问一下，并做相应的配置 https://localhost:9443 `重要！！！ 9443是SSL端口，一定需要使用https来访问`,好像ibmmq这个版本的web服务有bug,重启Docker后，会占用端口，之后就启动不了了，原因没有深究，测试的话，默认创建好后就可以使用，要不要Web管理端都无所谓。
### IBM MQ登录页
![](/images/posts/mq/ibm-mq-index.png)
输入账号密码登录，默认账号密码是： `admin/passw0rd`
### IBM MQ登录后的欢迎页
![](/images/posts/mq/ibm-mq-admin-index.png)
### 版本信息
![](/images/posts/mq/ibm-mq-version.png)
新版本的IBM MQ的界面还是很漂亮的


## 测试IBM MQ
添加ibmmq的依赖
### pom.xml
```xml
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>com.fasterxml.jackson.core</groupId>
			<artifactId>jackson-databind</artifactId>
		</dependency>
		<dependency>
			<groupId>com.ibm.mq</groupId>
			<artifactId>mq-jms-spring-boot-starter</artifactId>
			<version>2.4.2</version>
		</dependency>
```
### 添加配置项
application.properties
spring boot 默认都是开启8080端口，因为我开发机上要运行多个，所以改下端口。
```ini
# tomcat default 8080
server.port=1080

# ibm mq config
ibm.mq.queueManager=QM1
ibm.mq.channel=DEV.APP.SVRCONN
ibm.mq.connName=172.25.0.201(1414)
ibm.mq.user=app
ibm.mq.password=abcd1234
ibm.mq.sendQueueName=DEV.QUEUE.1
ibm.mq.recvQueueName=DEV.QUEUE.1
```
### 修改main
TarsMqServerApplication
这里需要用到spring boot 的web服务，所以这里恢复到项目初始状态，上传tars时，协议先非tars协议即可。
```java
@SpringBootApplication
@EnableTarsServer
public class TarsMqServerApplication {

	public static void main(String[] args) {
		SpringApplication.run(TarsMqServerApplication.class, args);
		// 关闭 spring boot 自带的web服务 目前场景只用到了rpc服务
		// SpringApplication app = new SpringApplication(TarsMqServerApplication.class);
		// app.setWebApplicationType(WebApplicationType.NONE);
		// app.run(args);
	}

}
```
### 添加Rest控制器
这里先做个简单的测试，主要目的是用最简单的代码测试ibmmq的连通性
src/main/java/com/example/tarsmqserver/web/MQController.java
```java
@RestController
@EnableJms
@RequestMapping("/api/mq")
public class MQController { 
    @Autowired
	private JmsTemplate jmsTemplate;

	@GetMapping("send") 
	String send() {
		try {
			jmsTemplate.convertAndSend("DEV.QUEUE.1", "Hello World!");
			return "OK";
		} catch (JmsException ex) {
			ex.printStackTrace();
			return "FAIL";
		}
	}

	@GetMapping("recv")
	String recv() {
		try {
			return jmsTemplate.receiveAndConvert("DEV.QUEUE.1").toString();
		} catch (JmsException ex) {
			ex.printStackTrace();
			return "FAIL";
		}
	}
    
}
```
### 提交tars-web测试
#### 打包
```sh
$ mvn clean package
```
#### 上传
上传办法详细看第1节 

#### 测试
```sh
$ curl http://localhost:1080/api/mq/recv
Hello World!%

$ curl http://localhost:1080/api/mq/send
OK%
```
看到上面的信息，证明IBM MQ服务正常跑起来了，jms测试代码也起了作用，后面就是改良代码了。

## JMS
### 服务介绍
关于JMS到网上抄了一份作业,了解一下运行逻辑。
![](/images/posts/mq/jms.png)
由于刚接触Java，到网上找了许多资料，我剪裁了一下，最后模样如下
### 开始打造基于tars的JMS服务了
### ibmmq 绑定配置文件
目录建得比较随意，网上抄作业看到有人用这种目录方式，就直接抄了。
src/main/java/com/example/tarsmqserver/domain/JmqConfig.java
```java
@Component
public class JmqConfig {

    @Value("${ibm.mq.sendQueueName}")
    private String sendQueueName;

    @Value("${ibm.mq.recvQueueName}")
    private  String recvQueueName;


    public String getSendQueueName() {
        return sendQueueName;
    }

    public String getRecvQueueName(){
        return recvQueueName;
    }

}
```
### 消息生产者
src/main/java/com/example/tarsmqserver/service/mqserver/Producer.java
```java
@Service("producer")
@EnableJms
public class Producer {
    /**
     * 也可以注入JmsTemplate，JmsMessagingTemplate对JmsTemplate进行了封装
     */
    @Autowired
    private JmsMessagingTemplate jmsTemplate;

    /**
     * 发送消息，destinationName 是IBM MQ 发送队列的名称，message是待发送的消息
     *
     * @param destinationName
     * @param message
     */
    public boolean sendMessage(String destinationName, String message) {
        boolean ok = false;
        try {
            jmsTemplate.convertAndSend(destinationName, message);
            ok = true;
        } catch (JmsException ex) {
            ex.printStackTrace();
            ok = false;
        }
        return ok;
    }
}
```
### 消息消费者
src/main/java/com/example/demo/jmq/ConsumerListener.java
```java
@Component
public class ConsumerListener extends MessageListenerAdapter {
    private static Logger logger = LoggerFactory.getLogger(ConsumerListener.class);

    /**
     * 使用JmsListener配置消费者监听的队列
     * 
     * @param receivedMsg 接收到的消息
     */
    @JmsListener(destination = "#{@conf.recvQueueName}")
    public void receiveQueue(String receivedMsg) {
        logger.info("Consumer收到的报文为: {}", receivedMsg);
    }

    @Bean
    public JmqConfig conf() {
        return new JmqConfig();
    }
}
```
### 默认入口
关闭本地tomcat服务，只开启rpc服务端口
src/main/java/com/example/demo/DemoApplication.java
```java
@SpringBootApplication
@EnableTarsServer
public class TarsMqServerApplication {

	public static void main(String[] args) {
		// 关闭 spring boot 自带的web服务 目前场景只用到了rpc服务
		SpringApplication app = new SpringApplication(TarsMqServerApplication.class);
		app.setWebApplicationType(WebApplicationType.NONE);
		app.run(args);
	}
}
```
### 完善tars接口
用的是上一章定义的接口
tars接口定义传送门 [第1天 通过docker整合springboot和tars](/posts/java/hello-java-1-day)
src/main/java/com/example/tarsmqserver/service/mqserver/impl/MessageServantImpl.java
```java
@TarsServant("messageObj")
@Component
public class MessageServantImpl implements MessageServant {
    @Autowired
    private Producer producer;

    @Autowired
    private JmqConfig jmqConfig;

    /**
     * 发送消息到IBM MQ
     */
    @Override
    public boolean send(String msg) {
        if (StringUtils.isEmpty(msg)) {
            return false;
        }

        String queueName = jmqConfig.getSendQueueName();
        return producer.sendMessage(queueName, msg);
    }

    @Override
    public boolean encode(String sign, Holder<String> enStr) {
        // TODO 先简单走流程 后面实现具体加密签名之类的
        if (sign.length() == 0) {
            return false;
        }
        enStr.setValue("123" + sign + "456");
        return true;
    }

    @Override
    public boolean encodeWithSend(String msg, String sign) {
        if (StringUtils.isEmpty(msg) || StringUtils.isEmpty(sign)) {
            return false;
        }

        Holder<String> enStr = new Holder<String>();
        boolean ok = encode(sign, enStr);
        if (!ok) {
            return false;
        }

        return send(enStr.getValue() + msg);
    }
}
```
### 生成jar包/上传/测试/观察
```sh
$ mvn clean package
```
查看日志，有类似信息
```log
2021-01-29 13:14:34.416 INFO 4474 --- [enerContainer-1] c.e.t.service.mqserver.ConsumerListener : Consumer收到的报文为: 123中文456aaaaaaaaaaaaaaa
2021-01-29 13:14:56.194 INFO 4474 --- [enerContainer-1] c.e.t.service.mqserver.ConsumerListener : Consumer收到的报文为: 春树暮云
```
整合完成

## 汇总
### 最后项目目录
```sh
$ tree
.
├── mvnw
├── mvnw.cmd
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── example
    │   │           └── tarsmqserver
    │   │               ├── TarsMqServerApplication.java
    │   │               ├── domain
    │   │               │   └── JmqConfig.java
    │   │               ├── service
    │   │               │   └── mqserver
    │   │               │       ├── ConsumerListener.java
    │   │               │       ├── MessageServant.java
    │   │               │       ├── Producer.java
    │   │               │       └── impl
    │   │               │           └── MessageServantImpl.java
    │   │               └── web
    │   │                   └── MQController.java
    │   └── resources
    │       ├── application.properties
    │       └── mqserver.tars
```
### 最后pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
	xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.4.2</version>
		<relativePath/> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.example</groupId>
	<artifactId>tars-mq-server</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>tars-mq-server</name>
	<description>Demo project for Spring Boot</description>
	<properties>
		<java.version>1.8</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>

		<dependency>
			<groupId>com.tencent.tars</groupId>
			<artifactId>tars-spring-boot-starter</artifactId>
			<version>1.7.2</version>
		</dependency>

		<dependency>
			<groupId>com.ibm.mq</groupId>
			<artifactId>mq-jms-spring-boot-starter</artifactId>
			<version>2.4.2</version>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
	</dependencies>

	<build>
		<finalName>mqserver</finalName>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
			<!--tars2java plugin -->
			<plugin>
				<groupId>com.tencent.tars</groupId>
				<artifactId>tars-maven-plugin</artifactId>
				<version>1.7.2</version>
				<configuration>
					<tars2JavaConfig>
						<!-- tars file location -->
						<tarsFiles>
							<tarsFile>${basedir}/src/main/resources/mqserver.tars</tarsFile>
						</tarsFiles>
						<!-- Source file encoding -->
						<tarsFileCharset>UTF-8</tarsFileCharset>
						<!-- Generate server code -->
						<servant>true</servant>
						<!-- Generated source code encoding -->
						<charset>UTF-8</charset>
						<!-- Generated source code directory -->
						<srcPath>${basedir}/src/main/java</srcPath>
						<!-- Generated source code package prefix -->
						<packagePrefixName>com.example.tarsmqserver.service.</packagePrefixName>
					</tars2JavaConfig>
				</configuration>
			</plugin>
			<!--package plugin-->
		</plugins>
	</build>

</project>
```
## 源代码
[学习JAVA第2天 实现基于tars springboot ibm mq的微服务](https://github.com/aomirun/demo/tree/main/day-2)

## 结语
由于才接触java，springboot很多功能都是通过java专有的注解完成的。学习时间有限，很多东西还没有理解，中间调试的时候花了不少时间去查资料和调试。更因为我是使用vscode+tars+springboot来弄的，发生了错误也不知道哪里出问题了。搜索的资料只能做参考，然后自己消化学习，最终调试完成。
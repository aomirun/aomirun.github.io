---
title: "Hello Java 2 Day 学习JAVA第2天"
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
第1天学习了java的项目创建，打包，上传到tars中，收获良多，今天折腾一下项目中需要用到的IBM MQ消息服务
> 本章与第1章项目有所不同，还是因为中间调试出了许多问题，后来重新建了项目后，才完成，后续有空我把两章的代码弄成同一个项目，便于连续的文章

## 搭建IBM MQ
同样还是使用docker来创建
```sh
$ docker volume create qm1data  # ibm mq 有些权限要求，配置不当有可能启动不了，所以使用卷的方式比较简单

$ mkdir ~/docker-app/jmq -p
$ cd ~/docker-app/jmq 
$ touch docker-compose.yml
```
docker-compose.yml 的内容如下：
```yml
version: '3'
services:
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

networks:
  tars:
    external: true     

volumes:
  qm1data:
   
```
运行起来
```sh
$ docker-compose up -d

Status: Downloaded newer image for ibmcom/mq:latest
Creating ibmmq ... done
```
打开浏览器访问一下，并做相应的配置 https://localhost:9443 `重要！！！ 9443是SSL端口，一定需要使用https来访问`
### IBM MQ登录页
![](/images/posts/mq/ibm-mq-index.png)
输入账号密码登录，默认账号密码是： `admin/passw0rd`
### IBM MQ登录后的欢迎页
![](/images/posts/mq/ibm-mq-admin-index.png)
### 版本信息
![](/images/posts/mq/ibm-mq-version.png)
新版本的IBM MQ的界面还是很漂亮的


## 测试IBM MQ
先弄一个Demo代码，测试一下IBM MQ是否正常运行
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
### application.properties
```ini
ibm.mq.queueManager=QM1
ibm.mq.channel=DEV.APP.SVRCONN
ibm.mq.connName=localhost(1414)
ibm.mq.user=app
ibm.mq.password=abcd1234
```
### DemoApplication
```java
@SpringBootApplication
@RestController
@EnableJms
public class DemoApplication {

	@Autowired
	private JmsTemplate jmsTemplate;

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}

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
### 运行程序
```sh
2021-01-28 12:25:14.847  INFO 26166 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2021-01-28 12:25:15.219  INFO 26166 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2021-01-28 12:25:15.232  INFO 26166 --- [           main] com.example.demo.DemoApplication         : Started DemoApplication in 3.663 seconds (JVM running for 4.099)
2021-01-28 12:25:21.125  INFO 26166 --- [nio-8080-exec-1] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring DispatcherServlet 'dispatcherServlet'
2021-01-28 12:25:21.125  INFO 26166 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Initializing Servlet 'dispatcherServlet'
2021-01-28 12:25:21.125  INFO 26166 --- [nio-8080-exec-1] o.s.web.servlet.DispatcherServlet        : Completed initialization in 0 ms
```
访问Web方法 通过Curl或浏览器都行
```sh
$ curl http://localhost:8080/send
OK%

$ curl http://localhost:8080/recv
Hello World!%
```
看到上面的信息，证明IBM MQ服务正常跑起来了，jms测试代码也起了作用，后面就是改良代码了。
## JMS服务
关于JMS到网上抄了一份作业,了解一下运行逻辑。
![](/images/posts/mq/jms.png)
由于刚接触Java，到网上找了许多资料，我剪裁了一下，最后模样如下
### pom.xml
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
	<artifactId>demo</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>demo</name>
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
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
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
		<!-- log -->
        <!-- https://mvnrepository.com/artifact/org.slf4j/slf4j-api -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-api</artifactId>
            <version>1.7.25</version>
        </dependency>
        <!-- https://mvnrepository.com/artifact/org.slf4j/slf4j-log4j12 -->
        <dependency>
            <groupId>org.slf4j</groupId>
            <artifactId>slf4j-log4j12</artifactId>
            <version>1.7.25</version>
            <!-- <scope>test</scope> -->
        </dependency>
	</dependencies>

	<build>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
		</plugins>
	</build>

</project>
```
### 配置文件
src/main/resources/application.properties
```ini
ibm.mq.queueManager=QM1
ibm.mq.channel=DEV.APP.SVRCONN
ibm.mq.connName=localhost(1414)
ibm.mq.user=app
ibm.mq.password=abcd1234

ibm.mq.sendQueueName=DEV.QUEUE.1
ibm.mq.recvQueueName=DEV.QUEUE.1
```
### 配置类
src/main/java/com/example/demo/jmq/JmqConfig.java
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
src/main/java/com/example/demo/jmq/Producer.java
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
    public boolean sendMessage(String destinationName, final String message) {
        boolean ok = false;
		try {
            jmsTemplate.convertAndSend(destinationName, message);
			ok=true;
		} catch (JmsException ex) {
			ex.printStackTrace();
			ok=false;
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
src/main/java/com/example/demo/DemoApplication.java
```java
@SpringBootApplication
public class DemoApplication {

	public static void main(String[] args) {
		SpringApplication.run(DemoApplication.class, args);
	}

}
```
### 测试程序
src/test/java/com/example/demo/DemoApplicationTests.java
```java
@SpringBootTest
class DemoApplicationTests {

	@Autowired
	private Producer producer; 

	@Autowired
    private JmqConfig jmqConfig;

	@Test
	public void jmsIBMMqTest() throws InterruptedException {
		String destination = jmqConfig.getSendQueueName();

		for (int i = 0; i < 21; i++) {
			producer.sendMessage(destination, String.format("My name is aomi%s", i));
		}
	}

	@Test
	void contextLoads() {
	}

}
```
### 测试
由于我对Java测试不熟，我看到maven在打包前会自动运行测试用例，那我就直接打包。
```sh
$ mvn clean package

2021-01-28 16:43:38.258  INFO 7834 --- [           main] com.example.demo.DemoApplicationTests    : Started DemoApplicationTests in 5.512 seconds (JVM running for 7.027)
2021-01-28 16:43:38.934  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi0
2021-01-28 16:43:38.945  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi1
2021-01-28 16:43:38.949  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi2
2021-01-28 16:43:38.953  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi3
2021-01-28 16:43:38.956  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi4
2021-01-28 16:43:38.960  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi5
2021-01-28 16:43:38.964  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi6
2021-01-28 16:43:38.966  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi7
2021-01-28 16:43:38.968  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi8
2021-01-28 16:43:38.971  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi9
2021-01-28 16:43:38.975  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi10
2021-01-28 16:43:38.978  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi11
2021-01-28 16:43:38.980  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi12
2021-01-28 16:43:38.983  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi13
2021-01-28 16:43:38.985  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi14
2021-01-28 16:43:38.988  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi15
2021-01-28 16:43:38.991  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi16
2021-01-28 16:43:38.994  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi17
2021-01-28 16:43:38.996  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi18
2021-01-28 16:43:38.998  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi19
2021-01-28 16:43:39.001  INFO 7834 --- [enerContainer-1] com.example.demo.jmq.ConsumerListener    : Consumer收到的报文为: My name is aomi20
[INFO] Tests run: 2, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 7.249 s - in com.example.demo.DemoApplicationTests
```
看到可以收发消息了，接下来就是把JMS整合到tars中去了，整合完成则后面就是丰富功能了。
## 整合tars
实现接口中，加入今天写的JMS服务。有关tars与java的整合介绍走传送门 [第1天](/posts/java/hello-java-1-day)
### tars接口定义
以昨天的文件为例，直接拿过来
```
module demo {
    //消息发送接口
    interface Message{
        bool Send(string msg);                          //接收字符串，把整个字符串发送到IBM MQ中，返回 true/false
        string Hash(string plain);                      //接收明文字符串，返回加密后的字符串
        bool HashWithSend(string msg, string plain);    //接收明文字符串，加密后直接发送到IBM MQ
    };
};
```
### 生成java接口文件
```sh
$ mvn tars:tars2java
```
### 实现接口
src/main/java/com/example/demo/impl/MessageServantImpl.java
```java
@Component
@TarsServant("MsgObj")
public class MessageServantImpl implements MessageServant {

    @Autowired
    private Producer producer;

    @Autowired
    private JmqConfig jmqConfig;

    /**
     * 发送消息到IBM MQ
     */
    @Override
    public boolean Send(String message) {
        String destination = jmqConfig.getSendQueueName();
        return producer.sendMessage(destination, message);
    }

    /**
     * 对plain进行加密，返回加密后的内容
     */
    @Override
    public String Hash(String plain) {
        if (plain.length() == 0) {
            return null;
        }
        String encodeStr = "1231#!@#!";
        return encodeStr;

    }

    /**
     * 对plain进行加密，并同msg一起发送到IBM MQ
     */
    @Override
    public boolean HashWithSend(String msg, String plain) {
        if (msg.length() == 0 || plain.length() == 0) {
            return false;
        }

        String hashStr = this.Hash(plain);
        boolean ok = this.Send(hashStr + msg);
        return ok;
    }
}
```
### 修改IBM MQ的IP配置
tars的环境与IBM MQ在同一个网段中，改成内网IP
```ini
ibm.mq.connName=172.25.0.201(1414)
```
### 生成jar包
> 先要注解或删除掉`src/test/java/com/example/demo/DemoApplicationTests.java`文件，因为缺少tars专有的配置文件等，以后再来研究基于tars的本地测试。
```sh
$ mvn clean package
```
### 上传jar包
由于我在tars的obj定义的是 `@TarsServant("MsgObj")` 所以对应的obj要改成MsgObj，服务名称要和上传的jar包名相同，具体详细的要去看tars官方文档
### 在线测试
上传demo.tars即可，最后测试结果
![](/images/posts/mq/demo-msg-test.png)

查看日志文件
```log
[dule-executor-1] TARS_CLIENT_LOGGER : ServantNodeRefresher run(tars.tarsregistry.QueryObj), use: 1
[enerContainer-1] com.example.demo.jmq.ConsumerListener : Consumer收到的报文为: 1231#!@#!1211
```
整合完成

## 汇总
### 最后项目目录
```sh
$ tree
.
├── HELP.md
├── mvnw
├── mvnw.cmd
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── com
│   │   │       └── example
│   │   │           └── demo
│   │   │               ├── DemoApplication.java
│   │   │               ├── MessageServant.java
│   │   │               ├── MsgServant.java
│   │   │               ├── impl
│   │   │               │   └── MessageServantImpl.java
│   │   │               └── jmq
│   │   │                   ├── ConsumerListener.java
│   │   │                   ├── JmqConfig.java
│   │   │                   └── Producer.java
│   │   └── resources
│   │       ├── application.properties
│   │       └── demo.tars
```
### 最后pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0" xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.4.2</version>
		<relativePath /> <!-- lookup parent from repository -->
	</parent>
	<groupId>com.example</groupId>
	<artifactId>demo</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>demo</name>
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
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-web</artifactId>
		</dependency>
		<dependency>
			<groupId>com.fasterxml.jackson.core</groupId>
			<artifactId>jackson-databind</artifactId>
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
		<!-- log -->
		<!-- https://mvnrepository.com/artifact/org.slf4j/slf4j-api -->
		<dependency>
			<groupId>org.slf4j</groupId>
			<artifactId>slf4j-api</artifactId>
			<version>1.7.25</version>
		</dependency>
		<!-- https://mvnrepository.com/artifact/org.slf4j/slf4j-log4j12 -->
		<dependency>
			<groupId>org.slf4j</groupId>
			<artifactId>slf4j-log4j12</artifactId>
			<version>1.7.25</version>
			<!-- <scope>test</scope> -->
		</dependency>
	</dependencies>

	<build>
		<finalName>demo</finalName> 
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
							<tarsFile>${basedir}/src/main/resources/demo.tars</tarsFile>
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
						<packagePrefixName>com.example.</packagePrefixName>
					</tars2JavaConfig>
				</configuration>
			</plugin>
			<!--package plugin-->

		</plugins>
	</build>

</project>
```

## 结语
由于才接触java，springboot很多功能都是通过java专有的注解完成的。学习时间有限，很多东西还没有理解，中间调试的时候花了不少时间去查资料和调试。更因为我是使用vscode+tars+springboot来弄的，发生了错误也不知道哪里出问题了。搜索的资料只能做参考，然后自己消化学习，最终调试完成。
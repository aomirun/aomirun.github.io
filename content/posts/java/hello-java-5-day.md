---
title: "实现收到IBMMQ的数据之后存储到数据库(微服务系列第五天)"
date: 2021-02-01T18:14:10+08:00
draft: false
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
今天的目标是： 实现收到MQ队列消息，实现收到IBMMQ的数据之后存储到数据库。
> 注意!!! 这是个学习使用mybatis的示例工程,正式项目则不应该如此做,耦合很严重.正确的做法是把消息发送到topic上面,由存储服务去topic上面取数据再做保存,这样就可以解耦了.

## 修改mqserver项目
### pom.xml
#### 添加依赖
此处用的是阿里的连接池 druid
```xml
		<dependency>
			<groupId>org.mybatis.spring.boot</groupId>
			<artifactId>mybatis-spring-boot-starter</artifactId>
			<version>2.1.4</version>
		</dependency>
		<dependency>
			<groupId>mysql</groupId>
			<artifactId>mysql-connector-java</artifactId>
			<scope>runtime</scope>
		</dependency>
		<dependency>
			<groupId>com.alibaba</groupId>
			<artifactId>druid</artifactId>
			<version>1.2.4</version>
		</dependency>
```
### JDBC 配置
这个是从官网例子上拿来的配置数据
```ini

# 只有下面三个是必填项（使用内嵌数据库的话这三个也可以不用填，会使用默认配置），其他配置不是必须的
spring.datasource.url=jdbc:mysql://172.25.0.2:3306/db1
spring.datasource.username=root
spring.datasource.password=123456
# driver-class-name 非必填可根据url推断
spring.datasource.driver-class-name=com.mysql.cj.jdbc.Driver

# Druid 数据源配置，继承spring.datasource.* 配置，相同则覆盖
spring.datasource.initial-size=2
spring.datasource.max-active=30
spring.datasource.min-idle=2
spring.datasource.max-wait=1234
spring.datasource.pool-prepared-statements=true
spring.datasource.max-pool-prepared-statement-per-connection-size=5
# spring.datasource.max-open-prepared-statements= #等价于上面的max-pool-prepared-statement-per-connection-size
spring.datasource.validation-query=select 1
spring.datasource.validation-query-timeout=1
spring.datasource.test-on-borrow=true
spring.datasource.test-on-return=true
spring.datasource.test-while-idle=true
spring.datasource.time-between-eviction-runs-millis=10000
spring.datasource.min-evictable-idle-time-millis=30001
spring.datasource.async-close-connection-enable=true


# spring.datasource.aop-patterns=com.alibaba.spring.boot.demo.service.*

# 自定义StatFilter 配置 其他 Filter 不再演示
spring.datasource.filter.stat.db-type=h2
spring.datasource.filter.stat.log-slow-sql=true
spring.datasource.filter.stat.slow-sql-millis=2000

# JPA
spring.jpa.show-sql= true
spring.jpa.hibernate.ddl-auto=create-drop

# 配置下面参数用于启动监控页面，考虑安全问题，默认是关闭的，按需开启
spring.datasource.stat-view-servlet.enabled=true
spring.datasource.filter.stat.enabled=true
spring.datasource.web-stat-filter.enabled=true

# 更多配置属性见 DruidDataSource 内成员变量（只要有set方法便支持），或者根据IDE提示，或者查看官方文档
```
### Druid类
src/main/java/com/example/dbserver/DruidConfig.java
```java
@Configuration
public class DruidConfig {

    /**
     * 数据源
     * 
     * @return
     */
    @Bean
    @ConfigurationProperties(prefix = "spring.datasource")
    public DataSource druid() {
        return new DruidDataSource();
    }
}
```
#### 测试代码
```java
    @Test
    public void testDataSourceExists() throws Exception {
        assertThat(dataSource).isNotNull();
    }

    @Test
    public void testDataSourcePropertiesOverridden() throws Exception {
        assertThat(dataSource.getUrl()).isEqualTo("jdbc:mysql://172.25.0.2:3306/db1");
        assertThat(dataSource.getInitialSize()).isEqualTo(2);
        assertThat(dataSource.getMaxActive()).isEqualTo(30);
        assertThat(dataSource.getMinIdle()).isEqualTo(2);
        assertThat(dataSource.getMaxWait()).isEqualTo(1234);
        assertThat(dataSource.isPoolPreparedStatements()).isTrue();
        //assertThat(dataSource.getMaxOpenPreparedStatements()).isEqualTo(5); //Duplicated with following
        assertThat(dataSource.getMaxPoolPreparedStatementPerConnectionSize()).isEqualTo(5);
        assertThat(dataSource.getValidationQuery()).isEqualTo("select 1");
        assertThat(dataSource.getValidationQueryTimeout()).isEqualTo(1);
        assertThat(dataSource.isTestWhileIdle()).isTrue();
        assertThat(dataSource.isTestOnBorrow()).isTrue();
        assertThat(dataSource.isTestOnReturn()).isTrue();
        assertThat(dataSource.getTimeBetweenEvictionRunsMillis()).isEqualTo(10000);
        assertThat(dataSource.getMinEvictableIdleTimeMillis()).isEqualTo(30001);
        assertThat(dataSource.isAsyncCloseConnectionEnable()).isEqualTo(true);
    }
```
### 创建消息实体
这个写法，有点把类当结构体用的感觉
src/main/java/com/example/tarsmqserver/domain/MessageModel.java
#### 正常写法
```java
public class MessageModel {
    /**
     * 消息ID
     */
    private long msgid;

    /**
     * 消息内容
     */
    private String content;

    /**
     * 消息保存到数据库的时间
     */
    private long created;

    public void setContent(String content) {
        this.content = content;
    }

    public void setCreated(long created) {
        this.created = created;
    }

    public void setMsgid(long msgid) {
        this.msgid = msgid;
    }

    public String getContent() {
        return content;
    }

    public long getCreated() {
        return created;
    }

    public long getMsgid() {
        return msgid;
    }

}
```
#### @data 注解写法
```java
@Data
public class MessageData {
    /**
     * 消息ID
     */
    private long msgid;

    /**
     * 消息内容
     */
    private String content;

    /**
     * 消息保存到数据库的时间
     */
    private long created;

}
```
两种方法都能通过测试
### 创建 mapper
src/main/java/com/example/tarsmqserver/dao/MessageMapper.java
```java
@Repository
public interface MessageMapper {
    /**
     * 从nsq的topic获取到数据保存到db中存储
     * @param msg
     */
    @Insert("insert into message (content,created) values (#{content},#{created})")
    void save(Message msg);

    /**
     * 从nsq的topic获取到数据保存到db中存储       测试@data写法数据的提交
     * @param msg
     */
    @Insert("insert into message (content,created) values (#{content},#{created})")
    void saveData(MessageData msg);
}
```
### 创建服务
#### 服务接口
src/main/java/com/example/tarsmqserver/service/MessageService.java
```java
public interface MessageService {
    /**
     * 从nsq的topic获取到数据保存到db中存储
     */
    void save(Message msg);
    void saveData(MessageData msg);
}
```
#### 接口实现
src/main/java/com/example/tarsmqserver/service/MessageServiceImpl.java
```java
@Component
public class MessageServiceImpl implements MessageService {

    @Autowired
    MessageMapper messageMapper;

    @Override
    public void save(Message msg) {
        // TODO Auto-generated method stub
        messageMapper.save(msg);
    }

    @Override
    public void saveData(MessageData msg) {
        // TODO Auto-generated method stub
        messageMapper.saveData(msg);
    }
}
```
### 修改 main
主要是添加注解 `@MapperScan("com.example.dbserver.dao")` 意思是让 mybatis 怎么去找到这个文件，然后产生绑定关系
```java
@SpringBootApplication
@EnableTarsServer
@MapperScan("com.example.tarsmqserver.dao")
public class TarsMqServerApplication {

	public static void main(String[] args) {
		// 关闭 spring boot 自带的web服务 目前场景只用到了rpc服务
		SpringApplication app = new SpringApplication(TarsMqServerApplication.class);
		app.setWebApplicationType(WebApplicationType.NONE);
		app.run(args);
	}

}
```
### 测试用例
```java
    @Autowired
    MessageServiceImpl imp;

    @Test
    public void saveMessage() {
        Message message = new Message();
        message.setContent("abcdefg");
        message.setCreated(System.currentTimeMillis());
        imp.save(message);
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

        MessageModel messagObject = new MessageModel();
        messagObject.setContent(receivedMsg);
        messagObject.setCreated(System.currentTimeMillis());

        try {
            imp.save(messagObject);
        } catch (Exception e) {
            e.printStackTrace();
            JMS_LISTENER.error(e.toString());
            //TODO: handle exception
        }
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
服务正常收到消息,查看数据库,也能看到数据正常被添加上,数据库表自行创建即可.

## 源代码
[实现收到IBMMQ的数据之后存储到数据库(微服务系列第五天)](https://github.com/aomirun/demo/tree/main/day-5/tars-mq-server)

## 结语
spring boot 2.x 对于快餐使用,还是很友好的,封装了很多东西,很多东西拿来就可以直接用.但对于真正去掌控,估计会很复杂.这个只有等技术深入后,再慢慢探究.
不过Java是真的吃内存,可能是用JVM的原因吧.这个时候我就比较想念GO了,相同的功能,对资源占用少得多.
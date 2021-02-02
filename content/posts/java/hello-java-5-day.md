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
## 新建项目 dbserver
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
spring.datasource.url=jdbc:mysql://127.0.0.1:3306/db1
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
        assertThat(dataSource.getUrl()).isEqualTo("jdbc:mysql://127.0.0.1:3306/db1");
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
#### 运行测试
```sh
$ mvn test
```
运行后可见测试成功,证明配置数据都能正常读取到。
### 创建消息实体
这个写法，有点把类当结构体用的感觉
src/main/java/com/example/dbserver/Message.java
#### 正常写法
```java
public class Message {
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
这里建了一个dao目录，主要是测试启动类的@MapperScan("com.example.dbserver.dao")效果
src/main/java/com/example/dbserver/dao/MessageMapper.java
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
src/main/java/com/example/dbserver/MessageService.java
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
src/main/java/com/example/dbserver/MessageServiceImpl.java
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
@MapperScan("com.example.dbserver.dao")
public class DbserverApplication {

	public static void main(String[] args) {
		SpringApplication.run(DbserverApplication.class, args);
	}

}
```
### 添加测试用例
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
### 运行测试
```sh
$ mvn test

2021-02-03 01:47:34.014  INFO 14285 --- [           main] c.e.dbserver.DbserverApplicationTests    : Started DbserverApplicationTests in 1.239 seconds (JVM running for 1.97)
2021-02-03 01:47:34.460  INFO 14285 --- [           main] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} inited
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 2.107 s - in com.example.dbserver.DbserverApplicationTests
2021-02-03 01:47:34.567  INFO 14285 --- [extShutdownHook] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} closing ...
2021-02-03 01:47:34.573  INFO 14285 --- [extShutdownHook] com.alibaba.druid.pool.DruidDataSource   : {dataSource-1} closed
[INFO] 
[INFO] Results:
[INFO] 
[INFO] Tests run: 3, Failures: 0, Errors: 0, Skipped: 0
```
测试通过，然后去数据库查看，则能看到添加了两条消息，接下来就是要给服务添加订阅nsq消息和整合到tars平台中了
### 订阅消息
#### pom.xml 添加依赖
```xml
		<dependency>
			<groupId>com.sproutsocial</groupId>
			<artifactId>nsq-j</artifactId>
			<version>1.2</version>
		</dependency>
```
#### 修改服务接口
src/main/java/com/example/dbserver/MessageService.java
```java

---
title: "Hello Java 3 Day 学习JAVA第3天"
date: 2021-01-28T18:16:06+08:00
draft: true
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
前面搞了那么多环境，主要还是为了折腾多语言微服务开发架构，这样就不用只限定某一种开发语言了，各种语言谁行谁上。今天主要目的就是把前面做的服务端程序，使用客户端来调用。

## 客户端同步/异步调用服务
### 构建客户端工程项目,提供WEB代理服务
具体种自开发环境不同，自行创建maven项目
### pom.xml
```xml
<?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
	xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance" xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 https://maven.apache.org/xsd/maven-4.0.0.xsd">
	<modelVersion>4.0.0</modelVersion>
	<parent>
		<groupId>org.springframework.boot</groupId>
		<artifactId>spring-boot-starter-parent</artifactId>
		<version>2.4.2</version>
		<relativePath/>
		<!-- lookup parent from repository -->
	</parent>
	<groupId>com.example</groupId>
	<artifactId>client</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>client</name>
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
			<groupId>com.tencent.tars</groupId>
			<artifactId>tars-client</artifactId>
			<version>1.7.2</version>
			<type>jar</type>
		</dependency>
		<dependency>
			<groupId>com.tencent.tars</groupId>
			<artifactId>tars-spring-boot-starter</artifactId>
			<version>1.7.2</version>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-data-rest</artifactId>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-validation</artifactId>
		</dependency>
	</dependencies>

	<build>
		<finalName>restDemo</finalName>
		<plugins>
			<plugin>
				<groupId>org.springframework.boot</groupId>
				<artifactId>spring-boot-maven-plugin</artifactId>
			</plugin>
			<plugin>
				<groupId>com.tencent.tars</groupId>
				<artifactId>tars-maven-plugin</artifactId>
				<version>1.7.2</version>
				<configuration>
					<tars2JavaConfig>
						<!-- tars文件位置 -->
						<tarsFiles>
							<tarsFile>${basedir}/src/main/resources/demo.tars</tarsFile>
						</tarsFiles>
						<!-- 源文件编码 -->
						<tarsFileCharset>UTF-8</tarsFileCharset>
						<!-- 生成代码，PS：客户端调用，这里需要设置为false -->
						<servant>false</servant>
						<!-- 生成源代码编码 -->
						<charset>UTF-8</charset>
						<!-- 生成的源代码目录 -->
						<srcPath>${basedir}/src/main/java</srcPath>
						<!-- 生成源代码包前缀 -->
						<packagePrefixName>com.example.client.service.</packagePrefixName>
					</tars2JavaConfig>
				</configuration>
			</plugin>

		</plugins>
	</build>

</project>
```
### demo.tars 内容
`demo.tars` 这个是从服务端那里复制过来的，用于rest服务的调用 
src/main/resources/demo.tars
```
module Demo
{
	interface Message
	{
                bool send(string msg);
                string encode(string sign);
                bool encodeWithSend(string msg, string sign);
	};
};
```
### 生成代码
```sh
$ mvn tars:tars2java
```
### main
```java
@SpringBootApplication
@EnableTarsServer
public class ClientApplication {

	public static void main(String[] args) {
		SpringApplication.run(ClientApplication.class, args);
		// 关闭web服务
		// SpringApplication app = new SpringApplication(ClientApplication.class);
		// app.setWebApplicationType(WebApplicationType.NONE);
		// app.run(args);
	}
}
```
### 配置文件
spring boot 默认都是开启8080端口，因为我开发机上要运行多个，所以改下端口。
```ini
server.port=1080
```
### 自定义了一个返回
目前还比较粗造，后面再改进
src/main/java/com/example/client/utils/Result.java
```java
public class Result<T> {

    /*错误码*/
    private Integer code;

    /*提示信息 */
    private String msg;

    /*具体内容*/
    private  T data;

    public Integer getCode() {
        return code;
    }

    public void setCode(Integer code) {
        this.code = code;
    }

    public String getMsg() {
        return msg;
    }

    public void setMsg(String msg) {
        this.msg = msg;
    }

    public T getData() {
        return data;
    }

    public void setData(T data) {
        this.data = data;
    }
}
```
src/main/java/com/example/client/utils/Resp.java
```java
public class Resp {
    /**
     * 请求成功返回
     * 
     * @param object
     * @return
     */
    public static Result success(Object object) {
        Result msg = new Result();
        msg.setCode(200);
        msg.setMsg("请求成功");
        msg.setData(object);
        return msg;
    }

    public static Result success() {
        return success(null);
    }

    public static Result error(Integer code, String resultmsg) {
        Result msg = new Result();
        msg.setCode(code);
        msg.setMsg(resultmsg);
        return msg;
    }

}
```
### 弄了一个post参数验证
src/main/java/com/example/client/domain/RPCMessage.java
```java
public class RPCMessage {
    @NotBlank(message = "请输入消息")
    private String msg;

    @NotBlank(message = "请输入签名要素")
    private String sign;

    public String getMsg() {
        return msg;
    }

    public String getSign() {
        return sign;
    }
}
```
### 同步调用注解
通过注解绑定了tars客户端
```java
    @TarsClient("example.demo.MsgObj")
    MessagePrx messagePrx;
```

### 编写restful 实现同步请求
src/main/java/com/example/client/controller/MessageController.java
```java
@RestController
@TarsHttpService("restMsgObj")
@RequestMapping("/api/mq")
public class MessageController {
    @TarsClient("example.demo.MsgObj")
    MessagePrx messagePrx;

    final static String ERR_REQUEST = "错误的请求";
    final static String ERR_NETWORK = "网络请求错误，请重试";

    @GetMapping("/send/{msg}")
    public Result rpcSend(@PathVariable("msg") String msg) {
        if (msg == null) {
            return Resp.error(400, ERR_REQUEST);
        }

        boolean ok = false;
        try {
            ok = messagePrx.send(msg);
        } catch (Exception e) {
            e.printStackTrace();
        }

        if (ok) {
            return Resp.success();
        }
        return Resp.error(400, ERR_NETWORK);
    }

    @GetMapping("/encode/{sign}")
    public Result rpcEncode(@PathVariable("sign") String sign) {
        if (sign == null) {
            return Resp.error(400, ERR_REQUEST);
        }

        String enStr = null;
        boolean isErr = false;
        try {
            enStr = messagePrx.encode(sign);
            if (enStr != null) {
            }
        } catch (Exception e) {
            isErr = true;
            e.printStackTrace();
        }

        if (isErr) {
            return Resp.error(400, ERR_NETWORK);
        }

        if (enStr == null) {
            return Resp.error(400, ERR_REQUEST);
        }
        return Resp.success(enStr);
    }

    @PostMapping("")
    public Result rpcEncodeWithSend(@Valid @RequestBody RPCMessage body) {
        boolean ok = false;
        try {
            ok = messagePrx.encodeWithSend(body.getMsg(), body.getSign());
        } catch (Exception e) {
            e.printStackTrace();
        }

        if (ok) {
            return Resp.success();
        }
        return Resp.error(400, ERR_NETWORK);
    }
}
```
### 部署
通过之前的方法，打包->上传->测试数据->观察结果
#### 打包
```sh
$ mvn clean package
```
#### 上传
通过tars-web上传
#### 测试
通过浏览器或curl测试
```sh
$ curl http://localhost:1080/api/mq/send/asdfsfmasdfasmdf

{"code":200,"msg":"请求成功","data":null}%

$ curl http://localhost:1080/api/mq/encode/asdfsfmasdfasmdf
{"code":200,"msg":"请求成功","data":"1231#!@#!"}% 

$ curl --location --request POST 'http://localhost:1080/api/mq' \
--header 'Content-Type: application/json' \
--data-raw '{
    "msg": "abc",
    "sign": "aaa"
}'

{"code":200,"msg":"请求成功","data":null}%
```
#### 查看结果
在tars-web后面查看日志，可以看到收到了消息，同步消息测试完成。
```log
[enerContainer-1] com.example.demo.jms.ConsumerListener : Consumer收到的报文为: asdfsfmasdfasmdf
```
### 异步代码
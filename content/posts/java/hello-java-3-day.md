---
title: "实现客户端同步和异步调用RPC微服务(微服务系列第三天)"
date: 2021-01-28T18:16:06+08:00
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
> 学习java的第3天 实现目标： 学习tars 客户端同步和异步调用RPC服务

前面搞了那么多环境，主要还是为了折腾多语言微服务开发架构，这样就不用只限定某一种开发语言了，各种语言谁行谁上。今天主要目的就是把前面做的服务端程序，使用客户端来调用。

## 客户端同步/单向/异步调用服务
### 构建客户端工程项目,提供WEB代理服务
创建方法有很多种，具体不多说了，自行创建maven项目
### 添加依赖
```xml
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
```
### 添加插件
```xml
	<build>
		<finalName>mqwebservice</finalName>
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
						<!-- tars文件位置 -->
						<tarsFiles>
							<tarsFile>${basedir}/src/main/resources/mqserver.tars</tarsFile>
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
						<packagePrefixName>com.example.mqwebservice.service.</packagePrefixName>
					</tars2JavaConfig>
				</configuration>
			</plugin>
			<!--package plugin-->
		</plugins>
	</build>
```
### 注解main
```java
@SpringBootApplication
@EnableTarsServer
public class MqwebserviceApplication {

	public static void main(String[] args) {
		SpringApplication.run(MqwebserviceApplication.class, args);
	}
}
```
### 生成mqserver.tars客户端文件
文件用的是第一天服务端的接口定义文件 [传送门](/posts/java/hello-java-1-day/)
```sh
$ mvn tars:tars2java
```
### 创建Web控制器
src/main/java/com/example/mqwebservice/web/MQController.java
```java
@TarsHttpService("mqwebserviceObj")
@RestController()
@RequestMapping("/api/mq")
public class MQController {

    private final static Logger RPC_LOGGER = LoggerFactory.getLogger("rpc");

    @TarsClient("example.mqserver.messageObj")
    MessagePrx messagePrx;

    
    /**
     * 远程调用rpc服务，发送消息给ibmmq
     * 
     * @param msg
     * @return
     */
    @GetMapping("/send/{msg}")
    public boolean rpcSend(@PathVariable("msg") String msg) {
        if (StringUtils.isBlank(msg)) {
            return false;
        }
        RPC_LOGGER.info("发送的报文为: {}", msg);
        return messagePrx.send(msg);
    }

    @GetMapping("/encode/{sign}")
    public String rpcEncode(@PathVariable("sign") String sign) {
        Holder<String> enStr = new Holder<String>();
        messagePrx.encode(sign, enStr);
        String ret = enStr.value;
        RPC_LOGGER.info("收到的密文是: {}", enStr.value);
        return ret;
    }

    @GetMapping("/send/{msg}/encode/{sign}")
    public boolean rpcEncodeWithSend(@PathVariable("msg") String msg, @PathVariable("sign") String sign) {
        if (StringUtils.isBlank(msg) || StringUtils.isBlank(sign)) {
            return false;
        }
        boolean ok =messagePrx.encodeWithSend(msg, sign);
        if (!ok) {
            return false;
        }
        RPC_LOGGER.info("发送的报文为: {}", msg);
        return true;
    }
}
```

### main
```java
@SpringBootApplication
@EnableTarsServer
public class MqwebserviceApplication {

	public static void main(String[] args) {
		SpringApplication.run(MqwebserviceApplication.class, args);
	}
}
```
### 同步调用注解
通过注解绑定了tars客户端
```java
    @TarsClient("example.mqserver.messageObj")
    MessagePrx messagePrx;
```

### 编写restful 实现同步/异步请求
src/main/java/com/example/client/controller/MessageController.java
```java
// 开启tars的web服务，整合spring boot tomcat 的web服务
@TarsHttpService("mqwebserviceObj")
@RestController()
@RequestMapping("/api/mq")
public class MQController {

    private final static Logger RPC_LOGGER = LoggerFactory.getLogger("rpc");

    @TarsClient("example.mqserver.messageObj")
    MessagePrx messagePrx;

    /**
     * 远程调用rpc服务，发送消息给ibmmq
     * 
     * @param msg
     * @return
     */
    @GetMapping("/send/{msg}")
    public boolean rpcSend(@PathVariable("msg") String msg) {
        if (StringUtils.isBlank(msg)) {
            return false;
        }

        // 异步调用
        RPC_LOGGER.info("开始异步发送的报文为: {}", msg);
        messagePrx.async_send(new MessagePrxCallback() {
            @Override
            public void callback_expired() {
            }

            @Override
            public void callback_exception(Throwable ex) {
            }

            @Override
            public void callback_send(boolean ret) {
                RPC_LOGGER.info("接收到异步返回状态: {}, ",ret);
            }

            @Override
            public void callback_encode(boolean ret, String enStr){}

            @Override
            public void callback_encodeWithSend(boolean ret){}
        }, msg);
        
        // 单向调用 所谓单向调用，表示客户端只管发送数据，而不接收服务端的响应，也不管服务端是否接收到请求。
        messagePrx.async_send(null, msg);
        RPC_LOGGER.info("单向调用发送的报文为: {}", msg);


        // 同步调用
        boolean isSend = messagePrx.send(msg);
        RPC_LOGGER.info("同步发送的报文为: {}, 状态： {}", msg, isSend);
        return isSend;
    }

    @GetMapping("/encode/{sign}")
    public String rpcEncode(@PathVariable("sign") String sign) {
        Holder<String> enStr = new Holder<String>();
        messagePrx.encode(sign, enStr);
        String ret = enStr.value;
        RPC_LOGGER.info("收到的密文是: {}", enStr.value);
        return ret;
    }

    @GetMapping("/send/{msg}/encode/{sign}")
    public boolean rpcEncodeWithSend(@PathVariable("msg") String msg, @PathVariable("sign") String sign) {
        if (StringUtils.isBlank(msg) || StringUtils.isBlank(sign)) {
            return false;
        }
        boolean ok = messagePrx.encodeWithSend(msg, sign);
        if (!ok) {
            return false;
        }
        RPC_LOGGER.info("发送的报文为: {}", msg);
        return true;
    }
}
```
### @TarsHttpService 用途
把springboot tomcat的端口配置变成tars的指定的商品，可以在tars-web平台上绑定
```log
[SERVER] server starting at tcp -h 127.0.0.1 -p 1080 -t 3000...
[SERVER] server started at tcp -h 127.0.0.1 -p 1080 -t 3000...
[SERVER] The application started successfully.
The session manager service started...
[SERVER] server is ready...
2021-01-30 14:25:14.886 INFO 26735 --- [ main] o.s.b.w.embedded.tomcat.TomcatWebServer : Tomcat started on port(s): 1080 (http) with context path ''
```
### 部署
通过之前的方法，打包->上传->测试数据->观察结果
#### 打包
```sh
$ mvn clean package -DskipTests 
```
#### 上传
通过tars-web上传
#### 测试
通过浏览器或curl测试
```sh
$ curl http://localhost:1080/api/mq/send/abcd
{"success":true,"code":200,"msg":null,"data":true}% 
```
#### 查看结果
在tars-web后面查看日志，可以看到收到了消息，同步消息测试完成。
mqserver
```log
2021-01-30 14:28:30.033 INFO 26862 --- [ageObjAdapter-5] msg : send: DEV.QUEUE.1 -> abcd
2021-01-30 14:28:30.034 INFO 26862 --- [ageObjAdapter-2] msg : send: DEV.QUEUE.1 -> abcd
2021-01-30 14:28:30.035 INFO 26862 --- [ageObjAdapter-1] msg : send: DEV.QUEUE.1 -> abcd
2021-01-30 14:28:30.037 INFO 26862 --- [enerContainer-1] flow : Consumer收到的报文为: abcd
2021-01-30 14:28:30.126 INFO 26862 --- [enerContainer-1] flow : Consumer收到的报文为: abcd
2021-01-30 14:28:30.130 INFO 26862 --- [enerContainer-1] flow : Consumer收到的报文为: abcd
```

mqwebservice
```log
2021-01-30 14:28:30.027 INFO 27613 --- [0.5-1080-exec-1] rpc : 开始异步发送的报文为: abcd
2021-01-30 14:28:30.033 INFO 27613 --- [0.5-1080-exec-1] rpc : 单向调用发送的报文为: abcd
2021-01-30 14:28:30.040 INFO 27613 --- [ient-executor-2] rpc : 接收到异步返回状态: true,
2021-01-30 14:28:30.136 INFO 27613 --- [0.5-1080-exec-1] rpc : 同步发送的报文为: abcd, 状态： true
```
为了展示例子，项目代码还是比较简单的。

## 用golang给java服务发消息
后面想想加个其它语言开发的客户端来调用，体验多语言的优势，只是简要的完成，用于测试
```go
package main

import (
	"fmt"
	"log"

	"github.com/TarsCloud/TarsGo/tars"

	"aomi.run/goclient/mqserver"
)

func main() {
	comm := tars.NewCommunicator()
	obj := fmt.Sprintf("example.mqserver.messageObj")
	app := new(mqserver.Message)
	comm.StringToProxy(obj, app)

	ret, err := app.Send("通过golang发送的消息")
	check(ret, err)

	enStr := ""
	ret, err = app.Encode("通过golang发送的加密文本", &enStr)
	check(ret, err)

	ret, err = app.EncodeWithSend("通过golang发送的消息", "通过golang发送的加密文本")
	check(ret, err)

}

func check(status bool, err error) {
	if err != nil {
		fmt.Println(err)
		log.Fatal(err)
	}
	log.Println(status)
}
```
### 日志消息
```log
onSessionDestroyed: 0
2021-01-30 19:05:09.369 INFO 18188 --- [ageObjAdapter-3] msg : send: DEV.QUEUE.1 -> 通过golang发送的消息
2021-01-30 19:05:09.381 INFO 18188 --- [enerContainer-1] flow : Consumer收到的报文为: 通过golang发送的消息
2021-01-30 19:05:09.381 INFO 18188 --- [ageObjAdapter-4] msg : encode: 通过golang发送的加密文本 -> 123通过golang发送的加密文本456
2021-01-30 19:05:09.383 INFO 18188 --- [ageObjAdapter-5] msg : encode: 通过golang发送的加密文本 -> 123通过golang发送的加密文本456
2021-01-30 19:05:09.383 INFO 18188 --- [ageObjAdapter-5] msg : send: DEV.QUEUE.1 -> 123通过golang发送的加密文本456通过golang发送的消息
2021-01-30 19:05:09.387 INFO 18188 --- [enerContainer-1] flow : Consumer收到的报文为: 123通过golang发送的加密文本456通过golang发送的消息
```
## 本地调试
如果所有开发的项目都要打包/上传/远程调试/查看日志，无疑是非常麻烦的，所以就需要有本地调试。查看tars官方文档中有描述说明
> - 首先在工程目录下执行 mvn tars:build -Dapp=TestApp -Dserver=HelloJavaServer -DjvmParams="-Xms1024m -Xmx1024m -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Xdebug -Xrunjdwp:transport=dt_socket,address=9000,server=y,suspend=n"
> - 在工程目录target/tars/bin/tars_start 启动服务

```sh
$ mvn tars:build -Dapp=example -Dserver=mqserver -DjvmParams="-Xms1024m -Xmx1024m -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Xdebug -Xrunjdwp:transport=dt_socket,address=9000,server=y,suspend=n" 
```
我调试了一段时间，会报一些错误，最后提示错误为，一直没有解决
```
Failed to tars build: failed to find WEB-INF/servants.xml,  servants will be disabled
```
如何配置使`tars:build`找到这个目录和文件，还没学会，以后学会了再回来修改这一段。
现在我使用另外一个方法来解决这个问题,以第2天的代码为例子，来做本地调试，因为今天的例子是客户端，依赖第2天做的服务端。
### 从tars服务器找文件
```sh
$ tree ~/docker-app/tars/data/node/tarsnode-data/example.mqserver

├── bin
│   ├── example.mqserver.jar
│   └── tars_start.sh
├── conf
│   ├── example.mqserver.config.conf
│   └── example.mqserver.config.conf.bak
└── data
    └── tarsnodes.dat

3 directories, 5 files

$ cp ~/docker-app/tars/data/node/tarsnode-data/example.mqserver/bin/tars_start.sh .
$ cp ~/docker-app/tars/data/node/tarsnode-data/example.mqserver/conf/example.mqserver.config.conf .
```
### 修改 example.mqserver.config.conf
day-2/tars-mq-server/src/main/resources/example.mqserver.config.conf
```xml
<tars>
	<application>
		enableset=N
		setdivision=NULL
		<client>
			locator=
			sync-invoke-timeout=20000
			async-invoke-timeout=20000
			refresh-endpoint-interval=60000
			stat=tars.tarsstat.StatObj
			property=tars.tarsproperty.PropertyObj
			report-interval=60000
			modulename=mqserver.messageJavaServer
		</client>
		<server>
			node=
            		app=mqserver
            		server=Message
            		localip=127.0.0.1
            		local=tcp -h 127.0.0.1 -p 18601 -t 3000
            		basepath=.
            		datapath=.
            		logpath=.
            		loglevel=DEBUG
            		logsize=15M
            		log=
            		config=tars.tarsconfig.ConfigObj
            		notify=tars.tarsnotify.NotifyObj
            		mainclass=com.qq.tars.server.startup.Main
            		jvmparams=-Xms1024m -Xmx1024m
            		sessiontimeout=120000
            		sessioncheckinterval=60000
            		tcpnodelay=true
            		udpbuffersize=8192
            		charsetname=UTF-8
			<mqserver.Message.messageObjAdapter>
				allow
				endpoint=tcp -h 127.0.0.1 -p 18600 -t 60000
				handlegroup=mqserver.Message.messageObjAdapter
				maxconns=200000
				protocol=tars
				queuecap=10000
				queuetimeout=60000
				servant=mqserver.Message.messageObj
				shmcap=0
				shmkey=0
				threads=100
			</mqserver.Message.messageObjAdapter>
		</server>
	</application>
</tars>
```
### 修改tars_start.sh
```sh
#!/bin/sh

java -Dconfig=target/classes/example.mqserver.config.conf -Dcom.sun.management.jmxremote.ssl=false -Dcom.sun.management.jmxremote.authenticate=false -Xms2000m -Xmx2000m -Xmn1000m -Xss1000k -XX:CMSInitiatingOccupancyFraction=60 -XX:+CMSParallelRemarkEnabled -XX:+CMSScavengeBeforeRemark -verbosegc -XX:+PrintGCDetails -XX:ErrorFile=target/mqserver/logs/jvm_error.log -jar target/mqserver.jar
```
### 启动本地测试
```sh
$ sh tars_start.sh


  .   ____          _            __ _ _
 /\\ / ___'_ __ _ _(_)_ __  __ _ \ \ \ \
( ( )\___ | '_ | '_| | '_ \/ _` | \ \ \ \
 \\/  ___)| |_)| | | | | || (_| |  ) ) ) )
  '  |____| .__|_| |_|_| |_\__, | / / / /
 =========|_|==============|___/=/_/_/_/
 :: Spring Boot ::                (v2.4.2)
 ...
 ...
 ...
```
这样本地就启动起来了，调试方便多了。
既然已经折腾这么久了，后续还会很频繁的使用，弄个脚本文件来处理吧。
### Makefile
用Linux做开发的人肯定很熟了，我这里就简单的弄了一个脚本。
其实就是做了mvn一些命令的别名，另外加了一个上传jar文件的命令。
```sh
$ make help
===============A common Makefile for java maven tars programs==============
Copyright (C) 2021 aomi.run
The following targets aresupport:

 test                                   - mvn test
 run                                    - tars local run
 build                                  - mvn clean package -DskipTests
 test-build                             - mvn clean package
 upload                                 - upload jar package to tars node
 build-upload                           - build and upload
 test-build-upload                      - test-build and upload
 help                                   - print help information

To make a target, do 'make [target]'
=============================== Version 1.7.2 =============================
```
`Makefile`文件存放在我的github上面，需要的可以进入[传送门](https://github.com/aomirun/demo/blob/main/day-2/tars-mq-server/Makefile)查看。

## 源代码
[实现客户端同步和异步调用RPC微服务(微服务系列第三天)](https://github.com/aomirun/demo/tree/main/day-3/mqwebservice)

## 结语
本打算今天弄简单点，不想到又加了个本地调试，和打包上传脚本，超纲了！顺便梳理了这3个文章，把项目名称和文件名称理了一下，顺便把源代码也存放到github上面，方便今后自己忘记了可以查阅。
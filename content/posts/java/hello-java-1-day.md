---
title: "Hello Java 1 Day 学习JAVA第1天 通过docker整合springboot和tars 构建微服务环境"
date: 2021-01-26T11:39:02+08:00
draft: false
toc: true
categories: 
  - java
images:
tags: 
  - java
  - tars
  - docker
  - micro serivce
  - 微服务
---

## 前言
> 实现目标：通过docker整合springboot和tars

先花了些时间折腾了一下Java的开发环境，平常主要用vscode做开发，就在VScode上弄了一套Java的开发环境，基于[win10 wsl2 vscode](/posts/wsl2-vscode-openjdk-install/) 的，具体环境折腾可以看我那篇环境搭建的文章。然后花了几个小时时间学习一下Java的基本语法，有哪些保留字，变量的作用域。包、接口、类还有继承关系，和其它语言参照学习一下。打算使用Java做一些项目的补充，则需要多语言混合开发，我选了较熟的Tars来做。Tars原生支持SpringBoot，OK开始折腾。
## tars 环境搭建
为了快速开始，我使用Docker来搭建开发环境,Docker的安装及使用搜索网络文章介绍即可。
### 创建 Docker 网络
```sh
$ docker network create -d bridge --subnet=172.25.0.0/16 --gateway=172.25.0.1 tars
```
### 创建 docker 目录
```sh
$ mkdir ~/docker-app/tars/framework/data -p
$ mkdir ~/docker-app/tars/framework-slave/data -p
$ mkdir ~/docker-app/tars/node/data -p
$ mkdir ~/docker-app/tars/mysql/data -p
$ mkdir ~/docker-app/tars/mysql/conf -p
```
### 创建 ~/docker-app/tars/mysql/conf/my.cnf 文件
```ini
[mysqld]
user=root
default-storage-engine=INNODB
character-set-server=utf8
[client]
default-character-set=utf8
[mysql]
default-character-set=utf8
```
### 创建 ~/docker-app/tars/docker-compose.yml 文件

```yml
version: "3"

services:
  mysql:
    image: mysql:5.6
    container_name: tars-mysql
    ports:
      - "3307:3306"
    # restart: always
    privileged: true
    environment:
      MYSQL_ROOT_PASSWORD: "123456"
    volumes:
      - ./mysql/data:/var/lib/mysql
      - ./mysql/conf/my.cnf:/etc/my.cnf
      - /etc/localtime:/etc/localtime
    networks:
      tars:
        ipv4_address: 172.25.0.2
  framework:
    # image: tarscloud/framework:stable
    image: tarscloud/framework:latest
    container_name: tars-framework
    ports:
      - "3000:3000"
      - "3001:3001"
    # restart: always
    networks:
      tars:
        ipv4_address: 172.25.0.3
    environment:
      MYSQL_HOST: "172.25.0.2"
      MYSQL_ROOT_PASSWORD: "123456"
      MYSQL_USER: "root"
      MYSQL_PORT: 3306
      REBUILD: "false"
      INET: eth0
      SLAVE: "false"
    volumes:
      - ./framework/data:/data/tars:rw
      - /etc/localtime:/etc/localtime
    depends_on:
      - mysql
  framework-slave:
    # image: tarscloud/framework:stable
    image: tarscloud/framework:latest
    container_name: tars-framework-slave
    # restart: always
    networks:
      tars:
        ipv4_address: 172.25.0.4
    environment:
      MYSQL_HOST: "172.25.0.2"
      MYSQL_ROOT_PASSWORD: "123456"
      MYSQL_USER: "root"
      MYSQL_PORT: 3306
      REBUILD: "false"
      INET: eth0
      SLAVE: "true"
    volumes:
      - ./framework-slave/data:/data/tars:rw
      - /etc/localtime:/etc/localtime
    depends_on:
      - mysql
  node:
    # image: tarscloud/tars-node:stable
    image: tarscloud/tars-node:latest
    container_name: tars-node
    # restart: always
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
networks:
  tars:
    external: true
```
### 运行 docker-compose
```sh
$ cd ~/docker-app/tars
$ docker-compose up -d
```
等一段时间，把镜像下载完后，会自动运行起来
```sh
Creating tars-mysql ... done
Creating tars-framework-slave ... done
Creating tars-framework       ... done
Creating tars-node            ... done
```
打开浏览器，访问 `http://localhost:3000/` 即可打开 `tars web`，关于Tars更详细的请查看 [官方文档](https://tarscloud.gitbook.io/)
## 创建JAVA项目
这是我是用`vscode + maven + tars`创建好的一个`IBM MQ CLIENT`的一个初始项目，关于vscode怎么设置Java开发环境及Maven的，请看我另外一篇文章 [Wsl2 Vscode Openjdk Install](/posts/wsl2-vscode-openjdk-install/) ，最后项目目录为
```sh
$ tree
.
├── HELP.md
├── mvnw
├── mvnw.cmd
├── pom.xml
└── src
    ├── main
    │   ├── java
    │   │   └── com
    │   │       └── example
    │   │           └── tarsmqserver
    │   │               └── TarsMqServerApplication.java
    │   └── resources
    │       └── application.properties
    └── test
        └── java
            └── com
                └── example
                    └── tarsmqserver
                        └── TarsMqServerApplicationTests.java

12 directories, 7 files
```
### 添加依赖
```xml
		<dependency>
			<groupId>com.tencent.tars</groupId>
			<artifactId>tars-spring-boot-starter</artifactId>
			<version>1.7.2</version>
		</dependency>
```
### 关掉测试用例
由于导入了 tars 包后,测试就有了 tars 专有依赖了,具体如何测试有 tars 的项目,后面再弄,现在玩关掉,要不然mvn package会通不过
```java
package com.example.tarsmqserver;

// import org.junit.jupiter.api.Test;
// import org.springframework.boot.test.context.SpringBootTest;

// @SpringBootTest
// class TarsMqServerApplicationTests {

// 	@Test
// 	void contextLoads() {
// 	}

// }
```
### 创建tars协议接口文件
```
module mqserver
{
	interface Message
	{
        bool send(string msg);
        bool encode(string sign, out string enStr);
        bool encodeWithSend(string msg, string sign);
	};
};
```
### 添加 tars 插件
同时定义好 jar 的文件名,最后build看起来像这样
```xml
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
```
### tars2java
通过tars协议接口,生成java接口文件
```sh
$ mvn tars:tars2java

[INFO] Scanning for projects...
[INFO] 
[INFO] ---------------------< com.example:tars-mq-server >---------------------
[INFO] Building tars-mq-server 0.0.1-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- tars-maven-plugin:1.7.2:tars2java (default-cli) @ tars-mq-server ---
[INFO] Parse /mnt/d/projects/java/demo/day-1/tars-mq-server/src/main/resources/mqserver.tars ...
[INFO] generate code for namespace : mqserver ...
[INFO] module mqserver >>
[INFO] generate Servant MessageServant
[INFO] module mqserver <<
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
```
文件生成到了 `src/main/java/com/example/tarsmqserver/service/mqserver/MessageServant.java`,打开看看
```java
@Servant
public interface MessageServant {

	 boolean send(@TarsMethodParameter(name="msg")String msg);

	 boolean encode(@TarsMethodParameter(name="sign")String sign, @TarsHolder(name="enStr") Holder<String> enStr);

	 boolean encodeWithSend(@TarsMethodParameter(name="msg")String msg, @TarsMethodParameter(name="sign")String sign);
}
```
### 实现接口
src/main/java/com/example/tarsmqserver/service/mqserver/impl/MessageServantImpl.java
tars 接口的类名需要添加注解 @TarsServant("messageObj") 
messageObj 的名称是要和tars-web上自己定义的名称相同,方法实现需要注解 @Override 
```java
package com.example.tarsmqserver.service.mqserver.impl;

import com.example.tarsmqserver.service.mqserver.MessageServant;
import com.qq.tars.common.support.Holder;
import com.qq.tars.spring.annotation.TarsServant;

/**
 * @author aomi.run
 */
@TarsServant("messageObj") 
public class MessageServantImpl implements MessageServant {
    @Override
    public boolean send(String msg) {
        // TODO 先简单走流程 后面实现具体与ibmmq的对接
        if (msg == null) {
            return false;
        }
        return true;
    }

    @Override
    public boolean encode(String sign, Holder<String> enStr) {
        // TODO 先简单走流程 后面实现具体与ibmmq的对接
        if (sign == null) {
            return false;
        }
        enStr.setValue("123" + sign + "456");
        return true;
    }

    @Override
    public boolean encodeWithSend(String msg, String sign) {
        // TODO 先简单走流程 后面实现具体与ibmmq的对接
        if (sign == null || msg == null) {
            return false;
        }

        Holder<String> enStr = new Holder<String>();
        boolean ok = encode(sign, enStr);
        if (!ok) {
            return false;
        }

        return send(enStr + msg);
    }
}
```

### 开启 tars 服务监听
在 main 入口处添加注解 `@EnableTarsServer`,并关闭自带的Web服务.
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
### Jar打包
```sh
$ mvn clean package
```
最后的目录结构是
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
│   │   │           └── tarsmqserver
│   │   │               ├── TarsMqServerApplication.java
│   │   │               └── service
│   │   │                   └── mqserver
│   │   │                       ├── MessageServant.java
│   │   │                       └── impl
│   │   │                           └── MessageServantImpl.java
│   │   └── resources
│   │       ├── application.properties
│   │       └── mqserver.tars
│   └── ...
└── target
    ...
    ├── mqserver.jar
    ├── mqserver.jar.original
    └── test-classes

35 directories, 22 files
```
mqserver.jar 就是tars的包,上传到tars-web上面
### tars 的部署与测试
> 首次使用tars-web,我尽量弄些图解,后续文档中有很多涉及到tars-web的就简单文字带过了
#### 创建应用名称
![](/images/posts/tars/tars-web-add-app.png)
#### 创建服务
![](/images/posts/tars/tars-web-add-server.png)
#### 上传jar包
![](/images/posts/tars/tars-web-push-manage.png)
![](/images/posts/tars/tars-web-update-push.png)
#### 上传接口定义文件
就是前面写的mqserver.tars文件
![](/images/posts/tars/tars-web-interface-test.png)
调试时可以看到，返回值与自己代码中定制的值相同，代表成功部署
```java
    @Override
    public boolean encode(String sign, Holder<String> enStr) {
        // TODO 先简单走流程 后面实现具体与ibmmq的对接
        if (sign == null) {
            return false;
        }
        enStr.setValue("123" + sign + "456");
        return true;
    }
```
### 源代码
源代码几天后做了整理，现上传到github上面了 传送门 -> [通过docker整合springboot和tars](https://github.com/aomirun/demo/tree/main/day-1/tars-mq-server)
### 结语
学习java的第一天折腾了一下项目大致流程，主要是在 `vscode` 中使用 `maven` 来创建 `springboot` 项目，并加入 `tars` 的支持，然后打包上传到 `tars` 中运行并调试。中间也碰到了一些问题，比如加载了tars包后`test`文件就有依赖了，不加入依赖就会报找不到tars 配置文件的错误，开始一直保留在项目中，打包`jar`的时候就会报错，总之自己是java菜鸟，碰到各种错误也属于正常了，整体还是很成功的。一经在 `tars` 中使用后，就和开发语言无关了。之后喜欢用哪种开发语言去调用服务都可以，这也算是微服务的特点之一吧。
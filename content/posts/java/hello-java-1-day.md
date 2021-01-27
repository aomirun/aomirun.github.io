---
title: "Hello Java 1 Day 学习JAVA第1天"
date: 2021-01-26T11:39:02+08:00
draft: false
toc: false
categories: 
  - java
images:
tags: 
  - java
---

## 前言
先花了些时间折腾了一下Java的开发环境，平常主要用vscode做开发，就在VScode上弄了一套Java的开发环境，基于[win10 wsl2 vscode](https://aomi.run/posts/wsl2-vscode-openjdk-install/) 的，具体环境折腾可以看我那篇环境搭建的文章。然后花了几个小时时间学习一下Java的基本语法，有哪些保留字，变量的作用域。包、接口、类还有继承关系，和其它语言参照学习一下。打算使用Java做一些项目的补充，则需要多语言混合开发，我选了较熟的Tars来做。Tars原生支持SpringBoot，OK开始折腾。
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
    restart: always
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
    restart: always
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
    restart: always
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
    restart: always
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
这是我是用`vscode + maven + tars`创建好的一个`IBM MQ CLIENT`的一个初始项目，关于vscode怎么设置Java开发环境及Maven的，请看我另外一篇文章 [Wsl2 Vscode Openjdk Install](https://aomi.run/posts/wsl2-vscode-openjdk-install/) ，最后项目目录为
```sh
➜  jmq tree
.
├── HELP.md
├── mvnw
├── mvnw.cmd
├── pom.xml
├── src
│   ├── main
│   │   ├── java
│   │   │   └── run
│   │   │       └── aomi
│   │   │           └── jmq
│   │   │               └── JmqApplication.java
│   │   └── resources
│   │       └── application.properties
│   └── test
│       └── java
│           └── run
│               └── aomi
│                   └── jmq
│                       └── JmqApplicationTests.java
└── target
    ├── classes
    │   ├── application.properties
    │   └── run
    │       └── aomi
    │           └── jmq
    │               └── JmqApplication.class
    └── test-classes
        └── run
            └── aomi
                └── jmq
                    └── JmqApplicationTests.class

```
### 开启 tars 支持
在 main 入口处添加注解 `@EnableTarsServer`
```java
@SpringBootApplication
@EnableTarsServer
public class JmqApplication {

	public static void main(String[] args) {
		SpringApplication.run(JmqApplication.class, args);
	}

}
```
### 定义tars接口文件
kit目录是通过tars2java生成的，具体过程为先在`resources`目录下创建一个Tars文件，如
src/main/resources/jmq.tars
```
module kit {
    //消息发送接口
    interface Msg{
        bool Send(string msg);                          //接收字符串，把整个字符串发送到IBM MQ中，返回 true/false
        string Hash(string plain);                      //接收明文字符串，返回加密后的字符串
        bool HashWithSend(string msg, string plain);    //接收明文字符串，加密后直接发送到IBM MQ
    };
};
```
### 添加tars依赖和插件
pom.xml
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
	<groupId>run.aomi</groupId>
	<artifactId>jmq</artifactId>
	<version>0.0.1-SNAPSHOT</version>
	<name>jmq</name>
	<description>jms for ibm mq</description>
	<properties>
		<java.version>8</java.version>
	</properties>
	<dependencies>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter</artifactId>
		</dependency>

		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-devtools</artifactId>
			<scope>runtime</scope>
			<optional>true</optional>
		</dependency>
		<dependency>
			<groupId>org.springframework.boot</groupId>
			<artifactId>spring-boot-starter-test</artifactId>
			<scope>test</scope>
		</dependency>
		<dependency>
			<groupId>com.tencent.tars</groupId>
			<artifactId>tars-spring-boot-starter</artifactId>
			<version>1.7.2</version>
		</dependency>
	</dependencies>

	<build>
		<finalName>jmq</finalName>
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
							<tarsFile>${basedir}/src/main/resources/jmq.tars</tarsFile>
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
						<packagePrefixName>run.aomi.jmq.</packagePrefixName>
					</tars2JavaConfig>
				</configuration>
			</plugin>
			<!--package plugin-->
		</plugins>
	</build>

</project>
```
### 生成接口文件
在项目目录下运行`mvn tars:tars2java`,完成后显示如下 
src/main/java/run/aomi/jmq/kit/MsgServant.java
```sh
$ mvn tars:tars2java

[INFO] ----------------------------< run.aomi:jmq >----------------------------
[INFO] Building jmq 0.0.1-SNAPSHOT
[INFO] --------------------------------[ jar ]---------------------------------
[INFO] 
[INFO] --- tars-maven-plugin:1.7.2:tars2java (default-cli) @ jmq ---
[INFO] Parse /mnt/d/projects/java/cnaps2/jmq/src/main/resources/jmq.tars ...
[INFO] generate code for namespace : kit ...
[INFO] module kit >>
[INFO] generate Servant msgServant
[INFO] module kit <<
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  0.486 s
[INFO] Finished at: 2021-01-26T15:24:56+08:00
[INFO] ------------------------------------------------------------------------
```
### 实现接口
在实现接口的同时,添加 注解 `@Override`
src/main/java/run/aomi/jmq/kit/MsgServantImpl.java
```java
package run.aomi.jmq.kit;

public class MsgServantImpl implements MsgServant {
    /**
     * 发送消息到IBM MQ
     */
    @Override
    public boolean Send(String msg) {
        if (msg.length() == 0) {
            return false;
        }
        return true;
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

        var hashStr = this.Hash(plain);
        var ok = this.Send(hashStr + msg);
        return ok;
    }
}
```
### 服务暴露配置
接口的实现类通过注解@TarsServant来暴露服务，其中填写的'MsgObj'为servant名，该名称与管理平台上的名称对应即可。
```java
package run.aomi.jmq.kit;

import com.qq.tars.spring.annotation.TarsServant;

@TarsServant("MsgObj")
public class MsgServantImpl implements MsgServant {
  @Override
  public boolean Send(String msg) {
    ...
  }
  ...

}

```
### Jar打包
先删除测试文件夹,主要还没空研究Java里的测试,后续学习后再来做测试包
```sh
$ mvn clean package

[INFO] Building jar: /mnt/d/projects/java/cnaps2/jmq/target/jmq.jar
[INFO] 
[INFO] --- spring-boot-maven-plugin:2.4.2:repackage (repackage) @ jmq ---
[INFO] Replacing main artifact with repackaged archive
[INFO] 
[INFO] --- spring-boot-maven-plugin:2.4.2:repackage (default) @ jmq ---
[INFO] Replacing main artifact with repackaged archive
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  4.980 s
[INFO] Finished at: 2021-01-26T17:50:25+08:00
[INFO] ------------------------------------------------------------------------
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
│   └── main
│       ├── java
│       │   └── run
│       │       └── aomi
│       │           └── jmq
│       │               ├── JmqApplication.java
│       │               └── kit
│       │                   ├── MsgServant.java
│       │                   └── impl
│       │                       └── MsgServantImpl.java
│       └── resources
│           ├── application.properties
│           └── jmq.tars
└── target
    ├── classes
    │   ├── application.properties
    │   ├── jmq.tars
    │   └── run
    │       └── aomi
    │           └── jmq
    │               ├── JmqApplication.class
    │               └── kit
    │                   ├── MsgServant.class
    │                   └── impl
    │                       └── MsgServantImpl.class
    ├── generated-sources
    │   └── annotations
    ├── jmq.jar
    ├── jmq.jar.original
    ├── maven-archiver
    │   └── pom.properties
    └── maven-status
        └── maven-compiler-plugin
            └── compile
                └── default-compile
                    ├── createdFiles.lst
                    └── inputFiles.lst

23 directories, 19 files
```
可以看出来,已经成功生成了Jar包,把这个包上传到TarsWeb页面来测试一下看.
### 部署与测试
#### 上传jar包
![](/images/posts/tars/tars-upload.png)
![](/images/posts/tars/tars-web-index.png)
#### 上传对应的测试文件 jmq.tars
![](/images/posts/tars/tars-web-debug.png)
调试时可以看到，返回值与自己代码中定制的值相同，代表成功部署
```java
    /**
     * 对plain进行加密，返回加密后的内容
     */
    @Override
    public String Hash(final String plain) {
        if (plain.length() == 0) {
            return null;
        }
        final String encodeStr = "1231#!@#!";
        return encodeStr;

    }
```

### 结语
学习java的第一天折腾了一下项目大致流程，主要是在 `vscode` 中使用 `maven` 来创建 `springboot` 项目，并加入 `tars` 的支持，然后打包上传到 `tars` 中运行并调试。中间也碰到了一些问题，比如没有实现`test`文件，保留在项目中，打包`jar`的时候就会报错，总之自己是java菜鸟，碰到各种错误也属于正常了，整体还是很成功的。一经在 `tars` 中使用后，就和开发语言无关了。之后喜欢用哪种开发语言去调用服务都可以，这也算是微服务的特点之一吧。
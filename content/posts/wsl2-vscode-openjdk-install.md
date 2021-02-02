---
title: "Wsl2 Vscode Openjdk Install"
date: 2021-01-25T11:46:59+08:00
draft: false
toc: true
categories: 
  - java
images:
tags: 
  - java
  - openjdk
  - vscode
  - wsl
---

## 准备
进入wsl命令行，先更新系统,更新慢的可以换apt源处理，换源自行搜索。
```shell
$ sudo apt update -y
$ sudo apt upgrade -y
```

## 安装JDK
### 安装 JDK 8
```shell
$ sudo apt install openjdk-8-jdk -y

/usr/lib/jvm/java-8-openjdk-amd64 #我电脑安装的位置

$ java -version
openjdk version "1.8.0_275"
OpenJDK Runtime Environment (build 1.8.0_275-8u275-b01-0ubuntu1~20.04-b01)
OpenJDK 64-Bit Server VM (build 25.275-b01, mixed mode)
```
### 装个 JDK 11
```sh
$ sudo apt install openjdk-11-jdk -y

/usr/lib/jvm/java-11-openjdk-amd64 #我电脑安装的位置
```
### JDK版本切换
```sh
$ sudo update-alternatives --install /usr/local/jdk jdk /usr/lib/jvm/java-8-openjdk-amd64 8
$ sudo update-alternatives --install /usr/local/jdk jdk /usr/lib/jvm/java-11-openjdk-amd64 11

$ sudo update-alternatives --config jdk

There are 2 choices for the alternative jdk (providing /usr/local/jdk).

  Selection    Path                                Priority   Status
------------------------------------------------------------
  0            /usr/lib/jvm/java-11-openjdk-amd64   11        auto mode
  1            /usr/lib/jvm/java-11-openjdk-amd64   11        manual mode
* 2            /usr/lib/jvm/java-8-openjdk-amd64    8         manual mode

Press <enter> to keep the current choice[*], or type selection number:
```
最后的8和11是优先级的意思，数字越大优先级就越高，默认最大值，我这里最大是11 默认就是JDK 11，输入数字选择
选择 0=auto 自动选择数值最大的 选择 1=jdk11 选择 2=jdk8
根据项目需要，随时可以切换版本
## 安装maven
安装maven
```sh
$ sudo apt install maven -y

$ mvn -v
Apache Maven 3.6.3
Maven home: /usr/share/maven
Java version: 1.8.0_275, vendor: Private Build, runtime: /usr/lib/jvm/java-8-openjdk-amd64/jre
Default locale: en, platform encoding: UTF-8
OS name: "linux", version: "4.19.104-microsoft-standard", arch: "amd64", family: "unix"
```

添加国内镜像到配置文件`/usr/share/maven/conf/settings.xml`，到网上找了一个镜像配置，如下：
```xml
  <mirrors>
        <mirror>
        <id>aliyun-public</id>
        <mirrorOf>*</mirrorOf>
        <name>aliyun public</name>
        <url>https://maven.aliyun.com/repository/public</url>
    </mirror>

    <mirror>
        <id>aliyun-central</id>
        <mirrorOf>*</mirrorOf>
        <name>aliyun central</name>
        <url>https://maven.aliyun.com/repository/central</url>
    </mirror>

    <mirror>
        <id>aliyun-spring</id>
        <mirrorOf>*</mirrorOf>
        <name>aliyun spring</name>
        <url>https://maven.aliyun.com/repository/spring</url>
    </mirror>

    <mirror>
        <id>aliyun-spring-plugin</id>
        <mirrorOf>*</mirrorOf>
        <name>aliyun spring-plugin</name>
        <url>https://maven.aliyun.com/repository/spring-plugin</url>
    </mirror>

    <mirror>
        <id>aliyun-apache-snapshots</id>
        <mirrorOf>*</mirrorOf>
        <name>aliyun apache-snapshots</name>
        <url>https://maven.aliyun.com/repository/apache-snapshots</url>
    </mirror>

    <mirror>
        <id>aliyun-google</id>
        <mirrorOf>*</mirrorOf>
        <name>aliyun google</name>
        <url>https://maven.aliyun.com/repository/google</url>
    </mirror>

    <mirror>
        <id>aliyun-gradle-plugin</id>
        <mirrorOf>*</mirrorOf>
        <name>aliyun gradle-plugin</name>
        <url>https://maven.aliyun.com/repository/gradle-plugin</url>
    </mirror>

    <mirror>
        <id>aliyun-jcenter</id>
        <mirrorOf>*</mirrorOf>
        <name>aliyun jcenter</name>
        <url>https://maven.aliyun.com/repository/jcenter</url>
    </mirror>

    <mirror>
        <id>aliyun-releases</id>
        <mirrorOf>*</mirrorOf>
        <name>aliyun releases</name>
        <url>https://maven.aliyun.com/repository/releases</url>
    </mirror>

    <mirror>
        <id>aliyun-snapshots</id>
        <mirrorOf>*</mirrorOf>
        <name>aliyun snapshots</name>
        <url>https://maven.aliyun.com/repository/snapshots</url>
    </mirror>

    <mirror>
        <id>aliyun-grails-core</id>
        <mirrorOf>*</mirrorOf>
        <name>aliyun grails-core</name>
        <url>https://maven.aliyun.com/repository/grails-core</url>
    </mirror>

    <mirror>
        <id>aliyun-mapr-public</id>
        <mirrorOf>*</mirrorOf>
        <name>aliyun mapr-public</name>
        <url>https://maven.aliyun.com/repository/mapr-public</url>
    </mirror>
 <!-- mirror
     | Specifies a repository mirror site to use instead of a given repository. The repository that
     | this mirror serves has an ID that matches the mirrorOf element of this mirror. IDs are used
     | for inheritance and direct lookup purposes, and must be unique across the set of mirrors.
     |
    <mirror>
      <id>mirrorId</id>
      <mirrorOf>repositoryId</mirrorOf>
      <name>Human Readable Name for this Mirror.</name>
      <url>http://my.repository.com/repo/path</url>
    </mirror>
     -->
  </mirrors>
```
## 添加环境变量
```sh
$ echo export MAVEN_HOME=/usr/share/maven >> ~/.bash_profile # zsh 是 ~/.zshrc
$ echo export PATH=$PATH:$MAVEN_HOME/bin >> ~/.bash_profile # zsh 是 ~/.zshrc

$ echo export JAVA_HOME=/usr/local/jdk >> ~/.bash_profile # zsh 是 ~/.zshrc
$ echo export PATH=$PATH:$JAVA_HOME/bin >> ~/.bash_profile # zsh 是 ~/.zshrc
$ source ~/.bash_profile # zsh 是 ~/.zshrc

$ echo $JAVA_HOME
/usr/local/jdk

$ echo $MAVEN_HOME
/usr/share/maven
```
## 测试切换不同环境
```sh
$ sudo update-alternatives --config jdk
There are 2 choices for the alternative jdk (providing /usr/local/jdk).

  Selection    Path                                Priority   Status
------------------------------------------------------------
  0            /usr/lib/jvm/java-11-openjdk-amd64   11        auto mode
  1            /usr/lib/jvm/java-11-openjdk-amd64   11        manual mode
* 2            /usr/lib/jvm/java-8-openjdk-amd64    8         manual mode
Press <enter> to keep the current choice[*], or type selection number: 0
update-alternatives: using /usr/lib/jvm/java-11-openjdk-amd64 to provide /usr/local/jdk (jdk) in auto mode

$ java -version
openjdk version "11.0.9.1" 2020-11-04
OpenJDK Runtime Environment (build 11.0.9.1+1-Ubuntu-0ubuntu1.20.04)
OpenJDK 64-Bit Server VM (build 11.0.9.1+1-Ubuntu-0ubuntu1.20.04, mixed mode, sharing)

$ javac -version
javac 11.0.9.1

$ mvn -v
Apache Maven 3.6.3
Maven home: /usr/share/maven
Java version: 11.0.9.1, vendor: Ubuntu, runtime: /usr/lib/jvm/java-11-openjdk-amd64
Default locale: en, platform encoding: UTF-8
OS name: "linux", version: "4.19.128-microsoft-standard", arch: "amd64", family: "unix"

$ sudo update-alternatives --config jdk
There are 2 choices for the alternative jdk (providing /usr/local/jdk).

  Selection    Path                                Priority   Status
------------------------------------------------------------
* 0            /usr/lib/jvm/java-11-openjdk-amd64   11        auto mode
  1            /usr/lib/jvm/java-11-openjdk-amd64   11        manual mode
  2            /usr/lib/jvm/java-8-openjdk-amd64    8         manual mode

Press <enter> to keep the current choice[*], or type selection number: 2
update-alternatives: using /usr/lib/jvm/java-8-openjdk-amd64 to provide /usr/local/jdk (jdk) in manual mode

$ javac -version
javac 1.8.0_275

$ java -version
openjdk version "1.8.0_275"
OpenJDK Runtime Environment (build 1.8.0_275-8u275-b01-0ubuntu1~20.04-b01)
OpenJDK 64-Bit Server VM (build 25.275-b01, mixed mode)

$ mvn -v
Apache Maven 3.6.3
Maven home: /usr/share/maven
Java version: 1.8.0_275, vendor: Private Build, runtime: /usr/lib/jvm/java-8-openjdk-amd64/jre
Default locale: en, platform encoding: UTF-8
OS name: "linux", version: "4.19.128-microsoft-standard", arch: "amd64", family: "unix"
```
可以正常切换环境，相对应的软件也会跟着切换环境，很美好。

## 使用vscode调试java
新建一个用于Java开发的项目文件夹
```sh
$ mkdir ~/projects/java/test -p
$ cd ~/projects/java/test && touch test.java
$ code .
```
这时会打开vscode,进入后，会提示下载Java扩展 vscjava.vscode-java-pack，如没有提示，自己去扩展那里搜索vscjava.vscode-java-pack
安装完后，可能会提示版本问题
> 如果遇到问题：【很抱歉，激活面向 Java 的 IntelliCode 支持时遇到问题。有关详细信息，请查看“针对 Java 的语言支持”和 “VS IntelliCode” 输出窗口】

手动降级language support…到0.64.1

编辑test.java文件
```java
class wsl {
    public static void main(String[] args) {
        System.out.println("hello java of wsl");
    }
}
```
点运行，可以看到输出
```sh
hello java of wsl
```

## 使用maven来创建Springboot项目
创建项目前，要配置一下vscode中的maven选项，告诉vscode配置文件地址，好使用国内镜像下载包。在vscode的设置中，搜索maven，global settings path to maven's global settings.xml选项中，输入 `/usr/share/maven/conf/settings.xml`

之后在vscode界面按 crtl+shift+p 调出命令输出 spring init 智能提示中看到有 create maven project相关字样，通过这个来创建。之后会跳出选择Springboot版本，我选了2.4 项目所用开发语言选java，groupid com.example,project demo,package type jar,jdk 选8或11，根据自己的环境来定，之后就是选择要用到的springboot开发工具了。就做个简单的测试，我选了一个rest repositories包，选好工具包后，就是选项目根目录了,自己建一个空文件夹，作为该项目的根目录。

创建好后，vscode会自动去下载相关的包，之后的项目结构是：
```sh
➜  demo tree      
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
│   │   │               └── DemoApplication.java
│   │   └── resources
│   │       └── application.properties
│   └── test
│       └── java
│           └── com
│               └── example
│                   └── demo
│                       └── DemoApplicationTests.java
└── target
    ├── classes
    │   ├── application.properties
    │   └── com
    │       └── example
    │           └── demo
    │               └── DemoApplication.class
    └── test-classes
        └── com
            └── example
                └── demo
                    └── DemoApplicationTests.class

21 directories, 10 files
```
在src/main/java/com/example/demo 目录下新建一个Hello.java文件，内容如下：
```java
@RestController
@RequestMapping("/user") 
public class Hello {
    @RequestMapping("/hello")
    public  String Hello(){
        return "Hello World";
    }
}
```
选中 `@RequestMapping("/hello")`按crtl+shift+o 会自动加载包，同时自己给类加个包名。`package com.example.demo;`,之后代码如下：
```java
package com.example.demo;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;

@RestController
@RequestMapping("/user") 
public class Hello {
    @RequestMapping("/hello")
    public  String Hello(){
        return "Hello World";
    }
}
```
之后在编辑器中点右键，选择run即可运行项目，之后在浏览器中打开`http://localhost:8080/user/hello`就能看到`Hello World`

## maven 打包
```sh
$ mvn clean package

[INFO] Results:
[INFO] 
[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] 
[INFO] --- maven-jar-plugin:3.2.0:jar (default-jar) @ demo ---
[INFO] Building jar: /mnt/d/projects/java/test2/demo2/demo/target/demo-0.0.1-SNAPSHOT.jar
[INFO] 
[INFO] --- spring-boot-maven-plugin:2.4.2:repackage (repackage) @ demo ---
[INFO] Replacing main artifact with repackaged archive
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
[INFO] Total time:  11.720 s
[INFO] Finished at: 2021-01-26T12:42:35+08:00
[INFO] ------------------------------------------------------------------------

$ java -jar target/demo-0.0.1-SNAPSHOT.jar

2021-01-26 12:44:39.326  INFO 21527 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat initialized with port(s): 8080 (http)
2021-01-26 12:44:39.344  INFO 21527 --- [           main] o.apache.catalina.core.StandardService   : Starting service [Tomcat]
2021-01-26 12:44:39.344  INFO 21527 --- [           main] org.apache.catalina.core.StandardEngine  : Starting Servlet engine: [Apache Tomcat/9.0.41]
2021-01-26 12:44:39.455  INFO 21527 --- [           main] o.a.c.c.C.[Tomcat].[localhost].[/]       : Initializing Spring embedded WebApplicationContext
2021-01-26 12:44:39.455  INFO 21527 --- [           main] w.s.c.ServletWebServerApplicationContext : Root WebApplicationContext: initialization completed in 2930 ms
2021-01-26 12:44:40.346  INFO 21527 --- [           main] o.s.s.concurrent.ThreadPoolTaskExecutor  : Initializing ExecutorService 'applicationTaskExecutor'
2021-01-26 12:44:40.982  INFO 21527 --- [           main] o.s.b.w.embedded.tomcat.TomcatWebServer  : Tomcat started on port(s): 8080 (http) with context path ''
2021-01-26 12:44:41.001  INFO 21527 --- [           main] com.example.demo.DemoApplication         : Started DemoApplication in 5.62 seconds (JVM running for 6.384)
```
请无视我的目录结构，由于刚刚接触Java，调试了一会儿再弄好，目录有些乱。
在浏览器中打开`http://localhost:8080/user/hello`就能看到`Hello World`，OK，简单调试完成。

## 结语
为啥要用vscode来折腾呢，因为我平常用vscode做go开发，正好有个项目只有java的包，就来学习一下Java，然后把功能做成RPC远程调用函数，就可以继续快乐的玩go了。
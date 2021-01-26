---
title: "启动 WSL 2时警告“参考的对象类型不支持尝试的操作”"
date: 2021-01-23T00:15:05+08:00
draft: false
toc: false
images:
categories: 
  - errors
tags: 
  - wsl
---

![](https://i.loli.net/2019/12/30/8tUdMrmAfWqKHTw.png)

出现图中所示错误的原因是 代理软件与 wsl2 的端口冲突。

使用 NoLsp.exe [下载链接](http://www.proxifier.com/tmp/Test20200228/NoLsp.exe) 如果浏览器打不开 可以用迅雷之类的来下载

使用管理员身份运行以下命令:
```shell
NoLsp.exe C:\Windows\system32\wsl.exe
```

> 参数为 wsl 的绝对路径（默认为 C:\Windows\system32\wsl.exe）

问题原因及解决方案的讨论见 [Gihub Issue](https://github.com/microsoft/WSL/issues/4177#issuecomment-597736482)
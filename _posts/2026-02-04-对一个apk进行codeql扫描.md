---
title:  "对一个apk进行codeql扫描"
excerpt: "对一个apk进行codeql扫描"
author: "lcc"
layout: single
tags:
  - java
  - 自动化代码审计
  - codeql
categories:
  - web安全
---

# 对一个apk进行codeql扫描

目标：我想要研究某个jar包是否存在指定的漏洞

大致步骤：

- 反编译apk包
- 使用codeql创建数据库
- 自定义ql语句进行扫描



## 反编译apk包

使用jadx-gui加载apk然后选择导出项目即可



## 创建数据库





如果出现缺少大量java文件，可能是jdk版本不一致，保持运行codeql的jdk版本、java_home以及运行jadx-gui的java版本一致，也可能是使用jadx-gui反编译为java源代码时存在编码或其他问题，根据以下配置尝试：

![image-20250710204659656](../assets/images/web_sec/image-20250710204659656.png)

![image-20250710204723833](../assets/images/web_sec/image-20250710204723833.png)

![image-20250710204741918](../assets/images/web_sec/image-20250710204741918.png)

![image-20250710204748504](../assets/images/web_sec/image-20250710204748504.png)



![image-20250710204755000](../assets/images/web_sec/image-20250710204755000.png)
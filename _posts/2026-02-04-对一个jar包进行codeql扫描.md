---
title:  "对一个jar包进行codeql扫描"
excerpt: "对一个jar包进行codeql扫描"
author: "lcc"
layout: single
tags:
  - java
  - 自动化代码审计
  - codeql
categories:
  - web安全
---
# 对一个jar包进行codeql扫描

目标：我想要研究某个jar包是否存在指定的漏洞

大致步骤：

- 反编译jar包
- 使用codeql创建数据库
- 使用ql语句进行扫描

主要聚焦前两步



## 反编译jar包



下载fernflower：https://github.com/fesh0r/fernflower.git

使用gradle进行构建然后获取jar包

```
./gradlew :installDist
```

jar包位置在：`build/install`路径下，找到对应的jar包即可

```
java -jar fernflower.jar [-<option>=<value>]* [<source>]+ <destination>
```

更多操作，如反混淆可以看readme



示例：

```
D:\Local_AI_auxiliary_system\envs\java\jdk17\bin\java.exe -jar fernflower.jar -dgs=1 fastjson-1.2.83.jar .\fastjson\1.2.83
```

注意要新建`/fastjson/1.2.68/`目录

反编译完成后将jar包解压即可



## 创建数据库

### 报错

#### Finalizing database at xxxxxx

CodeQL detected code written in Java/Kotlin but could not process any of it. For more information, review our troubleshooting guide at https://gh.io/troubleshooting-code-scanning/no-source-code-seen-during-build.

对反编译后的项目进行maven构建，最后没有一个class文件，所以此处错误应该就是由于maven构建出错



因此采用不构建的模式：

```
codeql database create database\fastjson_1_2_83 --language=java -j 4 --source-root=xxxx --build-mode=none
```


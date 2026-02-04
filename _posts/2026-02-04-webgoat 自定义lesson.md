---
title:  "webgoat 自定义lesson"
excerpt: "webgoat 自定义lesson"
author: "lcc"
layout: single
tags:
  - 早期文档
categories:
  - web安全
---
# webgoat 自定义lesson



以文件上传漏洞为例，首先在lesson目录下新建一个upload目录，然后在Category.java(src\main\java\org\owasp\webgoat\container\lessons\Category.java)中添加枚举变量，比如：

```
TEST1("(T1) File upload vulnerability", 311)  ,
```

然后在upload目录下新建一个UPLOAD类，按如下方式写：

upload/UPLOAD.java:

```
package org.owasp.webgoat.lessons.upload;

import org.owasp.webgoat.container.lessons.Category;
import org.owasp.webgoat.container.lessons.Lesson;
import org.springframework.stereotype.Component;

@Component
public class UPLOAD extends Lesson {
    @Override
    public Category getDefaultCategory() {
        return Category.TEST1;
    }

    @Override
    public String getTitle() {
        return "文件上传漏洞";
    }
}
```

重新编译运行项目，将在页面中看到一个新的lesson



添加新的Controller用于处理文件上传：

upload/SimpleUpload.java:

```
package org.owasp.webgoat.lessons.upload;

import org.apache.commons.io.FileUtils;
import org.owasp.webgoat.container.assignments.AssignmentEndpoint;
import org.owasp.webgoat.container.assignments.AttackResult;
import org.springframework.web.bind.annotation.*;
import org.springframework.web.multipart.MultipartFile;

import java.io.File;
import java.io.IOException;

@RestController
public class SimpleUpload extends AssignmentEndpoint {
    @PostMapping(path = "/upload/simple_upload")
    @ResponseBody
    public AttackResult upload(@RequestParam("file")MultipartFile file) throws IOException {
        if (file.isEmpty()){
            return failed(this).feedback("file is empty").build();
        }
        //获取文件名
        String filename = file.getOriginalFilename();

        //文件路径
        String filePath = "uploads/"+filename;

        File file1 = new File(filePath);
        if (!file1.exists()) {
            file1.createNewFile();
        }
        FileUtils.copyInputStreamToFile(file.getInputStream(), file1);
        return success(this).build();



    }

}

```



为了给这些lesson添加新的关卡页面，需要在resources/lessons/upload目录下添加以下目录：



- css，复制其他目录下的即可
- document，用于添加到关卡中的内容
- html
- 等等

其中，html目录下需要放和UPLOAD类名一致的文件，UPLOAD.html，在该文件内可以写

```
<!DOCTYPE html>

<html xmlns:th="http://www.thymeleaf.org">

<div class="lesson-page-wrapper">
    <div class="adoc-content" th:replace="~{doc:lessons/upload/documentation/upload_intro.adoc}"></div>
</div>


<div class="lesson-page-wrapper">
    <div class="adoc-content" th:replace="~{doc:lessons/upload/documentation/simple_upload.adoc}"></div>
    <form action="upload/simple_upload" method="post" enctype="multipart/form-data">
        <div><input type="file" multiple="multiple" accept="image/*" name="file"></div>
        <div><input type="submit" value="上传"></div>
    </form>

</div>

```

此处将设置两个关卡，每个关卡都应该在`<div class="lesson-page-wrapper">`标签内部，在内部实现更多的标签可以做更多的操作




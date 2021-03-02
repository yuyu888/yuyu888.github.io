---
layout: mypost
title: SpringBoot快速搭建一个web服务
categories: [JAVA]
---

## 初始化一个SpringBoot包

访问： https://start.spring.io/  下载一个初始化的SpringBoot包

![图例](1614676930010.jpg)

## 配置

### 配置端口：  
在 {$yourpath}/src/main/resources/application.properties 文件中增加  
> server.port=8082

## 第一个web接口

````
cd {$yourpath}/src/main/java/com/myproject/demo/   
mkdir controller  
vim HelloController.java
````
HelloController.java

````JAVA
package com.myproject.demo.Controller;

import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
 
@RestController
public class HelloController {
 
    @RequestMapping("/hello")
    public String hello(){
        return "Hello World!";
    }
}
````

## 运行

> ./mvnw spring-boot:run

## 结果

![图例](1614677808979.jpg)

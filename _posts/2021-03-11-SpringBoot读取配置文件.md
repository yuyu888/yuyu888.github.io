---
layout: mypost
title: SpringBoot读取配置文件
categories: [JAVA]
---

## 准备工作
1、在 src/main/resources/ 下创建文件    
application.properties
````
server.port=8082
env-number=1
# spring.profiles.active=dev
````

2、在 src/main/java/com/myproject/javalearn/config/ 下创建  
BasicConfig.java
````java
package com.myproject.javalearn.config;
import lombok.Data;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;
//import org.springframework.context.annotation.PropertySource;

import java.util.HashMap;
import java.util.Map;

@Data
@Configuration
public class BasicConfig {
    @Value("${env-number}")
    private Integer envNumber;

    @Bean("BasicConfig")
    public Map<String, Integer> BasicConfig() {
        Map<String, Integer> retMap = new HashMap<>();
        retMap.put("envNumber", envNumber);
        return retMap;
    }
}
````

3、在controller 里新增获取config的的方法，
src/main/java/com/myproject/javalearn/controller/HelloController.java

````java
package com.myproject.javalearn.controller;

import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.web.bind.annotation.RequestMapping;
import org.springframework.web.bind.annotation.RestController;
import com.myproject.javalearn.config.BasicConfig;

import java.util.Map;

@Slf4j
@RestController
public class HelloController {
    @Autowired
    private Map<String, Integer> BasicConfig;

    @RequestMapping("/hello")
    public String hello(){
        return "Hello World!";
    }

    @RequestMapping("/config")
    public String config(){
        log.info("1111");
        Integer envNumber = BasicConfig.get("envNumber");
        return "envNumber:"+envNumber;
    }
}


````

通过：http://localhost:8082/config 读取配置信息
成功输出：
![图例](1615446153339.jpg)


## 通过application.yml 读取配置
springboot 默认加载 src/main/resources/application.properties   
不过 我们一般喜欢使用 yml 格式的配置文件，我们只需要再 src/main/resources/ 下建立一个    
application.yml
````yml
server:
  port: 8082
env-number: 2
````

通过：http://localhost:8082/config 读取配置信息
输出：
> envNumber:1

结果不是我们预期值：envNumber:2

由此我们可以看出， **如果两个文件同时同存在，application.properties 配置优先级高于 application.yml**

为了让application.yml文件中的配置项「 env-number: 2 」生效；  
1、把 application.properties 中的「 env-number=1 」 注释掉
````
server.port=8082
#env-number=1
````
则成功输出： 
> envNumber:2

2、把application.properties 重命名为application.properties2 或者application2.properties  或者直接删除  
则成功输出： 
> envNumber:2

3、保留application.properties 并不改变任何配置；在 application.yml 里新增一个配置

    config-test: 1112

修改对应的 BasicConfig.java 和 HelloController.java 文件 把该配置打印出来

成功输出
> envNumber:1 configTest:1112



## 区分测试环境与线上环境做配置

application.properties 文件中新增一个配置：  

    spring.profiles.active=dev

并新增文件 application-dev.yml
````
env-number: 3
config-test: 3333
````

application-prod.yml
````
env-number: 4
config-test: 4444
````

访问接口 输出：
> envNumber:3 configTest:3333

**application-dev.yml 的配置会覆盖 application.properties 和 application.yml 里的相同的配置信息**

## 读取指定文件
新增文件 src/main/resources/custom.yml
````
env-number: 5
config-test: 5555
````
BasicConfig.java 里增加一个注解 @PropertySource("classpath:custom.yml")

![图例](1615449807181.jpg)

访问接口 输出：
> envNumber:3 configTest:3333

没有生效， 当把所有 application.properties 和 application.yml， application-dev.yml 中的 关于 env-number config-test 的配置全都去掉的时候  
访问接口成功 输出
> envNumber:5 configTest:5555

**当使用@PropertySource 指定读取文件的时候， 如果之前有配置， 则该文件中的配置不生效**

目录结构  
![图例](1615452364638.jpg)
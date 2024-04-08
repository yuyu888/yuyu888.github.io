---
layout: mypost
title: 通过maven创建FlinkJob并打包，发布
categories: [FlinkCDC, JAVA]
---

## 环境准备

先搭建好flink，参考[这篇](https://yuyu888.github.io/posts/2024/03/28/FlinkCDC%E5%90%8C%E6%AD%A5mysql%E6%95%B0%E6%8D%AE%E8%87%B3mysql.html){:target="_blank"} 里的 “Flink 搭建”

## 创建一个MAVEN项目

项目创建完毕后的目录结构如图：

![目录结构](image1.png)

### pom.xml

 ````xml
 <?xml version="1.0" encoding="UTF-8"?>
<project xmlns="http://maven.apache.org/POM/4.0.0"
         xmlns:xsi="http://www.w3.org/2001/XMLSchema-instance"
         xsi:schemaLocation="http://maven.apache.org/POM/4.0.0 http://maven.apache.org/xsd/maven-4.0.0.xsd">
    <modelVersion>4.0.0</modelVersion>
    <groupId>org.example.flinkcdc</groupId>
    <artifactId>flink-cdc-demo</artifactId>
    <version>1.0-SNAPSHOT</version>

    <properties>
        <java.version>1.8</java.version>
        <maven.compiler.source>${java.version}</maven.compiler.source>
        <maven.compiler.target>${java.version}</maven.compiler.target>
    </properties>

    <dependencies>
        <!-- Flink dependencies -->
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-java</artifactId>
            <version>1.19.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-streaming-java_2.12</artifactId>
            <version>1.14.4</version>
        </dependency>

        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-connector-jdbc_2.12</artifactId>
            <version>1.14.4</version>
        </dependency>

        <!-- Flink CDC dependencies -->
        <dependency>
            <groupId>com.ververica</groupId>
            <artifactId>flink-sql-connector-mysql-cdc</artifactId>
            <version>3.0.0</version>
        </dependency>
        <dependency>
            <groupId>org.apache.flink</groupId>
            <artifactId>flink-table-api-java-bridge</artifactId>
            <version>1.19.0</version>
        </dependency>
    </dependencies>

    <!-- 构建配置 -->
    <build>
        <plugins>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-compiler-plugin</artifactId>
                <version>3.8.1</version>
                <configuration>
                    <source>${java.version}</source>
                    <target>${java.version}</target>
                </configuration>
            </plugin>
            <plugin>
                <groupId>org.apache.maven.plugins</groupId>
                <artifactId>maven-shade-plugin</artifactId>
                <version>3.2.4</version>
                <executions>
                    <execution>
                        <phase>package</phase>
                        <goals>
                            <goal>shade</goal>
                        </goals>
                        <configuration>
                            <finalName>flink-cdc-project</finalName>
                            <filters>
                                <filter>
                                    <artifact>*:*</artifact>
                                    <excludes>
                                        <exclude>META-INF/*.SF</exclude>
                                        <exclude>META-INF/*.DSA</exclude>
                                        <exclude>META-INF/*.RSA</exclude>
                                    </excludes>
                                </filter>
                            </filters>
                        </configuration>
                    </execution>
                </executions>
            </plugin>
        </plugins>
    </build>
</project>


 ````
 
### FlinkMysqlToMysql  
 
 ````java
import libs.ResetValueFunction;
import org.apache.flink.api.common.restartstrategy.RestartStrategies;
import org.apache.flink.api.common.time.Time;
import org.apache.flink.configuration.Configuration;
import org.apache.flink.streaming.api.environment.StreamExecutionEnvironment;
import org.apache.flink.table.api.StatementSet;
import org.apache.flink.table.api.bridge.java.StreamTableEnvironment;

import java.util.concurrent.TimeUnit;

public class FlinkMysqlToMysql {
    public static void main(String[] args) throws Exception {
        StreamExecutionEnvironment env = StreamExecutionEnvironment.getExecutionEnvironment();

        // flink程序在开发环境已经运行成功的情况下，部署到独立的flink集群（start-cluster）中，可能遇到不能正常运行的情况。
//        env.setRestartStrategy(RestartStrategies.noRestart()); // 不重启
        env.setRestartStrategy(RestartStrategies.failureRateRestart(
                3, // 一个时间段内的最大失败次数
                Time.of(5, TimeUnit.MINUTES), // 衡量失败次数的是时间段
                Time.of(3, TimeUnit.SECONDS) // 间隔
        ));
        StreamTableEnvironment tableEnv = StreamTableEnvironment.create(env);
        Configuration configuration = tableEnv.getConfig().getConfiguration();
        configuration.setString("pipeline.name", "mysql2mysql-test"); // 设置jobname


        tableEnv.createTemporarySystemFunction("ResetValue", ResetValueFunction.class);

        // 从数据表a读取数据
        tableEnv.executeSql(
                "CREATE TABLE `logtest1`("
                        + "  `id` INT NOT NULL,"
                        + "  `uid` INT NOT NULL,"
                        + "  `operate_type` INT NOT NULL,"
                        + "  `opeate_detail` STRING NOT NULL,"
                        + "  `create_time` TIMESTAMP NOT NULL,"
                        + "  PRIMARY KEY (`id`) NOT ENFORCED "
                        + ") WITH ("
                        + " 'connector' = 'mysql-cdc', "
                        + " 'hostname' = '127.0.0.1',"
                        + " 'port' = '3306', "
                        + " 'username' = 'root', "
                        + "  'password' = '123456', "
                        + " 'database-name' = 'mytest', "
                        + " 'table-name' = 'logtest1' "
                        + ")"
        );

        // 将数据写入数据表b
        tableEnv.executeSql(
                "CREATE TABLE `logtest`("
                        + "  `id` INT NOT NULL, "
                        + "  `uid` INT NOT NULL, "
                        + "  `operate_type` INT NOT NULL, "
                        + "  `opeate_detail` STRING NOT NULL,"
                        + "  `create_time` TIMESTAMP NOT NULL,"
                        + "  PRIMARY KEY (`id`) NOT ENFORCED "
                        + ") WITH ("
                        // 这里添加你的数据库连接参数
                        + "  'connector' = 'jdbc', "
                        + "  'url' = 'jdbc:mysql://127.0.0.1:3306/test?characterEncoding=utf8&useSSL=true&serverTimezone=Asia/Shanghai', "
                        + "  'driver' = 'com.mysql.cj.jdbc.Driver', "
                        + "  'username' = 'root', "
                        + "  'password' = '123456', "
                        + "  'table-name' = 'logtest' "
                        + ")"
        );


//        StatementSet statementSet = tableEnv.createStatementSet();
//        statementSet.addInsertSql(
//                "insert into logtest select id, uid, operate_type,  ResetValue(opeate_detail), create_time from logtest1 where operate_type=1"
//        );
//        statementSet.execute();

        // 对数据进行处理并写入数据表b
        tableEnv.executeSql(
                "insert into logtest select id, uid, operate_type,  ResetValue(opeate_detail), create_time from logtest1 where operate_type=1"
        );
    }
}

 ````

 这里我用了 ResetValue 这个自定义方法来清洗数据

 需要 通过 tableEnv.createTemporarySystemFunction("ResetValue", ResetValueFunction.class);注册后才能用

 >         tableEnv.createTemporarySystemFunction("ResetValue", ResetValueFunction.class);  

### ResetValueFunction

````java
package libs;

import org.apache.flink.table.annotation.DataTypeHint;
import org.apache.flink.table.annotation.InputGroup;
import org.apache.flink.table.functions.ScalarFunction;

public class ResetValueFunction extends ScalarFunction {
    // 接受任意类型输入，返回 String 型输出
    public String eval(@DataTypeHint(inputGroup = InputGroup.ANY) Object o) {
        return String.valueOf(o)+"yyyyyy";
    }
}
````

### 目录结构图

最终的目录结构如下：  
![目录结构](image2.png)


## 打包

用 IDEA 的话直接执行 mvn clean package 会在target目录生成一个flink-cdc-project.jar 包

![maven-clean-package](image3.png)

## 上传 job

点这里上传 刚才生成的jar包 

![uploadjar](image4.png)

## submit job

上传成功后可以看到刚才上传的jar 包

![joblist](image6.png)


点击改jar 后， 填入 entry class： FlinkMysqlToMysql 点提交
![submit job](image5.png)

之后看到job 成功运行！

![running job](image7.png)


经过测试 插入/修改 mytest.logtest1； 成功同步到 test.logtest1；

大功告成！
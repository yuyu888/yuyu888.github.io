---
layout: mypost
title: Mybatis多租户解决方案
categories: [JAVA]
---

## 前言

当前项目是一个公司内部的办公系统， 随着公司业务的扩展， 不断有新的公司分离出去，也有些新的公司被收购纳入旗下， 所以原先的系统需要改造，以适用多个公司使用， 各个公司数据需要被隔离， 同时部分业务还可以综合（不考虑数据隔离）查询， 这就需要在现有的基础上，实施改造，同时希望成本尽可能的低！

## 多租户

多租户技术（英语：multi-tenancy technology）或称多重租赁技术，是一种软件架构技术，它是在探讨与实现如何于多用户的环境下共用相同的系统或程序组件，并且仍可确保各用户间数据的隔离性。

多租户技术可以实现多个租户之间共享系统实例，同时又可以实现租户的系统实例的个性化定制。通过使用多租户技术可以保证系统共性的部分被共享，个性的部分被单独隔离。通过在多个租户之间的资源复用，运营管理维护资源，有效节省开发应用的成本。

多租户技术的实现重点，在于不同租户间应用程序环境的隔离（application context isolation）以及数据的隔离（data isolation)，以维持不同租户间应用程序不会相互干扰，同时数据的保密性也够强。

## MybatisPlus的TenantLineInnerInterceptor插件使用

### MAVEN

版本太低不行

````
		<dependency>
			<groupId>com.baomidou</groupId>
			<artifactId>mybatis-plus-boot-starter</artifactId>
			<version>3.5.3.1</version>
		</dependency>
````

### TenantLineInnerInterceptor 的使用

本质上是对 TenantLineHandler 实现

TenantLineHandler
````java
package com.baomidou.mybatisplus.extension.plugins.handler;

public interface TenantLineHandler {
    net.sf.jsqlparser.expression.Expression getTenantId();

    default java.lang.String getTenantIdColumn() { /* compiled code */ }

    default boolean ignoreTable(java.lang.String tableName) { /* compiled code */ }

    default boolean ignoreInsert(java.util.List<net.sf.jsqlparser.schema.Column> columns, java.lang.String tenantIdColumn) { /* compiled code */ }
}
````

在MybatisPlusConfig类中添加多租户插件
````

import com.baomidou.mybatisplus.extension.plugins.MybatisPlusInterceptor;
import com.baomidou.mybatisplus.extension.plugins.handler.TenantLineHandler;
import com.baomidou.mybatisplus.extension.plugins.inner.TenantLineInnerInterceptor;
import net.sf.jsqlparser.expression.Expression;
import net.sf.jsqlparser.expression.StringValue;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.context.annotation.Bean;
import org.springframework.context.annotation.Configuration;

@Configuration
public class MybatisPlusMultiTenancyConfig {
    /* 租户管理器 */
    @Autowired
    private TenantManger tenantManager;

    @Bean
    public MybatisPlusInterceptor tenantInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();
        // 多租户插件
        TenantLineInnerInterceptor tenantInterceptor = new TenantLineInnerInterceptor();
        tenantInterceptor.setTenantLineHandler(new TenantLineHandler() {
            @Override
            public Expression getTenantId() {
                // 返回当前用户的租户ID
//                return new LongValue(tenantManager.getTenantId());
                return new StringValue(tenantManager.getTenantId());
            }

            @Override
            public String getTenantIdColumn(){
                // 返回当前用户的租户ID字段
                return tenantManager.getTenantIdColumn();
            }

            @Override
            public boolean ignoreTable(String tableName) {
//                return true;
                return tenantManager.getIgnore(tableName);
            }
        });
        interceptor.addInnerInterceptor(tenantInterceptor);
        return interceptor;
    }

}

````

这里我通过 TenantManger 来管理，设置并获取租户id， 租户字段， 以及是否需要使用租户插件


###  TenantManger 的实现
未来保证线程安全， 这里用到了 ThreadLocal 来存储租户id， 租户字段， 以及是否需要使用租户插件的设置

TenantMangerTenantIdStorage
````java
import org.springframework.stereotype.Component;

@Component
public class TenantMangerTenantIdStorage {
    private final ThreadLocal<String> TenantId = new ThreadLocal<>();

    public void setTenantId(String tenantId){
        TenantId.set(tenantId);
    }

    public String getTenantId(){
        return  TenantId.get();
    }

    public void remove(){
        TenantId.remove();
    }
}
````

TenantMangerTenantIdColumnStorage
```` java
import org.springframework.stereotype.Component;

@Component
public class TenantMangerTenantIdColumnStorage {
    private final ThreadLocal<String> TenantId = new ThreadLocal<>();

    public void setTenantIdColumn(String columnName){
        TenantId.set(columnName);
    }

    public String getTenantIdColumn(){
        return  TenantId.get();
    }

    public void remove(){
        TenantId.remove();
    }
}
````

TenantMangerIgnoreStorage
```` java
import org.springframework.stereotype.Component;

@Component
public class TenantMangerIgnoreStorage {
    private final ThreadLocal<Boolean> TenantId = new ThreadLocal<>();

    public void setIgnore(Boolean ignore){
        TenantId.set(ignore);
    }

    public Boolean getIgnore(){
        return  TenantId.get();
    }

    public void remove(){
        TenantId.remove();
    }
}
````

TenantManger 的实现

```` java
import org.apache.commons.lang.StringUtils;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

@Component
public class TenantManger {

    @Autowired
    private TenantMangerTenantIdStorage tenantMangerTenantIdStorage;

    @Autowired
    private TenantMangerTenantIdColumnStorage tenantMangerTenantIdColumnStorage;

    @Autowired
    private TenantMangerIgnoreStorage tenantMangerIgnoreStorage;

    @Autowired
    private TenantContext tenantContext;

    final String tenantId = "0";
    final String tenantIdColumn = "tenant_id";
    final Boolean ignore = true; // true:不启用， false：启用

    /**
     * 获取租户 ID 值表达式，只支持单个 ID 值
     * @return 租户 ID 值表达式
     */
    public String getTenantId() {
        if(StringUtils.isNotEmpty(tenantMangerTenantIdStorage.getTenantId())){
                return tenantMangerTenantIdStorage.getTenantId();
        }

        // 获取系统设置的租户
        // String sysTenantId = tenantContext.getTenantId();
        // if(StringUtils.isNotEmpty(sysTenantId)){
        //     return sysTenantId;
        // }
        return  tenantId;
    }

    /**
     * 获取租户字段名。默认字段名叫: tenant_id
     * @return 租户字段名
     */
    public String getTenantIdColumn() {
        if(StringUtils.isNotEmpty(tenantMangerTenantIdColumnStorage.getTenantIdColumn())){
            return tenantMangerTenantIdColumnStorage.getTenantIdColumn();
        }
        return tenantIdColumn;
    }

    /**
     * 根据表名判断是否忽略拼接多租户条件。 默认都要进行解析并拼接多租户条件
     * @return 是否忽略, true:表示忽略，false:需要解析并拼接多租户条件
     */
    public boolean getIgnore(String tableName)
    {
        if(tenantMangerIgnoreStorage.getIgnore()!=null){
            return tenantMangerIgnoreStorage.getIgnore();
        }
        if(TenantTables.useTenantList.contains(tableName)){
            return false;
        }
        if(TenantTables.unUseTenantList.contains(tableName)){
            return true;
        }

//        return userIgnore(tableName);
        return ignore;
    }

    private Boolean userIgnore(String tableName){
        // todo 根据用户身份判断是否需要使用租户字段查询
        return true;
    }

    public String getCustomTenantId() {
        return tenantMangerTenantIdStorage.getTenantId();
    }

    public void setCustomTenantId(String id) {
        tenantMangerTenantIdStorage.setTenantId(id);
    }

    public void removeCustomTenantId() {
        tenantMangerTenantIdStorage.remove();
    }

    public String getCustomTenantIdColumn() {
        return tenantMangerTenantIdColumnStorage.getTenantIdColumn();
    }

    public void setCustomTenantIdColumn(String columName) {
        tenantMangerTenantIdColumnStorage.setTenantIdColumn(columName);
    }

    public void removeCustomTenantIdColumn() {
        tenantMangerTenantIdColumnStorage.remove();
    }

    public Boolean getCustomIgnore() {
        return tenantMangerIgnoreStorage.getIgnore();
    }

    public void setCustomIgnore(Boolean isIgnore) {
        tenantMangerIgnoreStorage.setIgnore(isIgnore);
    }

    public void removeCustomIgnore() {
        tenantMangerIgnoreStorage.remove();
    }

    public void setStorage(String id, String columName, Boolean isIgnore){
        tenantMangerTenantIdStorage.setTenantId(id);
        tenantMangerTenantIdColumnStorage.setTenantIdColumn(columName);
        tenantMangerIgnoreStorage.setIgnore(isIgnore);
    }

    public void removeStorage(){
        tenantMangerTenantIdStorage.remove();
        tenantMangerTenantIdColumnStorage.remove();
        tenantMangerIgnoreStorage.remove();
    }

}
````

TenantTables
```` java
import java.util.List;

public class TenantTables {
    public static List<String> useTenantList =  List.of(
            "user"
    );

    public static List<String> unUseTenantList =  List.of(
            "visit_log"
    );
}
````

### 通过注解灵活定义

@TenantPlugin

````java
import java.lang.annotation.*;

@Target({ElementType.TYPE, ElementType.METHOD})
@Retention(RetentionPolicy.RUNTIME)
@Documented
public @interface TenantPlugin {
    String TenantIdColumn() default "tenant_id";
    String TenantId() default "DEFAULT";
    boolean Ignore() default false;
}
````

TenantPluginAspect
```` java
import cn.hutool.core.date.DateUtil;
import com.fasterxml.jackson.databind.ObjectMapper;
import com.xxx.infocenter.common.mybatisplus.TenantManger;
import com.xxx.infocenter.common.mybatisplus.annotation.TenantPlugin;
import com.xxx.infocenter.infrastructure.common.exception.RRException;
import lombok.extern.slf4j.Slf4j;
import org.apache.commons.lang.StringUtils;
import org.aspectj.lang.JoinPoint;
import org.aspectj.lang.ProceedingJoinPoint;
import org.aspectj.lang.Signature;
import org.aspectj.lang.annotation.Around;
import org.aspectj.lang.annotation.Aspect;
import org.aspectj.lang.annotation.Before;
import org.aspectj.lang.annotation.Pointcut;
import org.aspectj.lang.reflect.MethodSignature;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.stereotype.Component;

import java.lang.reflect.Method;
import java.util.Date;


@Slf4j
@Aspect
@Component
public class TenantPluginAspect {
    @Autowired
    ObjectMapper objectMapper;

    @Autowired
    private TenantManger tenantManger;

    @Pointcut("@annotation(com.xxx.infocenter.common.mybatisplus.annotation.TenantPlugin)"
            + "|| @within(com.xxx.infocenter.common.mybatisplus.annotation.TenantPlugin)")
    public void tenantPluginPointCut() {

    }

    @Before("@annotation(tenantPlugin)")
    public void before(JoinPoint joinPoint, TenantPlugin tenantPlugin)  throws RRException {

    }

//    @Around("@annotation(tenantPlugin)"+"|| @within(tenantPlugin)")
    @Around("tenantPluginPointCut()")
    public Object around(ProceedingJoinPoint joinPoint) throws Throwable {
        Signature signature = joinPoint.getSignature();
        MethodSignature signatureMethod = (MethodSignature) joinPoint.getSignature();
        Method thisMethod = signatureMethod.getMethod();
        TenantPlugin tenantPlugin = thisMethod.getAnnotation(TenantPlugin.class);
        if(tenantPlugin==null){
            tenantPlugin = joinPoint.getTarget().getClass().getAnnotation(TenantPlugin.class);
        }

        String logString = tenantPlugin.TenantId() + "\t" + tenantPlugin.TenantIdColumn() + "\t" + tenantPlugin.Ignore() + "\t" + signature.getDeclaringTypeName() + "\t" + signature.getName() + "\t";

        if(tenantManger.getCustomTenantId()==null &&  !tenantPlugin.TenantId().equals("DEFAULT")){
            tenantManger.setCustomTenantId(tenantPlugin.TenantId());
        }

        if(tenantManger.getCustomTenantIdColumn()==null && StringUtils.isNotEmpty(tenantPlugin.TenantIdColumn())){
            tenantManger.setCustomTenantIdColumn(tenantPlugin.TenantIdColumn());
        }

        if(tenantManger.getCustomIgnore()==null){
            tenantManger.setCustomIgnore(tenantPlugin.Ignore());
        }


        //方法执行前逻辑处理
        System.out.println("开始");
        logString += "start_time:" + DateUtil.formatDateTime(new Date()) + "\t";
        Object proceedObj = joinPoint.proceed();
        //输出执行结果
        System.out.println(proceedObj.toString());
        //方法执行后逻辑处理
        logString += "end_time:" + DateUtil.formatDateTime(new Date()) + "\t";
        System.out.println("结束");

        tenantManger.removeStorage();
        log.info("===============================");
        log.info(logString);
        return proceedObj;
    }
}

````

测试：

```` java
    public void multiTenancyTest(Integer general) {

        // 默认查询 既不在白名单， 也不在黑名单； 不影响原有逻辑
        // SELECT * FROM hrm_grade LIMIT 1
        QueryWrapper<HrmGradeEntity> queryWrapper1 = new QueryWrapper<HrmGradeEntity>();
        queryWrapper1.last("limit 1");
        List<HrmGradeEntity> hrmGradeEntityList = hrmGradeService.list(queryWrapper1);
        log.info("==================111111==================");
        log.info(String.valueOf(hrmGradeEntityList));

        // 使用了类的注解 方法的注解设置的参数，方法优先级高于类注解的设置的参数，高于默认设置
        // SELECT id, unique_id, c_username, status, gender FROM user WHERE status = 2 LIMIT 1
        QueryWrapper<HecUcUserEntity> queryWrapper2 = new QueryWrapper<HecUcUserEntity>();
        queryWrapper2.select("id, unique_id, c_username, status, gender");
        queryWrapper2.last("limit 1");
        List<HecUcUserEntity> userList = hecUcUserService.list(queryWrapper2);
        log.info("==================2222222==================");
        log.info(String.valueOf(userList));

        // 自定义查询
        // SELECT * FROM hrm_grade WHERE GradeCode = 3333333 LIMIT 1
        tenantManager.setStorage("3333333","GradeCode",false);
        List<HrmGradeEntity> hrmGradeEntityList2 = hrmGradeService.list(queryWrapper1);
        log.info("==================3333333==================");
        log.info(String.valueOf(hrmGradeEntityList2));

        //tenantManager.removeStorage();

        // 由于 上面逻辑没有添加 tenantManager.removeStorage(); 所以对于字段值以及是否忽略的设置依然存在影响
        // SELECT * FROM hrm_grade WHERE GradeCode = 44444 LIMIT 1
        tenantManager.setCustomTenantId("44444");

        List<HrmGradeEntity> hrmGradeEntityList3 = hrmGradeService.list(queryWrapper1);
        log.info("==================44444==================");
        log.info(String.valueOf(hrmGradeEntityList3));

        // 自定义配置优先级超过默认设置，以及注解的配置
        // SELECT id, unique_id, c_username, status, gender FROM user WHERE unique_id = 55555555 LIMIT 1
        tenantManager.setStorage("55555555","unique_id",false);
        List<HecUcUserEntity> userList5 = hecUcUserService.list(queryWrapper2);
        log.info("==================55555555==================");
        log.info(String.valueOf(userList5));

        //　 这里不受影响， 恢复默认， 结果与hrmGradeEntityList1 相同
        // SELECT * FROM hrm_grade LIMIT 1
        List<HrmGradeEntity> hrmGradeEntityList6 = hrmGradeService.list(queryWrapper1);
        log.info("==================6666==================");
        log.info(String.valueOf(hrmGradeEntityList6));
    }

````


## 动态表名插件DynamicTableNameInnerInterceptor

有时候我们可能需要通过不同的租户分配不同的表

```` java
@Configuration
public class MybatisPlusConfig {
    @Bean
    public MybatisPlusInterceptor mybatisPlusInterceptor() {
        MybatisPlusInterceptor interceptor = new MybatisPlusInterceptor();

        //动态表名插件
        DynamicTableNameInnerInterceptor dynamicTableNameInnerInterceptor = new DynamicTableNameInnerInterceptor();
        dynamicTableNameInnerInterceptor.setTableNameHandler((sql, tableName) -> {
            int random = new Random().nextInt(10);
            return tableName + random;
        });
        interceptor.addInnerInterceptor(dynamicTableNameInnerInterceptor);
        return interceptor;
    }
}

````
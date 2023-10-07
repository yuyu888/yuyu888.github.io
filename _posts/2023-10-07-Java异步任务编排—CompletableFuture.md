---
layout: mypost
title: java 异步任务编排—CompletableFuture
categories: [JAVA]
---

## 前言
在实际开发过程中，很多时候因为业务流程非常长，后台执行耗时过久，导致用户在页面操作的体验很差，甚至出现超时的现象。为了解决这一痛点，消息中间件实现流程异步化是个不错的选择，但是如果要汇总多个线程任务的结果，然后再执行主线程，用消息中间实现异步化的方案显得有些捉襟见肘了，这时候我们jdk自带的CompletableFuture，实现多任务编排的业务需求，充分利用系统资源，大大提高业务流程执行中的效率，降低前后台交互过程中响应耗时。

## 什么场景需要CompletableFuture？
当多线程任务出现了相互依赖，需要按照一定的顺序执行的时候，  
比如：当线程1执行完后，线程2才能执行 或 线程2的运行需要使用到线程1的执行结果  
类似这种或者更复杂的情况，就需要用到CompletableFuture多线程任务异步编排  
同时，CompletableFuture对于以下几种情况也是支持的  
1、单个任务  
2、两任务的编排  
3、三任务的编排  
4、多任务的编排  

## 使用案例

### 异步编排主要API介绍

````java
// 执行异步操作，有返回值
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier)
 
// 执行异步操作，有返回值，并且可以使用自定义线程池
public static <U> CompletableFuture<U> supplyAsync(Supplier<U> supplier, Executor executor)
 
// 执行异步操作，无返回值
public static CompletableFuture<Void> runAsync(Runnable runnable)
 
// 执行异步操作，无返回值，并且可以使用自定义线程池
public static CompletableFuture<Void> runAsync(Runnable runnable, Executor executor)
 
// 执行上一个异步操作后新的异步操作，获取上一个任务返回的结果，并返回当前任务的返回值
public <U> CompletableFuture<U> thenApply(Function<? super T,? extends U> fn)
 
// 执行上一个异步操作后新的异步操作，获取上一个任务返回的结果，并返回当前任务的返回值，异步操作
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn)
 
// 执行上一个异步操作后新的异步操作，获取上一个任务返回的结果，并返回当前任务的返回值，异步操作加可以自定义线程池
public <U> CompletableFuture<U> thenApplyAsync(Function<? super T,? extends U> fn, Executor executor)
 
// 消费上一个异步操作处理结果。接收上一个异步操作任务的处理结果，并消费处理结果，无返回结果
public CompletableFuture<Void> thenAccept(Consumer<? super T> action)
 
// 消费上一个异步操作处理结果。接收上一个异步操作任务的处理结果，并消费处理结果，无返回结果，异步操作
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action)
 
// 消费上一个异步操作处理结果。接收上一个异步操作任务的处理结果，并消费处理结果，无返回结果，异步操作，异步操作加可以自定义线程池
public CompletableFuture<Void> thenAcceptAsync(Consumer<? super T> action, Executor executor)
 
// 上一个异步操作任务执行后的业务操作
public CompletableFuture<Void> thenRun(Runnable action)
 
// 上一个异步操作任务执行后的业务操作，异步操作
public CompletableFuture<Void> thenRunAsync(Runnable action)
 
// 上一个异步操作任务执行后的业务操作，异步操作，异步操作加可以自定义线程池
public CompletableFuture<Void> thenRunAsync(Runnable action, Executor executor)
 
// 任务组合执行
public static CompletableFuture<Void> allOf(CompletableFuture<?>... cfs)
 
// 任意一个任务组合执行完就结束执行
public static CompletableFuture<Object> anyOf(CompletableFuture<?>... cfs)

````

### 代码实践  

````java
package com.example.fang.demodruid.service.impl;

import com.example.fang.demodruid.entity.TOrderInfo;
import com.example.fang.demodruid.entity.UserVo;
import com.example.fang.demodruid.mapper.TOrderInfoMapper;
import com.example.fang.demodruid.service.ISysUserService;
import com.example.fang.demodruid.service.ITOrderInfoService;
import com.baomidou.mybatisplus.extension.service.impl.ServiceImpl;
import lombok.extern.slf4j.Slf4j;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.scheduling.concurrent.ThreadPoolTaskExecutor;
import org.springframework.stereotype.Service;

import java.time.LocalDateTime;
import java.time.temporal.ChronoUnit;
import java.util.concurrent.CompletableFuture;

/**
 * <p>
 *  服务实现类
 * </p>
 *
 * @author fangchen
 * @since 2022-03-20
 */
@Service
@Slf4j
public class TOrderInfoServiceImpl extends ServiceImpl<TOrderInfoMapper, TOrderInfo> implements ITOrderInfoService {

    @Autowired
    private ISysUserService iSysUserService;

    @Autowired
    private ThreadPoolTaskExecutor threadPoolExecutor;

    @Override
    public String selectByParallel() {
        LocalDateTime startTime = LocalDateTime.now();
        UserVo userVo = new UserVo();
        //1.开始异步执行业务1
        CompletableFuture<Integer> businessOneCompletableFuture = CompletableFuture.supplyAsync(() -> {
            log.info("开始执行线程1相关逻辑");
            Integer businessOneResult = getBusinessOneResult();
            userVo.setBusinessOne(businessOneResult);
            return businessOneResult;
        }, threadPoolExecutor);
       //2.异步执行业务2,但基于结果1
        CompletableFuture<Void> businessTwoCompletableFuture = businessOneCompletableFuture.thenAcceptAsync(businessOneResult -> {
            log.info("开始执行线程2相关逻辑");
            Integer businessThreeResult = getBusinessTwoResult(businessOneResult);
            userVo.setBusinessTwo(businessThreeResult);
        }, threadPoolExecutor);
       //3.异步执行业务3,基于结果1
        CompletableFuture<Void> businessThreeCompletableFuture = businessOneCompletableFuture.thenAcceptAsync(businessOneResult -> {
            log.info("开始执行线程3相关逻辑");
            Integer businessThreeResult = getBusinessThreeResult(businessOneResult);
            userVo.setBusinessThree(businessThreeResult);
        }, threadPoolExecutor);
       //4.异步执行业务4,基于结果1
        CompletableFuture<Void> businessFourResultCompletableFuture  = businessOneCompletableFuture.thenAcceptAsync(businessOneResult -> {
            log.info("开始执行线程4相关逻辑");
            Integer businessThreeResult = getBusinessFourResult(businessOneResult);
            userVo.setBusinessFour(businessThreeResult);
        }, threadPoolExecutor);
        //5.业务5结果处理,基于结果1
        CompletableFuture<Void> businessFiveResultCompletableFuture = businessOneCompletableFuture.thenAcceptAsync(businessOneResult -> {
            log.info("开始执行线程5相关逻辑");
            Integer businessFiveResult = getBusinessFiveResult(businessOneResult);
            userVo.setBusinessFive(businessFiveResult);
        }, threadPoolExecutor);
        //6.多任务组合执行
        CompletableFuture[] args={businessOneCompletableFuture,businessTwoCompletableFuture,
                                  businessThreeCompletableFuture,businessFourResultCompletableFuture,businessFiveResultCompletableFuture};
        CompletableFuture.allOf(args).join();
        log.info("用户信息:{}",userVo);
        LocalDateTime endTime = LocalDateTime.now();
        log.info("============多任务组合执行共计耗时:{}s================", ChronoUnit.SECONDS.between(startTime, endTime));
        return ChronoUnit.SECONDS.between(startTime, endTime)+"";
    }

    @Override
    // 串形执行
    public String selectBySerial() {
        LocalDateTime startTime = LocalDateTime.now();
        UserVo userVo = new UserVo();
        log.info("开始串行执行相关逻辑");
        //1.业务1结果处理
        Integer businessOneResult = getBusinessOneResult();
        userVo.setBusinessOne(businessOneResult);
        //2.业务2结果处理,基于结果1
        Integer businessTwoResult = getBusinessTwoResult(businessOneResult);
        userVo.setBusinessTwo(businessTwoResult);
        //3.业务3结果处理,基于结果1
        Integer businessThreeResult = getBusinessThreeResult(businessOneResult);
        userVo.setBusinessThree(businessThreeResult);
        //4.业务4结果处理,基于结果1
        Integer businessFourResult = getBusinessFourResult(businessOneResult);
        userVo.setBusinessFour(businessFourResult);
        //5.业务5结果处理,基于结果1
        Integer businessFiveResult = getBusinessFiveResult(businessOneResult);
        userVo.setBusinessFive(businessFiveResult);
        LocalDateTime endTime = LocalDateTime.now();
        log.info("用户信息:{}",userVo);
        log.info("============Serial total execute time:{}s================", ChronoUnit.SECONDS.between(startTime, endTime));
        return ChronoUnit.SECONDS.between(startTime, endTime)+"";
    }


    private Integer getBusinessOneResult() {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return 1;
    }

    private Integer getBusinessTwoResult(Integer businessOneResult) {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return businessOneResult+2;
    }

    private Integer getBusinessThreeResult(Integer businessOneResult) {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return businessOneResult + 3;
    }

    private Integer getBusinessFourResult(Integer businessOneResult) {
        try {
            Thread.sleep(1000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return businessOneResult + 4;
    }

    private Integer getBusinessFiveResult(Integer businessOneResult) {
        try {
            Thread.sleep(2000);
        } catch (InterruptedException e) {
            e.printStackTrace();
        }
        return businessOneResult + 5;
    }

}

````


### 注意
 如果子线程在执行过程中有一个任务异常，主线程还能执行吗？  
不能。因为我们的需求是汇总多个子线程结果后继续执行，如果一个子线程异常当然是不能继续执行主线程的任务。我们可以把这种异常抛出去，然后告知程序异常的原因。  

CompletableFuture可以指定异步处理流程：  
thenAccept()处理正常结果；  
exceptional()处理异常结果；  
thenApplyAsync()用于串行化另一个CompletableFuture  
anyOf()和allOf()用于并行化多个CompletableFuture  
在CompletableFuture执行Task的时候，是需要使用线程池还是用当前的线程去执行。这个需要根据具体的情况来定。使用的时候要尽可能的小心  

## 参考
转载自： https://blog.csdn.net/fangchen1007/article/details/127585657  
其他：  
    https://www.cnblogs.com/ludongguoa/p/15316488.html  
    https://www.jianshu.com/p/06636ae1f26b  
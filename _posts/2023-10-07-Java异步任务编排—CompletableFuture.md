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
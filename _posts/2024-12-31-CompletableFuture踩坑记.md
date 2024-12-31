---
layout: mypost
title: CompletableFuture踩坑记
categories: [JAVA]
---

## 背景

有一个业务，需要做一个批量提交，每一行数据都需要检查，由于检查逻辑比较复杂也比较耗时，所以想采用CompletableFuture做任务编排，实现并行处理

## 情况说明


代码片段

````java
// 去重
        ids = ids.stream().distinct().collect(Collectors.toList());

        List<CompletableFuture<Void>> futures = new ArrayList<>();
        List<Map<String, Object>> errorData = Collections.synchronizedList(new ArrayList<>());
        List<HecFbiProvisionAdProvisionEntity> entityList = Collections.synchronizedList(new ArrayList<>());
        for (Integer id : ids) {
            CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
                // 获取当前线程的上下文类加载器
                // ClassLoader originalClassLoader = Thread.currentThread().getContextClassLoader();

                // 设置当前线程的上下文类加载器为加载当前类的类加载器
                // Thread.currentThread().setContextClassLoader(getClass().getClassLoader());
                try {
                    HecFbiProvisionAdProvisionEntity entity = hecFbiProvisionAdProvisionService.getById(id);

                    List<String> errorList = hecFbiProvisionProvisionCommonLogic.checkData(entity);
                    if (errorList != null && !errorList.isEmpty()) {
                        Map<String, Object> errorInfo = new HashMap<>();
                        errorInfo.put("id", id);
                        errorInfo.put("error_list", errorList);
                        errorData.add(errorInfo);
                    } else {
                        entityList.add(entity);
                    }
                } catch (Exception e) {
                    e.printStackTrace();
                    throw  e;
                }finally {
                    // Thread.currentThread().setContextClassLoader(originalClassLoader);
                }
            });

            futures.add(future);
        }

        // 等待所有异步任务完成
        try {
            CompletableFuture.allOf(futures.toArray(new CompletableFuture[0])).get();
        } catch (InterruptedException e) {
            e.printStackTrace();
            throw new RRException("系统运行错误!!");
        } catch (ExecutionException e) {
            e.printStackTrace();
            throw new RRException("系统运行错误");
        }
````

其中 List<String> errorList = hecFbiProvisionProvisionCommonLogic.checkData(entity); 这个方法里调用了若干feignapi， 运行时会报java.lang.IllegalArgumentException: Could not find class [org.springframework.boot.autoconfigure.condition.OnPropertyCondition] 这个错误，如果不使用CompletableFuture，直接循环执行则不会报错，并且本地运行不报错，一旦放到测试环境就会报错，且部署到开发环境也不会报错；开发环境与测试环境都是用同一个image 使用docker 容器部署(宿主机不一样)， 非常诡异

## 问题原因

询问chatgpt 回答是

> 类加载器问题：当你在异步任务中运行代码时，可能使用了不同的类加载器，导致某些类无法找到。特别是在使用框架（如Spring）的情况下，某些类可能在异步执行环境中不可用。  
>
> 环境差异：直接执行代码和在异步任务中执行代码，可能会处于不同的Spring上下文或配置环境中。确保异步任务中使用的配置和直接执行时一致。

最终解决办法是，添加如下逻辑， 即代码里被注视掉的部分

````java
// 获取当前线程的上下文类加载器
ClassLoader originalClassLoader = Thread.currentThread().getContextClassLoader();

// 设置当前线程的上下文类加载器为系统类加载器
Thread.currentThread().setContextClassLoader(getClass().getClassLoader());

 Thread.currentThread().setContextClassLoader(originalClassLoader);


````

## 另一个解决办法

使用 指定线程池 ThreadPoolTaskExecutor

````java
@Configuration
@EnableAsync
public class ThreadPoolConfig {
	
	@Bean(name = "fooThreadPool")
	public ThreadPoolTaskExecutor fooThreadPool() {
		ThreadPoolTaskExecutor executor = new ThreadPoolTaskExecutor();
		// 设置核心线程数,它是可以同时被执行的线程数量
        executor.setCorePoolSize(10);
        // 设置最大线程数,缓冲队列满了之后会申请超过核心线程数的线程
        executor.setMaxPoolSize(20);
        // 设置缓冲队列容量,在执行任务之前用于保存任务
        executor.setQueueCapacity(1000);
        // 设置线程生存时间（秒）,当超过了核心线程出之外的线程在生存时间到达之后会被销毁
        executor.setKeepAliveSeconds(60);
        // 设置线程名称前缀
        executor.setThreadNamePrefix("fooPool-");
        // 设置拒绝策略
        executor.setRejectedExecutionHandler(new ThreadPoolExecutor.CallerRunsPolicy());
        // 等待所有任务结束后再关闭线程池
        executor.setWaitForTasksToCompleteOnShutdown(true);
        //初始化
        executor.initialize();
		return executor;
	}
}
````

````java
    @Autowired
    @Qualifier("fooThreadPool")
    private ThreadPoolTaskExecutor taskExecutor;
````

引入taskExecutor， 然后   

````java
CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
  // 业务代码
  },taskExecutor);
````

实际改造如下  

````java

  CompletableFuture<Void> future = CompletableFuture.runAsync(() -> {
      try {
          HecFbiProvisionAdProvisionEntity entity = hecFbiProvisionAdProvisionService.getById(id);
          List<String> errorList = hecFbiProvisionProvisionCommonLogic.checkData(entity);
          if (errorList != null && !errorList.isEmpty()) {
              Map<String, Object> errorInfo = new HashMap<>();
              errorInfo.put("id", id);
              errorInfo.put("error_list", errorList);
              errorData.add(errorInfo);
          } else {
              entityList.add(entity);
          }
      } catch (Exception e) {
          e.printStackTrace();
          throw  e;
      }
  }, taskExecutor);
````


## 知识点

````md
在使用 `CompletableFuture.runAsync()` 时，代码会在一个新的线程中运行，而这个线程的上下文类加载器可能与主线程不同。这可能导致一些类在新线程中无法正确加载，尤其是在使用诸如 Spring Boot 这样的框架时。

### 原因分析
上下文类加载器：
Java线程有一个上下文类加载器（Context ClassLoader），它用于加载类和资源。默认情况下，`CompletableFuture.runAsync()` 使用的线程池可能会使用与主线程不同的类加载器。
`ClassLoader.getSystemClassLoader()`：
`ClassLoader.getSystemClassLoader()` 获取的是系统类加载器。这通常是应用程序类加载器，但在某些环境下（如某些应用服务器或特殊的启动脚本），它可能无法加载应用程序中定义的类或依赖项。
`getClass().getClassLoader()`：
`getClass().getClassLoader()` 获取的是加载当前类的类加载器。这通常是应用程序类加载器或其子类加载器，能够正确加载应用程序的所有类和资源。
### 解决方法的解释
当你在新线程中设置 `Thread.currentThread().setContextClassLoader(getClass().getClassLoader())`，你确保新线程使用的类加载器与加载当前类的类加载器相同。这保证了所有应用程序的类和资源都可以正常加载，包括通过 Feign 调用的部分。
之所以 `getClass().getClassLoader()` 能解决问题，是因为它指向的类加载器具有访问应用程序代码和依赖项的权限，而 `ClassLoader.getSystemClassLoader()` 在某些环境下则不具备这种能力。
### 建议

对于使用多线程的应用程序，特别是涉及到复杂框架（如 Spring）的情况，确保在新线程中设置正确的上下文类加载器是非常重要的。这样可以避免类加载问题，确保所有组件在不同线程中正常工作。
````
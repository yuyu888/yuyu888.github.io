---
layout: mypost
title: FeignApi通过header头传递tenantId
categories: [JAVA]
---

## 前言

承接上一章： [Mybatis多租户解决方案](https://yuyu888.github.io/posts/2023/03/27/MybatisPlus%E5%A4%9A%E7%A7%9F%E6%88%B7%E8%A7%A3%E5%86%B3%E6%96%B9%E6%A1%88.html)； 我们的数据有时候需要通过feignapi调用其他服务， 同时要把tenantId传递过去，又不能改变feignapi的调用参数

## 数据发送

这里是通过RequestInterceptor拦截器实现的

````java
@Slf4j
// @Configuration 可以通过 @FeignClient 中的 configuration指定，而不需要全局生效
public class FeignConfiguration {

	@Autowired
	private TenantContext tenantContext;
	
	@Bean
    public RequestInterceptor headerInterceptor() {
        return new RequestInterceptor() {
            @Override
            public void apply(RequestTemplate template) {
            	String tenantId = tenantContext.getTenantId(); // 这里是获取的 ThreadLocal 的设定好的数据
            	log.info("Feign RequestInterceptor::tenantId:" + tenantId);
                template.header("feign_api_tenant_id", tenantId);
            }
        };
    }
}
````

## 接收参数

通过注解+切面的方式，先获取到tenantId 再放到 ThreadLocal 里需要的时候取 这样就不用改feignapi的具体实现了, 具体实现略


从http header 中获取租户id
````java
		HttpServletRequest request = ((ServletRequestAttributes) RequestContextHolder.getRequestAttributes()).getRequest();
		String tenantId = request.getHeader("feign_api_tenant_id");
````


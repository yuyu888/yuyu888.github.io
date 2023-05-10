---
layout: mypost
title: SpringCloud获取nacos里注册的服务实例
categories: [JAVA]
---

有时候我们需要获取所有注册进来的服务实例，有时候需要知道本次负载均衡到那台机器上了

````java


    @Autowired
    private DiscoveryClient discoveryClient;

    @Autowired
    private LoadBalancerClient loadBalancerClient;

    public void getInstance(){
        //这里的 xxxservice 即为nacos控制面板上的服务名
        List<ServiceInstance> serviceInstanceList = discoveryClient.getInstances("xxxservice");
        serviceInstanceList.forEach(serviceInstance -> {
            //获取host
            serviceInstance.getHost();
            //获取端口号
            serviceInstance.getPort();
            serviceInstance.getServiceId();
            log.info("================xxxxxx==================");
            log.info(String.valueOf(serviceInstance.getHost()));
            log.info(String.valueOf(serviceInstance.getPort()));
            log.info(String.valueOf(serviceInstance.getServiceId()));

        });

        // 　获取负载均衡选择的实例
        ServiceInstance instance = loadBalancerClient.choose("xxxservice");
        instance.getHost();
        instance.getPort();
        log.info("================666666==================");
        log.info(String.valueOf(instance.getHost()));
        log.info(String.valueOf(instance.getPort()));
    }

````


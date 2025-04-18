---
layout: mypost
title: ThreadLocalCache
categories: [JAVA]
---

## 前言

有些代码历史比较长，链路也比较长，有改动的时候，希望获取新的变量，以支持逻辑修改；通过传递参数的方式， 改动就太大了， 风险也比较高， 适当的使用ThreadLocal， 可以一定程度的解决问题，并减少改动；不过也要慎重， 用完就清理；甚至为了防止干扰， 进入的时候就先清理一遍

## ThreadLocalCache.Class

````java
import java.util.HashMap;
import java.util.Map;

/**
 * ThreadLocalCache
 * 
 */
public class ThreadLocalCache {

    /**
     * 实例字段，每个线程一个store，每个线程生产一个{@code ThreadLocalCache} INSTANCE
     */

    private static final ThreadLocal<Map<Object, Object>> store = ThreadLocal.withInitial(HashMap::new);

    public static void put(Object key, Object value) {
        store.get().put(key, value);
    }

    public static Object get(Object key) {
        return store.get().get(key);
    }
    public static void remove(){
        store.remove();
    }
}
````
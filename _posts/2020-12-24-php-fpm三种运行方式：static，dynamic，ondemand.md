---
layout: mypost
title: php-fpm 三种运行方式：static，dynamic，ondemand
categories: [PHP]
---

php-fpm的进程数可以根据设置分为动态和静态
>- 静态：直接开启指定数量的php-fpm进程，不再增加或者减少；配置：pm=static
>- 动态：开始的时候开启一定数量的php-fpm进程，当请求量变大的时候，动态的增加php-fpm进程数到上限，当空闲的时候自动释放空闲的进程数到一个下限。配置：pm=dynamic/ondemand

下面我们将分别阐述php-fpm的进程数管理的三种不同模式

## pm=static
如果pm设置为static，那么只有**pm.max_children**这个参数生效, 系统会创建设置的数量个php-fpm进程;

如果设置 pm.max_children=500 那么系统会创建500个php-fpm进程来监听nginx过来的请求，假如我当前的请求数是100，那么会有400个闲置，突然流量又增加了300， 那么服务器依然有足够多的php-fpm进程来处理请求，不用创建新的进程，请求来了也能迅速处理；

设置pm=static时需要注意配置pm.max_requests参数；该参数含义是每个进程处理多少个请求之后自动终止，可以有效防止内存溢出，如果为0则不会自动终止，默认为0；由于在具体使用中， 可能引用不明来路的第三方库，或者自己写的不够严谨，不可避免的造成内存泄漏，使用过一段时间后单个php-fpm进程内存占用轻松达到20-30M；设置pm.max_requests就是希望在php-fpm进程处理若干次请求后退出，以便释放内存，避免内存泄漏造成的麻烦；[踩坑经历](https://yuyu888.github.io/posts/2020/12/23/%E8%AE%B0%E4%B8%80%E6%AC%A1php%E8%B0%83%E4%BC%98.html)

如果你的内存足够，建议使用这个配置， 虽然会有一些闲置进程，但是有内存不用也是浪费；pm.max_requests这个配置，如果对自己的程序比较有信心可以适当调大


## pm=dynamic
该配置一般为默认配置，运行时fork出pm.start_servers个进程，随着负载的情况，动态的调整，最多不超过pm.max_children个进程。同时，保证闲置进程数不少于pm.min_spare_servers数量，否则新的进程会被创建，当然也不是无限制的创建，最多闲置进程不超过pm.max_spare_servers数量，超过则一些启动时间最长闲置进程会被清理。

主要参数说明：

- **pm.max_children**：pm 设置为 static 时表示创建的子进程的数量，pm 设置为 dynamic 时表示最大可创建的子进程的数量。必须设置。
-  **pm.start_servers**：设置启动时创建的子进程数目。仅在 pm 设置为 dynamic 时使用。默认值：min_spare_servers + (max_spare_servers - min_spare_servers) / 2。
- **pm.min_spare_servers**：设置空闲服务进程的最低数目。仅在 pm 设置为 dynamic 时使用。必须设置。
- **pm.max_spare_servers**：设置空闲服务进程的最大数目。仅在 pm 设置为 dynamic 时使用。必须设置。

以下列配置为例：

    pm.max_children = 500

    pm.start.servers = 100

    pm.min_spare_servers = 20

    pm.max_spare_servers = 60

    pm.max_requests=2000


服务启动时就会fork 100个php-fpm进程，当请求不断进来，超过80正在处理时，服务器会不断fork出新的进程，保证有至少20个闲置进程，随时应对新的请求；假如服务来了一个波小高峰，同时处理请求数达到200，然后持续回落稳定在80，那么实际存活的php-fpm进程数应该在140，有60个闲置进程；

这种方式，可以根据实际请求动态的fork一定数量的php-fpm进程，一定程度上会节约内存使用，但是运行一段时间，尤其是经历过一个高峰时，依然会保留pm.max_spare_servers个闲置进程；另外不断的fork新进程和回收进程，一定程度上也会消耗系统资源；fork新的进程也需要一定时间，遇到瞬时高峰对服务的平顺性也有一定影响

## pm=ondemand
服务启动后，按需新增进程，最大不超过 pm.max_children 设置值；每个闲置进程，在持续闲置了pm.process_idle_timeout秒后就会被杀掉，有了这个模式，到了服务器低峰期内存自然会降下来，如果服务器长时间没有请求，就只会有一个php-fpm主进程，当然弊端是，遇到高峰期或者如果pm.process_idle_timeout的值太短的话，无法避免服务器频繁创建进程的问题，因此pm = dynamic和pm = ondemand谁更适合视实际情况而定。


<br/>

----

[_参考文档_]

[https://www.jianshu.com/p/c9a028c834ff?hmsr=toutiao.io](https://www.jianshu.com/p/c9a028c834ff?hmsr=toutiao.io){:target="_blank"}
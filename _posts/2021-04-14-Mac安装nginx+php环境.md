---
layout: mypost
title: Mac安装nginx+php环境
categories: [PHP]
---

## nginx 安装

安装命令  
````
  brew install nginx、
````
安装成功后的默认路径  
````
  项目目录：/usr/local/var/www
  nginx配置目录：/usr/local/etc/nginx
````
默认只有nginx.conf.default， 需要手动copy一下
  sudo cp /usr/local/etc/nginx/nginx.conf.default  /usr/local/etc/nginx/nginx.conf

执行 tail nginx.conf 发现  
````

    #    location / {
    #        root   html;
    #        index  index.html index.htm;
    #    }
    #}
    include servers/*;
}
````
最后有一行： include servers/*;

执行： 
```` 
  mkdir servers
````
这样我们就可以站点的配置文件放到 servers 目录下

服务启动  
````
  brew services start nginx
````
nginx 执行的命令
````
  sudo nginx    #启动nginx服务
  sudo nginx -s reload    #重新载入配置文件
  sudo nginx -s stop    #停止nginx服务
````
当我们启动nginx时访问 http://localhost:8080/ 时就能看到欢迎界面

## php安装

Mac上默认安装了php和php-fpm 但对应的配置文件只有默认的。  
````
  /private/etc/php-fpm.conf.default
  /private/etc/php.ini.default
  /etc/php-fpm.d/www.conf.default
````
需要：  
````
  sudo cp /private/etc/php-fpm.conf.default /private/etc/php-fpm.conf
  sudo cp /private/etc/php.ini.default /private/etc/php.ini
  sudo cp /etc/php-fpm.d/www.conf.default   /etc/php-fpm.d/www.conf
````
配置 php-fpm.conf  
```` 
  error_log = /usr/local/var/log/php-fpm.log
````
##  配置nginx

在 servers 目录下新建 www.conf 文件
````
server {
        listen       8182;
        server_name  localhost;
	      root /usr/local/var/www;
        #charset koi8-r;

        #access_log  logs/host.access.log  main;

        location / {
            root   html;
            index  index.html index.htm;
        }

        #error_page  404              /404.html;

        # redirect server error pages to the static page /50x.html
        #
        error_page   500 502 503 504  /50x.html;
        location = /50x.html {
            root   html;
        }

        # proxy the PHP scripts to Apache listening on 127.0.0.1:80
        #
        #location ~ \.php$ {
        #    proxy_pass   http://127.0.0.1;
        #}

        # pass the PHP scripts to FastCGI server listening on 127.0.0.1:9000
        #
        location ~ \.php$ {
            root           html;
            fastcgi_pass   127.0.0.1:9000;
            fastcgi_index  index.php;
            fastcgi_param  SCRIPT_FILENAME  $document_root$fastcgi_script_name;
            include        fastcgi_params;
        }

        # deny access to .htaccess files, if Apache's document root
        # concurs with nginx's one
        #
        #location ~ /\.ht {
        #    deny  all;
        #}
}
````

在 /usr/local/var/www 创建 info.php
、、、、
<?php
  phpinfo();
?>
、、、、

重启nginx 
````  
  sudo nginx -s reload
````
启动php-fpm  
````
  sudo php-fpm
````

访问：http://localhost:8182/info.php 

## php-fpm 重启
````
//查询当前fpm的master进程号
ps aux|grep php-fpm | grep master
//平滑重启fpm，42891是master进程号
kill -USR2 42891
//立即终止fpm
kill -QUIT 42891
//查看状态
ps aux|grep php-fpm
 
 
 
php 5.3.3 以后的php-fpm 不再支持 php-fpm 以前具有的 /usr/local/php/sbin/php-fpm (start|stop|reload)等命令，所以不要再看这种老掉牙的命令了，需要使用信号控制：
 
master进程可以理解以下信号
 
INT, TERM 立刻终止
QUIT 平滑终止
USR1 重新打开日志文件
USR2 平滑重载所有worker进程并重新载入配置和二进制模块
````

## php 降级

mac 自带的php版本比较新，是7+； 与我的项目不兼容， 故需要降级到php5.6

### 安装

>curl -s http://php-osx.liip.ch/install.sh | bash -s 5.6

安装成功后的地址为：   
  /usr/local/php5

修改  .bash_profile 增加一行  
  export PATH=/usr/local/php5/bin:/usr/local/php5/sbin:$PATH;

>source .bash_profile

### 验证

php -v

重启php-fpm 后访问 http://localhost:8182/info.php 
---
layout: mypost
title: OpenResty 安装（ubuntu）
categories: [OpenResty]
---

### 安装
openresty官网  http://openresty.org/cn/linux-packages.html

本例主要讲述ubuntu平台下的安装

在Ubuntu 系统中添加APT 仓库，这样就可以便于未来安装或更新我们的软件包（通过 apt-get update 命令）。 运行下面的命令就可以添加仓库（每个系统只需要运行一次）：

````
# 安装导入 GPG 公钥时所需的几个依赖包（整个安装过程完成后可以随时删除它们）：
sudo apt-get -y install --no-install-recommends wget gnupg ca-certificates

# 导入GPG 密钥：
wget -O - https://openresty.org/package/pubkey.gpg | sudo apt-key add -

# 添加官方 APT 仓库：
# 下面的命令lsb_release找不到的情况下，执行 apt-get install lsb-core -y 进行安装
# echo "deb http://openresty.org/package/ubuntu $(lsb_release -sc) main" \
#    | sudo tee /etc/apt/sources.list.d/openresty.list

# 实际安装的时候，我绕了下  $(lsb_release -sc)  的是 focal
echo "deb http://openresty.org/package/ubuntu focal main"     |  tee /etc/apt/sources.list.d/openresty.list

# 更新 APT 索引：
sudo apt-get update

# 安装
apt-get -y install openresty

````

### 配置

nginx 执行文件地址： /usr/local/openresty/nginx/sbin/

nginx conf 地址: /usr/local/openresty/nginx/conf

lualib 地址：/usr/local/openresty/lualib

配置环境变量
````
vi ~/.bashrc
export PATH=$PATH:/usr/local/openresty/nginx/sbin
source ~/.bashrc
````

nginx 命令：

- nginx -s quit 
 
- nginx -s stop

- nginx -s reload


### 测试

````
vim /usr/local/openresty/nginx/conf/nginx.conf
````
在http模块下配置
````
#lua模块路径，多个之间”;”分隔，其中”;;”表示默认搜索路径，默认到/usr/local/openresty/lualib下找  
lua_package_path "/usr/local/openresty/lualib/?.lua;;";  #lua 模块  
lua_package_cpath "//usr/local/openresty/lualib/?.so;;";  #c模块   
````
实际部署的时候最好把lualib 放到项目文件中，防止有的服务器忘记复制依赖而造成缺少依赖的情况。

在server模块下配置
````
location /lua {
    content_by_lua '
        ngx.say("Hello, Lua!")
    ';
}
````
````
nginx -s reload
curl http://127.0.0.1/lua
>> Hello, Lua!
````

### 以lua文件的方式运行

````
vim /usr/local/openresty/nginx/conf/nginx.conf
````
在http模块下配置
````
include vhost/*.conf; 
````
新增lua_example.conf
````
cd vhost
vim lua_example.conf
````
````
server {  
    listen       80;  
    server_name  _;  
  
    location /lua {  
        default_type 'text/html';  
        lua_code_cache off;  
        content_by_lua_file /usr/work/lua/test.lua;  
    }  
} 
````
lua_code_cache 

默认情况下lua_code_cache  是开启的，即缓存lua代码，即每次lua代码变更必须reload nginx才生效，如果在开发阶段可以通过lua_code_cache  off;关闭缓存，这样调试时每次修改lua代码不需要reload nginx；但是正式环境一定记得开启缓存。 

test.lua
````
ngx.say("Hello, Lua!")
````

### 学习资料

[https://moonbingbing.gitbooks.io/openresty-best-practices/content/](https://moonbingbing.gitbooks.io/openresty-best-practices/content/)

[lua ngxAPI](https://openresty-reference.readthedocs.io/en/latest/Lua_Nginx_API/#ngxreqget_method)

[http://www.daileinote.com/computer/openresty/](http://www.daileinote.com/computer/openresty/)

[lor 框架](http://lor.sumory.com/docs/getting-started-cn)

[nana 框架](https://github.com/horan-geeker/nana)
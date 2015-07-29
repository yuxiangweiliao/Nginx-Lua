# 安装 Nginx+Lua 开发环境  
  
首先我们选择使用 [OpenResty](http://openresty.org/)，其是由 Nginx 核心加很多第三方模块组成，其最大的亮点是默认集成了 Lua 开发环境，使得 Nginx 可以作为一个 Web Server 使用。借助于 Nginx 的事件驱动模型和非阻塞 IO，可以实现高性能的 Web 应用程序。而且 OpenResty 提供了大量组件如 Mysql、Redis、Memcached 等等，使在 Nginx 上开发Web 应用更方便更简单。目前在京东如实时价格、秒杀、动态服务、单品页、列表页等都在使用Nginx+Lua 架构，其他公司如淘宝、去哪儿网等。  

## 安装环境

安装步骤可以参考 [http://openresty.org/#Installation](http://openresty.org/#Installation)。
 
### 创建目录 /usr/servers，以后我们把所有软件安装在此目录  

Java **代码**  收藏代码   
 
```
mkdir -p /usr/servers  
cd /usr/servers/  
```  

### 安装依赖（我的环境是 ubuntu，可以使用如下命令安装，其他的可以参考 openresty 安装步骤）  

Java **代码**  收藏代码  
```
apt-get install libreadline-dev libncurses5-dev libpcre3-dev libssl-dev perl  
```  

### 下载 ngx_openresty-1.7.7.2.tar.gz 并解压  

Java **代码**  收藏代码  
```
wget http://openresty.org/download/ngx_openresty-1.7.7.2.tar.gz  
tar -xzvf ngx_openresty-1.7.7.2.tar.gz  
```  

ngx_openresty-1.7.7.2/bundle 目录里存放着 nginx 核心和很多第三方模块，比如有我们需要的 Lua 和 LuaJIT。
 
### 安装 LuaJIT  

Java **代码**  收藏代码  
  
```
cd bundle/LuaJIT-2.1-20150120/  
make clean && make && make install  
ln -sf luajit-2.1.0-alpha /usr/local/bin/luajit  
```  

### 下载 ngx_cache_purge 模块，该模块用于清理 nginx 缓存  

Java **代码**  收藏代码  
  
```
cd /usr/servers/ngx_openresty-1.7.7.2/bundle  
wget https://github.com/FRiCKLE/ngx_cache_purge/archive/2.3.tar.gz  
tar -xvf 2.3.tar.gz  
```  

### 下载 nginx_upstream_check_module 模块，该模块用于 ustream 健康检查  

Java **代码**  收藏代码  
  
```
cd /usr/servers/ngx_openresty-1.7.7.2/bundle  
wget https://github.com/yaoweibin/nginx_upstream_check_module/archive/v0.3.0.tar.gz  
tar -xvf v0.3.0.tar.gz   
```  
  
### 安装 ngx_openresty    

Java **代码**  收藏代码  
  
```
cd /usr/servers/ngx_openresty-1.7.7.2  
./configure --prefix=/usr/servers --with-http_realip_module  --with-pcre  --with-luajit --add-module=./bundle/ngx_cache_purge-2.3/ --add-module=./bundle/nginx_upstream_check_module-0.3.0/ -j2  
make && make install  
```   

--with***                安装一些内置/集成的模块  
--with-http_realip_module  取用户真实 ip 模块  
-with-pcre               Perl 兼容的达式模块  
--with-luajit              集成 luajit 模块  
--add-module            添加自定义的第三方模块，如此次的 ngx_che_purge
 
### 到 /usr/servers 目录下   

Java **代码**  收藏代码  
  
```
cd /usr/servers/    
ll   
```  
 
会发现多出来了如下目录，说明安装成功  
**/usr/servers/luajit** ：luajit 环境，luajit 类似于 java 的 jit，即即时编译，lua 是一种解释语言，通过 luajit 可以即时编译 lua 代码到机器代码，得到很好的性能；  
**/usr/servers/lualib**：要使用的 lua 库，里边提供了一些默认的 lua 库，如 redis，json 库等，也可以把一些自己开发的或第三方的放在这；  
/usr/servers/nginx ：安装的 nginx；
 
通过 /usr/servers/nginx/sbin/nginx  -V 查看 nginx 版本和安装的模块
 
### 启动 nginx  

/usr/servers/nginx/sbin/nginx
 
接下来该配置 nginx+lua 开发环境了
 
## 配置环境

配置及 Nginx HttpLuaModule 文档在可以查看 [http://openresty.org/#Installation](http://openresty.org/#Installation)。
 
 
### 编辑 nginx.conf 配置文件   
  
Java **代码**  收藏代码  
  
```
vim /usr/servers/nginx/conf/nginx.conf  
```  
  
### 在 http 部分添加如下配置  
 
Java **代码**  收藏代码  
  
```
\#lua模块路径，多个之间”;”分隔，其中”;;”表示默认搜索路径，默认到/usr/servers/nginx下找  
lua_package_path "/usr/servers/lualib/?.lua;;";  #lua 模块  
lua_package_cpath "/usr/servers/lualib/?.so;;";  #c模块   
```  

### 为了方便开发我们在 /usr/servers/nginx/conf 目录下创建一个 lua.conf   

Java **代码**  收藏代码  
  
```
\#lua.conf  
server {  
    listen       80;  
    server_name  _;  
}  
```  
 
### 在 nginx.conf 中的 http 部分添加 include lua.conf 包含此文件片段  
 
Java **代码**  收藏代码  
  
```
include lua.conf;  
```  
  
### 测试是否正常  
 
Java **代码**  收藏代码  

/usr/servers/nginx/sbin/nginx  -t   
 
如果显示如下内容说明配置成功  

nginx: the configuration file /usr/servers/nginx/conf/nginx.conf syntax is ok
nginx: configuration file /usr/servers/nginx/conf/nginx.conf test is successful
 
## HelloWorld

### 在 lua.conf 中 server 部分添加如下配置  
   
Java **代码**  收藏代码  
  
```
location /lua {  
    default_type 'text/html';  
        content_by_lua 'ngx.say("hello world")';  
}  
```  
         
### 测试配置是否正确   

Java **代码**  收藏代码  

/usr/servers/nginx/sbin/nginx  -t  
   
### 重启 nginx   

Java **代码**  收藏代码  

/usr/servers/nginx/sbin/nginx  -s reload  
 
### 访问如 [http://192.168.1.6/lua](http://192.168.1.6/lua)（自己的机器根据实际情况换 ip），可以看到如下内容 
hello world
 
### lua 代码文件  

我们把 lua 代码放在 nginx 配置中会随着 lua 的代码的增加导致配置文件太长不好维护，因此我们应该把 lua 代码移到外部文件中存储。   

Java **代码**  收藏代码  
  
```
vim /usr/servers/nginx/conf/lua/test.lua    
```  

Java **代码**  收藏代码  
  
```
\#添加如下内容  
ngx.say("hello world");   
```  

然后 lua.conf 修改为  
   
Java **代码**  收藏代码  
  
```
location /lua {  
    default_type 'text/html';  
    content_by_lua_file conf/lua/test.lua; #相对于nginx安装目录  
}    
```  
 
此处 conf/lua/test.lua 也可以使用绝对路径 /usr/servers/nginx/conf/lua/test.lua。
 
### lua_code_cache   

默认情况下 lua_code_cache  是开启的，即缓存 lua 代码，即每次 lua 代码变更必须reload nginx 才生效，如果在开发阶段可以通过 lua_code_cache  off;关闭缓存，这样调试时每次修改 lua 代码不需要 reload nginx；但是正式环境一定记得开启缓存。  
 
Java **代码**  收藏代码  
  
```
    location /lua {  
        default_type 'text/html';  
        lua_code_cache off;  
        content_by_lua_file conf/lua/test.lua;  
}  
```  
 
开启后 reload nginx 会看到如下报警  

nginx: [alert] lua_code_cache is off; this will hurt performance in /usr/servers/nginx/conf/lua.conf:8
 
### 错误日志
 
如果运行过程中出现错误，请不要忘记查看错误日志。  
 
Java **代码**  收藏代码  
  
```
tail -f /usr/servers/nginx/logs/error.log  
```   

到此我们的基本环境搭建完毕。
 
## nginx+lua 项目构建

以后我们的 nginx lua 开发文件会越来越多，我们应该把其项目化，已方便开发。项目目录结构如下所示：  

example  

    example.conf     ---该项目的nginx 配置文件
    lua              ---我们自己的lua代码
      test.lua
    lualib            ---lua依赖库/第三方依赖
      *.lua
      *.so
 
其中我们把 lualib 也放到项目中的好处就是以后部署的时候可以一起部署，防止有的服务器忘记复制依赖而造成缺少依赖的情况。
 
我们将项目放到到 /usr/example 目录下。
 
/usr/servers/nginx/conf/nginx.conf 配置文件如下(此处我们最小化了配置文件)  

Java **代码**  收藏代码  
  
```
\#user  nobody;  
worker_processes  2;  
error_log  logs/error.log;  
events {  
    worker_connections  1024;  
}  
http {  
    include       mime.types;  
    default_type  text/html;  
  
    #lua模块路径，其中”;;”表示默认搜索路径，默认到/usr/servers/nginx下找  
    lua_package_path "/usr/example/lualib/?.lua;;";  #lua 模块  
    lua_package_cpath "/usr/example/lualib/?.so;;";  #c模块  
    include /usr/example/example.conf;  
}    
```  

通过绝对路径包含我们的 lua 依赖库和 nginx 项目配置文件。
 
/usr/example/example.conf 配置文件如下   

Java **代码**  收藏代码  
  
```
server {  
    listen       80;  
    server_name  _;  
  
    location /lua {  
        default_type 'text/html';  
        lua_code_cache off;  
        content_by_lua_file /usr/example/lua/test.lua;  
    }  
}   
```   
 
lua 文件我们使用绝对路径 /usr/example/lua/test.lua。 
 
到此我们就可以把 example 扔 svn 上了。
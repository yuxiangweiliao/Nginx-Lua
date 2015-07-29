# 流量复制 /AB 测试/协程  
  
## 流量复制

在实际开发中经常涉及到项目的升级，而该升级不能简单的上线就完事了，需要验证该升级是否兼容老的上线，因此可能需要并行运行两个项目一段时间进行数据比对和校验，待没问题后再进行上线。这其实就需要进行流量复制，把流量复制到其他服务器上，一种方式是使用如 tcpcopy 引流；另外我们还可以使用 nginx 的 HttpLuaModule 模块中的ngx.location.capture\_multi 进行并发执行来模拟复制。
 
构造两个服务  

Java **代码**  
  
```
location /test1 {  
    keepalive_timeout 60s;   
    keepalive_requests 1000;  
    content_by_lua '  
        ngx.print("test1 : ", ngx.req.get_uri_args()["a"])  
        ngx.log(ngx.ERR, "request test1")  
    ';  
}  
location /test2 {  
    keepalive_timeout 60s;   
    keepalive_requests 1000;  
    content_by_lua '  
        ngx.print("test2 : ", ngx.req.get_uri_args()["a"])  
        ngx.log(ngx.ERR, "request test2")  
    ';  
}  
```  
  
通过 ngx.location.capture\_multi 调用  

Java **代码**  
  
```
location /test {  
     lua_socket_connect_timeout 3s;  
     lua_socket_send_timeout 3s;  
     lua_socket_read_timeout 3s;  
     lua_socket_pool_size 100;  
     lua_socket_keepalive_timeout 60s;  
     lua_socket_buffer_size 8k;  
  
     content_by_lua '  
         local res1, res2 = ngx.location.capture_multi{  
               { "/test1", { args = ngx.req.get_uri_args() } },  
               { "/test2", { args = ngx.req.get_uri_args()} },  
         }  
         if res1.status == ngx.HTTP_OK then  
             ngx.print(res1.body)  
         end  
         if res2.status ~= ngx.HTTP_OK then  
            --记录错误  
         end  
     ';  
}      
```  

此处可以根据需求设置相应的超时时间和长连接连接池等；ngx.location.capture 底层通过cosocket 实现，而其支持 Lua 中的协程，通过它可以以同步的方式写非阻塞的代码实现。
 
此处要考虑记录失败的情况，对失败的数据进行重放还是放弃根据自己业务做处理。
 
## AB 测试

AB 测试即多版本测试，有时候我们开发了新版本需要灰度测试，即让一部分人看到新版，一部分人看到老版，然后通过访问数据决定是否切换到新版。比如可以通过根据区域、用户等信息进行切版本。
 
比如京东商城有一个 cookie 叫做\_\_jda，该 cookie 是在用户访问网站时种下的，因此我们可以拿到这个 cookie，根据这个 cookie 进行版本选择。
 
比如两次清空 cookie 访问发现第二个数字串是变化的，即我们可以根据第二个数字串进行判断。  
\_\_jda=122270672.1059377902.1425691107.1425691107.1425699059.1
\_\_jda=122270672.556927616.1425699216.1425699216.1425699216.1。
 
判断规则可以比较多的选择，比如通过尾号；要切 30% 的流量到新版，可以通过选择尾号为 1，3,5 的切到新版，其余的还停留在老版。
 
### 使用 map 选择版本  
 
Java **代码**  
  
```
map $cookie___jda $ab_key {  
    default                                       "0";  
    ~^\d+\.\d+(?P<k>(1|3|5))\.                    "1";  
}  
```  
  
使用 map 映射规则，即如果是到新版则等于 "1"，到老版等于 “0”； 然后我们就可以通过ngx.var.ab\_key 获取到该数据。  

Java **代码**  
  
```
location /abtest1 {  
    if ($ab_key = "1") {  
        echo_location /test1 ngx.var.args;  
    }  
    if ($ab_key = "0") {  
        echo_location /test2 ngx.var.args;  
    }  
}    
```  

此处也可以使用 proxy\_pass 到不同版本的服务器上  
 
Java **代码**  
  
```
location /abtest2 {  
    if ($ab_key = "1") {  
        rewrite ^ /test1 break;  
        proxy_pass http://backend1;  
    }  
    rewrite ^ /test2 break;  
    proxy_pass http://backend2;  
}  
```  
 
### 直接在 Lua 中使用 lua-resty-cookie 获取该 Cookie 进行解析  

首先下载 lua-resty-cookie  

Java **代码**  
  
```
cd /usr/example/lualib/resty/  
wget https://raw.githubusercontent.com/cloudflare/lua-resty-cookie/master/lib/resty/cookie.lua  
```  
 
Java **代码**  
  
```
location /abtest3 {  
    content_by_lua '  
  
         local ck = require("resty.cookie")  
         local cookie = ck:new()  
         local ab_key = "0"  
         local jda = cookie:get("__jda")  
         if jda then  
             local v = ngx.re.match(jda, [[^\d+\.\d+(1|3|5)\.]])  
             if v then  
                ab_key = "1"  
             end  
         end  
  
         if ab_key == "1" then  
             ngx.exec("/test1", ngx.var.args)  
         else  
             ngx.print(ngx.location.capture("/test2", {args = ngx.req.get_uri_args()}).body)  
         end  
    ';  
  
}    
```  

 首先使用 [lua-resty-cookie](https://github.com/cloudflare/lua-resty-cookie) 获取 cookie，然后使用 ngx.re.match 进行规则的匹配，最后使用 ngx.exec 或者 ngx.location.capture 进行处理。此处同时使用 ngx.exec 和ngx.location.capture 目的是为了演示，此外没有对 ngx.location.capture 进行异常处理。
 
## 协程

Lua 中没有线程和异步编程编程的概念，对于并发执行提供了协程的概念，个人认为协程是在A运行中发现自己忙则把 CPU 使用权让出来给B使用，最后 A 能从中断位置继续执行，本地还是单线程，CPU 独占的；因此如果写网络程序需要配合非阻塞 I/O 来实现。
 
ngx\_lua 模块对协程做了封装，我们可以直接调用 ngx.thread API 使用，虽然称其为“轻量级线程”，但其本质还是 Lua 协程。 该 API 必须配合该 ngx\_lua 模块提供的非阻塞 I/O API 一起使用，比如我们之前使用的 ngx.location.capture\_multi 和 lua-resty-redis、lua-resty-mysql 等基于 cosocket 实现的都是支持的。
 
通过 Lua 协程我们可以并发的调用多个接口，然后谁先执行成功谁先返回，类似于 BigPipe 模型。
 
### 依赖的 API  
Java **代码**  
  
```
location /api1 {  
    echo_sleep 3;  
    echo api1 : $arg_a;  
}  
location /api2 {  
    echo_sleep 3;  
    echo api2 : $arg_a;  
}   
```  
 
 我们使用 echo\_sleep 等待3秒。
 
### 串行实现  

Java **代码**  
  
```
location /serial {  
    content_by_lua '  
        local t1 = ngx.now()  
        local res1 = ngx.location.capture("/api1", {args = ngx.req.get_uri_args()})  
        local res2 = ngx.location.capture("/api2", {args = ngx.req.get_uri_args()})  
        local t2 = ngx.now()  
        ngx.print(res1.body, "<br/>", res2.body, "<br/>", tostring(t2-t1))  
    ';  
}  
```  
  
即一个个的调用，总的执行时间在6秒以上，比如访问 http://192.168.1.2/serial?a=22  

Java **代码**  
  
```
api1 : 22   
api2 : 22   
6.0040001869202  
```  
 
### ngx.location.capture\_multi 实现  

Java **代码**  
  
```
location /concurrency1 {  
    content_by_lua '  
        local t1 = ngx.now()  
        local res1,res2 = ngx.location.capture_multi({  
              {"/api1", {args = ngx.req.get_uri_args()}},  
              {"/api2", {args = ngx.req.get_uri_args()}}  
  
        })  
        local t2 = ngx.now()  
        ngx.print(res1.body, "<br/>", res2.body, "<br/>", tostring(t2-t1))  
    ';  
}    
```  

直接使用 ngx.location.capture\_multi 来实现，比如访问 http://192.168.1.2/concurrency1?a=22  

Java **代码**  
  
```
api1 : 22   
api2 : 22   
3.0020000934601  
```  
    
### 协程 API 实现  
 
Java **代码**  
  
```
location /concurrency2 {  
    content_by_lua '  
        local t1 = ngx.now()  
        local function capture(uri, args)  
           return ngx.location.capture(uri, args)  
        end  
        local thread1 = ngx.thread.spawn(capture, "/api1", {args = ngx.req.get_uri_args()})  
        local thread2 = ngx.thread.spawn(capture, "/api2", {args = ngx.req.get_uri_args()})  
        local ok1, res1 = ngx.thread.wait(thread1)  
        local ok2, res2 = ngx.thread.wait(thread2)  
        local t2 = ngx.now()  
        ngx.print(res1.body, "<br/>", res2.body, "<br/>", tostring(t2-t1))  
    ';  
}    
```  

使用 ngx.thread.spawn 创建一个轻量级线程，然后使用 ngx.thread.wait 等待该线程的执行成功。比如访问 http://192.168.1.2/concurrency2?a=22  

Java **代码**  
  
```
api1 : 22   
api2 : 22   
3.0030000209808  
```  
  
其有点类似于 Java 中的线程池执行模型，但不同于线程池，其每次只执行一个函数，遇到 IO 等待则让出 CPU 让下一个执行。我们可以通过下面的方式实现任意一个成功即返回，之前的是等待所有执行成功才返回。   
 
Java **代码**  
  
```
local  ok, res = ngx.thread.wait(thread1, thread2)    
```
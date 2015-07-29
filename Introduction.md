# Nginx+Lua 开发入门  
  
## Nginx 入门  

本文目的是学习 Nginx+Lua 开发，对于 Nginx 基本知识可以参考如下文章：  

nginx 启动、关闭、重启  

[http://www.cnblogs.com/derekchen/archive/2011/02/17/1957209.html](http://www.cnblogs.com/derekchen/archive/2011/02/17/1957209.html)  

agentzh 的 Nginx 教程  

[http://openresty.org/download/agentzh-nginx-tutorials-zhcn.html](http://openresty.org/download/agentzh-nginx-tutorials-zhcn.html)  

Nginx+Lua 入门 

[http://17173ops.com/2013/11/01/17173-ngx-lua-manual.shtml](http://17173ops.com/2013/11/01/17173-ngx-lua-manual.shtml)  

nginx 配置指令的执行顺序  

[http://zhongfox.github.io/blog/server/2013/05/15/nginx-exec-order/](http://zhongfox.github.io/blog/server/2013/05/15/nginx-exec-order/)  

nginx 与 lua 的执行顺序和步骤说明  

[http://www.mrhaoting.com/?p=157](http://www.mrhaoting.com/?p=157)  

Nginx 配置文件 nginx.conf 中文详解  

[http://www.ha97.com/5194.html](http://www.ha97.com/5194.html)  

Tengine 的 Nginx 开发从入门到精通  

[http://tengine.taobao.org/book/](http://tengine.taobao.org/book/)  

官方文档  

[http://wiki.nginx.org/Configuration](http://wiki.nginx.org/Configuration)
 
# Lua 入门  

本文目的是学习 Nginx+Lua 开发，对于 Lua 基本知识可以参考如下文章：  

Lua 简明教程  

[http://coolshell.cn/articles/10739.html](http://coolshell.cn/articles/10739.html)  

lua 在线 lua 学习教程  

[http://book.luaer.cn/](http://book.luaer.cn/)  

Lua 5.1 参考手册  

[http://www.codingnow.com/2000/download/lua_manual.html](http://www.codingnow.com/2000/download/lua_manual.html)  

Lua 5.3 参考手册  

[http://cloudwu.github.io/lua53doc/](http://cloudwu.github.io/lua53doc/)  

## Nginx Lua API  

和一般的 Web Server 类似，我们需要接收请求、处理并输出响应。而对于请求我们需要获取如请求参数、请求头、Body 体等信息；而对于处理就是调用相应的 Lua 代码即可；输出响应需要进行响应状态码、响应头和响应内容体的输出。因此我们从如上几个点出发即可。
 
### 接收请求  

#### example.conf 配置文件   

Java 代码  收藏代码  
  
```
location ~ /lua_request/(\d+)/(\d+) {  
    #设置nginx变量  
    set $a $1;   
    set $b $host;  
    default_type "text/html";  
    #nginx内容处理  
    content_by_lua_file /usr/example/lua/test_request.lua;  
    #内容体处理完成后调用  
    echo_after_body "ngx.var.b $b";  
}    
```  

#### test_request.lua   

Java 代码  收藏代码  
  
```
--nginx变量  
local var = ngx.var  
ngx.say("ngx.var.a : ", var.a, "<br/>")  
ngx.say("ngx.var.b : ", var.b, "<br/>")  
ngx.say("ngx.var[2] : ", var[2], "<br/>")  
ngx.var.b = 2;  
  
ngx.say("<br/>")  
  
--请求头  
local headers = ngx.req.get_headers()  
ngx.say("headers begin", "<br/>")  
ngx.say("Host : ", headers["Host"], "<br/>")  
ngx.say("user-agent : ", headers["user-agent"], "<br/>")  
ngx.say("user-agent : ", headers.user_agent, "<br/>")  
for k,v in pairs(headers) do  
    if type(v) == "table" then  
        ngx.say(k, " : ", table.concat(v, ","), "<br/>")  
    else  
        ngx.say(k, " : ", v, "<br/>")  
    end  
end  
ngx.say("headers end", "<br/>")  
ngx.say("<br/>")  
  
--get请求uri参数  
ngx.say("uri args begin", "<br/>")  
local uri_args = ngx.req.get_uri_args()  
for k, v in pairs(uri_args) do  
    if type(v) == "table" then  
        ngx.say(k, " : ", table.concat(v, ", "), "<br/>")  
    else  
        ngx.say(k, ": ", v, "<br/>")  
    end  
end  
ngx.say("uri args end", "<br/>")  
ngx.say("<br/>")  
  
--post请求参数  
ngx.req.read_body()  
ngx.say("post args begin", "<br/>")  
local post_args = ngx.req.get_post_args()  
for k, v in pairs(post_args) do  
    if type(v) == "table" then  
        ngx.say(k, " : ", table.concat(v, ", "), "<br/>")  
    else  
        ngx.say(k, ": ", v, "<br/>")  
    end  
end  
ngx.say("post args end", "<br/>")  
ngx.say("<br/>")  
--请求的http协议版本  
ngx.say("ngx.req.http_version : ", ngx.req.http_version(), "<br/>")  
--请求方法  
ngx.say("ngx.req.get_method : ", ngx.req.get_method(), "<br/>")  
--原始的请求头内容  
ngx.say("ngx.req.raw_header : ",  ngx.req.raw_header(), "<br/>")  
--请求的body内容体  
ngx.say("ngx.req.get_body_data() : ", ngx.req.get_body_data(), "<br/>")  
ngx.say("<br/>")   
```   

**ngx.var** ： nginx 变量，如果要赋值如 ngx.var.b = 2，此变量必须提前声明；另外对于 nginx location 中使用正则捕获的捕获组可以使用 ngx.var [捕获组数字]获取；  

**ngx.req.get_headers**：获取请求头，默认只获取前100，如果想要获取所以可以调用ngx.req.get_headers(0)；获取带中划线的请求头时请使用如 headers.user_agent 这种方式；如果一个请求头有多个值，则返回的是 table；  

**ngx.req.get_uri_args**：获取 url 请求参数，其用法和 get_headers 类似；  

**ngx.req.get_post_args**：获取 post 请求内容体，其用法和 get_headers 类似，但是必须提前调用 ngx.req.read_body() 来读取 body 体（也可以选择在 nginx 配置文件使用lua_need_request_body on;开启读取 body 体，但是官方不推荐）；  

**ngx.req.raw_header**：未解析的请求头字符串；  

**ngx.req.get_body_data**：为解析的请求 body 体内容字符串。
 
如上方法处理一般的请求基本够用了。另外在读取 post 内容体时根据实际情况设置client_body_buffer_size 和 client_max_body_size 来保证内容在内存而不是在文件中。
 
使用如下脚本测试  

Java **代码**  收藏代码  
  
```
wget --post-data 'a=1&b=2' 'http://127.0.0.1/lua_request/1/2?a=3&b=4' -O -   
```  

### 输出响应   

#### example.conf 配置文件  

Java **代码**  收藏代码  
  
```
location /lua_response_1 {  
    default_type "text/html";  
    content_by_lua_file /usr/example/lua/test_response_1.lua;  
}    
```  

#### test_response_1.lua   

Java **代码**  收藏代码  
  
```
--写响应头  
ngx.header.a = "1"  
--多个响应头可以使用table  
ngx.header.b = {"2", "3"}  
--输出响应  
ngx.say("a", "b", "<br/>")  
ngx.print("c", "d", "<br/>")  
--200状态码退出  
return ngx.exit(200)  
ngx.header：输出响应头；
ngx.print：输出响应内容体；
ngx.say：通ngx.print，但是会最后输出一个换行符；
ngx.exit：指定状态码退出。
```  

#### example.conf 配置文件  

Java **代码**  收藏代码  
  
```
location /lua_response_2 {  
    default_type "text/html";  
    content_by_lua_file /usr/example/lua/test_response_2.lua;  
}  
```  

#### test_response_2.lua  

Java **代码**  收藏代码  
  
```
ngx.redirect("http://jd.com", 302)    
```  

**ngx.redirect**：重定向； 
 
ngx.status= 状态码，设置响应的状态码；ngx.resp.get_headers() 获取设置的响应状态码；ngx.send_headers() 发送响应状态码，当调用 ngx.say/ngx.print 时自动发送响应状态码；可以通过 ngx.headers_sent=true 判断是否发送了响应状态码。
 
### 其他 API  

#### example.conf 配置文件  

Java **代码**  收藏代码  
  
```
location /lua_other {  
    default_type "text/html";  
    content_by_lua_file /usr/example/lua/test_other.lua;  
}  
```  
 
#### test_other.lua  

Java **代码**  收藏代码   
  
```
--未经解码的请求uri  
local request_uri = ngx.var.request_uri;  
ngx.say("request_uri : ", request_uri, "<br/>");  
--解码  
ngx.say("decode request_uri : ", ngx.unescape_uri(request_uri), "<br/>");  
--MD5  
ngx.say("ngx.md5 : ", ngx.md5("123"), "<br/>")  
--http time  
ngx.say("ngx.http_time : ", ngx.http_time(ngx.time()), "<br/>")  
```   
 
ngx.escape_uri/ngx.unescape_uri ： uri 编码解码；
ngx.encode_args/ngx.decode_args：参数编码解码；
ngx.encode_base64/ngx.decode_base64：BASE64 编码解码；
ngx.re.match：nginx 正则表达式匹配；
 
更多 Nginx Lua API 请参考 [http://wiki.nginx.org/HttpLuaModule#Nginx_API_for_Lua](http://wiki.nginx.org/HttpLuaModule#Nginx_API_for_Lua)。
 
### Nginx 全局内存  

使用过如 Java 的朋友可能知道如 Ehcache 等这种进程内本地缓存，Nginx 是一个 Master 进程多个 Worker 进程的工作方式，因此我们可能需要在多个 Worker 进程中共享数据，那么此时就可以使用 ngx.shared.DICT 来实现全局内存共享。
 
#### 首先在 nginx.conf 的 http 部分分配内存大小  

Java **代码**  收藏代码  
  
```
\#共享全局变量，在所有worker间共享  
lua_shared_dict shared_data 1m;  
```   
 
#### example.conf 配置文件  

Java **代码**  收藏代码  
  
```
location /lua_shared_dict {  
    default_type "text/html";  
    content_by_lua_file /usr/example/lua/test_lua_shared_dict.lua;  
}    
```  

#### test_lua_shared_dict.lua  

Java **代码**  收藏代码  
  
```
--1、获取全局共享内存变量  
local shared_data = ngx.shared.shared_data  
  
--2、获取字典值  
local i = shared_data:get("i")  
if not i then  
    i = 1  
    --3、惰性赋值  
    shared_data:set("i", i)  
    ngx.say("lazy set i ", i, "<br/>")  
end  
--递增  
i = shared_data:incr("i", 1)  
ngx.say("i=", i, "<br/>")   
```  
 
更多 API 请参考 [http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT](http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT)。 
 
 
到此基本的 Nginx Lua API 就学完了，对于请求处理和输出响应如上介绍的 API 完全够用了，更多 API 请参考官方文档。
 
## Nginx Lua 模块指令  

Nginx 共11个处理阶段，而相应的处理阶段是可以做插入式处理，即可插拔式架构；另外指令可以在 http、server、server if、location、location if 几个范围进行配置：  

<table style="border-collapse: collapse; border: none;" cellpadding="0" border="1" cellspacing="0" class="MsoTableGrid">
<tr>
<td style="width: 118.8pt; padding: 0cm 5.4pt 0cm 5.4pt;" width="158">
<p class="MsoNormal">指令</p>
</td>
<td style="width: 78.0pt; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="104">
<p class="MsoNormal">所处处理阶段</p>
</td>
<td style="width: 70.85pt; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="94">
<p class="MsoNormal">使用范围</p>
</td>
<td style="width: 158.45pt; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="211">
<p class="MsoNormal">解释</p>
</td>
</tr>
<tr>
<td style="width: 118.8pt; border-top: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="158">
<p class="MsoNormal">init_by_lua</p>
<p class="MsoNormal">init_by_lua_file</p>
</td>
<td style="width: 78.0pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="104">
<p class="MsoNormal">loading-config</p>
</td>
<td style="width: 70.85pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="94">
<p class="MsoNormal">http</p>
</td>
<td style="width: 158.45pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="211">
<p class="MsoNormal">nginx Master进程加载配置时执行；</p>
<p class="MsoNormal">通常用于初始化全局配置/预加载Lua模块</p>
</td>
</tr>
<tr>
<td style="width: 118.8pt; border-top: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="158">
<p class="MsoNormal">init_worker_by_lua</p>
<p class="MsoNormal">init_worker_by_lua_file</p>
</td>
<td style="width: 78.0pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="104">
<p class="MsoNormal">starting-worker</p>
</td>
<td style="width: 70.85pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="94">
<p class="MsoNormal">http</p>
</td>
<td style="width: 158.45pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="211">
<p class="MsoNormal">每个Nginx Worker进程启动时调用的计时器，如果Master进程不允许则只会在init_by_lua之后调用；</p>
<p class="MsoNormal">通常用于定时拉取配置/数据，或者后端服务的健康检查</p>
</td>
</tr>
<tr>
<td style="width: 118.8pt; border-top: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="158">
<p class="MsoNormal">set_by_lua</p>
<p class="MsoNormal">set_by_lua_file</p>
</td>
<td style="width: 78.0pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="104">
<p class="MsoNormal">rewrite</p>
</td>
<td style="width: 70.85pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="94">
<p class="MsoNormal">server,server if,location,location if</p>
</td>
<td style="width: 158.45pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="211">
<p class="MsoNormal">设置nginx变量，可以实现复杂的赋值逻辑；此处是阻塞的，Lua代码要做到非常快；</p>
</td>
</tr>
<tr>
<td style="width: 118.8pt; border-top: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="158">
<p class="MsoNormal">rewrite_by_lua</p>
<p class="MsoNormal">rewrite_by_lua_file</p>
</td>
<td style="width: 78.0pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="104">
<p class="MsoNormal">rewrite tail</p>
</td>
<td style="width: 70.85pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="94">
<p class="MsoNormal">http,server,location,location if</p>
</td>
<td style="width: 158.45pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="211">
<p class="MsoNormal">rrewrite阶段处理，可以实现复杂的转发/重定向逻辑；</p>
</td>
</tr>
<tr>
<td style="width: 118.8pt; border-top: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="158">
<p class="MsoNormal">access_by_lua</p>
<p class="MsoNormal">access_by_lua_file</p>
</td>
<td style="width: 78.0pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="104">
<p class="MsoNormal">access tail</p>
</td>
<td style="width: 70.85pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="94">
<p class="MsoNormal">http,server,location,location if</p>
</td>
<td style="width: 158.45pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="211">
<p class="MsoNormal">请求访问阶段处理，用于访问控制</p>
</td>
</tr>
<tr>
<td style="width: 118.8pt; border-top: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="158">
<p class="MsoNormal">content_by_lua</p>
<p class="MsoNormal">content_by_lua_file</p>
</td>
<td style="width: 78.0pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="104">
<p class="MsoNormal">content</p>
</td>
<td style="width: 70.85pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="94">
<p class="MsoNormal">location，location if</p>
</td>
<td style="width: 158.45pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="211">
<p class="MsoNormal">内容处理器，接收请求处理并输出响应</p>
</td>
</tr>
<tr>
<td style="width: 118.8pt; border-top: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="158">
<p class="MsoNormal">header_filter_by_lua</p>
<p class="MsoNormal">header_filter_by_lua_file</p>
</td>
<td style="width: 78.0pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="104">
<p class="MsoNormal">output-header-filter</p>
</td>
<td style="width: 70.85pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="94">
<p class="MsoNormal">http，server，location，location if</p>
</td>
<td style="width: 158.45pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="211">
<p class="MsoNormal">设置header和cookie</p>
</td>
</tr>
<tr>
<td style="width: 118.8pt; border-top: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="158">
<p class="MsoNormal">body_filter_by_lua</p>
<p class="MsoNormal">body_filter_by_lua_file</p>
</td>
<td style="width: 78.0pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="104">
<p class="MsoNormal">output-body-filter</p>
</td>
<td style="width: 70.85pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="94">
<p class="MsoNormal">http，server，location，location if</p>
</td>
<td style="width: 158.45pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="211">
<p class="MsoNormal">对响应数据进行过滤，比如截断、替换。</p>
</td>
</tr>
<tr>
<td style="width: 118.8pt; border-top: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="158">
<p class="MsoNormal">log_by_lua</p>
<p class="MsoNormal">log_by_lua_file</p>
</td>
<td style="width: 78.0pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="104">
<p class="MsoNormal">log</p>
</td>
<td style="width: 70.85pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="94">
<p class="MsoNormal">http，server，location，location if</p>
</td>
<td style="width: 158.45pt; border-top: none; border-left: none; padding: 0cm 5.4pt 0cm 5.4pt;" width="211">
<p class="MsoNormal">log阶段处理，比如记录访问量/统计平均响应时间</p>
</td>
</tr>
</table>
 
更详细的解释请参考 [http://wiki.nginx.org/HttpLuaModule#Directives](http://wiki.nginx.org/HttpLuaModule#Directives)。如上指令很多并不常用，因此我们只拿其中的一部分做演示。
 
### init_by_lua  

每次 Nginx 重新加载配置时执行，可以用它来完成一些耗时模块的加载，或者初始化一些全局配置；在 Master 进程创建 Worker 进程时，此指令中加载的全局变量会进行 Copy-OnWrite，即会复制到所有全局变量到 Worker 进程。
 
#### nginx.conf 配置文件中的 http 部分添加如下代码  

Java **代码**  收藏代码  
  
```
\#共享全局变量，在所有worker间共享  
lua_shared_dict shared_data 1m;  
  
init_by_lua_file /usr/example/lua/init.lua;  
```  
 
#### init.lua  

Java **代码**  收藏代码  
  
```
--初始化耗时的模块  
local redis = require 'resty.redis'  
local cjson = require 'cjson'  
  
--全局变量，不推荐  
count = 1  
  
--共享全局内存  
local shared_data = ngx.shared.shared_data  
shared_data:set("count", 1)  
```  
 
#### test.lua  

Java **代码**  收藏代码  
  
```
count = count + 1  
ngx.say("global variable : ", count)  
local shared_data = ngx.shared.shared_data  
ngx.say(", shared memory : ", shared_data:get("count"))  
shared_data:incr("count", 1)  
ngx.say("hello world")  
```  
   
#### 访问如 http://192.168.1.2/lua 会发现全局变量一直不变，而共享内存一直递增
global variable : 2 , shared memory : 8 hello world 
 
另外注意一定在生产环境开启 lua_code_cache，否则每个请求都会创建 Lua VM 实例。
 
### init_worker_by_lua  

用于启动一些定时任务，比如心跳检查，定时拉取服务器配置等等；此处的任务是跟 Worker 进程数量有关系的，比如有2个 Worker 进程那么就会启动两个完全一样的定时任务。
 
#### nginx.conf 配置文件中的 http 部分添加如下代码  

Java **代码**  收藏代码  
  
```
init_worker_by_lua_file /usr/example/lua/init_worker.lua;  
```   
  
#### init_worker.lua  

Java **代码**  收藏代码  
  
```
local count = 0  
local delayInSeconds = 3  
local heartbeatCheck = nil  
  
heartbeatCheck = function(args)  
   count = count + 1  
   ngx.log(ngx.ERR, "do check ", count)  
  
   local ok, err = ngx.timer.at(delayInSeconds, heartbeatCheck)  
  
   if not ok then  
      ngx.log(ngx.ERR, "failed to startup heartbeart worker...", err)  
   end  
end  
  
heartbeatCheck()  
```  
  
**ngx.timer.at**：延时调用相应的回调方法；ngx.timer.at(秒单位延时，回调函数，回调函数的参数列表)；可以将延时设置为0即得到一个立即执行的任务，任务不会在当前请求中执行不会阻塞当前请求，而是在一个轻量级线程中执行。
 
另外根据实际情况设置如下指令  

lua_max_pending_timers 1024;  \#最大等待任务数
lua_max_running_timers 256;    \#最大同时运行任务数
 
 
**set_by_lua**   

设置 nginx 变量，我们用的 set 指令即使配合 if 指令也很难实现负责的赋值逻辑；
 
#### example.conf 配置文件  

Java **代码**  收藏代码   
  
```
location /lua_set_1 {  
    default_type "text/html";  
    set_by_lua_file $num /usr/example/lua/test_set_1.lua;  
    echo $num;  
}    
```  

**set_by_lua_file**：语法 set_by_lua_file $var lua_file arg1 arg2...; 在 lua代码中可以实现所有复杂的逻辑，但是要执行速度很快，不要阻塞；
 
#### test_set_1.lua  

Java **代码**  收藏代码  
  
```
local uri_args = ngx.req.get_uri_args()  
local i = uri_args["i"] or 0  
local j = uri_args["j"] or 0  
  
return i + j   
```  
 
得到请求参数进行相加然后返回。
 
访问如 http://192.168.1.2/lua_set_1?i=1&j=10 进行测试。 如果我们用纯 set 指令是无法实现的。
 
再举个实际例子，我们实际工作时经常涉及到网站改版，有时候需要新老并存，或者切一部分流量到新版
 
#### 首先在 example.conf 中使用 map 指令来映射 host 到指定 nginx 变量，方便我们测试  

Java **代码**  收藏代码  
  
```
############ 测试时使用的动态请求  
map $host $item_dynamic {  
    default                     "0";  
    item2014.jd.com            "1";  
}   
```  
 
如绑定 hosts  

192.168.1.2 item.jd.com;
192.168.1.2 item2014.jd.com;
 
此时我们想访问 item2014.jd.com 时访问新版，那么我们可以简单的使用如  

Java **代码**  收藏代码  
  
```
if ($item_dynamic = "1") {  
   proxy_pass http://new;  
}  
proxy_pass http://old;  
```  
 
但是我们想把商品编号为为8位(比如品类为图书的)没有改版完成，需要按照相应规则跳转到老版，但是其他的到新版；虽然使用if指令能实现，但是比较麻烦，基本需要这样  

Java **代码**  收藏代码  
  
```
set jump "0";  
if($item_dynamic = "1") {  
    set $jump "1";  
}  
if(uri ~ "^/6[0-9]{7}.html") {  
   set $jump "${jump}2";  
}   
\#非强制访问新版，且访问指定范围的商品  
if (jump == "02") {  
   proxy_pass http://old;  
}  
proxy_pass http://new;   
```  
   
以上规则还是比较简单的，如果涉及到更复杂的多重 if/else 或嵌套 if/else 实现起来就更痛苦了，可能需要到后端去做了；此时我们就可以借助 lua 了：  

Java **代码**  收藏代码  
  
```
set_by_lua $to_book '  
     local ngx_match = ngx.re.match  
     local var = ngx.var  
     local skuId = var.skuId  
     local r = var.item_dynamic ~= "1" and ngx.re.match(skuId, "^[0-9]{8}$")  
     if r then return "1" else return "0" end;  
';  
set_by_lua $to_mvd '  
     local ngx_match = ngx.re.match  
     local var = ngx.var  
     local skuId = var.skuId  
     local r = var.item_dynamic ~= "1" and ngx.re.match(skuId, "^[0-9]{9}$")  
     if r then return "1" else return "0" end;  
';  
\#自营图书  
if ($to_book) {  
    proxy_pass http://127.0.0.1/old_book/$skuId.html;  
}  
\#自营音像  
if ($to_mvd) {  
    proxy_pass http://127.0.0.1/old_mvd/$skuId.html;  
}  
\#默认  
proxy_pass http://127.0.0.1/proxy/$skuId.html;  
```  
 
### rewrite_by_lua  

执行内部 URL 重写或者外部重定向，典型的如伪静态化的 URL 重写。其默认执行在 rewrite处理阶段的最后。
 
#### example.conf 配置文件  

Java **代码**  收藏代码  
  
```
location /lua_rewrite_1 {  
    default_type "text/html";  
    rewrite_by_lua_file /usr/example/lua/test_rewrite_1.lua;  
    echo "no rewrite";  
}  
```   
 
#### test_rewrite_1.lua  

Java **代码**  收藏代码  
  
```
if ngx.req.get_uri_args()["jump"] == "1" then  
   return ngx.redirect("http://www.jd.com?jump=1", 302)  
end    
```  

当我们请求 http://192.168.1.2/lua_rewrite_1 时发现没有跳转，而请求http://192.168.1.2/lua_rewrite_1?jump=1 时发现跳转到京东首页了。 此处需要301/302跳转根据自己需求定义。
 
#### example.conf 配置文件  

Java 代码  收藏代码  
  
```
location /lua_rewrite_2 {  
    default_type "text/html";  
    rewrite_by_lua_file /usr/example/lua/test_rewrite_2.lua;  
    echo "rewrite2 uri : $uri, a : $arg_a";  
}  
```  
 
#### test_rewrite_2.lua  

Java 代码  收藏代码  
  
```
if ngx.req.get_uri_args()["jump"] == "1" then  
   ngx.req.set_uri("/lua_rewrite_3", false);  
   ngx.req.set_uri("/lua_rewrite_4", false);  
   ngx.req.set_uri_args({a = 1, b = 2});  
end     
```  

**ngx.req.set_uri(uri, false)**：可以内部重写 uri（可以带参数），等价于 rewrite ^ /lua_rewrite_3；通过配合 if/else 可以实现 rewrite ^ /lua_rewrite_3 break；这种功能；此处两者都是 location 内部 url 重写，不会重新发起新的 location 匹配；  

**ngx.req.set_uri_args**：重写请求参数，可以是字符串(a=1&b=2)也可以是 table；
 
访问如 http://192.168.1.2/lua_rewrite_2?jump=0 时得到响应
rewrite2 uri : /lua_rewrite_2, a :
 
访问如http://192.168.1.2/lua_rewrite_2?jump=1时得到响应
rewrite2 uri : /lua_rewrite_4, a : 1
 
#### example.conf 配置文件  

Java **代码**  收藏代码  
  
```
location /lua_rewrite_3 {  
    default_type "text/html";  
    rewrite_by_lua_file /usr/example/lua/test_rewrite_3.lua;  
    echo "rewrite3 uri : $uri";  
}  
```  

####c test_rewrite_3.lua  

Java **代码**  收藏代码  
  
```
if ngx.req.get_uri_args()["jump"] == "1" then  
   ngx.req.set_uri("/lua_rewrite_4", true);  
   ngx.log(ngx.ERR, "=========")  
   ngx.req.set_uri_args({a = 1, b = 2});  
end    
```  

**ngx.req.set_uri(uri, true)**：可以内部重写 uri，即会发起新的匹配 location 请求，等价于 rewrite ^ /lua_rewrite_4 last；此处看 error log 是看不到我们记录的log。
 
所以请求如 http://192.168.1.2/lua_rewrite_3?jump=1 会到新的 location 中得到响应，此处没有 /lua_rewrite_4，所以匹配到 /lua 请求，得到类似如下的响应
global variable : 2 , shared memory : 1 hello world
 
即
rewrite ^ /lua_rewrite_3;                 等价于  ngx.req.set_uri("/lua_rewrite_3", false);
rewrite ^ /lua_rewrite_3 break;       等价于  ngx.req.set_uri("/lua_rewrite_3", false); 加 if/else判断/break/return
rewrite ^ /lua_rewrite_4 last;           等价于  ngx.req.set_uri("/lua_rewrite_4", true);
 
注意，在使用 rewrite_by_lua 时，开启 rewrite_log on;后也看不到相应的 rewrite log。
 
### access_by_lua  

用于访问控制，比如我们只允许内网 ip 访问，可以使用如下形式  

Java **代码**  收藏代码  
  
```
allow     127.0.0.1;  
allow     10.0.0.0/8;  
allow     192.168.0.0/16;  
allow     172.16.0.0/12;  
deny      all;  
```  
 
#### example.conf 配置文件  

Java **代码**  收藏代码  
  
```
location /lua_access {  
    default_type "text/html";  
    access_by_lua_file /usr/example/lua/test_access.lua;  
    echo "access";  
}  
```  

#### test_access.lua  

Java **代码**  收藏代码  
  
```
if ngx.req.get_uri_args()["token"] ~= "123" then  
   return ngx.exit(403)  
end    
```   

即如果访问如 http://192.168.1.2/lua_access?token=234 将得到 403 Forbidden 的响应。这样我们可以根据如 cookie/ 用户 token 来决定是否有访问权限。
 
 
### content_by_lua     

此指令之前已经用过了，此处就不讲解了。
 
另外在使用PCRE进行正则匹配时需要注意正则的写法，具体规则请参考 [http://wiki.nginx.org/HttpLuaModule](http://wiki.nginx.org/HttpLuaModule)中的Special PCRE Sequences部 分。还有其他的注意事项也请阅读官方文档。
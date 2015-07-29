# Lua 模块开发  
  
在实际开发中，不可能把所有代码写到一个大而全的 lua 文件中，需要进行分模块开发；而且模块化是高性能 Lua 应用的关键。使用 require 第一次导入模块后，所有 Nginx 进程全局共享模块的数据和代码，每个 Worker 进程需要时会得到此模块的一个副本（Copy-On-Write），即模块可以认为是每 Worker 进程共享而不是每 Nginx Server 共享；另外注意之前我们使用init\_by\_lua 中初始化的全局变量是每请求复制一个；如果想在多个 Worker 进程间共享数据可以使用 ngx.shared.DICT 或如 Redis 之类的存储。
 
在 /usr/example/lualib 中已经提供了大量第三方开发库如 cjson、redis 客户端、mysql客户端：  

cjson.so
resty/
   aes.lua
   core.lua
   dns/
   lock.lua
   lrucache/
   lrucache.lua
   md5.lua
   memcached.lua
   mysql.lua
   random.lua
   redis.lua
   ……
 
 
需要注意在使用前需要将库在 nginx.conf 中导入：  

Java **代码**    
  
```
\#lua模块路径，其中”;;”表示默认搜索路径，默认到/usr/servers/nginx下找  
lua_package_path "/usr/example/lualib/?.lua;;";  #lua 模块  
lua_package_cpath "/usr/example/lualib/?.so;;";  #c模块    
```  
 
使用方式是在lua中通过如下方式引入  

Java **代码**  
  
```
local cjson = require(“cjson”)  
local redis = require(“resty.redis”)    
```  
 
接下来我们来开发一个简单的 lua 模块。  

Java **代码**  
  
```
vim /usr/example/lualib/module1.lua   
```  
 
Java **代码**  
   
```
local count = 0  
local function hello()  
   count = count + 1  
   ngx.say("count : ", count)  
end  
  
local _M = {  
   hello = hello  
}  
  
return _M    
```  

开发时将所有数据做成局部变量/局部函数；通过 _M 导出要暴露的函数，实现模块化封装。
 
接下来创建 test_module_1.lua  

Java **代码**   
  
```
vim /usr/example/lua/test_module_1.lua  
```  
  
Java **代码**  
  
```
local module1 = require("module1")  
  
module1.hello()   
```  
 
 使用 local var = require ("模块名")，该模块会到 lua\_package\_path 和lua\_package\_cpath 声明的的位置查找我们的模块，对于多级目录的使用 require ("目录1.目录2.模块名")加载。
 
example.conf 配置  

Java **代码**  
  
```
location /lua_module_1 {  
    default_type 'text/html';  
    lua_code_cache on;  
    content_by_lua_file /usr/example/lua/test_module_1.lua;  
}  
```  
 
访问如 http://192.168.1.2/lua_module_1 进行测试，会得到类似如下的数据，count 会递增
count : 1
count ：2
……
count ：N
 
此时可能发现 count 一直递增，假设我们的 worker_processes  2，我们可以通过 kill -9 nginx worker process 杀死其中一个 Worker 进程得到 count 数据变化。
 
 
假设我们创建了 vim /usr/example/lualib/test/module2.lua 模块，可以通过 local module2 = require("test.module2") 加载模块
 
基本的模块开发就完成了，如果是只读数据可以通过模块中声明 local 变量存储；如果想在每Worker 进程共享，请考虑竞争；如果要在多个 Worker 进程间共享请考虑使用ngx.shared.DICT 或如 Redis 存储。
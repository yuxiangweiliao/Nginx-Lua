# 常用 Lua 开发库 1-redis、mysql、http 客户端  
  
对于开发来说需要有好的生态开发库来辅助我们快速开发，而 Lua 中也有大多数我们需要的第三方开发库如 Redis、Memcached、Mysql、Http 客户端、JSON、模板引擎等。
一些常见的 Lua 库可以在 github 上搜索，[https://github.com/search?utf8=%E2%9C%93&q=lua+resty](https://github.com/search?utf8=%E2%9C%93&q=lua+resty)。
 
## Redis 客户端   

lua-resty-redis 是为基于 cosocket API 的 ngx_lua 提供的 Lua redis 客户端，通过它可以完成 Redis 的操作。默认安装 OpenResty 时已经自带了该模块，使用文档可参考[https://github.com/openresty/lua-resty-redis](https://github.com/openresty/lua-resty-redis)。
 
在测试之前请启动 Redis 实例：  

nohup /usr/servers/redis-2.8.19/src/redis-server  /usr/servers/redis-2.8.19/redis_6660.conf &
 
### 基本操作
 
编辑 test\_redis\_baisc.lua  

Java **代码**   
  
```
local function close_redis(red)  
    if not red then  
        return  
    end  
    local ok, err = red:close()  
    if not ok then  
        ngx.say("close redis error : ", err)  
    end  
end  
  
local redis = require("resty.redis")  
  
--创建实例  
local red = redis:new()  
--设置超时（毫秒）  
red:set_timeout(1000)  
--建立连接  
local ip = "127.0.0.1"  
local port = 6660  
local ok, err = red:connect(ip, port)  
if not ok then  
    ngx.say("connect to redis error : ", err)  
    return close_redis(red)  
end  
--调用API进行处理  
ok, err = red:set("msg", "hello world")  
if not ok then  
    ngx.say("set msg error : ", err)  
    return close_redis(red)  
end  
  
--调用API获取数据  
local resp, err = red:get("msg")  
if not resp then  
    ngx.say("get msg error : ", err)  
    return close_reedis(red)  
end  
--得到的数据为空处理  
if resp == ngx.null then  
    resp = ''  --比如默认值  
end  
ngx.say("msg : ", resp)  
  
close_redis(red)    
```  

基本逻辑很简单，要注意此处判断是否为 nil，需要跟 ngx.null 比较。
 
example.conf 配置文件  

Java **代码**    
  
```
 location /lua_redis_basic {  
    default_type 'text/html';  
    lua_code_cache on;  
    content_by_lua_file /usr/example/lua/test_redis_basic.lua;  
}  
```  
  
访问如 http://192.168.1.2/lua_redis_basic 进行测试，正常情况得到如下信息
msg : hello world
 
### 连接池  

建立 TCP 连接需要三次握手而释放 TCP 连接需要四次握手，而这些往返时延仅需要一次，以后应该复用 TCP 连接，此时就可以考虑使用连接池，即连接池可以复用连接。  

我们只需要将之前的 close\_redis 函数改造为如下即可：   

Java **代码**    
  
```
local function close_redis(red)  
    if not red then  
        return  
    end  
    --释放连接(连接池实现)  
    local pool_max_idle_time = 10000 --毫秒  
    local pool_size = 100 --连接池大小  
    local ok, err = red:set_keepalive(pool_max_idle_time, pool_size)  
    if not ok then  
        ngx.say("set keepalive error : ", err)  
    end  
end    
```  

即设置空闲连接超时时间防止连接一直占用不释放；设置连接池大小来复用连接。
 
此处假设调用 red:set\_keepalive()，连接池大小通过 nginx.conf 中 http 部分的如下指令定义：  

\#默认连接池大小，默认 30
lua_socket_pool_size 30;
\#默认超时时间,默认 60s
lua\_socket\_keepalive\_timeout 60s;
 
注意：
1、 连接池是每 Worker 进程的，而不是每 Server 的；  
2、 当连接超过最大连接池大小时，会按照 LRU 算法回收空闲连接为新连接使用；  
3、 连接池中的空闲连接出现异常时会自动被移除；  
4、 连接池是通过 ip 和 port 标识的，即相同的 ip 和 port 会使用同一个连接池（即使是不同类型的客户端如 Redis、Memcached）；  
5、 连接池第一次 set\_keepalive 时连接池大小就确定下了，不会再变更；  
6、 cosocket 的连接池 http://wiki.nginx.org/HttpLuaModule#tcpsock:setkeepalive。
 
### pipeline  

pipeline 即管道，可以理解为把多个命令打包然后一起发送；MTU（Maxitum Transmission Unit 最大传输单元）为二层包大小，一般为 1500 字节；而 MSS（Maximum Segment Size 最大报文分段大小）为四层包大小，其一般是 1500-20（IP 报头）-20（TCP 报头）=1460 字节；因此假设我们执行的多个 Redis 命令能在一个报文中传输的话，可以减少网络往返来提高速度。因此可以根据实际情况来选择走 pipeline 模式将多个命令打包到一个报文发送然后接受响应，而 Redis 协议也能很简单的识别和解决粘包。
 
修改之前的代码片段  

Java **代码**  
  
```
red:init_pipeline()  
red:set("msg1", "hello1")  
red:set("msg2", "hello2")  
red:get("msg1")  
red:get("msg2")  
local respTable, err = red:commit_pipeline()  
  
--得到的数据为空处理  
if respTable == ngx.null then  
    respTable = {}  --比如默认值  
end  
  
--结果是按照执行顺序返回的一个table  
for i, v in ipairs(respTable) do  
   ngx.say("msg : ", v, "<br/>")  
end    
```  

通过 init\_pipeline() 初始化，然后通过 commit\_pipieline() 打包提交 init\_pipeline() 之后的Redis命令；返回结果是一个 lua table，可以通过 ipairs 循环获取结果；
 
配置相应 location，测试得到的结果  

msg : OK
msg : OK
msg : hello1
msg : hello2
 
 
Redis Lua 脚本  

利用 Redis 单线程特性，可以通过在 Redis 中执行 Lua 脚本实现一些原子操作。如之前的red:get("msg") 可以通过如下两种方式实现：  

1、直接 eval：  

Java **代码**   
  
``` 
local resp, err = red:eval("return redis.call('get', KEYS[1])", 1, "msg");  
```  
   
2、script load 然后 evalsha  SHA1 校验和，这样可以节省脚本本身的服务器带宽：
Java **代码**  
  
``` 
local sha1, err = red:script("load",  "return redis.call('get', KEYS[1])");  
if not sha1 then  
   ngx.say("load script error : ", err)  
   return close_redis(red)  
end  
ngx.say("sha1 : ", sha1, "<br/>")  
local resp, err = red:evalsha(sha1, 1, "msg");  
```  
  
首先通过 script load 导入脚本并得到一个 sha1 校验和（仅需第一次导入即可），然后通过evalsha 执行 sha1 校验和即可，这样如果脚本很长通过这种方式可以减少带宽的消耗。 
 
此处仅介绍了最简单的 redis lua 脚本，更复杂的请参考官方文档学习使用。
 
另外 Redis 集群分片算法该客户端没有提供需要自己实现，当然可以考虑直接使用类似于Twemproxy 这种中间件实现。  

Memcached 客户端使用方式和本文类似，本文就不介绍了。
 
## Mysql 客户端   

lua-resty-mysql 是为基于 cosocket API 的 ngx_lua 提供的 Lua Mysql 客户端，通过它可以完成 Mysql 的操作。默认安装 OpenResty 时已经自带了该模块，使用文档可参考[https://github.com/openresty/lua-resty-mysql](https://github.com/openresty/lua-resty-mysql)。
 
### 编辑 test\_mysql.lua  

Java 代码    
  
```
local function close_db(db)  
    if not db then  
        return  
    end  
    db:close()  
end  
  
local mysql = require("resty.mysql")  
--创建实例  
local db, err = mysql:new()  
if not db then  
    ngx.say("new mysql error : ", err)  
    return  
end  
--设置超时时间(毫秒)  
db:set_timeout(1000)  
  
local props = {  
    host = "127.0.0.1",  
    port = 3306,  
    database = "mysql",  
    user = "root",  
    password = "123456"  
}  
  
local res, err, errno, sqlstate = db:connect(props)  
  
if not res then  
   ngx.say("connect to mysql error : ", err, " , errno : ", errno, " , sqlstate : ", sqlstate)  
   return close_db(db)  
end  
  
--删除表  
local drop_table_sql = "drop table if exists test"  
res, err, errno, sqlstate = db:query(drop_table_sql)  
if not res then  
   ngx.say("drop table error : ", err, " , errno : ", errno, " , sqlstate : ", sqlstate)  
   return close_db(db)  
end  
  
--创建表  
local create_table_sql = "create table test(id int primary key auto_increment, ch varchar(100))"  
res, err, errno, sqlstate = db:query(create_table_sql)  
if not res then  
   ngx.say("create table error : ", err, " , errno : ", errno, " , sqlstate : ", sqlstate)  
   return close_db(db)  
end  
  
--插入  
local insert_sql = "insert into test (ch) values('hello')"  
res, err, errno, sqlstate = db:query(insert_sql)  
if not res then  
   ngx.say("insert error : ", err, " , errno : ", errno, " , sqlstate : ", sqlstate)  
   return close_db(db)  
end  
  
res, err, errno, sqlstate = db:query(insert_sql)  
  
ngx.say("insert rows : ", res.affected_rows, " , id : ", res.insert_id, "<br/>")  
  
--更新  
local update_sql = "update test set ch = 'hello2' where id =" .. res.insert_id  
res, err, errno, sqlstate = db:query(update_sql)  
if not res then  
   ngx.say("update error : ", err, " , errno : ", errno, " , sqlstate : ", sqlstate)  
   return close_db(db)  
end  
  
ngx.say("update rows : ", res.affected_rows, "<br/>")  
--查询  
local select_sql = "select id, ch from test"  
res, err, errno, sqlstate = db:query(select_sql)  
if not res then  
   ngx.say("select error : ", err, " , errno : ", errno, " , sqlstate : ", sqlstate)  
   return close_db(db)  
end  
  
  
for i, row in ipairs(res) do  
   for name, value in pairs(row) do  
     ngx.say("select row ", i, " : ", name, " = ", value, "<br/>")  
   end  
end  
  
ngx.say("<br/>")  
--防止sql注入  
local ch_param = ngx.req.get_uri_args()["ch"] or ''  
--使用ngx.quote_sql_str防止sql注入  
local query_sql = "select id, ch from test where ch = " .. ngx.quote_sql_str(ch_param)  
res, err, errno, sqlstate = db:query(query_sql)  
if not res then  
   ngx.say("select error : ", err, " , errno : ", errno, " , sqlstate : ", sqlstate)  
   return close_db(db)  
end  
  
for i, row in ipairs(res) do  
   for name, value in pairs(row) do  
     ngx.say("select row ", i, " : ", name, " = ", value, "<br/>")  
   end  
end  
  
--删除  
local delete_sql = "delete from test"  
res, err, errno, sqlstate = db:query(delete_sql)  
if not res then  
   ngx.say("delete error : ", err, " , errno : ", errno, " , sqlstate : ", sqlstate)  
   return close_db(db)  
end  
  
ngx.say("delete rows : ", res.affected_rows, "<br/>")  
  
  
close_db(db)  
```  
 
对于新增/修改/删除会返回如下格式的响应：  

Java **代码**    
  
```
{  
    insert_id = 0,  
    server_status = 2,  
    warning_count = 1,  
    affected_rows = 32,  
    message = nil  
}  
```  
  
affected\_rows表示操作影响的行数，insert\_id是在使用自增序列时产生的 id。
 
对于查询会返回如下格式的响应：  

Java **代码**  
  
```
{  
    { id= 1, ch= "hello"},  
    { id= 2, ch= "hello2"}  
}   
```  
 
null 将返回 ngx.null。
 
### example.conf 配置文件  

Java **代码**   
  
```
location /lua_mysql {  
   default_type 'text/html';  
   lua_code_cache on;  
   content_by_lua_file /usr/example/lua/test_mysql.lua;  
}  
```  
 
### 访问如 http://192.168.1.2/lua_mysql?ch=hello 进行测试，得到如下结果  

Java **代码**    
  
```
insert rows : 1 , id : 2  
update rows : 1  
select row 1 : ch = hello  
select row 1 : id = 1  
select row 2 : ch = hello2  
select row 2 : id = 2  
select row 1 : ch = hello  
select row 1 : id = 1  
delete rows : 2    
```  

客户端目前还没有提供预编译 SQL 支持（即占位符替换位置变量），这样在入参时记得使用ngx.quote\_sql\_str 进行字符串转义，防止 sql 注入；连接池和之前 Redis 客户端完全一样就不介绍了。
 
对于 Mysql 客户端的介绍基本够用了，更多请参考 [https://github.com/openresty/lua-resty-mysql](https://github.com/openresty/lua-resty-mysql)。
 
其他如 MongoDB 等数据库的客户端可以从 github 上查找使用。
 
## Http 客户端  

OpenResty 默认没有提供 Http 客户端，需要使用第三方提供；当然我们可以通过ngx.location.capture 去方式实现，但是有一些限制，后边我们再做介绍。
 
我们可以从 github 上搜索相应的客户端，比如 https://github.com/pintsized/lua-resty-http。
 
### lua-resty-http
 
#### 下载 lua-resty-http 客户端到 lualib   

Java **代码**     
  
```
cd /usr/example/lualib/resty/  
wget https://raw.githubusercontent.com/pintsized/lua-resty-http/master/lib/resty/http_headers.lua  
wget https://raw.githubusercontent.com/pintsized/lua-resty-http/master/lib/resty/http.lua  
```  

#### test\_http\_1.lua  
  
Java **代码**    
  
```
local http = require("resty.http")  
--创建http客户端实例  
local httpc = http.new()  
  
local resp, err = httpc:request_uri("http://s.taobao.com", {  
    method = "GET",  
    path = "/search?q=hello",  
    headers = {  
        ["User-Agent"] = "Mozilla/5.0 (Windows NT 6.1; WOW64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/40.0.2214.111 Safari/537.36"  
    }  
})  
  
if not resp then  
    ngx.say("request error :", err)  
    return  
end  
  
--获取状态码  
ngx.status = resp.status  
  
--获取响应头  
for k, v in pairs(resp.headers) do  
    if k ~= "Transfer-Encoding" and k ~= "Connection" then  
        ngx.header[k] = v  
    end  
end  
--响应体  
ngx.say(resp.body)  
  
httpc:close()  
```  
 
响应头中的 Transfer-Encoding 和 Connection 可以忽略，因为这个数据是当前 server 输出的。
 
#### example.conf 配置文件  

Java **代码**  
  
```  
location /lua_http_1 {  
   default_type 'text/html';  
   lua_code_cache on;  
   content_by_lua_file /usr/example/lua/test_http_1.lua;  
}   
```  
 
#### 在 nginx.conf 中的 http 部分添加如下指令来做 DNS 解析  

Java **代码**  
  
```
resolver 8.8.8.8;    
```  

记得要配置 DNS 解析器 resolver 8.8.8.8，否则域名是无法解析的。   

#### 访问如 http://192.168.1.2/lua_http\_1 会看到淘宝的搜索界面。  

使用方式比较简单，如超时和连接池设置和之前 Redis 客户端一样，不再阐述。更多客户端使用规则请参考 [https://github.com/pintsized/lua-resty-http](https://github.com/pintsized/lua-resty-http)。
 
### ngx.location.capture  

ngx.location.capture 也可以用来完成 http 请求，但是它只能请求到相对于当前 nginx 服务器的路径，不能使用之前的绝对路径进行访问，但是我们可以配合 nginx upstream 实现我们想要的功能。
 
#### 在 nginx.cong中 的 http 部分添加如下 upstream 配置    

Java **代码**   
  
```
upstream backend {  
    server s.taobao.com;  
    keepalive 100;  
}    
```  

即我们将请求 upstream 到 backend；另外记得一定要添加之前的 DNS 解析器。
 
#### 在 example.conf 配置如下 location  

Java **代码**  
  
```
location ~ /proxy/(.*) {  
   internal;  
   proxy_pass http://backend/$1$is_args$args;  
}    
```  

internal 表示只能内部访问，即外部无法通过 url 访问进来； 并通过 proxy\_pass 将请求转发到 upstream。
 
#### test\_http\_2.lua  

Java **代码**    
    
```
local resp = ngx.location.capture("/proxy/search", {  
    method = ngx.HTTP_GET,  
    args = {q = "hello"}  
  
})  
if not resp then  
    ngx.say("request error :", err)  
    return  
end  
ngx.log(ngx.ERR, tostring(resp.status))  
  
--获取状态码  
ngx.status = resp.status  
  
--获取响应头  
for k, v in pairs(resp.header) do  
    if k ~= "Transfer-Encoding" and k ~= "Connection" then  
        ngx.header[k] = v  
    end  
end  
--响应体  
if resp.body then  
    ngx.say(resp.body)  
end   
```  
 
通过 ngx.location.capture 发送一个子请求，此处因为是子请求，所有请求头继承自当前请求，还有如 ngx.ctx和ngx.var 是否继承可以参考官方文档 [http://wiki.nginx.org/HttpLuaModule#ngx.location.capture](http://wiki.nginx.org/HttpLuaModule#ngx.location.capture)。 另外还提供了 ngx.location.capture_multi用于并发发出多个请求，这样总的响应时间是最慢的一个，批量调用时有用。
 
#### example.conf 配置文件  

Java **代码**    
  
```
location /lua_http_2 {  
   default_type 'text/html';  
   lua_code_cache on;  
   content_by_lua_file /usr/example/lua/test_http_2.lua;  
}  
```  
 
#### 访问如 http://192.168.1.2/lua_http_2 进行测试可以看到淘宝搜索界面。
 
我们通过 upstream+ngx.location.capture 方式虽然麻烦点，但是得到更好的性能和upstream 的连接池、负载均衡、故障转移、proxy cache 等特性。
 
不过因为继承在当前请求的请求头，所以可能会存在一些问题，比较常见的就是 gzip 压缩问题，ngx.location.capture 不会解压缩后端服务器的 GZIP 内容，解决办法可以参考[https://github.com/openresty/lua-nginx-module/issues/12](https://github.com/openresty/lua-nginx-module/issues/12)；因为我们大部分这种http 调用的都是内部服务，因此完全可以在 proxy location 中添加proxy\_pass\_request\_headers off; 来不传递请求头。
# JSON 库、编码转换、字符串处理  
  
## JSON 库
 
在进行数据传输时 JSON 格式目前应用广泛，因此从 Lua 对象与 JSON 字符串之间相互转换是一个非常常见的功能；目前 Lua 也有几个 JSON 库，本人用过 cjson、dkjson。其中 cjson的语法严格（比如 unicode \u0020\u7eaf ），要求符合规范否则会解析失败（如 \u002），而 dkjson 相对宽松，当然也可以通过修改 cjson 的源码来完成一些特殊要求。而在使用dkjson 时也没有遇到性能问题，目前使用的就是 dkjson。使用时要特别注意的是大部分 JSON库都仅支持 UTF-8 编码；因此如果你的字符编码是如 GBK 则需要先转换为 UTF-8 然后进行处理。  

### test_cjson.lua  

Java **代码**   
  
``` 
local cjson = require("cjson")  
  
--lua对象到字符串  
local obj = {  
    id = 1,  
    name = "zhangsan",  
    age = nil,  
    is_male = false,  
    hobby = {"film", "music", "read"}  
}  
  
local str = cjson.encode(obj)  
ngx.say(str, "<br/>")  
  
--字符串到lua对象  
str = '{"hobby":["film","music","read"],"is_male":false,"name":"zhangsan","id":1,"age":null}'  
local obj = cjson.decode(str)  
  
ngx.say(obj.age, "<br/>")  
ngx.say(obj.age == nil, "<br/>")  
ngx.say(obj.age == cjson.null, "<br/>")  
ngx.say(obj.hobby[1], "<br/>")  
  
  
--循环引用  
obj = {  
   id = 1  
}  
obj.obj = obj  
-- Cannot serialise, excessive nesting  
--ngx.say(cjson.encode(obj), "<br/>")  
local cjson_safe = require("cjson.safe")  
--nil  
ngx.say(cjson_safe.encode(obj), "<br/>")    
```  

null 将会转换为 cjson.null；循环引用会抛出异常 Cannot serialise, excessive nesting，默认解析嵌套深度是 1000，可以通过 cjson.encode\_max\_depth() 设置深度提高性能；使用 cjson.safe 不会抛出异常而是返回 nil。 
 
### example.conf 配置文件  

Java 代码   
  
```
location ~ /lua_cjson {  
   default_type 'text/html';  
   lua_code_cache on;  
   content_by_lua_file /usr/example/lua/test_cjson.lua;  
}  
```  

### 访问如 http://192.168.1.2/lua\_cjson 将得到如下结果  

Java 代码  
  
```
{"hobby":["film","music","read"],"is_male":false,"name":"zhangsan","id":1}  
null  
false  
true  
film  
nil  
```  
 
lua-cjson 文档 [http://www.kyne.com.au/~mark/software/lua-cjson-manual.html](http://www.kyne.com.au/~mark/software/lua-cjson-manual.html)。
 
接下来学习下 dkjson。
 
### 下载 dkjson 库  

Java **代码**  
   
```
cd /usr/example/lualib/  
wget http://dkolf.de/src/dkjson-lua.fsl/raw/dkjson.lua?name=16cbc26080996d9da827df42cb0844a25518eeb3 -O dkjson.lua  
```  
 
### test_dkjson.lua  

Java **代码**  
  
``` 
local dkjson = require("dkjson")  
  
--lua对象到字符串  
local obj = {  
    id = 1,  
    name = "zhangsan",  
    age = nil,  
    is_male = false,  
    hobby = {"film", "music", "read"}  
}  
  
local str = dkjson.encode(obj, {indent = true})  
ngx.say(str, "<br/>")  
  
--字符串到lua对象  
str = '{"hobby":["film","music","read"],"is_male":false,"name":"zhangsan","id":1,"age":null}'  
local obj, pos, err = dkjson.decode(str, 1, nil)  
  
ngx.say(obj.age, "<br/>")  
ngx.say(obj.age == nil, "<br/>")  
ngx.say(obj.hobby[1], "<br/>")  
  
--循环引用  
obj = {  
   id = 1  
}  
obj.obj = obj  
--reference cycle  
--ngx.say(dkjson.encode(obj), "<br/>")  
```  
                                       
默认情况下解析的 json 的字符会有缩排和换行，使用 {indent = true} 配置将把所有内容放在一行。和 cjson 不同的是解析 json 字符串中的 null 时会得到 nil。   
 
### example.conf 配置文件  

Java **代码**  
  
```  
location ~ /lua_dkjson {  
   default_type 'text/html';  
   lua_code_cache on;  
   content_by_lua_file /usr/example/lua/test_dkjson.lua;  
}  
```  
 
### 访问如 http://192.168.1.2/lua_dkjson 将得到如下结果  

Java 代码    
  
````
{ "hobby":["film","music","read"], "is_male":false, "name":"zhangsan", "id":1 }  
nil  
true  
film   
```  
 
dkjson 文档 [http://dkolf.de/src/dkjson-lua.fsl/home和http://dkolf.de/src/dkjson-lua.fsl/wiki?name=Documentation](http://dkolf.de/src/dkjson-lua.fsl/home和http://dkolf.de/src/dkjson-lua.fsl/wiki?name=Documentation)。
 
## 编码转换  

我们在使用一些类库时会发现大部分库仅支持 UTF-8 编码，因此如果使用其他编码的话就需要进行编码转换的处理；而 Linux 上最常见的就是 iconv，而 lua-iconv 就是它的一个 Lua API 的封装。
 
安装 lua-iconv 可以通过如下两种方式：  

ubuntu下可以使用如下方式  

Java **代码**   
   
```
apt-get install luarocks  
luarocks install lua-iconv   
cp /usr/local/lib/lua/5.1/iconv.so  /usr/example/lualib/    
```  

源码安装方式，需要有 gcc 环境  

Java **代码**  
  
```
wget https://github.com/do^Cloads/ittner/lua-iconv/lua-iconv-7.tar.gz  
tar -xvf lua-iconv-7.tar.gz  
cd lua-iconv-7  
gcc -O2 -fPIC -I/usr/include/lua5.1 -c luaiconv.c -o luaiconv.o -I/usr/include  
gcc -shared -o iconv.so -L/usr/local/lib luaiconv.o -L/usr/lib  
cp iconv.so  /usr/example/lualib/  
```  
 
### test\_iconv.lua  

Java 代码    
  
```
ngx.say("中文")    
```  

此时文件编码必须为 UTF-8，即Lua文件编码为什么里边的字符编码就是什么。
  
### example.conf 配置文件  

Java **代码**  
  
```  
location ~ /lua_iconv {  
   default_type 'text/html';  
   charset gbk;  
   lua_code_cache on;  
   content_by_lua_file /usr/example/lua/test_iconv.lua;  
}  
```  
  
通过 charset 告诉浏览器我们的字符编码为 gbk。  
 
### 访问 http://192.168.1.2/lua\_iconv 会发现输出乱码；
 
此时需要我们将 test\_iconv.lua 中的字符进行转码处理：  

Java **代码**  
  
```
local iconv = require("iconv")  
local togbk = iconv.new("gbk", "utf-8")  
local str, err = togbk:iconv("中文")  
ngx.say(str)    
```  

通过转码我们得到最终输出的内容编码为 gbk， 使用方式 iconv.new (目标编码, 源编码)。
 
有如下可能出现的错误：  

Java **代码**     
  
```
nil     
    没有错误成功。  
iconv.ERROR_NO_MEMORY  
    内存不足。  
iconv.ERROR_INVALID  
    有非法字符。  
iconv.ERROR_INCOMPLETE  
    有不完整字符。  
iconv.ERROR_FINALIZED  
    使用已经销毁的转换器，比如垃圾回收了。  
iconv.ERROR_UNKNOWN   
    未知错误  
```  
 
iconv 在转换时遇到非法字符或不能转换的字符就会失败，此时可以使用如下方式忽略转换失败的字符  

Java **代码**  
  
```
local togbk_ignore = iconv.new("GBK//IGNORE", "UTF-8")  
```  
 
另外在实际使用中进行 UTF-8 到 GBK 转换过程时，会发现有些字符在 GBK 编码表但是转换不了，此时可以使用更高的编码 GB18030 来完成转换。 
 
更多介绍请参考 [http://ittner.github.io/lua-iconv/](http://ittner.github.io/lua-iconv/)。
  
## 位运算  

Lua 5.3 之前是没有提供位运算支持的，需要使用第三方库，比如 LuaJIT 提供了 bit 库。  

### test\_bit.lua   

Java **代码**   
  
``` 
local bit = require("bit")  
ngx.say(bit.lshift(1, 2))    
```  

lshift 进行左移位运算，即得到4。
  
其他位操作 API 请参考 [http://bitop.luajit.org/api.html](http://bitop.luajit.org/api.html)。Lua 5.3 的位运算操作符 [http://cloudwu.github.io/lua53doc/manual.html#3.4.2](http://cloudwu.github.io/lua53doc/manual.html#3.4.2). 
 
## cache  

ngx\_lua 模块本身提供了全局共享内存 ngx.shared.DICT 可以实现全局共享，另外可以使用如 Redis 来实现缓存。另外还一个 lua-resty-lrucache 实现，其和 ngx.shared.DICT 不一样的是它是每 Worker 进程共享，即每个 Worker 进行会有一份缓存，而且经过实际使用发现其性能不如 ngx.shared.DICT。但是其好处就是不需要进行全局配置。
 
### 创建缓存模块来实现只初始化一次：  

Java **代码**  
  
```  
vim /usr/example/lualib/mycache.lua    
```  
 
Java **代码**   
  
```
local lrucache = require("resty.lrucache")  
--创建缓存实例，并指定最多缓存多少条目  
local cache, err = lrucache.new(200)  
if not cache then  
   ngx.log(ngx.ERR, "create cache error : ", err)  
end  
  
local function set(key, value, ttlInSeconds)  
    cache:set(key, value, ttlInSeconds)  
end  
  
local function get(key)  
    return cache:get(key)  
end  
  
local _M = {  
  set = set,  
  get = get  
}  
  
return _M   
```  
 
此处利用了模块的特性实现了每个 Worker 进行只初始化一次 cache 实例。
 
### test\_lrucache.lua    

Java **代码**  
  
```
local mycache = require("mycache")  
local count = mycache.get("count") or 0  
count = count + 1  
mycache.set("count", count, 10 * 60 * 60) --10分钟  
ngx.say(mycache.get("count"))     
```  
             
可以实现诸如访问量统计，但仅是每 Worker 进程的。   
 
### example.conf 配置文件  

Java **代码**   
  
```
location ~ /lua_lrucache {  
   default_type 'text/html';  
   lua_code_cache on;  
   content_by_lua_file /usr/example/lua/test_lrucache.lua;  
}    
```  

访问如 http://192.168.1.2/lua\_lrucache 测试。
 
更多介绍请参考 [https://github.com/openresty/lua-resty-lrucache](https://github.com/openresty/lua-resty-lrucache)。
 
## 字符串处理  

Lua 5.3 之前没有提供字符操作相关的函数，如字符串截取、替换等都是字节为单位操作；在实际使用时尤其包含中文的场景下显然不能满足需求；即使 Lua 5.3 也仅提供了基本的 UTF-8 操作。
 
Lua UTF-8 库
https://github.com/starwing/luautf8
 
LuaRocks安装  

Java **代码**  
  
```
\#首先确保git安装了  
apt-get install git  
luarocks install utf8  
cp /usr/local/lib/lua/5.1/utf8.so  /usr/example/lualib/  
```   

源码安装  

Java 代码   
  
```
wget https://github.com/starwing/luautf8/archive/master.zip  
unzip master.zip  
cd luautf8-master/  
gcc -O2 -fPIC -I/usr/include/lua5.1 -c utf8.c -o utf8.o -I/usr/include  
gcc -shared -o utf8.so -L/usr/local/lib utf8.o -L/usr/lib  
```  

### test\_utf8.lua  

Java 代码    
  
```
local utf8 = require("utf8")  
local str = "abc中文"  
ngx.say("len : ", utf8.len(str), "<br/>")  
ngx.say("sub : ", utf8.sub(str, 1, 4))   
```  
 
文件编码必须为 UTF8，此处我们实现了最常用的字符串长度计算和字符串截取。  

### example.conf 配置文件  

Java 代码    
  
```
location ~ /lua_utf8 {  
   default_type 'text/html';  
   lua_code_cache on;  
   content_by_lua_file /usr/example/lua/test_utf8.lua;  
}  
```  
  
### 访问如 http://192.168.1.2/lua\_utf8 测试得到如下结果  

len : 5
sub : abc中
 
字符串转换为 unicode 编码：  

Java **代码**   
  
```
local bit = require("bit")  
local bit_band = bit.band  
local bit_bor = bit.bor  
local bit_lshift = bit.lshift  
local string_format = string.format  
local string_byte = string.byte  
local table_concat = table.concat  
  
local function utf8_to_unicode(str)  
    if not str or str == "" or str == ngx.null then  
        return nil  
    end  
    local res, seq, val = {}, 0, nil  
    for i = 1, #str do  
        local c = string_byte(str, i)  
        if seq == 0 then  
            if val then  
                res[#res + 1] = string_format("%04x", val)  
            end  
  
           seq = c < 0x80 and 1 or c < 0xE0 and 2 or c < 0xF0 and 3 or  
                              c < 0xF8 and 4 or --c < 0xFC and 5 or c < 0xFE and 6 or  
                              0  
            if seq == 0 then  
                ngx.log(ngx.ERR, 'invalid UTF-8 character sequence' .. ",,," .. tostring(str))  
                return str  
            end  
  
            val = bit_band(c, 2 ^ (8 - seq) - 1)  
        else  
            val = bit_bor(bit_lshift(val, 6), bit_band(c, 0x3F))  
        end  
        seq = seq - 1  
    end  
    if val then  
        res[#res + 1] = string_format("%04x", val)  
    end  
    if #res == 0 then  
        return str  
    end  
    return "\\u" .. table_concat(res, "\\u")  
end  
  
ngx.say("utf8 to unicode : ", utf8_to_unicode("abc中文"), "<br/>")    
```  

如上方法将输出 utf8 to unicode : \u0061\u0062\u0063\u4e2d\u6587。
 
删除空格：  

Java **代码**   
  
``` 
local function ltrim(s)  
    if not s then  
        return s  
    end  
    local res = s  
    local tmp = string_find(res, '%S')  
    if not tmp then  
        res = ''  
    elseif tmp ~= 1 then  
        res = string_sub(res, tmp)  
    end  
    return res  
end  
  
local function rtrim(s)  
    if not s then  
        return s  
    end  
    local res = s  
    local tmp = string_find(res, '%S%s*$')  
    if not tmp then  
        res = ''  
    elseif tmp ~= #res then  
        res = string_sub(res, 1, tmp)  
    end  
  
    return res  
end  
  
local function trim(s)  
    if not s then  
        return s  
    end  
    local res1 = ltrim(s)  
    local res2 = rtrim(res1)  
    return res2  
end  
```  

字符串分割：  

Java **代码**  
  
```  
function split(szFullString, szSeparator)  
    local nFindStartIndex = 1  
    local nSplitIndex = 1  
    local nSplitArray = {}  
    while true do  
       local nFindLastIndex = string.find(szFullString, szSeparator, nFindStartIndex)  
       if not nFindLastIndex then  
        nSplitArray[nSplitIndex] = string.sub(szFullString, nFindStartIndex, string.len(szFullString))  
        break  
       end  
       nSplitArray[nSplitIndex] = string.sub(szFullString, nFindStartIndex, nFindLastIndex - 1)  
       nFindStartIndex = nFindLastIndex + string.len(szSeparator)  
       nSplitIndex = nSplitIndex + 1  
    end  
    return nSplitArray  
end    
```  

如 split("a,b,c", ",") 将得到一个分割后的 table。
 
到此基本的字符串操作就完成了，其他 luautf8 模块的 API 和 LuaAPI 类似可以参考
[http://cloudwu.github.io/lua53doc/manual.html#6.4](http://clhttp://cloudwu.github.io/lua53doc/manual.html#6.5oudwu.github.io/lua53doc/manual.html#6.4)
[http://cloudwu.github.io/lua53doc/manual.html#6.5](http://cloudwu.github.io/lua53doc/manual.html#6.5)
 
另外对于 GBK 的操作，可以先转换为 UTF-8，最后再转换为 GBK 即可。
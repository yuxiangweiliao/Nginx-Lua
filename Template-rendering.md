# 常用 Lua 开发库3-模板渲染   
  
动态 web 网页开发是 Web 开发中一个常见的场景，比如像京东商品详情页，其页面逻辑是非常复杂的，需要使用模板技术来实现。而 Lua 中也有许多模板引擎，如目前我在使用的 lua-resty-template，可以渲染很复杂的页面，借助 LuaJIT 其性能也是可以接受的。
 
如果学习过 JavaEE 中的 servlet 和 JSP 的话，应该知道 JSP 模板最终会被翻译成Servlet 来执行；而 lua-resty-template 模板引擎可以认为是 JSP，其最终会被翻译成 Lua 代码，然后通过 ngx.print 输出。
 
而 lua-resty-template 和大多数模板引擎是类似的，大体内容有：
模板位置：从哪里查找模板；
变量输出/转义：变量值输出；
代码片段：执行代码片段，完成如 if/else、for 等复杂逻辑，调用对象函数/方法；
注释：解释代码片段含义；
include：包含另一个模板片段；
其他：lua-resty-template 还提供了不需要解析片段、简单布局、可复用的代码块、宏指令等支持。
 
首先需要下载 lua-resty-template  

Java **代码**  
  
```
cd /usr/example/lualib/resty/  
wget https://raw.githubusercontent.com/bungle/lua-resty-template/master/lib/resty/template.lua  
mkdir /usr/example/lualib/resty/html  
cd /usr/example/lualib/resty/html   
wget https://raw.githubusercontent.com/bungle/lua-resty-template/master/lib/resty/template/html.lua    
```  

接下来就可以通过如下代码片段引用了  

Java **代码**  
  
```
local template = require("resty.template")  
```  
 
## 模板位置  

我们需要告诉 lua-resty-template 去哪儿加载我们的模块，此处可以通过 set 指令定义template\_location、template\_root 或者从 root 指令定义的位置加载。
 
如我们可以在 example.conf 配置文件的 server 部分定义  

Java **代码**  
  
```
\#first match ngx location  
set $template_location "/templates";  
\#then match root read file  
set $template_root "/usr/example/templates";  
也可以通过在server部分定义root指令  
```  

Java **代码**  
  
```
root /usr/example/templates;  
```  
 
其顺序是  

Java **代码**  
  
```
local function load_ngx(path)  
    local file, location = path, ngx_var.template_location  
    if file:sub(1)  == "/" then file = file:sub(2) end  
    if location and location ~= "" then  
        if location:sub(-1) == "/" then location = location:sub(1, -2) end  
        local res = ngx_capture(location .. '/' .. file)  
        if res.status == 200 then return res.body end  
    end  
    local root = ngx_var.template_root or ngx_var.document_root  
    if root:sub(-1) == "/" then root = root:sub(1, -2) end  
    return read_file(root .. "/" .. file) or path  
end   
``` 
 
1. 通过 ngx.location.capture从template\_location 查找，如果找到（状态为为 200 ）则使用该内容作为模板；此种方式是一种动态获取模板方式；
2. 如果定义了 template\_root，则从该位置通过读取文件的方式加载模板；
3. 如果没有定义 template\_root，则默认从 root 指令定义的 document\_root 处加载模板。
 
此处建议首先 template\_root，如果实在有问题再使用 template\_location，尽量不要通过 root 指令定义的 document\_root 加载，因为其本身的含义不是给本模板引擎使用的。
 
接下来定义模板位置  

Java **代码**  
  
```
mkdir /usr/example/templates  
mkdir /usr/example/templates2  
```  
 
example.conf 配置 server 部分   

Java **代码**  
  
```
\#first match ngx location  
set $template_location "/templates";  
\#then match root read file  
set $template_root "/usr/example/templates";  
  
location /templates {  
     internal;  
     alias /usr/example/templates2;  
}    
```  

首先查找 /usr/example/template2，找不到会查找 /usr/example/templates。
 
然后创建两个模板文件  

Java **代码**  
  
```
vim /usr/example/templates2/t1.html  
```  
  
内容为   

Java **代码**  
  
```
template2  
```  
 
Java **代码**  
  
```
vim /usr/example/templates/t1.html   
```  
 
内容为   

Java **代码**  
  
```
template1  
```  

test\_temlate\_1.lua   

Java **代码**  
  
```
local template = require("resty.template")  
template.render("t1.html")  
```  
 
example.conf 配置文件  

Java **代码**  
  
```
location /lua_template_1 {  
    default_type 'text/html';  
    lua_code_cache on;  
    content_by_lua_file /usr/example/lua/test_template_1.lua;  
}    
```  

访问如 http://192.168.1.2/lua\_template\_1 将看到 template2 输出。然后 rm /usr/example/templates2/t1.html，reload nginx 将看到 template1 输出。
 
接下来的测试我们会把模板文件都放到 /usr/example/templates 下。
 
### API  

使用模板引擎目的就是输出响应内容；主要用法两种：直接通过 ngx.print 输出或者得到模板渲染之后的内容按照想要的规则输出。
 
#### test\_template\_2.lua  

Java **代码**  
  
```
local template = require("resty.template")  
--是否缓存解析后的模板，默认true  
template.caching(true)  
--渲染模板需要的上下文(数据)  
local context = {title = "title"}  
--渲染模板  
template.render("t1.html", context)  
  
  
ngx.say("<br/>")  
--编译得到一个lua函数  
local func = template.compile("t1.html")  
--执行函数，得到渲染之后的内容  
local content = func(context)  
--通过ngx API输出  
ngx.say(content)    
```   

常见用法即如下两种方式：要么直接将模板内容直接作为响应输出，要么得到渲染后的内容然后按照想要的规则输出。
 
#### examle.conf 配置文件  

Java **代码**  
  
```
location /lua_template_2 {  
    default_type 'text/html';  
    lua_code_cache on;  
    content_by_lua_file /usr/example/lua/test_template_2.lua;  
}  
```  
 
## 使用示例   
  
### test\_template\_3.lua  

Java **代码**  
  
```
local template = require("resty.template")  
  
local context = {  
    title = "测试",  
    name = "张三",  
    description = "<script>alert(1);</script>",  
    age = 20,  
    hobby = {"电影", "音乐", "阅读"},  
    score = {语文 = 90, 数学 = 80, 英语 = 70},  
    score2 = {  
        {name = "语文", score = 90},  
        {name = "数学", score = 80},  
        {name = "英语", score = 70},  
    }  
}  
 
template.render("t3.html", context)    
```    

请确认文件编码为 UTF-8；context 即我们渲染模板使用的数据。 
 
### 模板文件 /usr/example/templates/t3.html  

Java **代码**  
  
```
{(header.html)}  
   <body>  
      {# 不转义变量输出 #}  
      姓名：{* string.upper(name) *}<br/>  
      {# 转义变量输出 #}  
      简介：{{description}}<br/>  
      {# 可以做一些运算 #}  
      年龄: {* age + 1 *}<br/>  
      {# 循环输出 #}  
      爱好：  
      {% for i, v in ipairs(hobby) do %}  
         {% if i > 1 then %}，{% end %}  
         {* v *}  
      {% end %}<br/>  
  
      成绩：  
      {% local i = 1; %}  
      {% for k, v in pairs(score) do %}  
         {% if i > 1 then %}，{% end %}  
         {* k *} = {* v *}  
         {% i = i + 1 %}  
      {% end %}<br/>  
      成绩2：  
      {% for i = 1, #score2 do local t = score2[i] %}  
         {% if i > 1 then %}，{% end %}  
          {* t.name *} = {* t.score *}  
      {% end %}<br/>  
      {# 中间内容不解析 #}  
      {-raw-}{(file)}{-raw-}  
{(footer.html)}    
```  

{(include_file)}：包含另一个模板文件；
 
{* var *}：变量输出；
{{ var }}：变量转义输出；
{% code %}：代码片段；
{# comment #}：注释；
{-raw-}：中间的内容不会解析，作为纯文本输出；
 
模板最终被转换为Lua代码进行执行，所以模板中可以执行任意Lua代码。 
 
### example.conf 配置文件  

Java **代码**  
  
```
location /lua_template_3 {  
    default_type 'text/html';  
    lua_code_cache on;  
    content_by_lua_file /usr/example/lua/test_template_3.lua;  
}    
```  

访问如http://192.168.1.2/lua\_template\_3进行测试。 
 
基本的模板引擎使用到此就介绍完了。 
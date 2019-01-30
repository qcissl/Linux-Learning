#Nginx Lua API

和一般的Web Server类似，我们需要接收请求、处理并输出响应。而对于请求我们需要获取如请求参数、请求头、Body体等信息；而对于处理就是调用相应的Lua代码即可；输出响应需要进行响应状态码、响应头和响应内容体的输出。因此我们从如上几个点出发即可。

 
##接收请求

###1、example.conf配置文件 

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

###2、test_request.lua 

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

ngx.var ： nginx变量，如果要赋值如ngx.var.b = 2，此变量必须提前声明；另外对于nginx location中使用正则捕获的捕获组可以使用ngx.var[捕获组数字]获取；

ngx.req.get_headers：获取请求头，默认只获取前100，如果想要获取所以可以调用ngx.req.get_headers(0)；获取带中划线的请求头时请使用如headers.user_agent这种方式；如果一个请求头有多个值，则返回的是table；

ngx.req.get_uri_args：获取url请求参数，其用法和get_headers类似；

ngx.req.get_post_args：获取post请求内容体，其用法和get_headers类似，但是必须提前调用ngx.req.read_body()来读取body体（也可以选择在nginx配置文件使用lua_need_request_body on;开启读取body体，但是官方不推荐）；

ngx.req.raw_header：未解析的请求头字符串；

ngx.req.get_body_data：为解析的请求body体内容字符串。

如上方法处理一般的请求基本够用了。另外在读取post内容体时根据实际情况设置client_body_buffer_size和client_max_body_size来保证内容在内存而不是在文件中。

 

使用如下脚本测试

    wget --post-data 'a=1&b=2' 'http://127.0.0.1/lua_request/1/2?a=3&b=4' -O -   

##输出响应 

### 1.1、example.conf配置文件

    location /lua_response_1 {  
        default_type "text/html";  
        content_by_lua_file /usr/example/lua/test_response_1.lua;  
    }  

### 1.2、test_response_1.lua 

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

 

### 2.1、example.conf配置文件

    location /lua_response_2 {  
        default_type "text/html";  
        content_by_lua_file /usr/example/lua/test_response_2.lua;  
    }  

 

### 2.2、test_response_2.lua

    ngx.redirect("http://jd.com", 302)  

ngx.redirect：重定向； 

ngx.status=状态码，设置响应的状态码；ngx.resp.get_headers()获取设置的响应状态码；
ngx.send_headers()发送响应状态码，当调用ngx.say/ngx.print时自动发送响应状态码；
可以通过ngx.headers_sent=true判断是否发送了响应状态码。

 

##其他API

###1、example.conf配置文件**

    location /lua_other {  
        default_type "text/html";  
        content_by_lua_file /usr/example/lua/test_other.lua;  
    }  

 

###2、test_other.lua

    --未经解码的请求uri  
    local request_uri = ngx.var.request_uri;  
    ngx.say("request_uri : ", request_uri, "<br/>");  
    --解码  
    ngx.say("decode request_uri : ", ngx.unescape_uri(request_uri), "<br/>");  
    --MD5  
    ngx.say("ngx.md5 : ", ngx.md5("123"), "<br/>")  
    --http time  
    ngx.say("ngx.http_time : ", ngx.http_time(ngx.time()), "<br/>")  

 

ngx.escape_uri/ngx.unescape_uri ： uri编码解码；

ngx.encode_args/ngx.decode_args：参数编码解码；

ngx.encode_base64/ngx.decode_base64：BASE64编码解码；

ngx.re.match：nginx正则表达式匹配；

更多Nginx Lua API请参考 http://wiki.nginx.org/HttpLuaModule#Nginx_API_for_Lua。

##Nginx全局内存

使用过如Java的朋友可能知道如Ehcache等这种进程内本地缓存，Nginx是一个Master进程多个Worker进程的工作方式，因此我们可能需要在多个Worker进程中共享数据，那么此时就可以使用ngx.shared.DICT来实现全局内存共享。

### 1、首先在nginx.conf的http部分分配内存大小

    #共享全局变量，在所有worker间共享  
    lua_shared_dict shared_data 1m;  

###2、example.conf配置文件

    location /lua_shared_dict {  
        default_type "text/html";  
        content_by_lua_file /usr/example/lua/test_lua_shared_dict.lua;  
    }  

###3、 test_lua_shared_dict.lua

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

更多API请参考http://wiki.nginx.org/HttpLuaModule#ngx.shared.DICT。 

到此基本的Nginx Lua API就学完了，对于请求处理和输出响应如上介绍的API完全够用了，更多API请参考官方文档。

#Nginx Lua模块指令

Nginx共11个处理阶段，而相应的处理阶段是可以做插入式处理，即可插拔式架构；另外指令可以在http、server、server if、location、location if几个范围进行配置：

![](https://i.imgur.com/uPvzIjf.png)

更详细的解释请参考http://wiki.nginx.org/HttpLuaModule#Directives。如上指令很多并不常用，因此我们只拿其中的一部分做演示。

 
##init_by_lua

每次Nginx重新加载配置时执行，可以用它来完成一些耗时模块的加载，或者初始化一些全局配置；在Master进程创建Worker进程时，此指令中加载的全局变量会进行Copy-OnWrite，即会复制到所有全局变量到Worker进程。

###1、nginx.conf配置文件中的http部分添加如下代码

    #共享全局变量，在所有worker间共享  
    lua_shared_dict shared_data 1m;  
      
    init_by_lua_file /usr/example/lua/init.lua;  

###2、init.lua

    --初始化耗时的模块  
    local redis = require 'resty.redis'  
    local cjson = require 'cjson'  
      
    --全局变量，不推荐  
    count = 1  
      
    --共享全局内存  
    local shared_data = ngx.shared.shared_data  
    shared_data:set("count", 1)  

###3、test.lua

    count = count + 1  
    ngx.say("global variable : ", count)  
    local shared_data = ngx.shared.shared_data  
    ngx.say(", shared memory : ", shared_data:get("count"))  
    shared_data:incr("count", 1)  
    ngx.say("hello world")  

###4、访问如http://192.168.1.2/lua 会发现全局变量一直不变，而共享内存一直递增

	global variable : 2 , shared memory : 8 hello world 

 另外注意一定在生产环境开启lua_code_cache，否则每个请求都会创建Lua VM实例。

 
##init_worker_by_lua

用于启动一些定时任务，比如心跳检查，定时拉取服务器配置等等；此处的任务是跟Worker进程数量有关系的，比如有2个Worker进程那么就会启动两个完全一样的定时任务。

###1、nginx.conf配置文件中的http部分添加如下代码

    init_worker_by_lua_file /usr/example/lua/init_worker.lua;  

###2、init_worker.lua

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

ngx.timer.at：延时调用相应的回调方法；ngx.timer.at(秒单位延时，回调函数，回调函数的参数列表)；可以将延时设置为0即得到一个立即执行的任务，任务不会在当前请求中执行不会阻塞当前请求，而是在一个轻量级线程中执行。

另外根据实际情况设置如下指令

lua_max_pending_timers 1024;  #最大等待任务数

lua_max_running_timers 256;    #最大同时运行任务数

##set_by_lua 

设置nginx变量，我们用的set指令即使配合if指令也很难实现负责的赋值逻辑；

###1.1、example.conf配置文件

    location /lua_set_1 {  
        default_type "text/html";  
        set_by_lua_file $num /usr/example/lua/test_set_1.lua;  
        echo $num;  
    }  

set_by_lua_file：语法set_by_lua_file $var lua_file arg1 arg2...; 在lua代码中可以实现所有复杂的逻辑，但是要执行速度很快，不要阻塞；

 

###1.2、test_set_1.lua

    local uri_args = ngx.req.get_uri_args()  
    local i = uri_args["i"] or 0  
    local j = uri_args["j"] or 0  
      
    return i + j  

得到请求参数进行相加然后返回。

 

访问如http://192.168.1.2/lua_set_1?i=1&j=10进行测试。 如果我们用纯set指令是无法实现的。

 

再举个实际例子，我们实际工作时经常涉及到网站改版，有时候需要新老并存，或者切一部分流量到新版

 

###2.1、首先在example.conf中使用map指令来映射host到指定nginx变量，方便我们测试

    ############ 测试时使用的动态请求  
    map $host $item_dynamic {  
        default                     "0";  
        item2014.jd.com            "1";  
    }  

如绑定hosts

192.168.1.2 item.jd.com;

192.168.1.2 item2014.jd.com;

 

此时我们想访问item2014.jd.com时访问新版，那么我们可以简单的使用如

    if ($item_dynamic = "1") {  
       proxy_pass http://new;  
    }  
    proxy_pass http://old;  

 

但是我们想把商品编号为为8位(比如品类为图书的)没有改版完成，需要按照相应规则跳转到老版，但是其他的到新版；虽然使用if指令能实现，但是比较麻烦，基本需要这样

    set jump "0";  
    if($item_dynamic = "1") {  
        set $jump "1";  
    }  
    if(uri ~ "^/6[0-9]{7}.html") {  
       set $jump "${jump}2";  
    }  
    #非强制访问新版，且访问指定范围的商品  
    if (jump == "02") {  
       proxy_pass http://old;  
    }  
    proxy_pass http://new;  

以上规则还是比较简单的，如果涉及到更复杂的多重if/else或嵌套if/else实现起来就更痛苦了，可能需要到后端去做了；此时我们就可以借助lua了：

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
    #自营图书  
    if ($to_book) {  
        proxy_pass http://127.0.0.1/old_book/$skuId.html;  
    }  
    #自营音像  
    if ($to_mvd) {  
        proxy_pass http://127.0.0.1/old_mvd/$skuId.html;  
    }  
    #默认  
    proxy_pass http://127.0.0.1/proxy/$skuId.html;  

  

##rewrite_by_lua 

执行内部URL重写或者外部重定向，典型的如伪静态化的URL重写。其默认执行在rewrite处理阶段的最后。

 

###1.1、example.conf配置文件

    location /lua_rewrite_1 {  
        default_type "text/html";  
        rewrite_by_lua_file /usr/example/lua/test_rewrite_1.lua;  
        echo "no rewrite";  
    }  

 

###1.2、test_rewrite_1.lua

    if ngx.req.get_uri_args()["jump"] == "1" then  
       return ngx.redirect("http://www.jd.com?jump=1", 302)  
    end  

当我们请求http://192.168.1.2/lua_rewrite_1时发现没有跳转，而请求http://192.168.1.2/lua_rewrite_1?jump=1时发现跳转到京东首页了。 此处需要301/302跳转根据自己需求定义。

 

###2.1、example.conf配置文件

    location /lua_rewrite_2 {  
        default_type "text/html";  
        rewrite_by_lua_file /usr/example/lua/test_rewrite_2.lua;  
        echo "rewrite2 uri : $uri, a : $arg_a";  
    }  

 

###2.2、test_rewrite_2.lua

    if ngx.req.get_uri_args()["jump"] == "1" then  
       ngx.req.set_uri("/lua_rewrite_3", false);  
       ngx.req.set_uri("/lua_rewrite_4", false);  
       ngx.req.set_uri_args({a = 1, b = 2});  
    end   

ngx.req.set_uri(uri, false)：可以内部重写uri（可以带参数），等价于 rewrite ^ /lua_rewrite_3；通过配合if/else可以实现 rewrite ^ /lua_rewrite_3 break；这种功能；此处两者都是location内部url重写，不会重新发起新的location匹配；

ngx.req.set_uri_args：重写请求参数，可以是字符串(a=1&b=2)也可以是table；

 

访问如http://192.168.1.2/lua_rewrite_2?jump=0时得到响应

	rewrite2 uri : /lua_rewrite_2, a :

 

访问如http://192.168.1.2/lua_rewrite_2?jump=1时得到响应

	rewrite2 uri : /lua_rewrite_4, a : 1

 

###3.1、example.conf配置文件


    location /lua_rewrite_3 {  
        default_type "text/html";  
        rewrite_by_lua_file /usr/example/lua/test_rewrite_3.lua;  
        echo "rewrite3 uri : $uri";  
    }  

 

###3.2、test_rewrite_3.lua

    if ngx.req.get_uri_args()["jump"] == "1" then  
       ngx.req.set_uri("/lua_rewrite_4", true);  
       ngx.log(ngx.ERR, "=========")  
       ngx.req.set_uri_args({a = 1, b = 2});  
    end  

ngx.req.set_uri(uri, true)：可以内部重写uri，即会发起新的匹配location请求，等价于 rewrite ^ /lua_rewrite_4 last；此处看error log是看不到我们记录的log。

 

所以请求如http://192.168.1.2/lua_rewrite_3?jump=1会到新的location中得到响应，此处没有/lua_rewrite_4，所以匹配到/lua请求，得到类似如下的响应

	global variable : 2 , shared memory : 1 hello world

 

即

rewrite ^ /lua_rewrite_3;                 等价于  ngx.req.set_uri("/lua_rewrite_3", false);

rewrite ^ /lua_rewrite_3 break;       等价于  ngx.req.set_uri("/lua_rewrite_3", false); 加 if/else判断/break/return

rewrite ^ /lua_rewrite_4 last;           等价于  ngx.req.set_uri("/lua_rewrite_4", true);

 

注意，在使用rewrite_by_lua时，开启rewrite_log on;后也看不到相应的rewrite log。

 

##access_by_lua 

用于访问控制，比如我们只允许内网ip访问，可以使用如下形式

    allow     127.0.0.1;  
    allow     10.0.0.0/8;  
    allow     192.168.0.0/16;  
    allow     172.16.0.0/12;  
    deny      all;  

 

###1.1、example.conf配置文件

    location /lua_access {  
        default_type "text/html";  
        access_by_lua_file /usr/example/lua/test_access.lua;  
        echo "access";  
    }  

 
###1.2、test_access.lua

    if ngx.req.get_uri_args()["token"] ~= "123" then  
       return ngx.exit(403)  
    end  

即如果访问如http://192.168.1.2/lua_access?token=234将得到403 Forbidden的响应。这样我们可以根据如cookie/用户token来决定是否有访问权限。


##content_by_lua   

此指令之前已经用过了，此处就不讲解了。

 

另外在使用PCRE进行正则匹配时需要注意正则的写法，具体规则请参考http://wiki.nginx.org/HttpLuaModule中的Special PCRE Sequences部分。还有其他的注意事项也请阅读官方文档。
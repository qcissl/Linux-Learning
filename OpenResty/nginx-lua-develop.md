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

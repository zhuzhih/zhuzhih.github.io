# nginx常用代理配置
[原文](http://www.cnblogs.com/fanzhidongyzby/p/5194895.html)

因为业务系统需求，需要对web服务作nginx代理，在不断的尝试过程中，简单总结了一下常见的nginx代理配置。
* 1. 最简反向代理配置

在http节点下，使用upstream配置服务地址，使用server的location配置代理映射。

```
upstream my_server {                                                         
    server 10.0.0.2:8080;                                                
    keepalive 2000;
}
server {
    listen       80;                                                         
    server_name  10.0.0.1;                                               
    client_max_body_size 1024M;

    location /my/ {
        proxy_pass http://my_server/;
        proxy_set_header Host $host:$server_port;
    }
}
```

通过该配置，访问nginx地址http://10.0.0.1:80/my的请求会被转发到my_server服务地址http://10.0.0.2:8080/。

需要注意的是，如果按照如下配置：

```
upstream my_server {                                                         
    server 10.0.0.2:8080;                                                
    keepalive 2000;
}
server {
    listen       80;                                                         
    server_name  10.0.0.1;                                               
    client_max_body_size 1024M;

    location /my/ {
        proxy_pass http://my_server;
        proxy_set_header Host $host:$server_port;
    }
}
```

那么，访问nginx地址http://10.0.0.1:80/my的请求会被转发到my_server服务地址http://10.0.0.2:8080/my。这是因为proxy_pass参数中如果不包含url的路径，则会将location的pattern识别的路径作为绝对路径。
* 2. 重定向报文代理

即便配置了nginx代理，当服务返回重定向报文时（http code为301或302），会将重定向的目标url地址放入http response报文的header的location字段内。用户浏览器收到重定向报文时，会解析出该字段并作跳转。此时新的请求报文将直接发送给服务地址，而非nginx地址。为了能让nginx拦截此类请求，必须修改重定向报文的location信息。

```
location /my/ {
    proxy_pass http://my_server;
    proxy_set_header Host $host:$server_port;

    proxy_redirect / /my/;
}
```

使用proxy_redirect可以修改重定向报文的location字段，例子中会将所有的根路径下的url代理到nginx的/my/路径下返回给用户。比如服务返回的重定向报文的location原始值为/login，那么经过nginx代理后，用户收到的报文的location字段为/my/login。此时，浏览器将会跳转到nginx的/my/login地址进行访问。

需要注意的是，服务返回的重定向报文的location字段有时会填写绝对路径（包含服务的ip/域名和端口），有时候会填写相对路径，此时需要根据实际情况进行甄别。

```
location /my/ {
    proxy_pass http://my_server;
    proxy_set_header Host $host:$server_port;

    proxy_redirect http://my_server/ http://$host:$server_port/my/;
}
```

上述配置便是将my_server服务的根路径下的所有路径代理到nginx地址的/my/路径下。当nginx配置只有一个server时，http://$host:$server_port前缀可以省略。
* 3. 报文数据替换

使用nginx代理最牛（dan）逼（sui）的情况就是http响应报文内写死了服务地址或web绝对路径。写死服务地址的情况比较少见，但也偶尔存在。最棘手的是写死了web绝对路径，尤其是绝对路径都没有公共前缀。举个例子来说：

一般的web页面会包含如下类似路径：

    /public：用于静态页面资源，如js脚本/public/js，样式表/public/css，图片/public/img等。
    /static：和/public类似。
    /api：用于后台服务API接口。
    /login：用于登录验证。
    其他。

对于这样的服务，可能的代理配置如下：

```
location /my/ {
    proxy_pass http://my_server/;
    proxy_set_header Host $host:$server_port;

    proxy_redirect / /my/;
}
location /login/ {
    proxy_pass http://my_server/public;
    proxy_set_header Host $host:$server_port;
}
location /public/ {
    proxy_pass http://my_server/public;
    proxy_set_header Host $host:$server_port;
}
location /api/ {
    proxy_pass http://my_server/api;
    proxy_set_header Host $host:$server_port;
}
```

由于web页面或静态资源内写死了类似的绝对路径，那么对于用户来说，通过页面内的链接进行跳转时，都会请求到nginx服务对应的路径上。一旦存在另一个服务也包含类似的路径，也需要nginx进行代理，那么矛盾就出现了：访问nginx的同一个路径下的请求究竟转发给哪一个服务？

要解决这个问题，必须在用户收到报文前，将报文的数据中包含的绝对路径都添加统一的前缀，如/my/public，/my/api，/my/login，这样nginx代理配置则可以简化为：

```
location /my/ {
    proxy_pass http://my_server/;
    proxy_set_header Host $host:$server_port;

    proxy_redirect / /my/;
}
location /other/ {
    proxy_pass http://other_server/;
    proxy_set_header Host $host:$server_port;

    proxy_redirect / /other/;
}
```

nginx的ngx_http_sub_module模块提供了类似的报文数据替换功能，该模块默认不会安装，需要在编译nginx时添加--with-http_sub_module参数，或者直接下载nginx的rpm包。

使用sub_filter对数据包进行替换的语法如下：

location /my/ {
    proxy_pass http://my_server/;
    proxy_set_header Host $host:$server_port;
    
    sub_filter 'href="/' 'href="/my/';
    sub_filter 'src="/' 'src="/my/';
    sub_filter_types text/html;
    sub_filter_once  off;
}

上述配置会将/my/下的所有响应报文内容的href="/替换为href="/my，以及src="/替换为src="/my，即为所有的绝对路径添加公共前缀。

注意，如果需要配置多个sub_filter，必须保证nginx是1.9.4版本之上的。
4. 总结

即便如此，sub_filter也不能解决所有问题。目前流行的js框架都会有自动渲染url的功能，也就是说，很多绝对路径并非写死在静态页面内，也是由js代码框架动态生成的，面对这样的情况，sub_filter也是无能为力了。对于这样的情况，笔者只能由衷地奉劝，还是安静的改代码吧！

---
layout:     post
title:      "Nginx基础——Rewrite规则"
subtitle:   "\"nginx基础知识\""
date:       2016-03-10 12:00:00
author:     "wangyapu"
header-img: "img/post_bg_boy.jpg"
tags:
    - nginx
---

# Rewrite规则学习记录

rewrite是nginx一个特别重要的指令，该指令可以使用正则表达式改写URI。可以指定一个或多个rewrite指令，按顺序匹配。
    
## 正则匹配规则

    ~  区分大小写匹配
    ~* 不区分大小写匹配
    !~ 和 !~* 区分大小写不匹配及不区分大小写不匹配
    
## 文件及目录匹配

    -f和!-f 判断是否存在文件
    -d和!-d 判断是否存在目录
    -e和!-e 判断是否存在文件或目录
    -x和!-x 判断文件是否可执行
    
## rewrite基本语法

    set
    if
    return
    break
    rewrite

### break指令

    使用范围：server，location，if;
    中断当前相同作用域的其他nginx配置。

### if指令

    使用范围：server，location
    检查一个条件是否符合。If指令不支持嵌套，不支持多个条件&&和||处理。
    
### return指令

    格式：return code ;
    使用范围：server，location，if;
    结束规则的执行并返回状态码给客户端。
    
### set指令

    使用环境：server，location，if
    定义一个变量，并给变量赋值。变量的值可以为文本、变量或者变量的组合。
    set $var "hello world"

### rewrite指令格式

    rewrite regex replacement [flag]

    flag标志位有四种：
    break：停止rewrite检测,也就是说当含有break flag的rewrite语句被执行时,该语句就是rewrite的最终结果。 
    last：停止rewrite检测,但是跟break有本质的不同,last的语句不一定是最终结果。
    redirect：返回302临时重定向，一般用于重定向到完整的URL(包含http:部分) 
    permanent：返回301永久重定向，一般用于重定向到完整的URL(包含http:部分)

### 应用实例(摘自网络)

> 当访问的文件和目录不存在时，重定向到某个php文件

```bash
if( !-e $request_filename )
{
    rewrite ^/(.*)$ index.php last;
}
```
    
> 目录对换 /123456/xxxx  ====>  /xxxx?id=123456

```bash
rewrite ^/(\d+)/(.+)/  /$2?id=$1 last;
```
    
> 如果客户端使用的是IE浏览器，则重定向到/ie目录下

```bash
if( $http_user_agent ~ MSIE)
{
    rewrite ^(.*)$ /ie/$1 break;
}
```    
    
> 禁止访问以/data开头的文件

```bash
location ~ ^/data
{
    deny all;
}
```
    
> 禁止访问以.sh，.flv，.mp3为文件后缀名的文件

```bash
location ~ .*\.(sh|flv|mp3)$
{
    return 403;
}
```

> 设置某些类型文件的浏览器缓存时间

```bash
location ~ .*\.(gif|jpg|jpeg|png|bmp|swf)$
{
    expires 30d;
}
```

> 文件反盗链并设置过期时间

```bash
location ~*^.+\.(jpg|jpeg|gif|png|swf|rar|zip|css|js)$ 
{
    valid_referers none blocked *.linuxidc.com*.linuxidc.net localhost 208.97.167.194;
    if ($invalid_referer) {
        rewrite ^/ http://img.linuxidc.net/leech.gif;
        return 412;
        break;
    }
    access_log  off;
    root /opt/lampp/htdocs/web;
    expires 3d;
    break;
}
```

> 将多级目录下的文件转成一个文件，增强seo效果

```bash
/job-123-456-789.html 指向/job/123/456/789.html

rewrite^/job-([0-9]+)-([0-9]+)-([0-9]+)\.html$ /job/$1/$2/jobshow_$3.html last;
```

> 域名跳转

```bash
server
{
    listen 80;
    server_name jump.linuxidc.com;
    index index.html index.htm index.php;
    root /opt/lampp/htdocs/www;
    rewrite ^/ http://www.linuxidc.com/;
    access_log off;
}
```
    
> 多域名转向

```bash
server_name www.linuxidc.comwww.linuxidc.net;
index index.html index.htm index.php;
root  /opt/lampp/htdocs;
if ($host ~ "linuxidc\.net") {
    rewrite ^(.*) http://www.linuxidc.com$1permanent;
}
```

# 附录 —— nginx全局变量

    arg_PARAMETER #这个变量包含GET请求中，如果有变量PARAMETER时的值。
    args #这个变量等于请求行中(GET请求)的参数，如：foo=123&bar=blahblah;
    binary_remote_addr #二进制的客户地址。
    body_bytes_sent #响应时送出的body字节数数量。即使连接中断，这个数据也是精确的。
    content_length #请求头中的Content-length字段。
    content_type #请求头中的Content-Type字段。
    cookie_COOKIE #cookie COOKIE变量的值
    document_root #当前请求在root指令中指定的值。
    document_uri #与uri相同。
    host #请求主机头字段，否则为服务器名称。
    hostname #Set to themachine’s hostname as returned by gethostname
    http_HEADER
    is_args #如果有args参数，这个变量等于”?”，否则等于”"，空值。
    http_user_agent #客户端agent信息
    http_cookie #客户端cookie信息
    limit_rate #这个变量可以限制连接速率。
    query_string #与args相同。
    request_body_file #客户端请求主体信息的临时文件名。
    request_method #客户端请求的动作，通常为GET或POST。
    remote_addr #客户端的IP地址。
    remote_port #客户端的端口。
    remote_user #已经经过Auth Basic Module验证的用户名。
    request_completion #如果请求结束，设置为OK。 当请求未结束或如果该请求不是请求链串的最后一个时，为空(Empty)。
    request_filename #当前请求的文件路径，由root或alias指令与URI请求生成。
    request_uri #包含请求参数的原始URI，不包含主机名，如：”/foo/bar.php?arg=baz”。不能修改。
    scheme #HTTP方法（如http，https）。
    server_protocol #请求使用的协议，通常是HTTP/1.0或HTTP/1.1。
    server_addr #服务器地址，在完成一次系统调用后可以确定这个值。
    server_name #服务器名称。
    server_port #请求到达服务器的端口号。
    


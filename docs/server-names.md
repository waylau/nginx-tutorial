# 配置服务器名称

服务器名称是用`server_name`指令来定义的，并且它决定了哪一个`server`块将用来处理给定的请求。可以使用精确名称、通配符、正则表达式来定义服务器名称。

```
server {
    listen       80;
    server_name  example.org  www.example.org;
    ...
}

server {
    listen       80;
    server_name  *.example.org;
    ...
}

server {
    listen       80;
    server_name  mail.*;
    ...
}

server {
    listen       80;
    server_name  ~^(?<user>.+)\.example\.net$;
    ...
}
```


当寻找一个虚拟服务器的名字，如果指定的名称匹配多个变体，例如，通配符和正则表达式都匹配，将会按照以下的顺序选择第一个匹配的变体：

* 精确名称
* 以星号（*）开头的最长的通配符，例如“*.example.org”
* 以星号（*）结尾的最长的通配符，例如“mail.*”
* 第一个匹配的正则表达式（根据在配置文件中出现的顺序）


## 通配符名称

通配符名称包含的星号（*）只能在名称的开头或结尾，并且只能在点号（.）的边上。像“www.*.example.org”和“w*.example.org”都是不可用的。然而，这样的名称可以使用正则表达式来指定，例如，“~^www\..+\.example\.org$” 和  “~^w.*\.example\.org$”。星号可以匹配好几个名称部分。“*.example.org”不仅能匹配`www.example.org`，还能匹配`www.sub.example.org`。

一种特殊形式的通配符“.example.org”可以用来匹配精确名称“example.org”和通配符“*.example.org”。

## 正则表达式名称

NGINX 中使用的正则表达式兼容 Perl 编程语言中的正则表达式（PCRE）。使用正则表达式时，服务器名称必须使用波浪线（~）开头：

```
server_name  ~^www\d+\.example\.net$;
```

要不然就会被当做精确名称对待，或者如果包含星号（*）的话，会被当成通配符（当然多数情况下会被当成不可用的）。在正则表达式中不要忘记使用“^”和"$"标记，并不是语法上的要求，而是逻辑上的要求。同时还要记住，域名中的点号（.）之前要加反斜杠转义。如果正则表达式中包含中括号“{”和“}”，需要用引号括起来：

```
server_name  "~^(?<name>\w\d{1,3}+)\.example\.net$";
```

要不然 NGINX 会启动失败并且显示错误信息：

```
directive "server_name" is not terminated by ";" in ...
```

指定正则表达式捕获片段之后可以当做一个变量来使用：


```
server {
    server_name   ~^(www\.)?(?<domain>.+)$;

    location / {
        root   /sites/$domain;
    }
}
```

PCRE库支持以下语法使用捕获片段：

* `?<name>`： Perl 5.10兼容语法，从PCRE-7.0开始支持
* `?'name'`： Perl 5.10兼容语法，从PCRE-7.0开始支持
* `?P<name>`： Python兼容语法，从PCRE-4.0开始支持

如果 NGINX 启动失败，并且显示错误信息：

```
pcre_compile() failed: unrecognized character after (?< in ...
```

这就意味着PCRE库太老了，这种语法“?P<name>”应该被替换。捕获片段也可以使用数字形式引用：


```
server {
    server_name   ~^(www\.)?(.+)$;

    location / {
        root   /sites/$2;
    }
}
```


然而，这种使用方式尽量限制在简单案例中使用（就像上面这样），因为数字引用形式很容易被覆盖。

## 混杂名称

有些服务器名称被区别对待。

如果没有“Host”头信息的请求需要被一个不是默认的`server`块处理，一个空的名称要被指定：

```
server {
    listen       80;
    server_name  example.org  www.example.org  "";
    ...
}
```

如果在一个`server`块中没有`server_name`被指定，NGINX 会使用空名称作为服务器名称。

> 在NGINX 0.8.48之前的版本中，机器的hostname（主机名称）会被用作服务器名称。

如果服务器名称被定义成“$hostname”（从0.9.4版开始），机器的hostname（主机名称）会被使用。

如果处理请求时想使用IP地址代替服务器名称，“Host”请求头会包含IP地址，并且会把IP地址当做服务器名称来处理请求：

```
server {
    listen       80;
    server_name  example.org
                 www.example.org
                 ""
                 192.168.1.1
                 ;
    ...
}
```

万能服务器示例中可以看到奇怪的名字“_”：

```
server {
    listen       80  default_server;
    server_name  _;
    return       444;
}
```

这个名称并不奇怪，这只是众多的与真实名称永远不会相交的无效域名中的一个。其他的无效名称像“— —”和“!@#”也可以同样适用。

NGINX 0.6.25之前的版本支持特殊的名称“*”，这被错误地解释为万能的名称。它从来都不具备万能的或通配符名称的功能。作为替代，它之前提供的功能现在被“server_name_in_redirect”指令取代。特殊名称“*”现在已经不建议使用了，而建议使用“server_name_in_redirect”指令。请注意使用“server_name”指令没有办法指定万能名称或默认服务器。这是“listen”指令的属性而不是“server_name”指令的属性。可以在`*：80`和`*：8080`上监听，一个将`*：8080`端口设为默认，另一个将`*：80`端口设为默认：

```
server {
    listen       80;
    listen       8080  default_server;
    server_name  example.net;
    ...
}

server {
    listen       80  default_server;
    listen       8080;
    server_name  example.org;
    ...
}
```

## 优化

精确名称，以星号开头的通配符，以星号结尾的通配符被存储在绑定到监听端口上的三张哈希表中。哈希表的大小在配置阶段被优化，因此一个名称在最少CPU缓存空隙被找到。

精确名称的哈希表最先被搜索。如果没找到一个名字，接着以星号开头的通配符哈希表会被搜索。如果还没找到，以星号结尾的哈希表才会被搜索。

搜索通配符名称的哈希表速度会比搜索精确名称的哈希表慢，因为名称是根据域名部分搜索的。记住，这种特殊的通配符形式“.example.org”是存储在通配符名称的哈希表中而不是存储在精确名称的哈希表中。

正则表达式会被按序测试，因为它是最慢的并且不可伸缩。

基于这些原因，尽可能使用精确名称会更好。例如，最经常被请求的服务器名称是`example.org`和`www.example.org`，像下面这样定义会更高效：

```
server {
    listen       80;
    server_name  example.org  www.example.org  *.example.org;
    ...
}
```

比这种形式更高效：

```
server {
    listen       80;
    server_name  .example.org;
    ...
}
```

如果大量的服务器名称被定义，或者特别长的服务器名称被定义，在`http`块中使用`server_names_hash_max_size`和`server_names_hash_bucket_size`就显得有必要了。根据CPU缓存行的大小，默认的`server_names_hash_bucket_size`指令的值可能是32、64或其他值。如果默认值是32，服务器名称被定义为`too.long.server.name.example.org`，NGINX 就会启动失败并显示错误信息：

```
could not build the server_names_hash,
you should increase server_names_hash_bucket_size: 32
```

在这种情况下，这个指令的值应该增加到两倍的值：

```
http {
    server_names_hash_bucket_size  64;
    ...
```


如果是大量的服务器名称被定义，另一个错误信息会显示：

```
could not build the server_names_hash,
you should increase either server_names_hash_max_size: 512
or server_names_hash_bucket_size: 32
```


在这种情况下，首先尝试去设置`server_names_hash_max_size`的值接近于服务器名称的数量。只有当这种情况无效时，或者 NGINX 的启动时间太长无法接受，再去增大`server_names_hash_bucket_size`。

如果一个端口上只定义了一个服务器名称，那么 NGINX 就不会再去测试服务器名称了（也不会在端口上建立哈希表）。然而，有一种例外的情况。如果服务器名称是带有捕捉片段的正则表达式，那么必须去执行表达式获取捕捉片段。
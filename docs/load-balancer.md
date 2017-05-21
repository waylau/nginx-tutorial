# 作为 HTTP 负载均衡器

跨多个应用程序实例的负载均衡是优化资源利用率，最大限度地提高吞吐量，降低延迟，并确保容错配置一个常用的技术。

可以使用 NGINX 作为非常有效的 HTTP 负载平衡器，将流量分配给多个应用服务器，并通过 NGINX 提高 Web 应用程序的性能、可扩展性和可靠性。

## 负载均衡的方法

NGINX 支持如下负载均衡的机制（或方法）：

* **轮询（round-robin）** : 以循环的方式将请求分发到应用服务；
* **最少连接（least-connected） **: 下次请求指派到激活连接数最少的服务；
* **IP 哈希（ip-hash）** : 基于客户端的IP地址，利用哈希函数，决定下次请求应该选中哪个服务。

### 1. 轮询 

如果没有指定负载均衡的方法，那么 NGINX 默认采用的是轮询的方式。最简单的负载均衡配置如下：

```
http {
    upstream myapp1 {
        server srv1.example.com;
        server srv2.example.com;
        server srv3.example.com;
    }

    server {
        listen 80;

        location / {
            proxy_pass http://myapp1;
        }
    }
}
```

3个同样实例的应用（srv1-srv3）是采用轮询方式。所有请求被代理到一组服务`myapp1`，同时，NGINX 运用 HTTP 负载均衡来分发请求。

反向代理被应用在 NGINX 内，包括负载均衡针对 HTTP、HTTPS、FASTCGI、uwsgi、SCGI 以及  memcached。

配置负载均衡针对 HTTPS 替代 HTTP 的话，仅仅使用 https 协议即可（proxy_pass https://myapp1）。

在为 FASTCGI、uwsgi、SCGI 或  memcached 设置负载均衡时，分别使用 fastcgi_pass、uwsgi_pass、scgi_pass 和  memcached_pass 指令。

### 2. 最少连接

在一些请求需要更长时间才能完成的情况下，最少连接可以更公正地控制应用程序实例的负载。

使用最少连接的负载平衡，NGINX 将不会加重一个有过多请求的应用服务负担，而是将它分发新的请求给最不繁忙的服务器。

在 NGINX 中需要通过设置`least_conn`来激活最少连接的负载均衡策略配置：

```
upstream myapp1 {
    least_conn;
    server srv1.example.com;
    server srv2.example.com;
    server srv3.example.com;
}
```

### 3. IP 哈希（会话持久）

注意，采用轮询或者最少连接的负载均衡策略，每个客户端的后续请求可能被分配带不同的服务器，不能保证同一个客户端总是指向同一个服务。如果需要告诉客户端分配到一个特定的应用服务，换句话，就是保持客户端的会话粘性（sticky）或者会话持久性（persitent），即总是尝试选着同一个特定的服务器，IP 哈希 负载均衡机制可以被使用。

采用 IP 哈希的策略，客户端的 IP 地址被用作一个哈希 key，决定哪个服务应该被选中来服务客户端的请求。这种方式，确保了同一个客户端来的请求将总是被指向同一个服务，除非这个服务不可用了。

配置IP 哈希负载均衡，只需要通过设置`ip_hash`来激活：

```
upstream myapp1 {
    ip_hash;
    server srv1.example.com;
    server srv2.example.com;
    server srv3.example.com;
}
```

## 权重（weight）

可以通过使用服务器的权重来影响 NGINX 的负载均衡算法，在上述轮询、最少请求、基于IP 哈希负载均衡配置中，服务器的权重没有配置，意味着所有服务器的权重都是一样的。特别是轮询，它意味着或多或少平等的分发请求到服务器（请求够多，并且请求以均匀方式进行处理，并完成够快）

当配置了一个 weight 变量到一个指定的服务后，权重被作为一个 NGINX 的负载均衡的决定的一部分：

```
upstream myapp1 {
    server srv1.example.com weight=3;
    server srv2.example.com;
    server srv3.example.com;
}
```

采用上面的配置，如果来了5个请求，3个到srv1，1个到srv2,1个到srv3。在最近的NGINX版本中，同样可以使用权重针对最少连接和IP 哈希的负载均衡策略。

## 健康监测

反向代理在 NGINX 中实现了被动的健康监测，如果响应从一个特定的服务器失败，携带着错误，NGINX 将标记这个服务器是失败的，并将尝试一段时间避免选择这个服务器作为后续请求的服务器。

`fail_timeout` 和  `max_fails` 用于设定指定时间内，应该发生连续不成功的数目。默认`max_fail`等于1，如果设置成0，相当于关闭这个服务器的健康监测。`fail_timeout`参数，定义多久服务器被标识失败。过了服务器`fail_timeout`失败超时间隔后，NGINX 将开始探测存活的客户端的请求，如果探测成功，服务被标识成存活状态。

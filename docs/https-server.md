# 配置 HTTPS 服务器

HTTPS（Hyper Text Transfer Protocol over Secure Socket Layer）是基于 SSL/TLS 安全连接的 HTTP 协议。HTTPS 是通过 SSL/TLS 提供的数据加密、身份验证和消息完整性验证等安全机制，来为 Web 访问提供了安全性保证，并广泛应用于网上银行、电子商务等领域。近年以来，在主要互联网公司和浏览器开发商的推动之下，HTTPS 在加速普及，而 HTTP 正在被加速淘汰。不加密的 HTTP 连接是不安全的，数据能在传输过程中被任何中间人能轻易地读取和操纵。

本章节将演示如何将  NGINX 配置为  HTTPS 服务器。

## 修改 nginx.conf

修改 `conf/nginx.conf` 文件，必须在配置文件 server 块中的监听指令 listen 后启用 ssl 参数，并且指定服务器证书 `ssl_certificate` 和私钥 `ssl_certificate_key` 的位置：

```
server {
    listen              443 ssl;
    server_name         www.example.com;
    ssl_certificate     www.example.com.crt;
    ssl_certificate_key www.example.com.key;
    ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
    ssl_ciphers         HIGH:!aNULL:!MD5;
    ...
}
```


服务器证书是一个公共实体，它被发送给连接到服务器的每一个客户机。私钥是一个安全实体，应该存储在具有受限访问的文件中，但它必须可被nginx主进程读取。私钥也可以存储在与服务器证书相同的文件中：

```
    ssl_certificate     www.example.com.cert;
    ssl_certificate_key www.example.com.cert;
```

在这种情况下，这个证书文件的访问权限也应受到限制。虽然证书和密钥存储在一个文件中，但只有证书被发送到客户端。

指令 `ssl_protocols` 和 `ssl_ciphers` 可用于限制仅包括强版本和密码的 SSL/TLS 连接。 默认情况下，NGINX 使 用`ssl_protocols TLSv1 TLSv1.1 TLSv1.2`版本和`ssl_ciphers HIGH:!aNULL:!MD5`密码，因此通常不需要显式地配置它们。需要注意的是，这些指令的默认值在不同的版本里面已经变更好几次了。如果想确认下不同版本的默认值，请参阅<http://nginx.org/en/docs/http/configuring_https_servers.html#compatibility>。

## HTTPS 服务器优化

SSL 操作会消耗额外的 CPU 资源。 在多处理器系统上，应该运行不少于可用 CPU 内核数的多个 工作进程 。最耗 CPU 的操作是 SSL 握手。有两种方法来最小化每个客户端执行这些操作的次数：

* **启用 `keepalive_timeout`参数**。来让这些 keepalive 的连接在一个连接中发送多个请求
* **重用 SSL 会话参数**。可以避免并行和后续连接的 SSL 握手。这些会话存储在 NGINX 工作程序的共享 SSL 会话缓存中，并由`ssl_session_cache`指令配置。 1M 的高速缓存包含大约4000个会话。默认的缓存超时时间为5分钟，可以通过使用`ssl_session_timeout`配置来增大。 下面是针对具有 10M 共享缓存的多核心系统的优化示例配置： 

```
worker_processes auto;

http {
    ssl_session_cache   shared:SSL:10m;
    ssl_session_timeout 10m;

    server {
        listen              443 ssl;
        server_name         www.example.com;
        keepalive_timeout   70;

        ssl_certificate     www.example.com.crt;
        ssl_certificate_key www.example.com.key;
        ssl_protocols       TLSv1 TLSv1.1 TLSv1.2;
        ssl_ciphers         HIGH:!aNULL:!MD5;
        ...
```

## SSL 证书链

有些浏览器可能警示由知名证书颁发机构签名的证书，而其他浏览器却能无问题的接受这些证书。这是因为这些证书颁发机构使用了中间证书来签署服务器证书，所签署的证书不存在于特定浏览器发行时内置的可信证书颁发机构颁发的证书库中。在这种情况下，颁发机构提供一组与颁发的服务器证书（根证书）串接的捆绑证书，并让服务器证书（根证书）出现在合并后文件（证书链）的捆绑证书之前：

```shell
$ cat www.example.com.crt bundle.crt > www.example.com.chained.crt
```


生成的证书链文件应该放在 `ssl_certificate`配置中：

```
server {
    listen              443 ssl;
    server_name         www.example.com;
    ssl_certificate     www.example.com.chained.crt;
    ssl_certificate_key www.example.com.key;
    ...
}
```

如果根证书和捆绑证书使用了错误的链接顺序，NGINX 将会启动失败并显示如下错误信息：

```shell
SSL_CTX_use_PrivateKey_file(" ... /www.example.com.key") failed
   (SSL: error:0B080074:x509 certificate routines:
    X509_check_private_key:key values mismatch)
```

这是因为 NGINX 尝试去使用私钥与捆绑后证书的第一个证书验证而不是它本该去验证的服务器证书。

浏览器通常会存储他们接收到的由可信证书颁发机构签发的中间证书，因此被活跃使用的浏览器可能已经拥有所需的中间证书，这样证书即使没有捆绑到证书链也不会有问题。为了确保服务器发送的是完整的证书链，可以使用 `openssl`命令行来查看，例如：

```shell
$ openssl s_client -connect www.godaddy.com:443
...
Certificate chain
 0 s:/C=US/ST=Arizona/L=Scottsdale/1.3.6.1.4.1.311.60.2.1.3=US
     /1.3.6.1.4.1.311.60.2.1.2=AZ/O=GoDaddy.com, Inc
     /OU=MIS Department/CN=www.GoDaddy.com
     /serialNumber=0796928-7/2.5.4.15=V1.0, Clause 5.(b)
   i:/C=US/ST=Arizona/L=Scottsdale/O=GoDaddy.com, Inc.
     /OU=http://certificates.godaddy.com/repository
     /CN=Go Daddy Secure Certification Authority
     /serialNumber=07969287
 1 s:/C=US/ST=Arizona/L=Scottsdale/O=GoDaddy.com, Inc.
     /OU=http://certificates.godaddy.com/repository
     /CN=Go Daddy Secure Certification Authority
     /serialNumber=07969287
   i:/C=US/O=The Go Daddy Group, Inc.
     /OU=Go Daddy Class 2 Certification Authority
 2 s:/C=US/O=The Go Daddy Group, Inc.
     /OU=Go Daddy Class 2 Certification Authority
   i:/L=ValiCert Validation Network/O=ValiCert, Inc.
     /OU=ValiCert Class 2 Policy Validation Authority
     /CN=http://www.valicert.com//emailAddress=info@valicert.com
...
```

在本示例中， www.GoDaddy.com 证书链中的 #0 号证书的证书请求者("s")由签发者("i")签发，而签发者("i")本身又是 ＃1 号证书的请求者("s")，它的证书签发者是 ＃2 号证书的请求者，它请求的证书由知名发布者 ValiCert 公司 签发，其证书存储在浏览器的内置证书库中。

如果捆绑证书没有被添加到证书链，那只有 #0 号证书会被展示出来。


## 单个 HTTP/HTTPS 虚拟主机

现在，在单个 NGINX 虚拟主上可以配置同时处理 HTTP 和 HTTPS 请求：

```
server {
    listen              80;
    listen              443 ssl;
    server_name         www.example.com;
    ssl_certificate     www.example.com.crt;
    ssl_certificate_key www.example.com.key;
    ...
}
```

在0.7.14之前的版本无法向上面那样为单个侦听套接字选择性启用SSL，而只能使用`ssl`指令为整个服务器启用 SSL，从而无法设置单个 HTTP/HTTPS 虚拟主机服务器，所以在 `listen`指令后增加了`ssl` 参数来解决此问题。因此不建议在现代版本中使用 `ssl` 这个指令。


## 基于名称的 HTTPS 服务器

当配置两个或多个HTTPS虚拟主机服务器侦听同一个 IP 地址时会出现常见问题：


```
server {
    listen          443 ssl;
    server_name     www.example.com;
    ssl_certificate www.example.com.crt;
    ...
}

server {
    listen          443 ssl;
    server_name     www.example.org;
    ssl_certificate www.example.org.crt;
    ...
}
```

使用这种配置，浏览器接收默认服务器的证书，即"www.example.com" 而不管请求的实际服务器名称，这是由SSL协议行为造成的。 SSL连接建立在浏览器发送 HTTP 请求之前，这时候 NGINX 还不知道请求的服务器名称。因此，它只能提供默认的服务器证书。

解决此问题最古老和最可靠的方法是为每个 HTTPS 虚拟主机服务器指定一个单独的IP地址：

```
server {
    listen          192.168.1.1:443 ssl;
    server_name     www.example.com;
    ssl_certificate www.example.com.crt;
    ...
}

server {
    listen          192.168.1.2:443 ssl;
    server_name     www.example.org;
    ssl_certificate www.example.org.crt;
    ...
}
```

## 包含多个名称的 SSL 证书

还有其他方法允许在几个HTTPS虚拟主机服务器之间共享单个IP地址。然而，他们都有自己的缺点。其中一种方法是在证书的 SubjectAltName 字段中使用多个名称，例如 `www.example.com` 和 `www.example.org` 。 但是， SubjectAltName 字段长度有限。

另一种方法是使用带有通配符名称的证书，例如 `*.example.org` 。 通配符证书能保护指定域的所有子域，但只限一个级别。此证书与 `www.example.org` 匹配，但不匹配 `example.org` 和 `www.sub.example.org` 。这两种方法也可以结合。证书可以在 SubjectAltName 字段中包含完全匹配和通配符名称，例如 `example.org` 和 `*.example.org` 。

最好在配置文件的 `http`区块中放置具有多个名称的证书文件及其私钥文件，以在所有其下的虚拟主机服务器中继承其单个内存副本：

```
ssl_certificate     common.crt;
ssl_certificate_key common.key;

server {
    listen          443 ssl;
    server_name     www.example.com;
    ...
}

server {
    listen          443 ssl;
    server_name     www.example.org;
    ...
}
```

## 服务器名称指示

单个 IP 地址上运行多个 HTTPS 虚拟服务器的更通用的解决方案是 TLS 服务器名称指示扩展 (SNI，RFC 6066)，其允许浏览器在 SSL 握手期间同时发送请求的服务器名称，因此，服务器就知道它应该给这个连接使用哪个证书。然而，SNI 限制了它支持的浏览器。 目前支持从以下浏览器版本及其后的版本：

* Opera 8.0;
* IE 7.0 (Windows Vista及更高版本);
* Firefox 2.0 及其他使用 Mozilla Platform rv:1.8.1 的浏览器;
* Safari 3.2.1 (Windows版本支持Windows Vista及更高版本);
* Chrome (Windows版本支持Windows Vista及更高版本).

只有域名可以在 SNI 中传递，然而如果请求包含 IP 地址，一些浏览器可能错误地把服务器的 IP 地址作为其名称进行传递，我们不能依赖于这个。

为了在nginx中使用 SNI，必须在构建 NGINX 的 OpenSSL 库以及运行时的动态链接库中支持它。OpenSSL从0.9.8f版本支持SNI，如果在编译时给`config`增加了 `--enable-tlsext` 选项；从OpenSSL 0.9.8j版本开始默认启用此选项。如果NGINX是以支持SNI方式构建的，当使用“-V”参数运行时，NGINX 会显示这一信息：


```
$ nginx -V
...
TLS SNI support enabled
...
```

但是，如果启用 SNI 的 NGINX与没有 SNI 支持的 OpenSSL 库动态链接，NGINX 将显示警告：

```
nginx was built with SNI support, however, now it is linked
dynamically to an OpenSSL library which has no tlsext support,
therefore SNI is not available
```


# 压缩与解压

压缩响应通常会显着减少传输数据的大小。 然而，由于压缩在运行时发生，所以会增加处理开销，这可能会对性能产生负面影响。 

在向客户端发送响应之前，NGINX 会执行压缩，但不会“重复压缩”已经压缩过的响应。

## 启用压缩

要启用压缩，在 gzip 指令上请使用`on`参数:

```
gzip on;
```

默认情况下，NGINX 仅压缩使用MIME 类型 为 `text/html`的响应。要压缩其他 MIME 类型的响应，请包含`gzip_types`指令并列出相应的类型。

```
gzip_types text/plain application/xml;
```

要指定要压缩的响应的最小长度，请使用`gzip_min_length`指令。默认值为20字节，下面示例调整为1000：

```
gzip_min_length 1000;
```

默认情况下，NGINX 不会压缩对代理请求的响应（来自代理服务器的请求）。请求是否来自代理服务器是由请求中`Via`头字段的是否存来确定的。要配置这些响应的压缩，请使用`gzip_proxied`指令。该指令具有多个参数来指定 NGINX 应压缩哪种代理请求。例如，仅对不会在代理服务器上缓存的请求进行压缩响应，为此，`gzip_proxied`指令具有指示 NGINX 在响应中检查`Cache-Control`头字段的参数，如果值是 no-cache、no-store 或 private，则压缩响应。另外，您必须包括 expired 的参数来检查`Expires`头字段的值。这些参数在以下示例中与 auth 参数一起设置，该参数检查`Authorization`头字段的存在（授权响应特定于最终用户，并且通常不被缓存）：

```
gzip_proxied no-cache no-store private expired auth;
```

与大多数其他指令一样，配置压缩的指令可以包含在`http`上下文中，也可以包含在 `server` 或 `location` 块中。

gzip 压缩的整体配置可能如下所示。

```
server {
    gzip on;
    gzip_types      text/plain application/xml;
    gzip_proxied    no-cache no-store private expired auth;
    gzip_min_length 1000;
    ...
}
```

## 启用解压

某些客户端不支持使用 gzip 编码方法的响应。同时，可能需要存储压缩数据，或者即时压缩响应并将它们存储在缓存中。为了都能成功地服务于接受或者不接受压缩数据的客户端，针对后一种类型的客户端时，NGINX 可以在将数据发送时即时解压缩数据。

要启用运行时解压缩，请使用`gunzip`指令。

```
location /storage/ {
    gunzip on;
    ...
}
```


`gunzip`指令可以在与`gzip`指令相同的上下文中指定：

```
server {
    gzip on;
    gzip_min_length 1000;
    gunzip on;
    ...
}
```


请注意，此指令在单独的模块中定义（见`ngx_http_gunzip_module`<http://nginx.org/en/docs/http/ngx_http_gunzip_module.html>），默认情况下可能不包含在开源 NGINX 构建中。

## 发送压缩文件

要将文件的压缩版本发送到客户端而不是常规文件，请在适当的上下文中将`gzip_static`指令设置为 on。


```
location / {
    gzip_static on;
}
```


在这种情况下，为了服务`/path/to/file`的请求，NGINX 尝试查找并发送文件`/path/to/file.gz`。如果文件不存在，或客户端不支持 gzip，则 NGINX 将发送未压缩版本的文件。

请注意，`gzip_static`指令不启用即时压缩。它只是使用压缩工具预先压缩的文件。要在运行时压缩内容（而不仅仅是静态内容），请使用`gzip`指令。

该指令在单独的模块中定义（见`ngx_http_gzip_static_module`<http://nginx.org/en/docs/http/ngx_http_gzip_static_module.html>），默认情况下可能不包含在开源NGINX构建中。
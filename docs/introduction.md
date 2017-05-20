# NGINX 简介

## 什么是 NGINX


NGINX是一个免费的、开源的、高性能的 HTTP 服务器和反向代理，以及一个  IMAP/POP3 代理服务器。 NGINX以其高性能、稳定性、丰富的功能集、简单的配置和低资源消耗而闻名。

NGINX 是为解决[C10K](http://www.kegel.com/c10k.html) 问题而编写的少数服务器之一。与传统服务器不同，NGINX 不依赖于线程来处理请求。相反，它使用更加可扩展的事件驱动（异步）架构。这种架构在负载下使用小的但更重要的是可预测的内存量。即使您不希望处理数千个并发请求，您仍然可以从 NGINX 的高性能和小内存中获益。 NGINX 在各个方向扩展：从最小的 VPS 一直到大型服务器集群。

> 所谓 C10K 问题，指的是服务器同时支持成千上万个客户端的问题，也就是"Concurrent 10000 Connection"（这也是c10k这个名字的由来）。由于硬件成本的大幅度降低和硬件技术的进步，如果一台服务器同时能够服务更多的客户端，那么也就意味着服务每一个客户端的成本大幅度降低，从这个角度来看，C10K 问题显得非常有意义。

NGINX拥有诸如 Netflix、Hulu、Pinterest、CloudFlare、Airbnb、WordPress.com、GitHub、SoundCloud、Zynga、Eventbrite、Zappos、Media Temple、Heroku、RightScale、Engine、Yard、MaxCDN 等众多高知名度网站。
。

NGINX 它具有有很多非常优越的特性:

* **作为 Web 服务器**：相比 Apache，NGINX 使用更少的资源，支持更多的并发连接，体现更高的效率，这点使 NGINX 尤其受到虚拟主机提供商的欢迎；
* **作为负载均衡服务器**：NGINX 既可以在内部直接支持 Rails 和 PHP，也可以支持作为  HTTP 代理服务器 对外进行服务。NGINX 用 C 编写，系统资源开销小， CPU 使用效率高；
* **作为邮件代理服务器**: NGINX 同时也是一个非常优秀的邮件代理服务器（最早开发这个产品的目的之一也是作为邮件代理服务器）。

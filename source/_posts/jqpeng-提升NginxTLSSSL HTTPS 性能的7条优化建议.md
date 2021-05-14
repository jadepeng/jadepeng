---
title: 提升NginxTLSSSL HTTPS 性能的7条优化建议
tags: ["nginx","jqpeng"]
categories: ["博客","jqpeng"]
date: 2021-02-04 21:27
---
文章作者:jqpeng
原文链接: [提升NginxTLSSSL HTTPS 性能的7条优化建议](https://www.cnblogs.com/xiaoqi/p/nginx-tls.html)

自2018年7月起，谷歌浏览器开始将“ HTTP”网站标记为“不安全”。在过去的几年中，互联网已经迅速过渡到HTTPS，Chrome浏览器的流量超过70％，并且Web排名前100位的网站中有80多个现在默认使用HTTPS 当前Nginx作为最常见的服务器，广泛用于负载均衡（LB)、网关、反向代理。考虑到这一点，让我们看一下Nginx调优技巧，改善Nginx + HTTPS的性能以获得更好的TTFB和更少的延迟。

![提升Nginx SSL/HTTPS性能的7条建议](https://p3-tt.byteimg.com/origin/dfic-imagehandler/0f5b86a5-a1d0-4d4e-a386-43fa9dbda992?from=pc)

HTTPS 优化

# 1. 开启 HTTP/2

HTTP/2最初是在Nginx版本1.9.5中实现的，以取代spdy。在Nginx上启用HTTP/2模块很简单。

原先的配置：


    listen 443 ssl;


修改为：


    listen 443 ssl http2;


可以通过curl来验证：


    curl --http2 -I https://domain.com/


# 2. 开启 SSL session 缓存

启用 SSL Session 缓存可以减少 TLS 的反复验证，减少 TLS 握手。 1M 的内存就可以缓存 4000 个连接，非常划算，现在内存便宜，尽量开启。


    ssl_session_cache shared:SSL:50m; # 1m 4000个，
    ssl_session_timeout 1h; # 1小时过期 1 hour during which sessions can be re-used.


# 3. 禁用 SSL session tickets

由于Nginx中尚未实现SSL session tickets，可以关闭。


    ssl_session_tickets off;


# 4. 禁用 TLS version 1.0

1.3已经出来。1.0可以丢进历史垃圾堆


    ssl_protocols TLSv1 TLSv1.1 TLSv1.2 TLSv1.3;


修改为


    ssl_protocols TLSv1.2 TLSv1.3;


# 5. 启用OCSP Stapling

如果不启用 OCSP Stapling 的话，在用户连接你的服务器的时候，需要去验证证书，这个验证证书的时间不可控，我们开启OCSP Stapling后，可以省掉这一步。


    ssl_stapling on;
    ssl_stapling_verify on;
    ssl_trusted_certificate /path/to/full_chain.pem;
    resolver 8.8.8.8 8.8.4.4 valid=300s;
    resolver_timeout 5s;


# 6. 减小ssl buffer size

ssl\_buffer\_size 控制在发送数据时的 buffer 大小，默认情况下，缓冲区设置为16k，为了最大程度地减少TTFB（至第一个字节的时间），最好使用较小的值，这样TTFB可以节省大约30 – 50ms。


    ssl_buffer_size 4k;


# 7. 调整 Cipher 优先级

更新更快的 Cipher放前面，这样延迟更小。


    # 手动启用 cipher 列表
    ssl_prefer_server_ciphers on;  # prefer a list of ciphers to prevent old and slow ciphers
    ssl_ciphers 'EECDH+AESGCM:EDH+AESGCM:AES256+EECDH:AES256+EDH';



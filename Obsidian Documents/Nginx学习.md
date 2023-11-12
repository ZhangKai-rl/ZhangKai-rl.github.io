
[学习地址](https://www.bilibili.com/video/BV1F5411J7vK/?spm_id_from=333.337.search-card.all.click&vd_source=1e5e6020713271f4be4391eae1c22f08 “黑马”)

Nginx是一款轻量级高性能HTTPWeb服务器、反向代理服务器可用于负载均衡，是一个中间件。

![](Pasted%20image%2020230506174422.png)
Nginx作为服务器1，中间件，完成反向代理、负载均衡功能。

==反向代理== ：一个服务器来自动接受客户端请求，并转发给相应的服务器。

-------


>Nginx作用

- 反向代理
- 负载均衡
- 动静分离

正向代理、反向代理区别：正向代理代理客户端，反向代理代理服务端。也就是说正向代理客户端主动向指定代理服务器发请求；反向代理客户端是被被动代理。
![](Pasted%20image%2020230506175314.png)

>Nginx负载均衡两种策略：

- 内置策略：轮询、加权轮询、ip hash
- 扩展策略
ip hash根据hash结果将同一个客户端ip请求发给同一台服务器进行处理，可以解决session不共享问题。但是如果这台服务器挂了，那么全部信息丢失，所以推荐redis

>动静分离理解

动态静态文件分离。不需要经过后台处理的文件：如css html等称为静态文件。
![](Pasted%20image%2020230506180036.png)



---

> 常用命令

开启nginx：
`service nginx start   /     cd /usr/sbin   &&  ./nginx `
关闭nginx：
`nginx -s stop`
重新加载配置文件：
`nginx -s reload`
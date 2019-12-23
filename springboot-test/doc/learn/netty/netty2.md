# netty

## 为什么不自己开发NIO
做一个NIO服务器，如果单独的进行NIO开发，不免要解决很多难题如，处理网络的闪断，客户端的重复接入，客户端的安全认证，消息的编解码，半包读写等
，同时java原生Nio的api比较复杂，需要完全掌握channel，buffer，selector，socketChannel，serversocketChannel等，除此之外，NIO设计Rector模式和多线程，你需要熟悉网络编程和多线程编程等  
jdk的NIObug，如epoll会导致selector空轮询，最终导致cpu 100%。

## why netty
netty是大多数开源框架都在用的，都嵌入其中如dubbo等，  
①：api使用简单
②：功能强大，预置多种解码编码
③：定制能力强 
④： 性能高，稳定
⑤：社区活跃，使用广泛

##
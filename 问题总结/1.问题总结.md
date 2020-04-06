#####

## 网络问题
### org.apache.http.NoHttpResponseException:ip : port failed to respond
产生的原因肯可能有两种
- 当服务器端由于负载过大等情况发生时，可能会导致在收到请求后无法处理(比如没有足够的线程资源)，会直接丢弃链接而不进行处理。此时客户端就回报错：NoHttpResponseException。 建议出现这种情况时，可以选择重试。
- 同一个线程的多个请求可以复用同一个长连接。客户端使用长链接的时候，服务端已经关闭了该连接，导致出现NoHttpResponseException。解决的放哪可以.
1. http请求使用重发机制，捕获NohttpResponseException的异常，重新发送请求，重发3次后还是失败才停止。由于不知道客户端捕获到NohttpResponseException这个异常后，客户端是否自动关闭了这个连接，每次重发都需要新建连接发送。新建连接不存在太长的空闲时间问题，因此能够通过重发解决交易失败的问题。
2. 我方系统主动检查每个连接的空闲时间，允许设置连接的最大空闲时间M，即客户端建立的连接空闲M秒后，自动发起断开连接。只要这个M时间小于服务端的最大空闲时间，将完全避免服务端主动断开连接导致的异常。
[apache——http文档](http://hc.apache.org/httpcomponents-client-ga/tutorial/html/connmgmt.html#d5e659)

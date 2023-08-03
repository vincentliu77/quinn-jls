# Quinn-jls
[中文](./README.md)|[English](./README_en.md)

Quinn-jls 是 [quinn](https://github.com/quinn-rs/quinn) 的fork 分支，该库将QUIC的tls层替换为 [JLS](https://github.com/JimmyHuang454/JLS)协议，以实现
高性能，低延迟的抗白名单封锁的代理协议

## 特性
* 基于QUIC的多路复用
* 用户层的BBR拥塞控制
* 基于QUIC和tls1.3的0rtt握手
* SNI伪装（Http3网站）
* 无需域名和证书（自签或自动生成）
* 非JLS客户端自动进行UDP转发

## 关于 0 RTT
即便0rtt被使能了，也不一定能实现0rtt握手，0rtt握手有
严格的要求，不符合这些要求的握手将退化为1rtt握手：
* 服务器和客户端必须同时在内存中有0rtt密钥，也就是说，二者
必须曾经建立过连接，内存中有对方的缓存信息。因此刚启动的客户端一定会使用1rtt握手
* 0rtt密钥没有过期，如果服务器和客户端曾经建立过连接，但连接断开时间过久（数个小时），此时0rtt密钥过期，0rtt握手会被拒绝，此时将使用1rtt握手


### 0 RTT 的安全问题
0 RTT基于TLS1.3. TLS1.3 0RTT功能被广泛认为会遭受重放攻击。然而，个人认为这一安全问题被放大了，尤其是使用单服务器的代理。成功的重放攻击的条件是及其严苛的，它要求至少
有两个服务器使用同一域名（通常大型公司的网站）

根据 [RFC 8446 Section 8](https://datatracker.ietf.org/doc/html/rfc8446#page-98). 通常有两种类型的
* 第一种重放攻击直接重放0RTT握手，一般TLS协议实现的0RTT密钥至多只能使用一次，该攻击不会生效。

* 第二种更复杂一点。它要求至少有两台服务器使用同一域名，
假设客户端和服务器A建立过连接，连接断开后尝试使用0RTT握手，
此时中间人捕获并拦截了0RTT信息。由于重连接机制的存在，客户端会尝试使用1RTT进行连接，此时中间人将该连接重定向到使用同一证书的服务器B，然后用捕获的0RTT信息去和服务器A进行握手，此时数据可能被重放。

对大多数用户来说，他们不会使用服务器集群部署QUIC，因此
不会遭受重放攻击。
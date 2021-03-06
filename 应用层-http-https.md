## **http**

- 应用层超文本传输协议
- 应用于一种无状态、无连接(1.0之前)，以请求和应答方式运行的协议

## **https**

### **出现的原因**

**安全性**

- http协议不安全、未加密，在网络传输的过程中容易被黑客截获
- https 在应用层中加入了SSL子层，用于对报文进行加密传输

### **实现**

- 在应用层和传输层之间，加入了SSL子层，归属于应用层
- 对应用层的数据进行加解密安全传输

## **SSL TLS**

### **简述**

- **SSL**： Secure Socket Layer，安全套接字层
- 作用在应用层的HTTP和传输层之间，在TCP之上建立安全通道，为通过TCP传输的应用层数据提供安全保障
- TLS：Transport Layer Security，运输层安全

- **TLS**在SSL 3.0基础上进行设计，是SSL 3.0的升级版本

- 包含**记录协议**和**握手协议**

  - SSL记录协议（SSL Record Protocol）：它建立在可靠的传输协议（如TCP）之上，为高层协议提供数据封装、压缩、加密等基本功能的支持
  - SSL握手协议（SSL Handshake Protocol）：它建立在SSL记录协议之上，用于在实际的数据传输开始前，通讯双方进行身份认证、协商加密算法、交换加密密钥等

  <img src="images/SSL结构图.png" alt="image-20220108214214970" style="zoom: 50%;" />

### **原理**

- SSL身份认证，认证完成后开始传输

**发送方**

- SSL 子层 通过SSL套接字接收来自应用层的数据（如HTTP和IMAP报文）
- 在SSL子层对数据进行加密，然后传给TCP套接字

**接收方**

- SSL子层 从TCP套接字读取数据
- 解密后，通过SSL套接字把数据传递给应用层

### **关键步骤**

- server端传递 **数字证书** 到client，client接收到server端的数字证书
  - 其中数字证书中包括：CA相关信息、server端的公钥、数字签名等
- client端 收到server端公钥后，用该公钥对 **会话密钥** 进行加密传递给server端
  - 会话密钥用于之后数据安全传输
- server端收到加密后的**会话密钥**，用自己的私钥进行解密
  - 会话密钥的加解密过程属于**非对称加解密算法**
- 数据传输
  - 用会话密钥，在client端和server端用**对称加解密算法**进行加密传输

<img src="C:/Users/45043/AppData/Roaming/Typora/typora-user-images/image-20220108215541180.png" alt="image-20220108215541180" style="zoom:50%;" />



<img src="images/SSL流程图.png" alt="image-20220108221136419" style="zoom: 67%;" />

## **对称加密算法**

### **简述**

- 加密和解密用相同的密钥

### **优点**

- **高效** 适用于大量数据加密的场景

### **问题**

- 算法公开，安全性取决于密钥的大小，密钥越大越安全效率也会越低，因此要在安全性和效率之间做权衡



### **缺点**

- **安全性问题**	算法加解密本身是安全的，但是对称加密需要的是加解密用同一密钥，这就要求客户端服务器端使用同一密钥，网络传输的过程中不仅要传输密文，同样传输密钥，如果密文、密钥同时被黑客截获，造成不安全性

### **相关算法**

- DES：Date Encryption Standard
- 3DES：Triple DES
- AES：Advanced Encryption Standard

## **非对称加密算法**

### **简述**

- 加密和解密用的不同密钥，公钥加密，私钥解密
- 数字签名：用于证明数据发送方主体身份
- 数字证书：安全性认证保护传输，其中包含了数字签名、server端公钥等信息

### **优点**

- **安全性高** 公钥加密的数据只有对应的私钥才能解密

### **缺点**

- 加解密复杂、效率低、耗时较长

### **相关算法**

- RSA
- DSA
- ECC
- DH

## **http 1.x http2.0 https的区别**

### **http1.x 与 https的区别**

- http1.x基础上增加了SSL、TSL这样的安全协议，因此出现了https
  - 具体的话在应用层与传输层之间增加了一个SSL子层，归属于应用层
  - 在TCP建连之后，需要对应用层数据进行加密传输

### **http 1.x**

**http1.0**

- 1996年提出
- 请求头 + 请求体（响应头+响应体）、GET\POST
- **效率低** 每个请求作为一个单独的连接，client与server完成一次请求均需要再次重新建立连接后再断开；没有做到连接的复用



**http 1.1**

- 1997年提出
- 请求头中增加了 **connection：keep-alive** 字段，实现了连接的复用，避免了频繁的建连，提高了效率
  - connection：keep-alive 可设置默认的建连时间



**http 1.x存在的问题**

- 明文传输，基于文本格式传输
- 传输问题，同一个域名最多建立6个连接，并且顺序传输，server顺序接收，存在阻塞问题
- header冗长，甚至可能比传输的数据还长
- server不能主动push问题

### **http 2.0**

http 2.0的出现主要去解决http 1.x中存在的问题

- 效率较大的提升

- 使用二进制分帧策略，传输二进制
- 实现了单连接多路复用：单连接 + 允许帧乱序传输，server根据收到的帧id进行重组
- header压缩
- server端推送（可主动push）
  - 在未经客户端请求的情况下，server端向client发送报文

### **http3.0**

- 基于TCP 改为 UDP



## **cookie session**

- 共同解决了http无状态的问题
- 如：登录一个网站，点击一个页面跳转后，不用再去重新登陆



### **cookie**

- 保存在client端
- 由 server端生成，通过相应的报文 set-cookie 字段发送给client
- 作用
  - 用于server 去识别client身份
  - cookie中保存了client的一些操作，如：client购物时加入购物车的操作或者浏览历史



**具体实现**

- client访问server，发送请求报文给server
- server 在响应报文中生成一个set-cookie的字段一起发送给client，set-cookie中保存了session id
- client 端接收并且保存该cookie（cookie中保存了session id，用于server端提取缓存用）
- 再次请求server的时候，会发送带cookie的报文给server，server根据cookie中的session id去识别client身份以及缓存

### **session**

- 保存在server端，一种保存数据的机制
- server端根据 cookie中的session id去识别client身份，并且索引到server端session中保存的client的相关数据



### **cookie 和 session的区别**

- cookie 保存在client，session保存在server端
- session用户无法查看和修改，cookie用户可以查看修改
- session和cookie的存储容量不同
- session依赖于cookie中session id，可以理解 session是基于cookie实现的一种数据存储方式
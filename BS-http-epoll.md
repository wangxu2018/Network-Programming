# **B/S-http-epoll**

- 实现B/S通信，http协议
- 用epoll I/O多路复用技术

## **请求协议**

- 浏览器组织，发送

```
GET /hello.c Http1.1\r\n
2. Host: localhost:2222\r\n
3. User-Agent: Mozilla/5.0 (X11; Ubuntu; Linux i686; rv:24.0) Gecko/201001    01 Firefox/24.0\r\n
4. Accept: text/html,application/xhtml+xml,application/xml;q=0.9,*/*;q=0.8\r\n
5. Accept-Language: zh-cn,zh;q=0.8,en-us;q=0.5,en;q=0.3\r\n
6. Accept-Encoding: gzip, deflate\r\n
7. Connection: keep-alive\r\n
8. If-Modified-Since: Fri, 18 Jul 2014 08:36:36 GMT\r\n
【空行】\r\n
```

## **应答协议**

```
Http1.1 200 OK\r\n
2. Server: xhttpd\r\n
Content-Type：text/plain; charset=iso-8859-1\r\n 
3. Date: Fri, 18 Jul 2014 14:34:26 GMT\r\n
5. Content-Length: 32\r\n  （ 要么不写 或者 传-1， 要写务必精确 ！ ）
6. Content-Language: zh-CN\r\n
7. Last-Modified: Fri, 18 Jul 2014 08:36:36 GMT\r\n
8. Connection: close\r\n
【空行】\r\n
[数据起始。。。。。
。。。。
。。。数据终止]
```



## **流程**

```
- getline() 获取 http协议的第一行

- 从首行中拆分  GET、文件名、协议版本。 获取用户请求的文件名

- 判断文件是否存在，stat()

- 判断是文件还是目录

- 是文件-- open -- read -- 准备写回给浏览器

- 先写 http 应答协议头
	http/1.1 200 ok
    Content-Type：text/plain; charset=iso-8859-1

- 再写数据
```

## **代码**



- code/bs-http-epoll

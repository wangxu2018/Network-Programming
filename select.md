## **select**

### **简述**

- select实现服务器I/O多路复用技术
- 原理：借助内核， select 来监听， 客户端连接、数据通信事件。

### **优点**

- **跨平台** win、linux、macOS、Unix

### **缺点**

- 监听上限受文件描述符限制，最大 1024

- 在检测满足条件的fd，即正在监听的fd的时候，需要O(n)的复杂度去依次读取列表中监听的fd判断
  - 也可以用一些额外的算法或者结构体辅助记录，但是增加了编码的难度
- 用户态和内核态频繁的传递fd数据，开销大

### **相关函数**

	int select(int nfds, fd_set *readfds, fd_set *writefds,fd_set *exceptfds, struct timeval *timeout);
	
		nfds：监听的所有文件描述符中，最大文件描述符+1
	
		readfds： 读 文件描述符监听集合。	传入、传出参数
	
		writefds：写 文件描述符监听集合。	传入、传出参数		NULL
	
		exceptfds：异常 文件描述符监听集合	传入、传出参数		NULL
	
		timeout： > 0: 	设置监听超时时长
				 NULL:	阻塞监听
				 0：	非阻塞监听，轮询
		返回值：
			> 0：所有监听集合（3个）中， 满足对应事件的总数
	
			0：没有满足监听条件的文件描述符
	
			-1：errno

```
清空一个文件描述符集合：	
	void FD_ZERO(fd_set *set)

		fd_set rset;
		FD_ZERO(&rset);

将待监听的文件描述符，添加到监听集合中
	void FD_SET(int fd, fd_set *set)

		FD_SET(3, &rset);

将一个文件描述符从监听集合中 移除
	void FD_CLR(int fd, fd_set *set)

		FD_CLR（4， &rset）;
		
判断一个文件描述符是否在监听集合中
	int  FD_ISSET(int fd, fd_set *set)

		返回值： 在：1；不在：0；
		FD_ISSET（4， &rset）;
```

### **思路设计**

```
	int maxfd = 0；

	lfd = socket() ;			创建套接字

	maxfd = lfd；

	bind();					绑定地址结构

	listen();				设置监听上限

	fd_set rset， allset;			创建r监听集合

	FD_ZERO(&allset);				将r监听集合清空

	FD_SET(lfd, &allset);			将 lfd 添加至读集合中。

	while（1） {

		rset = allset；			保存监听集合
	
		ret  = select(lfd+1， &rset， NULL， NULL， NULL);		监听文件描述符集合对应事件。

		if（ret > 0） {							有监听的描述符满足对应事件
		
			if (FD_ISSET(lfd, &rset)) {				// 1 在。 0不在。

				cfd = accept（）；				建立连接，返回用于通信的文件描述符

				maxfd = cfd；

				FD_SET(cfd, &allset);				添加到监听通信描述符集合中。
			}

			for （i = lfd+1； i <= 最大文件描述符; i++）{
				if(FD_ISSET(i, &rset)){
					与客户端完成相关功能的交互
				}	
			}
		}
	}
```



### **代码实现**

```c
code/select
```




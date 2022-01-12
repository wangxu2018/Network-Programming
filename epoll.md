# **epoll**

## **简述**

- 内核态红黑树实现
- 通过epoll_wait 从内核态拷贝符合条件的链表到用户态 让用户进行处理

## **优点**

- 高效
- 突破1024文件描述符
- 支持边缘触发和水平触发；而select 和 poll 只支持水平触发

## **缺点**

- 不能跨平台，只能Linux

## **相关函数**

```
创建一棵监听红黑树
	int epoll_create(int size)
		参数 size：创建的红黑树的监听节点数量（仅供内核参考。）
		返回值：指向新创建的红黑树的根节点的 fd; -1,errno
```

	操作监听红黑树
	int epoll_ctl(int epfd, int op, int fd, struct epoll_event *event)
	参数：
		epfd：epoll_create 函数的返回值
		op：对该监听红黑数所做的操作
			EPOLL_CTL_ADD 添加fd到 监听红黑树
			EPOLL_CTL_MOD 修改fd在 监听红黑树上的监听事件
			EPOLL_CTL_DEL 将一个fd 从监听红黑树上摘下（取消监听）
		fd：待监听的fd
		event：本质 struct epoll_event 结构体 地址
			typedef union epoll_data {
				void        *ptr; // 用于之后的反应堆模型
				int          fd;	// 对应监听事件的 fd
				uint32_t     u32;
				uint64_t     u64;
			} epoll_data_t;
	
			struct epoll_event {
				uint32_t events; // Epoll events：EPOLLIN / EPOLLOUT / EPOLLERR
				epoll_data_t data; // User data variable
			};
	
		返回值：成功 0； 失败： -1 errno


	阻塞监听
	int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout)
	参数：
		epfd：epoll_create 函数的返回值
		events：传出参数，满足监听条件的结构体
		maxevents：数组 元素的总个数
		timeout：-1:阻塞;0:不阻塞;>0:超时时间（毫秒）
	返回值：
		> 0: 满足监听的 总个数。 可以用作循环上限。
		0： 没有fd满足监听事件
		-1：失败。 errno

## **epoll实现多路IO转接思路**

```
lfd = socket(); // 监听连接事件lfd
bind();
listen();

int epfd = epoll_create(1024); // epfd, 监听红黑树的树根

struct epoll_event tep, ep[1024]; // tep, 用来设置单个fd属性，ep 是 epoll_wait() 传出的满足监听事件的数组

tep.events = EPOLLIN; // 初始化 lfd的监听属性
tep.data.fd = lfd

epoll_ctl(epfd,EPOLL_CTL_ADD,lfd,&tep); // 将 lfd 添加到监听红黑树上

while (1) {
	ret = epoll_wait(epfd, ep, 1024, -1); // 实施监听

    for (i = 0; i < ret; i++) {
        if (ep[i].data.fd == lfd) {	// lfd 满足读事件，有新的客户端发起连接请求
            cfd = Accept();

            tep.events = EPOLLIN; // 初始化 cfd的监听属性
            tep.data.fd = cfd;
            epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &tep);
        } else { 	// cfd满足读事件，有客户端写数据来
            n = read(ep[i].data.fd, buf, sizeof(buf));

            if ( n == 0) {
                close(ep[i].data.fd);
                epoll_ctl(epfd， EPOLL_CTL_DEL, ep[i].data.fd , NULL);	// 将关闭的cfd，从监听树上摘下
            } else if （n > 0） {
                小--大
                write(ep[i].data.fd, buf, n);
            }
        }
    }
}
```

## **epoll 两种事件模型**

	ET模式:
		边沿触发：
			缓冲区剩余未读尽的数据不会导致 epoll_wait 返回, 新的事件满足, 才会触发
			struct epoll_event event;
			event.events = EPOLLIN | EPOLLET;
	LT模式：
		水平触发(默认即为水平触发)
		缓冲区剩余未读尽的数据会导致 epoll_wait 返回
		
	结论：
		epoll 的 ET模式，高效模式，但是只支持 非阻塞模式。 --- 忙轮询。
	
	epoll ET模式设置非阻塞模式方式：
		struct epoll_event event;
		event.events = EPOLLIN | EPOLLET;
		epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &event);
		// 设置 cfd为非阻塞模式
		int flg = fcntl(cfd, F_GETFL);	
		flg |= O_NONBLOCK;
		fcntl(cfd, F_SETFL, flg);

## **epoll 反应堆模型**

**实现**

​	epoll ET模式 + 非阻塞、轮询 + void *ptr

**原先监听模型设计**

socket、bind、listen -- epoll_create 创建监听 红黑树 --  返回 epfd -- epoll_ctl() 向树上添加一个监听fd -- while（1）--

-- epoll_wait 监听 -- 对应监听fd有事件产生 -- 返回 监听满足数组 -- 判断返回数组元素 -- lfd满足 -- Accept -- cfd 满足 

-- read() --- 小->大 -- write回去

**问题**

- read 完成后，在write的时候，可能已经断掉了
- 因此需要读写事件分离

**反应堆模型设计**

- 反应堆：不但要监听 cfd 的读事件、还要监听cfd的写事件
- 设计

	socket、bind、listen -- epoll_create 创建监听 红黑树 --  返回 epfd -- epoll_ctl() 向树上添加一个监听fd -- while（1）
			-- epoll_wait 监听 -- 对应监听fd有事件产生 -- 返回 监听满足数组 -- 判断返回数组元素 
			-- lfd满足 -- Accept 
			-- cfd 满足 -- read() --- 小->大 
				-- cfd从监听红黑树上摘下 -- EPOLLOUT -- 回调函数 
				-- epoll_ctl() -- EPOLL_CTL_ADD 重新放到红黑上监听写事件 
				-- 等待 epoll_wait 返回 -- 说明 cfd 可写 -- write回去 
				-- cfd从监听红黑树上摘下 -- EPOLLIN
				-- epoll_ctl() -- EPOLL_CTL_ADD 重新放到红黑上监听读事件
			-- epoll_wait 监听


	eventset函数：
		设置回调函数	lfd --> acceptconn()
					cfd --> recvdata()
					cfd --> senddata()
	eventadd函数：
		将一个fd，添加到 监听红黑树。设置监听 读事件，还是监听写事件



## **代码**

- 管道实现LT、ET两种事件模型

```
code/epoll/
```


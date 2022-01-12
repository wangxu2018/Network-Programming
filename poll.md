# **poll**

## **函数**

int poll(struct pollfd *fds, nfds_t nfds, int timeout)

```
参数
    fds：监听的文件描述符数组
        struct pollfd {
            int fd：待监听的文件描述符
            short events：待监听的文件描述符对应的监听事件
                        取值：POLLIN、POLLOUT、POLLERR
            short revnets：传入时，给0；
            			>0 表示传出监控事件中满足条件返回的事件 --> POLLIN、POLLOUT、POLLERR
        }

    nfds: 监听数组的，实际有效监听个数

    timeout: >0：超时时长。单位：毫秒
            -1：阻塞等待
             0：不阻塞

返回值
	返回满足对应监听事件的文件描述符 总个数
```

## **优点**

- 自带数组结构。 可以将 监听事件集合 和 返回事件集合 分离
- 拓展 监听上限。 超出 1024限制

## **缺点**

- 不能跨平台，只能Linux

 - 无法直接定位满足监听事件的文件描述符， 编码难度较大
   	- 看似监听时间和返回事件分离，但又貌似没有分离，依然要遍历数组链表
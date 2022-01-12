**read 函数返回值**

```
>0：实际读到的字节数
=0：socket中，表示对端关闭。close（）
-1：如果 errno == EINTR   被异常终端。 需要重启
	如果 errno == EAGIN 或 EWOULDBLOCK 以非阻塞方式读数据，但是没有数据。需要，再次读
	如果 errno == ECONNRESET  说明连接被 重置。 需要 close（），移除监听队列
	else 错误
```

**设置文件描述符上限**

```
查看当前计算机所能打开的最大文件个数，受硬件影响
	cat /proc/sys/fs/file-max 

当前用户下的进程，默认打开文件描述符个数
	ulimit -a
	缺省为 1024

修改：
	打开 sudo vi /etc/security/limits.conf， 写入：
	* soft nofile 65536：设置默认值， 可以直接借助命令修改。 【注销用户，使其生效】
	* hard nofile 100000：命令修改上限
```

**网络编程中读写函数**

	read --- recv()
	write --- send()


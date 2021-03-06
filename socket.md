# **socket**

## **简述**

- 网络套接字
- 一个文件描述符指向一个套接字（该套接字内部由内核借助两个缓冲区(读写)实现）

- 在通信过程中， 套接字一定是成对出现的



## **网络字节序**

- 小端法：pc本地存储；高位存高地址、低位存低地址 （小低低）

- 大端法：网络存储；高位存低地址、低位存高地址

**相关转换函数**

- htonl
  - IP  本地 --> 网络
  - 192.168.1.11 --> string --> atoi --> int --> htonl --> 网络字节序

- htons
  - Port 本地 --> 网络

- ntohl
  - IP 网络 --> 本地

- ntohs
  - Port 网络 --> 本地

**IP地址转换函数**

	本地字节序（string IP） ---> 网络字节序
	int inet_pton(int af, const char *src, void *dst)
	参数：
		af：AF_INET、AF_INET6
	
		src：传入，IP地址（点分十进制）
	
		dst：传出，转换后的 网络字节序的 IP地址。 
	
	返回值：
	    成功： 1
	    异常： 0， 说明src指向的不是一个有效的ip地址。
	    失败：-1
	
	网络字节序 ---> 本地字节序（string IP）
	const char *inet_ntop(int af, const void *src, char *dst, socklen_t size)
	参数：
		af：AF_INET、AF_INET6
	
		src: 网络字节序IP地址
	
		dst：本地字节序（string IP）
	
		size： dst 的大小。
	
	返回值：
		成功：dst；失败：NULL

## **相关函数**

#### **sockaddr地址结构**

	struct sockaddr_in addr;
	
	addr.sin_family = AF_INET/AF_INET6				man 7 ip
	
	addr.sin_port = htons(9527);
	
	// 本地字节序IP转为网络字节序IP
	// IP赋值方法1
	int dst;
	inet_pton(AF_INET, "192.157.22.45", (void *)&dst);
	addr.sin_addr.s_addr = dst;
	
	// IP赋值方法2：取出系统中有效的任意IP地址
	addr.sin_addr.s_addr = htonl(INADDR_ANY);

#### **socket**

```
int socket(int domain, int type, int protocol);		创建一个 套接字
参数：
	domain：AF_INET、AF_INET6、AF_UNIX
	type：SOCK_STREAM、SOCK_DGRAM
	protocol: 0 
返回值：
	成功：新套接字所对应文件描述符
	失败: -1 errno
```

#### **bind**

- 给socket绑定一个 地址结构

```
int bind(int sockfd, const struct sockaddr *addr, socklen_t addrlen)

参数：

	sockfd: socket 函数返回值
	
        struct sockaddr_in addr;
        addr.sin_family = AF_INET;
        addr.sin_port = htons(8888);
        addr.sin_addr.s_addr = htonl(INADDR_ANY);

	addr: 传入参数(struct sockaddr *)&addr

	addrlen: sizeof(addr) 地址结构的大小。
	
返回值:
	成功：0
	失败：-1 errno
	
如果不使用bind绑定客户端地址结构, 客户端会采用"隐式绑定".
```

#### **listen**

- 设置同时与服务器建立连接的上限数。（同时进行3次握手的客户端数量）

```
int listen(int sockfd, int backlog);

参数：
	sockfd: socket 函数返回值
	backlog：上限数值。最大值 128

返回值：
	成功：0
	失败：-1 errno
```

#### **accept**

- 阻塞等待客户端建立连接，成功的话，返回一个与客户端成功连接的socket文件描述符

```
int accept(int sockfd, struct sockaddr *addr, socklen_t *addrlen);	

参数：
	sockfd: socket 函数返回值
	
	addr：传出参数。成功与服务器建立连接的那个客户端的地址结构（IP+port）
	
		socklen_t clit_addr_len = sizeof(addr);

	addrlen：传入传出。 &clit_addr_len

		 入：addr的大小。 出：客户端addr实际大小。

返回值：

	成功：能与客户端进行数据通信的 socket 对应的文件描述。

	失败： -1 ， errno
```

#### **connect**

- 使用现有的 socket 与服务器建立连接

```
int connect(int sockfd, const struct sockaddr *addr, socklen_t addrlen);	  

参数：
	sockfd： socket 函数返回值

		struct sockaddr_in srv_addr;		// 服务器地址结构

		srv_addr.sin_family = AF_INET;

		srv_addr.sin_port = 9527 	跟服务器bind时设定的 port 完全一致。

		inet_pton(AF_INET, "服务器的IP地址"，&srv_adrr.sin_addr.s_addr);

	addr：传入参数。服务器的地址结构
	
	addrlen：服务器的地址结构的大小

返回值：
	成功：0
	失败：-1 errno
```





#### TCP实现的C/S模型

	server:
		1. socket()	创建socket
	
		2. bind()	绑定服务器地址结构
	
		3. listen()	设置监听上限
	
		4. accept()	阻塞监听客户端连接
	
		5. read(fd)	读socket获取客户端数据
	
		6. 小--大写	toupper()
	
		7. write(fd)
	
		8. close();
	
	client:
	
		1. socket()	创建socket
	
		2. connect();	与服务器建立连接
	
		3. write()	写数据到 socket
	
		4. read()	读转换后的数据。
	
		5. 显示读取结果
	
		6. close()



## **相关代码**

- **client端实现socket**
  - socket_client.c

```c
#include <stdio.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <pthread.h>

#define SERV_PORT 9527

void sys_err(const char *str)
{
	perror(str);
	exit(1);
}

int main(int argc, char *argv[])
{
    int cfd;
    int conter = 10;
    char buf[BUFSIZ];
    
    struct sockaddr_in serv_addr;          //服务器地址结构

    serv_addr.sin_family = AF_INET;
    serv_addr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr.s_addr);
    // inet_pton(AF_INET, "127.0.0.1", &serv_addr.sin_addr);

    cfd = socket(AF_INET, SOCK_STREAM, 0);
    if (cfd == -1) {
        sys_err("socket error");
    }

    int ret = connect(cfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr));
    if (ret != 0){
        sys_err("connect err");
    }

    while (--conter) {
        write(cfd, "hello\n", 6);
        ret = read(cfd, buf, sizeof(buf));
        write(STDOUT_FILENO, buf, ret);
        sleep(1);
    }

    close(cfd);

	return 0;
}
```



- **server端实现socket**
  - socket_server.c

```c
#include <stdio.h>
#include <ctype.h>
#include <sys/socket.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <string.h>
#include <unistd.h>
#include <errno.h>
#include <pthread.h>

#define SERV_PORT 9527


void sys_err(const char *str)
{
    perror(str);
    exit(1);
}

int main(int argc, char *argv[])
{
    int lfd = 0, cfd = 0;
    int ret, i;
    char buf[BUFSIZ], client_IP[1024];

    struct sockaddr_in serv_addr, clit_addr;        // 定义服务器地址结构 和 客户端地址结构
    socklen_t clit_addr_len;				        // 客户端地址结构大小

    serv_addr.sin_family = AF_INET;				    // IPv4
    serv_addr.sin_port = htons(SERV_PORT);		    // 转为网络字节序的 端口号
    serv_addr.sin_addr.s_addr = htonl(INADDR_ANY);	// 获取本机任意有效IP

    lfd = socket(AF_INET, SOCK_STREAM, 0);		    // 创建一个 socket
    if (lfd == -1) {
        sys_err("socket error");
    }

    bind(lfd, (struct sockaddr *)&serv_addr, sizeof(serv_addr));    //给服务器socket绑定地址结构（IP+port)

    listen(lfd, 128);					//	设置监听上限

    clit_addr_len = sizeof(clit_addr);	// 	获取客户端地址结构大小

    cfd = accept(lfd, (struct sockaddr *)&clit_addr, &clit_addr_len);	// 阻塞等待客户端连接请求
    if (cfd == -1) {
        sys_err("accept error");
    }

    printf("client ip:%s port:%d\n", 
            inet_ntop(AF_INET, &clit_addr.sin_addr.s_addr, client_IP, sizeof(client_IP)), 
            ntohs(clit_addr.sin_port));			// 根据accept传出参数，获取客户端 ip 和 port

    while (1) {
        ret = read(cfd, buf, sizeof(buf));		// 读客户端数据
        write(STDOUT_FILENO, buf, ret);			// 写到屏幕查看

        for (i = 0; i < ret; i++) {				// 小写 -- 大写
            buf[i] = toupper(buf[i]);
        }

        write(cfd, buf, ret);					// 将大写，写回给客户端。
    }

    close(lfd);
    close(cfd);

    return 0;
}
```


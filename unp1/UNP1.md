#UNP1笔记
------

经典必读，把关键几个章节记录并做笔记

[TOC]

------
## 第二章：传输层TCP与UDP
传输层的协议，即TCP/UDP，他们使用网络层IP协议，其实可以绕过传输层直接使用IPV4/V6接口，被称之为原始套接字接口。
![TCP/IP协议概貌]() todo
traceroute是用了ICMP和IP套接字接口，后续将会尝试开发一个IPV4/IPV6的ping与traceroute.
>* 通过在bind指定IP地址来说明他只接受某个特定本地接口的外来连接，使用INADDR_ANY来指定任意的本地IP。
2.9 缓冲区大小及限制
	>* 如果发送缓冲区容不下应用程序的所有数据，或者应用程序的缓冲区大于套接字发送缓冲区，或者套接字还有其他数据，应用程序将被挂起（前提是套接字是阻塞的）。内核直到将数据拷贝到套接字接口发送缓冲区才write接口返回。所以write调用返回仅仅**意味着我们可以重新使用应用进程的缓冲区**。
##第三章：套接字编程接口简介

作者说IPV4/IPV6的地址转换函数不一样，前者是inet_addr,inet_ntoa函数，而适用两者的新函数是inet_pton,inet_ntop。他开发了新函数以无关协议的方式使用套接字地址结构。
### 3.2 套接字地址结构
```
struct in_addr{
	in_addr_t s_addr;
};
struct sockaddr_in{
	uint8_t sin_let;
	sa_family_t sin_family;
	In_port_t sin_port;
	struct in_addr sin_addr;
	char sin_zero[8];
};//地址结构仅给定主机上使用，并不参与通信
```
从进程到内核传递地址的结构有三个函数，bind，connect，sendto，他们接受一个地址结构的指针，以及结构地址的大小(sizeof)，于此才知道要拷贝多少数据。
以此方向相反的是accept，recvfrom，gesocketname，getpeername，他们的参数是套接字结构指针和指向表示结构大小的整数指针，用来告诉application内核确切存储了多少信息。

###3.4 字节排序函数
将低序字节存储在起始位置，这是小端字节序，将高序字节存在起始地址，这是大端字节序。一个简单的判断方式是：
```
union{
	short s;
	char c[sizeof(short)];
};
un.s=0x0102;
un.c[0]==1 && un.c[1]== 2;//大端
un.c[0]==2 && un.c[1]== 1;//小端
```
以上说的都是主机字节序号，网络字节序使用的是大端字节序。使用
```
uint16_t  htons(uint16_t val);
uint32_t  htonl(uint32_t val); // host to network long 返回网络字节序

uint16_t  ntohs(uint16_t val);
uint32_t  ntohl(uint32_t val); // network to host long 返回主机字节序 / 
```
###3.9 readn,writen,readline函数
字节流套接字上的read、write函数所表现的行为不同于通常的文件IO，可能比要求的少，但这不是错误，原因可能是内核总套接字的buffer可能已经到达了极限，此时所需的是再次调用read、write函数以得到剩余的字节。**在读socket时很常见，但在写非阻塞套接字接口时才会出现**，上代码：
```c
/* Read "n" bytes from a descriptor. */
ssize_t readn(int fd, void *vptr, size_t n)
{
	size_t	nleft;
	ssize_t	nread;
	char	*ptr = vptr;
	nleft = n;
	while (nleft > 0)
	{
		if ( (nread = read(fd, ptr, nleft)) < 0) 
		{
			if (errno == EINTR)
			/* 系统调用被一个捕获的信号中断，
			这时候则重新调用一次read*/
				nread = 0;
			else
				return(-1);
		}else 
		if (nread == 0)
		{
			/*读完了*/
			break;			
		}

		nleft -= nread;
		ptr   += nread;//下次写的位置
	}
	return(n - nleft);/* return >= 0 */
}

ssize_t writen(int fd, const void *vptr, size_t n)
{
	size_t		nleft;
	ssize_t		nwritten;
	const char	*ptr;

	ptr = vptr;
	nleft = n;
	while (nleft > 0) {
		if ( (nwritten = write(fd, ptr, nleft)) <= 0) {
			if (errno == EINTR)
				nwritten = 0;
			else
				return(-1);			/* error */
		}

		nleft -= nwritten;
		ptr   += nwritten;
	}
	return(n);
}
```

##第四章 基本套接字接口
客户端调用connect时TCP三次握手，会返回的错误有:
>* 发出SYN，如果没收到这个SYN的ACK，返回ETIMEDOUT，有的bsd实现会退避重试。
>* 对SYN返回的是RST，表明对端没有启动该端口的服务，收到RST，connect立即返回ECONNREFUSED
>* SYN引发了一个目的地不可达ICMP错误，返回EHOSTUNREACH
当connect返回错误，必须关闭套接字，重新调用socket()...

bind的用途，嗯，意思是捆绑地址A，端口P到套接字S，把特定地址（必须是本机）捆绑到socket有何用途呢？
>* 对于客户端，为此套接字上发送的IP数据报分配了源IP地址
>* 对于服务器，为套接字只接受那些目的地为此IP地址的客户端连接
>* 客户端一般不bind客户端，而是由内核根据输出来选择源IP地址。

listen函数仅被服务器调用，将客户端调用connect的套接字转换为被动套接字，指示内核接受指向此套接字的连接请求，TCP状态图指示，listen将套接字从close转换为listen状态。listen的backlog指示此套接字排队的长度，需要较大backlog长度的原因是：**未完成连接队列中的挑明随着客户端SYN分节的到达并等待三次握手的完成而增大 **
[listen队列示意图]() todo listen队列示意图

accept函数由服务器调用，从已完成连接队列头返回下一个已完成连接，若为空，则进程睡眠（阻塞方式），accept返回内核自动生成的全新套接字（称为连接套接字）。
fork函数最神奇的部分是调用一次，却返回两次，在调用进程返回一次，返回的是子进程pid，在新派生的进程返回0。
accept返回前连接夭折的情况：
三次握手完成，连接建立后，客户端返回一个RST，模拟场景是，服务器调用socket，bind，listen，然后先sleep再accept，此时客户端的connect返回后，立即调用SOCKET-OPT，SO_LINGER以生成RST，Posix.1g是内核返回给application，提示错误ECONNABORTED，此时只要忽略错误，简单地再调用一次accept即可。
###客户端停止
这是正常行为，但由于父进程未处理SIGCHLD信号，子进程变成僵尸进程。unix系统的这个设定是为了给父进程维护id，CPU，内存等资源使用信息， 下图描述了客户端停止的时序图。todo
###处理SIGCHLD信号
信号可以由内核发给进程，或者进程发给进程（包括自己），捕获SIGCHLD信号，并在处理函数中调用wait，即可处理僵尸进程。需要注意的是**unix信号是不排队的**，这带来的一个问题是当有N个子进程被终止，父进程同时收到SIGCHLD信号，但使用wait函数时只信号handle只处理了一次，导致仍有0~N-1个僵尸进程。所以作者强调，使用waitpid在有未终止的子进程时，不要阻塞。
```
void sig_chld(int signo)
{
	pid_t	pid;
	int		stat;

	while((pid = waitpid(-1, &stat, WNOHANG)) > 0) {
		printf("child %d terminated\n", pid);
	}
	return;
}
```
###服务器进程终止
client->srv正常连接后，kill掉服务器，服务器发出FIN分节，客户端以ACK响应，信号SIGCHLD被发往父进程并正确处理。
客户端收到FIN意味着服务器关闭了连接服务器的一端，服务器不再发任何数据。但客户端仍阻塞在fget上。接下来：
client调用wirte，这是允许的，仅接受到FIN而且为内核并未通知application客户端，这时候，服务器返回一个RST。
接下来调用了readline，由于已经收到过FIN，readline立即返回0，根据时序，得到的错误一般是服务器过早结束
```
if (Readline(sockfd, recvline, MAXLINE) == 0)
	err_quit("str_cli: server terminated prematurely");
```
问题来了：如果客户端在读之前写两次，第一次引发了一次RST，第二次内核会发出一个SIGPIPE信号，应该捕获这个信号阻止他的默认退出行为。然后写操作将会返回EPIPE错误。
	
###服务器主机崩溃
主机崩溃时，服务器发不出任何东西，在客户端发出字符串后，主机崩溃，用tcpdump可以看到tcp持续重传了12次，约9分钟。然后返回一个ETIMEDOUT错误，如果是路由器判断的不可用，并以目的地不可达的ICMP响应，则返回EHOSTUNREACH。
>*  我们发送数据才检测出对方主机崩溃，而且要等9分钟，一种办法是调用read时设置超时时间。
>* 如果不发送数据也想检测这种错误，使用SO_KEEPALIVE技术，晚点讨论。

##第六章 IO复用：select与poll函数
select原型:
```
#include<sys/time.h>
int select(int maxfdp1,fd_set* readset,fd_set* writeset,fd_set* exceptset,const struct timeval* timeout);
```
如果一个或者N个IO条件满足，我们可以被通知到，这个能力就叫IO复用，使用场景：
>* 客户端处理多个描述字（交互式输入和网络IO）
>* 客户端处理多个套接字
>* TCP服务器既要监听，也要处理已连接的套接字

五种IO模型：
>* 阻塞/非阻塞IO
>* IO复用
>* 信号驱动IO
>* 异步IO
对于select，如果将readset、writeset、exceptset中三个指针都设置为空，则有了一个比sleep更精确的定时器，APUE利用此实现了微秒的睡眠函数sleep_us。

###描述字准备好读条件
>* 接受socket的buffer>=设置的低潮限度，TCP/UDP的buffer低潮限度可用SO_RCVLOWAT设置，默认为1字节
>* 连接读这一半关闭（接收了FIN的TCP连接），读操作时返回0表示文件结束符
>* 是监听套接字，并且已完成的连接数为非0（accept仍可能阻塞）正常情况下accept是不会阻塞的。
>* 有一个socket错误等待处理，读取errno即可，或者设置SO_ERROR，然后调用getsockopt也可以取得并清除。

###套接字准备好写条件
>* socket发送buffer字节数>=socket发送缓冲区低潮限度的value，利用SO_SNDLOWAT来设置低潮限度，一般默认为2048
>* 连接的写这一半关闭，对该socket写会产生信号SIGPIPE
>* socket存在外带数据或者处于外带标记，那他有异常条件待处理(21章描述外带数据，todo）

**一个套接字出错时，它由select标记为即可读又可写**
低潮限度的目的是select返回可读可写前，application可以对有多少数据可读或者多大空间可写进行控制，例如，为了防止小于64bytes的数据准备好select就唤醒我们，就将低潮限度设置为64个字节。

###重写str_cli函数
上节中，服务器主动关闭，套接字发生了些事情，客户端却阻塞在fgets上，新版本目标是让其阻塞在select上，或是等待标准输入，或是等待套接字可读。
三个条件，图，todo：
1.socket可读，read返回大于0的val==>即字节数
2.对方发一个FIN（进程关闭），套接字可读，返回一个0（即EOF）
3.对方发一个RST（主机重启），套接字可读，返回一个-1，errno有明确的错误码。
```
void str_cli(FILE *fp, int sockfd)
{
	int	maxfdp1;
	fd_set rset;
	char sendline[MAXLINE], recvline[MAXLINE];

	FD_ZERO(&rset);
	for ( ; ; )
	{
		//fp调用时设置为stdin
		FD_SET(fileno(fp), &rset);
		//新建的connect成功的fd
		FD_SET(sockfd, &rset);
		maxfdp1 = max(fileno(fp), sockfd) + 1;
		Select(maxfdp1, &rset, NULL, NULL, NULL);
		/* socket is readable */
		if (FD_ISSET(sockfd, &rset))
		{	
			if(Readline(sockfd, recvline, MAXLINE) == 0)
			err_quit("str_cli: server terminated prematurely");
			Fputs(recvline, stdout);
		}
		/* input is readable */
		if (FD_ISSET(fileno(fp), &rset)) 
		{  
		// bug，输入文件EOF并不意味着已经完成了socket的所有读
			if (Fgets(sendline, MAXLINE, fp) == NULL)
				return;		/* all done */
			Writen(sockfd, sendline, strlen(sendline));
		}
	}
}
```
上述代码有bug，如果fgets从输入得到EOF，立即返回main函数，于是socket被关闭，在server端此时会提示，
**readline error: Connection reset by peer**，我们需要一种告诉服务器一个FIN，我们已经完成数据发送，仍为读开放的套接字，考虑使用shutdown。
###使用shutdown函数
close函数是终止网络连接的正常方法，但它有几个限制：
>* close将访问计数-1，仅为0时我们才关闭套接字，详细见4.8节，shutdown是触发FIN分节
>* close同时终止了读和写两个方向，导致上节讨论的问题

图，shutdown关闭了一半的TCP连接，todo
三种关闭方式
>* SHUT_RD关闭读的这一半，不再接收套接字的数据而且，现留在buffer的数据都作废，不能执行任何读操作了
>* SHUT_WR关闭写的这一半，当前在buffer内的数据都被发送，但此后不该执行任何写相干操作。
```c
void str_cli(FILE *fp, int sockfd)
{
	int			maxfdp1, stdineof;
	fd_set		rset;
	char		sendline[MAXLINE],recvline[MAXLINE];
	stdineof = 0;
	FD_ZERO(&rset);
	for ( ; ; )
	{
		if (stdineof == 0)
			FD_SET(fileno(fp), &rset);
		FD_SET(sockfd, &rset);
		maxfdp1 = max(fileno(fp), sockfd) + 1;
		Select(maxfdp1, &rset, NULL, NULL, NULL);
		if (FD_ISSET(sockfd, &rset)) {	/* socket is readable */
			if (Readline(sockfd, recvline, MAXLINE) == 0) {
				if (stdineof == 1)
					return;		/* normal termination */
				else
					err_quit("str_cli: server terminated prematurely");
			}
			Fputs(recvline, stdout);
		}
		 /* input is readable */
		if (FD_ISSET(fileno(fp), &rset)) { 
			if (Fgets(sendline, MAXLINE, fp) == NULL) {
				stdineof = 1;
				// 发送一个FIN给服务器，表明我没有跟多数据要发送了
				Shutdown(sockfd, SHUT_WR);
				FD_CLR(fileno(fp), &rset);
				continue;
			}
			Writen(sockfd, sendline, strlen(sendline));
		}
	}
}
```
###改造单进程服务器（不用fork）
使用select来重写任意客户端的单进程服务端
```
int main(int argc, char **argv)
{
	int					i, maxi, maxfd, listenfd, connfd, sockfd;
	int					nready, client[FD_SETSIZE];
	ssize_t				n;
	fd_set				rset, allset;
	char				line[MAXLINE];
	socklen_t			clilen;
	struct sockaddr_in	cliaddr, servaddr;
	listenfd = Socket(AF_INET, SOCK_STREAM, 0);
	bzero(&servaddr, sizeof(servaddr));
	servaddr.sin_family      = AF_INET;
	servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
	servaddr.sin_port        = htons(SERV_PORT);
	Bind(listenfd, (SA *) &servaddr, sizeof(servaddr));
	Listen(listenfd, LISTENQ);
	maxfd = listenfd;			/* initialize */
	maxi = -1;					/* index into client[] array */
	for (i = 0; i < FD_SETSIZE; i++)
		client[i] = -1;			/* -1 indicates available entry */
	FD_ZERO(&allset);
	FD_SET(listenfd, &allset);
	for ( ; ; ) 
	{
		//新客户建立连接，或是数据，fin或者RST到达
		rset = allset;		/* structure assignment */
		nready = Select(maxfd+1, &rset, NULL, NULL, NULL);

		//如果是listenfd的读事件，accept之
		if (FD_ISSET(listenfd, &rset)) 
		{	/* new client connection */
			clilen = sizeof(cliaddr);
			connfd = Accept(listenfd, (SA *) &cliaddr, &clilen);
			//按顺序存各个FD
			for (i = 0; i < FD_SETSIZE; i++)
				if (client[i] < 0) {
					client[i] = connfd;	/* save descriptor */
					break;
				}
			if (i == FD_SETSIZE)
				err_quit("too many c ients");

			FD_SET(connfd, &allset);	/* add new descriptor to set */
			if (connfd > maxfd)
				maxfd = connfd;			/* for select */
			if (i > maxi)
				maxi = i;				/* max index in client[] array */

			//有其他fd
			if (--nready <= 0)
				continue;				/* no more readable descriptors */
		}
		//如果还有其他FD，全都试试
		for (i = 0; i <= maxi; i++)//
		{	/* check all clients for data */
			if ( (sockfd = client[i]) < 0)
				continue;
			if (FD_ISSET(sockfd, &rset)) 
			{
				if ( (n = Readline(sockfd, line, MAXLINE)) == 0) 
				{
						/*4connection closed by client */
					Close(sockfd);
					FD_CLR(sockfd, &allset);
					client[i] = -1;
				} else
				{
					Writen(sockfd, line, n);
				}

				if (--nready <= 0)
					break;				/* no more readable descriptors */
			}
		}
	}
}

```
###pselect
select原型:
```
#include<sys/time.h>
int select(int maxfdp1,fd_set* readset,fd_set* writeset,fd_set* exceptset,const struct timeval* timeout);
```
pselect原型：
```
#include<sys/select.h>
#include<signal.h>
#include<time.h>
int pselect(int maxfdp1,fd_set* readset,fd_set* writeset,fd_set* exceptset,const struct timespec* timeout,const sygset_t* sigmask);
```
新的timespec结构规定了纳秒数，老结构tv_usec则规定了微秒数。
```
#include<poll.h>
int poll(struct pollfd *fd,unsigned long nfds,int timeout);
struct pollfd{
int fd;
short events;//对fd要测试的条件
short revents;//fd的状态
};
```
用到的常量
```
既可作为输入，也可以作为输出
POLLIN
POLLRDNMRM
POLLRDBAND
POLLPRI
POLLOUT
POLLWRNORM
POLLWRBAND
以下不可用作events输入，之作为输出
POLLERR
POLLUP
POLLNVAL
```


 


> Written with [StackEdit](https://stackedit.io/).
> http://koo987.tumblr.com/post/112124265861/unp1-unp1
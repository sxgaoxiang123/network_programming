# select 系统调用

select通过文件集合的形式来管理网络套接字的读写

```c
#include <sys/select.h>
int select(int nfds, fd_set *readfds, fd_set *writefds, fd_set *exceptfds, struct timeval *timeout);
```

select方法可以让一个程序阻塞等待一个或多个文件描述符变为“ready”，等待I/O操作

三个相互独立的文件描述符集合将被同时监控

- nfds：三个集合中最大的文件描述符的值再加一

- readfds：监控是否有数据可读
- writefds：监控是否可以写入数据
- exceptfds：监控是否有异常
- timeout：设定阻塞等待时长（结构体定义在time中）
- 返回值：成功返回三个集合中包含的文件描述符个数，如果timeout前没有任何感兴趣的事件发生则返回0，失败返回-1并设置errno

文件集合的操作通过如下宏定义实现

```c
void FD_CLR(int fd, fd_set *set); //将对应的文件描述符从集合中删除
int FD_ISSET(int fd, fd_set *set); //判断集合中是否包含某个文件描述符
void FD_SET(int fd, fd_set *set); //将对应的文件描述符加入到集合中
void FD_ZERO(fd_set *set); //清空集合中的所有文件描述符
```



# select实现服务器

```c
#include <stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <sys/select.h>
#include <string.h>
#include <strings.h>
#include <netinet/in.h>
#include <stdlib.h>
#include <unistd.h>
#include <arpa/inet.h>
#include <ctype.h>

#define SERV_PORT 8000
#define MAXLINE 80

#define prrexit(msg) {\
    perror(msg);\
    exit(1);\
}

int main() {
    int i, listenfd, connfd, sockfd, maxfd, maxi, nready, n;
    int client[FD_SETSIZE]; //创建一个数组用于记录链接的客户端文件描述符
    struct sockaddr_in servaddr, cliaddr;
    socklen_t cliaddr_len;
    fd_set allset, rset;
    char str[INET_ADDRSTRLEN];
    char buf[MAXLINE];
    //生成套接字
    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    //设置服务器地址结构体
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(SERV_PORT);
    //绑定套接字文件描述符和服务器地址结构体
    bind(listenfd, (struct sockaddr *) &servaddr, sizeof(servaddr));
    //设置监听网络套接字
    listen(listenfd, 20);
    maxfd = listenfd;
    maxi = -1;

    //把客户端数组的初始状态设置为-1
    for (i = 0;i < FD_SETSIZE; ++i) {
        client[i] = -1;
    }
    //设置监听集合
    FD_ZERO(&allset);
    FD_SET(listenfd, &allset);
    
    while (1) {
        rset = allset;
        //监听是否有数据准备就绪可读
        nready = select(maxfd + 1, &rset, NULL, NULL, NULL);
        if (nready < 0) prrexit("select");
        //判断listenfd是否还在集合中/是否有客户端连接上来
        if (FD_ISSET(listenfd, &rset)) {
            //连接新的客户端 
            cliaddr_len = sizeof(cliaddr); //初始化客户端地址的长度
            connfd = accept(listenfd, (struct sockaddr *) &cliaddr, &cliaddr_len);
            printf("received from %s:%d\n", 
                   inet_ntop(AF_INET, &cliaddr.sin_addr, str, sizeof(str)),
                    ntohs(cliaddr.sin_port));

            //将connfd放入客户端数组中进行管理
            for (i = 0; i < FD_SETSIZE; ++i) {
                if (client[i] < 0) {
                    client[i] = connfd;
                    break;
                }
            }
            if (i == FD_SETSIZE) { //如果客户端数组已填满就只能等待
                fputs("服务器正忙，请等待\n",stderr);
                exit(1);
            }
            //将新连入的客户端对应套接字fd放入监听集合中
            FD_SET(connfd, &allset);
            maxfd = connfd > maxfd ? connfd : maxfd; 
            maxi = i > maxi ? i : maxi;
            //完成了listenfd的数据可读的操作处理
            if (--nready == 0) continue;
        }

        //如果是客户端的数据准备可读了
        for (i = 0; i <= maxi; ++i) {
            sockfd = client[i];
            if (sockfd < 0) continue;
            if (FD_ISSET(sockfd, &rset)) {
                //sockfd上有数据可读 
                n = read(sockfd, buf, MAXLINE);
                if (n == 0) { //对方已断开
                    close(sockfd);
                    FD_CLR(sockfd, &allset);
                    client[i] = -1;
                } else {
                    int j;
                    for (j = 0; j < n; ++j) {
                        buf[j] = toupper(buf[j]);
                    }
                    write(sockfd, buf, n);
                }
            }
            if (--nready == 0) break;
        }
    }
    return 0;
}
```


# 服务器端程序

```c
#include<stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/ip.h>
#include <strings.h>
#include <unistd.h>
#include <ctype.h>
#include <arpa/inet.h>
#include <stdlib.h>
#include <string.h>

#define SERV_PORT 8000
#define MAXLINE 80

#define prrexit(msg) {\
    perror(msg);\
    exit(1);\
}

int main() {

    int listenfd, connfd;
    struct sockaddr_in serveraddr, cliaddr;
    socklen_t cliaddr_len;
    char buf[MAXLINE];
    char str[INET_ADDRSTRLEN]; //用于存储IP地址字符串
    int n;
    
    //生成一个监听套接字描述符
    listenfd = socket(AF_INET, SOCK_STREAM, 0);
    if (listenfd < 0) prrexit("socket");

    //服务器ip地址及端口初始化
    bzero(&serveraddr, sizeof(serveraddr)); //清空内存区域
    serveraddr.sin_family = AF_INET; //IPv4协议组
    serveraddr.sin_port = htons(SERV_PORT); //端口，需要转换为网络字节序
    serveraddr.sin_addr.s_addr = htonl(INADDR_ANY); //指定任意IP，即所有IP都可以，需要转化为网络字节序

    //绑定监听套接字描述符到源IP的对应端口
    if (bind(listenfd, (struct sockaddr *)&serveraddr, sizeof(serveraddr)) < 0) prrexit("bind");

    //设置监听
    if (listen(listenfd, 2) < 0) prrexit("listen");
    printf("Accepting connections...\n");
    while (1) {
        cliaddr_len = sizeof(cliaddr);
        connfd = accept(listenfd, (struct sockaddr *) &cliaddr, &cliaddr_len);
        if (connfd < 0) prrexit("accept");
        printf("package received from %s:%d\n", \
               inet_ntop(AF_INET, &cliaddr.sin_addr, str, sizeof(str)), \
               ntohs(cliaddr.sin_port));
        while (1) {
            n = read(connfd, buf, MAXLINE);
            if (!strncmp(buf, "quit", 4)) break;
            write(1, buf, n);
            for (int i = 0; i < n; ++i) {
                buf[i] = toupper(buf[i]);
            }
            write(connfd, buf, n);
        }
        //while (1) {
            //sleep(1);
        //}
        close(connfd);
    }
    return 0;
}
```



# 客户端程序

```c
#include<stdio.h>
#include <sys/types.h>
#include <sys/socket.h>
#include <netinet/in.h>
#include <netinet/ip.h>
#include <strings.h>
#include <string.h>
#include <unistd.h>
#include <ctype.h>
#include <arpa/inet.h>

#define SERV_PORT 8000
#define MAXLINE 80

int main() {
    int sockfd = socket(AF_INET, SOCK_STREAM, 0);
    int n;
    struct sockaddr_in serveraddr;
    char buf[MAXLINE] = {"hello tcp"};

    //服务器ip地址及端口初始化
    bzero(&serveraddr, sizeof(serveraddr)); //清空内存区域
    serveraddr.sin_family = AF_INET; //IPv4协议组
    serveraddr.sin_port = htons(SERV_PORT); //端口，需要转换为网络字节序
    inet_pton(AF_INET, "127.0.0.1", &serveraddr.sin_addr);
    connect(sockfd,(struct sockaddr *) &serveraddr, sizeof(serveraddr));

    while (n = read(0, buf, MAXLINE)) {
        if (!strncmp(buf, "quit", 4)) break;
        write(sockfd,buf, n);
        n = read(sockfd, buf, MAXLINE);
        printf("response from server:\n");
        write(1, buf, n);
    }
    //while (1) {
    //    sleep(1);
    //}
    close(sockfd);
    return 0;
}
```


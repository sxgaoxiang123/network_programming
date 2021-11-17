# 基于UDP协议的网络程序

## UDP客户端与服务器通讯过程

![image-20211117213307080](/home/gaoxiang/.config/Typora/typora-user-images/image-20211117213307080.png)

由于UDP不需要维护连接，程序逻辑简单很多，但UDP协议是不可靠的，实际上有很多保证通讯可靠性的机制需要在应用层实现



## 代码实现

### 服务器代码

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
#include <stdlib.h>
#include <sys/wait.h>
#include <pthread.h>

#define SERV_PORT 8000
#define MAXLINE 80

#define prrexit(msg) {\
    perror(msg);\
    exit(1);\
}

int main() {
    int sockfd;
    int n;
    socklen_t cliaddr_len;
    struct sockaddr_in servaddr, cliaddr;
    char buf[MAXLINE];
    char str[INET_ADDRSTRLEN];
    sockfd = socket(AF_INET, SOCK_DGRAM, 0);

    //初始化服务器套接字结构体
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_addr.s_addr = htonl(INADDR_ANY);
    servaddr.sin_port = htons(SERV_PORT);

    bind(sockfd, (struct sockaddr *) &servaddr, sizeof(servaddr));
    printf("UDP server ready:\n");

    while (1) {
        //服务器接收客户端的访问 
        n = recvfrom(sockfd, buf, MAXLINE, 0, (struct sockaddr *) &cliaddr, &cliaddr_len);
        if (n < 0) prrexit("recvfrom");
        printf("received from %s:%d\n",
               inet_ntop(AF_INET, &cliaddr.sin_addr, str, sizeof(str)),
               ntohs(cliaddr.sin_port));
        for (int i = 0; i < n; ++i) {
            buf[i] = toupper(buf[i]);
        }
        sendto(sockfd, buf, n, 0, (struct sockaddr *) &cliaddr, sizeof(cliaddr));
        printf("transfer finished\n");
    }

    close (sockfd);

    return 0;
}
```

### 客户端代码

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
#include <stdlib.h>
#include <sys/wait.h>
#include <pthread.h>

#define SERV_PORT 8000
#define MAXLINE 80

#define prrexit(msg) {\
    perror(msg);\
    exit(1);\
}

int main() {
    struct sockaddr_in servaddr;
    int sockfd;
    int n;
    char buf[MAXLINE];
    sockfd = socket(AF_INET, SOCK_DGRAM, 0);

    //初始化服务器套接字结构体
    bzero(&servaddr, sizeof(servaddr));
    servaddr.sin_family = AF_INET;
    servaddr.sin_port = htons(SERV_PORT);
    inet_pton(AF_INET, "127.0.0.1", &servaddr.sin_addr);

    while (n = read(0, buf, MAXLINE)) {
        //客户端向服务器发送数据
        sendto(sockfd, buf, n, 0, (struct sockaddr *) &servaddr, sizeof(servaddr)); 
        //读取服务器返回的处理结果
        n = recvfrom(sockfd, buf, MAXLINE, 0, NULL, 0);
        if (n <= 0) prrexit("recvfrom"); 
        write(1, buf, n);
    }
    close(sockfd);
    return 0;
}
```



## UDP传输特点

UDP协议与TCP不同，服务器和客户端不是建立在连接的基础之上的

- 单独断开客户段，再打开客户端并向服务器发送请求，服务器仍然可以应答
- 单独断开服务器，再打开服务器，并用处于打开状态的客户端向服务器发送请求，服务器也可以应答


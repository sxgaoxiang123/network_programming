# 线程池

线程池是基于一个任务队列实现的

1. 服务器构建任务池队列
2. 服务器构建线程池
3. 服务器建立TCP连接，并监听客户端请求
4. 当客户端发起请求时，服务器做如下操作
   1. 先将客户端任务入队
   2. 子线程循环从任务队列中提取任务
   3. 子线程循环读取客户端发送的数据



## 服务器代码

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
#include <sys/wait.h>
#include <pthread.h>

#define SERV_PORT 8000
#define MAXLINE 80

#define prrexit(msg) {\
    perror(msg);\
    exit(1);\
}

//构建任务队列
typedef struct Task {
    int fd;
    struct Task *next;
} Task;

typedef struct TaskPool {
    Task *head;
    Task *tail;
    pthread_mutex_t lock; //多线程锁
    pthread_cond_t havetask; //多线程条件
} TaskPool;

//初始化任务队列
TaskPool *task_pool_init() {
    TaskPool *tp = (TaskPool *) malloc(sizeof(TaskPool));
    tp -> head = NULL;
    tp -> tail = NULL;
    pthread_mutex_init(&(tp -> lock), NULL);
    pthread_cond_init(&(tp -> havetask), NULL);
    return tp;
}

//队列push（尾插法）
void task_pool_push(TaskPool *tp, int fd) {
    pthread_mutex_lock(&tp -> lock);
    Task *t = (Task *) malloc(sizeof(Task)); 
    t -> fd = fd;
    t -> next = NULL;
    if (!tp -> tail) {
        tp -> head = tp -> tail = t;
    } else {
        tp -> tail -> next = t;
        tp -> tail = t;
    }
    pthread_cond_broadcast(&tp -> havetask);
    pthread_mutex_unlock(&tp -> lock);
}

//队列pop（头删法）
Task task_pool_pop(TaskPool *tp) {
    pthread_mutex_lock(&tp -> lock);
    //队列中有节点才能pop
    while (!tp -> head) {
        pthread_cond_wait(&tp -> havetask, &tp -> lock);
    }

    Task tmp, *k;
    k = tp -> head;
    tmp = *k;
    tp -> head = tp -> head -> next;
    if (!(tp -> head)) tp -> tail = NULL;
    free(k);
    pthread_mutex_unlock(&tp -> lock);
    return tmp;
}

//释放任务队列
void task_pool_free(TaskPool *tp) {
    pthread_mutex_lock(&tp -> lock);
    Task *p, *k;
    p = tp -> head;
    while (p) {
        k = p;
        p = p -> next;
        free(k);
    }
    pthread_mutex_unlock(&tp -> lock);
    pthread_cond_destroy(&tp -> havetask);
    pthread_mutex_destroy(&tp -> lock);
    free(tp);
}

//子线程操作函数
void *up_server(void *arg) {
    pthread_detach(pthread_self()); //将子线程切换到detach状态，当线程结束时自动归还资源
    char buf[MAXLINE];
    int n;
    TaskPool *tp = arg;

    while (1) { //循环获取任务
        Task tmp = task_pool_pop(tp);
        int connfd = tmp.fd;
        printf("get task fd = %d\n", connfd);
        while (1) { //循环读入客户端输入
            n = read(connfd, buf, MAXLINE);
            if (!strncmp(buf, "quit", 4)) break;
            write(1, buf, n);
            for (int i = 0; i < n; ++i) {
                buf[i] = toupper(buf[i]);
            }
            write(connfd, buf, n);
        }
        printf("task %d finished\n", connfd);
        close(connfd);
    }

    return (void *) 0;
}

int main() {

    int listenfd, connfd;
    struct sockaddr_in serveraddr, cliaddr;
    socklen_t cliaddr_len;
    char str[INET_ADDRSTRLEN]; //用于存储IP地址字符串
    int n;

    //初始化线程池队列
    TaskPool *tp = task_pool_init();

    //创建多线程,等待任务
    pthread_t tid;
    for (int i = 0; i < 4; ++i) {
        pthread_create(&tid, NULL, up_server, (void *) tp);
        printf("new thread is %#lx\n", tid);
    }

    //主线程工作 
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
        task_pool_push(tp, connfd); //将客户端请求入队
    }

    task_pool_free(tp);

    return 0;
}
```



## 客户端代码

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
        write(sockfd,buf, n);
        if (!strncmp(buf, "quit", 4)) break;
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


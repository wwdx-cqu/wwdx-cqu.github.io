---
title: 2026-1-18-io_uring性能测试与epoll的对比
date: 2026-1-18 14:20 +0800
authors: stonly
categories: [任务, 0voice]
tags: [learning]    
---

[TOC]
# io_uring的测试工具test_qps_tcpclient

通过编写命令行测试工具来对该io_uring服务器进行测试。要支持类似这样的效果：
```shell
./test_qps_tcp_client -s 127.0.0.1 -p 2048 -t 50 -c 100 -n 10000
```
-s：服务器地址
-p：端口号
-t：线程数量
-c：连接数量
-n：请求数量
## test_context_t 结构体
将这些信息存在一个结构体中
```cpp
typedef struct test_context_s{
    char serverip[16];
    int port;
    int threadnum;
    int connection;
    int requestion;
#if 1
    int failed;
#endif
}test_context_t;
```
## main()函数
```cpp
    int ret = 0;
    test_context_t ctx = {0};
    int opt;
    /* getopt函数：
    	"a"		支持 -a，无参数
		"b:"	支持 -b 参数
		"abc"	支持 -a -b -c
		"a:b:c"	-a arg -b arg -c arg
	*/
    while( (opt = getopt(argc,argv,"s:p:t:c:n:?")) != -1){
        switch(opt){
            case 's':
                printf("-s: %s\n",optarg);
                strcpy(ctx.serverip, optarg);
                break;
            case 'p':
                printf("-p: %s\n",optarg);
                ctx.port = atoi(optarg);
                break;
            case 't':
                printf("-t: %s\n",optarg);
                ctx.threadnum = atoi(optarg);
                break;
            case 'c':
                printf("-c: %s\n",optarg);
                ctx.connection = atoi(optarg);
                break;
            case 'n':
                printf("-n: %s\n",optarg);    
                ctx.requestion = atoi(optarg);
                break;
            default:
                return -1;
        }
    }
    //开threadnum个线程
    pthread_t *ptid = malloc(sizeof(pthread_t)*ctx.threadnum);
    int i = 0;
    struct timeval tv_begin; 
    gettimeofday(&tv_begin, NULL);
    for(i = 0; i < ctx.threadnum; i++){
        pthread_create(&ptid[i], NULL, test_qps_entry, &ctx);
    }
    for(i = 0; i < ctx.threadnum; i++){
        pthread_join(ptid[i], NULL);
    }
    struct timeval tv_end;
    gettimeofday(&tv_end, NULL);

    int time_used = TIME_SUB_MS(tv_end, tv_begin);
     
    printf("success: %d, failed: %d, time_used: %d\n", ctx.requestion - ctx.failed, ctx.failed, time_used);
clean:
    free(ptid);
    return 0;
```
## test_qps_entry() 函数
```cpp
static void *test_qps_entry(void *arg){
    test_context_t *pctx = (test_context_t *)arg;
    // 连接对应的tcp服务器。
    int connfd = connect_tcpserver(pctx->serverip, pctx->port);
    if(connfd < 0){
        printf("connect_tcpserver failed\n");
        return NULL;
    }
    int count = pctx->requestion / pctx->threadnum;
    int i = 0;
    int res;
    while(i++ < count){
    	//发送与接收数据时的数据包
        res = send_recv_tcppkt(connfd);
        if(res != 0){
            printf("send_recv_tcppkt failed!");
            pctx->failed++;
            continue;
        }
    }
    return NULL;
}
```
## connect_tcpserver() 函数（经典）
```cpp
int connect_tcpserver(const char *ip, int port){
    int connfd = socket(AF_INET, SOCK_STREAM, 0);

    struct sockaddr_in tcpserver_addr;
    memset(&tcpserver_addr, 0, sizeof (struct sockaddr_in));

    tcpserver_addr.sin_family = AF_INET;
    tcpserver_addr.sin_addr.s_addr = inet_addr(ip);
    tcpserver_addr.sin_port = htons(port);

    int ret = connect(connfd, (struct sockaddr*)&tcpserver_addr, sizeof(struct sockaddr_in));
    if(ret){
        perror("connect\n");
        return -1;
    }
    return connfd; 
}
```
## send_recv_tcppkt()函数
```cpp
#define TEST_MESSAGE "ABCDEFGHIJKLMNOPQRSTUVWXYZ1234567890abcdefghijklmnopqrstuvwxyz\r\n"
int send_recv_tcppkt(int fd){
    char wbuffer[WBUFFER_LENGTH] = {0};
    int i = 0;
    //这里当tcp数据包过大时，大于协议栈的MTU时需要重新封装
    for(i = 0; i < 2; i++){
        strcpy(wbuffer + i*strlen(TEST_MESSAGE), TEST_MESSAGE);
    }
    int res = send(fd, wbuffer, strlen(wbuffer),0);
    if(res < 0){
        exit(1);
    }
    char rbuffer[RBUFFER_LENGTH] = {0};
    res = recv(fd, rbuffer, wbuffer, 0);
    if(res <= 0){
        exit(1);
    }
    if(strcmp(rbuffer,wbuffer) != 0){
        printf("failed: '%s' != '%s'\n", rbuffer, wbuffer);
        return -1;
    }
 }
```
## 测试结果
uring_tcp_server的结果。
![uring](/assets/img/2026-1-18-1.png)
epoll_tcp_server的结果。
![epoll](/assets/img/2026-1-18-2.png)
由于一次测试需150秒，故仅测了这一组数据，后续可自行进行测试。
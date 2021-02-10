# 【源码剖析】Webbench —— 简洁而优美的压力测试工具

Webbench 是一个古老而著名的网站压力测试工具，简单而实用。如果你不清楚你的网站能承受多大的压力，或者你想分析对比两个网站的性能，webbench 再好用不过了。

### 项目地址
Gitbub 地址：[点我](https://github.com/AngryHacker/code-with-comments/tree/master/webbench)

### 安装

很简单，cd 进项目主页后进行 make install clean 就好了。

### 用法
想要知道用法可以在安装后直接输入 webbench 或 webbench -h 或 webbench --help. 可以看到：

```
webbench [option]... URL
  -f|--force               Don't wait for reply from server.
  -r|--reload              Send reload request - Pragma: no-cache.
  -t|--time <sec>          Run benchmark for <sec> seconds. Default 30.
  -p|--proxy <server:port> Use proxy server for request.
  -c|--clients <n>         Run <n> HTTP clients at once. Default one.
  -9|--http09              Use HTTP/0.9 style requests.
  -1|--http10              Use HTTP/1.0 protocol.
  -2|--http11              Use HTTP/1.1 protocol.
  --get                    Use GET request method.
  --head                   Use HEAD request method.
  --options                Use OPTIONS request method.
  --trace                  Use TRACE request method.
  -?|-h|--help             This information.
  -V|--version             Display program version.
  ```

 说一下主要的几个选项： 指定 -f 时不等待服务器数据返回，  -t 为指定压力测试运行时间， -c 指定由多少个客户端发起测试请求。

-9 -1 -2 分别为指定 HTTP/0.9 HTTP/1.0 HTTP/1.1。

### 示例
访问 http://baidu.com/ , 一个客户端，持续 60 秒。

```
➜  webbench-1.5  webbench -t 60 http://www.baidu.com/
Webbench - Simple Web Benchmark 1.5
Copyright (c) Radim Kolar 1997-2004, GPL Open Source Software.

Benchmarking: GET http://www.baidu.com/
1 client, running 60 sec.

Speed=36 pages/min, 54150 bytes/sec.
Requests: 36 susceed, 0 failed.
```

访问 http://baidu.com/ , 10 个客户端，持续 60 秒，socket 提前关闭。

```
➜  webbench-1.5  webbench -f -c 10 -t 60 http://www.baidu.com/
Webbench - Simple Web Benchmark 1.5
Copyright (c) Radim Kolar 1997-2004, GPL Open Source Software.

Benchmarking: GET http://www.baidu.com/
10 clients, running 60 sec, early socket close.

Speed=586 pages/min, 0 bytes/sec.
Requests: 586 susceed, 0 failed.
```

### 代码分析

其实我把项目 clone 之后做的第一件事是把代码风格全部调整了....强迫症面对不一样的缩进和花括号都不能忍...

项目源码都位于 socket.c 和 webbench.c 两个 c 文件里面。其他都是说明文档或用于编译。

可能涉及知识点：命令行参数解析（getopt_long） 信号处理（sigaction） socket  管道读写。这些都可以通过 man 相关函数或者直接百度，不赘述了。

#### 主要函数

* alarm_handler 信号处理函数，时钟结束时进行调用。
* usage 输出 webbench 命令用法
* main 提供程序入口...
* build_request 构造 HTTP 请求
* bench 派生子进程，父子进程管道通信最后输出计算结果。
* benchcore 每个子进程的实际发起请求函数。

#### 主要流程
说一下程序执行的主要流程：

1. 解析命令行参数，根据命令行指定参数设定变量，可以认为是初始化配置。
2. 根据指定的配置构造 HTTP 请求报文格式。
3. 开始执行 bench 函数，先进行一次 socket 连接建立与断开，测试是否可以正常访问。
4. 建立管道，派生根据指定进程数派生子进程。
5. 每个子进程调用 benchcore 函数，先通过 sigaction 安装信号，用 alarm 设置闹钟函数，接着不断建立 socket 进行通信，与服务器交互数据，直到收到信号结束访问测试。子进程将访问测试结果写进管道。
6. 父进程读取管道数据，汇总子进程信息，收到所有子进程消息后，输出汇总信息，结束。

#### 流程图
![webbench](https://raw.githubusercontent.com/AngryHacker/articles/master/img/webbench.png)                                              

#### 注释源码

##### socket.c

```C

/* $Id: socket.c 1.1 1995/01/01 07:11:14 cthuang Exp $
 *
 * This module has been modified by Radim Kolar for OS/2 emx
 */

/***********************************************************************
  module:       socket.c
  program:      popclient
  SCCS ID:      @(#)socket.c    1.5  4/1/94
  programmer:   Virginia Tech Computing Center
  compiler:     DEC RISC C compiler (Ultrix 4.1)
  environment:  DEC Ultrix 4.3
  description:  UNIX sockets code.
 ***********************************************************************/

#include <sys/types.h>
#include <sys/socket.h>
#include <fcntl.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <netdb.h>
#include <sys/time.h>
#include <string.h>
#include <unistd.h>
#include <stdio.h>
#include <stdlib.h>
#include <stdarg.h>

int Socket(const char *host, int clientPort)
{
    int sock;
    unsigned long inaddr;
    struct sockaddr_in ad;
    struct hostent *hp;

    /* 初始化地址 */
    memset(&ad, 0, sizeof(ad));
    ad.sin_family = AF_INET;

    /* 尝试把主机名转化为数字 */
    inaddr = inet_addr(host);
    if (inaddr != INADDR_NONE)
        memcpy(&ad.sin_addr, &inaddr, sizeof(inaddr));
    else
    {
        /* 取得 ip 地址 */
        hp = gethostbyname(host);
        if (hp == NULL)
            return -1;
        memcpy(&ad.sin_addr, hp->h_addr, hp->h_length);
    }
    ad.sin_port = htons(clientPort);

    /* 建立 socket */
    sock = socket(AF_INET, SOCK_STREAM, 0);
    if (sock < 0)
        return sock;
    /* 建立链接 */
    if (connect(sock, (struct sockaddr *)&ad, sizeof(ad)) < 0)
        return -1;
    return sock;
}
```

##### webbench.c

```C
/*
 * (C) Radim Kolar 1997-2004
 * This is free software, see GNU Public License version 2 for
 * details.
 *
 * Simple forking WWW Server benchmark:
 *
 * Usage:
 *   webbench --help
 *
 * Return codes:
 *    0 - sucess
 *    1 - benchmark failed (server is not on-line)
 *    2 - bad param
 *    3 - internal error, fork failed
 *
 */
#include "socket.c"
#include <unistd.h>
#include <sys/param.h>
#include <rpc/types.h>
#include <getopt.h>
#include <strings.h>
#include <time.h>
#include <signal.h>

/* values */
volatile int timerexpired = 0;
int speed = 0;
int failed = 0;
int bytes = 0;

/* globals */
int http10 = 1; /* 0 - http/0.9, 1 - http/1.0, 2 - http/1.1 */

/* Allow: GET, HEAD, OPTIONS, TRACE */
#define METHOD_GET 0
#define METHOD_HEAD 1
#define METHOD_OPTIONS 2
#define METHOD_TRACE 3
#define PROGRAM_VERSION "1.5"

/* 默认设置 */
int method = METHOD_GET; /* GET 方式 */
int clients = 1; /* 只模拟一个客户端 */
int force = 0; /* 等待响应 */
int force_reload = 0; /* 失败时重新请求 */
int proxyport = 80; /* 访问端口 */
char *proxyhost = NULL; /* 代理服务器 */
int benchtime = 30; /* 模拟请求时间 */

/* internal */
int mypipe[2]; /* 管道 */
char host[MAXHOSTNAMELEN]; /* 网络地址 */
#define REQUEST_SIZE 2048
char request[REQUEST_SIZE]; /* 请求 */

static const struct option long_options[]=
{
    {"force",no_argument,&force,1},
    {"reload",no_argument,&force_reload,1},
    {"time",required_argument,NULL,'t'},
    {"help",no_argument,NULL,'?'},
    {"http09",no_argument,NULL,'9'},
    {"http10",no_argument,NULL,'1'},
    {"http11",no_argument,NULL,'2'},
    {"get",no_argument,&method,METHOD_GET},
    {"head",no_argument,&method,METHOD_HEAD},
    {"options",no_argument,&method,METHOD_OPTIONS},
    {"trace",no_argument,&method,METHOD_TRACE},
    {"version",no_argument,NULL,'V'},
    {"proxy",required_argument,NULL,'p'},
    {"clients",required_argument,NULL,'c'},
    {NULL,0,NULL,0}
};

/* prototypes */
static void benchcore(const char* host,const int port, const char *request);
static int bench(void);
static void build_request(const char *url);

static void alarm_handler(int signal)
{
    timerexpired = 1;
}

/* help 信息 */
static void usage(void)
{
   fprintf(stderr,
	"webbench [option]... URL\n"
	"  -f|--force               Don't wait for reply from server.\n"
	"  -r|--reload              Send reload request - Pragma: no-cache.\n"
	"  -t|--time <sec>          Run benchmark for <sec> seconds. Default 30.\n"
	"  -p|--proxy <server:port> Use proxy server for request.\n"
	"  -c|--clients <n>         Run <n> HTTP clients at once. Default one.\n"
	"  -9|--http09              Use HTTP/0.9 style requests.\n"
	"  -1|--http10              Use HTTP/1.0 protocol.\n"
	"  -2|--http11              Use HTTP/1.1 protocol.\n"
	"  --get                    Use GET request method.\n"
	"  --head                   Use HEAD request method.\n"
	"  --options                Use OPTIONS request method.\n"
	"  --trace                  Use TRACE request method.\n"
	"  -?|-h|--help             This information.\n"
	"  -V|--version             Display program version.\n"
	);
};

int main(int argc, char *argv[])
{
    int opt = 0;
    int options_index = 0;
    char *tmp = NULL;

    /* 不带参数时直接输出 help 信息 */
    if(argc == 1)
    {
        usage();
        return 2;
    }

    /* getopt_long 为命令行解析的库函数，可通过 man 3 getopt_long 查看 */
    while((opt = getopt_long(argc,argv,"912Vfrt:p:c:?h",long_options,&options_index)) != EOF )
    {
        /* 如果有返回对应的命令行参数 */
        switch(opt)
        {
            case  0 : break;
            case 'f': force = 1;break;
            case 'r': force_reload = 1;break;
            case '9': http10 = 0;break;
            case '1': http10 = 1;break;
            case '2': http10 = 2;break;
            case 'V':
                      printf(PROGRAM_VERSION"\n");
                      exit(0);
            case 't':
                      benchtime = atoi(optarg);
                      break;
            case 'p':
                      /* proxy server parsing server:port */
                      tmp = strrchr(optarg,':');
                      proxyhost = optarg;
                      if(tmp == NULL)
                      {
                          break;
                      }
                      if(tmp == optarg)
                      {
                          fprintf(stderr,"Error in option --proxy %s: Missing hostname.\n",optarg);
                          return 2;
                      }
                      if(tmp == optarg + strlen(optarg)-1)
                      {
                          fprintf(stderr,"Error in option --proxy %s Port number is missing.\n",optarg);
                          return 2;
                      }
                      *tmp = '\0';
                      proxyport = atoi(tmp+1);
                      break;
            case ':':
            case 'h':
            case '?': usage();return 2;break;
            case 'c': clients = atoi(optarg);break;
        }
    }

    /* optind 被 getopt_long 设置为命令行参数中未读取的下一个元素下标值 */
    if(optind == argc)
    {
        fprintf(stderr,"webbench: Missing URL!\n");
        usage();
        return 2;
    }

    /* 不能指定客户端数和请求时间为 0 */
    if(clients == 0) clients = 1;
    if(benchtime == 0) benchtime = 60;

    /* Copyright */
    fprintf(stderr,"Webbench - Simple Web Benchmark "PROGRAM_VERSION"\n"
	 "Copyright (c) Radim Kolar 1997-2004, GPL Open Source Software.\n"
     );

    /* 构造 HTTP 请求到 request 数组 */
    build_request(argv[optind]);

    /* 以下到函数结束为输出提示信息 */
    /* print bench info */
    printf("\nBenchmarking: ");
    switch(method)
    {
        case METHOD_GET:
        default:
            printf("GET");break;
        case METHOD_OPTIONS:
            printf("OPTIONS");break;
        case METHOD_HEAD:
            printf("HEAD");break;
        case METHOD_TRACE:
            printf("TRACE");break;
    }

    printf(" %s",argv[optind]);
    switch(http10)
    {
        case 0: printf(" (using HTTP/0.9)");break;
        case 2: printf(" (using HTTP/1.1)");break;
    }

    printf("\n");

    if(clients == 1) printf("1 client");
    else
        printf("%d clients",clients);

    printf(", running %d sec", benchtime);
    if(force) printf(", early socket close");
    if(proxyhost != NULL) printf(", via proxy server %s:%d",proxyhost,proxyport);
    if(force_reload) printf(", forcing reload");
    printf(".\n");

    /* 开始压力测试，返回 bench 函数执行结果 */
    return bench();
}

void build_request(const char *url)
{
    char tmp[10];
    int i;

    /* 初始化 */
    bzero(host,MAXHOSTNAMELEN);
    bzero(request,REQUEST_SIZE);

    /* 判断应该使用的 HTTP 协议 */
    if(force_reload && proxyhost != NULL && http10 < 1) http10 = 1;
    if(method == METHOD_HEAD && http10 < 1) http10 = 1;
    if(method == METHOD_OPTIONS && http10 < 2) http10 = 2;
    if(method == METHOD_TRACE && http10 < 2) http10 = 2;

    /*填写 method 方法 */
    switch(method)
    {
        default:
        case METHOD_GET: strcpy(request,"GET");break;
        case METHOD_HEAD: strcpy(request,"HEAD");break;
        case METHOD_OPTIONS: strcpy(request,"OPTIONS");break;
        case METHOD_TRACE: strcpy(request,"TRACE");break;
    }

    strcat(request," ");

    /* URL 合法性判断 */
    if(NULL == strstr(url,"://"))
    {
        fprintf(stderr, "\n%s: is not a valid URL.\n",url);
        exit(2);
    }

    if(strlen(url)>1500)
    {
        fprintf(stderr,"URL is too long.\n");
        exit(2);
    }

    if(proxyhost == NULL)
        if(0 != strncasecmp("http://",url,7))
        {
            /* 只支持 HTTP 地址 */
            fprintf(stderr,"\nOnly HTTP protocol is directly supported, set --proxy for others.\n");
            exit(2);
        }

    /* 找到主机名开始的地方 */
    /* protocol/host delimiter */
    i = strstr(url,"://")-url+3;

    /* 必须以 / 结束*/
    /* printf("%d\n",i); */
    if(strchr(url+i,'/')==NULL)
    {
        fprintf(stderr,"\nInvalid URL syntax - hostname don't ends with '/'.\n");
        exit(2);
    }

    if(proxyhost == NULL)
    {
        /* get port from hostname */
        if(index(url+i,':') != NULL && index(url+i,':') < index(url+i,'/'))
        {
            strncpy(host,url+i,strchr(url+i,':')-url-i);
            /* 端口 */
            bzero(tmp,10);
            strncpy(tmp,index(url+i,':')+1,strchr(url+i,'/')-index(url+i,':')-1);
            /* printf("tmp=%s\n",tmp); */
            /* 设置端口 */
            proxyport = atoi(tmp);
            if(proxyport==0) proxyport=80;

        } else {
            strncpy(host,url+i,strcspn(url+i,"/"));
        }
        // printf("Host=%s\n",host);

        strcat(request+strlen(request),url+i+strcspn(url+i,"/"));
    } else {
        // printf("ProxyHost=%s\nProxyPort=%d\n",proxyhost,proxyport);
        strcat(request,url);
    }

    if(http10 == 1)
        strcat(request," HTTP/1.0");
    else if (http10==2)
        strcat(request," HTTP/1.1");
    strcat(request,"\r\n");

    if(http10 > 0)
        strcat(request,"User-Agent: WebBench "PROGRAM_VERSION"\r\n");

    if(proxyhost == NULL && http10 > 0)
    {
        strcat(request,"Host: ");
        strcat(request,host);
        strcat(request,"\r\n");
    }

    if(force_reload && proxyhost != NULL)
    {
        strcat(request,"Pragma: no-cache\r\n");
    }

    if(http10 > 1)
        strcat(request,"Connection: close\r\n");

    /* add empty line at end */
    if(http10>0) strcat(request,"\r\n");
    // printf("Req=%s\n",request);
}

/* vraci system rc error kod */
static int bench(void)
{
    int i,j,k;
    pid_t pid = 0;
    FILE *f;

    /* 作为测试地址是否合法 */
    /* check avaibility of target server */
    i = Socket(proxyhost == NULL?host:proxyhost, proxyport);
    if(i < 0){
        fprintf(stderr,"\nConnect to server failed. Aborting benchmark.\n");
        return 1;
    }
    close(i);

    /* 建立管道 */
    /* create pipe */
    if(pipe(mypipe))
    {
        perror("pipe failed.");
        return 3;
    }

    /* not needed, since we have alarm() in childrens */
    /* wait 4 next system clock tick */
    /*
     * cas=time(NULL);
     * while(time(NULL)==cas)
     * sched_yield();
     * */

    /* 派生子进程 */
    /* fork childs */
    for(i = 0;i < clients;i++)
    {
        pid = fork();
        if(pid <= (pid_t)0)
        {
            /* child process or error*/
            sleep(1); /* make childs faster */
            break; /* 子进程立刻跳出循环，要不就子进程继续 fork 了 */
        }
    }

    if( pid < (pid_t)0)
    {
        fprintf(stderr,"problems forking worker no. %d\n",i);
        perror("fork failed.");
        return 3;
    }

    if(pid == (pid_t)0)
    {
        /* 子进程发出实际请求 */
        /* I am a child */
        if(proxyhost == NULL)
            benchcore(host,proxyport,request);
        else
            benchcore(proxyhost,proxyport,request);

        /* 打开管道写 */
        /* write results to pipe */
        f = fdopen(mypipe[1],"w");
        if(f == NULL)
        {
            perror("open pipe for writing failed.");
            return 3;
        }

        /* fprintf(stderr,"Child - %d %d\n",speed,failed); */
        fprintf(f,"%d %d %d\n",speed,failed,bytes);
        fclose(f);

        return 0;

    } else {
        /* 父进程打开管道读 */
        f = fdopen(mypipe[0],"r");
        if(f == NULL)
        {
            perror("open pipe for reading failed.");
            return 3;
        }

        setvbuf(f,NULL,_IONBF,0);
        speed = 0; /* 传输速度 */
        failed = 0; /* 失败请求数 */
        bytes = 0; /* 传输字节数 */

        while(1)
        {
            pid = fscanf(f,"%d %d %d",&i,&j,&k);
            if(pid<2)
            {
                fprintf(stderr,"Some of our childrens died.\n");
                break;
            }
            speed += i;
            failed += j;
            bytes += k;
            /* fprintf(stderr,"*Knock* %d %d read=%d\n",speed,failed,pid); */
            /* 子进程是否读取完 */
            if(--clients == 0) break;
        }
        fclose(f);

        /* 结果计算 */
        printf("\nSpeed=%d pages/min, %d bytes/sec.\nRequests: %d susceed, %d failed.\n",
                (int)((speed+failed)/(benchtime/60.0f)),
                (int)(bytes/(float)benchtime),
                speed,
                failed);
    }
    return i;
}

void benchcore(const char *host,const int port,const char *req)
{
    int rlen;
    char buf[1500];
    int s,i;
    struct sigaction sa;

    /*安装信号 */
    /* setup alarm signal handler */
    sa.sa_handler = alarm_handler;
    sa.sa_flags = 0;
    if(sigaction(SIGALRM,&sa,NULL))
        exit(3);

    /* 设置闹钟函数 */
    alarm(benchtime);

    rlen = strlen(req);

nexttry:
    while(1){
        /* 收到信号则 timerexpired = 1 */
        if(timerexpired)
        {
            if(failed > 0)
            {
                /* fprintf(stderr,"Correcting failed by signal\n"); */
                failed--;
            }
            return;
        }
        /* 建立 socket, 进行 HTTP 请求 */
        s = Socket(host,port);
        if(s < 0)
        {
            failed++;
            continue;
        }
        if(rlen!=write(s,req,rlen))
        {
            failed++;
            close(s);
            continue;
        }
        /* HTTP 0.9 的处理 */
        if(http10==0)
            /* 如果关闭不成功 */
            if(shutdown(s,1))
            {
                failed++;
                close(s);
                continue;
            }

        /* -f 选项时不读取服务器回复 */
        if(force == 0)
        {
            /* read all available data from socket */
            while(1)
            {
                if(timerexpired) break;
                i = read(s,buf,1500);
                /* fprintf(stderr,"%d\n",i); */
                if(i<0)
                {
                    failed++;
                    close(s);
                    goto nexttry;
                }
                else
                    if(i == 0) break;
                    else bytes+=i;
            }
        }
        if(close(s))
        {
            failed++;
            continue;
        }
        speed++;
    }
}
```

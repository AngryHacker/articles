# 【源码剖析】tinyhttpd —— C 语言实现最简单的 HTTP 服务器

tinyhttpd 是一个不到 500 行的超轻量型 Http Server，用来学习非常不错，可以帮助我们真正理解服务器程序的本质。

看完所有源码，真的感觉有很大收获，无论是 unix 的编程，还是 GET/POST 的 Web 处理流程，都清晰了不少。废话不说，开始我们的 Server 探索之旅。

## 项目主页
http://sourceforge.net/projects/tinyhttpd/

## 主要函数
这是所有函数的声明：

```c
void accept_request(int);
void bad_request(int);
void cat(int, FILE *);
void cannot_execute(int);
void error_die(const char *);
void execute_cgi(int, const char *, const char *, const char *);
int get_line(int, char *, int);
void headers(int, const char *);
void not_found(int);
void serve_file(int, const char *);
int startup(u_short *);
void unimplemented(int);
```

先简单地解释每个函数的作用：

* accept_request:  处理从套接字上监听到的一个 HTTP 请求，在这里可以很大一部分地体现服务器处理请求流程。

* bad_request: 返回给客户端这是个错误请求，HTTP 状态吗 400 BAD REQUEST.

* cat: 读取服务器上某个文件写到 socket 套接字。

* cannot_execute: 主要处理发生在执行 cgi 程序时出现的错误。

* error_die: 把错误信息写到 perror 并退出。

* execute_cgi: 运行 cgi 程序的处理，也是个主要函数。

* get_line: 读取套接字的一行，把回车换行等情况都统一为换行符结束。

* headers: 把 HTTP 响应的头部写到套接字。

* not_found: 主要处理找不到请求的文件时的情况。

* sever_file: 调用 cat 把服务器文件返回给浏览器。

* startup: 初始化 httpd 服务，包括建立套接字，绑定端口，进行监听等。

* unimplemented: 返回给浏览器表明收到的 HTTP 请求所用的 method 不被支持。

建议源码阅读顺序： main -> startup -> accept_request -> execute_cgi, 通晓主要工作流程后再仔细把每个函数的源码看一看。

## 工作流程

1. 服务器启动，在指定端口或随机选取端口绑定 httpd 服务。
2. 收到一个 HTTP 请求时（其实就是 listen 的端口 accpet 的时候），派生一个线程运行 accept_request 函数
3. 取出 HTTP 请求中的 method (GET 或 POST) 和 url,。对于 GET 方法，如果有携带参数，则 query_string 指针指向 url 中 ？ 后面的 GET 参数。
4. 格式化 url 到 path 数组，表示浏览器请求的服务器文件路径，在 tinyhttpd 中服务器文件是在 htdocs 文件夹下。当 url 以 / 结尾，或 url 是个目录，则默认在 path 中加上 index.html，表示访问主页。
5. 如果文件路径合法，对于无参数的 GET 请求，直接输出服务器文件到浏览器，即用 HTTP 格式写到套接字上，跳到 10。其他情况（带参数 GET，POST 方式，url 为可执行文件），则调用 excute_cgi 函数执行 cgi 脚本。
6. 读取整个 HTTP 请求并丢弃，如果是 POST 则找出 Content-Length. 把 HTTP 200  状态码写到套接字。
7.  建立两个管道，cgi_input 和 cgi_output, 并 fork 一个进程。
8. 在子进程中，把 STDOUT 重定向到 cgi_outputt 的写入端，把 STDIN 重定向到 cgi_input 的读取端，关闭 cgi_input 的写入端 和 cgi_output 的读取端，设置 request_method 的环境变量，GET 的话设置 query_string 的环境变量，POST 的话设置 content_length 的环境变量，这些环境变量都是为了给 cgi 脚本调用，接着用 execl 运行 cgi 程序。
9. 在父进程中，关闭 cgi_input 的读取端 和 cgi_output 的写入端，如果 POST 的话，把 POST 数据写入 cgi_input，已被重定向到 STDIN，读取 cgi_output 的管道输出到客户端，该管道输入是 STDOUT。接着关闭所有管道，等待子进程结束。这一部分比较乱，见下图说明:

- 管道初始状态
![管道初始状态](https://github.com/AngryHacker/articles/blob/master/img/20141226173222750.jpg?raw=true)
- 管道最终状态
![管道最终状态](https://github.com/AngryHacker/articles/blob/master/img/20141226161119981.jpg?raw=true)
10. 关闭与浏览器的连接，完成了一次 HTTP 请求与回应，因为 HTTP 是无连接的。

## 注释版源码

源码已写了注释，放在 Github:  [这里](https://github.com/AngryHacker/code-with-comments/blob/master/tinyhttpd/httpd.c)

懒得跳转的同学看下面....

```c
/* J. David's webserver */
/* This is a simple webserver.
 * Created November 1999 by J. David Blackstone.
 * CSE 4344 (Network concepts), Prof. Zeigler
 * University of Texas at Arlington
 */
/* This program compiles for Sparc Solaris 2.6.
 * To compile for Linux:
 *  1) Comment out the #include <pthread.h> line.
 *  2) Comment out the line that defines the variable newthread.
 *  3) Comment out the two lines that run pthread_create().
 *  4) Uncomment the line that runs accept_request().
 *  5) Remove -lsocket from the Makefile.
 */
#include <stdio.h>
#include <sys/socket.h>
#include <sys/types.h>
#include <netinet/in.h>
#include <arpa/inet.h>
#include <unistd.h>
#include <ctype.h>
#include <strings.h>
#include <string.h>
#include <sys/stat.h>
#include <pthread.h>
#include <sys/wait.h>
#include <stdlib.h>

#define ISspace(x) isspace((int)(x))

#define SERVER_STRING "Server: jdbhttpd/0.1.0\r\n"

void accept_request(int);
void bad_request(int);
void cat(int, FILE *);
void cannot_execute(int);
void error_die(const char *);
void execute_cgi(int, const char *, const char *, const char *);
int get_line(int, char *, int);
void headers(int, const char *);
void not_found(int);
void serve_file(int, const char *);
int startup(u_short *);
void unimplemented(int);

/**********************************************************************/
/* A request has caused a call to accept() on the server port to
 * return.  Process the request appropriately.
 * Parameters: the socket connected to the client */
/**********************************************************************/
void accept_request(int client)
{
    char buf[1024];
    int numchars;
    char method[255];
    char url[255];
    char path[512];
    size_t i, j;
    struct stat st;
    int cgi = 0;      /* becomes true if server decides this is a CGI program */
    char *query_string = NULL;

    /*得到请求的第一行*/
    numchars = get_line(client, buf, sizeof(buf));
    i = 0; j = 0;
    /*把客户端的请求方法存到 method 数组*/
    while (!ISspace(buf[j]) && (i < sizeof(method) - 1))
    {
        method[i] = buf[j];
        i++; j++;
    }
    method[i] = '\0';

    /*如果既不是 GET 又不是 POST 则无法处理 */
    if (strcasecmp(method, "GET") && strcasecmp(method, "POST"))
    {
        unimplemented(client);
        return;
    }

    /* POST 的时候开启 cgi */
    if (strcasecmp(method, "POST") == 0)
        cgi = 1;

    /*读取 url 地址*/
    i = 0;
    while (ISspace(buf[j]) && (j < sizeof(buf)))
        j++;
    while (!ISspace(buf[j]) && (i < sizeof(url) - 1) && (j < sizeof(buf)))
    {
        /*存下 url */
        url[i] = buf[j];
        i++; j++;
    }
    url[i] = '\0';

    /*处理 GET 方法*/
    if (strcasecmp(method, "GET") == 0)
    {
        /* 待处理请求为 url */
        query_string = url;
        while ((*query_string != '?') && (*query_string != '\0'))
            query_string++;
        /* GET 方法特点，? 后面为参数*/
        if (*query_string == '?')
        {
            /*开启 cgi */
            cgi = 1;
            *query_string = '\0';
            query_string++;
        }
    }

    /*格式化 url 到 path 数组，html 文件都在 htdocs 中*/
    sprintf(path, "htdocs%s", url);
    /*默认情况为 index.html */
    if (path[strlen(path) - 1] == '/')
        strcat(path, "index.html");
    /*根据路径找到对应文件 */
    if (stat(path, &st) == -1) {
        /*把所有 headers 的信息都丢弃*/
        while ((numchars > 0) && strcmp("\n", buf))  /* read & discard headers */
            numchars = get_line(client, buf, sizeof(buf));
        /*回应客户端找不到*/
        not_found(client);
    }
    else
    {
        /*如果是个目录，则默认使用该目录下 index.html 文件*/
        if ((st.st_mode & S_IFMT) == S_IFDIR)
            strcat(path, "/index.html");
      if ((st.st_mode & S_IXUSR) || (st.st_mode & S_IXGRP) || (st.st_mode & S_IXOTH)    )
          cgi = 1;
      /*不是 cgi,直接把服务器文件返回，否则执行 cgi */
      if (!cgi)
          serve_file(client, path);
      else
          execute_cgi(client, path, method, query_string);
    }

    /*断开与客户端的连接（HTTP 特点：无连接）*/
    close(client);
}

/**********************************************************************/
/* Inform the client that a request it has made has a problem.
 * Parameters: client socket */
/**********************************************************************/
void bad_request(int client)
{
    char buf[1024];

    /*回应客户端错误的 HTTP 请求 */
    sprintf(buf, "HTTP/1.0 400 BAD REQUEST\r\n");
    send(client, buf, sizeof(buf), 0);
    sprintf(buf, "Content-type: text/html\r\n");
    send(client, buf, sizeof(buf), 0);
    sprintf(buf, "\r\n");
    send(client, buf, sizeof(buf), 0);
    sprintf(buf, "<P>Your browser sent a bad request, ");
    send(client, buf, sizeof(buf), 0);
    sprintf(buf, "such as a POST without a Content-Length.\r\n");
    send(client, buf, sizeof(buf), 0);
}

/**********************************************************************/
/* Put the entire contents of a file out on a socket.  This function
 * is named after the UNIX "cat" command, because it might have been
 * easier just to do something like pipe, fork, and exec("cat").
 * Parameters: the client socket descriptor
 *             FILE pointer for the file to cat */
/**********************************************************************/
void cat(int client, FILE *resource)
{
    char buf[1024];

    /*读取文件中的所有数据写到 socket */
    fgets(buf, sizeof(buf), resource);
    while (!feof(resource))
    {
        send(client, buf, strlen(buf), 0);
        fgets(buf, sizeof(buf), resource);
    }
}

/**********************************************************************/
/* Inform the client that a CGI script could not be executed.
 * Parameter: the client socket descriptor. */
/**********************************************************************/
void cannot_execute(int client)
{
    char buf[1024];

    /* 回应客户端 cgi 无法执行*/
    sprintf(buf, "HTTP/1.0 500 Internal Server Error\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "Content-type: text/html\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "<P>Error prohibited CGI execution.\r\n");
    send(client, buf, strlen(buf), 0);
}

/**********************************************************************/
/* Print out an error message with perror() (for system errors; based
 * on value of errno, which indicates system call errors) and exit the
 * program indicating an error. */
/**********************************************************************/
void error_die(const char *sc)
{
    /*出错信息处理 */
    perror(sc);
    exit(1);
}

/**********************************************************************/
/* Execute a CGI script.  Will need to set environment variables as
 * appropriate.
 * Parameters: client socket descriptor
 *             path to the CGI script */
/**********************************************************************/
void execute_cgi(int client, const char *path, const char *method, const char *query_string)
{
    char buf[1024];
    int cgi_output[2];
    int cgi_input[2];
    pid_t pid;
    int status;
    int i;
    char c;
    int numchars = 1;
    int content_length = -1;

    buf[0] = 'A'; buf[1] = '\0';
    if (strcasecmp(method, "GET") == 0)
        /*把所有的 HTTP header 读取并丢弃*/
        while ((numchars > 0) && strcmp("\n", buf))  /* read & discard headers */
            numchars = get_line(client, buf, sizeof(buf));
    else    /* POST */
    {
        /* 对 POST 的 HTTP 请求中找出 content_length */
        numchars = get_line(client, buf, sizeof(buf));
        while ((numchars > 0) && strcmp("\n", buf))
        {
            /*利用 \0 进行分隔 */
            buf[15] = '\0';
            /* HTTP 请求的特点*/
            if (strcasecmp(buf, "Content-Length:") == 0)
                content_length = atoi(&(buf[16]));
            numchars = get_line(client, buf, sizeof(buf));
        }
        /*没有找到 content_length */
        if (content_length == -1) {
            /*错误请求*/
            bad_request(client);
            return;
        }
    }

    /* 正确，HTTP 状态码 200 */
    sprintf(buf, "HTTP/1.0 200 OK\r\n");
    send(client, buf, strlen(buf), 0);

    /* 建立管道*/
    if (pipe(cgi_output) < 0) {
        /*错误处理*/
        cannot_execute(client);
        return;
    }
    /*建立管道*/
    if (pipe(cgi_input) < 0) {
        /*错误处理*/
        cannot_execute(client);
        return;
    }

    if ((pid = fork()) < 0 ) {
        /*错误处理*/
        cannot_execute(client);
        return;
    }
    if (pid == 0)  /* child: CGI script */
    {
        char meth_env[255];
        char query_env[255];
        char length_env[255];

        /* 把 STDOUT 重定向到 cgi_output 的写入端 */
        dup2(cgi_output[1], 1);
        /* 把 STDIN 重定向到 cgi_input 的读取端 */
        dup2(cgi_input[0], 0);
        /* 关闭 cgi_input 的写入端 和 cgi_output 的读取端 */
        close(cgi_output[0]);
        close(cgi_input[1]);
        /*设置 request_method 的环境变量*/
        sprintf(meth_env, "REQUEST_METHOD=%s", method);
        putenv(meth_env);
        if (strcasecmp(method, "GET") == 0) {
            /*设置 query_string 的环境变量*/
            sprintf(query_env, "QUERY_STRING=%s", query_string);
            putenv(query_env);
        }
        else {   /* POST */
            /*设置 content_length 的环境变量*/
            sprintf(length_env, "CONTENT_LENGTH=%d", content_length);
            putenv(length_env);
        }
        /*用 execl 运行 cgi 程序*/
        execl(path, path, NULL);
        exit(0);
    } else {    /* parent */
        /* 关闭 cgi_input 的读取端 和 cgi_output 的写入端 */
        close(cgi_output[1]);
        close(cgi_input[0]);
        if (strcasecmp(method, "POST") == 0)
            /*接收 POST 过来的数据*/
            for (i = 0; i < content_length; i++) {
                recv(client, &c, 1, 0);
                /*把 POST 数据写入 cgi_input，现在重定向到 STDIN */
                write(cgi_input[1], &c, 1);
            }
        /*读取 cgi_output 的管道输出到客户端，该管道输入是 STDOUT */
        while (read(cgi_output[0], &c, 1) > 0)
            send(client, &c, 1, 0);

        /*关闭管道*/
        close(cgi_output[0]);
        close(cgi_input[1]);
        /*等待子进程*/
        waitpid(pid, &status, 0);
    }
}

/**********************************************************************/
/* Get a line from a socket, whether the line ends in a newline,
 * carriage return, or a CRLF combination.  Terminates the string read
 * with a null character.  If no newline indicator is found before the
 * end of the buffer, the string is terminated with a null.  If any of
 * the above three line terminators is read, the last character of the
 * string will be a linefeed and the string will be terminated with a
 * null character.
 * Parameters: the socket descriptor
 *             the buffer to save the data in
 *             the size of the buffer
 * Returns: the number of bytes stored (excluding null) */
/**********************************************************************/
int get_line(int sock, char *buf, int size)
{
    int i = 0;
    char c = '\0';
    int n;

    /*把终止条件统一为 \n 换行符，标准化 buf 数组*/
    while ((i < size - 1) && (c != '\n'))
    {
        /*一次仅接收一个字节*/
        n = recv(sock, &c, 1, 0);
        /* DEBUG printf("%02X\n", c); */
        if (n > 0)
        {
            /*收到 \r 则继续接收下个字节，因为换行符可能是 \r\n */
            if (c == '\r')
            {
                /*使用 MSG_PEEK 标志使下一次读取依然可以得到这次读取的内容，可认为接收窗口不滑动*/
                n = recv(sock, &c, 1, MSG_PEEK);
                /* DEBUG printf("%02X\n", c); */
                /*但如果是换行符则把它吸收掉*/
                if ((n > 0) && (c == '\n'))
                    recv(sock, &c, 1, 0);
                else
                    c = '\n';
            }
            /*存到缓冲区*/
            buf[i] = c;
            i++;
        }
        else
            c = '\n';
    }
    buf[i] = '\0';

    /*返回 buf 数组大小*/
    return(i);
}

/**********************************************************************/
/* Return the informational HTTP headers about a file. */
/* Parameters: the socket to print the headers on
 *             the name of the file */
/**********************************************************************/
void headers(int client, const char *filename)
{
    char buf[1024];
    (void)filename;  /* could use filename to determine file type */

    /*正常的 HTTP header */
    strcpy(buf, "HTTP/1.0 200 OK\r\n");
    send(client, buf, strlen(buf), 0);
    /*服务器信息*/
    strcpy(buf, SERVER_STRING);
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "Content-Type: text/html\r\n");
    send(client, buf, strlen(buf), 0);
    strcpy(buf, "\r\n");
    send(client, buf, strlen(buf), 0);
}

/**********************************************************************/
/* Give a client a 404 not found status message. */
/**********************************************************************/
void not_found(int client)
{
    char buf[1024];

    /* 404 页面 */
    sprintf(buf, "HTTP/1.0 404 NOT FOUND\r\n");
    send(client, buf, strlen(buf), 0);
    /*服务器信息*/
    sprintf(buf, SERVER_STRING);
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "Content-Type: text/html\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "<HTML><TITLE>Not Found</TITLE>\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "<BODY><P>The server could not fulfill\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "your request because the resource specified\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "is unavailable or nonexistent.\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "</BODY></HTML>\r\n");
    send(client, buf, strlen(buf), 0);
}

/**********************************************************************/
/* Send a regular file to the client.  Use headers, and report
 * errors to client if they occur.
 * Parameters: a pointer to a file structure produced from the socket
 *              file descriptor
 *             the name of the file to serve */
/**********************************************************************/
void serve_file(int client, const char *filename)
{
    FILE *resource = NULL;
    int numchars = 1;
    char buf[1024];

    /*读取并丢弃 header */
    buf[0] = 'A'; buf[1] = '\0';
    while ((numchars > 0) && strcmp("\n", buf))  /* read & discard headers */
        numchars = get_line(client, buf, sizeof(buf));

    /*打开 sever 的文件*/
    resource = fopen(filename, "r");
    if (resource == NULL)
        not_found(client);
    else
    {
        /*写 HTTP header */
        headers(client, filename);
        /*复制文件*/
        cat(client, resource);
    }
    fclose(resource);
}

/**********************************************************************/
/* This function starts the process of listening for web connections
 * on a specified port.  If the port is 0, then dynamically allocate a
 * port and modify the original port variable to reflect the actual
 * port.
 * Parameters: pointer to variable containing the port to connect on
 * Returns: the socket */
/**********************************************************************/
int startup(u_short *port)
{
    int httpd = 0;
    struct sockaddr_in name;

    /*建立 socket */
    httpd = socket(PF_INET, SOCK_STREAM, 0);
    if (httpd == -1)
        error_die("socket");
    memset(&name, 0, sizeof(name));
    name.sin_family = AF_INET;
    name.sin_port = htons(*port);
    name.sin_addr.s_addr = htonl(INADDR_ANY);
    if (bind(httpd, (struct sockaddr *)&name, sizeof(name)) < 0)
        error_die("bind");
    /*如果当前指定端口是 0，则动态随机分配一个端口*/
    if (*port == 0)  /* if dynamically allocating a port */
    {
        int namelen = sizeof(name);
        if (getsockname(httpd, (struct sockaddr *)&name, &namelen) == -1)
            error_die("getsockname");
        *port = ntohs(name.sin_port);
    }
    /*开始监听*/
    if (listen(httpd, 5) < 0)
        error_die("listen");
    /*返回 socket id */
    return(httpd);
}

/**********************************************************************/
/* Inform the client that the requested web method has not been
 * implemented.
 * Parameter: the client socket */
/**********************************************************************/
void unimplemented(int client)
{
    char buf[1024];

    /* HTTP method 不被支持*/
    sprintf(buf, "HTTP/1.0 501 Method Not Implemented\r\n");
    send(client, buf, strlen(buf), 0);
    /*服务器信息*/
    sprintf(buf, SERVER_STRING);
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "Content-Type: text/html\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "<HTML><HEAD><TITLE>Method Not Implemented\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "</TITLE></HEAD>\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "<BODY><P>HTTP request method not supported.\r\n");
    send(client, buf, strlen(buf), 0);
    sprintf(buf, "</BODY></HTML>\r\n");
    send(client, buf, strlen(buf), 0);
}

/**********************************************************************/

int main(void)
{
    int server_sock = -1;
    u_short port = 0;
    int client_sock = -1;
    struct sockaddr_in client_name;
    int client_name_len = sizeof(client_name);
    pthread_t newthread;

    /*在对应端口建立 httpd 服务*/
    server_sock = startup(&port);
    printf("httpd running on port %d\n", port);

    while (1)
    {
        /*套接字收到客户端连接请求*/
        client_sock = accept(server_sock,(struct sockaddr *)&client_name,&client_name_len);
        if (client_sock == -1)
            error_die("accept");
        /*派生新线程用 accept_request 函数处理新请求*/
        /* accept_request(client_sock); */
        if (pthread_create(&newthread , NULL, accept_request, client_sock) != 0)
            perror("pthread_create");
    }

    close(server_sock);

    return(0);
}
```

## 编译运行
这里我们如何让这个服务器跑起来呢？
* 首先，需要修改 Makefile 文件，如果直接执行 make 的话，你会遇到这个错误：cannot find -lsocket。
* 解决方法简单说就是在 linux 系统中没有这样一个库，而且这个库在 linux 中的实现位于 libc 中，编译时被默认包含，所以可以直接在 Makefile 中去掉 -lsocket。详见：stackoverflow
* 另外，在 htdocs 文件下，有 cgi 的程序和 html 代码，cgi 是用 perl 写的，但文件中声明的 perl 执行程序位置在我这是错的，我这里 perl 脚本位于 /usr/bin 中（用 which perl 可以查看），所以把  cgi 文件中的第一行改为：#!/usr/bin/perl -Tw
* 现在都没问题了。mkae 编译后，用 ./httpd 运行，在浏览器中输入 127.0.0.1:端口，就可以看到网页了：

![run](https://github.com/AngryHacker/articles/blob/master/img/20160711221626477.png?raw=true)
![demo](https://github.com/AngryHacker/articles/blob/master/img/20160711221642431.png?raw=true)

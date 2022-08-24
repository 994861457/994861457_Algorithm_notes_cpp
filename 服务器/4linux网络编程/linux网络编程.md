
# tcp通信实现
## 服务器端  
```cpp  
int main() {

    // 1.创建socket(用于监听的套接字)
    int lfd = socket(AF_INET, SOCK_STREAM, 0);

    if(lfd == -1) {
        perror("socket");
        exit(-1);
    }

    // 2.绑定
    struct sockaddr_in saddr;
    saddr.sin_family = AF_INET;
    // inet_pton(AF_INET, "192.168.193.128", saddr.sin_addr.s_addr);
    saddr.sin_addr.s_addr = INADDR_ANY;  // 0.0.0.0
    saddr.sin_port = htons(9999);
    int ret = bind(lfd, (struct sockaddr *)&saddr, sizeof(saddr));

    if(ret == -1) {
        perror("bind");
        exit(-1);
    }

    // 3.监听
    ret = listen(lfd, 8);
    if(ret == -1) {
        perror("listen");
        exit(-1);
    }

    // 4.接收客户端连接
    struct sockaddr_in clientaddr;
    int len = sizeof(clientaddr);
    int cfd = accept(lfd, (struct sockaddr *)&clientaddr, &len);
    
    if(cfd == -1) {
        perror("accept");
        exit(-1);
    }

    // 输出客户端的信息
    char clientIP[16];
    inet_ntop(AF_INET, &clientaddr.sin_addr.s_addr, clientIP, sizeof(clientIP));
    unsigned short clientPort = ntohs(clientaddr.sin_port);
    printf("client ip is %s, port is %d\n", clientIP, clientPort);

    // 5.通信
    char recvBuf[1024] = {0};
    while(1) {
        
        // 获取客户端的数据
        int num = read(cfd, recvBuf, sizeof(recvBuf));
        if(num == -1) {
            perror("read");
            exit(-1);
        } else if(num > 0) {
            printf("recv client data : %s\n", recvBuf);
        } else if(num == 0) {
            // 表示客户端断开连接
            printf("clinet closed...");
            break;
        }

        char * data = "hello,i am server";
        // 给客户端发送数据
        write(cfd, data, strlen(data));
    }
   
    // 关闭文件描述符
    close(cfd);
    close(lfd);

    return 0;
}
```
## 客户端  
```cpp
int main() {

    // 1.创建套接字
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if(fd == -1) {
        perror("socket");
        exit(-1);
    }

    // 2.连接服务器端
    struct sockaddr_in serveraddr;
    serveraddr.sin_family = AF_INET;
    inet_pton(AF_INET, "192.168.193.128", &serveraddr.sin_addr.s_addr);
    serveraddr.sin_port = htons(9999);
    int ret = connect(fd, (struct sockaddr *)&serveraddr, sizeof(serveraddr));

    if(ret == -1) {
        perror("connect");
        exit(-1);
    }

    
    // 3. 通信
    char recvBuf[1024] = {0};
    while(1) {

        char * data = "hello,i am client";
        // 给客户端发送数据
        write(fd, data , strlen(data));

        sleep(1);
        
        int len = read(fd, recvBuf, sizeof(recvBuf));
        if(len == -1) {
            perror("read");
            exit(-1);
        } else if(len > 0) {
            printf("recv server data : %s\n", recvBuf);
        } else if(len == 0) {
            // 表示服务器端断开连接
            printf("server closed...");
            break;
        }

    }

    // 关闭连接
    close(fd);

    return 0;
}  
```  
# 多进程实现并发服务器  
## 服务器
```cpp
void recyleChild(int arg) {
    while(1) {
        int ret = waitpid(-1, NULL, WNOHANG);
        if(ret == -1) {
            // 所有的子进程都回收了
            break;
        }else if(ret == 0) {
            // 还有子进程活着
            break;
        } else if(ret > 0){
            // 被回收了
            printf("子进程 %d 被回收了\n", ret);
        }
    }
}

int main() {

    struct sigaction act;
    act.sa_flags = 0;
    sigemptyset(&act.sa_mask);
    act.sa_handler = recyleChild;
    // 注册信号捕捉
    sigaction(SIGCHLD, &act, NULL);
    

    // 创建socket
    int lfd = socket(PF_INET, SOCK_STREAM, 0);
    if(lfd == -1){
        perror("socket");
        exit(-1);
    }

    struct sockaddr_in saddr;
    saddr.sin_family = AF_INET;
    saddr.sin_port = htons(9999);
    saddr.sin_addr.s_addr = INADDR_ANY;

    // 绑定
    int ret = bind(lfd,(struct sockaddr *)&saddr, sizeof(saddr));
    if(ret == -1) {
        perror("bind");
        exit(-1);
    }

    // 监听
    ret = listen(lfd, 128);
    if(ret == -1) {
        perror("listen");
        exit(-1);
    }

    // 不断循环等待客户端连接
    while(1) {

        struct sockaddr_in cliaddr;
        int len = sizeof(cliaddr);
        // 接受连接
        int cfd = accept(lfd, (struct sockaddr*)&cliaddr, &len);
        if(cfd == -1) {
            if(errno == EINTR) {
                continue;
            }
            perror("accept");
            exit(-1);
        }

        // 每一个连接进来，创建一个子进程跟客户端通信
        pid_t pid = fork();
        if(pid == 0) {
            // 子进程
            // 获取客户端的信息
            char cliIp[16];
            inet_ntop(AF_INET, &cliaddr.sin_addr.s_addr, cliIp, sizeof(cliIp));
            unsigned short cliPort = ntohs(cliaddr.sin_port);
            printf("client ip is : %s, prot is %d\n", cliIp, cliPort);

            // 接收客户端发来的数据
            char recvBuf[1024];
            while(1) {
                int len = read(cfd, &recvBuf, sizeof(recvBuf));

                if(len == -1) {
                    perror("read");
                    exit(-1);
                }else if(len > 0) {
                    printf("recv client : %s\n", recvBuf);
                } else if(len == 0) {
                    printf("client closed....\n");
                    break;
                }
                write(cfd, recvBuf, strlen(recvBuf) + 1);
            }
            close(cfd);
            exit(0);    // 退出当前子进程
        }

    }
    close(lfd);
    return 0;
}
```  
## 客户端  
```cpp
int main() {

    // 1.创建套接字
    int fd = socket(AF_INET, SOCK_STREAM, 0);
    if(fd == -1) {
        perror("socket");
        exit(-1);
    }

    // 2.连接服务器端
    struct sockaddr_in serveraddr;
    serveraddr.sin_family = AF_INET;
    inet_pton(AF_INET, "192.168.193.128", &serveraddr.sin_addr.s_addr);
    serveraddr.sin_port = htons(9999);
    int ret = connect(fd, (struct sockaddr *)&serveraddr, sizeof(serveraddr));

    if(ret == -1) {
        perror("connect");
        exit(-1);
    }
    
    // 3. 通信
    char recvBuf[1024];
    int i = 0;
    while(1) {
        
        sprintf(recvBuf, "data : %d\n", i++);
        
        // 给服务器端发送数据
        write(fd, recvBuf, strlen(recvBuf)+1);

        int len = read(fd, recvBuf, sizeof(recvBuf));
        if(len == -1) {
            perror("read");
            exit(-1);
        } else if(len > 0) {
            printf("recv server : %s\n", recvBuf);
        } else if(len == 0) {
            // 表示服务器端断开连接
            printf("server closed...");
            break;
        }

        sleep(1);
    }

    // 关闭连接
    close(fd);

    return 0;
}  
```  
# 多线程实现并发服务器  
```cpp  
struct sockInfo {
    int fd; // 通信的文件描述符
    struct sockaddr_in addr;
    pthread_t tid;  // 线程号
};

struct sockInfo sockinfos[128];

void * working(void * arg) {
    // 子线程和客户端通信   cfd 客户端的信息 线程号
    // 获取客户端的信息
    struct sockInfo * pinfo = (struct sockInfo *)arg;

    char cliIp[16];
    inet_ntop(AF_INET, &pinfo->addr.sin_addr.s_addr, cliIp, sizeof(cliIp));
    unsigned short cliPort = ntohs(pinfo->addr.sin_port);
    printf("client ip is : %s, prot is %d\n", cliIp, cliPort);

    // 接收客户端发来的数据
    char recvBuf[1024];
    while(1) {
        int len = read(pinfo->fd, &recvBuf, sizeof(recvBuf));

        if(len == -1) {
            perror("read");
            exit(-1);
        }else if(len > 0) {
            printf("recv client : %s\n", recvBuf);
        } else if(len == 0) {
            printf("client closed....\n");
            break;
        }
        write(pinfo->fd, recvBuf, strlen(recvBuf) + 1);
    }
    close(pinfo->fd);
    return NULL;
}

int main() {

    // 创建socket
    int lfd = socket(PF_INET, SOCK_STREAM, 0);
    if(lfd == -1){
        perror("socket");
        exit(-1);
    }

    struct sockaddr_in saddr;
    saddr.sin_family = AF_INET;
    saddr.sin_port = htons(9999);
    saddr.sin_addr.s_addr = INADDR_ANY;

    // 绑定
    int ret = bind(lfd,(struct sockaddr *)&saddr, sizeof(saddr));
    if(ret == -1) {
        perror("bind");
        exit(-1);
    }

    // 监听
    ret = listen(lfd, 128);
    if(ret == -1) {
        perror("listen");
        exit(-1);
    }

    // 初始化数据
    int max = sizeof(sockinfos) / sizeof(sockinfos[0]);
    for(int i = 0; i < max; i++) {
        bzero(&sockinfos[i], sizeof(sockinfos[i]));
        sockinfos[i].fd = -1;
        sockinfos[i].tid = -1;
    }

    // 循环等待客户端连接，一旦一个客户端连接进来，就创建一个子线程进行通信
    while(1) {

        struct sockaddr_in cliaddr;
        int len = sizeof(cliaddr);
        // 接受连接
        int cfd = accept(lfd, (struct sockaddr*)&cliaddr, &len);

        struct sockInfo * pinfo;
        for(int i = 0; i < max; i++) {
            // 从这个数组中找到一个可以用的sockInfo元素
            if(sockinfos[i].fd == -1) {
                pinfo = &sockinfos[i];
                break;
            }
            if(i == max - 1) {
                sleep(1);
                i--;
            }
        }

        pinfo->fd = cfd;
        memcpy(&pinfo->addr, &cliaddr, len);

        // 创建子线程
        pthread_create(&pinfo->tid, NULL, working, pinfo);

        pthread_detach(pinfo->tid);
    }

    close(lfd);
    return 0;
}

```  
# 半关闭和端口复用  
1.当通过服务端发起的关闭连接操作，引发了一个已有的TCP连接处于TIME_WAIT状态，此时，服务器重启，继续绑定原来ip与端口号，返回了Address already in use的错误。  

2.调用close或者cntl + c 关闭进程都视为发起close信号。先发起close信号的那一方在另一方正常发起close信号之后会进入timewait状态。  
netstat anp | grep 8888 查看端口号为8888的所有进程的状态 
```cpp
//写在绑定之前
int optval = 1;
setsockopt(lfd, SOL_SOCKET, SO_REUSEPORT, &optval, sizeof(optval));  
```  
# IO多路复用  
https://blog.csdn.net/m0_51319483/article/details/124264619
## select
```cpp
int main() {

    // 创建socket
    int lfd = socket(PF_INET, SOCK_STREAM, 0);
    struct sockaddr_in saddr;
    saddr.sin_port = htons(9999);
    saddr.sin_family = AF_INET;
    saddr.sin_addr.s_addr = INADDR_ANY;

    // 绑定
    bind(lfd, (struct sockaddr *)&saddr, sizeof(saddr));

    // 监听
    listen(lfd, 8);

    // 创建一个fd_set的集合，存放的是需要检测的文件描述符
    fd_set rdset, tmp;
    FD_ZERO(&rdset);
    FD_SET(lfd, &rdset);
    int maxfd = lfd;

    while(1) {

        tmp = rdset;

        // 调用select系统函数，让内核帮检测哪些文件描述符有数据
        int ret = select(maxfd + 1, &tmp, NULL, NULL, NULL);
        if(ret == -1) {
            perror("select");
            exit(-1);
        } else if(ret == 0) {
            continue;
        } else if(ret > 0) {
            // 说明检测到了有文件描述符的对应的缓冲区的数据发生了改变
            if(FD_ISSET(lfd, &tmp)) {
                // 表示有新的客户端连接进来了
                struct sockaddr_in cliaddr;
                int len = sizeof(cliaddr);
                int cfd = accept(lfd, (struct sockaddr *)&cliaddr, &len);

                // 将新的文件描述符加入到集合中
                FD_SET(cfd, &rdset);

                // 更新最大的文件描述符
                maxfd = maxfd > cfd ? maxfd : cfd;
            }

            for(int i = lfd + 1; i <= maxfd; i++) {
                if(FD_ISSET(i, &tmp)) {
                    // 说明这个文件描述符对应的客户端发来了数据
                    char buf[1024] = {0};
                    int len = read(i, buf, sizeof(buf));
                    if(len == -1) {
                        perror("read");
                        exit(-1);
                    } else if(len == 0) {
                        printf("client closed...\n");
                        close(i);
                        FD_CLR(i, &rdset);
                    } else if(len > 0) {
                        printf("read buf = %s\n", buf);
                        write(i, buf, strlen(buf) + 1);
                    }
                }
            }

        }

    }
    close(lfd);
    return 0;
}
```  
## poll  
```cpp  
int main() {

    // 创建socket
    int lfd = socket(PF_INET, SOCK_STREAM, 0);
    struct sockaddr_in saddr;
    saddr.sin_port = htons(9999);
    saddr.sin_family = AF_INET;
    saddr.sin_addr.s_addr = INADDR_ANY;

    // 绑定
    bind(lfd, (struct sockaddr *)&saddr, sizeof(saddr));

    // 监听
    listen(lfd, 8);

    // 初始化检测的文件描述符数组
    struct pollfd fds[1024];
    for(int i = 0; i < 1024; i++) {
        fds[i].fd = -1;
        fds[i].events = POLLIN;
    }
    fds[0].fd = lfd;
    int nfds = 0;

    while(1) {

        // 调用poll系统函数，让内核帮检测哪些文件描述符有数据
        int ret = poll(fds, nfds + 1, -1);
        if(ret == -1) {
            perror("poll");
            exit(-1);
        } else if(ret == 0) {
            continue;
        } else if(ret > 0) {
            // 说明检测到了有文件描述符的对应的缓冲区的数据发生了改变
            if(fds[0].revents & POLLIN) {
                // 表示有新的客户端连接进来了
                struct sockaddr_in cliaddr;
                int len = sizeof(cliaddr);
                int cfd = accept(lfd, (struct sockaddr *)&cliaddr, &len);

                // 将新的文件描述符加入到集合中
                for(int i = 1; i < 1024; i++) {
                    if(fds[i].fd == -1) {
                        fds[i].fd = cfd;
                        fds[i].events = POLLIN;
                        break;
                    }
                }

                // 更新最大的文件描述符的索引
                nfds = nfds > cfd ? nfds : cfd;
            }

            for(int i = 1; i <= nfds; i++) {
                if(fds[i].revents & POLLIN) {
                    // 说明这个文件描述符对应的客户端发来了数据
                    char buf[1024] = {0};
                    int len = read(fds[i].fd, buf, sizeof(buf));
                    if(len == -1) {
                        perror("read");
                        exit(-1);
                    } else if(len == 0) {
                        printf("client closed...\n");
                        close(fds[i].fd);
                        fds[i].fd = -1;
                    } else if(len > 0) {
                        printf("read buf = %s\n", buf);
                        write(fds[i].fd, buf, strlen(buf) + 1);
                    }
                }
            }

        }

    }
    close(lfd);
    return 0;
}
```
## epoll    
https://blog.csdn.net/weixin_46258483/article/details/122562266  
1.用buf接收比较短的数据的时候不要忘了最后要加上\0,这样字符串才知道终点在哪里，不然print的时候会把后面的信息也带上，直到遇到\0.  
在阻塞状态的read如果数据读到头会一直阻塞在那里，而不是会向非阻塞一样去返回0  

2.两种工作模式 et和lt
```cpp  
//lt
int main() {

    // 创建socket
    int lfd = socket(PF_INET, SOCK_STREAM, 0);
    struct sockaddr_in saddr;
    saddr.sin_port = htons(9999);
    saddr.sin_family = AF_INET;
    saddr.sin_addr.s_addr = INADDR_ANY;

    // 绑定
    bind(lfd, (struct sockaddr *)&saddr, sizeof(saddr));

    // 监听
    listen(lfd, 8);

    // 调用epoll_create()创建一个epoll实例
    int epfd = epoll_create(100);

    // 将监听的文件描述符相关的检测信息添加到epoll实例中
    struct epoll_event epev;
    epev.events = EPOLLIN;
    epev.data.fd = lfd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &epev);

    struct epoll_event epevs[1024];

    while(1) {

        int ret = epoll_wait(epfd, epevs, 1024, -1);
        if(ret == -1) {
            perror("epoll_wait");
            exit(-1);
        }

        printf("ret = %d\n", ret);

        for(int i = 0; i < ret; i++) {

            int curfd = epevs[i].data.fd;

            if(curfd == lfd) {
                // 监听的文件描述符有数据达到，有客户端连接
                struct sockaddr_in cliaddr;
                int len = sizeof(cliaddr);
                int cfd = accept(lfd, (struct sockaddr *)&cliaddr, &len);

                epev.events = EPOLLIN;
                epev.data.fd = cfd;
                epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &epev);
            } else {
                if(epevs[i].events & EPOLLOUT) {
                    continue;
                }   
                // 有数据到达，需要通信
                char buf[1024] = {0};
                int len = read(curfd, buf, sizeof(buf));
                if(len == -1) {
                    perror("read");
                    exit(-1);
                } else if(len == 0) {
                    printf("client closed...\n");
                    epoll_ctl(epfd, EPOLL_CTL_DEL, curfd, NULL);
                    close(curfd);
                } else if(len > 0) {
                    printf("read buf = %s\n", buf);
                    write(curfd, buf, strlen(buf) + 1);
                }

            }

        }
    }

    close(lfd);
    close(epfd);
    return 0;
}  

//et
int main() {

    // 创建socket
    int lfd = socket(PF_INET, SOCK_STREAM, 0);
    struct sockaddr_in saddr;
    saddr.sin_port = htons(9999);
    saddr.sin_family = AF_INET;
    saddr.sin_addr.s_addr = INADDR_ANY;

    // 绑定
    bind(lfd, (struct sockaddr *)&saddr, sizeof(saddr));

    // 监听
    listen(lfd, 8);

    // 调用epoll_create()创建一个epoll实例
    int epfd = epoll_create(100);

    // 将监听的文件描述符相关的检测信息添加到epoll实例中
    struct epoll_event epev;
    epev.events = EPOLLIN;
    epev.data.fd = lfd;
    epoll_ctl(epfd, EPOLL_CTL_ADD, lfd, &epev);

    struct epoll_event epevs[1024];

    while(1) {

        int ret = epoll_wait(epfd, epevs, 1024, -1);
        if(ret == -1) {
            perror("epoll_wait");
            exit(-1);
        }

        printf("ret = %d\n", ret);

        for(int i = 0; i < ret; i++) {

            int curfd = epevs[i].data.fd;

            if(curfd == lfd) {
                // 监听的文件描述符有数据达到，有客户端连接
                struct sockaddr_in cliaddr;
                int len = sizeof(cliaddr);
                int cfd = accept(lfd, (struct sockaddr *)&cliaddr, &len);

                // 设置cfd属性非阻塞
                int flag = fcntl(cfd, F_GETFL);
                flag | O_NONBLOCK;
                fcntl(cfd, F_SETFL, flag);

                epev.events = EPOLLIN | EPOLLET;    // 设置边沿触发
                epev.data.fd = cfd;
                epoll_ctl(epfd, EPOLL_CTL_ADD, cfd, &epev);
            } else {
                if(epevs[i].events & EPOLLOUT) {
                    continue;
                }  

                // 循环读取出所有数据
                char buf[5];
                int len = 0;
                while( (len = read(curfd, buf, sizeof(buf))) > 0) {
                    // 打印数据
                    // printf("recv data : %s\n", buf);
                    write(STDOUT_FILENO, buf, len);
                    write(curfd, buf, len);
                }
                if(len == 0) {
                    printf("client closed....");
                }else if(len == -1) {
                    if(errno == EAGAIN) {
                        printf("data over.....");
                    }else {
                        perror("read");
                        exit(-1);
                    }
                    
                }

            }

        }
    }

    close(lfd);
    close(epfd);
    return 0;
}

```  
# udp  
```cpp 
//客户端
int main() {

    // 1.创建一个通信的socket
    int fd = socket(PF_INET, SOCK_DGRAM, 0);
    
    if(fd == -1) {
        perror("socket");
        exit(-1);
    }   

    // 服务器的地址信息
    struct sockaddr_in saddr;
    saddr.sin_family = AF_INET;
    saddr.sin_port = htons(9999);
    inet_pton(AF_INET, "127.0.0.1", &saddr.sin_addr.s_addr);

    int num = 0;
    // 3.通信
    while(1) {

        // 发送数据
        char sendBuf[128];
        sprintf(sendBuf, "hello , i am client %d \n", num++);
        sendto(fd, sendBuf, strlen(sendBuf) + 1, 0, (struct sockaddr *)&saddr, sizeof(saddr));

        // 接收数据
        int num = recvfrom(fd, sendBuf, sizeof(sendBuf), 0, NULL, NULL);
        printf("server say : %s\n", sendBuf);

        sleep(1);
    }

    close(fd);
    return 0;
}  

//服务端
int main() {

    // 1.创建一个通信的socket
    int fd = socket(PF_INET, SOCK_DGRAM, 0);
    
    if(fd == -1) {
        perror("socket");
        exit(-1);
    }   

    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(9999);
    addr.sin_addr.s_addr = INADDR_ANY;

    // 2.绑定
    int ret = bind(fd, (struct sockaddr *)&addr, sizeof(addr));
    if(ret == -1) {
        perror("bind");
        exit(-1);
    }

    // 3.通信
    while(1) {
        char recvbuf[128];
        char ipbuf[16];

        struct sockaddr_in cliaddr;
        int len = sizeof(cliaddr);

        // 接收数据
        int num = recvfrom(fd, recvbuf, sizeof(recvbuf), 0, (struct sockaddr *)&cliaddr, &len);

        printf("client IP : %s, Port : %d\n", 
            inet_ntop(AF_INET, &cliaddr.sin_addr.s_addr, ipbuf, sizeof(ipbuf)),
            ntohs(cliaddr.sin_port));

        printf("client say : %s\n", recvbuf);

        // 发送数据
        sendto(fd, recvbuf, strlen(recvbuf) + 1, 0, (struct sockaddr *)&cliaddr, sizeof(cliaddr));

    }

    close(fd);
    return 0;
}
```  
# 广播  
```cpp
//客户端
int main() {

    // 1.创建一个通信的socket
    int fd = socket(PF_INET, SOCK_DGRAM, 0);
    if(fd == -1) {
        perror("socket");
        exit(-1);
    }   

    struct in_addr in;

    // 2.客户端绑定本地的IP和端口
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(9999);
    addr.sin_addr.s_addr = INADDR_ANY;

    int ret = bind(fd, (struct sockaddr *)&addr, sizeof(addr));
    if(ret == -1) {
        perror("bind");
        exit(-1);
    }

    // 3.通信
    while(1) {
        
        char buf[128];
        // 接收数据
        int num = recvfrom(fd, buf, sizeof(buf), 0, NULL, NULL);
        printf("server say : %s\n", buf);

    }

    close(fd);
    return 0;
}

//服务端
int main() {

    // 1.创建一个通信的socket
    int fd = socket(PF_INET, SOCK_DGRAM, 0);
    if(fd == -1) {
        perror("socket");
        exit(-1);
    }   

    // 2.设置广播属性
    int op = 1;
    setsockopt(fd, SOL_SOCKET, SO_BROADCAST, &op, sizeof(op));
    
    // 3.创建一个广播的地址
    struct sockaddr_in cliaddr;
    cliaddr.sin_family = AF_INET;
    cliaddr.sin_port = htons(9999);
    inet_pton(AF_INET, "192.168.193.255", &cliaddr.sin_addr.s_addr);

    // 3.通信
    int num = 0;
    while(1) {
       
        char sendBuf[128];
        sprintf(sendBuf, "hello, client....%d\n", num++);
        // 发送数据
        sendto(fd, sendBuf, strlen(sendBuf) + 1, 0, (struct sockaddr *)&cliaddr, sizeof(cliaddr));
        printf("广播的数据：%s\n", sendBuf);
        sleep(1);
    }

    close(fd);
    return 0;
}
```  
# 组播  
```cpp
//客户端  
int main() {

    // 1.创建一个通信的socket
    int fd = socket(PF_INET, SOCK_DGRAM, 0);
    if(fd == -1) {
        perror("socket");
        exit(-1);
    }   

    struct in_addr in;
    // 2.客户端绑定本地的IP和端口
    struct sockaddr_in addr;
    addr.sin_family = AF_INET;
    addr.sin_port = htons(9999);
    addr.sin_addr.s_addr = INADDR_ANY;

    int ret = bind(fd, (struct sockaddr *)&addr, sizeof(addr));
    if(ret == -1) {
        perror("bind");
        exit(-1);
    }

    struct ip_mreq op;
    inet_pton(AF_INET, "239.0.0.10", &op.imr_multiaddr.s_addr);
    op.imr_interface.s_addr = INADDR_ANY;

    // 加入到多播组
    setsockopt(fd, IPPROTO_IP, IP_ADD_MEMBERSHIP, &op, sizeof(op));

    // 3.通信
    while(1) {
        
        char buf[128];
        // 接收数据
        int num = recvfrom(fd, buf, sizeof(buf), 0, NULL, NULL);
        printf("server say : %s\n", buf);

    }

    close(fd);
    return 0;
}

//服务端
int main() {

    // 1.创建一个通信的socket
    int fd = socket(PF_INET, SOCK_DGRAM, 0);
    if(fd == -1) {
        perror("socket");
        exit(-1);
    }   

    // 2.设置多播的属性，设置外出接口
    struct in_addr imr_multiaddr;
    // 初始化多播地址
    inet_pton(AF_INET, "239.0.0.10", &imr_multiaddr.s_addr);
    setsockopt(fd, IPPROTO_IP, IP_MULTICAST_IF, &imr_multiaddr, sizeof(imr_multiaddr));
    
    // 3.初始化客户端的地址信息
    struct sockaddr_in cliaddr;
    cliaddr.sin_family = AF_INET;
    cliaddr.sin_port = htons(9999);
    inet_pton(AF_INET, "239.0.0.10", &cliaddr.sin_addr.s_addr);

    // 3.通信
    int num = 0;
    while(1) {
       
        char sendBuf[128];
        sprintf(sendBuf, "hello, client....%d\n", num++);
        // 发送数据
        sendto(fd, sendBuf, strlen(sendBuf) + 1, 0, (struct sockaddr *)&cliaddr, sizeof(cliaddr));
        printf("组播的数据：%s\n", sendBuf);
        sleep(1);
    }

    close(fd);
    return 0;
}
```  
# 本地套接字通信  
memcpy(seraddr.sun_path, "server.sock");//char数组为多个字符修改需要strcpy
```cpp
//客户端
int main() {

    unlink("client.sock");

    // 1.创建套接字
    int cfd = socket(AF_LOCAL, SOCK_STREAM, 0);
    if(cfd == -1) {
        perror("socket");
        exit(-1);
    }

    // 2.绑定本地套接字文件
    struct sockaddr_un addr;
    addr.sun_family = AF_LOCAL;
    strcpy(addr.sun_path, "client.sock");
    int ret = bind(cfd, (struct sockaddr *)&addr, sizeof(addr));
    if(ret == -1) {
        perror("bind");
        exit(-1);
    }

    // 3.连接服务器
    struct sockaddr_un seraddr;
    seraddr.sun_family = AF_LOCAL;
    strcpy(seraddr.sun_path, "server.sock");
    ret = connect(cfd, (struct sockaddr *)&seraddr, sizeof(seraddr));
    if(ret == -1) {
        perror("connect");
        exit(-1);
    }

    // 4.通信
    int num = 0;
    while(1) {

        // 发送数据
        char buf[128];
        sprintf(buf, "hello, i am client %d\n", num++);
        send(cfd, buf, strlen(buf) + 1, 0);
        printf("client say : %s\n", buf);

        // 接收数据
        int len = recv(cfd, buf, sizeof(buf), 0);

        if(len == -1) {
            perror("recv");
            exit(-1);
        } else if(len == 0) {
            printf("server closed....\n");
            break;
        } else if(len > 0) {
            printf("server say : %s\n", buf);
        }

        sleep(1);

    }

    close(cfd);
    return 0;
}  

//服务端  
int main() {

    unlink("server.sock");

    // 1.创建监听的套接字
    int lfd = socket(AF_LOCAL, SOCK_STREAM, 0);
    if(lfd == -1) {
        perror("socket");
        exit(-1);
    }

    // 2.绑定本地套接字文件
    struct sockaddr_un addr;
    addr.sun_family = AF_LOCAL;
    strcpy(addr.sun_path, "server.sock");
    int ret = bind(lfd, (struct sockaddr *)&addr, sizeof(addr));
    if(ret == -1) {
        perror("bind");
        exit(-1);
    }

    // 3.监听
    ret = listen(lfd, 100);
    if(ret == -1) {
        perror("listen");
        exit(-1);
    }

    // 4.等待客户端连接
    struct sockaddr_un cliaddr;
    int len = sizeof(cliaddr);
    int cfd = accept(lfd, (struct sockaddr *)&cliaddr, &len);
    if(cfd == -1) {
        perror("accept");
        exit(-1);
    }

    printf("client socket filename: %s\n", cliaddr.sun_path);

    // 5.通信
    while(1) {

        char buf[128];
        int len = recv(cfd, buf, sizeof(buf), 0);

        if(len == -1) {
            perror("recv");
            exit(-1);
        } else if(len == 0) {
            printf("client closed....\n");
            break;
        } else if(len > 0) {
            printf("client say : %s\n", buf);
            send(cfd, buf, len, 0);
        }

    }

    close(cfd);
    close(lfd);

    return 0;
}
```

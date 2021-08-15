---
title: 基于epoll和getaddrinfo_a的异步dns解析
date: 2021-08-15 11:30:31
tags:
    - linux
categories: 网络编程
---

## 1.介绍
多线程程序往往代表了高性能、高并发。但是多线程程序也比较容易导致难以调试的bug。单线程程序逻辑简单，很容易编写bugfree的代码，所以单线程程序；多进程部署也是一个不错的方案。  
- 单线程程序中我们需要尽量的避免堵塞操作，因为只有一个线程，堵塞之后整个进程都不工作，很多同步操作都需要进行异步实现。  
 
`gethostbyname`的作用为获取域名对应的ip地址，是一个同步操作，容易发生堵塞。本文就linux下基于`epoll`和`getaddrinfo_a`实现异步dns解析做一个简单介绍。

<!-- more -->

## 2. 实现

### 2.1 全局变量
```cpp
#define perror_then_exit(err) perror(err); exit(1)

const std::string hostname = "www.baidu.com";

int g_epollfd = 0;
int g_signalfd = 0;
static struct gaicb* g_gaicb = NULL;
```

### 2.2 创建`epoll`和信号
```cpp
void create_epollfd() {
    cout << __func__ << endl;
    g_epollfd = epoll_create(10);
    if (g_epollfd == -1) {
        perror_then_exit("epoll_create");
    }
}

void create_signalfd() {
    cout << __func__ << endl;
    sigset_t sigs;
    sigemptyset(&sigs);
    if (sigaddset(&sigs, SIGRTMIN) != 0) {
        perror_then_exit("sigaddset");
    }
    if (pthread_sigmask(SIG_BLOCK, &sigs, NULL)) { // 屏蔽该信号是为了避免信号被默认处理，实时信号默认处理为结束进程。
        perror_then_exit("pthread_sigmask");
    }
    g_signalfd = signalfd(-1, &sigs, SFD_NONBLOCK | SFD_CLOEXEC);
    if (g_signalfd == -1) {
        perror_then_exit("signalfd");
    }
}

void add_signalfd_to_epoll() {
    cout << __func__ << endl;
    struct epoll_event event;

    memset(&event, 0, sizeof(event));
    event.events = EPOLLIN | EPOLLET;
    event.data.fd = g_signalfd;
    if (epoll_ctl(g_epollfd, EPOLL_CTL_ADD, g_signalfd, &event) < 0) {
        perror_then_exit("epoll_ctl");
    }
}
```

### 2.3 调用`getaddrinfo_a`发起dns解析
```cpp
void raise_dns() {
    cout << __func__ << endl;
    if (!g_gaicb) {
        g_gaicb = (gaicb*)calloc(1, sizeof(gaicb));
    }

    struct addrinfo* hint = (addrinfo*)calloc(1, sizeof(struct addrinfo));
    hint->ai_socktype = SOCK_STREAM;
    hint->ai_protocol = IPPROTO_TCP;
    hint->ai_family = AF_INET;
    hint->ai_flags = AI_V4MAPPED;

    struct sigevent sev;
    sev.sigev_notify = SIGEV_SIGNAL; // 解析成功之后通知方式设置为信号
    sev.sigev_signo = SIGRTMIN;               
    sev.sigev_value.sival_ptr = g_gaicb;

    g_gaicb->ar_name = hostname.c_str();
    g_gaicb->ar_request = hint;
    
    if (getaddrinfo_a(GAI_NOWAIT, &g_gaicb, 1, &sev)) {
        perror_then_exit("getaddrinfo_a");
    }
}
```

### 2.4 调用`epoll_wait`等待dns解析结束。
```cpp
void parse_dns() {
    cout << __func__ << endl;
    struct signalfd_siginfo ssi;
    ssize_t size = 0;
    size = read(g_signalfd, &ssi, sizeof(ssi));
    if (size != sizeof(ssi)) {
        perror_then_exit("read signal fd");
    }

    struct gaicb* gaicb = (struct gaicb *)ssi.ssi_ptr;

    struct addrinfo *ai = gaicb->ar_result;
    char  ip[20] = {0};
    inet_ntop(AF_INET, &(((struct sockaddr_in *)ai->ai_addr)->sin_addr), ip, sizeof(ip));
    cout << "hostname: " << hostname << endl;
    cout << "ip: " << ip << endl;

    freeaddrinfo(gaicb->ar_result);
    free(gaicb);
}

void wait_dns() {
    epoll_event events[10];

    cout << "enter epoll_wait" << endl;
    int num = epoll_wait(g_epollfd, events, 10, -1);
    cout << "epoll_wait return, active events num:" << num << endl;
    if (num < 0) {
        perror_then_exit("epoll_wait");
    }
    if (num != 1) {
        cout << "events num not equal 1, num=" << num << endl;
        exit(1);
    }
    parse_dns();
}
```

### 2.5 调用顺序
```cpp 
int main () {    
    create_epollfd();
    create_signalfd();
    add_signalfd_to_epoll();
    raise_dns();
    wait_dns();
    return 0;
}
```
最后结果为：
```
create_epollfd
create_signalfd
add_signalfd_to_epoll
raise_dns
enter epoll_wait
epoll_wait return, active events num:1
parse_dns
hostname: www.baidu.com
ip: 183.232.231.174
```

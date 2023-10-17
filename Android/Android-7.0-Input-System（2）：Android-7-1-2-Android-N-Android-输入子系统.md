---
title: Android Input System（2）：Android 7.1.2 (Android N) Android 输入子系统 - Input System分析
cover: https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/hexo.themes/bing-wallpaper-2018.04.05.jpg
categories: 
  - Input
tags:
  - Android
  - Input
toc: true
abbrlink: 20180308
date: 2018-03-08 09:25:00
---


Android 输入子系统概述:
● 当时输入设备（如触摸屏，键盘等）可用时，Linux Kernel会在/dev/input/下创建名为event0~eventN的设备节点; 当输入设备不可用时，会将相应的设备节点删除。
● 当用户操作输入设备时，Linux Kernel会收到相应的硬件中断，然后会将中断加工成原始输入事件（raw input event），并写入到设备节点中。而后在用户空间就可以通过read()函数读取事件数据了。
● Android输入系统会监控/dev/input/下的所有设备节点，当某个结点有数据可读时，将数据读出并进行一系列处理，然后在当前系统中的所有窗口（Window）中寻找合适的接收者，并把事件派发给它。
● 具体来说，Linux Kernel将raw input event写入到设备节点后，InputReader会通过EventHub将原始事件读取出来并翻译加工为Android输入事件，而后把它交给InputDispatcher。InputDispatcher根据WMS（WindowManagerService）提供的窗口信息将事件传递给合适的窗口，若窗口为壁纸/SurfaceView等，则到了终点；否则会由该Window的ViewRoot继续分发到合适的View。
<!-- more -->

本章涉及的源代码文件名及位置：

**frameworks/base/services/java/com/android/server/**
● SystemServer.java

**frameworks/base/services/java/com/android/server/input/**
● InputManagerService.java

**frameworks/base/services/java/com/android/server/wm/**
● WindowManagerService.java
● WindowState.java
● InputMonitor.java

**frameworks/base/core/java/android/view/**
● View.java
● ViewGroup.java
● InputEventReceiver.java
● ViewRootImpl.java
● IWindowSession.aidl
● InputChannel.java

**frameworks/base/core/java/android/app/**
● Activity.java

**frameworks/base/services/jni/**
● android_view_InputChannel.cpp
● android_view_InputEventReceiver.cpp
● com_android_server_input_InputManagerService.cpp

**frameworks/native/services/inputflinger/**
● InputManager.cpp
● EventHub.h
● EventHub.cpp
● InputReader.h
● InputReader.cpp
● InputListener.h
● InputListener.cpp
● InputDispatcher.h
● InputDispatcher.cpp

**frameworks/native/libs/input/**
● InputTransport.cpp

**/frameworks/native/include/input/**
● InputTransport.h


## 一、Input系统必备Linux知识

注：必备知识可稍后遇到实际使用的地方再做详细了解。

## （一）、必备的Linux知识 inotify和epoll

### 1、INotify介绍与使用

INotify是一个Linux内核所提供的一种文件系统变化通知机制。它可以为应用程序监控文件系统的变化，如文件的新建、删除、读写等。INotify机制有两个基本对象，分别为inotify对象与watch对象，都使用文件描述符表示。 inotify对象对应了一个队列，应用程序可以向inotify对象添加多个监听。当被监听的事件发生时，可以通过read()函数从inotify对象中将事件信息读取出来。Inotify对象可以通过以下方式创建：

```c
int inotifyFd = inotify_init();
```

而watch对象则用来描述文件系统的变化事件的监听。它是一个二元组，包括监听目标和事件掩码两个元素。监听目标是文件系统的一个路径，可以是文件也可以是文件夹。而事件掩码则表示了需要需要监听的事件类型，掩码中的每一位代表一种事件。可以监听的事件种类很多，其中就包括文件的创建(IN_CREATE)与删除(IN_DELETE)。读者可以参阅相关资料以了解其他可监听的事件种类。以下代码即可将一个用于监听输入设备节点的创建与删除的watch对象添加到inotify对象中：

```c
int wd = inotify_add_watch (inotifyFd, “/dev/input”,IN_CREATE | IN_DELETE);
```

完成上述watch对象的添加后，当/dev/input/下的设备节点发生创建与删除操作时，都会将相应的事件信息写入到inotifyFd所描述的inotify对象中，此时可以通过read()函数从inotifyFd描述符中将事件信息读取出来。 事件信息使用结构体inotify_event进行描述：

```c
struct inotify_event {
   __s32           wd;             /* 事件对应的Watch对象的描述符 */
   __u32           mask;           /* 事件类型，例如文件被删除，此处值为IN_DELETE */
   __u32           cookie;
   __u32           len;            /* name字段的长度 */
   char            name[0];        /* 可变长的字段，用于存储产生此事件的文件路径*/
   };
```

当没有监听事件发生时，可以通过如下方式将一个或多个未读取的事件信息读取出来：

```c
size_t len = read (inotifyFd, events_buf,BUF_LEN);
```

其中events_buf是inotify_event的数组指针，能够读取的事件数量由取决于数组的长度。成功读取事件信息后，便可根据inotify_event结构体的字段判断事件类型以及产生事件的文件路径了。

总结一下INotify机制的使用过程：

· 通过inotify_init()创建一个inotify对象。

· 通过inotify_add_watch将一个或多个监听添加到inotify对象中。

· 通过read()函数从inotify对象中读取监听事件。当没有新事件发生时，inotify对象中无任何可读数据。

通过INotify机制避免了轮询文件系统的麻烦，但是还有一个问题，INotify机制并不是通过回调的方式通知事件，而需要使用者主动从inotify对象中进行事件读取。那么何时才是读取的最佳时机呢？这就需要借助Linux的另一个优秀的机制Epoll了。

使用inotify监听目录实例：

```c
#include <unistd.h>
#include <stdio.h>
#include <sys/inotify.h>
#include <string.h>
#include <errno.h>
//参考: frameworks\native\services\inputflinger\EventHub.cpp
//Usage: inotify <dir>
int read_process_inotify_fd(int fd)
{
int res;
char event_buf[512];
int event_size;
int event_pos = 0;
struct inotify_event *event;
/* read */    
res = read(fd, event_buf, sizeof(event_buf));
if(res < (int)sizeof(*event)) {
    if(errno == EINTR) return 0;
    printf("could not get event, %s\n", strerror(errno));
    return -1;
}
//读到的数据是1个或多个inotify_event,它们的长度不一样,逐个处理
while(res >= (int)sizeof(*event)) {
    event = (struct inotify_event *)(event_buf + event_pos);
    //printf("%d: %08x \"%s\"\n", event->wd, event->mask, event->len ? event->name : "");
    if(event->len) {
        if(event->mask & IN_CREATE) {
            printf("create file: %s\n", event->name);
        } else {
            printf("delete file: %s\n", event->name);
        }
    }
    event_size = sizeof(*event) + event->len;
    res -= event_size;
    event_pos += event_size;
}
return 0;
}
int main(int argc, char **argv)
{
int mINotifyFd;
int result;
if (argc != 2)
{
    printf("Usage: %s <dir>\n", argv[0]); return -1;
}
/* inotify_init */
mINotifyFd = inotify_init();
/* add watch */
result = inotify_add_watch(mINotifyFd, argv[1], IN_DELETE | IN_CREATE);
/* read */
while (1)
{
    read_process_inotify_fd(mINotifyFd);
}
return 0;
}
```

**编译与验证：** gcc -o inotify inotify.c //GCC编译 mkdir tmp //创建tmp文件夹 ./inotify tmp & //后台监测tmp文件夹

echo > tmp/1 //tmp文件夹新建文件1 echo > tmp/2 //tmp文件夹新建文件2 rm tmp/1 tmp/2 //移除tmp文件1/2

测试结果可以看到，inotify 成功的监测了tmp文件夹。 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N1-Android-Input-System-Inotify-Test.jpg)


### 2、Epoll介绍与使用

无论是从设备节点中获取原始输入事件还是从inotify对象中读取文件系统事件，都面临一个问题，就是这些事件都是偶发的。也就是说，大部分情况下设备节点、inotify对象这些文件描述符中都是无数据可读的，同时又希望有事件到来时可以尽快地对事件作出反应。为解决这个问题，我们不希望不断地轮询这些描述符，也不希望为每个描述符创建一个单独的线程进行阻塞时的读取，因为这都将会导致资源的极大浪费。

此时最佳的办法是使用Epoll机制。Epoll可以使用一次等待监听多个描述符的可读/可写状态。等待返回时携带了可读的描述符或自定义的数据，使用者可以据此读取所需的数据后可以再次进入等待。因此不需要为每个描述符创建独立的线程进行阻塞读取，避免了资源浪费的同时又可以获得较快的响应速度。

Epoll机制的接口只有三个函数，十分简单。

· epoll_create(int max_fds)：创建一个epoll对象的描述符，之后对epoll的操作均使用这个描述符完成。max_fds参数表示了此epoll对象可以监听的描述符的最大数量。

· epoll_ctl (int epfd, int op,int fd, struct epoll_event *event)：用于管理注册事件的函数。这个函数可以增加/删除/修改事件的注册。

· int epoll_wait(int epfd, structepoll_event * events, int maxevents, int timeout)：用于等待事件的到来。当此函数返回时，events数组参数中将会包含产生事件的文件描述符。

接下来以监控若干描述符可读事件为例介绍一下epoll的用法。

（1） 创建epoll对象

首先通过epoll_create()函数创建一个epoll对象：

Int epfd = epoll_create(MAX_FDS)

（2） 填充epoll_event结构体

接着为每一个需监控的描述符填充epoll_event结构体，以描述监控事件，并通过epoll_ctl()函数将此描述符与epoll_event结构体注册进epoll对象。epoll_event结构体的定义如下:

```c
struct epoll_event {

__uint32_tevents; /* 事件掩码，指明了需要监听的事件种类*/
 epoll_data_t data; /* 使用者自定义的数据，当此事件发生时该数据将原封不动地返回给使用者 */
 };
```

epoll_data_t联合体的定义如下，当然，同一时间使用者只能使用一个字段：

```c
typedef union epoll_data {
void*ptr;
int fd;
__uint32_t u32;
__uint64_t u64;
} epoll_data_t;
```

epoll_event结构中的events字段是一个事件掩码，用以指明需要监听的事件种类，同INotify一样，掩码的每一位代表了一种事件。常用的事件有EPOLLIN（可读），EPOLLOUT（可写），EPOLLERR（描述符发生错误），EPOLLHUP（描述符被挂起）等。更多支持的事件读者可参考相关资料。

data字段是一个联合体，它让使用者可以将一些自定义数据加入到事件通知中，当此事件发生时，用户设置的data字段将会返回给使用者。在实际使用中常设置epoll_event.data.fd为需要监听的文件描述符，事件发生时便可以根据epoll_event.data.fd得知引发事件的描述符。当然也可以设置epoll_event.data.fd为其他便于识别的数据。

填充epoll_event的方法如下：

```c
   structepoll_event eventItem;

   memset(&eventItem, 0, sizeof(eventItem));

   eventItem.events = EPOLLIN | EPOLLERR | EPOLLHUP; // 监听描述符可读以及出错的事件

   eventItem.data.fd= listeningFd; // 填写自定义数据为需要监听的描述符
```

接下来就可以使用epoll_ctl()将事件注册进epoll对象了。epoll_ctl()的参数有四个：

· epfd是由epoll_create()函数所创建的epoll对象的描述符。

· op表示了何种操作，包括EPOLL_CTL_ADD/DEL/MOD三种，分别表示增加/删除/修改注册事件。

· fd表示了需要监听的描述符。

· event参数是描述了监听事件的详细信息的epoll_event结构体。

注册方法如下：

```c
// 将事件监听添加到epoll对象中去

result =epoll_ctl(epfd, EPOLL_CTL_ADD, listeningFd, &eventItem);
```

重复这个步骤可以将多个文件描述符的多种事件监听注册到epoll对象中。完成了监听的注册之后，便可以通过epoll_wait()函数等待事件的到来了。

（3） 使用epoll_wait()函数等待事件

epoll_wait()函数将会使调用者陷入等待状态，直到其注册的事件之一发生之后才会返回，并且携带了刚刚发生的事件的详细信息。其签名如下：

int epoll_wait(int epfd, struct epoll_event *events, int maxevents, int timeout);

· epfd是由epoll_create()函数所创建的epoll对象描述符。

· events是一个epoll_event的数组，此函数返回时，事件的信息将被填充至此。

· maxevents表示此次调用最多可以获取多少个事件，当然，events参数必须能够足够容纳这么多事件。

· timeout表示等待超时的事件。

epoll_wait()函数返回值表示获取了多少个事件。

（4） 处理事件

epoll_wait返回后，便可以根据events数组中所保存的所有epoll_event结构体的events字段与data字段识别事件的类型与来源。

Epoll的使用步骤总结如下：

· 通过epoll_create()创建一个epoll对象。

· 为需要监听的描述符填充epoll_events结构体，并使用epoll_ctl()注册到epoll对象中。

· 使用epoll_wait()等待事件的发生。

· 根据epoll_wait()返回的epoll_events结构体数组判断事件的类型与来源并进行处理。

· 继续使用epoll_wait()等待新事件的发生。

使用inotify监听目录实例：

```c
#include <sys/epoll.h>
#include <stdio.h>
#include <unistd.h>
#include <fcntl.h>
#include <string.h>
#if 0
typedef union epoll_data {
void        *ptr;
 int          fd;
  uint32_t     u32;
   uint64_t     u64;
   } epoll_data_t;
#endif
#define DATA_MAX_LEN 500
/* usage: epoll <file1> [file2] [file3] ... */
int add_to_epoll(int fd, int epollFd)
{
int result;
struct epoll_event eventItem;
memset(&eventItem, 0, sizeof(eventItem));
eventItem.events = EPOLLIN;
eventItem.data.fd = fd;
result = epoll_ctl(epollFd, EPOLL_CTL_ADD, fd, &eventItem);
return result;
}
void rm_from_epoll(int fd, int epollFd)
{
epoll_ctl(epollFd, EPOLL_CTL_DEL, fd, NULL);
}
int main(int argc, char **argv)
{
int mEpollFd;
int i;
char buf[DATA_MAX_LEN];
// Maximum number of signalled FDs to handle at a time.
static const int EPOLL_MAX_EVENTS = 16;
// The array of pending epoll events and the index of the next event to be handled.
struct epoll_event mPendingEventItems[EPOLL_MAX_EVENTS];
if (argc < 2)
{
    printf("Usage: %s <file1> [file2] [file3] ...\n", argv[0]);
    return -1;
}

/* epoll_create */
mEpollFd = epoll_create(8);

// for each file:* open it
// add it to epoll: epoll_ctl(...EPOLL_CTL_ADD...)
for (i = 1; i < argc; i++)     
{
    //int tmpFd = open(argv[i], O_RDONLY|O_NONBLOCK);
    int tmpFd = open(argv[i], O_RDWR);
    add_to_epoll(tmpFd, mEpollFd);
}
/* epoll_wait */
while (1)
{
    int pollResult = epoll_wait(mEpollFd, mPendingEventItems, EPOLL_MAX_EVENTS, -1);
    for (i = 0; i < pollResult; i++)
    {
        printf("Reason: 0x%x\n", mPendingEventItems[i].events);
        int len = read(mPendingEventItems[i].data.fd, buf, DATA_MAX_LEN);
        buf[len] = '\0';
        printf("get data: %s\n", buf);
        //sleep(3);
    }
}
return 0;
}
```

epoll , fifo : [o-rdwr-on-named-pipes-with-poll](http://stackoverflow.com/questions/15055065/o-rdwr-on-named-pipes-with-poll)

使用fifo是, 我们的epoll程序是reader echo aa > tmp/1 是writer a. 如果reader以 O_RDONLY|O_NONBLOCK打开FIFO文件, 当writer写入数据时, epoll_wait会立刻返回; 当writer关闭FIFO之后, reader再次调用epoll_wait, 它也会立刻返回(原因是EPPLLHUP, 描述符被挂断) b. 如果reader以 O_RDWR打开FIFO文件 当writer写入数据时, epoll_wait会立刻返回; 当writer关闭FIFO之后, reader再次调用epoll_wait, 它并不会立刻返回, 而是继续等待有数据

**编译与验证：** gcc -o epoll epoll.c //GCC编译 mkdir tmp //创建tmp文件夹 mkfifo tmp/1 tmp/2 tmp/3 //创建文件1、2、3 ./epoll tmp/1 tmp/2 tmp/3 & //epoll后台监测文件1、2、3 echo aaa > tmp/1 //写人aaa到1 echo bbb > tmp/2 //写入bbb到2

测试结果可以看到，epoll成功的监测了文件内容的改变。 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N2-Android-Input-System-epoll-test.jpg)

### 3、INotify与Epoll的小结

INotify与Epoll这两套由Linux提供的事件监听机制以最小的开销解决了文件系统变化以及文件描述符可读可写状态变化的监听问题。它们是Reader子系统运行的基石，了解了这两个机制的使用方法之后便为对Reader子系统的分析学习铺平了道路。 参考：<https://github.com/weidongshan/APP_0006_inotify_epoll> inotify_epoll.c, 用它来监测tmp/目录: 有文件被创建/删除, 有文件可读出数据 a. 当在tmp/下创建文件时, 会立刻监测到，并且使用epoll监测该文件 b. 当文件有数据时，读出数据 c. 当tmp/下文件被删除时，会立刻监测到，并且把它从epoll中移除不再监测

inotify_epoll.c **编译与验证：**

gcc -o inotify_epoll inotify_epoll.c mkdir tmp ./inotify_epoll tmp/ & mkfifo tmp/1 tmp/2 tmp/3 echo aaa > tmp/1 echo bbb > tmp/2 rm tmp/3

由实例可知，使用inotify 和 epoll 结合就可以监测文件增加和移除 ，还能监测文件内容的改变。

用途简介[稍后进行input system详细分析]： /dev/input 下有多个event文件,对应多个输入设备，如:/dev/input/event0, /dev/input/mouse0, /dev/input/misc 使用inotify 和 epoll 就可以监听输入设备的变化、如Android新连接一个鼠标可检测到改变。同时可监听是否有输入事件。

[Lnux IO模式及 select、poll、epoll详解](https://segmentfault.com/a/1190000003063859)

## （二）、必备Linux知识_双向通信(scoketpair)

### 1、进程和APP通信

· 创建进程 · 读取、分发 · 进程发送输入事件给APP · 进程读取APP回应的事件 · 输入系统涉及双向的进程间通信

### 2、回顾Binder系统

· Server-- 单向发出请求 · Client -- 单向回复请求 · 每次请求只可以单方发出

### 3、引入Socketpair

原因：如果创建两组进程（Client，Server）进行双向通信，实现十分复杂 引入Socketpair： Socketpair();两次，获得两个fd，在内核获得缓冲区，一个作为sendbuf区一个作为receivebuf区 APP通过fd1将数据写入fd1的sendbuf区中，通过内核当中的socket机制就会写到fd2中receivebuf区，同理fd2也是如此 socketpair缺点：只适用于线程间、父子进程通信 解决方法：通过Binder机制通信可以访问任意进程，就解决了sockpair缺点

### 4、socketpair具体使用

创建一个线程--pthread_create(); 创建socketpair--socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets); 线程处理函数--往socket[1]写入数据，读取socket[0]读取数据 主函数--从socket[1]读取数据，往socket[0]写入数据

```c
#include <pthread.h>
#include <unistd.h>
#include <stdio.h>
#include <sys/types.h>          /* See NOTES */
#include <sys/socket.h>
#define SOCKET_BUFFER_SIZE      (32768U)
#define MAX 512
/* 参考:frameworks/native/libs/input/InputTransport.cpp
   */
   /* 线程执行函数 */
int *function_thread(void *arg)
{
int thread1_fd = (int)arg;int cnt=0;
int len;
char buf[MAX];
while(1){
    /* 向 main线程发出: Hello, main thread   */
    len = sprintf(buf,"Hello , main thread , cnt = %d",cnt++);
    write(thread1_fd,buf,len);
    /* 读取数据(main线程发回的数据) */
    len = read(thread1_fd,buf,MAX);
    buf[len] = '\0';
    printf("thread1 read : %s\n",buf);
    sleep(5);
}
close(thread1_fd);
return NULL;
}
int main(int argc,char *argv[])
{
pthread_t threadID;
int sockets[2];
int bufferSize = SOCKET_BUFFER_SIZE;
socketpair(AF_UNIX,SOCK_SEQPACKET,0,sockets);  //创建socketpair
//初始化
setsockopt(sockets[0], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
setsockopt(sockets[0], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));
setsockopt(sockets[1], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
setsockopt(sockets[1], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));
pthread_create(threadID,NULL,function_thread,(void *)sockets[1]);  //创建线程
int mainThread_fd = sockets[0];
int cnt=0;
int len;
char buf[MAX];
while(1){
    /* 读数据: 线程1发出的数据 */
    len = read(mainThread_fd,buf,MAX);
    buf[len] = '\0';
    printf("main thread read : %s\n",buf);
    /* main thread向thread1 发出: Hello, thread1 */
    len = sprintf(buf,"Hello , thread1 , cnt = %d",cnt++);
    write(mainThread_fd,buf,len);       
}
close(mainThread_fd);
return 0;
}
```

使用方法： gcc socketpair.c -o socketpair -pthread 注：出现少量警告，可以忽略 ./socketpair 可以看到main线程 和 thread1双向通信。 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N3-Android-Input-System-socketpair.jpg)

main 和 thread1属于两个线程： 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N4-Android-Input-System-socketpair-thread.jpg)

父子进程通信： 利用socketpair创建一对无名管道，然后通过sendmsg由服务器进程发送文件的fd给客户端进程，客户端进程通过recvmsg接收服务器进程发来的fd [socketpair实现父子进程通信](http://blog.csdn.net/yankai0219/article/details/8453377) **图示：** 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N5-Android-Input-System-socketpair-father-son.jpg)

## （三）、必备Linux知识_实现任意进程间双向通信(scoketpair+binder)

代码实例，由于代码较多，请往GitHub上查看。 [实现任意进程间双向通信(scoketpair+binder)](https://github.com/weidongshan/APP_0004_Binder_CPP_App)

由第二节最后可知socketpair可实现父子进程通信，图中父进程和子进程可双向通信，假如此时通过binder通信将文件句柄Fd[1]传给另外一个独立进程，我们知道Linux一切皆文件，那个独立进程就可以对Fd[1]读写了，也就是说父进程 就可以和 那个独立进程双向通信了，具体实现请研究上面的代码。

测试： 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N6-Android-Input-System-socketpair-binder.jpg)

可以看到两个没有任何关系的进程使用socketpair实现了双向通信。

用途简介[稍后进行input system详细分析]： InputManagerService获取事件后需要发送给App，假如App进程关掉了，需要告知IMS，就不需要接受事件了。可以看到需要进程间相互通信，这就是scoketpair+binder实际作用。

--------------------------------------------------------------------------------

## 二、输入系统的总体架构

### （一）、输入子系统分层解析

输入子系统的系统架构如下图所示： 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N7-Android-Input-System--all-arc.png)

Android输入系统系统综述： Linux内核会在/dev/input/下创建对应的名为event0~n或其他名称的设备节点。而当输入设备不可用时，则会将对应的节点删除。在用户空间可以通过ioctl的方式从这些设备节点中获取其对应的输入设备的类型、厂商、描述等信息。

当用户操作输入设备时，Linux内核接收到相应的硬件中断，然后将中断加工成原始的输入事件数据并写入其对应的设备节点中，在用户空间可以通过read()函数将事件数据读出。

Android输入系统的工作原理概括来说，就是监控/dev/input/下的所有设备节点，当某个节点有数据可读时，将数据读出并进行一系列的翻译加工，然后在所有的窗口中寻找合适的事件接收者，并派发给它。

#### 1、输入子系统分层解析

● Hardware层 硬件层主要就是按键、触摸屏、Sensor等各种输入设备。

● Kernel层

Kernel 层对Input相关处理只做简单的介绍。 Kernel 层主要分为三层，如下：

Input 设备驱动层: 采集输入设备的数据信息，通过 Input Core 的 API 上报数据。

Input Core（核心层）：为事件处理层和设备驱动层提供接口API。 Event Handler（事件处理层）：通过核心层的API获取输入事件上报的数据，定义API与应用层交互。

Event Handler： Event Handler 层以通用的 evdev.c 为例来解析，上层和 Kernel 层的交互在此文件完成。

● Framework 层 Android系统中Framework 层负责管理输入事件的主要是InputManagerService（IMS）。它主要的任务就是从设备中读事件数据，然后将输入事件发送到焦点窗口中去，另外还需要让系统有机会来处理一些系统按键。显然，要完成这个工作，IMS需要与其它模块打交道，其中最主要的就是WMS和ViewRootImpl。主要的几个模块示意如下： 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N8-Android-Input-System-framwork-arc.png)

● App层

--------------------------------------------------------------------------------

WindowManagerService(WMS)是窗口管理服务，核心维护了一个有序的窗口堆栈。PhoneWindowManager(PWM)里有关于手机策略的实现，和输入相关的主要是对系统按键的处理。InputManagerService是输入管理服务，主要干活的是Native层的InputManager。InputManager中的InputReader负责使用EventHub从Input driver中拿事件，然后让InputMapper解析。接着传给InputDispatcher，InputDispatcher负责一方面将事件通过InputManager，InputMonitor一路传给PhoneWindowManager来做系统输入事件的处理，另一方面将这些事件传给焦点及监视窗口。NativeInputManager实现InputReaderPolicyInterface和InputDispatcherPolicyInterface接口，在Native层的InputManager和Java层的IMS间起到一个胶水层的作用。InputMonitor实现了WindowManagerCallbacks接口，起到了IMS到WMS的连接作用。App这边，ViewRootImpl相当于App端一个顶层View的Controller。这个顶层View在WMS中对应一个窗口，用WindowState描述。WindowState中有InputWindowHandle代表一个接收输入事件的窗口句柄。InputDispatcher中的mFocusedWindowHandle指示了焦点窗口的句柄。InputDispatcher管理了一坨连接（一个连接对应一个注册到WMS的窗口），通过这些个连接InputDispatcher可以直接将输入事件发往App端的焦点窗口。输入事件从Driver开始的处理过程大致如下： 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N9-Android-Input-System-app-arc.png)

事件发往App端后，就进入事件分发阶段，这里简单提下，不做详细分析。

附： Kernel 层生成三个路径及相关设备文件：

```c
# /sys/class/input/
event0  event11 event4 event7 input0  input11 input4 input7
event1  event2  event5 event8 input1  input2  input5 input8
event10 event3  event6 event9 input10 input3  input6 input9
# /dev/input
event0 event10 event2 event4 event6 event8
event1 event11 event3 event5 event7 event9
# /proc/bus/input  
devices handlers
# cat devices  查看总线上的已经注册上的输入设备
I: Bus=0019 Vendor=0000 Product=0000 Version=0000
N: Name="ACCDET"
P: Phys=
S: Sysfs=/devices/virtual/input/input0
U: Uniq=
H: Handlers=gpufreq_ib event0
B: PROP=0
B: EV=3
B: KEY=40 0 0 0 0 0 0 1000000000 c000001800000 0
...
I: Bus=0019 Vendor=0000 Product=0000 Version=0001
N: Name="fingerprint_key"
P: Phys=
S: Sysfs=/devices/virtual/input/input2
U: Uniq=
H: Handlers=gpufreq_ib event2
B: PROP=0
B: EV=3
B: KEY=2000100000000000 180001f 8000000000000000
...
cat handlers // 查看注册的handler
N: Number=0 Name=gpufreq_ib
N: Number=1 Name=evdev Minor=64
```

#### 2、getevent与sendevent工具

Android系统提供了getevent与sendevent两个工具供开发者从设备节点中直接读取输入事件或写入输入事件。

getevent监听输入设备节点的内容，当输入事件被写入到节点中时，getevent会将其读出并打印在屏幕上。由于getevent不会对事件数据做任何加工，因此其输出的内容是由内核提供的最原始的事件。其用法如下：

```c
adb shell getevent [-选项] [device_path]
```

其中device_path是可选参数，用以指明需要监听的设备节点路径。如果省略此参数，则监听所有设备节点的事件。

打开模拟器，执行adb shell getevent –t（-t参数表示打印事件的时间戳），并按一下电源键（不要松手），可以得到以下一条输出，输出的部分数值会因机型的不同而有所差异，但格式一致：

```c
[1262.443489] /dev/input/event0: 0001 0074 00000001
```

松开电源键时，又会产生以下一条输出：

```c
[1262.557130] /dev/input/event0: 0001 0074 00000000
```

这两条输出便是按下和抬起电源键时由内核生成的原始事件。注意其输出是十六进制的。每条数据有五项信息：产生事件时的时间戳（[ 1262.443489]），产生事件的设备节点（/dev/input/event0），事件类型（0001），事件代码（0074）以及事件的值（00000001）。其中时间戳、类型、代码、值便是原始事件的四项基本元素。除时间戳外，其他三项元素的实际意义依照设备类型及厂商的不同而有所区别。在本例中，类型0x01表示此事件为一条按键事件，代码0x74表示电源键的扫描码，值0x01表示按下，0x00则表示抬起。这两条原始数据被输入系统包装成两个KeyEvent对象，作为两个按键事件派发给Framework中感兴趣的模块或应用程序。

注意一条原始事件所包含的信息量是比较有限的。而在Android API中所使用的某些输入事件，如触摸屏点击/滑动，包含了很多的信息，如XY坐标，触摸点索引等，其实是输入系统整合了多个原始事件后的结果。这个过程将在5.2.4节中详细探讨。

为了对原始事件有一个感性的认识，读者可以在运行getevent的过程中尝试一下其他的输入操作，观察一下每种输入所对应的设备节点及四项元素的取值。

输入设备的节点不仅在用户空间可读，而且是可写的，因此可以将将原始事件写入到节点中，从而实现模拟用户输入的功能。sendevent工具的作用正是如此。其用法如下：

```c
sendevent <节点路径> <类型><代码> <值>
```

可以看出，sendevent的输入参数与getevent的输出是对应的，只不过sendevent的参数为十进制。电源键的代码0x74的十进制为116，因此可以通过快速执行如下两条命令实现点击电源键的效果：

```c
adb shell sendevent /dev/input/event0 1 116 1 #按下电源键

adb shell sendevent /dev/input/event0 1 116 0 #抬起电源键
```

执行完这两条命令后，可以看到设备进入了休眠或被唤醒，与按下实际的电源键的效果一模一样。另外，执行这两条命令的时间间隔便是用户按住电源键所保持的时间，所以如果执行第一条命令后迟迟不执行第二条，则会产生长按电源键的效果----关机对话框出现了。很有趣不是么？输入设备节点在用户空间可读可写的特性为自动化测试提供了一条高效的途径。[1]

现在，读者对输入设备节点以及原始事件有了直观的认识，接下来看一下Android输入系统的基本原理。

#### 3、Input driver模拟驱动

代码实例： [Input driver模拟驱动-作者韦东山](https://github.com/weidongshan/DRV_0004_InputEmulator/)

```c
/* 参考drivers\input\keyboard\gpio_keys.c */
#include <linux/module.h>
#include <linux/version.h>
#include <linux/init.h>
#include <linux/fs.h>
#include <linux/input.h>
static struct input_dev *input_emulator_dev;
static int input_emulator_init(void)
{
int i;

/* 1\. 分配一个input_dev结构体 */
input_emulator_dev = input_allocate_device();;

/* 2\. 设置 */
/* 2.1 能产生哪类事件 */
set_bit(EV_KEY, input_emulator_dev->evbit);
set_bit(EV_REP, input_emulator_dev->evbit);

/* 2.2 能产生所有的按键 */
for (i = 0; i < BITS_TO_LONGS(KEY_CNT); i++)
    input_emulator_dev->keybit[i] = ~0UL;

/* 2.3 为android构造一些设备信息 */
input_emulator_dev->name = "InputEmulatorFrom100ask.net";
input_emulator_dev->id.bustype = 1;
input_emulator_dev->id.vendor  = 0x1234;
input_emulator_dev->id.product = 0x5678;
input_emulator_dev->id.version = 1;

/* 3\. 注册 */
input_register_device(input_emulator_dev);
return 0;
}
static void input_emulator_exit(void)
{
input_unregister_device(input_emulator_dev);
input_free_device(input_emulator_dev);
}
module_init(input_emulator_init);
module_exit(input_emulator_exit);
MODULE_LICENSE("GPL");
```

测试: insmod InputEmulator.ko

sendevent /dev/input/event5 1 2 1 // 1 2 1 : EV_KEY, KEY_1, down sendevent /dev/input/event5 1 2 0 // 1 2 0 : EV_KEY, KEY_1, up sendevent /dev/input/event5 0 0 0 // sync

sendevent /dev/input/event5 1 3 1 // 1 3 1 : EV_KEY, KEY_2, down sendevent /dev/input/event5 1 3 0 // 1 3 0 : EV_KEY, KEY_1, up sendevent /dev/input/event5 0 0 0 // sync 通过sendevent 最后会成功输入字符1、2。

--------------------------------------------------------------------------------

# 三、Android Input系统

## （一）、Android Input 系统关键类介绍

上一节讲述了输入事件的源头是位于/dev/input/下的设备节点，而输入系统的终点是由WMS管理的某个窗口。最初的输入事件为内核生成的原始事件，而最终交付给窗口的则是KeyEvent或MotionEvent对象。因此Android输入系统的主要工作是读取设备节点中的原始事件，将其加工封装，然后派发给一个特定的窗口以及窗口中的控件。这个过程由InputManagerService（以下简称IMS）系统服务为核心的多个参与者共同完成。

输入系统的总体流程和参与者如图3-1所示。 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N10-Android-Input-System-framwork-arc.png)

上图描述了输入事件的处理流程以及输入系统中最基本的参与者。它们是：

· **Linux内核**，接受输入设备的中断，并将原始事件的数据写入到设备节点中。

· **设备节点**，作为内核与IMS的桥梁，它将原始事件的数据暴露给用户空间，以便IMS可以从中读取事件。

· **InputManagerService**，一个Android系统服务，它分为Java层和Native层两部分。Java层负责与WMS的通信。而Native层则是InputReader和InputDispatcher两个输入系统关键组件的运行容器。

· **EventHub**，直接访问所有的设备节点。并且正如其名字所描述的，它通过一个名为getEvents()的函数将所有输入系统相关的待处理的底层事件返回给使用者。这些事件包括原始输入事件、设备节点的增删等。

· **InputReader**，I是IMS中的关键组件之一。它运行于一个独立的线程中，负责管理输入设备的列表与配置，以及进行输入事件的加工处理。它通过其线程循环不断地通过getEvents()函数从EventHub中将事件取出并进行处理。对于设备节点的增删事件，它会更新输入设备列表于配置。对于原始输入事件，InputReader对其进行翻译、组装、封装为包含了更多信息、更具可读性的输入事件，然后交给InputDispatcher进行派发。

· **InputReaderPolicy**，它为InputReader的事件加工处理提供一些策略配置，例如键盘布局信息等。

· **InputDispatcher**，是IMS中的另一个关键组件。它也运行于一个独立的线程中。InputDispatcher中保管了来自WMS的所有窗口的信息，其收到来自InputReader的输入事件后，会在其保管的窗口中寻找合适的窗口，并将事件派发给此窗口。

· **InputDispatcherPolicy**，它为InputDispatcher的派发过程提供策略控制。例如截取某些特定的输入事件用作特殊用途，或者阻止将某些事件派发给目标窗口。一个典型的例子就是HOME键被InputDispatcherPolicy截取到PhoneWindowManager中进行处理，并阻止窗口收到HOME键按下的事件。

· **WMS**，虽说不是输入系统中的一员，但是它却对InputDispatcher的正常工作起到了至关重要的作用。当新建窗口时，WMS为新窗口和IMS创建了事件传递所用的通道。另外，WMS还将所有窗口的信息，包括窗口的可点击区域，焦点窗口等信息，实时地更新到IMS的InputDispatcher中，使得InputDispatcher可以正确地将事件派发到指定的窗口。

· **ViewRootImpl**，对于某些窗口，如壁纸窗口、SurfaceView的窗口来说，窗口即是输入事件派发的终点。而对于其他的如Activity、对话框等使用了Android控件系统的窗口来说，输入事件的终点是控件（View）。ViewRootImpl将窗口所接收到的输入事件沿着控件树将事件派发给感兴趣的控件。

简单来说，内核将原始事件写入到设备节点中，InputReader不断地通过EventHub将原始事件取出来并翻译加工成Android输入事件，然后交给InputDispatcher。InputDispatcher根据WMS提供的窗口信息将事件交给合适的窗口。窗口的ViewRootImpl对象再沿着控件树将事件派发给感兴趣的控件。控件对其收到的事件作出响应，更新自己的画面、执行特定的动作。所有这些参与者以IMS为核心，构建了Android庞大而复杂的输入体系。

接下来详细讨论除Linux内核以外的其他参与者的工作原理。

## （二）、IMS的创建与启动

IMS分为Java层与Native层两个部分，其启动过程是从Java部分的初始化开始，进而完成Native部分的初始化。 IMS在SystemServer.startOtherServices()方法中启动的。IMS的诞生分为两个阶段：

· 创建新的IMS对象。

· 调用IMS对象的start()函数完成启动。

我们先看下整个启动过程的序列图，然后根据序列图来一步步分析。 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N11-Android-Input-System-input-create-thread.png)

## Step 1、 SystemServer.startOtherServices()

```java
    [->frameworks/base/services/java/com/android/server/SystemServer.java]

    private void startOtherServices() {
    ......
    try {
        ......
          // ① 新建IMS对象。
        traceBeginAndSlog("StartInputManagerService");
        inputManager = new InputManagerService(context);
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);

        traceBeginAndSlog("StartWindowManagerService");
        wm = WindowManagerService.main(context, inputManager,
                mFactoryTestMode != FactoryTest.FACTORY_TEST_LOW_LEVEL,
                !mFirstBoot, mOnlyCore);
         //将WindowManagerService加入到ServiceManager中
        ServiceManager.addService(Context.WINDOW_SERVICE, wm);
        //将InputManagerService加入到ServiceManager中
        ServiceManager.addService(Context.INPUT_SERVICE, inputManager);
        Trace.traceEnd(Trace.TRACE_TAG_SYSTEM_SERVER);

        mActivityManagerService.setWindowManager(wm);
         // 设置向WMS发起回调的callback对象
        inputManager.setWindowManagerCallbacks(wm.getInputMonitor());
        // ② 正式启动IMS
        inputManager.start();
        }
   }
```

在SystemServer中先构造了一个InputManagerService对象和一个WindowManagerService对象，然后将InputManagerService对象传给WindowManagerService对象，WindowManagerService中初始化了一个InputMonitor对象，调用InputManagerService.setWindowManagerCallbacks函数将InputMonitor传进去，后面native层回调时会调用到该InputMonitor对象。

## Step 2、 InputManagerService()

```java
    [->frameworks/base/services/core/java/com/android/server/input/InputManagerService.java]

    public InputManagerService(Context context) {
    this.mContext = context;
    //注意这里拿了DisplayThread的Handler，意味着IMS中的消息队列处理都是在单独的DisplayThread中进行的。
    //它是系统中共享的单例前台线程，主要用作输入输出的处理用。这样可以使用户体验敏感的处理少受其它工作的影响，减少延时。
    this.mHandler = new InputManagerHandler(DisplayThread.get().getLooper());
    //调用nativeInit来执行C++层的初始化操作
    mPtr = nativeInit(this, mContext, mHandler.getLooper().getQueue());
    LocalServices.addService(InputManagerInternal.class, new LocalService());
}
```

## Step 3、 InputManagerService.nativeInit()

```c
[->frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp]

static jlong nativeInit(JNIEnv* env, jclass /* clazz */,
    jobject serviceObj, jobject contextObj, jobject messageQueueObj) {
sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);
......
    // 新建了一个NativeInputManager对象，NativeInputManager，此对象将是Native层组件与
    //Java层IMS进行通信的桥梁
NativeInputManager* im = new NativeInputManager(contextObj, serviceObj,
        messageQueue->getLooper());
im->incStrong(0);
// 返回了NativeInputManager对象的指针给Java层的IMS，IMS将其保存在mPtr成员变量中
return reinterpret_cast<jlong>(im);
}
```

这个函数主要作用是创建一个NativeInputManager实例，并将其作为返回值保存在InputManagerService.java中的mPtr字段中。

## Step 4、NativeInputManager()

```c
[->frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp]

NativeInputManager::NativeInputManager(jobject contextObj,
    jobject serviceObj, const sp<Looper>& looper) :
    mLooper(looper), mInteractive(true) {
// 出现重点了， NativeInputManager创建了EventHub
//构造一个EventHub对象,最原始的输入事件都是通过它收集并且粗加工然后给到InputReader对象
sp<EventHub> eventHub = new EventHub();
// 接着创建了Native层的InputManager
mInputManager = new InputManager(eventHub, this, this);
}
```

NativeInputManager构造函数中创建了一个EventHub实例（稍后会详细介绍），并且将这个实例作为参数来创建一个InputManager对象，这个对象会做一些初始化的操作。

## Step 5、InputManager()

```c
[->frameworks/native/services/inputflinger/InputManager.cpp]

InputManager::InputManager(
    const sp<EventHubInterface>& eventHub,   
    const sp<InputReaderPolicyInterface>& readerPolicy,
    const sp<InputDispatcherPolicyInterface>& dispatcherPolicy) {
mDispatcher = new InputDispatcher(dispatcherPolicy);
mReader = new InputReader(eventHub, readerPolicy, mDispatcher);
initialize();
}
```

这里创建了InputDispatcher对象用于分发按键给当前focus的窗口的，同时创建了一个InputReader用于从EventHub中读取事件。

## Step 6、InputManager.initialize()

```c
[->frameworks/native/services/inputflinger/InputManager.cpp]

void InputManager::initialize() {
 // 创建供InputReader运行的线程InputReaderThread
mReaderThread = new InputReaderThread(mReader);
  // 创建供InputDispatcher运行的线程InputDispatcherThread
mDispatcherThread = new InputDispatcherThread(mDispatcher);
}
```

这里创建了一个InputReaderThread和InputDispatcherThread对象，前面构造函数中创建的InputReader实际上是通过InputReaderThread来读取事件，而InputDispatcher实际通过InputDispatcherThread来分发事件

### 图3-1：

![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N12-Android-Input-System-IMS-system.png)

InputManager的构造函数也比较简洁，它创建了四个对象，分别为IMS的核心参与者InputReader与InputDispatcher，以及它们所在的线程InputReaderThread与InputDispatcherThread。注意InputManager的构造函数的参数readerPolicy与dispatcherPolicy，它们都是NativeInputManager。

至此，IMS的创建完成了。在这个过程中，输入系统的重要参与者均完成创建，并得到了如图3-1所描述的一套体系。

依次初始化NativeInputManager，EventHub，InputManager, InputDispatcher，InputReader，InputReaderThread, InputDispatcherThread。NativeInputManager可看作IMS和InputManager的中间层，将IMS的请求转化为对InputManager及其内部对象的操作，同时将InputManager中模块的请求通过JNI调回IMS。InputManager是输入控制中心，它有两个关键线程InputReaderThread和InputDispatcherThread，它们的主要功能部分分别在InputReader和InputDispacher。前者用于从设备中读取事件，后者将事件分发给目标窗口。EventHub是输入设备的控制中心，它直接与input driver打交道。负责处理输入设备的增减，查询，输入事件的处理并向上层提供getEvents()接口接收事件。在它的构造函数中，主要做三件事（结合之前Linux必备知识）：

1. 创建epoll对象，之后就可以把各输入设备的fd挂在上面多路等待输入事件。
2. 建立用于唤醒的pipe，把读端挂到epoll上，以后如果有设备参数的变化需要处理，而getEvents()又阻塞在设备上，就可以调用wake()在pipe的写端写入，就可以让线程从等待中返回。
3. 利用inotify机制监听/dev/input目录下的变更，如有则意味着设备的变化，需要处理。 因为事件的处理是流水线，需要InputReader先读事件，然后InputDispatcher才能进一步处理和分发。因此InputDispatcher需要监听InputReader。这里使用了Listener模式，InputDispacher作为InputReader构造函数的第三个参数，它实现InputListenerInterface接口。到了InputReader的构造函数中，将之包装成QueuedInputListener。QueuedInputListener中的成员变量mArgsQueue是一个缓冲队列，只有在flush()时，才会一次性通知InputDispatcher。QueuedInputListener应用了Command模式，它通过包装InputDispatcher(实现InputListenerInterface接口)，将事件的处理请求封装成NotifyArgs，使其有了缓冲执行的功能。

## IMS的成员关系

![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N13-Android-Input-System-IMS-membership.png)

## （三）、IMS启动

IMS启动主要是将前面创建的InputReaderThread和InputDispatcherThread启动起来

## Step 1、InputManagerService.start()

```java
[->frameworks/base/services/core/java/com/android/server/input/InputManagerService.java]

public void start() {
    Slog.i(TAG, "Starting input manager");
    nativeStart(mPtr);
    ...
}
```

该函数主要调用了nativeStart进入native层启动

## Step 2\. InputManagerService.nativeStart()

```c
[->frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp]

static void nativeStart(JNIEnv* env, jclass /* clazz */, jlong ptr) {
NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);

status_t result = im->getInputManager()->start();
}
```

进入native层InputManager的start函数

## Step 3、InputManager.start()

```c
[->frameworks/native/services/inputflinger/InputManager.cpp]

status_t InputManager::start() {
status_t result = mDispatcherThread->run("InputDispatcher", PRIORITY_URGENT_DISPLAY);
result = mReaderThread->run("InputReader", PRIORITY_URGENT_DISPLAY);
return OK;
}
```

这个函数实际启动了一个InputReaderThread和InputDispatcherThread来从读取和分发键盘消息，调用它们的run方法后，就会进入threadLoop函数中，只要threadLoop函数返回true，该函数就会循环执行。

InputReaderThread不断调用InputReader的pollOnce()->getEvents()函数来得到事件，这些事件可以是输入事件，也可以是由inotify监测到设备增减变更所触发的事件，稍后会详细介绍。

## Step 4、InputReaderThread.threadLoop()

```c
[->frameworks/native/services/inputflinger/InputReader.cpp]

bool InputReaderThread::threadLoop() {
mReader->loopOnce();
return true;
}
```

这里调用前面创建的InputReaderThread对象的loopOnce进行一次线程循环

## Step5、InputReaderThread.loopOnce()

```c
[->frameworks/native/services/inputflinger/InputReader.cpp]

 void InputReader::loopOnce() {
......
/* ① 通过EventHub抽取事件列表。读取的结果存储在参数mEventBuffer中，返回值表示事件的个数
   当EventHub中无事件可以抽取时，此函数的调用将会阻塞直到事件到来或者超时 */
size_tcount = mEventHub->getEvents(timeoutMillis,mEventBuffer, EVENT_BUFFER_SIZE);
{
   AutoMutex _l(mLock);
    ......
    if(count) {
       // ② 如果有抽得事件，则调用processEventsLocked()函数对事件进行加工处理
       processEventsLocked(mEventBuffer, count);
    }
    ......
}
......
/* ③ 发布事件。 processEventsLocked()函数在对事件进行加工处理之后，便将处理后的事件存储在
  mQueuedListener中。在循环的最后，通过调用flush()函数将所有事件交付给InputDispatcher */
  mQueuedListener->flush();
  }
```

InputReader的一次线程循环的工作思路比较清晰，一共三步：

· 首先从EventHub中抽取未处理的事件列表。这些事件分为两类，一类是从设备节点中读取的原始输入事件，另一类则是输入设备可用性变化事件，简称为设备事件。

· 通过processEventsLocked()对事件进行处理。对于设备事件，此函数对根据设备的可用性加载或移除设备对应的配置信息。对于原始输入事件，则在进行转译、封装与加工后将结果暂存到mQueuedListener中。

· 所有事件处理完毕后，调用mQueuedListener.flush()将所有暂存的输入事件一次性地交付给InputDispatcher。

这便是InputReader的总体工作流程。而我们接下来将详细讨论这三步的实现。

## Step 6、InputDispatcherThread.threadLoop()

InputDisptacher的主要任务是把收到的输入事件发送到PhoneWIndowManager或App端的焦点窗口上，稍后详细介绍。

```c
[->frameworks/native/services/inputflinger/InputDispatcher.cpp]

bool InputDispatcherThread::threadLoop() {
mDispatcher->dispatchOnce();
return true;
}
```

这里调用前面创建的InputDispatcher对象的dispatchOnce函数进行一次按键分发

## Step 7、InputDispatcher.dispatchOnce()

```c
[->frameworks/native/services/inputflinger/InputDispatcher.cpp]

void InputDispatcher::dispatchOnce() {
nsecs_t nextWakeupTime = LONG_LONG_MAX;
{ // acquire lock
    AutoMutex _l(mLock);
    mDispatcherIsAliveCondition.broadcast();
    if (!haveCommandsLocked()) {
        dispatchOnceInnerLocked(&nextWakeupTime);
    }
    if (runCommandsLockedInterruptible()) {
        nextWakeupTime = LONG_LONG_MIN;
    }
} // release lock
// Wait for callback or timeout or wake.  (make sure we round up, not down)
nsecs_t currentTime = now();
int timeoutMillis = toMillisecondTimeoutDelay(currentTime, nextWakeupTime);
mLooper->pollOnce(timeoutMillis);
}
```

上述函数主要是调用dispatchOnceInnerLocked来进行一次按键分发，当没有按键消息时会走到mLooper->pollOnce(timeoutMillis)；这个函数会进入睡眠状态，当有按键消息发生时该函数会返回，然后走到dispatchOnceInnerLocked函数。这里mLooper->pollOnce为何会睡眠涉及到Android的Handler机制[☺再总结☺]。

## 小结：

完成IMS的创建之后，InputManagerService.start()函数以启动IMS。InputManager的创建过程分别为InputReader与InputDispatcher创建了承载它们运行的线程，然而并未将这两个线程启动，因此IMS的各员大将仍处于待命状态。此时start()函数的功能就是启动这两个线程，使得InputReader于InputDispatcher开始工作。

当两个线程启动后，InputReader在其线程循环中不断地从EventHub中抽取原始输入事件，进行加工处理后将加工所得的事件放入InputDispatcher的派发发队列中。InputDispatcher则在其线程循环中将派发队列中的事件取出，查找合适的窗口，将事件写入到窗口的事件接收管道中。窗口事件接收线程的Looper从管道中将事件取出，交由事件处理函数进行事件响应。整个过程共有三个线程首尾相接，像三台水泵似的一层层地将事件交付给事件处理函数。如下图所示。

![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N14-Android-Input-System-input-event-pop.png)

InputManagerService.start()函数的作用，就像为Reader线程、Dispatcher线程这两台水泵按下开关，而Looper这台水泵在窗口创建时便已经处于运行状态了。自此，输入系统动力十足地开始运转，设备节点中的输入事件将被源源不断地抽取给事件处理者。

--------------------------------------------------------------------------------

# 四、深入理解EventHub

InputReaderThread继承自C++的Thread类，Thread类封装了pthread线程工具，提供了与Java层Thread类相似的API。c的Thread类提供了一个名为threadLoop()的纯虚函数，当线程开始运行后，将会在内建的线程循环中不断地调用threadLoop()，直到此函数返回false，则退出线程循环，从而结束线程。 InputReaderThread启动后，其线程循环将不断地执行InputReader.loopOnce()函数。因此这个loopOnce()函数作为线程循环的循环体包含了InputReader的所有工作。前面一小节 Step5\. InputReaderThread.loopOnce() 已经说到InputReaderThread一次线程循环。接下来详细说明EventHub。

· 首先从EventHub中抽取未处理的事件列表。这些事件分为两类，一类是从设备节点中读取的原始输入事件，另一类则是输入设备可用性变化事件，简称为设备事件。

```c
 [->frameworks/native/services/inputflinger/InputReader.cpp]

 void InputReader::loopOnce() {
......
/* ① 通过EventHub抽取事件列表。读取的结果存储在参数mEventBuffer中，返回值表示事件的个数
   当EventHub中无事件可以抽取时，此函数的调用将会阻塞直到事件到来或者超时 */
size_tcount = mEventHub->getEvents(timeoutMillis,mEventBuffer, EVENT_BUFFER_SIZE);
......
}
```

首先贴一张EventHub->getEvents()工作时序图，跟着时序图一步步介绍。 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N15-Android-Input-System-input-reader-thread.png)

## （1）、深入理解EventHub

### 1、设备节点监听的建立

```c
frameworks/native/services/inputflinger/EventHub.cpp
EventHub::EventHub(void) :
    mBuiltInKeyboardId(NO_BUILT_IN_KEYBOARD), mNextDeviceId(1), mControllerNumbers(),
    mOpeningDevices(0), mClosingDevices(0),
    mNeedToSendFinishedDeviceScan(false),
    mNeedToReopenDevices(false), mNeedToScanDevices(true),
    mPendingEventCount(0), mPendingEventIndex(0), mPendingINotify(false) {
acquire_wake_lock(PARTIAL_WAKE_LOCK, WAKE_LOCK_ID);

// ① 首先使用epoll_create()函数创建一个epoll对象。EPOLL_SIZE_HINT指定最大监听个数为8
//这个epoll对象将用来监听设备节点是否有数据可读（有无事件）
mEpollFd = epoll_create(EPOLL_SIZE_HINT);
LOG_ALWAYS_FATAL_IF(mEpollFd < 0, "Could not create epoll instance.  errno=%d", errno);

// ② 创建一个inotify对象。这个inotify对象将被用来监听设备节点的增删事件
mINotifyFd = inotify_init();

 //将存储设备节点的路径/dev/input作为监听对象添加到inotify对象中。当此文件夹下的设备节点
 //发生创建与删除事件时，都可以通过mINotifyFd读取事件的详细信息
 //static const char *DEVICE_PATH = "/dev/input";
int result = inotify_add_watch(mINotifyFd, DEVICE_PATH, IN_DELETE | IN_CREATE);
LOG_ALWAYS_FATAL_IF(result < 0, "Could not register INotify for %s.  errno=%d",
        DEVICE_PATH, errno);

//③ 接下来将mINotifyFd作为epoll的一个监控对象。当inotify事件到来时，epoll_wait()将
//立刻返回，EventHub便可从mINotifyFd中读取设备节点的增删信息，并作相应处理
struct epoll_event eventItem;
memset(&eventItem, 0, sizeof(eventItem));
eventItem.events = EPOLLIN;// 监听mINotifyFd可读
eventItem.data.u32 = EPOLL_ID_INOTIFY; // 注意这里并没有使用fd字段，而使用了自定义的值EPOLL_ID_INOTIFY
result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mINotifyFd, &eventItem);// 将对mINotifyFd的监听注册到epoll对象中
LOG_ALWAYS_FATAL_IF(result != 0, "Could not add INotify to epoll instance.  errno=%d", errno);

//在构造函数剩余的代码中，EventHub创建了一个名为wakeFds的匿名管道，并将管道读取端的描述符
//的可读事件注册到epoll对象中。因为InputReader在执行getEvents()时会因无事件而导致其线程
//阻塞在epoll_wait()的调用里，然而有时希望能够立刻唤醒InputReader线程使其处理一些请求。此
//时只需向wakeFds管道的写入端写入任意数据，此时读取端有数据可读，使得epoll_wait()得以返回
//从而达到唤醒InputReader线程的目的
int wakeFds[2];
result = pipe(wakeFds);
LOG_ALWAYS_FATAL_IF(result != 0, "Could not create wake pipe.  errno=%d", errno);

mWakeReadPipeFd = wakeFds[0];
mWakeWritePipeFd = wakeFds[1];

result = fcntl(mWakeReadPipeFd, F_SETFL, O_NONBLOCK);
LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake read pipe non-blocking.  errno=%d",
        errno);

result = fcntl(mWakeWritePipeFd, F_SETFL, O_NONBLOCK);
LOG_ALWAYS_FATAL_IF(result != 0, "Could not make wake write pipe non-blocking.  errno=%d",
        errno);

eventItem.data.u32 = EPOLL_ID_WAKE;
result = epoll_ctl(mEpollFd, EPOLL_CTL_ADD, mWakeReadPipeFd, &eventItem);
LOG_ALWAYS_FATAL_IF(result != 0, "Could not add wake read pipe to epoll instance.  errno=%d",
        errno);

int major, minor;
getLinuxRelease(&major, &minor);
// EPOLLWAKEUP was introduced in kernel 3.5
mUsingEpollWakeup = major > 3 || (major == 3 && minor >= 5);
}
```

EventHub的构造函数初识化了Epoll对象和INotify对象，分别监听原始输入事件与设备节点增删事件。同时将INotify对象的可读性事件也注册到Epoll中，因此EventHub可以像处理原始输入事件一样监听设备节点增删事件了。

构造函数同时也揭示了EventHub的监听工作分为设备节点和原始输入事件两个方面，接下来将深入探讨这两方面的内容。

### 2、getEvents()函数的工作方式

正如前文所述，InputReaderThread的线程循环为Reader子系统提供了运转的动力，EventHub的工作也是由它驱动的。InputReader::loopOnce()函数调用EventHub::getEvents()函数获取事件列表，所以这个getEvents()是EventHub运行的动力所在，几乎包含了EventHub的所有工作内容，因此首先要将getEvents()函数的工作方式搞清楚。 getEvents()函数的签名如下：

```c
size_t EventHub::getEvents(int timeoutMillis,RawEvent* buffer, size_t bufferSize)
```

此函数将尽可能多地读取设备增删事件与原始输入事件，将它们封装为RawEvent结构体，并放入buffer中供InputReader进行处理。RawEvent结构体的定义如下： [EventHub.h-->RawEvent]

```c
struct RawEvent {
    nsecs_t when;             /* 发生事件时的时间戳 */
    int32_t deviceId;        //产生事件的设备Id，它是由EventHub自行分配的，InputReader
                            //以根据它从EventHub中获取此设备的详细信息
    int32_t type;             /* 事件的类型 */
    int32_t code;             /* 事件代码 */
    int32_t value;            /* 事件值 */
    };
```

可以看出，RawEvent结构体与getevent工具的输出十分一致，包含了原始输入事件的四个基本元素，因此用RawEvent结构体表示原始输入事件是非常直观的。RawEvent同时也用来表示设备增删事件，为此，EventHub定义了三个特殊的事件类型DEVICE_ADD、DEVICE_REMOVED以及FINISHED_DEVICE_SCAN，用以与原始输入事件进行区别。

由于getEvents()函数较为复杂，为了给后续分析铺平道路，本节不讨论其细节，先通过伪代码理解此函数的结构与工作方式，在后续深入分析时思路才会比较清晰。

**getEvents()函数的本质就是读取并处理Epoll事件与INotify事件**

参考以下代码：

[EventHub.cpp-->EventHub::getEvents()]

```c
size_t EventHub::getEvents(int timeoutMillis,RawEvent* buffer, size_t bufferSize) {

// event指针指向了在buffer下一个可用于存储事件的RawEvent结构体。每存储一个事件，
// event指针都回向后偏移一个元素 */
RawEvent* event = buffer;

// capacity记录了buffer中剩余的元素数量。当capacity为0时，表示buffer已满，此时需要停
// 继续处理新事件，并将已处理的事件返回给调用者
size_tcapacity = bufferSize;

// 接下来的循环是getEvents()函数的主体。在这个循环中，会先将可用事件放入到buffer中并返回。
// 如果没有可用事件，则进入epoll_wait()等待事件的到来，epoll_wait()返回后会重新循环将可用
// 将新事件放入buffer
for (;;){
   nsecs_t now = systemTime(SYSTEM_TIME_MONOTONIC);
    /* ① 首先进行与设备相关的工作。某些情况下，如EventHub创建后第一次执行getEvents()函数  */
    /* 时，需要扫描/dev/input文件夹下的所有设备节点并将这些设备打开。另外，当设备节点的发生增  */
    /*  动作生时，会将设备事件存入到buffer中 */
    ......
    /* ② 处理未被InputReader取走的输入事件与设备事件。epoll_wait()所取出的epoll_event */
    /* 存储在mPendingEventItems中，mPendingEventCount指定了mPendingEventItems数组 */
    /* 所存储的事件个数。而mPendingEventIndex指定尚未处理的epoll_event的索引 */
   while (mPendingEventIndex < mPendingEventCount) {
       const struct epoll_event& eventItem =
                             mPendingEventItems[mPendingEventIndex++];

       /* 在这里分析每一个epoll_event，如果是表示设备节点可读，则读取原始事件并放置到buffer */
       /* 中。如果是表示mINotifyFd可读，则设置mPendingINotify为true，当InputReader */
       /* 将现有的输入事件都取出后读取mINotifyFd中的事件，并进行相应的设备加载与卸载操作。 */
       /* 另外，如果此epoll_event表示wakeFds的读取端有数据可读，则设置awake标志为true， */
       /* 无论此次getEvents()调用有无取到事件，都不会再次进行epoll_wait()进行事件等待 */
       ......
    }
    // ③ 如果mINotifyFd有数据可读，说明设备节点发生了增删操作
    if(mPendingINotify && mPendingEventIndex >= mPendingEventCount) {
       /* 读取mINotifyFd中的事件，同时对输入设备进行相应的加载与卸载操作。这个操作必须当 */
       /* InputReader将现有输入事件读取并处理完毕后才能进行，因为现有的输入事件可能来自需要 */
       /* 被卸载的输入设备，InputReader处理这些事件依赖于对应的设备信息 */
        ......
        deviceChanged= true;
    }
    // 设备节点增删操作发生时，则重新执行循环体，以便将设备变化的事件放入buffer中
    if(deviceChanged) {
       continue;
    }
    // 如果此次getEvents()调用成功获取了一些事件，或者要求唤醒InputReader，则退出循环并
    // 结束getEvents()的调用，使InputReader可以立刻对事件进行处理
    if(event != buffer || awoken) {
       break;
    }
    /* ④ 如果此次getEvents()调用没能获取事件，说明mPendingEventItems中没有事件可用， */
    /* 于是执行epoll_wait()函数等待新的事件到来，将结果存储到mPendingEventItems里，并重 */
    /* 置mPendingEventIndex为0 */
   mPendingEventIndex = 0;
   ......
   intpollResult = epoll_wait(mEpollFd, mPendingEventItems, EPOLL_MAX_EVENTS,timeoutMillis);
   ......
    mPendingEventCount= size_t(pollResult);
    // 从epoll_wait()中得到新的事件后，重新循环，对新事件进行处理
}
// 返回本次getEvents()调用所读取的事件数量
returnevent - buffer;
}
```

--------------------------------------------------------------------------------

getEvents()函数使用Epoll的核心是mPendingEventItems数组，它是一个事件池。getEvents()函数会优先从这个事件池获取epoll事件进行处理，并将读取相应的原始输入事件返回给调用者。当因为事件池枯竭而导致调用者无法获得任何事件时，会调用epoll_wait()函数等待新事件的到来，将事件池重新注满，然后再重新处理事件池中的Epoll事件。从这个意义来说，getEvents()函数的调用过程，就是消费epoll_wait()所产生的Epoll事件的过程。因此可以将从epoll_wait()的调用开始，到将Epoll事件消费完毕的过程称为EventHub的一个监听周期。依据每次epoll_wait()产生的Epoll事件的数量以及设备节点中原始输入事件的数量，一个监听周期包含一次或多次getEvents()调用。周期中的第一次调用会因为事件池枯竭而直接进入epoll_wait()，而周期中的最后一次调用一定会将最后的事件取走。

注意getEvents()采用事件池机制的根本原因是buffer的容量限制。由于一次epoll_wait()可能返回多个设备节点的可读事件，每个设备节点又有可能读取多条原始输入事件，一段时间内原始输入事件的数量可能大于buffer的容量。因此需要一个事件池以缓存因buffer容量不够而无法处理的epoll事件，以便在下次调用时可以将这些事件优先处理。这是缓冲区操作的一个常用技巧。

当有INotify事件可以从mINotifyFd中读取时，会产生一个epoll事件，EventHub便得知设备节点发生了增删操作。在getEvents()将事件池中的所有事件处理完毕后，便会从mINotifyFd中读取INotify事件，进行输入设备的加载/卸载操作，然后生成对应的RawEvent结构体并返回给调用者。

通过上述分析可以看到，getEvents()包含了原始输入事件读取、输入设备加载/卸载等操作。这几乎是EventHub的全部工作了。如果没有geEvents()的调用，EventHub将对输入事件、设备节点增删事件置若罔闻，因此可以将一次getEvents()调用理解为一次心跳，EventHub的核心功能都会在这次心跳中完成。

getEvents()的代码还揭示了另外一个信息：在一个监听周期内的设备增删事件与Epoll事件的优先级。设备事件的生成逻辑位于Epoll事件的处理之前，因此getEvents()将优先生成设备增删事件，完成所有设备增删事件的生成之前不会处理Epoll事件，也就是不会生成原始输入事件。

接下来我们将从设备管理与原始输入事件处理两个方面深入探讨EventHub。

### 3、输入设备管理

因为输入设备是输入事件的来源，并且决定了输入事件的含义，因此首先讨论EventHub的输入设备管理机制。

输入设备是一个可以为接收用户操作的硬件，内核会为每一个输入设备在/dev/input/下创建一个设备节点，而当输入设备不可用时（例如被拔出），将其设备节点删除。这个设备节点包含了输入设备的所有信息，包括名称、厂商、设备类型，设备的功能等。除了设备节点，某些输入设备还包含一些自定义配置，这些配置以键值对的形式存储在某个文件中。这些信息决定了Reader子系统如何加工原始输入事件。EventHub负责在设备节点可用时加载并维护这些信息，并在设备节点被删除时将其移除。

EventHub通过一个定义在其内部的名为Device的私有结构体来描述一个输入设备。其定义如下：

[EventHub.h-->EventHub::Device]

```c
struct Device {
Device* next;  /* Device结构体实际上是一个单链表 */
int fd;         /* fd表示此设备的设备节点的描述符，可以从此描述符中读取原始输入事件 */
const int32_t id;     /* id在输入系统中唯一标识这个设备，由EventHub在加载设备时进行分配 */
const String8 path; /* path存储了设备节点在文件系统中的路径 */
const InputDeviceIdentifier identifier; /* 厂商信息，存储了设备的供应商、型号等信息
                                                   这些信息从设备节点中获得 */
uint32_t classes;  /* classes表示了设备的类别，键盘设备，触控设备等。一个设备可以同时属于
                          多个设备类别。类别决定了InputReader如何加工其原始输入事件 */
/* 接下来是一系列的事件位掩码，它们详细地描述了设备能够产生的事件类型。设备能够产生的事件类型
   决定了此设备所属的类型*/
uint8_t keyBitmask[(KEY_MAX + 1) / 8];
......
/* 配置信息。以键值对的形式存储在一个文件中，其路径取决于identfier字段中的厂商信息，这些
   配置信息将会影响InputReader对此设备的事件的加工行为 */
String8 configurationFile;
PropertyMap* configuration;
/* 键盘映射表。对于键盘类型的设备，这些键盘映射表将原始事件中的键盘扫描码转换为Android定义的
  的按键值。这个映射表也是从一个文件中加载的，文件路径取决于dentifier字段中的厂商信息 */
   VirtualKeyMap* virtualKeyMap;
KeyMap keyMap;
 sp<KeyCharacterMap> overlayKeyMap;
 sp<KeyCharacterMap> combinedKeyMap;
// 力反馈相关的信息。有些设备如高级的游戏手柄支持力反馈功能，目前暂不考虑
bool ffEffectPlaying;
int16_t ffEffectId;
};
```

Device结构体所存储的信息主要包括以下几个方面：

· 设备节点信息：保存了输入设备节点的文件描述符、文件路径等。

· 厂商信息：包括供应商、设备型号、名称等信息，这些信息决定了加载配置文件与键盘映射表的路径。

· 设备特性信息：包括设备的类别，可以上报的事件种类等。这些特性信息直接影响了InputReader对其所产生的事件的加工处理方式。

· 设备的配置信息：包括键盘映射表及其他自定义的信息，从特定位置的配置文件中读取。

另外，Device结构体还存储了力反馈所需的一些数据。在本节中暂不讨论。

EventHub用一个名为mDevices的字典保存当前处于打开状态的设备节点的Device结构体。字典的键为设备Id。

### （1）、输入设备的加载

EventHub在创建后在第一次调用getEvents()函数时完成对系统中现有输入设备的加载。

再看一下getEvents()函数中相关内容的实现：

[EventHub.cpp-->EventHub::getEvents()]

```c
size_t EventHub::getEvents(int timeoutMillis,RawEvent* buffer, size_t bufferSize) {
for (;;){
    // 处理输入设备卸载操作
   ......
    /* 在EventHub的构造函数中，mNeedToScanDevices被设置为true，因此创建后第一次调用
      getEvents()函数会执行scanDevicesLocked()，加载所有输入设备 */
    if(mNeedToScanDevices) {
       mNeedToScanDevices = false;
        /*scanDevicesLocked()将会把/dev/input下所有可用的输入设备打开并存储到Device
           结构体中 */
       scanDevicesLocked();
       mNeedToSendFinishedDeviceScan = true;
    }
   ......
}
returnevent – buffer;
}
```

加载所有输入设备由scanDevicesLocked()函数完成。看一下其实现：

[EventHub.cpp-->EventHub::scanDevicesLocked()]

```c
void EventHub::scanDevicesLocked() {
// 调用scanDirLocked()函数遍历/dev/input文件夹下的所有设备节点并打开
status_tres = scanDirLocked(DEVICE_PATH);
......// 错误处理
// 打开一个名为VIRTUAL_KEYBOARD的输入设备。这个设备时刻是打开着的。它是一个虚拟的输入设
   备，没有对应的输入节点。读者先记住有这么一个输入设备存在于输入系统中 */
if(mDevices.indexOfKey(VIRTUAL_KEYBOARD_ID) < 0) {
   createVirtualKeyboardLocked();
}
}
```

scanDirLocked()遍历指定文件夹下的所有设备节点，分别对其执行openDeviceLocked()完成设备的打开操作。在这个函数中将为设备节点创建并加载Device结构体。参考其代码：

[EventHub.cpp-->EventHub::openDeviceLocked()]

```c
status_t EventHub::openDeviceLocked(const char*devicePath) {
// 打开设备节点的文件描述符，用于获取设备信息以及读取原始输入事件
int fd =open(devicePath, O_RDWR | O_CLOEXEC);
// 接下来的代码通过ioctl()函数从设备节点中获取输入设备的厂商信息
InputDeviceIdentifier identifier;
......
// 分配一个设备Id并创建Device结构体
int32_tdeviceId = mNextDeviceId++;
Device*device = new Device(fd, deviceId, String8(devicePath), identifier);
// 为此设备加载配置信息。
 loadConfigurationLocked(device);
 // ① 通过ioctl函数获取设备的事件位掩码。事件位掩码指定了输入设备可以产生何种类型的输入事件
  ioctl(fd, EVIOCGBIT(EV_KEY, sizeof(device->keyBitmask)),device->keyBitmask);
......
 ioctl(fd, EVIOCGPROP(sizeof(device->propBitmask)),device->propBitmask);
  // 接下来的一大段内容是根据事件位掩码为设备分配类别，即设置classes字段。、
......
 /* ② 将设备节点的描述符的可读事件注册到Epoll中。当此设备的输入事件到来时，Epoll会在
  getEvents()函数的调用中产生一条epoll事件 */
   structepoll_event eventItem;
   memset(&eventItem, 0, sizeof(eventItem));
   eventItem.events = EPOLLIN;
   eventItem.data.u32 = deviceId; /* 注意，epoll_event的自定义信息是设备的Id
   if(epoll_ctl(mEpollFd, EPOLL_CTL_ADD, fd, &eventItem)) {
    ......
    }
     ......
     // ③ 调用addDeviceLocked()将Device添加到mDevices字典中
     addDeviceLocked(device);
     return0;
     }
```

openDeviceLocked()函数打开指定路径的设备节点，为其创建并填充Device结构体，然后将设备节点的可读事件注册到Epoll中，最后将新建的Device结构体添加到mDevices字典中以供检索之需。整个过程比较清晰，但仍有以下几点需要注意：

· openDeviceLocked()函数从设备节点中获取了设备可能上报的事件类型，并据此为设备分配了类别。整个分配过程非常繁琐，由于它和InputReader的事件加工过程关系紧密，因此这部分内容将在5.2.4节再做详细讨论。

· 向Epoll注册设备节点的可读事件时，epoll_event的自定义数据被设置为设备的Id而不是fd。

· addDeviceLocked()将新建的Device对象添加到mDevices字典中的同时也会将其添加到一个名为mOpeningDevices的链表中。这个链表保存了刚刚被加载，但尚未通过getEvents()函数向InputReader发送DEVICE_ADD事件的设备。

完成输入设备的加载之后，通过getEvents()函数便可以读取到此设备所产生的输入事件了。除了在getEvents()函数中使用scanDevicesLockd()一次性加载所有输入设备，当INotify事件告知有新的输入设备节点被创建时，也会通过opendDeviceLocked()将设备加载，稍后再做讨论。

### （2）、输入设备的卸载

输入设备的卸载由closeDeviceLocked()函数完成。由于此函数的工作内容与openDeviceLocked()函数正好相反，就不列出其代码了。设备的卸载过程为：

· 从Epoll中注销对描述符的监听。

· 关闭设备节点的描述符。

· 从mDevices字典中删除对应的Device对象。

· 将Device对象添加到mClosingDevices链表中，与mOpeningDevices类似，这个链表保存了刚刚被卸载，但尚未通过getEvents()函数向InputReader发送DEVICE_REMOVED事件的设备。

同加载设备一样，在getEvents()函数中有根据需要卸载所有输入设备的操作（比如当EventHub要求重新加载所有设备时，会先将所有设备卸载）。并且当INotify事件告知有设备节点删除时也会调用closeDeviceLocked()将设备卸载。

### （3）、设备增删事件

在分析设备的加载与卸载时发现，新加载的设备与新卸载的设备会被分别放入mOpeningDevices与mClosingDevices链表之中。这两个链表将是在getEvents()函数中向InputReader发送设备增删事件的依据。

参考getEvents()函数的相关代码，以设备卸载事件为例看一下设备增删事件是如何产生的：

[EventHub.cpp-->EventHub::getEvents()]

```c
size_t EventHub::getEvents(int timeoutMillis,RawEvent* buffer, size_t bufferSize) {
for (;;){
    // 遍历mClosingDevices链表，为每一个已卸载的设备生成DEVICE_REMOVED事件
   while (mClosingDevices) {
       Device* device = mClosingDevices;
       mClosingDevices = device->next;
       /* 分析getEvents()函数的工作方式时介绍过，event指针指向buffer中下一个可用于填充
          事件的RawEvent对象 */
       event->when = now; // 设置产生事件的事件戳
       event->deviceId =
               device->id ==mBuiltInKeyboardId ? BUILT_IN_KEYBOARD_ID : device->id;
       event->type = DEVICE_REMOVED; // 设置事件的类型为DEVICE_REMOVED
       event += 1; // 将event指针移动到下一个可用于填充事件的RawEvent对象
       delete device; // 生成DEVICE_REMOVED事件之后，被卸载的Device对象就不再需要了
       mNeedToSendFinishedDeviceScan = true; // 随后发送FINISHED_DEVICE_SCAN事件
        /* 当buffer已满则停止继续生成事件，将已生成的事件返回给调用者。尚未生成的事件
          将在下次getEvents()调用时生成并返回给调用者 */
       if (--capacity == 0) {
           break;
       }
    }
    // 接下来进行DEVICE_ADDED事件的生成，此过程与 DEVICE_REMOVED事件的生成一致
   ......
}
returnevent – buffer;
}
```

可以看到，在一次getEvents()调用中会尝试为所有尚未发送增删事件的输入设备生成对应的事件返回给调用者。表示设备增删事件的RawEvent对象包含三个信息：产生事件的事件戳、产生事件的设备Id，以及事件类型（DEVICE_ADDED或DEVICE_REMOVED）。

当生成设备增删事件时，会设置mNeedToSendFinishedDeviceSan为true，这个动作的意思是完成所有DEVICE_ADDED/REMOVED事件的生成之后，需要向getEvents()的调用者发送一个FINISHED_DEVICE_SCAN事件，表示设备增删事件的上报结束。这个事件仅包括时间戳与事件类型两个信息。

经过以上分析可知，EventHub可以产生的设备增删事件一共有三种，而且这三种事件拥有固定的优先级，DEVICE_REMOVED事件的优先级最高，DEVICE_ADDED事件次之，FINISHED_DEVICE_SCAN事件最低。而且，getEvents()完成当前高优先级事件的生成之前，不会进行低优先级事件的生成。因此，当发生设备的加载与卸载时，EventHub所生成的完整的设备增删事件序列如图5-5所示，其中R表示DEVICE_REMOVED，A表示DEVICE_ADDED，F表示FINISHED_DEVICE_SCAN。

![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N16-Android-Input-System-input-device-add-del.png)

图：设备增删事件的完整序列

由于参数buffer的容量限制，这个事件序列可能需要通过多次getEvents()调用才能完整地返回给调用者。另外，根据5.2.2节的讨论，设备增删事件相对于Epoll事件拥有较高的优先级，因此从R1事件开始生成到F事件生成之前，getEvents()不会处理Epoll事件，也就是说不会生成原始输入事件。

总结一下设备增删事件的生成原理：

· 当发生设备增删时，addDeviceLocked()函数与closeDeviceLocked()函数会将相应的设备放入mOpeningDevices和mClosingDevices链表中。

· getEvents()函数会根据mOpeningDevices和mClosingDevices两个链表生成对应DEVICE_ADDED和DEVICE_REMOVED事件，其中后者的生成拥有高优先级。

· DEVICE_ADDED和DEVICE_REMOVED事件都生成完毕后，getEvents()会生成FINISHED_DEVICE_SCAN事件，标志设备增删事件序列的结束。

### （4）、通过INotify动态地加载与卸载设备

通过前文的介绍知道了openDeviceLocked()和closeDeviceLocked()可以加载与卸载输入设备。接下来分析EventHub如何通过INotify进行设备的动态加载与卸载。在EventHub的构造函数中创建了一个名为mINotifyFd的INotify对象的描述符，用以监控/dev/input下设备节点的增删。之后将mINotifyFd的可读事件加入到Epoll中。于是可以确定动态加载与卸载设备的工作方式为：首先筛选epoll_wait()函数所取得的Epoll事件，如果Epoll事件表示了mINotifyFd可读，便从mINotifyFd中读取设备节点的增删事件，然后通过执行openDeviceLocked()或closeDeviceLocked()进行设备的加载与卸载。

看一下getEvents()中与INotify相关的代码：

[EventHub.cpp-->EventHub::getEvents()]

```c
size_t EventHub::getEvents(int timeoutMillis,RawEvent* buffer, size_t bufferSize) {
for (;;){
   ...... // 设备增删事件处理
    while(mPendingEventIndex < mPendingEventCount) {
       const struct epoll_event& eventItem =
                             mPendingEventItems[mPendingEventIndex++];
       /* ① 通过Epoll事件的data字段确定此事件表示了mINotifyFd可读
          注意EPOLL_ID_INOTIFY在EventHub的构造函数中作为data字段向
          Epoll注册mINotifyFd的可读事件 */
       if (eventItem.data.u32 == EPOLL_ID_INOTIFY) {
           if (eventItem.events & EPOLLIN) {
               mPendingINotify = true; // 标记INotify事件待处理
           } else { ...... }
           continue; // 继续处理下一条Epoll事件
       }
       ...... // 其他Epoll事件的处理
    }
    // 如果INotify事件待处理
    if(mPendingINotify && mPendingEventIndex >= mPendingEventCount) {
       mPendingINotify = false;
       /* ② 调用readNotifyLocked()函数读取并处理存储在mINotifyFd中的INotify事件
          这个函数将完成设备的加载与卸载 */
       readNotifyLocked();
       deviceChanged = true;
    }
    //③ 如果处理了INotify事件，则返回到循环开始处，生成设备增删事件
    if(deviceChanged) {
       continue;
    }
}
}
```

getEvents()函数中与INotify相关的代码共有三处：

· 识别表示mINotifyFd可读的Epoll事件，并通过设置mPendingINotify为true以标记有INotify事件待处理。getEvents()并没有立刻处理INotify事件，因为此时进行设备的加载与卸载是不安全的。其他Epoll事件可能包含了来自即将被卸载的设备的输入事件，因此需要将所有Epoll事件都处理完毕后再进行加载与卸载操作。

· 当epoll_wait()所返回的Epoll事件都处理完毕后，调用readNotifyLocked()函数读取mINotifyFd中的事件，并进行设备的加载与卸载操作。

· 完成设备的动态加载与卸载后，需要返回到循环最开始处，以便设备增删事件处理代码生成设备的增删事件。

其中第一部分与第三部分比较容易理解。接下来看一下readNotifyLocked()是如何工作的。

[EventHub.cpp-->EventHub::readNotifyLocked()]

```c
status_t EventHub::readNotifyLocked() {

    ......
    // 从mINotifyFd中读取INotify事件列表
    res =read(mINotifyFd, event_buf, sizeof(event_buf));
    ......
    // 逐个处理列表中的事件
       while(res >= (int)sizeof(*event)) {
       strcpy(filename, event->name); // 从事件中获取设备节点路径
       if(event->mask & IN_CREATE) {
           openDeviceLocked(devname); // 如果事件类型为IN_CREATE，则加载对应设备
        }else {
           closeDeviceByPathLocked(devname); // 否则卸载对应设备
        }
        ......// 移动到列表中的下一个事件
    }
    return0;
    }
```

### （5）、EventHub设备管理总结

至此，EventHub的设备管理相关的知识便讨论完毕了。在这里进行一下总结：

· EventHub通过Device结构体描述输入设备的各种信息。

· EventHub在getEvents()函数中进行设备的加载与卸载操作。设备的加载与卸载分为按需加载或卸载以及通过INotify动态加载或卸载特定设备两种方式。

· getEvents()函数进行了设备的加载与卸载操作后，会生成DEVICE_ADDED、DEVICE_REMOVED以及FINISHED_DEVICE_SCAN三种设备增删事件，并且设备增删事件拥有高于Epoll事件的优先级。 4．原始输入事件的监听与读取 本节将讨论EventHub另一个核心的功能，监听与读取原始输入事件。

回忆一下输入设备的加载过程，当设备加载时，openDeviceLocked()会打开设备节点的文件描述符，并将其可读事件注册进Epoll中。于是当设备的原始输入事件到来时，getEvents()函数将会获得一条Epoll事件，然后根据此Epoll事件读取文件描述符中的原始输入事件，将其填充到RawEvents结构体并放入buffer中被调用者取走。openDeviceLocked()注册了设备节点的EPOLLIN和EPOLLHUP两个事件，分别表示可读与被挂起（不可用），因此getEvents()需要分别处理这两种事件。

看一下getEvents()函数中的相关代码：

[EventHub.cpp-->EventHub::getEvents()]

```c
size_t EventHub::getEvents(int timeoutMillis,RawEvent* buffer, size_t bufferSize) {
for (;;){
   ...... // 设备增删事件处理
    while(mPendingEventIndex < mPendingEventCount) {
       const struct epoll_event& eventItem =
                             mPendingEventItems[mPendingEventIndex++];
       ...... // INotify与wakeFd的Epoll事件处理
       /* ① 通过Epoll的data.u32字段获取设备Id，进而获取对应的Device对象。如果无法找到
          对应的Device对象，说明此Epoll事件并不表示原始输入事件的到来，忽略之 */
       ssize_t deviceIndex = mDevices.indexOfKey(eventItem.data.u32);
       Device* device = mDevices.valueAt(deviceIndex);
       ......
       if (eventItem.events & EPOLLIN) {
           /* ② 如果Epoll事件为EPOLLIN，表示设备节点有原始输入事件可读。此时可以从描述符
              中读取。读取结果作为input_event结构体并存储在readBuffer中，注意事件的个数
               受到capacity的限制*/
           int32_t readSize = read(device->fd, readBuffer,
                    sizeof(structinput_event) * capacity);
           if (......) {                    ......// 一些错误处理 }
            else {
               size_t count = size_t(readSize) / sizeof(struct input_event);
               /* ② 将读取到的每一个input_event结构体中的数据转换为一个RawEvent对象，
                   并存储在buffer参数中以返回给调用者 */
               for (size_t i = 0; i < count; i++) {
                    const structinput_event& iev = readBuffer[i];
                    ......
                    event->when = now;
                    event->deviceId =deviceId;
                    event->type =iev.type;
                    event->code =iev.code;
                    event->value =iev.value;
                   event += 1; // 移动到buffer的下一个可用元素
               }
               /* 接下来的一个细节需要注意，因为buffer的容量限制，可能无法完全读取设备节点
                   中存储的原始事件。一旦buffer满了则需要立刻返回给调用者。设备节点中剩余的
                   输入事件将在下次getEvents()调用时继续读取，也就是说，当前的Epoll事件
                   并未处理完毕。mPendingEventIndex -= 1的目的就是使下次getEvents()调用
                   能够继续处理这个Epoll事件 */
               capacity -= count;
               if (capacity == 0) {
                    mPendingEventIndex -=1;
                    break;
               }
           }
       } else if (eventItem.events & EPOLLHUP) {
           deviceChanged = true; // 如果设备节点的文件描述符被挂起则卸载此设备
           closeDeviceLocked(device);
       } else { ...... }
    }
   ...... // 读取并处理INotify事件
    ......// 等待新的Epoll事件
}
return event – buffer；}
```

getEvents()通过Epoll事件的data.u32字段在mDevices列表中查找已加载的设备，并从设备的文件描述符中读取原始输入事件列表。从文件描述符中读取的原始输入事件存储在input_event结构体中，这个结构体的四个字段存储了事件的事件戳、类型、代码与值四个元素。然后逐一将input_event的数据转存到RawEvent中并保存至buffer以返回给调用者。

注意为了叙述简单，上述代码使用了调用getEvents()的时间作为输入事件的时间戳。由于调用getEvents()函数的时机与用户操作的时间差的存在，会使得此时间戳与事件的真实时间有所偏差。从设备节点中读取的input_event中也包含了一个时间戳，这个时间戳消除了getEvents()调用所带来的时间差，因此可以获得更精确的时间控制。可以通过打开HAVE_POSIX_CLOCKS宏以使用input_event中的时间而不是将getEvents()调用的时间作为输入事件的时间戳。

需要注意的是，由于Epoll事件的处理优先级低于设备增删事件，因此当发生设备加载与卸载动作时，不会产生设备输入事件。另外还需注意，在一个监听周期中，getEvents()在将一个设备节点中的所有原始输入事件读取完毕之前，不会读取其他设备节点中的事件。

### 5、EventHub总结

本节针对EventHub的设备管理与原始输入事件的监听读取两个核心内容介绍了EventHub的工作原理。EventHub作为直接操作设备节点的输入系统组件，隐藏了INotify与Epoll以及设备节点读取等底层操作，通过一个简单的接口getEvents()向使用者提供抽取设备事件与原始输入事件的功能。EventHub的核心功能都在getEvents()函数中完成，因此深入理解getEvents()的工作原理对于深入理解EventHub至关重要。

getEvents()函数的本质是通过epoll_wait()获取Epoll事件到事件池，并对事件池中的事件进行消费的过程。从epoll_wait()的调用开始到事件池中最后一个事件被消费完毕的过程称之为EventHub的一个监听周期。由于buffer参数的尺寸限制，一个监听周期可能包含多个getEvents()调用。周期中的第一个getEvents()调用一定会因事件池的枯竭而直接进行epoll_wait()，而周期中的最后一个getEvents()一定会将事件池中的最后一条事件消费完毕并将事件返回给调用者。前文所讨论的事件优先级都是在同一个监听周期内而言的。

在本节中出现了很多种事件，有原始输入事件、设备增删事件、Epoll事件、INotify事件等，存储事件的结构体有RawEvent、epoll_event、inotify_event、input_event等。图5-6可以帮助读者理清这些事件之间的关系。

![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N17-Android-Input-System-epoll.png)

图 5-6 EventHub的事件关联

另外，getEvents()函数返回的事件列表依照事件的优先级拥有特定的顺序。并且在一个监听周期中，同一输入设备的输入事件在列表中是相邻的。

至此，相信读者对EventHub的工作原理，以及EventHub的事件监听与读取机制有了深入的了解。接下来的内容将讨论EventHub所提供的原始输入事件如何被加工为Android输入事件，这个加工者就是Reader子系统中的另一员大将：InputReader。

![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N18-Android-Input-System-EventHub-Kernel.png)

--------------------------------------------------------------------------------

## 五、Input Reader

根据第四节的分析。输入设备扫描完成，并加入epoll中，监听事件。从前面的getEvents函数分析得知，当按键事件发生后，getEvents函数返回。 这里再贴一下Input 处理时间流程图，然后按步骤详细分析。 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N15-Android-Input-System-input-reader-thread.png)

以一次键盘按键为例，得到下面的6个事件

```c
EventHub: /dev/input/event2 got: time=4383.680195, type=4, code=4, value=458792
EventHub: /dev/input/event2 got: time=4383.680195, type=1, code=28, value=1
EventHub: /dev/input/event2 got: time=4383.680195, type=0, code=0, value=0
EventHub: /dev/input/event2 got: time=4383.760186, type=4, code=4, value=458792
EventHub: /dev/input/event2 got: time=4383.760186, type=1, code=28, value=0
EventHub: /dev/input/event2 got: time=4383.760186, type=0, code=0, value=0
```

上面的type是linux的输入系统里的事件，具体的值可以查看 查看input.h

```c
/* WARNING: DO NOT EDIT, AUTO-GENERATED CODE - SEE TOP FOR INSTRUCTIONS */
#define EV_SYN 0x00   同步事件
#define EV_KEY 0x01   按键事件
#define EV_REL 0x02   相对坐标
#define EV_ABS 0x03   绝对坐标
/* WARNING: DO NOT EDIT, AUTO-GENERATED CODE - SEE TOP FOR INSTRUCTIONS */
#define EV_MSC 0x04   其它
#define EV_SW  0x05   
#define EV_LED 0x11   LED
#define EV_SND 0x12   声音
/* WARNING: DO NOT EDIT, AUTO-GENERATED CODE - SEE TOP FOR INSTRUCTIONS */
#define EV_REP 0x14   Repeat
#define EV_FF  0x15   力反馈
#define EV_PWR 0x16   电源
#define EV_FF_STATUS 0x17 状态
```

上面6个事件，只有两个type为1的事件，是我们需要处理的按键事件，一个down，一个up

## Step 1、 InputReader::loopOnce()

返回到InputReader的loopOnce函数

```c
[->frameworks/native/services/inputflinger/InputReader.cpp]

void InputReader::loopOnce() {
  size_t count = mEventHub->getEvents(timeoutMillis, mEventBuffer, EVENT_BUFFER_SIZE);

  { // acquire lock
      AutoMutex _l(mLock);
      mReaderIsAliveCondition.broadcast();
      //当有按键事件发生时，count将不为0，以一次按键为例，这里应该是6个事件
      if (count) {
          processEventsLocked(mEventBuffer, count);
      }
   } // release lock
   }
```

当有按键事件发生时，count将不为0，之后会调用processEventsLocked来处理RawEvent。

## Step 2、InputReader.processEventsLocked()

```c
[->frameworks/native/services/inputflinger/InputReader.cpp]

void InputReader::processEventsLocked(const RawEvent* rawEvents, size_t count) {
 for (const RawEvent* rawEvent = rawEvents; count;) {
  int32_t type = rawEvent->type;
  size_t batchSize = 1;
  if (type < EventHubInterface::FIRST_SYNTHETIC_EVENT) {
      int32_t deviceId = rawEvent->deviceId;
      //依次处理rawEvent
      processEventsForDeviceLocked(deviceId, rawEvent, batchSize);
  }
  count -= batchSize;
  rawEvent += batchSize;
  }
}
```

该函数调用processEventsForDeviceLocked依次处理rawEvent

## Step 3、InputReader.processEventsForDeviceLocked()

```c
[->frameworks/native/services/inputflinger/InputReader.cpp]

void InputReader::processEventsForDeviceLocked(int32_t deviceId,
      const RawEvent* rawEvents, size_t count) {
  ssize_t deviceIndex = mDevices.indexOfKey(deviceId);
  InputDevice* device = mDevices.valueAt(deviceIndex);
  //调用InputDevice的process函数
  device->process(rawEvents, count);
}
```

这里根据deviceId获取到InputDevice，然后调用InputDevice的process函数

## Step 4、InputDevice.process()

```c
[->frameworks/native/services/inputflinger/InputReader.cpp]

void InputDevice::process(const RawEvent* rawEvents, size_t count) {
size_t numMappers = mMappers.size();
for (const RawEvent* rawEvent = rawEvents; count--; rawEvent++) {
    if (mDropUntilNextSync) {
        if (rawEvent->type == EV_SYN && rawEvent->code == SYN_REPORT) {
    } else if (rawEvent->type == EV_SYN && rawEvent->code == SYN_DROPPED) {
    } else {
        for (size_t i = 0; i < numMappers; i++) {    
            InputMapper* mapper = mMappers[i];  
            // InputMapper是做什么的呢，它是用于解析原始输入事件的。比如back, home等VirtualKey，
            // 传上来时是个Touch事件，这里要根据坐标转化为相应的按键事件。再比如多点触摸时，需要计算
            // 每个触摸点分别属于哪条轨迹，安卓系统中每种输入设备都对应了一种Mapper,比如
            // SwitchInputMapper, VibratorInputMapper,KeyBoardInputMapper
            mapper->process(rawEvent);           
        }
     }
   }
}
```

这里的mMappers成员变量保存了一系列输入设备事件处理对象，例如负责处理键盘事件的KeyboardKeyMapper对象以及负责处理触摸屏事件的TouchInputMapper对象， 它们是在InputReader类的成员函数createDeviceLocked中创建的。这里查询每一个InputMapper对象是否要对当前发生的事件进行处理。由于发生的是键盘事件，真正会对该事件进行处理的只有KeyboardKeyMapper对象。

## Step 5、KeyboardInputMapper.process()

```c
[->frameworks/native/services/inputflinger/InputReader.cpp]

void KeyboardInputMapper::process(const RawEvent* rawEvent) {
    switch (rawEvent->type) {
    case EV_KEY: {
        int32_t scanCode = rawEvent->code;
        int32_t usageCode = mCurrentHidUsage;
        mCurrentHidUsage = 0;

        if (isKeyboardOrGamepadKey(scanCode)) {
            int32_t keyCode;
            uint32_t flags;

            // 调用EventHub中的mapKey函数进行转化
            // 传入参数
            // scanCode：驱动程序上报的扫描码；keyCode：转化之后的Android使用的按键值

            if (getEventHub()->mapKey(getDeviceId(), scanCode, usageCode, &keyCode, &flags)) {
                keyCode = AKEYCODE_UNKNOWN;
                flags = 0;
            }
            //映射成功之后，处理该按键
            processKey(rawEvent->when, rawEvent->value != 0, keyCode, scanCode, flags);
        }
        break;
    }
    case EV_MSC: {
        if (rawEvent->code == MSC_SCAN) {
            mCurrentHidUsage = rawEvent->value;
        }
        break;
    }
    case EV_SYN: {
        if (rawEvent->code == SYN_REPORT) {
            mCurrentHidUsage = 0;
        }
    }
    }
}
```

函数首先调用isKeyboardOrGamepadKey来判断键盘扫描码是否正确，如果正确则调用processKey来进一步处理

## Step 6、KeyboardInputMapper.processKey()

```c
[->frameworks/native/services/inputflinger/InputReader.cpp]

void KeyboardInputMapper::processKey(nsecs_t when, bool down, int32_t keyCode,
        int32_t scanCode, uint32_t policyFlags) {

    // 根据扫描码scanCode、按键码keyCode、newMetaState、downTime按下的时间进行处理    
    // NotifyKeyArgs args(when, getDeviceId(), mSource, policyFlags,
    ......

    NotifyKeyArgs args(when, getDeviceId(), mSource, policyFlags,
            down ? AKEY_EVENT_ACTION_DOWN : AKEY_EVENT_ACTION_UP,
            AKEY_EVENT_FLAG_FROM_SYSTEM, keyCode, scanCode, newMetaState, downTime);
     // 通知Listener处理，Dispatch线程会监听该事件，并处理，下次博文会具体分析
    getListener()->notifyKey(&args);
}
```

这个函数首先对按键作一些处理，根据扫描码scanCode、按键码keyCode、newMetaState、downTime按下的时间进行处理

最后函数会调用：

```c
NotifyKeyArgs args(when, getDeviceId(), mSource, policyFlags,
            down ? AKEY_EVENT_ACTION_DOWN : AKEY_EVENT_ACTION_UP,
            AKEY_EVENT_FLAG_FROM_SYSTEM, keyCode, scanCode, newMetaState, downTime);

getListener()->notifyKey(&args);
```

这里getListener是InputReader初始化时传入的对象，即QueuedInputListener，则会调用QueuedInputListener的notifyKey函数

```c
void QueuedInputListener::notifyKey(const NotifyKeyArgs* args) {
mArgsQueue.push(new NotifyKeyArgs(*args));
}
```

InputReader的loopOnce()的结尾会调用QueuedInputListener::flush()统一回调缓冲队列中各元素的notify()接口：

```c
void QueuedInputListener::flush() {
size_t count = mArgsQueue.size();
for (size_t i = 0; i < count; i++) {
    NotifyArgs* args = mArgsQueue[i];
    args->notify(mInnerListener);
    delete args;
}
mArgsQueue.clear();
}
```

进一步调用：

```c
void NotifyConfigurationChangedArgs::notify(const sp<InputListenerInterface>& listener) const {
listener->notifyConfigurationChanged(this);
}
```

以按键事件为例，由于InputDispatcher 实现了InputListenerInterface接口的notifyConfigurationChanged()函数，所以最后会调用到InputDispatcher的notifyKey()函数中。

## Step 7、 InputDispatcher.notifyKey()

```c
[->frameworks/native/services/inputflinger/InputDispatcher.cpp]

void InputDispatcher::notifyKey(const NotifyKeyArgs* args) {

......
// 构造一个KeyEvent对象
KeyEvent event;
event.initialize(args->deviceId, args->source, args->action,
        flags, keyCode, args->scanCode, metaState, 0,
        args->downTime, args->eventTime);
// 调用mPolicy的interceptKeyBeforeQueueing函数，该函数最后会调用到java层的PhoneWindowManagerService函数
mPolicy->interceptKeyBeforeQueueing(&event, /*byref*/ policyFlags);

bool needWake;
{ // acquire lock
    mLock.lock();
    ......
    //构造一个KeyEntry对象，调用enqueueInboundEventLocked函数将按键事件加入队列
    KeyEntry* newEntry = new KeyEntry(args->eventTime,
            args->deviceId, args->source, policyFlags,
            args->action, flags, keyCode, args->scanCode,
            metaState, repeatCount, args->downTime);

    needWake = enqueueInboundEventLocked(newEntry);
    mLock.unlock();
} // release lock

if (needWake) {
    mLooper->wake();
}
}
```

该函数首先调用validateKeyEvent来判断是否是有效按键事件，实际判断是否是UP/DOWN事件

然后构造一个KeyEvent对象，调用mPolicy的interceptKeyBeforeQueueing函数，该函数最后会调用到java层的PhoneWindowManagerService函数

```c
KeyEvent event;
    event.initialize(args->deviceId, args->source, args->action,
            flags, keyCode, args->scanCode, metaState, 0,
            args->downTime, args->eventTime);

    mPolicy->interceptKeyBeforeQueueing(&event, /*byref*/ policyFlags);
```

之后会调用构造一个KeyEntry对象，调用enqueueInboundEventLocked函数将按键事件加入队列，如果返回true，则调用mLooper.wake函数唤醒等待的InputDispatcher，进行按键分发。

```c
KeyEntry* newEntry = new KeyEntry(args->eventTime,
                args->deviceId, args->source, policyFlags,
                args->action, flags, keyCode, args->scanCode,
                metaState, repeatCount, args->downTime);

needWake = enqueueInboundEventLocked(newEntry);
```

## Step 8、InputDispatcher.enqueueInboundEventLocked()

```c
[->frameworks/native/services/inputflinger/InputDispatcher.cpp]

bool InputDispatcher::enqueueInboundEventLocked(EventEntry* entry) {
    bool needWake = mInboundQueue.isEmpty();
    mInboundQueue.enqueueAtTail(entry);
    traceInboundQueueLengthLocked();

    switch (entry->type) {
    case EventEntry::TYPE_KEY: {
        // ......
        KeyEntry* keyEntry = static_cast<KeyEntry*>(entry);
        if (isAppSwitchKeyEventLocked(keyEntry)) {
            if (keyEntry->action == AKEY_EVENT_ACTION_DOWN) {
                mAppSwitchSawKeyDown = true;
            } else if (keyEntry->action == AKEY_EVENT_ACTION_UP) {
                if (mAppSwitchSawKeyDown) {
                    mAppSwitchDueTime = keyEntry->eventTime + APP_SWITCH_TIMEOUT;
                    mAppSwitchSawKeyDown = false;
                    needWake = true;
                }
            }
        }
        break;
    }
  }
  return needWake;
}
```

将EventEntry加入到mInboundQueue中，该函数两种情况下会返回true,一是当加入该键盘事件到mInboundQueue之前，mInboundQueue为空，这表示InputDispatc herThread线程正在睡眠等待InputReaderThread线程的唤醒，因此，它返回true表示要唤醒InputDispatccherThread线程；二是加入该键盘事件到mInboundQueue之前，mInboundQueue不为空，但是此时用户按下的是Home键等需要切换APP的按键，我们知道，在切换App时，新的App会把它的键盘消息接收通道注册到InputDispatcher中去，并且会等待InputReader的唤醒，因此，在这种情况下，也需要返回true，表示要唤醒InputDispatccherThread线程。如果不是这两种情况，那么就说明InputDispatccherThread线程现在正在处理前面的键盘事件，不需要唤醒它。

至此，InputDispatcherThread被唤醒，开始进行按键分发。

## 总结：

InputReaderThread不断调用InputReader的pollOnce()->getEvents()函数来得到事件，这些事件可以是输入事件，也可以是由inotify监测到设备增减变更所触发的事件。第一次进入时会扫描/dev/input目录建立设备列表，存在mDevice成员变量中(EventHub中有设备列表KeyedVector

<int32_t, device*=""> mDevices；对应的，InputReader中也有设备列表KeyedVector<int32_t, inputdevice*=""> mDevices。这里先添加到前者，然后会在InputReader::addDeviceLocked()中添加到后者。)，同时将增加的fd加到epoll的等待集合中。在接下来的epoll_wait()等待时，如果有事件就会返回，同时返回可读事件数量。在这里，从Input driver读出的事件从原始的input_event结构转为RawEvent结构，放到getEvents()的输出参数buffer中。getEvents()返回后，InputReader调用processEventsLocked()处理事件，对于设备改变，会根据实际情况调用addDeviceLocked(), removeDeviceLocked()和handleConfigurationChangedLocked()。对于其它设备中来的输入事件，会调用processEventsForDeviceLocked()进一步处理。其中会根据当时注册的InputMapper对事件进行处理，然后将事件处理请求放入缓冲队列（QueuedInputListener中的mArgsQueue）。</int32_t,></int32_t,>

InputMapper是做什么的呢，它是用于解析原始输入事件的。比如back, home等VirtualKey，传上来时是个Touch事件，这里要根据坐标转化为相应的按键事件。再比如多点触摸时，需要计算每个触摸点分别属于哪条轨迹，这本质上是个二分图匹配问题，这也是在InputMapper中完成的。回到流程主线上，在InputReader的loopOnce()的结尾会调用QueuedInputListener::flush()统一回调缓冲队列中各元素的notify()接口。

以按键事件为例，最后会调用到InputDispatcher的notifyKey()函数中。这里先将参数封装成KeyEvent： 然后把它作为参数调用NativeInputManager的interceptKeyBeforeQueueing()函数。顾名思义，就是在放到待处理队列前看看是不是需要系统处理的系统按键，它会通过JNI调回Java世界，最终调到PhoneWindowManager的interceptKeyBeforeQueueing()。然后，基于输入事件信息创建KeyEntry对象，调用enqueueInboundEventLocked()将之放入队列等待InputDiaptcherThread线程拿出处理。

## 六、Input Dispatcher

InputDisptacher的主要任务是把前面收到的输入事件发送到PWM及App端的焦点窗口。前面提到InputReaderThread中收到事件后会调用notifyKey()来通知InputDispatcher，也就是放在mInboundQueue中，在InputDispatcher的dispatchOnce()函数中，会从这个队列拿出处理。

其中dispatchOnceInnerLocked()会根据拿出的EventEntry类型调用相应的处理函数，以Key事件为例会调用dispatchKeyLocked()

它会找到目标窗口，然后通过之前和App间建立的连接发送事件。如果是个需要系统处理的Key事件，这里会封装成CommandEntry插入到mCommandQueue队列中，后面的runCommandLockedInterruptible()函数中会调用doInterceptKeyBeforeDispatchingLockedInterruptible()来让PWM有机会进行处理。最后dispatchOnce()调用pollOnce()从和App的连接上接收处理完成消息。那么，InputDispatcher是怎么确定要往哪个窗口中发事件呢？这里的成员变量mFocusedWindowHandle指示了焦点窗口，然后findFocusedWindowTargetsLocked()会调用一系列函数（handleTargetsNotReadyLocked(), checkInjectionPermission(), checkWindowReadyForMoreInputLocked()等）检查mFocusedWindowHandle是否能接收输入事件。如果可以，将之以InputTarget的形式加到目标窗口数组中。然后就会调用dispatchEventLocked()进行发送。那么，这个mFocusedWindowHandle是如何维护的呢？为了更好地理解，这里回头分析下窗口连接的管理及焦点窗口的管理。 总体流程图： 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N19-Android-Input-System-InputDispatcher-structure.png)

再贴一张详细的总体流程图，然后根据步骤详细分析；

![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N20-Android-Input-System-input-dispatcher-thread.png)

## Step 1、InputDispatcher.dispatchOnce()

```c
[->frameworks/native/services/inputflinger/InputDispatcher.cpp]

void InputDispatcher::dispatchOnce() {
nsecs_t nextWakeupTime = LONG_LONG_MAX;
{ // acquire lock
    AutoMutex _l(mLock);
    mDispatcherIsAliveCondition.broadcast();

    // Run a dispatch loop if there are no pending commands.
    // The dispatch loop might enqueue commands to run afterwards.
    if (!haveCommandsLocked()) {
        dispatchOnceInnerLocked(&nextWakeupTime);
    }

    // Run all pending commands if there are any.
    // If any commands were run then force the next poll to wake up immediately.
    if (runCommandsLockedInterruptible()) {
        nextWakeupTime = LONG_LONG_MIN;
    }
} // release lock

// Wait for callback or timeout or wake.  (make sure we round up, not down)
nsecs_t currentTime = now();
int timeoutMillis = toMillisecondTimeoutDelay(currentTime, nextWakeupTime);
mLooper->pollOnce(timeoutMillis);
}
```

上述函数主要是调用dispatchOnceInnerLocked来进行一次按键分发，当没有按键消息时会走到mLooper->pollOnce(timeoutMillis);这个函数会进入睡眠状态，当有按键消息发生时该函数会返回，然后走到dispatchOnceInnerLocked函数。

## Step 2、InputDispatcher.dispatchOnceInnerLocked()

```c
[->frameworks/native/services/inputflinger/InputDispatcher.cpp]

void InputDispatcher::dispatchOnceInnerLocked(nsecs_t* nextWakeupTime) {
    nsecs_t currentTime = now();
    ...
    // Ready to start a new event.
    // If we don't already have a pending event, go grab one.
    if (! mPendingEvent) {
        //当InputReader往队列中插入了一个读取的键盘消息后，此处的mInboundQueue就不为空
        if (mInboundQueue.isEmpty()) {
            ...
        } else {
            // Inbound queue has at least one entry.
            mPendingEvent = mInboundQueue.dequeueAtHead();
            ...
        }
        ...
    }
    ...
    switch (mPendingEvent->type) {
    ...
    case EventEntry::TYPE_KEY: {

        KeyEntry* typedEntry = static_cast<KeyEntry*>(mPendingEvent);
        ...
        done = dispatchKeyLocked(currentTime, typedEntry, &dropReason, nextWakeupTime);
        break;
    }
    ...
    }

    if (done) {
        if (dropReason != DROP_REASON_NOT_DROPPED) {
            dropInboundEventLocked(mPendingEvent, dropReason);
        }
        mLastDropReason = dropReason;

        releasePendingEventLocked();
        *nextWakeupTime = LONG_LONG_MIN;  // force next poll to wake up immediately
    }
}
```

从前文InputReader读取键盘消息过程分析 InputReader读取到一个消息后会调用KeyboardInputMapper的processKey，该函数会调用InputDispatcher的notifyKey函数，然后InputDispatcher会调用enqueueInboundEventLocked函数，将EventEntry加入到mInboundQueue中，然后调用mLooper->wake函数会唤醒InputDispatcherThread线程，InputDispatcher中把队列的第一个事件取出来，因为这里是键盘事件，所以mPendingEvent->type是EventEntry::TYPE_KEY，然后调用dispatchKeyLocked函数

惯例先贴出序列图，按步骤一步步介绍。 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N21-Android-Input-System-dispatch-input-event.png)

## Step 3、InputDispatcher.dispatchKeyLocked()

```c
[->frameworks/native/services/inputflinger/InputDispatcher.cpp ]

bool InputDispatcher::dispatchKeyLocked(nsecs_t currentTime, KeyEntry* entry,
        DropReason* dropReason, nsecs_t* nextWakeupTime) {
    ...
    // Give the policy a chance to intercept the key.
    if (entry->interceptKeyResult == KeyEntry::INTERCEPT_KEY_RESULT_UNKNOWN) {
            if (entry->policyFlags & POLICY_FLAG_PASS_TO_USER) {
            CommandEntry* commandEntry = postCommandLocked(
                    & InputDispatcher::doInterceptKeyBeforeDispatchingLockedInterruptible);
            if (mFocusedWindowHandle != NULL) {
                commandEntry->inputWindowHandle = mFocusedWindowHandle;
            }
            ......
        } else {
            entry->interceptKeyResult = KeyEntry::INTERCEPT_KEY_RESULT_CONTINUE;
        }
    } ......
    // Identify targets.
    Vector<InputTarget> inputTargets;
    int32_t injectionResult = findFocusedWindowTargetsLocked(currentTime,
            entry, inputTargets, nextWakeupTime);
    ......
    setInjectionResultLocked(entry, injectionResult);
    ......
    addMonitoringTargetsLocked(inputTargets);

    // Dispatch the key.
    dispatchEventLocked(currentTime, entry, inputTargets);
    return true;
}
```

这个函数主要做了下面三件事 A. 如果按键是第一次分发，则将命令封装为CommandEntry加入队列，后续执行doInterceptKeyBeforeDispatchingLockedInterruptible，以给java层拦截按键的机会 B. 找到当前激活的Window窗口，并将其加入到Vector中，Android ANR就是在findFocusedWindowTargetsLocked()检测的 C. 找到需要主动监听按键的InputChannel,封装成InputTarget，加入到Vector中 D. 将按键分发到上面的Vector中的InputChannel中，这里存在多个

下面先分析如果将按键分发给InputChannel

```c
[->frameworks/native/services/inputflinger/InputDispatcher.cpp]

void InputDispatcher::dispatchEventLocked(nsecs_t currentTime,
    EventEntry* eventEntry, const Vector<InputTarget>& inputTargets) {
......
for (size_t i = 0; i < inputTargets.size(); i++) {
    const InputTarget& inputTarget = inputTargets.itemAt(i);
    ssize_t connectionIndex = getConnectionIndexLocked(inputTarget.inputChannel);
    if (connectionIndex >= 0) {
        sp<Connection> connection = mConnectionsByFd.valueAt(connectionIndex);
        prepareDispatchCycleLocked(currentTime, connection, eventEntry, &inputTarget);
      } ......
    }
  }
}
```

## Step 5、InputDispatcher.prepareDispatchCycleLocked()

```c
[->frameworks/native/services/inputflinger/InputDispatcher.cpp]

void InputDispatcher::prepareDispatchCycleLocked(nsecs_t currentTime,
        const sp<Connection>& connection, EventEntry* eventEntry, const InputTarget* inputTarget) {
    ...
    // Not splitting.  Enqueue dispatch entries for the event as is.
    enqueueDispatchEntriesLocked(currentTime, connection, eventEntry, inputTarget);
    }
```

函数前面还有一些状态检查，这里默认都是通过的。最后enqueueDispatchEntriesLocked函数进行将connection分装成DispatchEntry，加入到connection->outboundQueue的队列中

## Step 6\. InputDispatcher::enqueueDispatchEntriesLocked()

```c
[->frameworks/native/services/inputflinger/InputDispatcher.cpp]

void InputDispatcher::enqueueDispatchEntriesLocked(nsecs_t currentTime,
        const sp<Connection>& connection, EventEntry* eventEntry, const InputTarget* inputTarget) {
    bool wasEmpty = connection->outboundQueue.isEmpty();

    // Enqueue dispatch entries for the requested modes.
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
            InputTarget::FLAG_DISPATCH_AS_HOVER_EXIT);
    ......
    enqueueDispatchEntryLocked(connection, eventEntry, inputTarget,
            InputTarget::FLAG_DISPATCH_AS_SLIPPERY_ENTER);

    // If the outbound queue was previously empty, start the dispatch cycle going.
    if (wasEmpty && !connection->outboundQueue.isEmpty()) {
        startDispatchCycleLocked(currentTime, connection);
    }
}
```

这个函数首先获取以前的connection的outboundQueue是否为空，然后将该事件调用enqueueDispatchEntryLocked将事件加入到outboundQueue中，如果以前为空，现在不为空，则调用startDispatchCycleLocked开始分发，如果以前的outboundQueue不为空，说明当前的Activity正在处理前面的按键，则不需要再调用startDispatchCycleLocked，因为只要开始处理，会等到队列为空才会停止。

## Step 7、InputDispatcher.startDispatchCycleLocked()

```c
[->frameworks/native/services/inputflinger/InputDispatcher.cpp]

void InputDispatcher::startDispatchCycleLocked(nsecs_t currentTime,
        const sp<Connection>& connection) {

    while (connection->status == Connection::STATUS_NORMAL
            && !connection->outboundQueue.isEmpty()) {
        DispatchEntry* dispatchEntry = connection->outboundQueue.head;
        dispatchEntry->deliveryTime = currentTime;

        // Publish the event.
        status_t status;
        EventEntry* eventEntry = dispatchEntry->eventEntry;
        switch (eventEntry->type) {
        case EventEntry::TYPE_KEY: {
            KeyEntry* keyEntry = static_cast<KeyEntry*>(eventEntry);

            // Publish the key event.
            status = connection->inputPublisher.publishKeyEvent(dispatchEntry->seq,
                    keyEntry->deviceId, keyEntry->source,
                    dispatchEntry->resolvedAction, dispatchEntry->resolvedFlags,
                    keyEntry->keyCode, keyEntry->scanCode,
                    keyEntry->metaState, keyEntry->repeatCount, keyEntry->downTime,
                    keyEntry->eventTime);
            break;
        }
        ......
        }

        ......
        // Re-enqueue the event on the wait queue.
        connection->outboundQueue.dequeue(dispatchEntry);
        traceOutboundQueueLengthLocked(connection);
        connection->waitQueue.enqueueAtTail(dispatchEntry);
        traceWaitQueueLengthLocked(connection);
        }//end of while
}
```

该函数从outboundQueue中取出需要处理的键盘事件，交给connection的inputPublisher去分发，之后将事件加入到connection的waitQueue中。分发事件是通过InputPublisher的publishKeyEvent来完成的。

## Step 8、InputPublisher.publishKeyEvent

```c
[->frameworks/native/libs/input/InputTransport.cpp]

status_t InputPublisher::publishKeyEvent(
        uint32_t seq,int32_t deviceId,int32_t source,
        int32_t action,int32_t flags,int32_t keyCode,
        int32_t scanCode,int32_t metaState,int32_t repeatCount,
        nsecs_t downTime,nsecs_t eventTime) {

    InputMessage msg;
    msg.header.type = InputMessage::TYPE_KEY;
    ......
    msg.body.key.eventTime = eventTime;

    return mChannel->sendMessage(&msg);
}
```

该函数主要是将各个参数封装到InputMessage中，然后交给mChannel对象去分发 mChannel其实是socketpair的server端，其实就是创建的服务器InputChannel，其创建过程稍后详细分析。

## Step 9、InputChannel.sendMessage()

```c
[->frameworks/native/libs/input/InputTransport.cpp]

status_t InputChannel::sendMessage(const InputMessage* msg) {
    size_t msgLength = msg->size();
    ssize_t nWrite;
    do {
        nWrite = ::send(mFd, msg, msgLength, MSG_DONTWAIT | MSG_NOSIGNAL);
    } while (nWrite == -1 && errno == EINTR);
    return OK;
}
```

该函数主要是通过send函数往socketpair的server端写入InputMessage对象，应用程序这一侧正睡眠在client端的fd上，此时client端就会收到该InputMessage，client会进行按键按键分发，应用程序这一侧的按键分发请看下一节。

## 七、App注册消息监听过程分析

总体流程图 InputDispatcher会找到目标窗口，然后通过之前和App间建立的连接发送事件。如果是个需要系统处理的Key事件，这里会封装成CommandEntry插入到mCommandQueue队列中，后面的runCommandLockedInterruptible()函数中会调用doInterceptKeyBeforeDispatchingLockedInterruptible()来让PWM有机会进行处理。最后dispatchOnce()调用pollOnce()从和App的连接上接收处理完成消息。那么，InputDispatcher是怎么确定要往哪个窗口中发事件呢？这里的成员变量mFocusedWindowHandle指示了焦点窗口，然后findFocusedWindowTargetsLocked()会调用一系列函数（handleTargetsNotReadyLocked(), checkInjectionPermission(), checkWindowReadyForMoreInputLocked()等）检查mFocusedWindowHandle是否能接收输入事件。如果可以，将之以InputTarget的形式加到目标窗口数组中。然后就会调用dispatchEventLocked()进行发送。那么，这个mFocusedWindowHandle是如何维护的呢？为了更好地理解，这里回头分析下窗口连接的管理及焦点窗口的管理。 在App端，新的顶层窗口需要被注册到WMS中，这是在ViewRootImpl::setView()中做的。 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N22-Android-Input-System-inputchannel.png)

## Step 1、ViewRootImpl.setView()

```java
[->frameworks/base/core/java/android/view/ViewRootImpl.java]

public void setView(View view, WindowManager.LayoutParams attrs, View panelParentView) {
    // 调用requestLayout来通知InputManagerService当前的窗口是激活的窗口
    requestLayout();
    if ((mWindowAttributes.inputFeatures
            & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
        // 如果该窗口没有指定INPUT_FEATURE_NO_INPUT_CHANNEL属性，则创建消息接收通道InputChannel
        mInputChannel = new InputChannel();
    }
    try {
        // 通过binder调用，调用server端的Session对象来跟WindowManagerService通信，该函数最后会调      
        // 用到WindowManagerService的addWindow函数，函数中会创建一对InputChannel(server/client)，
        // 这样在函数调用结束后，mInputChannel就变成了client端的对象。在
        // frameworks/base/core/java/android/view/IWindowSession.aidl的
        // addToDisplay函数的声明中，InputChannel指定的数据流的流向是out，因此
        // WindowManagerService修改了mInputChannel,客户端就能拿到这个对象的数据了。
        res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
             getHostVisibility(), mDisplay.getDisplayId(),
             mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
             mAttachInfo.mOutsets, mInputChannel);
    } catch (Exception e) {
        ...
    }
    if (mInputChannel != null) {
        if (mInputQueueCallback != null) {
            mInputQueue = new InputQueue();
            mInputQueueCallback.onInputQueueCreated(mInputQueue);
            // 初始化WindowInputEventReceiver，按键消息会从native层传到该对象的onInputEvent函数
            // 中，onInputEvent函数是按键在应用端java层分发的起始端。
            mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
                    Looper.myLooper());
        }
    }
}
```

这个函数与注册键盘消息通道的相关主要有三个功能： 一是调用requestLayout函数来通知InputManagerService，这个Activity窗口是当前被激活的窗口,同时将所有的窗口注册到InputDispatcher中 二是调用mWindowSession的add成员函数来把键盘消息接收通道的server端注册端注册到CPP层的InputManagerService中，client端注册到本应用程序的消息循环Looper中，这样当InputManagerService监控到有键盘消息的时候，就会找到当前被激活的窗口，然后找到其在InputManagerService中对应的键盘消息接收通道(InputChannel)，通过这个通道在InputManagerService的server端来通知应用程序消息循环的client端，这样就把键盘消息分发给当前激活的Activity窗口了 三是应用程序这一侧注册消息接收通道

## Step 2、ViewRootImpl.requestLayout()

```java
[->frameworks/base/core/java/android/view/ViewRootImpl.java]

Override
public void requestLayout() {
    if (!mHandlingLayoutInLayoutRequest) {
        checkThread();
        mLayoutRequested = true;
        scheduleTraversals();
    }    
}
```

这里调用了scheduleTraversals函数来做进一步的操作，该函数调用mChoreographer来post一个Runnable到Looper中，之后Vsycn信号到来会执行mTraversalRunnable中的run方法，即调用doTraversal函数

参考文档：**【Android 7.1.2(Android N) Activity-Window加载显示流程】**

## Step 3、ViewRootImpl.doTraversal()

```java
[->frameworks/base/core/java/android/view/ViewRootImpl.java]

void doTraversal() {
    if (mTraversalScheduled) {
        mTraversalScheduled = false;
        mHandler.getLooper().getQueue().removeSyncBarrier(mTraversalBarrier);
        performTraversals();
    }
}
```

该函数主要是执行performTraversals()函数，进而调用relayoutWindow函数，在该函数中又会调用mWindowSession的relayout进入到java层的WindowManagerService的relayoutWindow函数，该函数会调用mInputMonitor.updateInputWindowsLw(true /force/);mInputMonitor是InputMonitor对象。

## Step 4、InputMonitor.updateInputWindowsLw()

```java
[->frameworks/base/services/core/java/com/android/server/wm/InputMonitor.java]

public void updateInputWindowsLw(boolean force) {
boolean addInputConsumerHandle = mService.mInputConsumer != null;

// Add all windows on the default display.
final int numDisplays = mService.mDisplayContents.size();
for (int displayNdx = 0; displayNdx < numDisplays; ++displayNdx) {
    WindowList windows = mService.mDisplayContents.valueAt(displayNdx).getWindowList();
    for (int winNdx = windows.size() - 1; winNdx >= 0; --winNdx) {
        final WindowState child = windows.get(winNdx);
        final InputChannel inputChannel = child.mInputChannel;
        final InputWindowHandle inputWindowHandle = child.mInputWindowHandle;
        ......
        addInputWindowHandleLw(inputWindowHandle, child, flags, type, isVisible, hasFocus,
                hasWallpaper);
    }
}
// Send windows to native code.
mService.mInputManager.setInputWindows(mInputWindowHandles);
}
```

这个函数将当前系统中带有InputChannel的Activity窗口都设置为InputManagerService的输入窗口，但是后面我们会看到，只有当前激活的窗口才会响应键盘消息。

## Step 5、InputManagerService.setInputWindows()

```java
[->frameworks/base/services/core/java/com/android/server/input/InputManagerService.java]

public void setInputWindows(InputWindowHandle[] windowHandles) {
    nativeSetInputWindows(mPtr, windowHandles);
}
```

这个函数调用了本地方法nativeSetInputWindows来进一步执行操作,mPtr是native层NativeInputManager实例，在调用InputManagerService.nativeInit函数时会在native层构造NativeInputManager对象并将其保存在mPtr中。nativeSetInputWindows会调用NativeInputManager的setInputWindows函数

## Step 6、NativeInputManager.setInputWindows()

```java
[->frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp]

void NativeInputManager::setInputWindows(JNIEnv* env, jobjectArray windowHandleObjArray) {
    Vector<sp<InputWindowHandle> > windowHandles;
    if (windowHandleObjArray) {
        jsize length = env->GetArrayLength(windowHandleObjArray);
        for (jsize i = 0; i < length; i++) {
            jobject windowHandleObj = env->GetObjectArrayElement(windowHandleObjArray, i);
            ......
            sp<InputWindowHandle> windowHandle =
                    android_server_InputWindowHandle_getHandle(env, windowHandleObj);
            if (windowHandle != NULL) {
                windowHandles.push(windowHandle);
            }
            env->DeleteLocalRef(windowHandleObj);
        }
    }

    mInputManager->getDispatcher()->setInputWindows(windowHandles);

    // Do this after the dispatcher has updated the window handle state.
    bool newPointerGesturesEnabled = true;
    size_t numWindows = windowHandles.size();
    for (size_t i = 0; i < numWindows; i++) {
        const sp<InputWindowHandle>& windowHandle = windowHandles.itemAt(i);
        const InputWindowInfo* windowInfo = windowHandle->getInfo();
        if (windowInfo && windowInfo->hasFocus && (windowInfo->inputFeatures
                & InputWindowInfo::INPUT_FEATURE_DISABLE_TOUCH_PAD_GESTURES)) {
            newPointerGesturesEnabled = false;
        }
    }}
```

这个函数首先将Java层的InputWindowHandle转换成C++层的NativeInputWindowHandle，然后放在windowHandles向量中，最后将这些输入窗口设置到InputDispatcher中去。

## Step 7、InputDispatcher.setInputWindows()

```c
[->frameworks/native/services/inputflinger/InputDispatcher.cpp]

void InputDispatcher::setInputWindows(const Vector<sp<InputWindowHandle> >& inputWindowHandles) {
    { // acquire lock
        AutoMutex _l(mLock);

        Vector<sp<InputWindowHandle> > oldWindowHandles = mWindowHandles;
        mWindowHandles = inputWindowHandles;

        sp<InputWindowHandle> newFocusedWindowHandle;
        bool foundHoveredWindow = false;
        for (size_t i = 0; i < mWindowHandles.size(); i++) {
            const sp<InputWindowHandle>& windowHandle = mWindowHandles.itemAt(i);
            if (!windowHandle->updateInfo() || windowHandle->getInputChannel() == NULL) {
                mWindowHandles.removeAt(i--);
                continue;
            }
            if (windowHandle->getInfo()->hasFocus) {
                newFocusedWindowHandle = windowHandle;
            }
        }

        if (mFocusedWindowHandle != newFocusedWindowHandle) {
            mFocusedWindowHandle = newFocusedWindowHandle;
        }
    } // release lock

    // Wake up poll loop since it may need to make new input dispatching choices.
    mLooper->wake();
}
```

这里InputDispatcher的成员变量mFocusedWindowHandle 就代表当前激活的窗口的。这个函数遍历inputWindowHandles，获取获得焦点的窗口，并赋值给mFocusedWindowHandle 这样，InputManagerService就把当前激活的窗口保存在InputDispatcher中了，后面就可以把键盘消息分发给它来处理。

## Step 8、mWindowSession.addToDisplay()

```java
if ((mWindowAttributes.inputFeatures
            & WindowManager.LayoutParams.INPUT_FEATURE_NO_INPUT_CHANNEL) == 0) {
        mInputChannel = new InputChannel();
    }
    try {
        res = mWindowSession.addToDisplay(mWindow, mSeq, mWindowAttributes,
             getHostVisibility(), mDisplay.getDisplayId(),
             mAttachInfo.mContentInsets, mAttachInfo.mStableInsets,
             mAttachInfo.mOutsets, mInputChannel);
    } catch (Exception e) {
        ...
    }
```

这里会调用到WindowManagerService的addWindow接口

```java
frameworks/base/services/core/java/com/android/server/wm/WindowManagerService.java

public int addWindow(Session session, IWindow client, int seq,
WindowManager.LayoutParams attrs, int viewVisibility, int displayId,
Rect outContentInsets, Rect outStableInsets, Rect outOutsets,
InputChannel outInputChannel) {
        ......
        final boolean openInputChannels = (outInputChannel != null
                && (attrs.inputFeatures & INPUT_FEATURE_NO_INPUT_CHANNEL) == 0);
        if  (openInputChannels) {
            win.openInputChannel(outInputChannel);
        }
        ......
    }
```

接着会调用WindowState的 openInputChannel()方法。

```java
frameworks/base/services/core/java/com/android/server/wm/WindowState.java

    void openInputChannel(InputChannel outInputChannel) {

    String name = makeInputChannelName();
    InputChannel[] inputChannels = InputChannel.openInputChannelPair(name);
    mInputChannel = inputChannels[0];
    mClientChannel = inputChannels[1];
    mInputWindowHandle.inputChannel = inputChannels[0];
    if (outInputChannel != null) {
        mClientChannel.transferTo(outInputChannel);
        mClientChannel.dispose();
        mClientChannel = null;
    } else {
        // If the window died visible, we setup a dummy input channel, so that taps
        // can still detected by input monitor channel, and we can relaunch the app.
        // Create dummy event receiver that simply reports all events as handled.
        mDeadWindowEventReceiver = new DeadWindowEventReceiver(mClientChannel);
    }
    mService.mInputManager.registerInputChannel(mInputChannel, mInputWindowHandle);
}
```

这里的outInputChannel即为前面创建的InputChannel，它不为NULL，因此，这里会通过InputChannel.openInputChannelPair函数来创建一对输入通道，其中一个位于WindowManagerService中，另外一个通过outInputChannel参数返回到应用程序中。

WindowManagerService会为每个窗口创建一个WindowState对象，然后将该InputChannel对的service端保存到WindowState中

## Step 10、InputChannel.openInputChannelPair()

```java
[->frameworks/base/core/java/android/view/InputChannel.java]

public static InputChannel[] openInputChannelPair(String name) {
    return nativeOpenInputChannelPair(name);
}
```

调用了nativeOpenInputChannelPair函数，在native创建一个InputChannel对

## Step 11、InputChannel.nativeOpenInputChannelPair()

```c
[->frameworks/base/core/jni/android_view_InputChannel.cpp]

static jobjectArray android_view_InputChannel_nativeOpenInputChannelPair(JNIEnv* env,
        jclass clazz, jstring nameObj) {
    const char* nameChars = env->GetStringUTFChars(nameObj, NULL);
    String8 name(nameChars);
    env->ReleaseStringUTFChars(nameObj, nameChars);

    sp<InputChannel> serverChannel;
    sp<InputChannel> clientChannel;
    status_t result = InputChannel::openInputChannelPair(name, serverChannel, clientChannel);
    ...
    return channelPair;
}
```

nativeOpenInputChannelPair函数调用InputChannel的openInputChannelPair函数创建一对InputChannel,该对象是Native层的InputChannel,跟java层是一一对应的。

## Step 12、InputChannel.openInputChannelPair()

```c
[->frameworks/native/libs/input/InputTransport.cpp]

status_t InputChannel::openInputChannelPair(const String8& name,
        sp<InputChannel>& outServerChannel, sp<InputChannel>& outClientChannel) {
    int sockets[2];
    if (socketpair(AF_UNIX, SOCK_SEQPACKET, 0, sockets)) {
        status_t result = -errno;
        ALOGE("channel '%s' ~ Could not create socket pair.  errno=%d",
                name.string(), errno);
        outServerChannel.clear();
        outClientChannel.clear();
        return result;
    }   

    int bufferSize = SOCKET_BUFFER_SIZE;
    //设置server端和client端的接收缓冲区和发送缓冲区
    setsockopt(sockets[0], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[0], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[1], SOL_SOCKET, SO_SNDBUF, &bufferSize, sizeof(bufferSize));
    setsockopt(sockets[1], SOL_SOCKET, SO_RCVBUF, &bufferSize, sizeof(bufferSize));

    String8 serverChannelName = name;
    serverChannelName.append(" (server)");
    outServerChannel = new InputChannel(serverChannelName, sockets[0]);

    String8 clientChannelName = name;
    clientChannelName.append(" (client)");
    outClientChannel = new InputChannel(clientChannelName, sockets[1]);
    return OK;
}
```

这里调用了socketpair系统调用创建了一对已经连接的UNIX租socket,这里可以把这一对socket当成pipe返回的文件描述符一样使用，pipe返回的管道是单向管道，即只能从一端写入，一端读出，但是socketpair是创建的管道是全双工的，可读可写。

创建好了server端和client端socketpair通道后，在WindowState.openInputChannel()方法中，一方面它把刚才创建的Client端的输入通道通过outInputChannel参数返回到应用程序中：

```c
inputChannels[1].transferTo(outInputChannel);
```

WindowSession.addToDisplay()通过Binder通信与WMS通信。IWindowSession.java为编译Android 7.1.2源码得到。 在此看一下通信详细过程,可以看到outInputChannel通过_arg8.writeToParcel()写入，然后通过跨进程方式传输，App端就可以得到Client端的InputChannel 了。 [->IWindowSession.java$ Stub]

```c
    case TRANSACTION_addToDisplay:
{
......
android.view.InputChannel _arg8;
_arg8 = new android.view.InputChannel();
int _result = this.addToDisplay(_arg0, _arg1, _arg2, _arg3, _arg4, _arg5, _arg6, _arg7, _arg8);
reply.writeNoException();
reply.writeInt(_result);
......
if ((_arg8!=null)) {
reply.writeInt(1);
_arg8.writeToParcel(reply, android.os.Parcelable.PARCELABLE_WRITE_RETURN_VALUE);
}
......
return true;
}
```

[->IWindowSession.java$ Proxy]

```c
//android/out/target/common/obj/JAVA_LIBRARIES/framework_intermediates/src/core/java/android/view/IWindowSession.java

@Override public int addToDisplay(android.view.IWindow window, int seq, android.view.WindowManager.LayoutParams attrs, int viewVisibility, int layerStackId, android.graphics.Rect outContentInsets, android.graphics.Rect outStableInsets, android.graphics.Rect outOutsets, android.view.InputChannel outInputChannel) throws android.os.RemoteException
{
android.os.Parcel _data = android.os.Parcel.obtain();
android.os.Parcel _reply = android.os.Parcel.obtain();
......
mRemote.transact(Stub.TRANSACTION_addToDisplay, _data, _reply, 0);
_result = _reply.readInt();
....
if ((0!=_reply.readInt())) {
outInputChannel.readFromParcel(_reply);
}
return _result;
}
```

另外还需要把server端的InputChannel注册到InputManagerService中：

```c
mInputManager.registerInputChannel(win.mInputChannel, win.mInputWindowHandle);
```

## Step 13、InputManagerService.registerInputChannel()

```java
[->frameworks/base/services/core/java/com/android/server/input/InputManagerService.java]

public void registerInputChannel(InputChannel inputChannel,
        InputWindowHandle inputWindowHandle) {
    nativeRegisterInputChannel(mPtr, inputChannel, inputWindowHandle, false);
    }
```

通过调用nativeRegisterInputChannel来将InputChannel注册到native层

## Step 14、InputManagerService.nativeRegisterInputChannel()

```c
[->frameworks/base/services/core/jni/com_android_server_input_InputManagerService.cpp]

static void nativeRegisterInputChannel(JNIEnv* env, jclass /* clazz */,
        jlong ptr, jobject inputChannelObj, jobject inputWindowHandleObj, jboolean monitor) {
    NativeInputManager* im = reinterpret_cast<NativeInputManager*>(ptr);

    sp<InputChannel> inputChannel = android_view_InputChannel_getInputChannel(env,
            inputChannelObj);

    sp<InputWindowHandle> inputWindowHandle =
            android_server_InputWindowHandle_getHandle(env, inputWindowHandleObj);

    status_t status = im->registerInputChannel(
            env, inputChannel, inputWindowHandle, monitor);
}
```

根据java层的InputWindowHandle获得native层的InputWindowHandle对象，根据java层的InputChannel获得native层的InputChannel对象，然后调用NativeInputManager的resgiterInputChannel，该函数又调用了InputDispatcher的registerInputChannel

## Step 15、InputDispatcher.registerInputChannel()

```c
[->frameworks/native/services/inputflinger/InputDispatcher.cpp]

status_t InputDispatcher::registerInputChannel(const sp<InputChannel>& inputChannel,
        const sp<InputWindowHandle>& inputWindowHandle, bool monitor) {
    { // acquire lock
        AutoMutex _l(mLock);

        sp<Connection> connection = new Connection(inputChannel, inputWindowHandle, monitor);

        int fd = inputChannel->getFd();
        mConnectionsByFd.add(fd, connection);

        if (monitor) {
            mMonitoringChannels.push(inputChannel);
        }    

        mLooper->addFd(fd, 0, ALOOPER_EVENT_INPUT, handleReceiveCallback, this);
    } // release lock

    // Wake the looper because some connections have changed.
    mLooper->wake();
    return OK;
}
```

创建Connection，可以看到用inputChannel初始化了inputPublisher(inputChannel)，这就是之前Input dispatcher小节 Step 8\. InputPublisher.publishKeyEvent()方法中的那个mChannel。

```c
// --- InputDispatcher::Connection ---
InputDispatcher::Connection::Connection(const sp<InputChannel>& inputChannel,
    const sp<InputWindowHandle>& inputWindowHandle, bool monitor) :
    status(STATUS_NORMAL), inputChannel(inputChannel), inputWindowHandle(inputWindowHandle),
    monitor(monitor),
    inputPublisher(inputChannel), inputPublisherBlocked(false) {
    }
```

将InputWindowHandle, InputChanel封装成Connection对象，然后fd作为key，Connection作为Value，保存在mConnectionsByFd中，如果传入的monitor是true，则需要将InputChannel放到mMonitoringChannels中,从上面的InputManagerService的registerInputChannel函数里传入的monitor是false，所以这里不加入到mMonitoringChannels。同时把fd加入到mLooper的监听中，并指定当该fd有内容可读时，Looper就会调用handleReceiveCallback函数。至此server端的InputChannel注册完成，InputDispatcher睡眠在监听的fds上，当有按键事件发生时，InputDispatcher就会往这些fd写入InputMessage对象，进而回调handleReceiveCallback函数。

![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N23-Android-Input-System-InputDispatcher-viewrootimpl.png)

至此，server端的InputChannel就注册完成了，再回到前面的WindowManagerService.addWindow上的第二步inputChannels[1].transferTo(outInputChannel);，这个是将创建的一对InputChannel的client端复制到传入的参数InputChannel上，当addWindow返回时，就回到ViewRootImpl.setView()方法中，执行应用程序这一侧的键盘消息接收通道。

```c
if (mInputChannel != null) {
    mInputEventReceiver = new WindowInputEventReceiver(mInputChannel,
        Looper.myLooper());
}
```

WindowInputEventReceiver继承自InputEventReceiver类。

## Step 16、InputEventReceiver()

```java
[->frameworks/base/core/java/android/view/InputEventReceiver.java]

public InputEventReceiver(InputChannel inputChannel, Looper looper) {

    mInputChannel = inputChannel;
    mMessageQueue = looper.getQueue();
    mReceiverPtr = nativeInit(new WeakReference<InputEventReceiver>(this),
            inputChannel, mMessageQueue);

    mCloseGuard.open("dispose");
}
```

调用了nativeInit执行native层的初始化

## Step 17.、InputEventReceiver.nativeInit()

```c
[->frameworks/base/core/jni/android_view_InputEventReceiver.cpp]

static jlong nativeInit(JNIEnv* env, jclass clazz, jobject receiverWeak,
        jobject inputChannelObj, jobject messageQueueObj) {
    sp<InputChannel> inputChannel = android_view_InputChannel_getInputChannel(env,
            inputChannelObj);

    sp<MessageQueue> messageQueue = android_os_MessageQueue_getMessageQueue(env, messageQueueObj);

    sp<NativeInputEventReceiver> receiver = new NativeInputEventReceiver(env,
            receiverWeak, inputChannel, messageQueue);
    status_t status = receiver->initialize();

    receiver->incStrong(gInputEventReceiverClassInfo.clazz); // retain a reference for the object
    return reinterpret_cast<jlong>(receiver.get());
}
```

函数创建了一个NativeInputEventReceiver对象，并调用其initialize函数

## Step 18.、NativeInputEventReceiver.initialize()

```c
[->frameworks/base/core/jni/android_view_InputEventReceiver.cpp]

status_t NativeInputEventReceiver::initialize() {
    setFdEvents(ALOOPER_EVENT_INPUT);
    return OK;
}
```

调用setFdEvents函数

## Step 19、NativeInputEventReceiver.setFdEvents()

```c
frameworks/base/core/jni/android_view_InputEventReceiver.cpp

void NativeInputEventReceiver::setFdEvents(int events) {
    if (mFdEvents != events) {
        mFdEvents = events;
        int fd = mInputConsumer.getChannel()->getFd();
        if (events) {
            mMessageQueue->getLooper()->addFd(fd, 0, events, this, NULL);
        } else {
            mMessageQueue->getLooper()->removeFd(fd);
        }
    }
}
```

这里调用传入的MessageQueue获取Looper对象，如果events是0，则表示要移除监听fd，如果events不为0，表示要监听fd，这个fd是前面WindowManagerService创建的一对InputChannel的client端，这样当Server端写入事件时，client端的looper就能被唤醒，并调用handleEvent函数（Looper::addFd函数可以指定LooperCallback对象，当fd可读时，会调用LooperCallback的handleEvent，而NativeInputEventReceiver继承自LooperCallback，所以这里会调用NativeInputEventReceiver的handleEvent函数） 贴上事件处理序列图。 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N24-Android-Input-System-Process-input-event.png)

## Step 20、NativeInputEventReceiver.handleEvent()

```c
[->frameworks/base/core/jni/android_view_InputEventReceiver.cpp]

int NativeInputEventReceiver::handleEvent(int receiveFd, int events, void* data) {
    if (events & ALOOPER_EVENT_INPUT) {
        JNIEnv* env = AndroidRuntime::getJNIEnv();
        status_t status = consumeEvents(env, false /*consumeBatches*/, -1, NULL);
        mMessageQueue->raiseAndClearException(env, "handleReceiveCallback");
        return status == OK || status == NO_MEMORY ? 1 : 0;
    }
}
```

该函数调用consumeEvents函数来处理接收一个按键事件

## Step 21、NativeInputEventReceiver.consumeEvents()

```c
[->frameworks/base/core/jni/android_view_InputEventReceiver.cpp]

status_t NativeInputEventReceiver::consumeEvents(JNIEnv* env,
        bool consumeBatches, nsecs_t frameTime, bool* outConsumedBatch) {
    ......
    ScopedLocalRef<jobject> receiverObj(env, NULL);
    bool skipCallbacks = false;
    for (;;) {
        uint32_t seq;
        InputEvent* inputEvent;
        // 处理接收一个按键事件
        status_t status = mInputConsumer.consume(&mInputEventFactory,
                consumeBatches, frameTime, &seq, &inputEvent);
        if (!skipCallbacks) {
            jobject inputEventObj;
            switch (inputEvent->getType()) {
            case AINPUT_EVENT_TYPE_KEY:
                inputEventObj = android_view_KeyEvent_fromNative(env,
                        static_cast<KeyEvent*>(inputEvent));
                break;
            }

            if (inputEventObj) {
                env->CallVoidMethod(receiverObj.get(),
                        gInputEventReceiverClassInfo.dispatchInputEvent, seq, inputEventObj);
                env->DeleteLocalRef(inputEventObj);
            }
        }

        if (skipCallbacks) {
            mInputConsumer.sendFinishedSignal(seq, false);
        }
    }
}
```

函数首先调用mInputConsumer.consume接收一个InputEvent对象,mInputConsumer在NativeInputEventReceiver构造函数中初始化

## Step 22、InputConsumer.consume()

```c
[->frameworks/native/libs/input/InputTransport.cpp]

status_t InputConsumer::consume(InputEventFactoryInterface* factory,
        bool consumeBatches, nsecs_t frameTime, uint32_t* outSeq, InputEvent** outEvent) {

    *outSeq = 0;
    *outEvent = NULL;

    // Fetch the next input message.
    // Loop until an event can be returned or no additional events are received.
    while (!*outEvent) {
        if (mMsgDeferred) {
            // mMsg contains a valid input message from the previous call to consume
            // that has not yet been processed.
            mMsgDeferred = false;
        } else {
            // Receive a fresh message.
            status_t result = mChannel->receiveMessage(&mMsg);
            if (result) {
                // Consume the next batched event unless batches are being held for later.
                if (consumeBatches || result != WOULD_BLOCK) {
                    result = consumeBatch(factory, frameTime, outSeq, outEvent);
                    if (*outEvent) {
                        break;
                    }   
                }   
                return result;
            }   
        }   

        switch (mMsg.header.type) {
        case InputMessage::TYPE_KEY: {
            KeyEvent* keyEvent = factory->createKeyEvent();
            if (!keyEvent) return NO_MEMORY;

            initializeKeyEvent(keyEvent, &mMsg);
            *outSeq = mMsg.body.key.seq;
            *outEvent = keyEvent;
            break;
        }   
   }
   return OK;
}
```

函数首先调用InputChannel的receiveMessage函数接收InputMessage对象，然后根据InputMessage对象调用initializeKeyEvent来构造KeyEvent对象。拿到可KeyEvent对象后，再对到consumeEvents中调用java层的InputEventReceiver.java的dispatchInputEvent函数

```c
env->CallVoidMethod(receiverObj.get(),
                        gInputEventReceiverClassInfo.dispatchInputEvent, seq, inputEventObj);
```

## Step 23、InputEventReceiver.dispatchInputEvent()

```java
[->frameworks/base/core/java/android/view/InputEventReceiver.java]

// Called from native code.
SuppressWarnings("unused")
private void dispatchInputEvent(int seq, InputEvent event) {
    mSeqMap.put(event.getSequenceNumber(), seq);
    onInputEvent(event);
}
```

进而调用onInputEvent函数。至此按键就开始了java层的分发(下一节详细介绍)。

回到主线，故事来没讲完。当App这端处理完输入事件调用ViewRootImpl.finishInputEvent()

```java
    [->/frameworks/base/core/java/android/view/ViewRootImpl.java]

    private void finishInputEvent(QueuedInputEvent q) {
    ......
    if (q.mReceiver != null) {
        boolean handled = (q.mFlags & QueuedInputEvent.FLAG_FINISHED_HANDLED) != 0;
        q.mReceiver.finishInputEvent(q.mEvent, handled);
    }......
    recycleQueuedInputEvent(q);
}
```

Java层InputEventReceiver.nativeFinishInputEvent() 通过JNI 调用android_view_InputEventReceiver.finishInputEvent()

```c
[->frameworks/base/core/jni/android_view_InputEventReceiver.cpp]

status_t NativeInputEventReceiver::finishInputEvent(uint32_t seq, bool handled) {
           if (kDebugDispatchCycle) {
              ALOGD("channel '%s' ~ Finished input event.", getInputChannelName());
          }
          status_t status = mInputConsumer.sendFinishedSignal(seq, handled);
   }
```

层层跳转最后会调用到InputConsumer.sendUnchainedFinishedSignal()发送一个InputMessage::TYPE_FINISHED消息。

```c
[->/frameworks/native/libs/input/InputTransport.cpp]

status_t InputConsumer::sendUnchainedFinishedSignal(uint32_t seq, bool handled) {
InputMessage msg;
msg.header.type = InputMessage::TYPE_FINISHED;
msg.body.finished.seq = seq;
msg.body.finished.handled = handled;
return mChannel->sendMessage(&msg);
}
```

在InputDispatcher.registerInputChannel()中添加了一个 handleReceiveCallback回调。

```c
 mLooper->addFd(fd, 0, ALOOPER_EVENT_INPUT, handleReceiveCallback, this);
```

然后通过和IMS中InputDispacher的通信管道InputChannel发了处理完成通知。那InputDispatcher这边收到后如何处理呢？

![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N25-Android-Input-System-InputDispatcher-handleReceiveCallback.jpg)

由前面分析 InputDispatcher会调用handleReceiveCallback()来处理TYPE_FINISHED信号。这里先是往Command队列里放一个处理事务执行doDispatchCycleFinishedLockedInterruptible()，后面在runCommandsLockedInterruptible()中会取出执行。在doDispatchCycleFinishedLockedInterruptible()函数中，会先调用afterKeyEventLockedInterruptible()。Android中可以定义一些Fallback键，即如果一个Key事件App没有处理，可以Fallback成另外默认的Key事件，这是在这里的dispatchUnhandledKey()函数中进行处理的。接着InputDispatcher会将该收到完成信号的事件项从等待队列中移除。同时由于上一个事件已被App处理完，就可以调用startDispatchCycleLocked()来进行下一轮事件的处理了。

```c
[->/frameworks/native/services/inputflinger/InputDispatcher.cpp]

if (dispatchEntry == connection->findWaitQueueEntry(seq)) {  
    connection->waitQueue.dequeue(dispatchEntry);  
    ...  
}  
 // Start the next dispatch cycle for this connection.  
  startDispatchCycleLocked(now(), connection);
```

startDispatchCycleLocked函数会检查相应连接的输出缓冲中(connection->outboundQueue)是否有事件要发送的，有的话会通过InputChannel发送出去。

总结： 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N26-Android-Input-System-Input-kernel-driver-framwork-app.png)

## 8、Android Input子系统之java层按键传递

Android开发中在自定义Activity以及View时经常会重写onKeyDown,onKeyUp,dispatchKeyEvent，同时View还有setOnKeyListener等，当一个按键事件发生时，这些方法将会被回调，但是到底哪个先回调，哪个后回调呢，一直不是特别清楚，只知道个大概，下面将详细讲述按键在java层的分发过程，其中重点关注按键事件在View层次中的分发

java层的按键分发从ViewRootImpl.java的WindowInputEventReceiver中的onInputEvent开始，从前面的应用程序注册消息监听过程分析和Input Dispatcher分析，InputDispatcher在处理按键事件时，会通过InputChannel::sendMessage函数将按键消息从server端写入，这里的InputChannel是当前获取焦点的窗口的InputChannel对的server端，这样应用程序端就可以收到该消息，然后调用NativeInputEventReceiver的handleEvent,最后调用到InputEventReceiver的onInputEvent函数（具体的可以看应用程序注册消息监听过程分析 的Step20-Step23）

序列图： 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N27-Android-Input-System-input-stage.png)

## Step 1、WindowInputEventReceiver.onInputEvent()

```java
[->frameworks/base/core/java/android/view/ViewRootImpl.java]

final class WindowInputEventReceiver extends InputEventReceiver {
    public WindowInputEventReceiver(InputChannel inputChannel, Looper looper) {
        super(inputChannel, looper);
    }

    @Override
    public void onInputEvent(InputEvent event) {
        enqueueInputEvent(event, this, 0, true);
    }
    ...
}
```

这里只列出部分代码，当一个按键按下时onInputEvent方法就会被回调，其中调用了ViewRootImpl::enqueueInputEvent(event, this, 0, true);

## Step 2、ViewRootImpl.enqueueInputEvent()

```java
[->frameworks/base/core/java/android/view/ViewRootImpl.java]

void enqueueInputEvent(InputEvent event,
        InputEventReceiver receiver, int flags, boolean processImmediately) {
    // 从队列中获取一个QueuedInputEvent，这里的flags传入的是0
    QueuedInputEvent q = obtainQueuedInputEvent(event, receiver, flags);
    ...
    if (processImmediately) {
        doProcessInputEvents();//这里传入的processImmediately是true，所以调用doProcessInputEvents
    } else {
        scheduleProcessInputEvents();
    }
}
```

从前面的参数可知，这里表示要立即处理，所以调用doProcessInputEvents函数.

## Step 3、ViewRootImpl.doProcessInputEvents()

```java
frameworks/base/core/java/android/view/ViewRootImpl.java
void doProcessInputEvents() {
    // Deliver all pending input events in the queue.
    while (mPendingInputEventHead != null) {
        QueuedInputEvent q = mPendingInputEventHead;
        mPendingInputEventHead = q.mNext;
        if (mPendingInputEventHead == null) {
            mPendingInputEventTail = null;
        }
        q.mNext = null;

        mPendingInputEventCount -= 1;
        ...
        // 分发按键事件
        deliverInputEvent(q);
    }
}
```

在deliverInputEvent函数中实际做按键的分发

## Step 4、ViewRootImpl.deliverInputEvent()

```java
[->frameworks/base/core/java/android/view/ViewRootImpl.java]

private void deliverInputEvent(QueuedInputEvent q) {
    Trace.asyncTraceBegin(Trace.TRACE_TAG_VIEW, "deliverInputEvent",
            q.mEvent.getSequenceNumber());
    if (mInputEventConsistencyVerifier != null) {
        mInputEventConsistencyVerifier.onInputEvent(q.mEvent, 0);
    }

    InputStage stage;
    if (q.shouldSendToSynthesizer()) {
        stage = mSyntheticInputStage;
    } else {
        //选择责任链的模式的入口，如果InputEvent需要跳过IME处理，则从mFirstPostImeInputStage（EarlyPostImeInputStage）开始,否则从mFirstInputStage(NativePreImeInputStage)开始分发
        stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage;
    }

    if (stage != null) {
        stage.deliver(q);
    } else {
        finishInputEvent(q);
    }
}
```

这里调用了InputStage的deliver方法分发，这里的InputStage代表了输入事件的处理阶段，是一种责任链模式 InputStage将输入事件的处理分成若干个阶段（Stage）, 如果当前有输入法窗口，则事件处理从 NativePreIme 开始，否则的话，从EarlyPostIme 开始。事件会依次经过每个Stage，如果该事件没有被标识为 "Finished"， 该Stage就会处理它，然后返回处理结果，Forward 或 Finish， Forward 运行下一Stage继续处理，而Finished事件将会简单的Forward到下一级，直到最后一级 Synthetic InputStage。流程图和每个阶段完成的事情如下图所示 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N28-Android-Input-System-input-stage-GUI.png)

责任链模式： 责任链模式（Chain of Responsibility）的目标是使多个对象都有机会处理请求，从而避免请求的发送者和接收者之间的耦合关系。将这些对象连成一条链，并沿着这条链传递请求，直到有一个对象处理它为止。

按键分发： 在ViewRootImpl的setView函数中会构造一个如图所示的InputStage的链，按键会从入口阶段，进入责任链，顺序处理，入口阶段根据QueuedInputEvent的状态来决定。q.shouldSendToSynthesizer() 这里一般是false，因此主要看stage = q.shouldSkipIme() ? mFirstPostImeInputStage : mFirstInputStage; 这里的shouldSkipIme其实是一个flag在构造QueuedInputEvent时传入的，从前面的onInputEvent调用的enqueueInputEvent(event, this, 0, true);可知，这里传入的flags是第三个参数0，那这里的shouldSkipIme就是false，那么按键会从mFirstPostImeInputStage 开始分发，就是图中的NativePreImeInputStage分发。

下面只从跟本文前面提到的Activity，View的按键分发流程相关的InputStage（ViewPostImeInputStage）开始分析

## Step 5、ViewPostImeInputStage.processKeyEvent()

```java
[->frameworks/base/core/java/android/view/ViewRootImpl.java]

private int processKeyEvent(QueuedInputEvent q) {
    final KeyEvent event = (KeyEvent)q.mEvent;
    ...
    // Deliver the key to the view hierarchy.
    // 调用成员变量mView的dispatchKeyEvent函数，这里mView是PhoneWindow.DecorView对象
    if (mView.dispatchKeyEvent(event)) {
        return FINISH_HANDLED;
    }
    ...
    // 如果按键是四向键或者是TAB键，则移动焦点
    // Handle automatic focus changes.
    if (event.getAction() == KeyEvent.ACTION_DOWN) {
        int direction = 0;
        switch (event.getKeyCode()) {
            case KeyEvent.KEYCODE_DPAD_LEFT:
                if (event.hasNoModifiers()) {
                    direction = View.FOCUS_LEFT;
                }
                break;
            ......
            case KeyEvent.KEYCODE_TAB:
                if (event.hasNoModifiers()) {
                    direction = View.FOCUS_FORWARD;
                } else if (event.hasModifiers(KeyEvent.META_SHIFT_ON)) {
                    direction = View.FOCUS_BACKWARD;
                }
                break;
        }
        if (direction != 0) {
            View focused = mView.findFocus();
            if (focused != null) {
                View v = focused.focusSearch(direction);
                if (v != null && v != focused) {
                    ......
                    focused.getFocusedRect(mTempRect);
                    if (mView instanceof ViewGroup) {
                        ((ViewGroup) mView).offsetDescendantRectToMyCoords(
                                focused, mTempRect);
                        ((ViewGroup) mView).offsetRectIntoDescendantCoords(
                                v, mTempRect);
                    }
                    if (v.requestFocus(direction, mTempRect)) {
                        playSoundEffect(SoundEffectConstants
                                .getContantForFocusDirection(direction));
                        return FINISH_HANDLED;
                    }
                }

                // Give the focused view a last chance to handle the dpad key.
                if (mView.dispatchUnhandledMove(focused, direction)) {
                    return FINISH_HANDLED;
                }
            } else {
                // find the best view to give focus to in this non-touch-mode with no-focus
                View v = focusSearch(null, direction);
                if (v != null && v.requestFocus(direction)) {
                    return FINISH_HANDLED;
                }
            }
        }
    }
    return FORWARD;
}
```

上述主要分两步： 第一步是调用PhoneWindow.DecorView的dispatchKeyEvent函数，DecorView是View层次结构的根节点，按键从根节点开始根据Focuse view的path自上而下的分发。 第二步是判断按键是否是四向键，或者是TAB键，如果是则需要移动焦点

## Step 6、mView.dispatchKeyEvent()

```java
public boolean dispatchKeyEvent(KeyEvent event) {  
    ...
    if (!isDestroyed()) {
        final Callback cb = getCallback();
        final boolean handled = cb != null && mFeatureId < 0 ? cb.dispatchKeyEvent(event)
                : super.dispatchKeyEvent(event);
        if (handled) {
            return true;
        }    
    }    

    return isDown ? PhoneWindow.this.onKeyDown(mFeatureId, event.getKeyCode(), event)
            : PhoneWindow.this.onKeyUp(mFeatureId, event.getKeyCode(), event);
}
```

主要的分发在下面开始，如果cb不为空并且mFeatureId小于0，则调用cb.dispatchKeyEvent开始分发，否则会调用DecorView的父类（View）的dispatchKeyEvent函数。cb是Window.Callback类型，Activity实现了Window.Callback接口，在attach函数中，会调用Window.setCallback函数将自己注册进PhoneWindow中，所以cb不为空。在PhoneWindow初始化时会调用installDecor函数生成DecorView对象，该函数中传入的mFeatureId是-1，所以mFeatureId也小于0。因此此处会调用Activity的dispatchKeyEvent函数，开始在View中分发按键。

下面来分析按键在View的层次结构中是如何分发的 DecorView的按键分发

接下来来看这里先看看Activity(Callback)的dispatchKeyEvent实现：

## Step 7、Activity.dispatchKeyEvent()

```java
[frameworks/base/core/java/android/app/Activity.java]

public boolean dispatchKeyEvent(KeyEvent event) {
    //调用自定义的onUserInteraction
    onUserInteraction();

    Window win = getWindow();
    //调用PhoneWindow的superDispatchKeyEvent,实际调用DecorView的superDispatchKeyEvent，从DecorView开始从顶层View往子视图传递
    if (win.superDispatchKeyEvent(event)) {  
        return true;
    }
    View decor = mDecor;
    if (decor == null) decor = win.getDecorView();
    //到这里如果view层次结构没有返回true则交给KeyEvent本身的dispatch方法，Activity的onKeyDown/onKeyUp/onKeyMultiple就会被触发
    return event.dispatch(this, decor != null
            ? decor.getKeyDispatcherState() : null, this);
}
```

接着看下PhoneWindow的superDispatchKeyEvent

## Step 8、PhoneWindow.superDispatchKeyEvent()

```java
<!-- PhoneWindow.java -->
Override
public boolean superDispatchKeyEvent(KeyEvent event) {
    return mDecor.superDispatchKeyEvent(event);
}

<!-- PhoneWindow.DecorView -->
public boolean superDispatchKeyEvent(KeyEvent event) {
    // Give priority to closing action modes if applicable.
    if (event.getKeyCode() == KeyEvent.KEYCODE_BACK) {
        final int action = event.getAction();
        // Back cancels action modes first.
        if (mPrimaryActionMode != null) {
            if (action == KeyEvent.ACTION_UP) {
                mPrimaryActionMode.finish();
            }
            return true;
         }
    }
    //进入View的层次结构，调用ViewGroup.dispatchKeyEvent
    return super.dispatchKeyEvent(event);
}
```

再看ViewGroup的dispatchKeyEvent函数

## Step 9、ViewGroup.dispatchKeyEvent()

```java
[->frameworks/base/core/java/android/view/ViewGroup.java]

public boolean dispatchKeyEvent(KeyEvent event) {
    ...
    if ((mPrivateFlags & (PFLAG_FOCUSED | PFLAG_HAS_BOUNDS))
            == (PFLAG_FOCUSED | PFLAG_HAS_BOUNDS)) {
        //如果此ViewGroup是focused并且具体的大小被设置了（有边界），则交给它处理，即调用View的实现
        if (super.dispatchKeyEvent(event)) {
            return true;
        }    
    } else if (mFocused != null && (mFocused.mPrivateFlags & PFLAG_HAS_BOUNDS)
            == PFLAG_HAS_BOUNDS) {
        //否则，如果此ViewGroup中有focused的child，且child有具体的大小，则交给mFocused处理
        if (mFocused.dispatchKeyEvent(event)) {
            return true;
        }    
    }    
    ...
    return false;
}
```

这里可以看出如果ViewGroup满足条件，则优先处理事件而不发给子视图去处理。

下面看下View的dispatchKeyEvent实现

## Step 10、View.dispatchKeyEvent()

```java
[->frameworks/base/core/java/android/view/View.java]

public boolean dispatchKeyEvent(KeyEvent event) {
    ...
    // Give any attached key listener a first crack at the event.
    //noinspection SimplifiableIfStatement
    ListenerInfo li = mListenerInfo;
    //调用onKeyListener，如果注册了OnKeyListener,并且View属于Enable状态，则触发
    if (li != null && li.mOnKeyListener != null && (mViewFlags & ENABLED_MASK) == ENABLED
            && li.mOnKeyListener.onKey(this, event.getKeyCode(), event)) {
        return true;
    }     
    //调用KeyEvent.dispatch方法，并将view作为参数传递进去，实际会回调View的onKeyUp/onKeyDown等方法
    if (event.dispatch(this, mAttachInfo != null
            ? mAttachInfo.mKeyDispatchState : null, this)) {
        return true;
    }     
    ...  
    return false;
}
```

## Step 11、View.onKeyDown/View.onKeyUp

```java

[->frameworks/base/core/java/android/view/View.java]

public boolean onKeyDown(int keyCode, KeyEvent event) {
    boolean result = false;
    //处理KEYCODE_DPAD_CENTER、KEYCODE_ENTER按键
    if (KeyEvent.isConfirmKey(keyCode)) {
        if ((mViewFlags & ENABLED_MASK) == DISABLED) {
            //disabled的view直接返回true，不再继续分发,即Activity的onKeyDown和onKeyUp无法收到KEYCODE_DPAD_CENTER、KEYCODE_ENTER事件
            return true;
        }
        // Long clickable items don't necessarily have to be clickable
        if (((mViewFlags & CLICKABLE) == CLICKABLE ||
                (mViewFlags & LONG_CLICKABLE) == LONG_CLICKABLE) &&
                (event.getRepeatCount() == 0)) {// clickable或者long_clickable且是第一次down事件
            setPressed(true);// 标记pressed，你可能设置了View不同的background，这时候就会有所体现（比如高亮效果）
            checkForLongClick(0);
            return true;
        }
    }
    return result;
}

public boolean onKeyUp(int keyCode, KeyEvent event) {
    //处理KEYCODE_DPAD_CENTER、KEYCODE_ENTER按键
    if (KeyEvent.isConfirmKey(keyCode)) {
        if ((mViewFlags & ENABLED_MASK) == DISABLED) {
            //disabled的view直接返回true，不再继续分发,即Activity的onKeyDown和onKeyUp无法收到KEYCODE_DPAD_CENTER、KEYCODE_ENTER事件
            return true;
        }
        if ((mViewFlags & CLICKABLE) == CLICKABLE && isPressed()) {
            setPressed(false);

            if (!mHasPerformedLongPress) {
                // This is a tap, so remove the longpress check
                removeLongPressCallback();
                return performClick();
            }
        }
    }
    return false;
}
```

## Step 12、Activity.onKeyDown/onKeyUp

```java
[->frameworks/base/core/java/android/app/Activity.java]

public boolean onKeyDown(int keyCode, KeyEvent event)  {
    //如果是back键则启动追踪
    if (keyCode == KeyEvent.KEYCODE_BACK) {
        if (getApplicationInfo().targetSdkVersion
                >= Build.VERSION_CODES.ECLAIR) {
            event.startTracking();
        } else {
            onBackPressed();
        }    
        return true;
    }    
    ...
}

public boolean onKeyUp(int keyCode, KeyEvent event) {
    if (getApplicationInfo().targetSdkVersion
            >= Build.VERSION_CODES.ECLAIR) {
        if (keyCode == KeyEvent.KEYCODE_BACK && event.isTracking()
                && !event.isCanceled()) {
            //如果是back键并且正在追踪该Event，则调用onBackPressed
            onBackPressed();
            return true;
        }
    }
    return false;
}
```

而Android常见Touch事件是通过dispatchPointerEvent(MotionEvent event)分发的，主要跟底层传上来的 输入事件相关，不同类型事件分别处理。 具体Touch事件分发机制可参考博客： [Android事件分发机制完全解析，带你从源码的角度彻底理解(上)](http://blog.csdn.net/guolin_blog/article/details/9097463/) [Android事件分发机制完全解析，带你从源码的角度彻底理解(下)](http://blog.csdn.net/guolin_blog/article/details/9153747/) [Android触摸屏事件派发机制详解与源码分析一(View篇)](http://blog.csdn.net/yanbober/article/details/45887547) [Android触摸屏事件派发机制详解与源码分析二(ViewGroup篇)](http://blog.csdn.net/yanbober/article/details/45912661) [Android触摸屏事件派发机制详解与源码分析三(Activity篇)](http://blog.csdn.net/yanbober/article/details/45932123) [Android Deeper(00) - Touch事件分发响应机制](http://hukai.me/android-deeper-touch-event-dispatch-process/) 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N29-Android-Input-System-touch-event.png)

## 九、总结：

再贴一下Input system总体框架图： 
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N30-Android-Input-System-input-system-framwork.png)
![Markdown](https://raw.githubusercontent.com/iizhoujinjian/PicGo/master/android.input/N31-Android-Input-System-Input-kernel-driver-framwork-app.png)

## （一）、IMS初始化&& IMS与App建立通信：

1. SystemServer初始化过程中，创建InputManagerService，IMS第一件事情就是初始化Native层，包括EventHub, InputReader 和 InputDispatcher

2. IMS以及其他的System Service 初始化完成之后，应用程序就开始启动。如果一个应用程序有Activity（只有Activit能够接受用户输入），它要将自己的Window(ViewRootImpl)通过setView()注册到WindowManagerService 中

3. 用户输入的捕捉和处理发生在不同的进程里（生产者：Input Reader 和 Input Dispatcher 在System Server 进程里，而消费者，应用程序运行在自己的进程里），因此用户输入事件（Event)的传递需要跨进程。在这里，Android使用了Socket + Binder来完成。OpenInputChannelPair 生成了两个Socket的FD， 代表一个双向通道的两端，向一端写入数据，另外一端便可以读出，反之依然，如果一端没有写入数据，另外一端去读，则陷入阻塞等待。OpenInputChannelPair() 发生在WindowManager Service.addWindow()中

4. 通过RegisterInputChannel, WindowManagerService 将刚刚创建的一个Socket FD，封装在InputWindowHandle(代表一个WindowState) 里传给InputManagerService

5. InputManagerService 通过JNI（NativeInputManager）最终调用到了InputDispatcher 的 RegisterInputChannel()方法，这里，一个Connection 对象被创建出来，代表与远端某个窗口(InputWindowHandle)的一条用户输入数据通道。一个Dispatcher可能有多个Connection（多个Window）同时存在。为了监听来自于Window的消息，InputDispatcher 通过AddFd 将这些个FD 加入到Looper中，这样，只要某个Window在Socket的另一端写入数据，Looper就会马上从睡眠中醒来，进行处理。

6. 到这里，ViewRootImpl mWindowSession.addToDisplay返回，WMS 将SocketPair的另外一个FD 放在返回参数 OutputChannel 里，即返回给APP进程。

7. 接着ViewRootImpl 创建了WindowInputEventReceiver 用于接受InputDispatchor 传过来的事件，App进程同样通过AddFd() 将读端的Socket FD 加入到Looper中，这样一旦InputDispatchor发送Event，Looper就会立即醒来处理。

## （二）、Eventhub 和 Input Reader

1. NativeInputManager的构造函数里第一件事情就是创建一个EventHub对象，EventHub构造函数里主要生成并初始化几个控制的FD

2. mINotifyFd: 用来监控""/dev/input"目录下是否有文件生成，有的话说明有新的输入设备接入，EventHub将从epool_wait中唤醒，来打开新加入的设备 mWakeReaderFD， mWakeWriterFD： 一个Pipe的两端，当往mWakeWriteFD 写入数据的时候，等待在mWakeReaderFD的线程被唤醒，这里用来给上层应用提供唤醒等待线程，比如说，当上层应用改变输入属性需要EventHub进行相应更新时

3. mEpollFD，用于epoll_wait()的阻塞等待，这里通过epoll_ctrl(EPOLL_ADD_FD, fd) 可以等待多个fd的事件，包括上面提到的mINotifyFD, mWakeReaderFD, 以及输入设备的FD。

4. InputManagerService启动InputReader 线程，进入无限的循环，每次循环调用loopOnce(). 第一次循环，会主动扫描 "/dev/input/" 目录，并打开下面的所有文件，通过ioctl()从底层驱动获取设备信息，并判断它的设备类型。这里处理的设备类型有：INPUT_DEVICE_CLASS_KEYBOARD， INPUT_DEVICE_CLASS_TOUCH， INPUT_DEVICE_CLASS_DPAD，INPUT_DEVICE_CLASS_JOYSTICK 等。

5. 找到每个设备对应的键值映射文件，读取并生产一个KeyMap 对象。一般来说，设备对应的键值映射文件是 "/system/usr/keylayout/Generic.kl".

6. 将刚才扫描到的/dev/input 下所有文件的FD 加到epool等待队列中，调用epool_wait() 开始等待事件的发生。

7. 某个时间发生，可能是用户按键输入，也可能是某个设备插入，亦或用户调整了设备属性，epoll_wait() 返回，将发生的Event 存放在mPendingEventItems 里。如果这是一个用户输入，系统调用Read() 从驱动读到这个按键的信息，存放在rawEvents里。

8. EventHub->getEvents() 返回,代表有新的input事件到来，进入InputReader的processEventLocked函数。

9. 通过rawEvent 找到产生时间的Device，再找到这个Device对应的InputMapper对象，最终生成一个NotifyArgs对象，将其放到NotifyArgs的队列中。

10. 调用NotifyArgs里面的Notify()方法，最终调用到InputDispatchor 对应的Notify接口（比如NotifyKey) 将接下来的处理交给InputDispatchor，EventHub 和 InputReader 工作结束，但马上又开始新的一轮等待，重复6～9的循环。

## （三）、Input Dispatcher

1. 接上节的最后一步，NotifyKey() 的实现在Input Dispatcher 内部，他首先做简单的校验，对于按键事件，只有Action 是 AKEY_EVENT_ACTION_DOWN 和 AKEY_EVENT_ACTION_UP，即按下和弹起这两个Event别接受。

2. Input Reader 传给Input Dispather的数据类型是 NotifyKeyArgs， 后者在这里将其转换为 KeyEvent, 然后交由 Policy 来进行第一步的解析和过滤，interceptKeyBeforeDispatching(), 对于手机产品，这个工作是在PhoneWindowManager 里完成，（不同类型的产品可以定义不同的WindowManager, 比如GoogleTV 里用到的是TVWindowManager)。KeyEvent 在这里将会被分为三类：

  1. System Key: 比如说 音量键，Power键，以及一些特殊的组合键，如用于截屏的音量+Power，等等。部分System Key 会在这里立即处理，比如说电话键，但有一些会放到后面去做处理，比如说音量键，但不管怎样，这些键不会传给应用程序，所以称为系统键。
  2. Global Key：最终产品中可能会有一些特殊的按键，它不属于某个特定的应用，在所有应用中的行为都是一样，但也不包含在Andrioid的系统键中，比如说GoogleTV 里会有一个"TV" 按键，按它会直接呼起"TV"应用然后收看电视直播，这类按键在Android定义为Global Key.
  3. User Key：除此之外的按键就是User Key, 它最终会传递到当前的应用窗口。

3. 此时，InputDispather 还在Looper中睡眠等待，mLooper->wake();将其唤醒，然后进入Input Dispatcher 线程。

4. InputDispatcher 大部分的工作在 dispatcherOnce 里完成。首先从mInBoundQueue 中读出队列头部的事件 mPendingEvent, 然后调用 pokeUserActivityLocked()。 poke的英文意思是"搓一下, 捅一下"， 这个函数的目的也就是"捅一下"PowerManagerService 提醒它"别睡眠啊，我还活着呢"，最终调用到PowerManagerService 的 updatePowerStateLocked()，防止手机进入休眠状态。需要注意的是，上述动作不会马上执行，而是存储在命令队列，mCommandQueue里，这里面的命令会在后面依次被执行。

5. 接下来是dispatchOnceInnerLocked()->dispatchKeyLocked() 第一次进去这个函数的时候，先检查Event是否已经过处理（doInterceptKeyBeforeDispatchingLockedInterruptible), 如果没有，则生成一个命令，同样放入mCommandQueue里。

6. runCommandsLockedInterruptible() 依次执行mCommandQueue 里的命令，前面说过，pokeUserActivity 会调用PowerManagerService 的 updatePowerStateLocked(), 而 interceptKeyBeforeDispatching() 则最终调用到PhoneWindowManager的同名函数。我们在interceptBeforeQueuing 里面提到的一些系统按键在这个被执行，比如 HOME/MENU/SEARCH 等。

7. 命令运行完之后，退出 dispatchOnce， 然后调用pollOnce 进入下一轮等待。但这里不会被阻塞，因为timeout值被设成了0.

8. 第二次进入dispatchKeyLocked(), 这是Event的状态已经设为"已处理"，这时候才真正进入了发射阶段。

9. 接下来调用 findFocusedWindowTargetLocked() 获取当前的焦点窗口，这里面会做一件非常重要的事情，就是检测目标应用是否有ANR发生，如果下诉条件满足，则说明可能发生了ANR：

  1. 目标应用不会空，而目标窗口为空。说明应用程序在启动过程中出现了问题。
  2. 目标 Activity 的状态是Pause，即不再是Focused的应用。
  3. 目标窗口还在处理上一个事件。这个我们下面会说到。

10. 如果目标窗口处于正常状态，调用dispatchEventLocked() 进入真正的发送程序。

11. 然后调用prepareDispatchCycleLocked() ,这里事件换了一件马甲，从EventEntry 变成 DispatchEntry, 并送人mOutBoundQueue。然后调用startDispatchCycleLocked() 开始发送。

12. 最终的发送发生在InputChannel的sendMessage()。这里就用到了我们前面提到的SocketPair, 一旦sendMessage() 执行，目标窗口所在进程的Looper线程就会被唤醒，然后读取键值并进行处理。

13. 乖乖，还没走完啊？是的，工作还差最后一步，Input Dispatcher给这个窗口发送下一个命令之前，必须等待该窗口的回复，如果超过5s没有收到，就会通过Input Manager Service 向Activity Manager 汇报，后者会弹出我们熟知的 "Application No Response" 窗口。所以，事件会放入mWaitQueue进行暂存。如果窗口一切正常，完成按键处理后它会调用InputConsumer的sendFinishedSignal() 往SocketPair 里写入完成信号，Input Dispatcher 从 Loop中醒来，并从Socket中读取该信号，然后从mWaitQueue 里清除该事件标志其处理完毕。

## （四）、Key Processing

略、请参考： [图解Android - Android GUI 系统 (5) - Android的Event Input System - 漫天尘沙 - 博客园](https://www.cnblogs.com/samchen2009/p/3368158.html)

## 参考文档(特别感谢)：

[韦东山第4期Android驱动深度开发视频源码-GitHub](https://github.com/weidongshan)
[韦东山第4期Android驱动深度开发视频-输入系统-100ask.org](http://www.100ask.org/index.html)
[Android输入子系统-ChenWeiaiYanYan](http://blog.csdn.net/chenweiaiyanyan/article/category/6948320)
[《深入理解Android 卷III》第五章 深入理解Android输入系统 - CSDN博客](http://blog.csdn.net/innost/article/details/47660387)
[图解Android - Android GUI 系统 (5) - Android的Event Input System - 漫天尘沙 - 博客园](https://www.cnblogs.com/samchen2009/p/3368158.html)
[Android 5.0(Lollipop)事件输入系统(Input System) - 世事难料，保持低调 - CSDN博客](http://blog.csdn.net/jinzhuojun/article/details/41909159)
[【Android】Android输入子系统 - Leo.cheng - 博客园](http://www.cnblogs.com/lcw/p/3506110.html)
[Android(Linux) 输入子系统解析 | Andy.Lee's Blog](http://huaqianlee.github.io/2017/11/23/Android/Android-Linux-input-system-analysis/)
[INPUT事件的读取和分发：INPUTREADER、INPUTDISPATCHER](http://www.cheelok.com/aosp/59)
[Android 触摸事件分发机制](http://www.hustmeituan.club/tag/android.html)
[深入理解Android之Touch事件的分发](https://www.jianshu.com/p/84b2e0038080)

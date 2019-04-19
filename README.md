# Libcurl
# libcurl Know How

[TOC]

### 1.`libcurl`概述

`libcurl`有两套调用机制，`easy`模式和`multi`模式。`easy`模式都是同步接口，能够同步地、高效地、迅速地进行通信；`multi`模式的接口比`easy`模式的接口更复杂，且`multi`的接口是依赖`easy`的接口，`multi`模式是`libcurl`的异步调用方式，可以在单线程和多线程中提供多个传输通道。

总结：

- [ ] `easy`接口是同步阻塞的，当程序需要进行多个`curl`并发请求时，`easy`接口无法正常处理，必须等待上一次结束。
- [ ] `multi`接口是异步非阻塞的，依赖`easy`接口，可以同时处理多请求，虽然`multi`的`perform`函数是非阻塞的，但是带来的缺点是需要我们自己去阻塞。

### 2.`libcurl easy`模式下函数基本使用

#### (1).基本函数使用方法：

- 全局初始化：**`CURLcode curl_global_init(long flags );`**

  程序必须在全局上初始化一些`libcurl`功能。 该函数只能调用一次，无论使用`lib`多少次。 在程序的整个生命周期中只调用一次。

  参数`flags`：

  `CURL_GLOBAL_ALL`：初始化所有的可能的调用

  `CURL_GLOBAL_SSL`：初始化支持 安全套接字层

  `CURL_GLOBAL_WIN32`：初始化win32套接字库

  `CURL_GLOBAL_NOTHING`：没有额外的初始化

  `CURL_GLOBAL_DEFAULT`：默认的初始化，初始化`SSL`和`WIN32`

  `CURL_GLOBAL_ACK_EINTR`：设置了这个标签后，当`curl`在连接或等待数据请求时，`curl`将接收`EINTR`条件，否则，`curl`会一直等待。

  **注：如果这个函数在curl_easy_init函数调用时还没被调用，它将由libcurl库自动调用，所以多线程下最好主动调用该函数以防止在多线程中curl_easy_init时多次调用。curl_global_init是不能保证线程安全的，所以不要在每个线程中都调用curl_global_init，应该将该函数的调用放在主线程中。**

- 全局`clean`：**`void curl_global_cleanup(void);`**

  该函数释放由`curl_global_init`函数请求的资源。该函数必须在`curl_global_init`函数之后调用，关闭`curl`库之前必须调用。

  **注：该函数也是非线程安全的。**

- 会话开始：**`CURL *curl_easy_init( );`**

  `curl_easy_init`用来初始化一个`CURL`的指针(有些像返回`FILE`类型的指针一样)， 相应的在调用结束时要用`curl_easy_cleanup`函数清理。
  一般`curl_easy_init`意味着一个会话的开始. 它会返回一个`easy_handle`(`CURL*`对象)，用在`easy`系列的函数中。

- 会话结束：**`void curl_easy_cleanup(CURL * handle );`**

  调用该函数来结束一个`curl easy`会话。关闭一个由`curl_easy_init`生成的*`handle`*。

- 设置属性和选项：**`CURLcode curl_easy_setopt(CURL *handle, CURLoption option, parameter);`**

  `curl_easy_setopt`函数设置适当的参数，改变`libcurl`的行为。基本所有的操作都是通过这个函数设置的。`option`参数是一系列的行为操作。`parameter`取决于`option`参数的设置。

- 执行会话：**`CURLcode curl_easy_perform(CURL * easy_handle );`**

  执行会话的操作。该函数在`curl_easy_init`函数和所有的`curl_easy_setopt`函数设置完后调用，执行所有设置的操作。

- 获取会话信息：**`CURLcode curl_easy_getinfo(CURL *curl, CURLINFO info, ... );`**

  发出`http/https`请求后，服务器会返回应答头信息和应答数据。该函数`info`参数就是我们需要获取的内容，下面是一些参数值:
  ​    获取应答码：`CURLINFO_RESPONSE_CODE`
  ​    头大小 ：`CURLINFO_HEADER_SIZE`
  ​    除了获取应答信息外，这个函数还能获取curl的一些内部信息，如请求时间、连接时间等等。

#### (2).实例说明：

```c++
#include <stdio.h>
#include "curl/curl.h"

struct TransFile {
    const char *filename;
    FILE *stream;
};

size_t write_data( void *buffer, size_t size, size_t nmemb, void *stream);

int main()
{
    CURL *curl;
    CURLcode ret;
    struct TransFile transfile = {
        "example.pdf",
        NULL
    };

	//初始化一个curl指针，初始化会话
    curl = curl_easy_init();
    if (!curl) {
        printf("couldn't init curl\n");
        return 0;
    }

    //设置要下载的文件的URL
    curl_easy_setopt(curl,CURLOPT_URL,"http://www.mysite.com/example.pdf");
    
    /*执行写入文件流操作*/
    curl_easy_setopt( curl, CURLOPT_WRITEFUNCTION, write_data);//调用回调函数
    curl_easy_setopt( curl,CURLOPT_WRITEDATA, &transfile);//传入回调函数需要的结构体的指针

    //执行会话,同步阻塞调用，该函数执行完成，返回后，再执行回调函数
    ret = curl_easy_perform(curl);

    //释放curl对象
    curl_easy_cleanup(curl);

    if(ret != CURLE_OK) {
        fprintf(stderr,"%d",ret);
    }

    //关闭文件流
    if(transfile.stream) {
        fclose(transfile.stream);
    }

    return 0;
}

size_t write_data( void *buffer, size_t size, size_t nmemb, void *stream)
{
    struct TransFile *out = (struct TransFile *)stream;
    printf("2.1\n");
    if(out && (!out->stream)) {
        printf("2.2\n");
        out->stream = fopen(out->filename, "wb");
        printf("2.3\n");
        if(!out -> stream) {
            printf("fopen failed!\n");
            return -1;
        }
    }

    return fwrite(buffer,size,nmemb,out->stream);
}
```



### 3.`libcurl multi`模式下函数基本使用

#### (1).基本函数使用方法：

`multi`的使用是在`easy`的基础之上，将多个`easy handler`加入到一个`stack`中，同时发送请求。与`easy` 不同，它是一种异步，非阻塞的传输方式。

`multi`的使用与`easy`相似：

- [ ] `curl_multi _init`初始化一个`multi handler`对象；
- [ ] 使用`curl_easy_setopt`进行相关设置；
- [ ] 调用`curl_multi _add_handle`把`easy handler`添加到`multi curl`对象中；
- [ ] 添加完毕后执行`curl_multi_perform`方法进行并发的访问；
- [ ] 访问结束后`curl_multi_remove_handle`移除相关`easy curl`对象，先用`curl_easy_cleanup`清除`easy handler`对象，最后`curl_multi_cleanup`清除`multi handler`对象。

`multi`模式缺陷：

- [ ] 每次都要检测所有的`curl`一遍，效率低;
- [ ] 一旦某个`curl`因某种原因死掉了，我该如何判断是哪一个`curl`挂了？

#### (2).实例说明：

参考：<https://blog.csdn.net/myvest/article/details/82899788>

```c
int main(int argc, char **argv)
{
  CURL *easy[NUM_HANDLES];
  CURLM *multi_handle;
  int i;
  int still_running = 0; /* keep number of running handles */

  if(argc > 1)
    /* if given a number, do that many transfers */
    num_transfers = atoi(argv[1]);

  if(!num_transfers || (num_transfers > NUM_HANDLES))
    num_transfers = 3; /* a suitable low default */

  /* init a multi stack */
  multi_handle = curl_multi_init();

  for(i = 0; i<num_transfers; i++) {
    easy[i] = curl_easy_init();
    /* set options */
    setup(easy[i], i);

    /* add the individual transfer */
    curl_multi_add_handle(multi_handle, easy[i]);
  }

  curl_multi_setopt(multi_handle, CURLMOPT_PIPELINING, CURLPIPE_MULTIPLEX);

  /* we start some action by calling perform right away */
  curl_multi_perform(multi_handle, &still_running);

  while(still_running) {
    struct timeval timeout;
    int rc; /* select() return code */
    CURLMcode mc; /* curl_multi_fdset() return code */

    fd_set fdread;
    fd_set fdwrite;
    fd_set fdexcep;
    int maxfd = -1;

    long curl_timeo = -1;

    FD_ZERO(&fdread);
    FD_ZERO(&fdwrite);
    FD_ZERO(&fdexcep);

    /* set a suitable timeout to play around with */
    timeout.tv_sec = 1;
    timeout.tv_usec = 0;

    curl_multi_timeout(multi_handle, &curl_timeo);
    if(curl_timeo >= 0) {
      timeout.tv_sec = curl_timeo / 1000;
      if(timeout.tv_sec > 1)
        timeout.tv_sec = 1;
      else
        timeout.tv_usec = (curl_timeo % 1000) * 1000;
    }

    /* get file descriptors from the transfers */
    mc = curl_multi_fdset(multi_handle, &fdread, &fdwrite, &fdexcep, &maxfd);

    if(mc != CURLM_OK) {
      fprintf(stderr, "curl_multi_fdset() failed, code %d.\n", mc);
      break;
    }

    /* On success the value of maxfd is guaranteed to be >= -1. We call
       select(maxfd + 1, ...); specially in case of (maxfd == -1) there are
       no fds ready yet so we call select(0, ...) --or Sleep() on Windows--
       to sleep 100ms, which is the minimum suggested value in the
       curl_multi_fdset() doc. */

    if(maxfd == -1) {
#ifdef _WIN32
      Sleep(100);
      rc = 0;
#else
      /* Portable sleep for platforms other than Windows. */
      struct timeval wait = { 0, 100 * 1000 }; /* 100ms */
      rc = select(0, NULL, NULL, NULL, &wait);
#endif
    }
    else {
      /* Note that on some platforms 'timeout' may be modified by select().
         If you need access to the original value save a copy beforehand. */
      rc = select(maxfd + 1, &fdread, &fdwrite, &fdexcep, &timeout);
    }

    switch(rc) {
    case -1:
      /* select error */
      break;
    case 0:
    default:
      /* timeout or readable/writable sockets */
      curl_multi_perform(multi_handle, &still_running);
      break;
    }
  }

  curl_multi_cleanup(multi_handle);

  for(i = 0; i<num_transfers; i++)
    curl_easy_cleanup(easy[i]);

  return 0;
}
```

### 4.`libcurl easy`模式陷阱

`libcurl`漏洞官方：<https://curl.haxx.se/mail/lib-2010-11/>

- 全局初始化问题

  `CURLcode curl_global_init(long flags);` 在多线程应用中，需要在主线程中调用这个函数。这个函数设置`libcurl`所需的环境。通常情况，如果不显式的调用它，第一次调用 `curl_easy_init`时，`curl_easy_init` 会调用 `curl_global_init`，在单线程环境下，这不是问题。但是多线程下就不行了，因为`curl_global_init`不是线程安全的。在多个线 程中调用`curl_easy_int`，如果两个线程同时发现`curl_global_init`还没有被调用，同时调用 `curl_global_init`，悲剧就发生了。这种情况发生的概率很小，但可能性是存在的。

- 超时问题

  总结：多线程使用设置超时时，需要设置`CURLOPT_NOSIGNAL`为1.

  背景：使用多线程`libcurl`发送请求，在未设置超时或长超时的情况下程序运行良好。但只要设置了较短超时（小于`180s`），程序就会出现随机的`coredump`。并且栈里面找不到任何有用的信息。

  问题：1.为什么未设置超时，或者长超时时间（比如`601s`）的情况下多线程`libcurl`不会`core`？

  问题：2.进程`coredump`并不是必现，是否在`libcurl`内多线程同时修改了全局变量导致？

  `CURLOPT_TIMEOUT`

  超时机制是使用`SIGALRM`信号量来实现的，并且在`unix-like`操作系统中又提到了另外一个选项`CURLOPT_NOSIGNAL`；

  `CURLOPT_NOSIGNAL`

  该选项说明提到，为了在多线程中允许程序去设置`timeout`选项，但不是使用`signals`，需要设置`CURLOPT_NOSIGNAL`为1 .

  问题：3.`timeout`机制实现机制是什么，为什么设置了选项`CURLOPT_NOSIGNAL`线程就安全了？

  具体分析参照：​ ​ <https://www.cnblogs.com/edgeyang/articles/3722035.html> 

  ​                             https://curl.haxx.se/mail/lib-2010-11/

  ​		  	     https://blog.csdn.net/delphiwcdj/article/details/18284429

- 多线程调用

  避免`libcurl`的多线程调用.

### 5.`IVI`中使用`libcurl`遇到的问题

#### (1). 关于`libcurl`的堵塞方式和非堵塞方式？

- [ ] `libcurl`的`easy`模式是阻塞方式，阻塞的含义是指执行会话的函数`curl_easy_perform`是阻塞形式的，只有该函数返回后，才会调用回调函数取到数据；该阻塞方式会导致下一次会话不能开始，必须等待上一次会话结束。

- [ ] `libcurl`的`multi`模式是非阻塞方式，是指`curl_multi_perform`是非阻塞的，会立即返回，该方式会使用`select`方式监听会话结果，`while()`方式进行监听同样会导致阻塞，但是不会影响下一次会话的开始，也就是说`curl_multi_perform`是可以并行处理多个`curl handle`；

- [ ] 总结：

  - `libcurl`的两种使用模式其实都是阻塞的，只是阻塞的行为不同，`easy`模式阻塞会话执行阶段，本次会话没有结束，影响下一次会话的开始；而`multi`模式在接收数据的地方会阻塞，需要循环监听会话的行为；

  - 其实这两种模式最主要的区别不是否阻塞，是`easy`模式只能执行一个会话，如果要执行多个会话那么每个会话必须在独立的线程下，而`multi`模式可以在同一个线程下执行多个会话，每个会话之间是没有干扰的；所以针对这两种模式都不能很好的解决`IVI`当前的需求，`IVI`需要的非阻塞是希望可以随时切断会话，但是`multi`模式也不能很好的解决该问题。当然`multi`也可以通过另一种方式满足`IVI`的需求，由于`multi`模式支持多会话，当用户需要切断的时候，给用户返回已经切断，实际没有真正的切断，`wireshark`应该可以测试出来，但是不影响下一次会话，这样的处理是否真的符合式样？？？需要确认。

#### (2).单线程，多线程`API`使用的说明？

`libcurl`的两种模式都支持多线程使用，只是同一个`handle`不可以在多个线程中进行会话。这个问题很容易想到，如果在多个线程中使用同一个`handle`进行会话，根本无法管控各个线程的执行情况，很容易导致`crash`。

#### (3).选择的依据是什么？

关于`libcurl`两种模式最初选择`easy`模式的依据，我也不太清楚，但是仔细分析学习两种模式的使用以及差别，可以得知在`IVI`中使用easy模式可能更好，因为我们不需要并行会话，`multi`模式使用复杂，且异常也是不可控的（1. 每次都要检测所有的`curl`一遍，效率低;2.一旦某个`curl`因某种原因死掉了，该如何判断是哪一个`curl`挂了？）。

最初选择`easy`模式，所以后续出现问题，也是查找`easy`模式的漏洞，并进行规避，主要规避点在文中第四章。

#### (4).堵塞模式下，怎么强制中断连接？

- [ ] `IVI`当前使用的是`easy`模式，该模式下的中断是真正的使会话中断。在执行`curl_easy_perform`以外的线程，通过`curl_easy_setopt`修改`timeout`时间的方式使`curl_easy_perform`超时立马返回。不影响用户再次会话。
- [ ] `multi`模式处理`IVI`中断，我思考可做的方式可能有以下两种：
  - 给用户返回会话已经结束，但实际会话并没有真的结束，收到会话结束标志后，不处理；用户选择新的会话会创建新的`handle`；
  - 寻找方法真正中断本次会话，该问题和`IVI`使用`easy`模式最初遇到的问题是一致的；

#### (5).多网卡环境下，网卡怎么指定？

该问题之前讨论过，如果`IVC`有固定的`IP`或者网口，可以通过指定`IP`或网口的方式来确定使用`IVC`进行通信的。当前没有获取`IVC`的`IP`地址或者网口的接口，`IP`和网口是否是固定值暂时也不确定。通过之前的设计时序图指定`IVC`通信的网口为`“eth0”`来指定通信接口，但其实网络接口是会发生变化的，这种选择显然不是最好的，该问题暂时通过`curl_easy_setopt(curl_handle, CURLOPT_INTERFACE, "eth0")`设定接口来区分`IVC`和`WIFI`通信，后续还需要进一步推进。


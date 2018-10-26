---
title: netty入门到精通-初识NIO
abbrlink: 56316
date: 2018-10-01 20:19:26
tags:
---

## NIO和BIO和AIO

| 名词   | 解释                                                         | 案例                                                         |
| ------ | ------------------------------------------------------------ | ------------------------------------------------------------ |
| 同步   | 指的是用户进程触发IO操作并等待或者轮询的去查看IO操作是否就绪 | 自己上街买衣服，自己亲自干这件事，别的事干不了。             |
| 异步   | 异步是指用户进程触发IO操作以后便开始做自己的事情，而当IO操作已经完成的时候会得到IO完成的通知（异步的特点就是通知） | 告诉朋友自己合适衣服的尺寸，大小，颜色，让朋友委托去卖，然后自己可以去干别的事。（使用异步IO时，Java将IO读写委托给OS处理，需要将数据缓冲区地址和大小传给OS） |
| 阻塞   | 所谓阻塞方式的意思是指, 当试图对该文件描述符进行读写时, 如果当时没有东西可读,或者暂时不可写, 程序就进入等待 状态, 直到有东西可读或者可写为止 | 去公交站充值，发现这个时候，充值员不在（可能上厕所去了），然后我们就在这里等待，一直等到充值员回来为止。（当然现实社会，可不是这样，但是在计算机里确实如此。） |
| 非阻塞 | 非阻塞状态下, 如果没有东西可读, 或者不可写, 读写函数马上返回, 而不会等待， | 银行里取款办业务时，领取一张小票，领取完后我们自己可以玩玩手机，或者与别人聊聊天，当轮我们时，银行的喇叭会通知，这时候我们就可以去了。 |

同步和异步是针对应用程序和内核的交互而言的,阻塞和非阻塞是针对于进程在访问数据的时候，根据IO操作的就绪状态来采取的不同方式。说白了是一种读取或者写入操作函数的实现方式，阻塞方式下读取或者写入函数将一直等待，而非阻塞方式下，读取或者写入函数会立即返回一个状态值。

#### 同步和异步:

进程中的IO调用步骤大致可以分为以下四步： 

1. 进程向操作系统请求数据 ;
2. 操作系统把外部数据加载到内核的缓冲区中; 
3. 操作系统把内核的缓冲区拷贝到进程的缓冲区 ;
4. 进程获得数据完成自己的功能 ;

当操作系统在把外部数据放到进程缓冲区的这段时间（即上述的第二，三步），如果应用进程是挂起等待的，那么就是同步IO，反之，就是异步IO，也就是AIO 。

#### BIO,NIO,AIO:

* BIO（同步阻塞IO，Blocking I/O）

同步并阻塞，服务器实现模式为一个连接一个线程，即客户端有连接请求时服务器端就需要启动一个线程进行处理，如果这个连接不做任何事情会造成不必要的线程开销，当然可以通过线程池机制改善。 

* NIO（同步非阻塞IO，New I/O）

同时支持阻塞和非阻塞，也就是异步阻塞，和同步非阻塞。这里以同步非阻塞来进行说明，服务器实现模式为一个请求一个线程，即客户端发送的连接请求都会注册到多路复用器上，多路复用器轮询到连接有I/O请求时才启动一个线程进行处理。用户进程也需要时不时的询问IO操作是否就绪，这就要求用户进程不停的去询问。 

* AIO（异步非阻塞IO，NIO2)

异步非阻塞，服务器实现模式为一个有效请求一个线程，客户端的I/O请求都是由OS先完成了再通知服务器应用去启动线程进行处理，

## 代码实现

BIO:

IOServer.java:

```java
package the.testio;

import java.io.IOException;
import java.io.InputStream;
import java.net.ServerSocket;
import java.net.Socket;

/**
 * @author 闪电侠
 * @version V1.0
 * @package the.testio
 * @date 2018-10-01 16:59
 */
public class IOServer {
    public static void main(String[] args) throws Exception {
        ServerSocket serverSocket = new ServerSocket(8000);

        new Thread(() -> {
            while (true) {
                try {
                    // (1) 阻塞方法获取新的连接
                    Socket socket = serverSocket.accept();

                    // (2) 每一个新的连接都创建一个线程，负责读取数据
                    new Thread(() -> {
                        try {
                            int len;
                            byte[] data = new byte[1024];
                            InputStream inputStream = socket.getInputStream();
                            // (3) 按字节流方式读取数据
                            while ((len = inputStream.read(data)) != -1) {
                                System.out.println(new String(data, 0, len));
                            }
                        } catch (IOException e) {
                        }
                    }).start();

                } catch (IOException e) {
                }
            }
        }).start();
    }
}
```

IOClient.java:

```
package the.testio;

import java.io.IOException;
import java.net.Socket;
import java.util.Date;

/**
 * @author 闪电侠
 * @version V1.0
 * @package the.testio
 * @date 2018-10-01 17:01
 */
public class IOClient {
    public static void main(String[] args) {
        new Thread(() -> {
            try {
                Socket socket = new Socket("127.0.0.1", 8000);
                while (true) {
                    try {
                        socket.getOutputStream().write((new Date() + ": hello world").getBytes());
                        Thread.sleep(2000);
                    } catch (Exception e) {
                    }
                }
            } catch (IOException e) {
            }
        }).start();
    }
}

```



## 参考链接

* https://blog.csdn.net/u013851082/article/details/53942947
* https://juejin.im/entry/598da7d16fb9a03c42431ed3#comment
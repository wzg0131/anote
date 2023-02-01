# NIO模型

[TOC]

## Server端Socket编程

### Socket通信原理

![image-20221226093929382](./images/NIO%E6%A8%A1%E5%9E%8B/image-20221226093929382.png)

（基于端口的进程间通信，另一种通信方式是存储:文件/内存）

### Server端Socket实现

#### 单线程: 串行处理，阻塞IO

```JAVA
public class Server {
    public void start() {
    	ServerSocket serverSocket = new ServerSocket(8080);
		while(true) {
        	try {
            	Socket accept = serverSocket.accept();//阻塞等待一个连接
            	handle(accept);//业务处理
        	} catch(IOException e) {
            	e.printStackTrace();
        	}
    	}
    }
}
```

#### 多线程：one Thread per Request

```java
public class Server {
    public void start() {
    	ServerSocket serverSocket = new ServerSocket(8080);
		while(true) {
        	try {
            	Socket accept = serverSocket.accept();//阻塞等待一个连接
            	new Thread(() -> handle(accept))//每个请求都创建一个线程
                    .start();
        	} catch(IOException e) {
            	e.printStackTrace();
        	}
    	}
    }
}
```

缺点：线程创建的开销大

#### 多线程优化：固定大小线程池

```java
public class Server {
    //线程池的大小可以多于CPU核心数，因为CPU需要等待IO,可以多一些线程
    ExecutorService executorService = Executors.newFixedThreadPool(40);
    
    public void start() {
    	ServerSocket serverSocket = new ServerSocket(8080);
		while(true) {
        	try {
            	Socket accept = serverSocket.accept();//阻塞等待一个连接
            	executorService.execute(() -> handle(accept))//将请求处理交给线程池
        	} catch(IOException e) {
            	e.printStackTrace();
        	}
    	}
    }
}
```

### IO密集型应用

服务器通信过程，存在两种类型的操作：

- CPU计算/业务处理
- IO操作与等待/网络、磁盘、数据库

我们的应用系统可以分成两类：

- 计算密集型应用
- 数据(IO)密集型应用

一般的**web服务器属于IO密集型**。

#### 深入讨论IO

<u>线程数和CPU等待IO的问题：</u>

如果**线程太少，CPU利用率低**，因为IO的等待时间相比CPU计算长太多了。

如果**线程太多，上下文切换需要消耗资源**。且相互竞争资源。

<u>更深层次，用户空间和内核空间的IO需要通过缓冲区来回复制，需要等待数据**就绪**：</u>

![image-20221226104356823](./images/NIO%E6%A8%A1%E5%9E%8B/image-20221226104356823.png)



## IO模型

同步异步是**通信模式**。

阻塞、非阻塞是**线程处理模式**。

五种IO模型：

![image-20221226104841468](./images/NIO%E6%A8%A1%E5%9E%8B/image-20221226104841468.png)

对于“读取”场景，可以分为两阶段：

1. 数据准备：数据进入内核缓冲区
2. 数据复制：数据从内核缓存区复制到用户空间

### 阻塞IO：BIO

![image-20221226110035280](./images/NIO%E6%A8%A1%E5%9E%8B/image-20221226110035280.png)

应用程序**调用一次recvfrom**后是完全阻塞的。

### 非阻塞IO

和阻塞IO比，**数据没有准备好内核会立即返回**，应用程序没有阻塞，可以将CPU时间用于其他事情；

当数据准备好后，**数据复制阶段还是会阻塞**。

![image-20221226110622880](./images/NIO%E6%A8%A1%E5%9E%8B/image-20221226110622880.png)

一般来说，**需要应用程序<u>主动多次轮询</u>调用recvfrom**。

### IO多路复用（事件驱动IO）：NIO

上面的两种模型都是针对单个套接字的；IO多路复用是**==在<u>单独的线程</u>里同时<u>监控多个套接字==</u>**，通过`select`或`poll`轮询所负责的所有socket，**当某个socket有数据到达了，就通知用户进程**：

![image-20221226111219872](./images/NIO%E6%A8%A1%E5%9E%8B/image-20221226111219872.png)

本质上，和非阻塞IO一样是通过轮询的。区别在于利用新的`select`系统调用，**由<u>内核来负责</u>本来请求进程该做的的<u>轮询</u>操作**。

**对于应用进程来说，两个阶段都是阻塞的**：它会先阻塞在`select/poll`等待任意一个可读套接字，再阻塞在数据复制阶段。因为是多个套接字一起查询，整体还是提高了效率。

#### select/poll vs epoll

<u>select/poll的缺点：</u>

1. 每次调用`select`，都需要把`fd集合`从用户态拷贝到内核态，当fd很多时开销会很大。

2. 每次调用`select`，都需要在内核**遍历这个fd集合**(链表，O(N))。
3. `select`支持的文件描述符数量太少，默认是1024。

后来Linux引入了`epoll`代替POSIX `select`和`poll`，**<u>`epoll`的改进有</u>：**

1. 内核与用户空间**共享一块内存（Buffer）**。（减少了fd集合的拷贝）
2. 通过**回调**解决遍历问题，并使用红黑树替代链表。（注册回调，不需要轮询fd集合来找准备就绪的套接字，就绪）
3. **fd没有限制**，可以支撑10万连接。

#### 基于Epoll的Reactor编程模型

Reactor是**编程模型**，还是**基于IO多路复用**这个IO模型；**基于事件和回调**。

![image-20221226140227765](./images/NIO%E6%A8%A1%E5%9E%8B/image-20221226140227765.png)

**由reactor线程负责轮询IO事件**，当有IO就绪事件后通知用户线程，这样**用户线程就不需要阻塞等待**了。

![image-20221226141013185](./images/NIO%E6%A8%A1%E5%9E%8B/image-20221226141013185.png)

Reactor编程模型虽然用户线程不需要等待，也不需要轮询，但终究还是需要一个reactor线程来承担这些工作。

#### NIO和BIO的比较

BIO:

![image-20221226171407536](./images/NIO%E6%A8%A1%E5%9E%8B/image-20221226171407536.png)

NIO:

![image-20221226171427517](./images/NIO%E6%A8%A1%E5%9E%8B/image-20221226171427517.png)

### 信号驱动IO

信号驱动IO与BIO，NIO最大的区别在于：在数据准备阶段不会阻塞用户进程；数据拷贝阶段仍然是阻塞的。

- 当用户进程需要数据的时候，向内核发送一个`信号`(表示想要什么数据)，就可以继续处理别的事情。
- 当内核中数据准备好后，内核再向用户进程发送一个`SIGIO信号`（表示**数据准备就绪**），用户进程收到后立马调用`recvfrom`拷贝数据（阻塞拷贝）。（针对单个socket）

![image-20221226142136136](./images/NIO%E6%A8%A1%E5%9E%8B/image-20221226142136136.png)

#### 服务器开发的发展趋势

数据密集型应用最重要的性能参数：**吞吐和延迟**。

1. 线程池

   随着负载增加，吞吐降低，延迟升高。

   ![image-20221226142509712](./images/NIO%E6%A8%A1%E5%9E%8B/image-20221226142509712.png)

2.  EDA（事件驱动）

   随着负载增加，吞吐提升，延迟也升高。

   ![image-20221226142531286](./images/NIO%E6%A8%A1%E5%9E%8B/image-20221226142531286.png)

3.  SEDA（分阶段事件驱动）

   多级缓冲模型。适应不同场景，将**吞吐提升的同时控制延迟**。

   ![image-20221226142624159](./images/NIO%E6%A8%A1%E5%9E%8B/image-20221226142624159.png)

### 异步IO

异步IO真正实现了IO全流程的非阻塞：用户发出系统调用后立即返回，内核负责**等待数据准备完成，并将数据拷贝到用户进程的缓冲区，然后发送信号告诉用户进程IO操作执行完毕**。（与`SIGIO`相比，<u>一个是告诉用户数据准备完毕，一个是整个IO操作执行完毕</u>）

![image-20221226143309571](./images/NIO%E6%A8%A1%E5%9E%8B/image-20221226143309571.png)

如Mac系统的kqueue，windows系统的iocp，都是异步IO。

Proactor编程模型：

![image-20221226143428408](./images/NIO%E6%A8%A1%E5%9E%8B/image-20221226143428408.png)

#### 一个例子比较Reactor和Proactor

去打印店打印文件：

|              | 排队（数据准备阶段）               | 打印（数据拷贝阶段）                     |
| ------------ | ---------------------------------- | ---------------------------------------- |
| **同步阻塞** | 一直排队，不做其他事情（阻塞）     | 轮到了，自己打印文件（阻塞）             |
| **reactor**  | 拿个号码，回去干别的事情（非阻塞） | 店主通知你过来，自己打印文件（阻塞）     |
| **proactor** | 拿个号码，回去干别的事情（非阻塞） | 店主帮你打印好文件，通知你来拿（非阻塞） |


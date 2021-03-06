标签：网络编程
# TCP连接的建立与终止
---
> TCP是一个面向连接的协议。无论哪一方向另一方发送数据之前，都必须先在双方之间建立一条连接。

## 建立TCP连接
## 终止TCP连接

## 重点
 - TCP状态图![](https://www4.cs.fau.de/Projects/JX/Projects/TCP/tcpstate.gif)
 - TCP发送RST的三种情况
    - 到不存在的端口的连接请求
    - 异常终止一个连接(**SO_LINEGER**选项)
    - 接收到一个根本不存在的连接上的packet(半打开连接)
 - TCP呼入请求队列

## 思考点
 1. shutdown/close的差别是什么？接收方能判断对方使用的是哪种方式吗？对后续的通信有什么影响？
 2. TCP收到RST应用层却未及时处理(可能正阻塞于某个调用)，最后会发生什么？如何应对？(SIGPIPE信号)
 3. **SO_REUSEPORT** 和 **SO_REUSEADDR** 选项的区别？
 4. 对方主机崩溃发送方会做怎么样的处理？崩溃后重启呢？
  
**设计几个实验验证所想。**

### 总结
- **shutdown和SO_LINGER选项**

| 函数 | 选项 | 说明 |
| --- | --- | --- |
| shutdown | SHUT_RD | 在套接字上不能再发出接收请求; 进程仍可往套接字发送数据; 套接字接收缓冲区中所有数据被丢弃; 再接收到的任何数据由TCP丢弃; 对套接字发送缓冲区没有任何影响 |
| shutdown | SHUT_WR | 在套接字上不能再发出发送请求; 进程仍可从套接字接收数据; 套接字发送缓冲区中的内容被发送到对端, 后跟正常的TCP连接终止序列(即发送FIN); 对套接字接收缓冲区无任何影响 |
| close | l_onoff=0 (默认) | 在套接字上不能再发出发送或接收请求; 套接字发送缓冲区中的内容被发送到对端。如果描述符引用计数变为0, 在发送完发送缓冲区的数据后, 跟以正常的TCP连接终止序列(即发送FIN); 套接字接收缓冲区内容被丢弃。|
| close | l_onoff=1 l_linger=0| 在套接字上不能再发送或接收请求。如果描述符引用计数变为0, RST被发送到对端, 连接的状态被置为CLOSED(**没有TIME_WAIT状态**); 套接字发送缓冲区和接收缓冲区中的数据被丢弃。  |
| close | l_onoff=1 l_linger!=0 | 在套接字上不能再发出发送或接收请求; 套接字发送缓冲区中的数据被发送到对端。如果描述符引用计数变为0, 在发送完发送缓冲区中的数据后, 跟以正常的TCP连接终止序列(即发送FIN); 套接字接收缓冲区中数据被丢弃; 如果在连接变为CLOSED状态前延滞时间到, 那么close返回EWOULDBLOCK错误。 |
- **检测各种TCP条件的方法**

| 情形 | 对端进程崩溃 | 对端主机崩溃|
| --- | -- | --- |
| 本端TCP正主动发送数据 -------------------------| 对端TCP发送一个FIN, 这通过使用select/poll判断可读条件立即能检测出来。如果本端TCP发送另外一个分节, 对端TCP就以RST响应。如果在本端TCP收到RST之后应用进程仍试图写套接字, 我们的套接字实现就给该进程发送一个SIGPIPE信号 | 本端TCP将超时, 且套接字的待处理错误将设置为ETIMEDOUT|
| 本端TCP正主动接收数据 | 对端TCP将发送一个FIN, 我们将把它作为一个(可能是过早的)EOF读入 | 我们将停止接收数据 |
| 连接空闲, 保持存活选项已设置 | 对端TCP发送一个FIN, 这通过使用select/poll判断可读条件立即能检测出来 | 在毫无动静2小时后, 发送9个保持存活探测器分节(数值依赖具体实现), 然后套接字的待处理错误被设置为ETIMEDOUT |
|连接空闲, 保持存活选项为设置 | 对端TCP发送一个FIN, 这通过使用select/poll判断可读条件立即能检测出来 | 无 |

## 参考
 - **TCP/IP详解 卷1: 协议**
 - **UNIX网络编程卷1: 套接字联网api(第三版)**
### TCP如何确保传输的可靠性
>TCP提供一种面向连接的、可靠的字节流服务

    面向连接：意味着两个使用TCP的应用（通常一个是客户端和一个服务器）在彼此交换数据之前必须先建立一个TCP连接。在一个TCP连接中，仅有两方进行彼此通信。
    
1. 应用数据被分割成TCP认为最合适发送的数据块。这和UDP完全不同，应用程序产生的数据报长度将保持不变。（<u>将数据截断为合理的长度</u>）
2. 当TCP发出一个段后，启动一个定时器，等待目的端确认收到这个报文段。没有及时收到确认将重新发送这个报文段。（<u>超时重发</u>）
3. 当TCP收到法子TCP连接另一端的数据，将发送一个确认。这个确认不是立即发送，通常将推迟几分之一秒。（<u>对于收到请求，给出确认回应</u>）（<u>推迟是因为对包的完整做校验</u>）
4. TCP将保持它收不和数据的校验和。这是一个端到端的校验和，目的是检测数据再传输过程中的任何变化。如果收到段的校验和有差错，TCP将丢弃这个报文段并且不确认收到此报文段。（<u>校验出包有错，丢弃报文段，不给出响应，TCP发送端将超时重新发送数据</u>）
5. 既然TCP报文段作为IP数据报来传输，而IP数据报到达可能会失序，因此TCP报文段的到达可能会失序。如果必要，TCP将对收到的数据进行重新排列，将收到的数据以正确的顺序交给应用层。（<u>对失序数据进行重新排列，再交给应用层</u>）
6. 既然IP数据包会发送城府，TCP的接收端必须丢弃重复的数据（<u>对于重复数据，能够丢弃重复数据</u>）
7. TCP能提供流量控制。TCP连接的每一方都有固定大小的缓冲空间。TCP的接收端只允许另一端发送接收端缓冲区所能接纳的数据。防止较快主机致使较慢主句的缓冲区溢出（<u>TCP可以进行流量控制，防止较快主机致使较慢主机的缓冲区溢出</u>）**滑动窗口协议**

### TCP为什么要四次挥手，TIME_WAIT为什么至少设置两倍的MSL时间？

>[TCP为什么要四次挥手，TIME_WAIT为什么至少设置两倍的MSL时间？](https://www.cnblogs.com/huenchao/p/6266352.html)

1. 为了保证最后一个ACK报文能够到达被动关闭端。这个ACK报文有可能丢失，因而使处在LAST-ACK状态的被动关闭端收不到对已发送FIN+ACK报文的确认。被动关闭端会超时重传这个FIN+ACK报文段，而主动关闭端就能在2MSL时间内收到这个重传的FIN+ACK报文段。如果主动关闭端在TIME-WAIT状态不等待一段时间，而是在发送完ACK报文后就立即释放链接，就无法收到重传的FIN+ACK报文段，因而也不会再发送一次确认报文。这样，被动关闭端就无法按照正常步骤进入CLOSED状态。
2. 主动关闭端在发送完ACK报文段后，再经过2MSL时间，就可以使本链接持续的时间所擅长的所有报文段都从网络中小时，这样就可以使下一个新的链接中不会出现这种旧的连接请求的报文段。

* * *
#### 什么是2MSL
>MSL: Maximun Segment Lifetime，报文最大生存时间
>TTL: time to live，存活时间
>TTL与MSL是有关系的但不是简单的相等的关系，MSL要大于等于TTL

MSL是任何报文在网络上存在的最长时间，超过这个时间报文将被丢弃。因为TCP报文是ip数据报的数据部分。而ip头中有一个TTL域，这个生存时间是有源主机设置初始值但不是存的具体时间，而是存储了一个ip数据保存可以经过的最大路由数，每经过一个处理他的路由器此值就减1，当此值为0则数据报将被丢弃，同事发送ICMP报文通知源主机。RFC 793中规定MSL为2分钟，实际应用中常用的是30秒，1分钟和2分钟等。

2MSL即两倍的MSL，TCP的TIME_WATE状态也称为2MSL等待状态，当TCP的一段发起主动关闭，在发出最后一个ACK包猴，即第三次握手后发送了四地次握手的ACK包猴就进入TIME_WAIT状态，必须在此状态上停留两倍MSL时间，等待2MSL时间主要目的是怕最后一个ACK包对方没有收到，那么对方在超时后将重新发送第三次握手的FIN包，主动关闭端接收重发FIN包后可以再发一个ACK应答包。再TIME)WAIT状态时两端的端口不能使用，要等到2MSL时间借宿才可以继续使用。当连接处于2SML等待阶段时任何迟到的报文段将被丢弃。不过在实际应用中可以通过设置SO_REUSEADDR选项达到不必等待2MSL时间结束再使用此端口。

### 三次握手

* 第一次握手：建立连接时，客户端发送SYN包（SYN=j）到服务器B，并进入SYN_SEND状态，等待服务器B确认。
* 第二次握手：服务器B收到SYN包，必须确认客户A的SYN（ACK=j+1），同时自己也发送一个SYN包（SYN=k），即SYN+ACK包，此时服务器B进入SYN_RECV状态。
* 第三次握手：客户端A收到服务器B的SYN+ACK包，想服务器B发送确认包ACK（ACK=k+1），此包发送完毕，客户端A和服务器B进入ESTABLESHED状态，完成三次握手。
>确认号: 其数值等于发送方的发送序号+1（即接受方期待接受的下一个序列号）

* * *
TCP的包头结构：
| 类型 | 位数 
| --- | --- 
|源端口|16位
|目标端口|16位
|序列号|32位
|回应序列|32位
|TCP头长度|4位
|reserved|6位
|控制代码|6位
|窗口大小|16位
|偏移量|16位
|校验和|16位
|选项|32位（可选）

这样我们得出了TCP包头的最小长度，为20字节

* 第一次握手：客户端发送一个TCP的SYN表示为1的包指明客户打算连接的服务器的端口，以及初始序号X，保存在包头的序列号（Sequence Number）字段里。
* 第二次握手：服务器发回确认包（ACK）应答。即SYN标志位和ACK标志位均为1同时，将确认序列号（Ackownledgement Number）设置为客户的ISN加1，即X+1。
* 第三次握手：客户端再次发送确认包（ACK）SYN标志位为0，ACK标志位为1，并且把服务器发来的ACK序号字段+1，放在确定字段中发送给对方，并且在数据段放ISN的+1。

### 四次挥手
由于TCP连接时全双工的，因此每个方向都必须单独进行关闭。这个原则是当一方完成它的数据发送任务后就能发送一个FIN来终止这个方向的连接。收到一个FIN只以为这这一方向上没有数据流动，一个TCP连接在收到FIN后仍能发送数。首先进行关闭的一方将执行主动关闭，而另一方执行被动关闭。
TCP的连接的拆除需要发送四个包。客户端或服务端均可主动发起挥手动作，在socket编程中任何一方执行close()操作即可产生挥手操作。

1. 客户端A发送一个FIN，用来关闭客户A到服务器B的数据传输。
2. 服务器B收到这个FIN，它返回一个ACK，确认序号为收到的序号加1。和SYN一样，一个FIN将占用一个序号。
3. 服务器B关闭与客户端A的链接，发送一个FIN给客户端A。
4. 客户端A发回ACK报文确认，并将确认序号设置为收到序号加1。
>总结来说：先关读，后关写

1. 服务器读通道关闭
2. 客户机写通道关闭
3. 客户机读通道关闭
4. 服务器写通道关闭

* * *
#### TCP的TIME_WAIT 和 Close_Wait状态

##### Close_Wait
    发起TCP连接关闭的一方称为client，被动关闭的一方称为server。被动关闭的server收到FIN后，但未发出ACK的TCP状态是CLOSE_WAIT。出现这种情况一般都是server端代码的问题。
##### TIME_WAIT
    根据TCP协议定义的4次挥手断开连接规定，发起socket主动关闭的一方socket将进入TIME_WAIT状态。TIME_WAIT状态将持续2个MSL，在Windows下默认为4分钟。TIME_WAIT状态下的socket不能被回收使用。具体现象是一个处理大量短连接的服务器，如果是由服务器主动关闭客户端的连接，将导致服务器端存在大量的TIME_WAIT状态的socket，甚至比处于Established状态下的socket多得多，严重影响服务器的处理性能，甚至耗尽可用的socket，停止服务。

#### TCP和UDP的区别

1. 基于连接和无连接
2. 对系统资源的要求
3. UDP程序结构简单
4. 流模式（TCP）和数据报模式（UDP）
5. TCP保证数据正确，UDP可能丢包，TCP保证数据顺序，UDP不保证


* * *
### OSI七层模型和TCP/IP四层模型
|OSI|TCP/IP|对应网络协议
|---|---|---
|应用层|应用层|HTTP、TFTP、FTP、NFS、WAIS、SMTP
|表示层|>|Telnet、Rlogin、SNMP、Gopher
|会话层|>|SMTP、DNS
|传输层|传输层|TCP、UDP
|网络层|网络层|IP, ICMP, ARP, RARP, AKP, UUCP
|数据链路层|数据链路层|FDDI, Ethernet, Arpanet, PDN, SLIP, PPP
|物理层|>|IEEE 802.1A, IEEE 802.2到IEEE 802.11,w        		
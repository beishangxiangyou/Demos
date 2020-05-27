## 一、TCP和UDP
TCP是**面向连接、可靠的、基于字节流的传输层协议**。

UDP是**面向无连接的传输层协议**。

和UDP相比，TCP有三大特征：
1. 面向连接。指客户端和服务器之间的连接，在双方通信之前，TCP需要三次握手建立连接，UDP没有。
2. 可靠性。TCP保证了连接的可靠性
   1. 有状态。TCP会精确记录哪些数据发送了，哪些数据被对方接收了，哪些没有被对方接收。
   2. 可控制。当意识了丢包或者网络不佳，TCP会控制自己的发送速度或者重发。
3. 面向字节流。UDP 的数据传输是基于数据报的，这是因为仅仅只是继承了 IP 层的特性，而 TCP 为了维护状态，将一个个 IP 包变成了**字节流**。


## 二、TCP三次握手
TCP的三次握手，就是确认双方的两个能力：**接收能力**和**发送能力**。

![](https://user-gold-cdn.xitu.io/2020/2/23/170723de9b8aa08b?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
- 首先服务器监听某个端口，进入了**LISTEN**状态。
- 然后客户端主动发起连接，发送SYN，客户端编程**SYN-SEND**状态
- 服务端接收到SYN，返回SYN和ACK（对应客户端发来的SYN），服务端变为**SYN-REVD**状态
- 最后客户端发送ACK给服务端，客户端状态变为**ESTABLISHED**状态，服务端收到`ACK`也变成**ESTABLISHED**状态

值得注意：
> 凡是需要对端确认的，一定消耗TCP报文的序列号。

SYN是对端的确认，而ACK不需要，因此SYN消耗一个序列号而ACK不需要。

### 为什么不是两次？
根本原因：无法确认客户端的接收能力

举个🌰：
1. 客户端发送SYN报文握手，但这个包滞留在了网络延迟中迟迟没有到达，此时TCP以为丢了包重传两次握手建立了连接。
2. 当连接关闭后，这个滞留的包如果到达了服务器，服务器接收到数据包再发送响应的数据包，这样连接又重新建立起来了，但客户端已经断开了。

### 为什么不是四次？
可以，但三次握手已经能确认客户端和服务器的接收能力和发送能力，没必要。

### 三次握手中可以携带数据吗？
第三次握手的时候可以携带，前两次不能。

如果前两次能携带数据，那么有人想攻击服务器，在第一次握手中的SYN中放大量数据，服务器会消耗更多的时间和内存控制去处理，增大了服务器被攻击的风险。

第三次握手的时候，客户端已经是**ESTABLISHED**状态了，并且已经确认了服务器的接收、发送能力，已经相对安全了。

## 三、TCP四次挥手
![](https://user-gold-cdn.xitu.io/2020/2/23/170723e5c0e05829?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
- 刚开始双方除了**ESTABLISHED**状态
- 客户端向服务器发送**FIN**报文，进入**CLOSED-WAIT1**（半关闭状态），这是客户端只能接收不能发送报文
- 服务器收到后向客户端确认，客户端变成了**CLOSED-WAIT2**状态
- 随后，服务器向客户端发送**FIN**报文，自己进入**LAST-ACK**状态
- 客户端收到服务器发来的FIN后，自己变成了**TIME-WAIT状态**，然后发送ACK给服务器
- 这个时候，客户端等待2个**MSL**（Maximum Segment Lifetime，报文最大生存时间），在这段时候客户端没有收到服务器的重发请求，那么表示ACK已经成功到达，握手结束，否则客户端重发ACK

### 等待2MSL的意义
不等待，客户端直接关闭，当服务端还有很多数据包要发给客户端且还在路上的时候，若客户端的端口此时正好被新的应用占用，就收到了无用数据包，造成数据包混乱

- 1个MSL确保四次挥手中主动关闭方最后的ACK报文能最终到达服务端
- 1个MSL确保对端没有收到ACK重传的FIN报文可以到达

### 为什么是四次挥手而不是三次？
因为服务端在接收到FIN, 往往不会立即返回FIN, 必须等到服务端所有的报文都发送完毕了，才能发FIN。因此先发一个ACK表示已经收到客户端的FIN，延迟一段时间才发FIN。这就造成了四次挥手。

三次挥手的话就是服务端把`ACK`和`FIN`的发送合并为一次挥手，这个长时间的延迟可能会导致客户端误以为`FIN`没有到达服务端，从而客户端不断的重发`FIN`

## 四、半连接队列和SYN Flood攻击
三次握手前，服务端的状态从`CLOSED`变为`LISTEN`, 同时在内部创建了两个队列：**半连接队列**和**全连接队列**，即**SYN队列**和**ACCEPT队列**。

### 半连接队列
当客户端发送`SYN`到服务端，服务端收到以后回复`ACK`和`SYN`，状态由`LISTEN`变为`SYN_RCVD`，此时这个连接就被推入了SYN队列，也就是半连接队列。

### 全连接队列
当客户端返回ACK, 服务端接收后，三次握手完成。这个时候连接等待被具体的应用取走，在被取走之前，它会被推入另外一个 TCP 维护的队列，也就是全连接队列(Accept Queue)。

### SYN FLOOD攻击
SYN FLOOD攻击原理就是客户端在短时间内伪造大量不存在的IP地址，冰箱服务端疯狂发送`SYN`，对于服务端，有两个危险的后果：

1. 处理大量的`SYN`包并返回对应`ACK`, 势必有大量连接处于`SYN_RCVD`状态，从而占满整个半连接队列，无法处理正常的请求。
2. 由于是不存在的 IP，服务端长时间收不到客户端的`ACK`，会导致服务端不断重发数据，直到耗尽服务端的资源。

### 如何应对SYN FLOOD攻击
1. 增加SYN连接，扩充半连接队列的容量
2. 减少SYN+ACK重试次数，避免大量的超时重发
3. 利用**SYN Cookie**技术，服务端收到`SYN`后，不立即分配连接资源，而是根据`SYN`算出`Cookie`，连通第二次握手传给客户端，客户端回复`ACK`时候带上这个Cookie，服务端验证Cookie合法之后才分配连接资源。

## 五、TCP报文头部字段
![](https://user-gold-cdn.xitu.io/2020/2/23/170723f106ff0306?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

### 源端口、目标端口
唯一标识连接：源IP、源端口号、目标IP、目标端口号

在IP层处理了IP，TCP负责记录端口号。

### 序列号
即`Sequence number`。

作用有二：
1. 在SYN报文中交换彼此的初始序列号
2. 保证数据包按正确的顺序组装

### ISN
即`Initial Sequence Number`（初始序列号）

在三次握手的过程当中，双方会用过SYN报文来交换彼此的 ISN。

ISN每4ms加1，增加了安全性，不易被攻击者猜测到。

### 确认号
即**ACK**(`Acknowledgment number`)

**用来告知对方下一个期待接收的序列号**

小于ACK的所有字节已经全部收到

### 标记位
常见的标记位有SYN,ACK,FIN,RST,PSH。

- FIN:  表示发送方准备断开连接
- RST:  用来强制断开连接
- PSH:  告知对方这些数据包收到后应该马上交给上层的应用，不能缓存

### 窗口大小
占用两个字节，也就是 16 位，但实际上是不够用的。因此 TCP 引入了窗口缩放的选项，作为窗口缩放的比例因子，这个比例因子的范围在 0 ~ 14，比例因子可以将窗口的值扩大为原来的 2 ^ n 次方。

### 校验和
占用两个字节，防止传输过程中数据包有损坏，如果遇到校验和有差错的报文，TCP 直接丢弃之，等待重传。

### 可选项
可选项格式如下：
![](https://user-gold-cdn.xitu.io/2020/2/23/170723f4facdfb22?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
常见的可选项：
- TimeStamp: TCP 时间戳，后面详细介绍。
- MSS: 指的是 TCP 允许的从对方接收的最大报文段。
- SACK: 选择确认选项。
- Window Scale： 窗口缩放选项。

## 六、TCP快速打开的原理（TFO）
一个TCP优化方法，也就是TCP快速打开（TCP Fast Open，即TFO）

用来解决SYN Flood攻击的SYN Cookie，这个Cookie不是浏览器的Cookie，但同样也可以用来实现TFO

TFO流程：
### 首轮三次握手
客户端发送SYN给服务端，服务端收到后，通过计算得到一个`SYN Cookie`，将这个`Cookie`放到TCP报文的`Fast Open`选项中，然后才给客户端返回。

客户端拿到这个`Cookie`缓存下来，后面正常完成三次握手。

### 后面的三次握手
在后面的三次握手中，客户端会将之前缓存的 `Cookie`、`SYN` 和HTTP请求(是的，你没看错)发送给服务端，服务端验证了 `Cookie` 的合法性，如果不合法直接丢弃；如果是合法的，那么就正常返回SYN + ACK。

这是最显著的改变，三次握手还没建立，仅仅验证了 Cookie 的合法性，就可以返回 HTTP 响应了。
![](https://user-gold-cdn.xitu.io/2020/2/23/170723f9bbfc467d?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)

注意: 客户端最后握手的 ACK 不一定要等到服务端的 HTTP 响应到达才发送，两个过程没有任何关系。

### TFO优势
TFO 的优势并不在与首轮三次握手，而在于后面的握手，在拿到客户端的 Cookie 并验证通过以后，可以直接返回 HTTP 响应，充分利用了1 个`RTT`(Round-Trip Time，往返时延)的时间**提前进行数据传输**，积累起来还是一个比较大的优势。

## 七、TCP报文中时间戳的作用
`timestamp`是TCP报文首部的一个可选项，一共占10个字节：
```
kind(1 字节) + length(1 字节) + info(8 个字节)
```
 info 有两部分构成: `timestamp`和`timestamp echo`，各占 4 个字节。

TCP报文中时间戳的作用有二：
1. 计算往返时延RTT
2. 防止序列号的回绕问题

### 计算往返时延RTT
比如现在a向b发送一个报文s1，b向a回复一个含ACK的报文s2，那么：
1. step 1: a 向 b 发送的时候，timestamp 中存放的内容就是 a 主机发送时的内核时刻 ta1。
2. step 2: b 向 a 回复 s2 报文的时候，timestamp 中存放的是 b 主机的时刻 tb, timestamp echo字段为从 s1 报文中解析出来的 ta1。
3. step 3: a 收到 b 的 s2 报文之后，此时 a 主机的内核时刻是 ta2, 而在 s2 报文中的 timestamp echo 选项中可以得到 ta1, 也就是 s2 对应的报文最初的发送时刻。然后直接采用 ta2 - ta1 就得到了 RTT 的值。

### 防止序列号回绕
序列号达到最大值后就循环到0，会造成相同的序列号，加上时间戳timestamp就可以避免这个问题。

## 八、TCP超时重传
TCP 具有超时重传机制，即间隔一段时间没有等到数据包的回复时，重传这个数据包。

这个重传间隔也叫做**超时重传时间**(Retransmission TimeOut, 简称RTO)，它与RTT密切相关计算得出

## 九、TCP的流量控制
对于发送端和接收端而言，TCP 需要把发送的数据放到**发送缓存区**, 将接收的数据放到**接收缓存区**。

而流量控制索要做的事情，就是在通过接收缓存区的大小，控制发送端的发送。如果对方的接收缓存区满了，就不能再继续发送了。

流量控制与之密切相关的就是**滑动窗口**，分为**发送窗口**、**接收窗口**
### 发送窗口
发送窗口结构：
![](https://user-gold-cdn.xitu.io/2020/2/23/17072401c4d59dcb?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
![](https://user-gold-cdn.xitu.io/2020/2/23/17072403ff8f9bec?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
发送窗口就是图中被框住的范围。SND 即`send`, WND 即`window`, UNA 即`unacknowledged`, 表示未被确认，NXT 即`next`, 表示下一个发送的位置。

### 接收窗口
![](https://user-gold-cdn.xitu.io/2020/2/23/17072406c803d2c7?imageView2/0/w/1280/h/960/format/webp/ignore-error/1)
REV 即 `receive`，NXT 表示下一个接收的位置，WND 表示接收窗口大小。

### 流量控制过程
首先双方三次握手，初始化各自的窗口大小，均为 200 个字节。
假如当前发送端给接收端发送 100 个字节，那么此时对于发送端而言，SND.NXT 当然要右移 100 个字节，也就是说当前的可用窗口减少了 100 个字节，这很好理解。
现在这 100 个到达了接收端，被放到接收端的缓冲队列中。不过此时由于大量负载的原因，接收端处理不了这么多字节，只能处理 40 个字节，剩下的 60 个字节被留在了缓冲队列中。
注意了，此时接收端的情况是处理能力不够用啦，你发送端给我少发点，所以此时接收端的接收窗口应该缩小，具体来说，缩小 60 个字节，由 200 个字节变成了 140 字节，因为缓冲队列还有 60 个字节没被应用拿走。
因此，接收端会在 ACK 的报文首部带上缩小后的滑动窗口 140 字节，发送端对应地调整发送窗口的大小为 140 个字节。
此时对于发送端而言，已经发送且确认的部分增加 40 字节，也就是 SND.UNA 右移 40 个字节，同时发送窗口缩小为 140 个字节。
这也就是流量控制的过程。尽管回合再多，整个控制的过程和原理是一样的。

## 十、TCP的拥塞控制
当发送端和接收端之间考虑到网络因素可以造成的丢包，发送端需要注意一些了，这就是**拥塞控制**需要处理的问题。

对于拥塞控制来说，TCP 每条连接都需要维护两个核心状态:
- 拥塞窗口（Congestion Window，cwnd）
- 慢启动阈值（Slow Start Threshold，ssthresh）

涉及到的算法有这几个:
- 慢启动
- 拥塞避免
- 快速重传和快速恢复

### 拥塞窗口
- 接收窗口(rwnd)是接收端给的限制
- 拥塞窗口(cwnd)是发送端的限制

接口窗口和拥塞窗口共同限制发送窗口的大小
```
发送窗口大小 = min(rwnd, cwnd)
```

### 慢启动
刚开始进入传输数据的时候，你是不知道现在的网路到底是稳定还是拥堵的，拥塞控制首先就是要采用一种保守的算法来慢慢地适应整个网路，这种算法叫**慢启动**。运作过程如下:
- 首先，三次握手，双方宣告自己的接收窗口大小
- 双方初始化自己的**拥塞窗口**(cwnd)大小
- 在开始传输的一段时间，发送端每收到一个 ACK，拥塞窗口大小加 1，也就是说，每经过一个 RTT，cwnd 翻倍。如果说初始窗口为 10，那么第一轮 10 个报文传完且发送端收到 ACK 后，cwnd 变为 20，第二轮变为 40，第三轮变为 80，依次类推。

当到达一个阀值，即**慢启动阈值**，当 cwnd 到达这个阈值之后增加速度减慢，这就是**拥塞避免**
### 拥塞避免
原来每收到一个 ACK，cwnd 加1，现在到达阈值了，cwnd 只能加这么一点: 1 / cwnd。那你仔细算算，一轮 RTT 下来，收到 cwnd 个 ACK, 那最后拥塞窗口的大小 cwnd 总共才增加 1。

当然，慢启动和拥塞避免是一起作用的，是一体的。

### 快速重传
在 TCP 传输的过程中，如果发生了丢包，即接收端发现数据段不是按序到达的时候，接收端的处理是重复发送之前的 ACK。

当接收端收到3个重复ACK时，意识到丢包了，马上重传，不等待RTO的时间到了才重传。

在收到发送端的报文后，接收端回复一个 ACK 报文，那么在这个报文首部的可选项中，就可以加上**SACK**这个属性，通过`left edge`和`right edge`告知发送端已经收到了哪些区间的数据报。此时发送端只会重传丢的包，这就是**选择性重传**

### 快速恢复
当然，发送端收到三次重复 ACK 之后，发现丢包，觉得现在的网络已经有些拥塞了，自己会进入**快速恢复**阶段。

发送端做出以下改变：
- 拥塞阈值降低为 cwnd 的一半
- cwnd 的大小变为拥塞阈值
- cwnd 线性增加

## 十一、Nagle算法和延迟确认
### Nagle算法
小包的频繁传输不光是传输的时延消耗，发送和确认本身也是需要耗时的，频繁的发送接收带来了巨大的时延。

Nagle算法就是避免小包的频繁发送：
- 当第一次发送数据时不用等待，就算是 `1byte` 的小包也立即发送
- 后面发送满足下面条件之一就可以发了:
  - 数据包大小达到最大段大小(Max Segment Size, 即 MSS)
  - 之前所有包的 ACK 都已接收到

### 延迟确认
试想这样一个场景，当我收到了发送端的一个包，然后在极短的时间内又接收到了第二个包，那我是一个个地回复，还是稍微等一下，把两个包的 ACK 合并后一起回复呢？

**延迟确认**(delayed ack)所做的事情，就是后者，稍稍延迟，然后合并 ACK，最后才回复给发送端。TCP 要求这个延迟的时延必须小于500ms，一般操作系统实现都不会超过200ms。

不过需要主要的是，有一些场景是不能延迟确认的，收到了就要马上回复:

- 接收到了大于一个 frame 的报文，且需要调整窗口大小
- TCP 处于 quickack 模式（通过tcp_in_quickack_mode设置）
- 发现了乱序包

## 十二、TCP的keep-alive
除了http 的keep-alive, 不过 TCP 层面也是有keep-alive机制，而且跟应用层不太一样。

试想一个场景，当有一方因为网络故障或者宕机导致连接失效，由于 TCP 并不是一个轮询的协议，在下一个数据包到达之前，对端对连接失效的情况是一无所知的。

这个时候就出现了 `keep-alive`, 它的作用就是探测对端的连接有没有失效。

在 Linux 下，可以这样查看相关的配置:
```
sudo sysctl -a | grep keepalive

// 每隔 7200 s 检测一次
net.ipv4.tcp_keepalive_time = 7200
// 一次最多重传 9 个包
net.ipv4.tcp_keepalive_probes = 9
// 每个包的间隔重传间隔 75 s
net.ipv4.tcp_keepalive_intvl = 75
```
不过，现状是大部分的应用并没有默认开启 TCP 的keep-alive选项，为什么？

站在应用的角度:

- 7200s 也就是两个小时检测一次，时间太长
- 时间再短一些，也难以体现其设计的初衷, 即检测长时间的死连接








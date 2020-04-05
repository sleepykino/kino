写在最前：
<font color=#a98175 size=2 face="微软雅黑">&nbsp;&nbsp;&nbsp;&nbsp;在学习渗透过程中，发觉自己对网络协议还是理解不深，查了很多资料，为防止忘记后再次查找的不便，以及全面发现知识漏洞，就进行了汇总记录（其中穿插了我的吐槽）</font>
我的学习过程是有个初步印象后不断补充，文章结构也是如此，后面会对前面补充。
**文中有大量的仅做一点改动的抄写内容，若有侵权，请联系我删改。文章所有引用与资料在末尾**
<font color=#0099ff size=5 face="微软雅黑">**全篇几乎无图预警**</font>

@[toc](目录)
<font color=#a98175 size=2 face="微软雅黑">您也可以先看后记_(:з」∠)_</font>
# 一、TCP首部
![TCP首部](https://img-blog.csdnimg.cn/20191019164015738.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4NTQ3NzQ0,size_16,color_FFFFFF,t_70)
先放一个tcp首部图放在这里，无论以前记没记住都没关系，需要的话再回来看一眼就可以了。
**源端口和目的端口**：指明传输端口号。
**偏移（或者叫首部长度）**：表明数据的开始位置（因为有可选选项是变长）
**保留**：先留着没准用得上
**填充**：可选选项是变长的，为了保证首部长度是4的倍数，有个填充区用0填充。
**包校验和**：数据部分hash值，用来保证数据部分传输无误。
*剩下的下面慢慢说*
对于tcp有两种解释规范
**BSD（linux默认）
RFC793**~~主要是793其实还有别的RFC~~ [http://www.rfcreader.com/](http://www.rfcreader.com/)
*两者稍有不同，后面会说一下*
# 二、基本传输
## 1.三次握手
说到tcp，三次握手肯定是要说的，这是可靠传输的基础。三次握手确定了**数据序列号**、**确认序列号**和**窗口大小**。
开始时客户端处于 Closed 的状态，服务端处于 Listen 状态。
**第一次握手**：客户端发送SYN报文，报文中携带 初始序列号（ISN）占用数据序列号位（seq）
报文中SYN=1，初始序号seq=x 

**第二次握手**：服务器回发SYN-ACK报文，收到客户端syn包后，会以自己的 SYN 报文作为应答，并且也是指定了自己的初始化序列号 ISN。同时会把客户端的 ISN + 1 作为ACK 的值
报文中SYN=1，ACK=1，确认号ack=x+1，初始序号seq=y，窗口字段=a（能接收的大小=a）

**第三次握手**：客户端收到 SYN 报文之后，会发送一个 ACK 报文，当然，也是一样把服务器的 ISN + 1 作为 ACK 的值，表示已经收到了服务端的 SYN 报文
报文中ACK=1，确认号ack=y+1，序号seq=x+1

然后就可以开始传输数据（第三次握手就可以携带数据了，服务器已经做好了准备，一般不带，防止包丢失浪费，等3次握手确认连接之后再发也不耽误）

<font color=#16859a face="微软雅黑">其实握手中也会同步别的信息，甚至可能不止三次握手。可选项中会解答</font>

**上面三次握手中有一些点需要解释**：
## 2.序列号
**序列号**是用来记录会话顺序，与判断这个包属于哪个会话。

**初始序列号ISN**:0-4,294,967,295（2^32）中的一个随机数字。
&nbsp;&nbsp;&nbsp;&nbsp;**RFC793**中可以看作是一个计时器，每4ms加1 ，这样可以避免会话上次已断开的连接，还存在网络中延迟到达的TCP报文段的序号与当前连接中等待报文段的序号相同。
但只根据时间很容易被猜测为了防止被别人猜测出序列号从而劫持会话。
&nbsp;&nbsp;&nbsp;&nbsp;**BSD**中为时钟还要再加一个偏移量。偏移量是在4元组的基础上，用加密散列函数得到的，并且散列函数每5分钟就会改变一次。32位中，高8位是一个保密序列号，后面24位是用散列函数得到的。前8位中的5位由一个定时器的数值取模32得到，这个定时器64秒加1，接着三位是对服务器最大段的编码值

**确认序列号**：确认序列号代表希望收到的序列号。为了简化模型，通常说+1
例：当发生Seq=1的包，接收端会传回ACK=2， 也就是将接受到的Seq+1 ， 
代表的是期望接受到的下一个包是 Seq=2的包。
这样可以减少开销，当收到连续几个包，seq=1，seq=2，seq=3，只用回ack=4就行了，一次确认表明收到了3个包

## 3.窗口字段
窗口字段是每个包都会有的（包括syn包），用来表明自己缓冲区的大小（还能接多少数据）是动态调整的。*关于滑动窗口与流量控制，这篇应该不会写了，建议看看别的dalao的*
当接受端发送包中窗口字段为0，发送端先暂停发送，等接收端有空地了，会发送一个窗口不为0的包，发送端收到后继续传输。
<font color=#16859a face="微软雅黑">只有16位，65535=64K 是不是感觉有些问题</font>

## 3.控制位（flags）
控制位就是中间6个
先来个简单解释
| flag | 作用 |
|--|--|
|URG| 为 1 表示紧急指针有效，为 0 则忽略紧急指针值 |
|ACK| 为 1 表示确认号有效，为 0 表示报文中不包含确认信息 |
|PSH| 为 1 表示PUSH 数据，接收方应该尽快将这个报文段交给应用层而不用等待缓冲区装满 |
|RST| 为1说明异常断开 |
|SYN|同步序号，为 1 表示连接请求，用于建立连接|
|FIN|用于释放连接，为 1 表示发送方已经没有数据发送了，即关闭本方数据流|
### SYN与ACK
三次握手中第一次客户端用SYN申请建立连接，第二次服务器用syn-ack回复，表明自己收到了包，请求与客户端建立连接，第三次客户端发送ack确认连接。
SYN用来请求来同步（连接）
ACK用来确认消息
>这之中会有一个等待过程，会产生半连接队列，不断地发送syn包（第一次握手），不回复ack（第三次握手）就会占用服务器资源，这就是dos中的syn中的syn攻击。这种攻击不会留下日志，隐蔽性好，但现在防护方法很多，效率不高
>(代码以后写scapy的时候可能会放一个_(:з」∠)_)
### PSH
名字就显示了，推送标志，
&nbsp;&nbsp;&nbsp;&nbsp;**RFC793**发送端设置了psh表示会立刻将缓冲区中数据打包发送，而不用等缓存区满
接收端收到psh标应会马上将这个报文段交给应用层而不用等待缓冲区装满
&nbsp;&nbsp;&nbsp;&nbsp;**BSD**发送端会在缓存区清空时自动加psh标志，告诉接受方发完了，可以交给应用层了
接收端数据默认直接提交给应用程序,所以接收端会忽略掉接收到的PSH标识
### URG与紧急指针urgpoint
紧急指针已经不是控制位中的了，但紧急指针只在URG工作时才有用

**URG紧急标志** 当有些数据需要“插队“时，设为1，接收端收到紧急标志后会先处理紧急数据.

*所以一些软件断开连接时会使用紧急标志，当接收端收到并处理了紧急标志中的断开指令，别的数据就不用再接收和处理了*。

但这个插队也只是个逻辑上的插队，紧急数据又被称为带外数据（out-of-band data），只是因为紧急数据需要使用一个oobrcve函数来接收，从而让内核优先处理这些数据，实际上并没有什么外部传输的通路，紧急数据和普通数据混在缓冲区里，用一条路。不过紧急数据都被标成了1，但还是与普通数据一起发送。

**紧急指针urgpoint**配合URG使用，虽然URG表示了紧急状态，但接收端并不知道这些混在一起的数据中那些是紧急的，需要紧急指针来只想他们
&nbsp;&nbsp;&nbsp;&nbsp;**RFC793**中紧急指针指向的是紧急字节之前（头）
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;紧急数据字节号(urgSeq)=TCP序列号(seq)+紧急指针(urgpoint)−1
&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;&nbsp;减一是因为除去第一个syn包
&nbsp;&nbsp;&nbsp;&nbsp;**BSD**中指向的是紧急字节之后（尾）

**紧急指针的一些问题**
&nbsp;&nbsp;&nbsp;&nbsp;紧急指针会被后一个被覆盖，当紧急指针为x的包还未发送时又发送一个紧急指针为y的包，紧急数据就被指向了y

### RST
RST 复位标志
主要有3个用处
1.	拒绝连接请求
当收到一些非法访问请求时，回复一个RST报文来拒绝连接
比如你给一个没有开放的端口，或者没有listen的服务器发送一个SYN包，服务器收到后会回复一个RST+ACK包，确认收到了SYN包，但是没开服务，把连接状态复位了
2.	复位（终止）异常连接
有时候会话中一方崩溃了或者出bug了，需要关闭连接，就一个RST
3.	终止一个长时间未响应的连接
有时候连接建立了，但长时间没有传送数据，超时了，就会发送一个RST包来断开
因为RST包时不需要ACK的，发送方在发送RST后会立刻关闭连接，不需要确认

&nbsp;&nbsp;&nbsp;&nbsp;RST属于异常释放，发送方会立刻清空发送缓冲区，不在发送，直接发送RST包
接收方也会判断这是一次异常释放，可能会进行后续操作（尝试重连之类）

><font color=#336699 face="微软雅黑">RST能立刻单方面关闭，那么如果能用来搞事情，是很厉害的。RST攻击，就是利用伪造的RST包来断开一个正常的连接。
攻击过程：
假设A和B正在正常通讯，C是攻击者，C可以伪造成A的地址向B发包，发送的是RST包，那么B就会抛弃A的数据并断开连接，如果发的是SYN包，那B就会觉得A坏了（聊着聊着又发了一个你好，肯定有问题）B就会给A发一个RST。
C想要伪造成A，需要把A的端口，序列号都填好，（A的地址、B的IP和端口号默认已知）
要不然B会拒收。A的端口号一般不是随机产生的，如果能知道其他信息（比如使用的通信软件）可能就能得到端口号，A的序列号是很难猜测的（原因上面讲了），但我们要做的不是劫持，而是让他们断开，可以用暴力法，假设滑动窗口大小为65535，序列号有2^32，只要发65537个包就可以落在一个窗口中，就能完成RST攻击</font>

### FIN
正常关闭标志
在四次挥手中会用到，用来表明我已经没有要发送的数据了
下面就说一下四次挥手
## 4.四次挥手
四次挥手是断开连接时所做的传输，比三次握手多一次，因为第二次握手时既回答（ACK）又请求同步（SYN），但关闭时不同，服务器收到FIN之后可能还有数据没发送完，不能直接发FIN

四次挥手过程
**第一次挥手**：客户端发送FIN报文（我已经没有了）  
报文中FIN = 1
**第二次挥手**：服务器回复ACK报文（了解，但我还有）
报文中ACK = 1
**第三次挥手**：服务器发送FIN报文（我也没有了）
报文中FIN = 1 ACK = 1
**第四次挥手**：客户端回复ACK（ok，再见）
报文中ACK = 1 (这里确认序列号也会把第三次中的序列号+1，只起到一个确认作用)

第四次挥手中客户端回复完ACK会进入TIME_WAIT状态，等待2个最大报文传输时间，防止服务器没收到ACK包无法关闭（服务器没收到ACK，重发第三次挥手，如果客户端已经关闭了，那服务器就无法关闭了）
最大报文传输时间就是报文传输需要的时间，等待2个（一个来回）没有收到服务器反馈，说明已经收到ACK正常关闭，客户端也可以进入close状态了
>关于状态部分也不详细说明了，上面说明都还算是正常情况。但现实经常会出意外，在任何一个阶段都有可能出现崩溃死机的情况，不同的状态有不同的反应，一般会有一个计时器，超时之后可能会发RST，也可能做其他操作
# 三、可选项
可选选项最大40字节，因为首部大小只有4位，1111=15，偏移单位是4，最大就60，前面已经用了20

<font color=#336699 size=4 face="微软雅黑">**TCP协议之所以能经久不衰，就是因为留出了这部分可以更新的地方**</font>

结构是这样的，先是一字节的选项号，然后是选项长度，后面是选项中的一些信息
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191019174556733.png)
可选择的类型(部分)
| 选项号| 内容|Reference|
|--|--|--|
| 0| End of Option List| RFC793|
|1|NOP|RFC793|
|2|MSS|RFC793|
|3|WSOPT-windows Scaling|RFC1323|
|5|SACK|RFC2018|
|8|TSOPT|RFC1323|
|30|MPTCP|RFC6824|
*0就是没有用选项，就不说了*
## ①.NOP
&nbsp;&nbsp;&nbsp;&nbsp;NOP就是没有意义，用来填充的，保证TCP头的大小是4的倍数，如果不是就用NOP填充
&nbsp;&nbsp;&nbsp;&nbsp;NOP也可以用来分隔不同选项，一个包里可能携带2个选项，中间会用NOP隔开
&nbsp;&nbsp;&nbsp;&nbsp;所以可能出现一个包里好几个NOP的情况
## ②.MSS
maximum segment size 占4byte
&nbsp;&nbsp;&nbsp;&nbsp;通知最大可接收量。发送SYN的TCP一端使用本选项通告对端它的最大分节大小（一个包的大小），也就是它在本连接的每个TCP分节中愿意接受的最大数据量。发送端TCP使用接收端的MSS值作为所发送字节的最大大小。
&nbsp;&nbsp;&nbsp;&nbsp;默认为536byte。MSS太小了，效率会很低，因为TCP+IP头至少20+20=40的大小，越小浪费越严重，但太大可能造成IP传输中分片，也增大开销，所以MSS应该是**保证别分片的最大值**
&nbsp;&nbsp;&nbsp;&nbsp;<font color=#16859a face="微软雅黑">MSS也会放在SYN包中传输</font>
## ③.窗口扩大
WSOPT-windows Scaling 占3byte
&nbsp;&nbsp;&nbsp;&nbsp;上面说活动窗口最大值是64k,满了就要发送方就要等ACK，有很大时延。现在缓冲区最大要还是64k，看个视频不得急坏了。
&nbsp;&nbsp;&nbsp;&nbsp;为了应对这种情况,就有了这个窗口扩大选项，可以扩大缓冲区的规模，其中的一个字节表示移位值S。新的窗口值等于TCP首部的窗口位数从16增大到（16+S）。这相当于把窗口值向左移动S位后获得实际的窗口大小。移位值准许使用的最大值是14，相当于窗口最大值增大到65535*2^14也就是1GB，可以大大减少等待时间，降低时延，提高效率。
&nbsp;&nbsp;&nbsp;&nbsp;当不需要扩大时，发送S=0，就可以还原。
&nbsp;&nbsp;&nbsp;&nbsp;<font color=#16859a face="微软雅黑">windows Scaling也会放在SYN包中传输</font>

## ④.SACK
Selective Acknowledgements 
&nbsp;&nbsp;&nbsp;&nbsp;选择确认项，主要用在少了一些包比如传了1，2，3，4，5，但只收到1，3，4，5，少了2，若用原来的确认字符号2，3，4，5全要重传，为了避免这种情况，只重传2，可以使用SACK选项，告诉对方我只缺少了2，重发一下。
&nbsp;&nbsp;&nbsp;&nbsp;那么怎么告诉缺的是2呢，发送的是1和3，用边界来表示缺少的范围。
&nbsp;&nbsp;&nbsp;&nbsp;一个边界占4字节，选项最大40，先用掉2个表明SACK与长度，两个边界是8字节，4x8+2=34，所以最多告知4段去缺失。*(要是报告4个前面的填充就用到了)*

## ⑤.时间戳
TSOPT    timestamps   占10byte 分为时间戳字段（4byte）和时间戳回答字段（4byte）
主要有2种用处：
1. 用来计算往返时间RTT
	发送方把当前的时间x写入时间戳字段，接收方收到后把x放到时间戳回答字段，并把当前的时间y放到时间戳中回复，发送方收到后就能算出RTT
2. 防止序列号重复（回绕）
	序列号是有限的（*RST攻击中说了*），在一些高速网络中可能会耗尽而导致重复，会导致会话的混乱（判断不了这个是新发的还是以前发的刚到），加入一个时间戳就能分辨出是不是新发的。

## ⑥.MPTCP
Multipath TCP 多路TCP
上面说了那么多，但都是单路的TCP。
&nbsp;&nbsp;&nbsp;&nbsp;现在多路已经很常见了，比如手机同时开着4G和WIFI，默认用WIFI浏览者网页，结果WIFI突然断了，你会立刻4G来进行来连接，而不至于中断浏览。
&nbsp;&nbsp;&nbsp;&nbsp;原来的TCP很定是不行的，TCP连接的4元组（源IP，源端口，目的IP，目的端口），其中源IP已经变了，会话就会终止。为了能够实现上面的那种功能，就需要用多路径。
**在一条TCP来连接中包含多个路径**
在协议栈中位于一般TCP之上，[http://www.rfcreader.com/#rfc6824](http://www.rfcreader.com/#rfc6824)中的图
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191019201135356.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4NTQ3NzQ0,size_16,color_FFFFFF,t_70)
这样设计能够保证兼容性，在不改变其他层的情况下完成了多路操作。
### 1.连接过程
在连接时需要先创建一条链接，然后将其他通路加入
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191019202623934.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4NTQ3NzQ0,size_16,color_FFFFFF,t_70)

具体操作：

**一、先开一条通路**
1. 客户端通过IP1会发送一个SYN数据包给服务器，这个数据包和TCP建立连接时发送的一样，只不过增加了TCP选项MP_CAPABLE字段，表明手机端支持MPTCP协议，以及一个key（用于将来继续添加子流时进行验证）。
2. 服务器端回应SYN+ACK数据包同样包含TCP选项MP_CAPABLE字段，以及一个key。
3. 客户端IP1再次回应ACK，此时建立了连接。

![在这里插入图片描述](https://img-blog.csdnimg.cn/20191019202851243.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4NTQ3NzQ0,size_16,color_FFFFFF,t_70)
&nbsp;&nbsp;&nbsp;&nbsp;<font color=#0099ff face="微软雅黑">这里还是3次握手</font>

**二、第二条路加入**
1. 客户端通过IP2向服务器发送SYN包，此时的SYN数据包中携带的TCP选项是MP_JOIN，并提供token（上面key的hash值）与一个rand（随机数），说明其要加入的MPTCP会话，并确认它是安全的。
2. 服务器端给IP2回复SYN+ACK包，包中包含MP_JOIN和HMAC（rand的hash值）
3. 客户端IP2发送ACK，包中包含MP_JOIN和HMAC
4. 服务器回复ACK
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191019203512871.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4NTQ3NzQ0,size_16,color_FFFFFF,t_70)

&nbsp;&nbsp;&nbsp;&nbsp;<font color=#0099ff face="微软雅黑">4次握手保证安全</font>

### 2.负载均衡
&nbsp;&nbsp;&nbsp;&nbsp;多路的好处就是可以负载均衡，合理运用资源，提高使用体验。（WIFI卡了用4G帮一下）
&nbsp;&nbsp;&nbsp;&nbsp;但负载均衡会有一个问题，接收端序列号混乱，包序列号不对可能会被抛弃，那就白均衡了。为了解决这个问题，多加了一个DSN。这时有两个序列号，一个是在TCP包头中的序列号，为子流的序列号，另一个是DSN（data sequence number）为所有传输数据的序列号。收包时，先使用子流序列号，将各个子流接收到的数据包进行重组，然后使用DSN对各个子流报文重组。

# 四、后记
1. 本篇只根据TCP首部浅显的说了一些，关与状态和流量控制几乎没有(~~可能以后会写？~~) ，因为我不喜欢篇幅太长（大脑缓冲区不够用，其实就是懒）。
2. 几乎没有图，因为图片能帮助理解也印象深刻，但很难把所有信息融合进去，需要设计一番（其实就是懒，懒得画甚至连截图都懒得动）
3. 文中肯定有错误，若您发现，希望能回复指出，您觉得有必要添加内容的也可以联系我，有任何问题或不懂得地方也欢迎提出。
4. 技术换代速度很快，早晚过时，但设计点冗余，保持更新，可以晚点=￣ω￣=
5. 不懂得很多，光说也很难懂，不如抓个包看看，那里面全是工具设计者对协议的理解

这是一个TCP包（序列号0是因为相对序列号，实际是有的，方便你阅读用了个相对）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191019210502819.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4NTQ3NzQ0,size_16,color_FFFFFF,t_70)
flags（这是一个SYN包）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191019210609767.png?x-oss-process=image/watermark,type_ZmFuZ3poZW5naGVpdGk,shadow_10,text_aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3FxXzM4NTQ3NzQ0,size_16,color_FFFFFF,t_70)
可选选项（想把MSS设为1460*以太网不分片最大值*，窗口扩大 移动7位，附带时间戳，SACK permitted表示开启SACK功能）
![在这里插入图片描述](https://img-blog.csdnimg.cn/20191019210651628.png)
# 五、参考资料
>1. 《TCP-IP详解卷一：协议》
>2. 《计算机网络》第7版
>3. http://www.rfcreader.com
>4. 数据中心内的负载均衡-MPTCP http://www.rfcreader.com
>5. 浅析TCP之头部可选项 https://blog.csdn.net/Mary19920410/article/details/72857764
>6. TCP 详解 https://blog.csdn.net/sinat_36629696/article/details/80740678
>7. TCP 协议（PSH 标志）https://blog.csdn.net/q1007729991/article/details/70154359
>8. TCP的初始序列号并非完全随机 https://blog.csdn.net/qq_43684922/article/details/89843306
>9. 带外数据和TCP紧急指针 https://blog.csdn.net/gbasp2008/article/details/47666421
>10. 理解TCP序列号（Sequence Number）和确认号（Acknowledgment Number）    https://blog.csdn.net/a19881029/article/details/38091243
>11.面试官，不要再问我三次握手和四次挥手 https://blog.csdn.net/hyg0811/article/details/102366854

# 原文
[QUIC](https://tools.ietf.org/html/draft-hamilton-early-deployment-quic-00)

# 2. 协定和一些定义 #
QUIC使用的所有整型数值都使用小端字节序而非网络序，包括长度值，版本号，类型等。QUIC对于动态大小的数据分片的类型并不强制要求对齐。  

以下是文章中使用的一些术语：  

- 客户端：QUIC连接的发起者端点。  
- 服务端：接受到来的QUIC连接的端点。  
- 终端：一个客户端或者服务端的连接的终点。  
- 流：在QUIC连接中通过特定的逻辑信道传递的双向数据流。  
- 连接：两个QUIC终端的会话，多路流在同一个连接将使用同一种加密方法。  
- 连接ID：QUIC连接的标识符。  
- QUIC包：一个可以被接受端解析的合法的UDP载荷。本文档中QUIC包大小可以参考UDP载荷大小。  

# 3. QUIC综述 #
这节我们要描述QUIC的主要机制和优势。QUIC机制上类似TCP+TLS+HTTP/2，但是是基于UDP的实现。QUIC和TCP+TLS+HTTP/2相比，优势如下：  

- 连接建立的延迟  
- 灵活的拥塞控制  
- 消除了head-of-line blocking的多路复用。[[HOL](https://en.wikipedia.org/wiki/Head-of-line_blocking)]  
- 经认证且加密的消息头和载荷  
- 流和连接的流量控制  
- 连接迁移  

## 3.1 连接建立的延迟 ##
QUIC合并了加密和传输握手，降低了建立加密连接所需要的消息交互来回次数。传统的TCP+TLS在发送数据之前需要经历1-3个来回的信息交互。与之相比，QUIC连接一般是0-RTT的，大部分QUIC连接，数据将被直接发送，不需要等待服务端的响应。  

QUIC使用了专用流(STREAM ID 1)来执行握手，但是具体的握手协议本文不进行描述。具体详见参考文档[2].QUIC当前的握手方式将在未来被TLS 1.3所替代。  

注：[[TLS 1.3 Zero-RTT](https://tlswg.github.io/tls13-spec/)]

## 3.2 灵活的拥塞控制 ##
QUIC拥有可插拔的拥塞控制，和TCP相比，提供了更丰富的信令，这让QUIC可以为拥塞控制算法提供更多的信息。当前默认的拥塞控制与TCP相同。我们正在实验一些可以替代TCP的拥塞控制算法。

举其中一个例子，我们在所有包(包括普通包和重传包)种加入了一个新的包序列号。这使得QUIC的发送方可以区分重传和普通的ACK报文，可以避免TCP的重传二义性问题(注：无法判断ACK响应的是普通包还是重传包，会影响RTT计算的精度，进而影响RTO)。QUIC的ACK报文携带了从包被接收到确认报文被发送的延迟，以及一个单调递增的包号，这些都可以提高RTT的计算精度。  

最后，QUIC的ACK分片支持最多256个ack块，因此QUIC在重排上相较于TCP的SACK更加具有弹性(注：SACK貌似每次发送SACK报文时候最多只能携带四组分组缺失信息)，也可以在丢包或者乱序时候保持更多的包(拥有更大的缓冲区)。同时客户端和服务端也都能更准确的知道对端的接收信息。  

## 3.3 流和连接的流量控制 ##

QUIC实现了流和连接级别的流量控制，与HTTP/2的流量控制类似。QUIC的流级别流量控制工作如下：一个QUIC接收端公布自己希望接收的最大数据偏移(注：单个连接中的多路复用，在一个数据流中，每个QUIC流具有自己的一个偏移)，当数据在某一条流中传输时，接收方发送WINDOW_UPDATE分片增加自己这条流的最大接收数据量，发送方可以相应的增加这条流上的数据发送量。  

除了每条流的流控外，QUIC还实现了连接级别的流量控制，根据接收端自己为每条连接分配的资源来限制总的buffer数量。连接级别的流量控制工作模式上和流的流量控制相同，不过分发和最大接收的字节是所有流的总和。  

类似于TCP接收窗口的自动调节，QUIC也实现了在流级别和连接级别的流量控制配额的自动调节。QUIC的自动调节会增加每一个WINDOW_UPDATE分片中的配额大小，限制发送端的速率，也可以在接收端应用速度慢的时候减少发送端的速率。  

## 3.4 多路复用 ##

基于TCP的HTTP/2会遇到HOL阻塞问题。因为HTTP/2将多个流在一个TCP连接上传输，一个TCP数据段的丢失会导致所有之后的数据段在重传包到来之前被阻塞，而无法顾及之后HTTP/2流中的被封装的其他数据端。  

因为QUIC被设计为多路复用操作，一个单独的流丢失的包所承载的数据一般只影响特定的流。每个流的分片在达到时可以被立刻分配到对应的流上，因此流没有数据丢失的时候可以持续组装并上报至应用层。

警告：QUIC当前通过HTTP/2 HPACK[[HPACK](https://tools.ietf.org/html/draft-ietf-httpbis-header-compression-06)]压缩HTTP头，这会导致HTTP头部分片的HOL。  

**注：消除了HOL可能是因为流是QUIC的内容，可以对流进行区分。而TCP不识别载荷的内容。**

## 3.5 可靠及加密的头部和载荷  ##

由于TCP头部在信道上是明文传输且未经过验证的，因此导致了许多注入和头部伪造的问题，比如接收窗口伪造和序列号重写的问题。当其中的一些攻击激活时，其他用于网络中的一些中间件的类似机制有时正在透明的提高TCP性能。然而，即使这些中间件正在增强TCP性能，但是仍然限制着传输协议的可进化性(原文：However, even "performance-enhancing" middleboxes still effectively limit the evolvability of the transport protocol，个人理解为TCP性能的演进)，正如从MPTCP(multi-path tcp)及其后续的部署问题。  
**注：个人理解，当某些攻击生效时，可能会破坏TCP的公平性，甚至会占满整个机器上的带宽。如伪造TCP连接，伪造ACK，使TCP认为链路状态较好，并不断提升该链路性能，导致其他链路质量下降。**    

QUIC包总是经过认证，且其载荷都是经过加密的。QUIC包头的一部分没有经过加密，但是这一部分会通过接收端进行验证，因此可以阻止第三方对任何包进行注入或者篡改。QUIC能保护端到端通信中那些受到有意或者无意的第三方篡改的连接。  

警告：重置一个连接的PUBLIC_RESET包当前是未验证的。  

## 3.6 连接迁移 ##

TCP连接由源地址、源端口、目的地址、目的端口这个四元组标识。TCP连接在IP改变(如wifi切换为蜂窝信号)或者端口改变(如客户端NAT端口绑定过期导致服务端可见的对端端口改变)时无法保持连接是一个众所周知的问题。当MPTCP面对TCP连接迁移的问题时，仍旧会因为缺少中间件的支持和操作系统的调度而变得麻烦。  

QUIC连接由客户端随机生成的一个64位连接ID标记。当地址发生变化或者NAT重新绑定，这个连接ID在这次连接迁移中保持不变，因此QUIC可以通过这种机制保持链路存活。QUIC同样提供了自动加密的认证，因此一个迁移后的客户端仍然可以使用相同的会话键值来对包进行加密解密。  

在一些场景下，当连接明通过四元组明确标识时，比如服务端使用一个临时端口向客户端发送报文，此时可以配置成为不发送连接ID来节约网络流量。  

# 4. 包类型及格式 #

QUIC包分为普通包和特殊包。其中特殊包包含两类，版本协商包和公共重置包。所有QUIC包都应该被限制大小以适应路径MTU，防止IP层分片。路径MTU发现是一项正在进行的工作，当前QUIC针对IPV6使用1350字节最大包大小，IPV4使用1370包大小，两个大小均不包含IP和UDP开销。

## 4.1 QUIC公共包头 ##

所有QUIC包的数据格式都有一个2到19字节的公共头。公共头的数据格式如下：

	     0        1        2        3        4            8
	+--------+--------+--------+--------+--------+---    ---+
	| 公共   |    连接标识  (0 or 64)    ...                 | ->
	|标志(8) |      (可变长度)                               |
	+--------+--------+--------+--------+--------+---    ---+

	     9       10       11        12
	+--------+--------+--------+--------+
	|      QUIC 版本号 (32)              | ->
	|         (可选)                     |
	+--------+--------+--------+--------+

	    13       14       15        16      17       18       19       20
	+--------+--------+--------+--------+--------+--------+--------+--------+
	|                              多种场景                                  | ->
	|                              （可选）                                  |
	+--------+--------+--------+--------+--------+--------+--------+--------+

	    21       22       23        24      25       26       27       28
	+--------+--------+--------+--------+--------+--------+--------+--------+
	|                          多种场景后续部分                               | ->
	|                              （可选）                                  |
	+--------+--------+--------+--------+--------+--------+--------+--------+
	
	    29       30       31        32      33       34       35       36
	+--------+--------+--------+--------+--------+--------+--------+--------+
	|                          多种场景后续部分                               | ->
	|                              （可选）                                  |
	+--------+--------+--------+--------+--------+--------+--------+--------+
	
	    37       38       39        40      41       42       43       44
	+--------+--------+--------+--------+--------+--------+--------+--------+
	|                           多种场景后续部分                              | ->
	|                              （可选）                                  |
	+--------+--------+--------+--------+--------+--------+--------+--------+
	
	
	    45      46       47        48       49       50
	+--------+--------+--------+--------+--------+--------+
	|           包序号 (8, 16, 32, or 48)                  |
	|                  (可变长度)                          |
	+--------+--------+--------+--------+--------+--------+

载荷中可能包含多种下述若干字节的类型相关的包头。

公共头字段描述如下：

- 公共标志位：

 - 0x01 = PUBLIC\_FLAG\_VERSION. 这个标志位的解释取决于这个包是由服务端发出的还是客户端发出的。如果是客户端发出的，设置了这个标志位表明这个头部包含了QUIC版本号信息（见下）。在服务端确认接受客户端请求的版本之前，客户端发送的所有包都必须要设置这个标志位。服务端接受这个版本则会发送未设置这个标志位的包。如果服务端设置了这个标志位，那么这么包就是一个版本协商包。版本协商将在后面具体描述。  
 - 0x02 = PUBLIC\_FLAG\_RESET. 设置了这个标志位表明这个包是一个重置包。  
 - 0x04 = 表明该头部中存在一个32字节大小的多种特殊场景信息。  
 - 0x08 = 表明该包中包含了完整的8字节大小的连接ID。这个标志位必须一直被设置，直到在某个数据方向上协商出一个不同的值(如，客户端请求一个较短的连接ID)。  
 - 0x30 的这两位共同表示了当前包序号所使用的字节数。这个标志位一般用在分片的包中。对于重置和版本协商包(服务端发出)不需要包序号，因此这两位没有使用必须置为0。  
 这两位分别表示：
		- 0x30 表示当前使用6字节的包序号。
		- 0x20 表示当前使用4字节的包序号。
		- 0x10 表示当前使用2字节的包序号。
		- 0x00 表示当前使用1字节的包序号。 
 - 0x40 预留给多路径使用。  
 - 0x80 当前未使用，需要设置为0。  

- 连接ID是一个由客户端选择的64比特长度的统计随机数，用来标识该条连接。QUIC连接设计的初衷就是为了客户端在漫游状态，使用IP四元组(源IP、目的IP、源端口、目的端口)不足以标识这条连接时，依然可以维持连接。对于每一个传播方向，当四元组可以明确的标识这条连接时，连接ID可能被忽略。
  
- QUIC版本号：用一个32位不透明的标签来代表QUIC协议的版本。这个字段只出现在公共标志位包含FLAG\_VERSION(如public\_flags & FLAG\_VERSION !=0)。客户端可以设置这个标志位，同时包含一个正确的建议版本，此外可以包含任意数据(遵循版本格式)。当服务端不支持客户端提议的版本，可以设置这个标志位，同时可以提供一个可接受版本列表(0或者更多)，但是**不可以**在版本信息之后包含任何数据。举个例子，近期实验性版本值为"Q025"，其第九字节为"Q",第十字节为"0"，以此类推。(第9-12字节是属于QUIC版本号的值字段)  
- 包序号：包序号的有效位是最低的8位、16位、32位或者48位，取决于公共标志位中的FLAG\_?BYTE\_SEQUENCE\_NUMBER的设置。发送端会给每一个普通包(不同于特殊的重置包和版本协商包)分配一个包序号。  任一终端发出的第一个包都应该以包序号1为起始，之后的包都在前一个包的包序号基础上加一。包序号的低64位用于加密场景的一部分。因此，QUIC的终端不能发送包序号无法用64位表示的包。如果一个QUIC终端发送了序号为(2^64 - 1)的包，那么这个包必须包含一个带有QUIC\_SEQUENCE\_NUMBER\_LIMIT\_REACHED错误信息的CONNECTION\_CLOSE分片，同时这个终端不允许再传输任何其他包。包序号最多只有低48位会被传输。为了使接收端可以明确的恢复包序号，一个QUIC终端不可以传输包序号超过接收端确认已经被收到的最大包序号加上(2^(bitlength-2))的包。因此链路中不可能有超过(2^46)个包。任何一个被截断的包序号都应该能被推断出其值最接近传输这个被截断的包序号的终端已知的最大包序号加一。被传输部分的包序号和最低位的推测值匹配。

公共标志位处理流程如下：

			检查公共头中的公共标志位
	                 |
	                 |
	                 V
	           +--------------+
	           | 重置标志      |    是
	           | 被设置？      |---------------> 重置包
	           +--------------+
	                 |
	                 | 否
	                 V
	           +------------+          +-------------+
	           | 版本标志    |   是     | 由服务端     |  是
	           | 被设置？    |--------->| 发出?       |--------> 版本协商包
	           +------------+          +-------------+  
	                 |                        |
	                 | 否                     | 否
	                 V                        V
	               普通包              带QUIC版本信息的普通包

## 4.2 特殊包 ##

### 4.2.1 版本协商包 ###

一个版本协商包仅由服务端发出。版本协商包以8位的公共标志位和64位的连接ID开始。公共标志位必须设置PUBLIC\_FLAG\_VERSION标志位且指明64位的连接ID。版本协商包的其余部分为服务端支持的一组4字节的版本号列表。  

	     0        1        2        3        4        5        6        7       8
	+--------+--------+--------+--------+--------+--------+--------+--------+--------+
	| 公共    |    连接ID (64)                                                     | ->
	|标志位(8)|                                                                       |
	+--------+--------+--------+--------+--------+--------+--------+--------+--------+
	
	     9       10       11        12       13      14       15       16       17
	+--------+--------+--------+--------+--------+--------+--------+--------+---...--+
	|      第一个服务端支持的版本号        |     第二个服务端支持的版本号         |   ...
	|              (32)                 |              (32)                 |
	+--------+--------+--------+--------+--------+--------+--------+--------+---...--+

### 4.2.2 重置包 ###

一个公共重置包由8位的公共标志位和64位连接ID开始。公共标志位必须设置PUBLIC\_FLAG\_RESET且指明64位的连接ID。如果这是PRST标签的一个加密握手信息，则重置包的其余部分需要被编码(见[[QUIC-CRYPTO](https://docs.google.com/document/d/1g5nIXAIkN_Y-7XJW5K45IblHd_L2f5LTaDUDwvZ5L6g/edit)])。   

	        0        1        2        3        4         8
	   +--------+--------+--------+--------+--------+--   --+
	   | 公共    |    连接ID (64)                    ...  | ->
	   |标志位(8)|                                           |
	   +--------+--------+--------+--------+--------+--   --+
	
	        9       10       11        12       13      14
	   +--------+--------+--------+--------+--------+--------+---
	   |      Quic 标签 (32)                |  标签-值映射表      ... ->
	   |         (PRST)                    |  (变长)
	   +--------+--------+--------+--------+--------+--------+---

标签-值映射表：标签-值映射表包含了如下几种标签-值对：  

- RNON (重置场景描述) 一个64位的无符号整数，强制选项。  
- RSEQ (拒绝包序号) 一个64位的包序号，强制选项。  
- CADR (客户端地址) 观察到的客户端IP和端口号。这个选项当前只用于调试，因此是一个可选择项。  

(TODO:重置包应该包含验证过的(目的端)服务端IP或端口。)  

## 4.3 普通包 ##

普通包是经过验证和加密的。包头经过验证但不是加密的，以第一个分片开始的部分是加密的。紧随头部之后的是AEAD(验证加密的关联数据)数据。数据必须依序解密以保证内容可以被正确解释。解密之后明文由按顺序排好的分片组成。  

(TODO:加解密的方法需要文本化描述。)  

### 4.3.1 数据分片 ###

数据分片数据由一系列类型前缀的数据分片所组成。分片类型在本文档后面章节介绍，这里给出一般的数据分片格式如下：  

	   +--------+---...---+--------+---...---+
	   | 类型    | 载荷    | 类型    | 载荷    |
	   +--------+---...---+--------+---...---+
	   
# 5. QUIC连接的生命周期 #

## 5.1 连接建立 ##

QUIC连接由QUIC客户端发起。QUIC连接建立时会把版本协商以及加密、传输握手过程交织在一起，以降低连接建立的延迟。我们首先描述版本协商的过程。  

每一个客户端发往服务端的初始化包需要设置版本标志位，同时指明自己使用的版本信息，直到客户端收到服务端关闭版本标志位的包。在服务端收到第一个关闭了版本标志位的包之后都必须忽略所有打开版本标志位的包(可能由于延迟导致)。  

当服务端收到一个新连接的包时，将要比较客户端和本地所支持的版本信息。如果客户端的版本是服务端支持的，那么服务端将在这条连接的生命周期中使用该版本。在这种情况下，服务端发送的所有包都会关闭版本标志位。  

如果客户端的版本服务端无法支持，那么会产生一个RTT的消息交互。服务端将发送一个版本协商包给客户端。这个包将设置版本标志位并携带服务端所支持的版本信息集合。  

当客户端收到从服务端发来的版本协商包，其将选择一个可以接受的版本并使用这个版本信息重新发送所有包。最后当客户端收到第一个服务端发来的普通包(比如不是版本协商包，根据上文也不应该为重置包)，表示版本协商结束，后续客户端发送的包都不需要设置版本标志位。  

为了避免降级攻击([[降级攻击](https://en.wikipedia.org/wiki/Downgrade_attack)])，客户端第一个包中指明版本的信息和服务端所支持的版本信息集合都需要包含在加密的握手数据中。客户端需要验证版本协商包中握手信息里的服务器的版本信息，而服务端要验证客户端握手信息中的协议版本是否可以支持。  

连接建立的其余部分在关于握手过程的文档中有描述[QUIC-CRYPTO]。加密的握手过程应用于加密的流(流号为1)。  

在连接建立的过程中还需要协商大量的传输参数，当前定义的传输参数见后续描述。  

## 5.2 数据传输 ##

QUIC连接实现了可靠传输，拥塞控制，流量控制。QUIC的流量控制紧密的遵循HTTP/2的流量控制。QUIC的可靠传输和拥塞控制在附录的文档中有描述。QUIC连接使用单一的包序列号空间，以便不同的连接共享相同的拥塞控制和丢包恢复。  

QUIC连接中传输的数据，包括加密的握手信息，除了QUIC的ACK确认包外，都是以流中的数据进行发送。  

本章概念性地描述了QUIC连接中的用于传输的流的使用。多种本章提及的分片信息主要描述了分片类型和格式。  

### 5.2.1 QUIC流的生命周期 ###

流是拥有独立序列的双向数据，被切分为若干个流分片。流可以被客户端或者服务端创建，若干条流的数据可以并行传输，也可以被取消。QUIC中流的生命周期严格的模拟自HTTP/2[[RFC-7540](https://tools.ietf.org/html/rfc7540)]\(HTTP\2使用QUIC流的细节在后文描述)。  

流是通过发送一个STREAM分片隐式创建的。为了避免流ID的冲突，服务端初始化的流的流ID为偶数，客户端初始化的流的流ID为奇数。0为无效流ID。流1预留给了客户端初始化的加密握手。当在HTTP\2中使用QUIC时，流3预留给了其他所有流传输压缩头，确保可靠有序的分发和处理这些头数据。  

来自每一边的流ID必须在新创建流时单调递增。举个例子，流2可能在流3后创建(因为不同端)，但是流7不应该在流9之后创建(因为在相同端)。对端可能不按照顺序接收到流数据。举个例子，服务端可能在收到流7的包9之前就已经收到了流9的包10，这种场景需要优雅的处理。  

当一个终端收到STREAM分片时并不想接受这个流，可以立刻回复一个RST_STREAM分片拒绝这条流(下表)。注意，此时正在初始化流的一端可能已经发送了若干数据，这些数据必须被忽略。  

一旦一个流成功被建立，则其可以用来发送和接收数据。一系列流分片可以通过这条流传递直到这条流的这个方向被终止。

每个QUIC终端都可以终止一条流，终止方式有以下三种：  

(1. 正常终止: 因为QUIC的流是双向的，因此有半关闭和关闭两种状态。当流的一端发送一个设置了FIN标志位的分片，这个流的这个方向就进入半关闭状态。FIN标志位意味着发送这个FIN标志位的一端将不会再发送数据。当一个QUIC终端既发送了FIN也收到了FIN，那么这个终端认为这条流已经进入关闭状态。当一个FIN包需要在流中的用户最后一个数据发送后发送时，FIN标志位可以在最后一个数据包发送之后，以一个空的流分片形式发出。  

(2. 突然终止:一条流的客户端和服务端都可以随时发出一个RST_STREAM分片。一个RST_STREAM分片包含了一个指明了失败原因的错误码(下表)。当流的发起者发出了RST_STREAM分片，意味着完成传输的过程中出现了异常，同时后续将不再有数据发送。当RST_STREAM由流的接收方发出，这时候的发送端不可以在这条流上再发送任何数据。

(3. 流将随着连接终止而终止，下节详述。  

### 5.3 连接终止 ###

连接在空闲后仍需保留一段预协商的周期长度的时间。当一个服务端决定终止一条空闲连接，应该可以不需要通知客户端以防止客户端唤醒移动终端上的无线设备。一个QUIC连接，一旦成功建立连接，有以下两种方法可以中断该连接：  

(1. 显式关闭： 一个终端向对端发送了一个CONNECTION\_CLOSE分片即开始了一次连接终止。此外，一个终端可以先于CONNECTION\_CLOSE之前发送GOAWAY分片，这个分片将告诉对端，这条连接上的存在的流将继续处理，但是不再接受新请求的流。要终止已经激活的流，需要发送CONNECTION\_CLOSE分片。当一个CONNECTION\_CLOSE被发出时还有未终止的流处于激活状态(有大于一条流未发出或者收到FIN位的分片或者RST\_STREAM分片)，对端必须嘉定这些流不完整或者被异常终止。  

(2. 隐式关闭： QUIC连接默认空闲超时时间为30秒，可以利用“ICSL”参数进行协商。最大值为10分钟。 当空闲超时时间内没有活跃的网络操作，这条连接就会被关闭。默认情况下会发送一个CONNECTION\_CLOSE分片。当发送一个显式关闭报文代价较大(如手机网络会唤醒无线接收器)时静默关闭选项可以被使能。  

任一终端也可能在任意时刻发送PUBLIC\_RESET包来突然断开连接。PUBLIC\_RESET等价于TCP中的RST包。  

## 6. 分片类型和格式  ##

QUIC分片包由分片填充。分片有一个分片类型，这个分片类型中包含了类型相关的分片头和其后类型相关的解释。所有分片被包含在单独的一个QUIC包中，且没有分片可以跨越QUIC包的边界。  

### 6.1 分片类型 ###

分片类型分为两种，特殊分片类型和常规分片类型。特殊分片类型在分片类型的字节中存放了分片类型和对应的一些标志位，而常规分片类型仅简单使用分片类型字节(写死)。  

当前定义的特殊分片类型如下：(注：这里的f、d、o等并不是值，而是标志位的标识符)

      +------------------+-----------------------------+
      | Type-field value |     Control Frame-type      |
      +------------------+-----------------------------+
      |     1fdooossB    |  STREAM                     |
      |     01ntllmmB    |  ACK                        |
      |     001xxxxxB    |  CONGESTION_FEEDBACK        |
      +------------------+-----------------------------+

当前定义的常规分片类型如下：

      +------------------+-----------------------------+
      | Type-field value |     Control Frame-type      |
      +------------------+-----------------------------+
      | 00000000B (0x00) |  PADDING                    |
      | 00000001B (0x01) |  RST_STREAM                 |
      | 00000010B (0x02) |  CONNECTION_CLOSE           |
      | 00000011B (0x03) |  GOAWAY                     |
      | 00000100B (0x04) |  WINDOW_UPDATE              |
      | 00000101B (0x05) |  BLOCKED                    |
      | 00000110B (0x06) |  STOP_WAITING               |
      | 00000111B (0x07) |  PING                       |
      +------------------+-----------------------------+

### 6.2 STREAM分片 ###

STREAM分片用于暗示创建一条流并用于传输数据，具体如下：

	     0        1       ...               SLEN
	+--------+--------+--------+--------+--------+
	|Type (8)| Stream ID (8, 16, 24, or 32 bits) |
	|        |    (Variable length SLEN bytes)   |
	+--------+--------+--------+--------+--------+
	
	  SLEN+1  SLEN+2     ...                                         SLEN+OLEN
	+--------+--------+--------+--------+--------+--------+--------+--------+
	|   Offset (0, 16, 24, 32, 40, 48, 56, or 64 bits) (variable length)    |
	|                    (Variable length: OLEN  bytes)                     |
	+--------+--------+--------+--------+--------+--------+--------+--------+
	
	  SLEN+OLEN+1   SLEN+OLEN+2
	+-------------+-------------+
	| Data length (0 or 16 bits)|
	|  Optional(maybe 0 bytes)  |
	+------------+--------------+
	
STREAM分片头字段如下：

- 分片类型： 分片类型字节是一个8比特值，包含以下标志位(1fdooossB)：
	- 最左边的1位设置成1表示这个分片是一个STREAM分片。
	- f比特是FIN比特，当其置位时表示发送端发送完成，并即将进入半关闭状态。
	- d比特表示该分片是否携带数据包长度信息。当其为0时表示这个STREAM分片将延展到包的末尾。
	- 接下来的三个"ooo"比特表示偏移编码的位数，对应长度0,16,24,32,40,48,56,64。
	- 接下来的两个"ss"比特表示流标识符的编码长度，对应长度8,16,24,32。

- 流标识符: 变长大小的无符号标识符来唯一标识这条流。
- 偏移: 变长大小的无符号数指明流中数据的字节偏移。
- 数据长度: 可选的16比特无符号数，表明这个流分片中的数据长度。只有这个包全是数据时可以忽略这个选项，避免数据被填充字段污染的风险。

一个STREAM分片必须包含非零长的数据或者FIN比特被置位。

### 6.3 ACK分片 ###

ACK分片被发送来告知对端本地受到了多少数据包，也有接收端丢包(可能需要重传)。ACK分片包含了1到256个ACK块。ACK块是意境确认的包的范围，类似于TCP的SACK块，但是与TCP累计ack略有不同，因为QUIC重传包会使用新的包序列号。  

为了将ACK块限于那些还未接收的包，对端会周期性发送STOP\_WAITING分片来表示接收端停止对某个序号之下的包进行确认，即接收端最小未确认的包序号的提升(个人理解：功能类似不丢包场景下的ACK)。ACK分片的发送端只汇报最小未确认的包和已知最大序号的包中间的块(个人理解：功能类似SACk)。发送端收到的ACK分片中的最大已经确认包应该和STOP\_WAITING分片中的最小未确认包进行相同的处理。(TODO)  

不同于TCP的SACK，QUIC的ACK块是不可取消的，当一个包被确认了，即使后来并没有出现在已经确认的块里，这个包仍然被认为是已经被确认的。(个人理解：假设第一个ACK包的某个块确认了包N，但是下一个ACK包的N却又是未确认的，这时候仍然认为这个包N是已经确认的)  

作为替代QUIC启用的熵方案，发送端在连接中故意跳过包号来引入熵。如果一个包没有发送，但是却被确认了，此时发送端必须关闭该链接，这种机制可以自动抵御某些潜在的攻击。ACK分片的格式对于描述块中丢失的包较有效率，因此发送端和接收端开销较小，且仅在有需要时提供了高达8位的熵信息，而不是招致恒定的开销和总是获取8位的熵信息。这8位时ack格式中ack范围之间最长的间隙。  

区间偏移：  

0: ACK分片的开始  

T: 时间戳区间与分片开始的字节偏移  

A: ACK块区间与分片开始的字节偏移  

N: 最大被确认的包序号的字节长度

	     0                            1  => N                     N+1 => A(即 N + 3)
	+---------+-------------------------------------------------+--------+--------+
	|   Type  |                   Largest Acked                 |  Largest Acked  |
	|   (8)   |    (8, 16, 32, or 48 bits, determined by ll)    | Delta Time (16) |
	|01nullmm |                                                 |                 |
	+---------+-------------------------------------------------+--------+--------+
	
	
	     A             A + 1  ==>  A + N
	+--------+----------------------------------------+
	| Number |             First Ack                  |
	|Blocks-1|           Block Length                 |
	| (opt)  |(8, 16, 32 or 48 bits, determined by mm)|
	+--------+----------------------------------------+
	
	  A + N + 1                A + N + 2  ==>  T(即 A + 2N + 1)
	+------------+-------------------------------------------------+
	| Gap to next|              Ack Block Length                   |
	| Block (8)  |   (8, 16, 32, or 48 bits, determined by mm)     |
	| (Repeats)  |       (repeats Number Ranges times)             |
	+------------+-------------------------------------------------+
	     T        T+1             T+2                 (Repeated Num Timestamps)
	+----------+--------+---------------------+ ...  --------+------------------+
	|   Num    | Delta  |     Time Since      |     | Delta  |       Time       |
	|Timestamps|Largest |    Largest Acked    |     |Largest |  Since Previous  |
	|   (8)    | Acked  |      (32 bits)      |     | Acked  |Timestamp(16 bits)|
	+----------+--------+---------------------+     +--------+------------------+

ACK分片字段如下：  

- 分片类型: 分片类型字节包含了下列标志位(01nullmmB)

	- 前两位设置为01表示这是一个ACK分片。  
	- n比特表示这个ACK分片是否包含超过1个的ACK范围。  
	- u比特未使用。  
	- 两个l比特表示最大已确认包号的长度，为1,2,4,6字节长度。  
	- 两个m比特表示丢包序号差字段的长度，为1,2,4,6字节长度。  

- 最大已确认包号: 变长大小无符号字段表示当前已经收到的对端的最大包序号。  
- 最大已确认的差量时间: 16位无符号浮点数，其中11位为尾数部分，5位为指数部分，这个值表明从收到最大的包到这个ACK分片发送的时间，单位为毫秒。比如，1毫秒表示为0x1，即指数部分(高5位)为0，尾数部分(低11位)为1。当指数部分大于零，则暗示尾数部分高位补上10，补至13位。比如: 一个浮点值为0x800，指数部分为1，明显大于0，且此时尾数位0，因此此时尾数位0x1000，即4096。此外，实际的指数总是比表面指数小1，因此指数应该为0，此时为2^0 * 4096，即4096毫秒。任何大于可表示范围的值都将表示为0xffff。  
- ACK块区间:  
	- 块数: 可选的8位无符号值，表示块数减一。仅当分片类型中的n标志位置位时生效。    
	- ACK块长度: 一个变长包序号差量，对于第一个丢失包的范围，ACK块开始于最大已确认包序号加一。对于后续的块，该值表示ACK块的长度。对于非首块，如果值为0表示该块对应的包都丢了。  
	- 到下一个块的间隙: 一个8位无符号数表示ACK块之间的包数。  

- 时间戳区间:   
	- 时间戳数量: 一个8位的无符号数表示这个ACK分片中包含的时间戳数量。后续内容由若干组<包号，时间戳>组成。  
	- 最大包的时间戳差量: 一个8位的无符号值表示第一个时间戳与最大的包号时间戳的差量。因此，当前包号等于最大的包号减去差量。  
	- 第一个时间戳: 一个32比特的无符号值表明毫秒级的时间差，由最大包的时间戳减去最大的时间戳差得到。  
	- 最大时间差: 同上。  
	- 与上一个时间戳的差: 一个16位无符号值表示与前一个时间戳的差值，编码格式同ACK延迟时间。  

### 6.4 STOP_WAITING分片 ###

STOP\_WAITING分片被发送来通知对端不应该持续等待包号小于某个指定值。包号长度为1，2，4，6字节长，和包头中指明的包号长度相同(QUIC帧包的公共标志位字段中指明)。分片格式如下:  

	        0        1        2        3         4       5       6
	   +--------+--------+--------+--------+--------+-------+-------+
	   |Type (8)|   Least unacked delta (8, 16, 32, or 48 bits)     |
	   |        |                       (variable length)           |
	   +--------+--------+--------+--------+--------+--------+------+

STOP\_WAITING分片的字段如下:  

- 分片类型: 分片类型字节是一个8位的值，必须设置为0x06表示这是一个STOP\_WAITING分片。  
- 最小未确认包差量: 一个边长包号差量，长度与包头中的包号长度相同。包头序号减去这个值就是最小未确认的包号。得到的最小未确认包就是发送端持续等待ACK的序号最小的包。如果如果接收端丢失任何小于这个值的包，那么接收端应该认为这些包是不可复原的丢失。

### 6.5 WINDOW_UPDATE分片

WINDOWS\_UPDATE分片用于通知对端增加一个终端的流量控制接收窗口。流标识符可以是0，表示这个WINDOWS\_UPDATE分片是用作于连接级别的; 流标识符大于零表示指定流增加其流量控制窗口。  

该分片中指定了一个绝对的字节偏移，WINDOWS\_UPDATE分片的接收端在指定流上最多只能发送这么多的字节数。违反流量控制，发送超出限制的字节数会导致接收端关闭连接。  

当某条流收到多个WINDOW\_UPDATE分片，只需要保持最大字节偏移的轨迹。  

流和会话的初始窗口值都为16KB，但是在握手的过程中可以增加。因此，一个终端应该在握手过程中协商SFCW(流级流量控制窗口)和CFCW(连接/会话级别流量控制窗口)参数。  

分片描述如下:  
	
	       0         1                 4        5                 12
	   +--------+--------+-- ... --+-------+--------+-- ... --+-------+
	   |Type(8) |    Stream ID (32 bits)   |  Byte offset (64 bits)   |
	   +--------+--------+-- ... --+-------+--------+-- ... --+-------+

WINDOW\_UPDATE分片的字段如下:  

- 分片类型: 分片类型字节为8位值必须设置为0x04表示这个分片是一个WINDOW\_UPDATE分片。  
- 流标识符: 需要更新流量控制窗口的流标识符，当值为0时表示增加的是连接级别的流量控制窗口。  
- 字节偏移: 64位无符号整型表示指定流可以发送的数据最大偏移。对于连接级别的流量控制，表示连接上所有流累积的发送字节。  

### 6.6 BLOCKED分片

BLOCKED分片用于通知远端本地终端已经准备发送数据(或者已经有数据待发送)，但是当前被流量控制所阻塞。这个分片单纯是一个信息分片，仅用于调试。BLOCKED分片的接收端应该简单的丢弃这个分片(在打印出相应的日志信息后)。  

	        0        1        2        3         4
	   +--------+--------+--------+--------+--------+
	   |Type(8) |          Stream ID (32 bits)      |
	   +--------+--------+--------+--------+--------+

BLOCKED分片字段如下:  

- 分片类型: 分片类型字节是一个8位的值必须设置为0x05表示这个分片是一个BLOCKED分片。  
- 流标识符: 32位无符号值表示被流控阻塞的流。非零的流标识符表示这个流被阻塞，当其为0时，表示整条连接都被连接级别的流控阻塞。  

### 6.7 CONGESTION_FEEDBACK分片

CONGESTION\_FEEDBACK分片是一个实验性的分片，当前未使用。这个分片的本意是在ACK分片之外提供额外的拥塞反馈信息。CONGESTION\_FEEDBACK分片类型的前三位必须为001，后5位预留。  

### 6.8 PADDING分片

PADDING分片填充一个包，其分片类型字节为0x00。当出现这个分片时，包的剩余部分都是填充字节。

### 6.9 RST_STREAM分片

RST_STREAM分片允许异常终止一条流。当这个分片是流的创建者发出的，表示创建者希望取消这条流。当接收端发送这个分片，表示有错误或者当前接收端不希望接收这个流，因此这个流应该被关闭。

	     0        1            4      5              12     8             16
	+-------+--------+-- ... ----+--------+-- ... ------+-------+-- ... ------+
	|Type(8)| StreamID (32 bits) | Byte offset (64 bits)| Error code (32 bits)|
	+-------+--------+-- ... ----+--------+-- ... ------+-------+-- ... ------+

RST_STREAM分片的字段如下:  

- 分片类型: 分片类型字节为8位值必须设置为0x01表示这是一个RST_STREAM分片。  
- 流标识符: 32位流标识符，表示将被中断的流。  
- 字节偏移: 64位无符号整型表示流数据的绝对字节偏移。  
- 错误码: 32位的QUIC错误码指明流被终端的原因，错误码后续列出。  

### 6.10 PING分片

PING分片用于保活。PING分片不包含任何载荷。PING分片的接收端需要对这个分片进行应答。PING分片需要在连接开启的时候对连接进行保活。默认PING分片的周期为15秒，短于大部分[NAT超时](http://www.cnblogs.com/sjjg/p/5830082.html)时间。PING分片只有分片类型字段，必须置为0x07。  

### 6.11 CONNECTION_CLOSE分片

CONNECTION\_CLOSE分片用来通知连接将被关闭。如果某些流仍然有数据在发送，那么这些流将隐式关闭(理论上一个GOAWAY分片应该已经被发送了足够的时间使所有流都关闭)。

	        0        1             4        5        6       7
	   +--------+--------+-- ... -----+--------+--------+--------+----- ...
	   |Type(8) | Error code (32 bits)| Reason phrase   |  Reason phrase
	   |        |                     | length (16 bits)|(variable length)
	   +--------+--------+-- ... -----+--------+--------+--------+----- ...

CONNECTION\_CLOSE分片的字段如下:  

- 分片类型: 8比特值必须设置为0x02表示这个分片是一个CONNECTION\_CLOSE分片。  
- 错误码: 32位字段包含了QUIC错误码表明连接关闭原因。  
- 原因描述长度: 可读的连接关闭原因长度。  
- 原因描述: 可读的连接关闭原因。  

### 6.12 GOAWAY分片

GOAWAY分片通知该连接将停止使用，将来将要被终止。任何激活的流在收到GOAWAY分片后将持续处理数据，但是GOAWAY分片的发送端将不再初始化任何附加的流，也不会接受任何新的流。  

	        0        1             4      5       6       7      8
	   +--------+--------+-- ... -----+-------+-------+-------+------+
	   |Type(8) | Error code (32 bits)| Last Good Stream ID (32 bits)| ->
	   +--------+--------+-- ... -----+-------+-------+-------+------+
	
	         9        10       11
	   +--------+--------+--------+----- ...
	   | Reason phrase   |  Reason phrase
	   | length (16 bits)|(variable length)
	   +--------+--------+--------+----- ... 

GOAWAY分片字段如下:  

- 分片类型: 8比特的值必须设置为0x06表示这个分片是一个GOAWAY分片。    
- 错误码: 32比特字段包含QUIC错误码表示关闭连接的原因。  
- 上一个好的流标识符: 上一个被GOAWAY发送端接收的流标识符。如果没有流可以用来回复，则这个值为0。  
- 原因描述长度: 可读的连接关闭原因长度。  
- 原因描述: 可读的连接关闭原因。  

## 7. QUIC传输参数

QUIC握手有责任为QUIC连接协商一些传输参数。  

### 7.1 必要参数

- SFCW: 流级别的流量控制窗口。字节级别大小。
- CFCW: 连接级别的流量控制窗口。字节级别大小。

### 7.2 可选参数

- SRBF: 套接字接收buffer字节大小。对端可能需要限制他们的最大拥塞窗口(?发送窗口还是拥塞窗口)，防止数据在内核缓冲区中产生延迟。默认为256kbytes，最小为16kbytes。  
- TCID: 连接标识符截断。表示支持截断的连接标识符。如果由对端发送，标志发送到对端的连接标识符必须被截断到0字节。一种有效的场景为: 当客户端仅用一个临时端口来使用一个单独的连接。  
- COPT: 连接选项是一个重复的标签字段。这些字段包含客户端或者服务端所请求的所有连接选项。主要用于实验，后续将进行演进。例如，使用这个参数来初始化拥塞控制算法和其相关参数。  

## 8. QUIC错误码

QUIC错误与错误码映射关系定义在src/net/quic/quic\_protocol.h中。  

QUIC\_NO\_ERROR: 无错误。这个值对于RST\_STREAM分片和CONNECTION\_CLOSE分片无效。  

QUIC\_STREAM\_DATA\_AFTER\_TERMINATION: 在FIN或者RESET状态之后仍然有数据到达。  

QUIC\_SERVER\_ERROR\_PROCESSING\_STREAM: 服务端处理流数据异常。  

QUIC\_MULTIPLE\_TERMINATION\_OFFSETS: 发送端的某个流收到了两个偏移不匹配的FIN或者RESET。  

QUIC\_BAD\_APPLICATION\_PAYLOAD: 发送端收到了损坏的应用层数据。  

QUIC\_INVALID\_PACKET\_HEADER: 发送端收到了异常的包头。  

QUIC\_INVALID\_FRAME\_DATA: 发送端收到一个分片数据，更多的细节的错误码会优先选择。  

QUIC\_INVALID\_FEC\_DATA: 异常的FEC数据。  

QUIC\_INVALID\_RST\_STREAM\_DATA: 流RST数据异常。  

QUIC\_INVALID\_CONNECTION\_CLOSE\_DATA: 连接关闭数据异常。  

QUIC\_INVALID\_ACK\_DATA: Ack数据异常。  

QUIC\_DECRYPTION\_FAILURE: 解密错误。  

QUIC\_ENCRYPTION\_FAILURE: 加密错误。  

QUIC\_PACKET\_TOO\_LARGE: 包大小超过最大值。  

QUIC\_PACKET\_FOR\_NONEXISTENT\_STREAM: 数据发送到一个不存在的流。  

QUIC\_CLIENT\_GOING\_AWAY: 客户端即将离开(浏览器关闭等)。  

QUIC\_SERVER\_GOING\_AWAY: 服务端即将离开(重启等)。  

QUIC\_INVALID\_STREAM\_ID: 无效的流标识符。  

QUIC\_TOO\_MANY\_OPEN\_STREAMS: 打开的流过多。  

QUIC\_CONNECTION\_TIMED\_OUT: 达到预协商(或者默认)的超时时间。  

QUIC\_CRYPTO\_TAGS\_OUT\_OF\_ORDER: 握手信息中包含了乱序的标签。  

QUIC\_CRYPTO\_TOO\_MANY\_ENTRIES: 握手信息中包含过多的实例。  

QUIC\_CRYPTO\_INVALID\_VALUE\_LENGTH: 握手信息中包含无效的长度值。  

QUIC\_CRYPTO\_MESSAGE\_AFTER\_HANDSHAKE\_COMPLETE: 握手完成后收到一个加密信息。  

QUIC\_INVALID\_CRYPTO\_MESSAGE\_TYPE: 接收到一个非法标签的加密信息。  

QUIC\_SEQUENCE\_NUMBER\_LIMIT\_REACHED: 一个附加数据可能导致包号重用。  


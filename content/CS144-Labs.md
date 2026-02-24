# CS144-Labs

## Check0
### 访问一个网页

学习一下 [[telnet]] 的使用，来了解 http 的工作逻辑。首先，`telnet cs144.keithw.org http` 用于和服务器建立 http 连接。连接上了以后，就可以向其传输信息了，依次发送：

```http
GET /hello HTTP/1.1
Host: cs144.keithw.org
Connection: close
<回车>
```

就可以接收到返回的信息了。telnet 的作用就是将上面的报文转换为流发送出去，然后将接收到的报文再打印出来。

再底层一些，telnet 做的就是将每一行缓存，当敲击回车时，就将一行的内容发送出去（TCP缓冲区）->对方接收到这一行的内容。如果是合法的请求头，就会继续等待，否则就会报错。

> 为什么已经建立了 http 连接，还是需要指定 Host？
> 这样才能实现一个 ip & 多个域名

### 邮箱通讯和 nc-telnet 通讯

邮箱通讯用的是 smtp 协议，由于无法连接所以跳过。

其实原理就是 TCP 协议套的一层壳。nc-telnet 建立信道后，一方发送消息另一方也能接收到，smtp 就是在接受到消息的时候加了一些 if-else，用于判断是否符合协议。

```bash
nc -l -p <port>
telnet localhost <port>
```

### 编写 webget

目标是用操作系统提供的 socket 来实现一个网络程序，用于访问网页。在开始之前要简单了解一下现代 C++ 规范：
- 尽可能不要用 molloc/free、new/delete 等成对出现的操作
- 字符串的处理不要用 char* s，改用 stf::string
- 方法、变量是否 const 要严格

在 `util/socket.hh` 和 `util/file_descriptor.hh` 中，已经封装好了一些工具：

| 动作 | C 语言 (xv6 / POSIX) | CS144 C++ (Minnow) |
| :--- | :--- | :--- |
| **创建/连接** | `socket()`, `connect()` | `TCPSocket sock; sock.connect(Address(host, service));` |
| **发送数据** | `write(fd, buf, len)` | `sock.write(string_view data);` |
| **结束发送** | `shutdown(fd, SHUT_WR)` | **`sock.shutdown(SHUT_WR);`** (至关重要) |
| **接收数据** | `read(fd, buf, len)` | `sock.read()` -> 返回 `string` |
| **检查结束** | `res == 0` | `sock.eof()` |

封装了这么多，其实要做的就比较简单了：
- 建立连接
- 向 socket 当中写数据
- 关闭写口，等待回信

```cpp
using namespace std;

namespace {
void get_URL( const string& host, const string& path )
{
  TCPSocket socket;
  socket.connect( Address( host, "http" ) );
  string request = "GET " + path + " HTTP/1.1\r\n" + "Host: " + host + "\r\n" + "Connection: close\r\n" + "\r\n";
  socket.write( request );    // 发送
  socket.shutdown( SHUT_WR ); // 告诉服务器关闭了写端
  while ( !socket.eof() ) {
    string response;
    socket.read( response );
    cout << response;
  }
  socket.close(); // 关闭连接，释放资源
}
}
```

### 实现 ByteStream

要求实现一个读写的 buffer 流，主要是用于学习、适应 C++，不会很难。

首先要看原有的代码结构，已经定义好了 ByteStream：

```cpp
class ByteStream
{
public:
  explicit ByteStream( uint64_t capacity );

  // Helper functions (provided) to access the ByteStream's Reader and Writer interfaces
  Reader& reader();
  const Reader& reader() const;
  Writer& writer();
  const Writer& writer() const;

  void set_error() { error_ = true; };       // Signal that the stream suffered an error.
  bool has_error() const { return error_; }; // Has the stream had an error?

protected:
  // Please add any additional state to the ByteStream here, and not to the Writer and Reader interfaces.
  uint64_t capacity_;
  bool error_ {};
  std::string buffer_ {}; // 用 string 来替代 deque
  bool closed_ = false;   // 是否写完
  uint64_t total_pushed_ = 0;
  uint64_t bytes_popped_ = 0;
  bool is_finished_ = false;
};
```

要实现的就是 Reader 和 Writer 两个对象当中的方法。原先的思路是用一个环形缓冲区来实现，通过数组 + 双指针 + 取余。但是 `Reader.peek()` 返回的是一个 string_view 类型，也就是连续的一块内存，如果是环形的话会比较麻烦，遂排除。

然后的想法是，buffer 通过 deque<char> 来实现，但是写道这里的时候还是卡住了：

```cpp
string_view Reader::peek() const {
  std::string str( buffer_.begin(), buffer_.end() ); // 1. 创建了一个局部变量 str
  std::string_view view( str );                      // 2. view 指向了 str
  return view;                                       // 3. 函数结束，str 被销毁，view 变成了悬垂指针
}
```

这里涉及的问题是对象-指针的作用域。没办法，只能用 string 了。

其他的内容就没什么特别的了，主要是边界条件的检查，在 pop、peek 之类的操作前要看容量；还有 ByteStream 的定义问题，一开始把本应在 .h 文件中初始化为 0 的数值也暴露给外界的初始化函数了……应该只能暴露 capacity。

## Check1
### 接收、发送 datagrams

发送直接用 `ping 8.8.8.8` 来代替实验里的内网环境，结果：

```bash
~/P/c/minnow (main|↑3|✔) $ ping 8.8.8.8
PING 8.8.8.8 (8.8.8.8) 56(84) 字节的数据。
64 字节，来自 8.8.8.8: icmp_seq=1 ttl=111 时间=51.3 毫秒
64 字节，来自 8.8.8.8: icmp_seq=2 ttl=111 时间=51.3 毫秒
64 字节，来自 8.8.8.8: icmp_seq=3 ttl=111 时间=50.4 毫秒
64 字节，来自 8.8.8.8: icmp_seq=4 ttl=111 时间=50.8 毫秒
^C
--- 8.8.8.8 ping 统计 ---
已发送 4 个包， 已接收 4 个包, 0% packet loss, time 3004ms
rtt min/avg/max/mdev = 50.402/50.955/51.341/0.384 ms
```

其中：
- rtt（round trip time） 表示发包到接受到回包的时间
- ttl （time to live） = 默认 ttl - 路由器数

然后用 wireshark 来抓包分析。

选定接口后，选择协议为 icmp（ping 的协议），然后再发送 ping，可以看到有一些数据包的收发，点击查看后有如下信息：
- Frame: 物理层信息。
- Ethernet II: 链路层，网卡的 MAC 地址。
- Internet Protocol Version 4: 这是要对齐 RFC 791 的地方！点开它，会看到：
  - Total Length: 包的大小。
  - Identification: 这个包的 ID（如果包被拆分了，重组器靠这个识别）。
  - Time to Live: 发送时的初始值。
- ICMP: 协议内容。

然后，可以用 `ping -s 2000 8.8.8.8` 来发送一个大包，来观察分片，在 IP 层可以看到 `Fragment Offset`，同时可以看到发送的大包被拆分开了。这也就是后面实验的内容，要怎么将被拆分开的包组合接收。

### 手动发送 datagram-proto5

首先是用一个自定义的协议 proto5 来进行通信。其实实现起来也比较容易，用的是已经有的库，负责组装就可以了，难点在于要读一点源码，不然 AI 给的也是错的。因为没有网络条件，所以就在本地回环做了。

大概的组成就是 header+payload，用的是 IPv4Header 这个对象。但是要注意，对象是无法发送的，所以要进行序列化。问题在于序列化、发送这边的函数、继承逻辑有一点绕。

```cpp
int main()
{
  // construct an Internet or user datagram here, and send using a RawSocket
  try {
    IPv4Header ip_hdr;
    ip_hdr.src = Address( "127.0.0.1" ).ipv4_numeric();
    ip_hdr.dst = Address( "127.0.0.1" ).ipv4_numeric();
    ip_hdr.proto = 5;
    string payload = "This is proto 5";
    ip_hdr.len = payload.size() + static_cast<size_t>( ip_hdr.hlen * 4 );
    ip_hdr.compute_checksum();
    // 序列化为字节字符串
    Serializer serializer;
    ip_hdr.serialize( serializer );
    auto serialized_parts = serializer.finish();
    string packet;
    for ( const auto& part : serialized_parts ) {
      packet += part.get();
    }
    packet += payload;

    // 创建套接字
    RawSocket sock;
    sock.send( packet, Address( "127.0.0.1" ) );
    cout << "Datagram sent successfully (Protocol 5).\n";

  } catch ( const exception& e ) {
    cerr << "Error: " << e.what() << "\n";
    return EXIT_FAILURE;
  }
  return EXIT_SUCCESS;
}
```

编译过后，用 `sudo tcpdump -i lo proto 5 -vv -X` 监听，lo 表示本地回环，-X 表示 16 进制和 ASCII 码同时输出。然后运行程序发送，可以得到：

```bash
~ $ sudo tcpdump -i lo proto 5 -vv -X
tcpdump: listening on lo, link-type EN10MB (Ethernet), snapshot length 262144 bytes
12:29:47.425422 IP (tos 0x0, ttl 128, id 0, offset 0, flags [DF], proto unknown (5), length 35)
    localhost > localhost:  st 15
	0x0000:  4500 0023 0000 4000 8005 fcd3 7f00 0001  E..#..@.........
	0x0010:  7f00 0001 5468 6973 2069 7320 7072 6f74  ....This.is.prot
	0x0020:  6f20 35                                  o.5
```

### 手动发送 UDP

UDP 也是类似的，只是协议的不同，导致 payload 的格式稍有变化，要加上端口号。UDP 头部没有给现成的库，所以直接压入序列化后的字符串即可。

大致的格式是：IP header + UDP header + payload。

要注意的就是序列化字符的处理了：

```cpp
int main()
{
  try {
    string payload = "Hello, UDP";
    uint16_t udp_len = 8 + payload.size();
    uint16_t src_port = 12345;
    uint16_t dst_port = 9999;
    // 构建 ip 头部
    IPv4Header ip_hdr;
    ip_hdr.dst = Address( "127.0.0.1" ).ipv4_numeric();
    ip_hdr.src = Address( "127.0.0.1" ).ipv4_numeric();
    ip_hdr.proto = 17;
    ip_hdr.len = 20 + udp_len;
    ip_hdr.compute_checksum();
    // 序列化所有内容
    Serializer serializer;
    ip_hdr.serialize( serializer );
    serializer.integer( src_port );
    serializer.integer( dst_port );
    serializer.integer( udp_len );
    serializer.integer<uint16_t>( 0 );
    // 拼接最终明文
    auto parts = serializer.finish();
    string packet;
    for ( const auto& part : parts ) {
      packet += part.get();
    }
    packet += payload;
    RawSocket socket;
    socket.send( packet, Address( "127.0.0.1" ) );
    cout << "UDP send Success \n";
  } catch ( const exception& e ) {
    cerr << e.what() << "\n";
  }
}
```

检测：

```bash
~/P/c/minnow (main|↑3|✚1) $ sudo tcpdump -i lo udp port 9999 -vv -X
tcpdump: listening on lo, link-type EN10MB (Ethernet), snapshot length 262144 bytes
15:45:28.846466 IP (tos 0x0, ttl 128, id 0, offset 0, flags [DF], proto UDP (17), length 38)
    localhost.italk > localhost.distinct: [no cksum] UDP, length 10
	0x0000:  4500 0026 0000 4000 8011 fcc4 7f00 0001  E..&..@.........
	0x0010:  7f00 0001 3039 270f 0012 0000 4865 6c6c  ....09'.....Hell
	0x0020:  6f2c 2055 4450                           o,.UDP
```

### 实现 Reassembler

上面的实验是观察各个零散的包，这里就要求处理这些零散包了。思路还是比较简单的，大致就是按照各个包的 index 顺序来把它们输入到 ByteStream 当中。

但是到了具体场景中，就比较复杂了，要考虑的有很多。首先，每个包可能会重放、可能包的范围会重叠、可能包会超出容纳范围……所以这里的处理逻辑得要严密一些。

理论方面，重要的就是 map 这个数据结构。map 的底层是用红黑树来实现的，用 index 进行排序，然后具体的 data 可以通过 index 取到。搜索的时间是 O(logN) 的，很稳定。然后可以通过 lowerbound、upperbound 快速地取得前一个、后一个元素，从而方便地合并字段。由于其中存储的是字符串，所以又可以用 string 的库函数来进行处理，比如 substring/erase/+ 等等.

具体来说，就是各个边界条件的处理比较麻烦，要考虑 first_index ~ data.size() 的种种情况，回想起来，其实画个图会好一些。

```cpp
#include "reassembler.hh"

using namespace std;

void Reassembler::insert( uint64_t first_index, string data, bool is_last_substring )
{
  if ( is_last_substring ) {
    is_eof_ = true;
    last_index_ = first_index + data.size();
  }
  // 处理右边界,如果 first_index 在接受区外，直接舍弃
  uint64_t end = first_index_ + output_.writer().available_capacity();
  if ( first_index > end ) {
    return;
  }
  if ( first_index + data.size() > end ) {
    data.resize( end - first_index );
  }

  // 处理左边界,如果 first_index 的末尾在接受区外，舍弃
  if ( first_index + data.size() <= first_index_ ) {
    if ( is_eof_ && first_index_ == last_index_ ) {
      output_.writer().close();
    }
    return;
  }
  if ( first_index < first_index_ ) {
    uint64_t cut_len = first_index_ - first_index;
    data.erase( 0, cut_len );
    first_index = first_index_;
  }
  // 处理堆叠
  auto it = buffer_.lower_bound( first_index ); // 找第一个 >= first_index 的作为起点
  // 尝试向左看
  if ( it != buffer_.begin() ) {
    auto prev_it = prev( it );
    // 前面的合并后面的头
    if ( prev_it->first + prev_it->second.size() >= first_index ) {
      if ( prev_it->first + prev_it->second.size() >= first_index + data.size() ) {
        return;
      }
      data = prev_it->second + data.substr( prev_it->first + prev_it->second.size() - first_index );
      first_index = prev_it->first;
      buffer_.erase( prev_it ); // 已经合并到 data，删除 prev_it
    }
  }
  // 尝试向右看
  it = buffer_.lower_bound( first_index );
  while ( first_index + data.size() >= it->first && it != buffer_.end() ) {
    uint64_t next_end = it->first + it->second.size();
    if ( first_index + data.size() < next_end ) {
      // 结尾盖住了后面的头
      data += it->second.substr( first_index + data.size() - it->first );
    }
    it = buffer_.erase( it ); // 删掉被合并的旧块
  }
  buffer_[first_index] = std::move( data );
  while ( !buffer_.empty() && buffer_.begin()->first == first_index_ ) {
    it = buffer_.begin();
    output_.writer().push( it->second );
    first_index_ += it->second.size();
    buffer_.erase( it );
  }
  if ( is_eof_ && first_index_ == last_index_ ) {
    output_.writer().close();
  }
}

// How many bytes are stored in the Reassembler itself?
// This function is for testing only; don't add extra state to support it.
uint64_t Reassembler::count_bytes_pending() const
{
  uint64_t total = 0;
  for ( const auto& kv : buffer_ ) {
    total += kv.second.size();
  }
  return total;
}
```

写的时候出现了几个错误：
- data、first_index 应该先保持不变，到最后再来修改，处理要统一
- `is_eof_` 和 `is_last_substring` 的意义要分清，而且读取到最后一个块并不能表示结束，只有 `first_index_ == last_index_` 时，才算读取完毕。

## Check2
### 实现 wrap\unwrap

目的是实现 Wrap 对象的两个函数。实现 32/64 位序列的转换。

原理方面，主要是要理解 32 位序列号生成的原理。序列号用于表示数据的位置，因为发送时是一个字节一个字节发的，为了整合乱序的数据，所以要给每一个数据都进行标号。如果用 64 位来标号，会导致损耗过大。而某一时段传输的数据往往不会很大量，所以可以用相对式的 32 位来进行标记。

由于序列号被压缩了，所以就需要特殊的转化，而这个转化建立在一个假设之上：瞬时处理的相对序列号对应的绝对序列号是相近的。在进行具体计算以前，有几个概念要分清：
- check_point：上一次的绝对序列号对应位置，用于确认这一次的相对位置
- Wrap32 对象，只存储 raw_value 绝对序列号
- wrap：用于将绝对序列号收缩为 32 位
- unwrap：反过来
- zero_point：指示 ISN（随机生成的序列号，用于指示开头）

64 位转换为 32 位比较简单，只是会丢失掉部分信息。再用 `uint_32` 来进行存储，就可以再省去一步取模操作。

```cpp
Wrap32 Wrap32::wrap( uint64_t n, Wrap32 zero_point )
{
  // 把 64 位的降维为 32 位的
  // 只需直接 n = (N + ISN) % 2^32 即可
  // zero_point: ISN，随机偏移量
  // n：绝对序列号
  // Wrap32：加上了偏移后的 32 位相对序列号
  return Wrap32( static_cast<uint32_t>( n ) + zero_point.raw_value_ );
}
```

而 32 位转 64 位就会麻烦一些。传入的参数是 `zero_point` 标记和 checkpoint 上次确定的位置。Wrap32 对象通过这个函数，将 `raw_value` 转换为 64 位。大概的思路是，首先将 checkpoint 也映射到 32 位，理论上由于距离相近，所以他们的位置差只会是 +-1 个周期。然后 checkpoint 加上相对位置，就可以得到绝对位置了。

```cpp
uint64_t Wrap32::unwrap( Wrap32 zero_point, uint64_t checkpoint ) const
{
  // 把 32 位的 wrap 升维成 64 位的
  // 已知 n/ISN，要还原出 n
  // checkpoint 最近一次成功的标志点
  // 计算周期差
  // 假设目标就在 checkpoint 附近！才能定位
  // 算出 checkpoint 映射的位置
  Wrap32 wrap_checkpoint = wrap( checkpoint, zero_point );
  int32_t min_len = this->raw_value_ - wrap_checkpoint.raw_value_;
  uint64_t result = checkpoint + min_len;
  if ( min_len < 0 && result > checkpoint ) {
    result = result + ( 1ULL << 32 );
  }
  return result;
}
```

### 实现 TCPReceiver

Receiver 是前面实现部分的更高层枢纽，用于将接收到的数据报处理、分发给 reassembler，并返回一定的信息给发送方。

写到这里需要对前面所做的有一个更全面的理解了，不然逻辑会转不过来。首先是接收的包到底有什么？看一下定义：

```cpp
struct TCPSenderMessage
{
  Wrap32 seqno { 0 };

  bool SYN {};
  std::string payload {};
  bool FIN {};

  bool RST {};

  // How many sequence numbers does this segment use?
  size_t sequence_length() const { return SYN + payload.size() + FIN; }
};
```

SYN 是起始包的标志，FIN 是结束包的标志，而 RST 是错误标志。那么收到这样一条信息之后，我们要做的就是通过标志位来判断，然后把包进行基本的解析以后再交给下一阶段的工具处理。

```cpp
void TCPReceiver::receive( TCPSenderMessage message )
{
  // debug，先判断是否有错误
  if ( message.RST ) {
    reassembler_.reader().set_error();
  }
  // 判断 SYN
  if ( !isn_.has_value() ) {
    if ( !message.SYN ) {
      return;
    }
    isn_ = message.seqno;
  }
  // 利用已经锁定的 ISN 进行解析
  uint64_t checkpoint = reassembler_.writer().bytes_pushed(); // 确定上一次的位置
  uint64_t abs_no = message.seqno.unwrap( isn_.value(), checkpoint );
  // 剥离协议头，SYN 是第 0 位
  uint64_t first_index = message.SYN ? 0 : abs_no - 1;
  // 交付重组
  reassembler_.insert( first_index, message.payload, message.FIN );
}
```

处理完收到的包以后，还要发送一些信息给发送方，以确定收包情况：

```cpp
TCPReceiverMessage TCPReceiver::send() const
{
  TCPReceiverMessage message;
  // 确认是否有错误
  if ( reassembler_.writer().has_error() ) {
    message.RST = true;
  }
  // 告知剩余长度
  message.window_size
    = reassembler_.writer().available_capacity() > 65535 ? 65535 : reassembler_.writer().available_capacity();
  // 确认是否受到 SYN，收到才发 ackno
  if ( isn_.has_value() ) {
    // 绝对位置
    uint64_t abs_no = reassembler_.writer().bytes_pushed() + 1;
    // 如果流结束，还要再加一位
    if ( reassembler_.writer().is_closed() ) {
      abs_no += 1;
    }
    message.ackno = Wrap32::wrap( abs_no, isn_.value() );
  }
  return message;
}
```

整体而言思路不是很难，但是各种概念有些绕。下面是一些辅助理解的图：

```
[ 网络波动层 ] ----> [ 协议控制层: TCPReceiver ] ----> [ 逻辑重组层: Reassembler ] ----> [ 数据交付层: ByteStream ]
      |                    |                         |                          |
      | 接收: seqno, payload | 1. 维护 ISN (zero_point)  | 1. 处理乱序 (Out-of-order) | 1. 顺序存储字节流
      | 标志: SYN, FIN, RST  | 2. 坐标转换 (32b -> 64b) | 2. 消除重叠 (Overlapping)  | 2. 提供给上层应用读取
      |                    | 3. 计算 ACK & Window     | 3. 判断流是否结束 (FIN)     | 3. 标记系统错误 (RST)
      +--------------------+-------------------------+--------------------------+-----------------------+
```

```
Seqno (32-bit, Wrapping): 网络上传输的“代号”。为了节省带宽，它只有 32 位，会溢出回绕。
Absolute Seqno (64-bit): 系统内部的“真值”。从 0 开始，永不回绕。
Stream Index (64-bit): Reassembler 看到的“净负载下标”。不包含 SYN。
```

TCPReceiver 只管协议头，Reassembler 只管拼图，ByteStream 只管存钱。

再提升一点，可以看见在不确定环境下确定交互的方式。通过头部来确定、同步、更新当前状态，用于保证协议确定性的是多次的发送-接收。

## Check3
### 实现 TCP Sender

写了一天……终于所有用例都过了。整体来说复杂了很多，不仅是逻辑上的，还有架构上的理解难度。

在写之前，要先看看 TCPSender 这个类是做什么的：

```cpp
class TCPSender
{
public:
  /*构造函数，给定 isn、initial_RTO*/
  TCPSender( ByteStream&& input, Wrap32 isn, uint64_t initial_RTO_ms )
    : input_( std::move( input ) ), isn_( isn ), timer_( initial_RTO_ms ), initial_RTO_ms_( initial_RTO_ms )
  {}

  /* Generate an empty TCPSenderMessage */
  TCPSenderMessage make_empty_message() const;

  /*接收返回的信息*/
  void receive( const TCPReceiverMessage& msg );

  /* Type of the `transmit` function that the push and tick methods can use to send messages */
  using TransmitFunction = std::function<void( const TCPSenderMessage& )>;

  /* Push bytes from the outbound stream */
  void push( const TransmitFunction& transmit );

  /* Time has passed by the given of milliseconds since the last time the tick() method was called */
  void tick( uint64_t ms_since_last_tick, const TransmitFunction& transmit );

  // Accessors
  uint64_t sequence_numbers_in_flight() const;  // How many sequence numbers are outstanding?
  uint64_t consecutive_retransmissions() const; // How many consecutive retransmissions have happened?
  const Writer& writer() const { return input_.writer(); }
  const Reader& reader() const { return input_.reader(); }
  Writer& writer() { return input_.writer(); }
  void set_error();

private:
  Reader& reader() { return input_.reader(); }
  ByteStream input_;
  Wrap32 isn_; // 起始偏移量，一开始就会被初始化
  Timer timer_;

  uint64_t initial_RTO_ms_; // 初始重传时间

  std::deque<TCPSenderMessage> outstanding_segment_ {}; // 未被确认的数据块
  uint64_t bytes_in_flight_ = 0;                        // 待确认数据块大小
  uint64_t consecutive_retransmissions_ = 0;            // 重放次数
  uint64_t window_size_ = 1;                            // 空闲窗口大小
  // 发送标志位
  bool syn_sent_ = false;
  bool fin_sent_ = false;
  bool is_error_ = false;
  uint64_t next_seqno_ = 0; // 动态增长的绝对序列号，也是 checkpoint 基准
};
```

可以看到，Sender 是拿来发送数据的，是 Stream 上的再一层抽象，用来完善各种特殊情况，并且提供超时重传逻辑。而超时重传的逻辑大致是：
- 发送包，并且将其加入到已发送队列
- 开始计时，在指定时间内如果没有接收到确认回包，则指数惩罚-重传
- 如果接收到回包，则刷新计数，更新队列，继续发包

完成的关键在于，要先对每个组件有比较确切的认识，包括：
- Timer：作为 Sender 的内部工具类，提供计时服务
- tick：调用 Timer，给 Timer 传入时间，并判断时序-状态的变换
- push：尝试发包，需要构造 payload、标志位
- receive：接收对方发来的消息，以便调整发送的逻辑

实现的时候，从 Timer 开始实现。对于一个计时器来说，需要有什么功能呢？其实就是它在各个状态间的流转，而 “时间” 这个概念本来是用硬件来计时的，但是在这里是在测试用例中注入的。具体来说，Timer 要支持 `start,reset,restart,double,stop` 操作；以及查询状态的 `is_running，is_expired`。这个工具类要怎么配合 Sender 呢？其实直接作为其内部类即可。

```cpp
class Timer
{
  // tick 可以调用的"定时器"
  // 用于判断时间
public:
  explicit Timer( uint64_t time_limit ) : time_limit_( time_limit ), time_current_( time_limit ) {}
  void start()
  {
    is_running_ = true;
    time_passed_ = 0;
    time_current_ = time_limit_;
  }
  bool is_running() const { return is_running_; }
  void stop()
  {
    is_running_ = false;
    time_passed_ = 0;
    time_current_ = time_limit_;
  }
  void dou()
  {
    time_current_ *= 2;
    time_passed_ = 0;
  }
  void reset()
  {
    time_passed_ = 0;
    time_current_ = time_limit_;
  }
  void restart() { time_passed_ = 0; }
  bool is_expired() const { return ( time_current_ <= time_passed_ ) && is_running_; }
  void tick( uint64_t passed )
  {
    if ( is_running_ ) {
      time_passed_ += passed;
    }
  }

private:
  bool is_running_ = false;
  uint64_t time_limit_;
  uint64_t time_current_;
  uint64_t time_passed_ = 0;
};
```

要注意的是区分概念。`time_limit` 是初始值，不应被改变，也是翻倍的基准。而 `time_current` 则是当前需要等待的时间，在不断翻倍时，通过 `consecutive_retransmissions_` 来确定重放次数、舍弃。

实现完了 Timer 以后，就可以进到具体的 tick 逻辑了。每一个时间经过以后，要检查是否需要重传、如果重传了，就要 double 惩罚。

```cpp
void TCPSender::tick( uint64_t ms_since_last_tick, const TransmitFunction& transmit )
{
  timer_.tick( ms_since_last_tick );
  // 用于记录时序状态，并作出对应变化
  // 判断重传
  if ( timer_.is_expired() ) {
    // 如果是已发送队列为空，无需重传
    if ( outstanding_segment_.empty() ) {
      timer_.stop();
      return;
    }
    // 重传最早的包
    transmit( outstanding_segment_.front() );
    // 如果对方 windows > 0，说明是网络堵塞，需要指数退避
    if ( window_size_ != 0 ) {
      consecutive_retransmissions_++;
      timer_.dou();
    }
    // 更新以后，重启计时器
    timer_.restart();
  }
}
```

这边主要是 restart 和 reset 的区分，要分清是否需要重置 `time_current` 限制。

有了时序逻辑以后，就是发包、收包了。发包要考虑的是对方的容量，从 Stream 当中不断读出 payload，并且完善标志位，再发送出去即可。主要的问题是标志位的判断，一开始没有自己添加上表示标志位的状态量，导致无从下手，其实直接在类中添加 `isn_、syn_sent、fin_sent` 等即可，然后在第一次发送、最后一次发送的时候修改就行了。

```cpp
void TCPSender::push( const TransmitFunction& transmit )
{
  // 按照 windows 大小来发送
  uint64_t cur_windows = window_size_ == 0 ? 1 : window_size_;
  // 只要有剩余空间，就不断造包发送
  while ( cur_windows > bytes_in_flight_ ) {
    TCPSenderMessage msg;
    if ( !syn_sent_ ) {
      syn_sent_ = true;
      msg.SYN = true;
    }
    if ( is_error_ || input_.has_error() ) {
      msg.RST = true;
    }
    uint64_t window_left = cur_windows - bytes_in_flight_;
    uint64_t len = std::min( { static_cast<uint64_t>( TCPConfig::MAX_PAYLOAD_SIZE ),
                               window_left - msg.sequence_length(),
                               input_.reader().bytes_buffered() } );
    auto payload = input_.reader().peek();
    msg.payload = string( payload.substr( 0, len ) );
    input_.reader().pop( len );
    // 处理 FIN
    if ( !fin_sent_ && input_.reader().is_finished() && ( msg.sequence_length() + 1 <= window_left ) ) {
      fin_sent_ = true;
      msg.FIN = true;
    }

    // 空包，直接退出
    if ( msg.sequence_length() == 0 ) {
      break;
    }
    // 发送
    msg.seqno = Wrap32::wrap( next_seqno_, isn_ );
    transmit( msg );
    // 记录
    outstanding_segment_.push_back( msg );
    bytes_in_flight_ += msg.sequence_length();
    next_seqno_ += msg.sequence_length();
    if ( !timer_.is_running() ) {
      timer_.start();
    }
    if ( msg.FIN ) {
      break;
    }
  }
}
```

这里的问题主要是判断窗口大小来添加 FIN 标志位。FIN 应当被看作一个 EOF 事件，而不是被看作一个 bool 位，所以在 `FIN=true` 以后，发生的效果是在 payload 后添加了一个 EOF 标志，而不是单纯地把 false 变为 true。

最后是接收，解析对方穿过来的 windowsize、标志位等信息，并且判断是否有已经被接收的包，然后从列表中弹出即可。

```cpp
void TCPSender::receive( const TCPReceiverMessage& msg )
{
  // 错误，取消发送
  if ( msg.RST ) {
    timer_.stop();
    is_error_ = true;
    input_.reader().set_error();
  }
  window_size_ = msg.window_size;
  if ( !msg.ackno.has_value() ) {
    return;
  }
  uint64_t abs_no = msg.ackno->unwrap( isn_, next_seqno_ );
  // 判断 ackno 是否合法，更新状态
  if ( abs_no > next_seqno_ ) {
    return;
  }
  // 合法，更新
  bool is_change = false;
  while ( !outstanding_segment_.empty() ) {
    const auto& first_msg = outstanding_segment_.front();
    uint64_t first_seqno = first_msg.seqno.unwrap( isn_, next_seqno_ );
    if ( first_seqno + first_msg.sequence_length() <= abs_no ) {
      // 第一个包完整收到了
      bytes_in_flight_ -= first_msg.sequence_length();
      outstanding_segment_.pop_front();
      is_change = true;
    } else {
      break;
    }
  }
  // 如果有被接收的包，需要重新开始计时再重传
  if ( is_change ) {
    consecutive_retransmissions_ = 0;
    if ( !outstanding_segment_.empty() ) {
      timer_.reset();
    } else {
      // 如果没有需要重传的包，停止计时
      timer_.stop();
    }
  }
}
```

还剩下一个 emptymessage：

```cpp
TCPSenderMessage TCPSender::make_empty_message() const
{
  TCPSenderMessage msg;
  msg.seqno = Wrap32::wrap( next_seqno_, isn_ );
  if ( is_error_ || input_.reader().has_error() ) {
    msg.RST = true;
  }
  return msg;
}
```

写完了以后，感觉主要的难点在理解各个状态间的流转，以及对各个组件的认识。在写之前就要想清楚需要记录什么状态、它们在什么时候会怎么被改变。然后就是实现的细节也很多，标志位的判断（尤其是 has-error），都是需要考虑到的。

## Check4
### 观察网络链路

通过不断地 ping 来抓取数据包的发送情况（用 traceroute 来看每一跳的具体情况）。然后可以通过分析 ttl、时间 来判断网络连接的变化情况（通过数据分析）

具体没有操作，大概了解一下网络拥堵、潮汐，以及实际情况中网络包的变动性。

## Check5
### 完成链路层 NetworkInterface

整体来说难度还是有的，但是没有 check3 那么迷了。前面花了比较多的时间来看，会更清楚自己应该做什么、怎么做。

大致的原理是，链路层是用来实现网络间传输的链接的。主要的区分是 MAC 和 IP，IP 是可以变化的，那么也就导致了发送信息的时候不可能只对着特定 IP 发送，而 MAC 是每个机器上的唯一标识，所以需要有一个 IP-MAC 的转换。

但是问题在于，既然不知道对方的特定地址，要如何知道对方的 MAC 呢？于是引入了 ARP 广播协议。发送方通过广播给范围内的所有设备，来询问 "谁的 IP 地址是 xxx？"，然后拥有指定地址的接收方回复自己的 MAC 地址，就可以告知发送方了。

具体实现要考虑的更多一点，在接收到对方传回的 MAC 以后，需要进行一段时间的缓存（这里是 30s）；而在接收到对方传回的 MAC 以前，如果还有包要发送/询问，则要进行一段时间的缓冲（这里是 5s），否则不断询问会形成链路风暴；由于未知 MAC 而暂时无法发送的包要进行存储，也需要一定的处理逻辑来控制存储的合理性。

从定义类开始，需要添加时间戳来控制时间间隔，以确定缓冲时间和清理过期的包；还需要有 map 来存储对应 IP 地址的相应信息，包括等待队列和等待的时间，以及对应 IP 地址缓冲的包。

```cpp
class NetworkInterface
{
public:
  // An abstraction for the physical output port where the NetworkInterface sends Ethernet frames
  class OutputPort
  {
  public:
    virtual void transmit( const NetworkInterface& sender, const EthernetFrame& frame ) = 0;
    virtual ~OutputPort() = default;
  };

  // Construct a network interface with given Ethernet (network-access-layer) and IP (internet-layer)
  // addresses
  NetworkInterface( std::string_view name,
                    std::shared_ptr<OutputPort> port,
                    const EthernetAddress& ethernet_address,
                    const Address& ip_address );

  // Sends an Internet datagram, encapsulated in an Ethernet frame (if it knows the Ethernet destination
  // address). Will need to use [ARP](\ref rfc::rfc826) to look up the Ethernet destination address for the next
  // hop. Sending is accomplished by calling `transmit()` (a member variable) on the frame.
  void send_datagram( InternetDatagram dgram, const Address& next_hop );

  // Receives an Ethernet frame and responds appropriately.
  // If type is IPv4, pushes the datagram to the datagrams_in queue.
  // If type is ARP request, learn a mapping from the "sender" fields, and send an ARP reply.
  // If type is ARP reply, learn a mapping from the "sender" fields.
  void recv_frame( EthernetFrame frame );

  // Called periodically when time elapses
  void tick( size_t ms_since_last_tick );
  // 新增，包装 ARP 构造函数
  void ARP_transmit( uint32_t target_ip );
  void transmit_ipv4( const InternetDatagram& dgram, const EthernetAddress& dst_mac );
  void flush_pending_datagrams( uint32_t ip, const EthernetAddress& mac );

  // Accessors
  const std::string& name() const { return name_; }
  const OutputPort& output() const { return *port_; }
  OutputPort& output() { return *port_; }
  std::queue<InternetDatagram>& datagrams_received() { return datagrams_received_; }

private:
  // Human-readable name of the interface
  std::string name_;

  // The physical output port (+ a helper function `transmit` that uses it to send an Ethernet frame)
  std::shared_ptr<OutputPort> port_;
  void transmit( const EthernetFrame& frame ) const { port_->transmit( *this, frame ); }

  // Ethernet (known as hardware, network-access-layer, or link-layer) address of the interface
  EthernetAddress ethernet_address_;

  // IP (known as internet-layer or network-layer) address of the interface
  Address ip_address_;

  // Datagrams that have been received
  std::queue<InternetDatagram> datagrams_received_ {};

  // 新增
  struct ARPEntry
  {
    EthernetAddress mac {};
    size_t time_stamp { 0 };

    ARPEntry() = default; // 保留默认构造（map 内部需要）
    ARPEntry( EthernetAddress m, size_t t ) : mac( m ), time_stamp( t ) {}
  };

  struct PendingQueue
  {
    std::vector<InternetDatagram> datagrams;
    std::optional<size_t> last_request_time_;
    PendingQueue()
      : datagrams()                        // 显式构造空向量
      , last_request_time_( std::nullopt ) // 显式置空
    {}
  };
  size_t current_time_ms_ = { 0 };
  static constexpr size_t ARP_TIMEOUT_MS = 30000;
  static constexpr size_t ARP_RETRY_INTERVAL_MS = 5000;
  std::map<uint32_t, ARPEntry> arp_table_;   // ARP 缓存表，ip -> MAC+过期时间
  std::map<uint32_t, PendingQueue> pending_; // 待处理队列，ip -> 等待该 ip 解析的数据包队列
};
```

然后就是要实现具体的方法了，先从处理时间的 tick 开始。主要的逻辑是：遍历 pending/arptable，确认其中是否有超过 ARP 缓存或 pending 等待时长的项，如果有就删掉。另外，还要考虑 ARP 广播的重发。

```cpp
void NetworkInterface::tick( const size_t ms_since_last_tick )
{
  current_time_ms_ += ms_since_last_tick;

  // 清理过期的 ARP 缓存
  for ( auto it = arp_table_.begin(); it != arp_table_.end(); ) {
    if ( current_time_ms_ - it->second.time_stamp > ARP_TIMEOUT_MS ) { // 30s
      it = arp_table_.erase( it );
    } else {
      ++it;
    }
  }

  // 清理过期的 Pending Entries
  // 这是系统的"断路器"
  for ( auto it = pending_.begin(); it != pending_.end(); ) {
    if ( current_time_ms_ - it->second.last_request_time_.value() > ARP_RETRY_INTERVAL_MS ) { // 5s
      // 丢弃所有待发送的数据报（Let them die）
      it = pending_.erase( it );
    } else {
      ++it;
    }
  }

  // 处理挂起的 ARP 请求
  for ( auto& [ip, entry] : pending_ ) {
    if ( current_time_ms_ - entry.last_request_time_.value() > ARP_RETRY_INTERVAL_MS ) { // 5s
      // 达到冷却时间，重发
      // 构造 ARP 请求帧
      ARP_transmit( ip );
      // 更新时间戳
      entry.last_request_time_ = current_time_ms_;
    }
  }
}
```

时间逻辑处理了以后，就要到具体的接收-发送了。对于发送而言，要考虑的是发送什么。如果是还没有登记的，就要进行 ARP 广播，而如果登记了的，就直接发送 IPV4 数据报。二者发送的都是 frame，区分的标志在于 frame.header 的 dst 和 type：ARP 的 dst 是 ff:...:ff，标志发送给全体。然后 ARP 的 payload 的区别是，ARP 要封装一个 message：

```cpp
struct ARPMessage
{
  static constexpr size_t LENGTH = 28;         // ARP message length in bytes
  static constexpr uint16_t TYPE_ETHERNET = 1; // ARP type for Ethernet/Wi-Fi as link-layer protocol
  static constexpr uint16_t OPCODE_REQUEST = 1;
  static constexpr uint16_t OPCODE_REPLY = 2;

  uint16_t hardware_type = TYPE_ETHERNET;             // Type of the link-layer protocol (generally Ethernet/Wi-Fi)
  uint16_t protocol_type = EthernetHeader::TYPE_IPv4; // Type of the Internet-layer protocol (generally IPv4)
  uint8_t hardware_address_size = sizeof( EthernetHeader::src );
  uint8_t protocol_address_size = sizeof( IPv4Header::src );
  uint16_t opcode {}; // Request or reply

  EthernetAddress sender_ethernet_address {};
  uint32_t sender_ip_address {};

  EthernetAddress target_ethernet_address {};
  uint32_t target_ip_address {};

  // Return a string containing the ARP message in human-readable format
  std::string to_string() const;

  // Is this type of ARP message supported by the parser?
  bool supported() const;

  void parse( Parser& parser );
  void serialize( Serializer& serializer ) const;
}
```

主要是区分发送/接收，以及把需要的信息发送出去。最后需要序列化才能够正常载入到 payload 中。一开始是硬编码写在发送函数里的，后续抽离出了两个函数来针对 ARP、IPV4 的发送：

```cpp
void NetworkInterface::ARP_transmit( uint32_t target_ip )
{
  EthernetFrame frame;
  frame.header.dst = ETHERNET_BROADCAST;
  frame.header.src = ethernet_address_;
  frame.header.type = EthernetHeader::TYPE_ARP;

  // 序列化 ARP 消息
  ARPMessage msg;
  msg.opcode = ARPMessage::OPCODE_REQUEST;
  msg.sender_ethernet_address = ethernet_address_;
  msg.sender_ip_address = ip_address_.ipv4_numeric();
  msg.target_ethernet_address = {};
  msg.target_ip_address = target_ip;

  Serializer serializer;
  msg.serialize( serializer );
  frame.payload = serializer.finish();

  // 发送广播
  transmit( frame );
}

void NetworkInterface::transmit_ipv4( const InternetDatagram& dgram, const EthernetAddress& dst_mac )
{
  EthernetFrame frame;
  frame.header.dst = dst_mac;                    // 目标
  frame.header.src = ethernet_address_;          // 源地址
  frame.header.type = EthernetHeader::TYPE_IPv4; // 帧类型

  Serializer serializer;
  dgram.serialize( serializer );
  frame.payload = serializer.finish(); // 序列化

  transmit( frame );
}
```

这边主要是序列化、反序列化有些不太清楚，绕了很久才写对。应该看一看函数原型再写的。

然后是具体的发送逻辑，先查表，如果表里没查到，就要发送 ARP；如果查到了，直接发送 IPV4 数据包。

```cpp
void NetworkInterface::send_datagram( InternetDatagram dgram, const Address& next_hop )
{
  const uint32_t next_hop_ip = next_hop.ipv4_numeric();

  // 先查缓存
  if ( auto it = arp_table_.find( next_hop_ip ); it != arp_table_.end() ) {
    // 立即封装并发送
    transmit_ipv4( dgram, it->second.mac );
    return;
  }

  // 没找到缓存，要先进入等待列表
  auto& entry = pending_[next_hop_ip];

  // 防御性编程：如果已经过期，先清空尸体
  if ( entry.last_request_time_.has_value()
       && current_time_ms_ - entry.last_request_time_.value() > ARP_RETRY_INTERVAL_MS ) {
    entry.datagrams.clear();
  }

  entry.datagrams.push_back( std::move( dgram ) );

  // 等待冷却期再发送查询请求
  // 检查是否需要发送ARP请求（冷却期控制）
  if ( entry.last_request_time_.has_value() ) {
    // 原先已经有在等待的了
    if ( current_time_ms_ - entry.last_request_time_.value() > ARP_RETRY_INTERVAL_MS ) {
      // 超过冷却期，重发
      ARP_transmit( next_hop_ip );
      entry.last_request_time_ = current_time_ms_;
    }
  } else {
    // 原先没有在等待的，直接重发
    ARP_transmit( next_hop_ip );
    entry.last_request_time_ = current_time_ms_;
  }
}
```

这里的问题是冷却期的逻辑要判断清楚，以及对已有列表的维护。

最后就是接收了。这里的接收是对上层的支撑，也就是说要判断返回的包的类型，如果是 ARP，就要再进一步判断是否是发给自己的、要怎么处理；如果是 IPV4，则要在判断以后交付给上层处理。

对于 ARP 而言，有一个优化是：如果对方发来的包是 REPLY（即告知源自己的 MAC），而目标不是自己，这个时候可以进行被动学习，即记录一个目前暂时无效的包到缓存表里。还有一个一开始疏漏的点是，如果包是 REQUEST（即告知源自己的 MAC，同时目标是自己），在发送 REPLY 告知自己网卡时，也可以直接把 pending 列表中的包给发过去！

所以最终的效果是，只要有新的条目进到 MAC 的缓存区，就要进行一次 flush-pending，所以抽离出一个函数：

```cpp
// 辅助函数：刷新等待队列，发送挂起的数据报
void NetworkInterface::flush_pending_datagrams( uint32_t ip, const EthernetAddress& mac )
{
  auto it = pending_.find( ip );
  if ( it == pending_.end() ) {
    return;
  }

  // 是回包，刷新等待队列
  auto& queue = it->second;
  for ( auto& dgram : queue.datagrams ) {
    transmit_ipv4( dgram, mac );
  }
  pending_.erase( it );
}
```

然后是具体的接收：

```cpp
void NetworkInterface::recv_frame( EthernetFrame frame )
{
  switch ( frame.header.type ) {
    case EthernetHeader::TYPE_ARP: {
      // 反序列化
      Parser parser( frame.payload );
      ARPMessage msg;
      msg.parse( parser );

      // 被动学习源 MAC
      arp_table_[msg.sender_ip_address] = { msg.sender_ethernet_address, current_time_ms_ };

      // 类型判断
      if ( msg.opcode == ARPMessage::OPCODE_REQUEST ) {
        // 如果访问的是本机，则发送 reply
        if ( msg.target_ip_address == ip_address_.ipv4_numeric() ) {
          EthernetFrame reply_frame;
          reply_frame.header.dst = msg.sender_ethernet_address; // 单播给请求者
          reply_frame.header.src = ethernet_address_;
          reply_frame.header.type = EthernetHeader::TYPE_ARP;

          ARPMessage reply;
          reply.opcode = ARPMessage::OPCODE_REPLY;
          reply.sender_ethernet_address = ethernet_address_;
          reply.sender_ip_address = ip_address_.ipv4_numeric();
          reply.target_ethernet_address = msg.sender_ethernet_address; // 填写对方MAC
          reply.target_ip_address = msg.sender_ip_address;             // 填写对方IP

          Serializer serializer;
          reply.serialize( serializer );
          reply_frame.payload = serializer.finish();
          transmit( reply_frame );
        }
        // 同时把包也返回（处理等待队列）
        flush_pending_datagrams( msg.sender_ip_address, msg.sender_ethernet_address );
      } else if ( msg.opcode == ARPMessage::OPCODE_REPLY ) {
        // 是回包，刷新等待队列
        flush_pending_datagrams( msg.sender_ip_address, msg.sender_ethernet_address );
      }
      // 未知 ARP 包，舍弃（通过switch默认处理）
      return;
    }

    // 如果是 ipv4 类型，交给上层处理
    case EthernetHeader::TYPE_IPv4: {
      if ( frame.header.dst != ethernet_address_ && frame.header.dst != ETHERNET_BROADCAST ) {
        // 不是发送给自己的
        return;
      }
      // 反序列化
      InternetDatagram dgram;
      Parser parser( frame.payload );
      dgram.parse( parser );

      datagrams_received_.push( std::move( dgram ) );
      return;
    }

    default:
      return;
  }
}
```

这个实验写完，对 cpp 的性质、模式也有了比较多一点的理解，以及分层、定义的结构。对于网络而言，大概理解了层级间的解析关系，也能感觉到其实原理上面的“层级”其实也是通过代码来实现、解析的。
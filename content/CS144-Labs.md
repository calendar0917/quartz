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
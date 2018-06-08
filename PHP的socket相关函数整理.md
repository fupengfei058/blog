要创建基于socket的应用程序，就需要详细了解socket的操作方法，这里列举PHP中一些重要的socket函数。

#### 1. socket_create ( int $domain , int $type , int $protocol )
此函数用于创建一个socket，它有三个参数，返回值是一个句柄（资源）。
$domain 指定创建socket时使用的通信协议族，其可选的值为：
* AF_INET： 基于IPv4的Internet协议
* AF_INET6：基于IPv6的Internet协议
* AF_UNIX：UNIX本地通信协议
$type 指定socket通信的交互类型，其可选的值为：
* SOCK_STREAM：提供序列化的、可靠的、全双工的、基于连接的字节流传输，支持TCP
* SOCK_DGRAM：提供数据报式的、无连接的、固定最大长度的、自动寻址功能的传输，支持UDP
* SOCK_SEQPACKET：提供序列化的、可靠的、双通道的、基于连接的数据报传输
* SOCK_RAW：提供原始的网络访问协议，可手工构建特殊协议类型的套接字，支持ICMP请求（如 ping）
* SOCK_RDM：提供可靠的数据报传输，无法保证顺序
$protocol 指定socket使用哪种具体的传输协议，包括ICMP、UDP、TCP，常量SOL_UDP对应UDP，常量SOL_TCP对应常量TCP。

#### 2. socket_bind ( resource $socket , string $address [, int $port = 0 ] )
此函数用于将IP地址和端口绑定到socket_create创建的句柄中，有三个参数，返回布尔值。
$socket 是必选参数，代表socket_create函数创建的句柄
$address 是必选参数，代表要绑定的IP地址
$port 是可选参数，代表要绑定的端口号，指定哪个端口用来监听socket连接，当socket_create函数的第一个参数为AF_INET时，需要指定这个参数。

#### 3. socket_listen ( resource $socket [, int $backlog = 0 ] )
该函数用于监听即将接入的socket连接，仅当socket的交互类型为SOCK_STREAM或SOCK_SEQPACKET时可
用，它有两个参数，返回布尔值。
$socket  是必选参数，代表socket_create函数创建的句柄（且已绑定了主机）
$backlog 是可选参数，表示队列中等候处理的（允许积压的）最大连接数。

#### 4. socket_set_block ( resource $socket )
该函数用于将socket句柄设置为阻塞模式，只有一个必选参数，返回布尔值。它可以将非阻塞模式的socket转换为阻塞模式。
当在一个阻塞模式的socket中执行某种操作（receive、send、connect、accept等）时，脚本将暂停执行，直到它收到一个信号或它完成了该操作。
$socket 是必选参数，代表一个有效的socket句柄（被socket_create或socket_accept创建的）。

#### 简要介绍一下阻塞模式和非阻塞模式的区别：
非阻塞是指函数操作在不能立刻得到结果之前，不会阻塞当前的线程，而会立即返回。而阻塞是指干不完就不准回来，必须得到对方的回应后才能继续下一步操作。特别是当用户比较多时，设置成非阻塞是很必要的。如果是阻塞模式，若两个客户端同时连接上，服务器端在处理一个客户端请求时，另外一个客户端的请求就会被阻塞，只有等到前一个客户端的事情处理完了之后，后一个客户端的请求才会被响应。

#### 5. socket_write ( resource $socket , string $buffer [, int $length = 0 ] )
该函数用于向socket中写入指定大小的缓冲数据，有三个参数，返回写入的数据的字节数。
$socket 是必选参数，代表一个有效的socket句柄。
$buffer 是必选参数，指定要写入的字符串数据。
$length 是可选参数，指定轮流写入socket中的数据的字节数，如果它的值大于$buffer的字节数，它会静默地截取至$buffer的字节数长度。

#### 6. socket_read ( resource $socket , int $length [, int $type = PHP_BINARY_READ ] )
该函数用于从socket中读取指定字节长度的数据，有三个参数，返回读取的字符串数据。
$socket 是必选参数，代表一个有效的socket句柄。
$length 是必选参数，指定读取的字节长度。
$type 是可选参数，默认值为PHP_BINARY_READ，即安全读取二进制数据；另一个可选的值为PHP_NORMAL_READ，表示当遇到 \r 或 \n 时，停止读取。

#### 7. pfsockopen ( string $hostname [, int $port = -1 [, int &$errno [, string &$errstr [, float $timeout = ini_get("default_socket_timeout") ]]]] )
该函数用于实现一个持久的socket连接，即长连接，返回一个句柄。它与 fsockopen 的区别在于，pfsockopen 建立的连接，在脚本执行完毕后，并不会断开。

#### 8. socket_set_option ( resource$socket , int$level , int$optname , mixed$optval )
该函数用于设置socket的控制选项，有四个参数，返回布尔值。
$socket 是必选参数，代表一个有效的socket句柄。
$level 是必选参数，指定option起作用的协议级别，一般取常量 SOL_SOCKET。
$optname 是必选参数，指定要控制的选项名称。
$optval 是必选参数，指定选项的值。

#### 9. socket_last_error ([ resource$socket ] )
该函数用于获取任何socket函数产生的最后错误代号，返回值为整型。

#### 10. socket_strerror ( int $errno )
该函数用于获取错误代号代表的错误描述，返回值为字符串。

作为一名非底层程序员，要想深入了解socket的内部实现机制是很困难的，我们只需明白socket是一套操作系统封装好的实现进程通信的函数，会创建和调用就够了。

PHP的语言特性和自身定位决定了它只适合做socket客户端，而不适合做socket服务器端。因为socket主要面向底层和网络服务开发，一般服务器端都是用 C 或 Java 等语言实现，这样能更好地操作底层，对网络服务开发中遇到的问题（如并发、阻塞等）也有成熟完善的解决方案，而PHP显然不适合这种应用场景。

实际上，PHP操作MySQL数据库也是通过socket进行的，这正是由于socket屏蔽了底层的协议，使得网络服务之间的互联互通变得简单。

除了传统的服务器端语言实现的socket外，随着HTML5的流行，浏览器客户端实现的WebSocket也逐渐兴起，对于这一点值得关注，FlashSocket也是一个不错的解决方案。

要在客户端操作socket，可使用fsockopen、socket_create 或 stream_socket_client 等函数实现，如果是PHP5，推荐使用stream_socket_client。

#### socket交互应用实例：使用socket提交表单
新建一个 test.php 文件，向 http://demo.com/index.php?id=1  提交表单数据，代码如下：
```php
<?php  
$data = array('comment'=>'this is a robot comment');  
$data = http_build_query($data);  
  
$out = "POST http://demo.com/index.php?id=1 HTTP/1.1\r\n";  // 通过POST方式发送数据  
$out .= "Host: demo.com\r\n";  
$out .= "Content-type: application/x-www-form-urlencoded; charset=UTF-8\r\n";  
$out .= "Content-length: ".strlen($data)."\r\n";  
$out .= "User-Agent: Mozilla/5.0 (Windows NT 6.1; rv:48.0) Gecko/20100101 Firefox/48.0"."\r\n";  
$out .= "Connection: close"."\r\n"."\r\n";    // 注意：此处有两个 \r\n  
  
$out .= $data."\r\n";   // 正文数据  
  
$fp = fsockopen("demo.com", 80, $errno, $errstr, 30);  // 创建socket客户端连接  
  
// $fp = stream_socket_client("tcp://demo.com:80", $errno, $errstr, 30);  推荐这种写法  
  
fwrite($fp, $out);    // 向服务器发送数据  
  
while (!feof($fp)) {  
    echo fgets($fp, 1280);    // 读取服务器响应的数据  
}  
fclose($fp);  // 关闭socket连接  
?>
```
#### 需要注意以下几点：
* fsockopen的第一个参数，也可以使用IP地址，不要带 http:// 字符串，除非使用SSL等
* 请求头（headers）不一定要带上所有的头域，一般只需带上几个核心的header即可
* 在最后一个header处，即 Connection 后有两个换行
* 注意编码问题

如果是PHP5，建议使用 stream_socket_client 代替 fsockopen，也就是将下面的代码：
$fp = fsockopen("demo.com", 80, $errno, $errstr, 30); 
改为：
$fp = stream_socket_client("tcp://demo.com:80", $errno, $errstr, 30); 

在PHP中，99.9%的socket应用属于流套接字的范畴，由于数据包套接字和原始套接字涉及比较底层的协议知识，这里就不作深究，有兴趣的朋友可自行学习。

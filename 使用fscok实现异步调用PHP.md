Web 服务器执行一个脚本，可能几毫秒就完成，也可能几分钟都完不成。如果程序执行缓慢，用户可能没有耐心等下去，就关闭浏览器了。

而有的时候，我们更本不关心这些耗时的脚本的执行结果，但却还要等他执行完返回，才能继续下一步。   

那么有没有什么办法，只是简单的触发调用这些耗时的脚本然后就继续下一步，让这些耗时的脚本在服务端慢慢执行？ 
     
接下来，我将使用fscokopen来实现这一功能。
    
PHP是支持socket编程的，就是fsockopen， 在以前做CMS的时候，我也曾经用过它做过smtp发信。

fscokopen返回一个到远程主机连接的句柄。你可以像使用fopen返回的句柄一样，对她进行写fwrite，读取fgets, fread等操作。
    
我们的异步PHP，主要想要的效果就是，触发一个PHP脚本，然后立即返回，留它在服务器端慢慢执行。前面我也写过一篇文章讨论过这个问题。

那么，我们就可以使用fsockopen连接到本地服务器，触发脚本执行，然后立即返回，不等待脚本执行完成。
```php
function triggerRequest($url, $post_data = array(), $cookie = array())…{
    $method = "GET";  //可以通过POST或者GET传递一些参数给要触发的脚本
    $url_array = parse_url($url); //获取URL信息，以便平凑HTTP HEADER
    $port = isset($url_array['port'])? $url_array['port'] : 80; 

    $fp = fsockopen($url_array['host'], $port, $errno, $errstr, 30); 
    if (!$fp) …{
            return FALSE;
    }
    $getPath = $url_array['path'] ."?". $url_array['query'];
    if(!empty($post_data))…{
            $method = "POST";
    }
    $header = $method . " " . $getPath;
    $header .= " HTTP/1.1\r\n";
    $header .= "Host: ". $url_array['host'] . "\r\n "; //HTTP 1.1 Host域不能省略
    /**//*以下头信息域可以省略
    $header .= "User-Agent: Mozilla/5.0 (Windows; U; Windows NT 5.1; en-US; rv:1.8.1.13) Gecko/20080311 Firefox/2.0.0.13 \r\n";
    $header .= "Accept: text/xml,application/xml,application/xhtml+xml,text/html;q=0.9,text/plain;q=0.8,image/png,q=0.5 \r\n";
    $header .= "Accept-Language: en-us,en;q=0.5 ";
    $header .= "Accept-Encoding: gzip,deflate\r\n";
     */

    $header .= "Connection:Close\r\n";
    if(!empty($cookie))…{
            $_cookie = strval(NULL);
            foreach($cookie as $k => $v)…{
                    $_cookie .= $k."=".$v."; ";
            }
            $cookie_str =  "Cookie: " . base64_encode($_cookie) ." \r\n";//传递Cookie
            $header .= $cookie_str;
    }
    if(!empty($post_data))…{
            $_post = strval(NULL);
            foreach($post_data as $k => $v)…{
                    $_post .= $k."=".$v."&";
            }
            $post_str  = "Content-Type: application/x-www-form-urlencoded\r\n";//POST数据
            $post_str .= "Content-Length: ". strlen($_post) ." \r\n";//POST数据的长度
            $post_str .= $_post."\r\n\r\n "; //传递POST数据
            $header .= $post_str;
    }
    fwrite($fp, $header);
    //echo fread($fp, 1024); //我们不关心服务器返回
    fclose($fp);
    return true;
}
```
现在，就可以通过这个函数来触发一个PHP脚本的执行，然后函数就会返回。 我们就可以接着执行下一步操作了。

还有一个问题就是，当客户端断开连接以后。也就是triggerRequest发送请求后，立即关闭了连接，那么可能会引起服务器端正在执行的脚本退出。

在 PHP 内部，系统维护着连接状态，其状态有三种可能的情况：

* 0 – NORMAL（正常）
* 1 – ABORTED（异常退出）
* 2 – TIMEOUT（超时）

当 PHP 脚本正常地运行 NORMAL 状态时，连接为有效。当客户端中断连接时，ABORTED 状态的标记将会被打开。远程客户端连接的中断通常是由用户点击 STOP 按钮导致的。当连接时间超过 PHP 的时限（请参阅 set_time_limit() 函数）时，TIMEOUT 状态的标记将被打开。

可以决定脚本是否需要在客户端中断连接时退出。有时候让脚本完整地运行会带来很多方便，即使没有远程浏览器接受脚本的输出。默认的情况是当远程客户端连接 中断时脚本将会退出。该处理过程可由 php.ini 的 ignore_user_abort 或由 Apache .conf 设置中对应的“php_value ignore_user_abort”以及 ignore_user_abort() 函数来控制。如果没有告诉 PHP 忽略用户的中断，脚本将会被中断，除非通过 register_shutdown_function() 设置了关闭触发函数。通过该关闭触发函数，当远程用户点击 STOP 按钮后，脚本再次尝试输出数据时，PHP 将会检测到连接已被中断，并调用关闭触发函数。

脚本也有可能被内置的脚本计时器中断。默认的超时限制为 30 秒。这个值可以通过设置 php.ini 的 max_execution_time 或 Apache .conf 设置中对应的“php_value max_execution_time”参数或者 set_time_limit() 函数来更改。当计数器超时的时候，脚本将会类似于以上连接中断的情况退出，先前被注册过的关闭触发函数也将在这时被执行。在该关闭触发函数中，可以通过调 用 connection_status() 函数来检查超时是否导致关闭触发函数被调用。如果超时导致了关闭触发函数的调用，该函数将返回 2。

需要注意的一点是 ABORTED 和 TIMEOUT 状态可以同时有效。这在告诉 PHP 忽略用户的退出操作时是可能的。PHP 将仍然注意用户已经中断了连接但脚本仍然在运行的情况。如果到了运行的时间限制，脚本将被退出，设置过的关闭触发函数也将被执行。在这时会发现函数 connection_status() 返回 3。

所以还在要触发的脚本中指明:
 ```
ignore_user_abort(TRUE); //如果客户端断开连接，不会引起脚本abort.
set_time_limit(0);//取消脚本执行延时上限
```
 或者，也可以使用:
 ```
 register_shutdown_function(callback fuction[, parameters]);//注册脚本退出时执行的函数
 ```
 链接：http://www.laruence.com/2008/04/16/98.html

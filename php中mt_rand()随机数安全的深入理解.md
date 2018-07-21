mt_rand()使用mersennetwister算法返回随机整数，mt_rand()并不是一个 真·随机数 生成函数,实际上绝大多数编程语言中的随机数函数生成的都都是伪随机数。关于真随机数和伪随机数的区别这里不展开解释，只需要简单了解一点：伪随机是由可确定的函数（常用线性同余），通过一个种子（常用时钟），产生的伪随机数。这意味着：如果知道了种子，或者已经产生的随机数，都可能获得接下来随机数序列的信息（可预测性）。

简单假设一下 mt_rand()内部生成随机数的函数为: rand = seed+(i * 10) 其中 seed 是随机数种子，i是第几次调用这个随机数函数。当我们同时知道 i 和 rand 两个值的时候，就能很容易的算出seed的值来。比如 rand=21 , i=2 代入函数 21=seed+(2*10) 得到 seed=1 。是不是很简单，当我们拿到seed之后，就能计算出当 i 为任意值时候的 rand 的值了。

#### PHP的自动播种

> Note: 自 PHP 4.2.0 起，不再需要用 srand() 或 mt_srand() 给随机数发生器播种 ，因为现在是由系统自动完成的。

那么问题就来了，到底系统自动完成播种是在什么时候，如果每次调用mt_rand()都会自动播种那么破解seed也就没意义了。关于这一点manual并没有给出详细信息。mtrand源码如下：

```c
PHPAPI void php_mt_srand(uint32_t seed)
{
 /* Seed the generator with a simple uint32 */
 php_mt_initialize(seed, BG(state));
 php_mt_reload();
 
 /* Seed only once */
 BG(mt_rand_is_seeded) = 1; 
}
/* }}} */
 
/* {{{ php_mt_rand
 */
PHPAPI uint32_t php_mt_rand(void)
{
 /* Pull a 32-bit integer from the generator state
 Every other access function simply transforms the numbers extracted here */
 
 register uint32_t s1;
 
 if (UNEXPECTED(!BG(mt_rand_is_seeded))) {
 php_mt_srand(GENERATE_SEED());
 }
 
 if (BG(left) == 0) {
 php_mt_reload();
 }
 --BG(left);
 
 s1 = *BG(next)++;
 s1 ^= (s1 >> 11);
 s1 ^= (s1 << 7) & 0x9d2c5680U;
 s1 ^= (s1 << 15) & 0xefc60000U;
 return ( s1 ^ (s1 >> 18) );
}
```

可以看到每次调用mt_rand()都会先检查是否已经播种。如果已经播种就直接产生随机数，否则调用php_mt_srand来播种。也就是说每个php cgi进程期间，只有第一次调用mt_rand()会自动播种。接下来都会根据这个第一次播种的种子来生成随机数。而php的几种运行模式中除了CGI(每个请求启动一个cgi进程，请求结束后关闭。每次都要重新读取php.ini 环境变量等导致效率低下，现在用的应该不多了)以外，基本都是一个进程处理完请求之后standby等待下一个，处理多个请求之后才会回收（超时也会回收）。

写个脚本测试一下

```php
<?php
//pid.php
echo getmypid();
```
```php
<?php
//test.php
$old_pid = file_get_contents('http://localhost/pid.php');
$i=1;
while(true){
 $i++;
 $pid = file_get_contents('http://localhost/pid.php');
 if($pid!=$old_pid){
 echo $i;
 break;
 }
}
```
测试结果：

apache 1000请求

nginx 500请求

当然这个测试仅仅确认了apache和nginx一个进程可以处理的请求数，再来验证一下刚才关于自动播种的结论：
```php
<?php
//pid1.php
if(isset($_GET['rand'])){
 echo mt_rand();
}else{
 echo getmypid();
}
```
```php
<?php
//pid2.php
echo mt_rand();
```
```php
<?php
//test.php
$old_pid = file_get_contents('http://localhost/pid1.php');
echo "old_pid:{$old_pid}\r\n";
while(true){
 $pid = file_get_contents('http://localhost/pid1.php');
 if($pid!=$old_pid){
 echo "new_pid:{$pid}\r\n";
 for($i=0;$i<20;$i++){
  $random = mt_rand(1,2);
  echo file_get_contents("http://localhost/pid".$random.".php?rand=1")." ";
 }
 
 break;
 }
}
```
通过pid来判断,当新进程开始的时候，随机获取两个页面其中一个的 mt_rand() 的输出:
```
old_pid:972 new_pid:7752 1513334371 2014450250 1319669412 499559587 117728762 1465174656 1671827592 1703046841 464496438 1974338231 46646067 981271768 1070717272 571887250 922467166 606646473 134605134 857256637 1971727275 2104203195
```
拿第一个随机数 1513334371 去爆破种子：
```
smldhz@vm:~/php_mt_seed-3.2$ ./php_mt_seed 1513334371 Found 0, trying 704643072 - 738197503, speed 28562751 seeds per second seed = 735487048 Found 1, trying 1308622848 - 1342177279, speed 28824291 seeds per second seed = 1337331453 Found 2, trying 3254779904 - 3288334335, speed 28811010 seeds per second seed = 3283082581 Found 3, trying 4261412864 - 4294967295, speed 28677071 seeds per second Found 3
```
爆破出了3个可能的种子，数量很少 手动一个一个测试:
```php
<?php
mt_srand(735487048);//手工播种
for($i=0;$i<21;$i++){
 echo mt_rand()." ";
}
```
输出:

前20位跟上面脚本获取的一模一样，确认种子就是 1513334371 。有了种子我们就能计算出任意次数调用mt_rand()生成的随机数了。比如这个脚本我生成了21位，最后一位是 1515656265 如果跑完刚才的脚本之后没访问过站点，那么打开 http://localhost/pid2.php 就能看到相同的 1515656265 。

所以我们得到结论：

php的自动播种发生在php cgi进程中第一次调用mt_rand()的时候。跟访问的页面无关，只要是同一个进程处理的请求，都会共享同一个最初自动播种的种子。

#### php_mt_seed

我们已经知道随机数的生成是依赖特定的函数，上面曾经假设为 rand = seed+(i*10) 。对于这样一个简单的函数，我们当然可以直接计算（口算）出一个（组）解来，但 mt_rand() 实际使用的函数可是相当复杂且无法逆运算的。有效的破解方法其实是穷举所有的种子并根据种子生成随机数序列再跟已知的随机数序列做比对来验证种子是否正确。php_mt_seed^phpmtseed就是这么一个工具，它的速度非常快，跑完2^32位seed也就几分钟。它可以根据单次mt_rand()的输出结果直接爆破出可能的种子（上面有示例），当然也可以爆破类似mt_rand(1,100)这样限定了MIN MAX输出的种子（下面实例中有用到）。

#### 安全问题
说了这么多，那到底随机数怎么不安全了呢？其实函数本身没有问题，官方也明确提示了生成的随机数不应用于安全加密用途（虽然中文版本manual没写）。问题在于开发者并没有意识到这并不是一个 真·随机数 。我们已经知道，通过已知的随机数序列可以爆破出种子。也就是说，只要任意页面中存在输出随机数或者其衍生值（可逆推随机值），那么其他任意页面的随机数将不再是“随机数”。常见的输出随机数的例子比如验证码，随机文件名等等。常见的随机数用于安全验证的比如找回密码校验值，比如加密key等等。一个理想中的攻击场景：

夜深人静，等待apache(nginx)收回所有php进程（确保下次访问会重新播种），访问一次验证码页面，根据验证码字符逆推出随机数，再根据随机数爆破出随机数种子。接着访问找回密码页面，生成的找回密码链接是基于随机数的。我们就可以轻松计算出这个链接,找回管理员的密码…………

链接：http://www.php.cn/php-weizijiaocheng-380106.html

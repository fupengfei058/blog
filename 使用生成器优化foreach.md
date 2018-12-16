#### 概念引入
生成器是PHP 5.5.0才引入的功能，在遍历大量的数据时可节省内存。先看一下这个简单的PHP函数：
```php
function createRange($number){
    $data = [];
    for($i=0;$i<$number;$i++){
        $data[] = time();
    }
    return $data;
}
```
这是一个非常常见的PHP函数，我们在处理一些数组的时候经常会使用。这里的代码也非常简单：

1. 我们创建一个函数。
2. 函数内包含一个 for 循环，我们循环的把当前时间放到$data里面
3. for循环执行完毕，把 $data 返回出去。
下面没完，我们继续。我们再写一个函数，把这个函数的返回值循环打印出来：
```php
$result = createRange(10); // 这里调用上面我们创建的函数
foreach($result as $value){
    sleep(1);//这里停顿1秒，我们后续有用
    echo $value.'<br />';
}
```
可以输出以下运行结果
```
1544940607
1544940607
1544940607
1544940607
1544940607
1544940607
1544940607
1544940607
1544940607
1544940607
```
这里非常完美，没有任何问题。（当然 sleep(1) 效果你们看不出来）

我们注意到，在调用函数 createRange 的时候给 $number 的传值是10，一个很小的数字。假设，现在传递一个值10000000（1000万）。

那么，在函数 createRange 里面，for循环就需要执行1000万次。且有1000万个值被放到 $data 里面，而$data数组在是被放在内存内。所以，在调用函数时候会占用大量内存。

这里，生成器就可以大显身手了。

#### 创建生成器
我们直接修改代码：
```php
function createRange($number){
    for($i=0;$i<$number;$i++){
        yield time();
    }
}
```
看下这段和刚刚很像的代码，我们删除了数组 $data ，而且也没有返回任何内容，而是在 time() 之前使用了一个关键字yield。
#### 使用生成器
我们再运行一下第二段代码：
```php
$result = createRange(10); // 这里调用上面我们创建的函数
foreach($result as $value){
    sleep(1);
    echo $value.'<br />';
}
```
输出结果变成了：
```
1544940811
1544940812
1544940813
1544940814
1544940815
1544940816
1544940817
1544940818
1544940819
1544940820
```
我们奇迹般的发现了，输出的值和第一次没有使用生成器的不一样。这里的值（时间戳）中间间隔了1秒。

这里的间隔一秒其实就是 sleep(1) 造成的后果。但是为什么第一次没有间隔？那是因为：

未使用生成器时： createRange 函数内的 for 循环结果被很快放到$data 中，并且立即返回。所以，foreach循环的是一个固定的数组。

使用生成器时： createRange 的值不是一次性快速生成，而是依赖于 foreach 循环。 foreach 循环一次， for 执行一次。

#### 深入理解生成器
下面我们来对于刚刚的代码进行剖析。
```php
function createRange($number){
    for($i=0;$i<$number;$i++){
        yield time();
    }
}

$result = createRange(10); // 这里调用上面我们创建的函数
foreach($result as $value){
    sleep(1);
    echo $value.'<br />';
}
```
我们来还原一下代码执行过程。

1. 首先调用 createRange 函数，传入参数10，但是 for 值执行了一次然后停止了，并且告诉 foreach 第一次循环可以用的值。
2. foreach 开始对 $result 循环，进来首先 sleep(1) ，然后开始使用 for 给的一个值执行输出。
3. foreach 准备第二次循环，开始第二次循环之前，它向 for 循环又请求了一次。
4. for 循环于是又执行了一次，将生成的时间戳告诉 foreach .
5. foreach 拿到第二个值，并且输出。由于 foreach 中 sleep(1) ，所以， for 循环延迟了1秒生成当前时间

所以，整个代码执行中，始终只有一个记录值参与循环，内存中也只有一条信息。

无论开始传入的 $number 有多大，由于并不会立即生成所有结果集，所以内存始终是一条循环的值。

到这里，你应该已经大概理解什么是生成器了。下面说下生成器原理。

首先明确一个概念：生成器yield关键字不是返回值，他的专业术语叫产出值，只是生成一个值

那么代码中 foreach 循环的是什么？其实是PHP在使用生成器的时候，会返回一个 Generator 类的对象。 foreach 可以对该对象进行迭代，每一次迭代，PHP会通过 Generator 实例计算出下一次需要迭代的值。这样 foreach 就知道下一次需要迭代的值了。

而且，在运行中 for 循环执行后，会立即停止。等待 foreach 下次循环时候再次和  for  索要下次的值的时候，循环才会再执行一次，然后立即再次停止。直到不满足条件不执行结束。

#### 实际开发应用
读取超大文件：

PHP开发很多时候都要读取大文件，比如csv文件、text文件，或者一些日志文件。这些文件如果很大，比如5个G。这时，直接一次性把所有的内容读取到内存中计算不太现实。

这里生成器就可以派上用场啦。简单看个例子：读取text文件

```php
<?php
header("content-type:text/html;charset=utf-8");
function readTxt()
{
    # code...
    $handle = fopen("./test.txt", 'rb');

    while (feof($handle)===false) {
        # code...
        yield fgets($handle);
    }

    fclose($handle);
}

foreach (readTxt() as $key => $value) {
    # code...
    echo $value.'<br />';
}
```
通过生成器每次被加载到内存中的文字只有一行，大大的减小了内存的使用。这样，即使读取上G的文本也不用担心，完全可以像读取很小文件一样编写代码。


### 什么是捕获组

我们先看一下PHP的正则匹配函数

> int preg_match ( string $pattern , string $subject [, array &$matches [, int $flags = 0 [, int $offset = 0 ]]] )
前面两项是我们常用的，$pattern是正则匹配模式，$string是要匹配的字符串。

array &$match,它是一个数组，&表示匹配出来的结果会被写入$match中。

int $flags 如果传递了这个标记, 对于每一个出现的匹配返回时会附加字符串偏移量(相对于目标字符串的)。

int $offset 用于指定从目标字符串的某个未知开始搜索(单位是字节)。

我们主要看一下$match的值里会有什么：

```
$mode = '/a=(\d+)b=(\d+)c=(\d+)/';

$str='**a=4b=98c=56**';

$res=preg_match($mode,$str,$match);

var_dump($match);
```
结果如下：

```
array (size=4)

  0 => string 'a=4b=98c=56' (length=11)

  1 => string '4' (length=1)

  2 => string '98' (length=2)

  3 => string '56' (length=2)
```
现在我们知道了什么是捕获组，捕获组是正则表达示中以()括起来的部分，每一对()是一个捕获组。

PHP会为它编号，从1开始。至于为什么会从1开始，那是因为PHP把匹配到的完整字符串编号为0。

如果有多个括号或嵌套括号，按左边括号出现的顺序来进行编号，如图：

![github](http://images2015.cnblogs.com/blog/819496/201511/819496-20151106104131071-2025639233.jpg)

按图中的匹配模式匹配时，捕获组的123号分别是红绿蓝。

### 捕获组的忽略与命名

我们还可以阻止PHP为匹配组的编号：在匹配组中模式前加  ?: 

$mode = '/a=(\d+)b=(?:\d+)c=(\d+)/';

这样，匹配结果就会变成：

```
array (size=3)

  0 => string 'a=4b=98c=56' (length=11)

  1 => string '4' (length=1)

  2 => string '56' (length=2)
```
当然，我们也可以在括号的内部为它给它独特的名字。

> 命名子组可以接受(?<name>), (?'name') 以及(?P<name>)语法. 之前版本仅接受(?P<name>)语法.
例如：$mode = '/a=(\d+)b=(?P<sec>\d+)c=(\d+)/';

使用时结果为：

```
array (size=5)

  0 => string 'a=4b=98c=56' (length=11)

  1 => string '4' (length=1)

  'sec' => string '98' (length=2)

  2 => string '98' (length=2)

  3 => string '56' (length=2)
```
在保留索引数组的同时，加上一个关联项，key值为捕获组名。

### 捕获组的反向引用

我们在用preg_replace()函数进行正则替换时，我们还可以使用 \n 或 $n 来引用第n个捕获组.

```
$mode = '/a=(\d+)b=(\d+)c=(\d+)/';

$str='**a=4b=98c=56**';

$rp='\1/$2/\3/';

echo preg_replace($mode,$rp,$str);//**4/98/56/**
```
\1表示捕获组1(4),$2为捕获组2(98),\3为捕获组3(56)。

### 非捕获组的用法：

为什么称为非捕获组呢？那是因为它们有捕获组的特性，在匹配模式的()中，但是匹配时，PHP不会为它们编组，它们只会影响匹配结果，并不作为结果输出。

/d(?=xxx)    匹配"后面是xxx的一个数字"。

注意格式：只能放在匹配模式字符串之后！

例如：

```
$pattern='/\d(?=abc)/';

$str="ab36abc8eg";

$res=preg_match($pattern,$str,$match);

var_dump($match);//6
```
匹配的6，因为只有它作为一个数字，后面还有abc。

(?<=xxx) /d 匹配"前面是xxx的一个数字"

注意格式：只能放在匹配模式字符串之前！

例如：

```
$pattern='/(?<=abc)\d/';

$str="ab36abc8eg";

$res=preg_match($pattern,$str,$match);

var_dump($match);//8
```
匹配的8，因为只有它作为一个数字，后面还有abc。

与(?=xxx)  (?<=xxx)相对的是(?!=xxx)  (?<!=xxx) 它们在=前加了非运算符 “!”

它表示前面/后面不是xxx的字符串，这里就不再举例了。

### PHP7比PHP5多了哪些新特性
* #### 空合并操作符（Null Coalesce Operator） 
```php
$name = $name ?? "NoName"; // 如果$name有值就取其值，否则设$name成"NoName"
```
* #### 飞船操作符（Spaceship Operator） 
形式：(expr) <=> (expr) 左边运算对象小，则返回-1；左、右两边运算对象相等，则返回0；左边运算对象大，则返回1。
```php
$name = ["Simen", "Suzy", "Cook", "Stella"];
 usort($name, function ($left, $right) {
     return $left <=> $right;
 });
 print_r($name);
 ```
* #### 常量数组（Constant Array） 
PHP 7 之前只允许类/接口中使用常量数组，现在 PHP 7 也支持非类/接口的普通常量数组了。
```php
define("USER", [
    "name"  => "Simen",
    "sex"   => "Male",
    "age"   => "38",
    "skill" => ["PHP", "MySQL", "C"]
]);
 // USER["skill"][2] = "C/C++";  // PHP Fatal error:  Cannot use temporary expression in write context in...
 ```
统一了变量语法
```php
$goo = [
    "bar" => [
        "baz" => 100,
        "cug" => 900
    ]
];

$foo = "goo";

$$foo["bar"]["baz"];  // 实际为：($$foo)['bar']['baz']; PHP 5 中为：${$foo['bar']['baz']};
                      // PHP 7 中一个笼统的判定规则是，由左向右结合。
```
* #### Throwable 接口 
这是 PHP 7 引进的一个值得期待的新特性，将极大增强 PHP 错误处理能力。PHP 5 的 try ... catch ... finally 无法处理传统错误，如果需要，你通常会考虑用 set_error_handler() 来 Hack 一下。但是仍有很多错误类型是 set_error_handler() 捕捉不到的。PHP 7引入 Throwable 接口，错误及异常都实现了 Throwable，无法直接实现 Throwable，但可以扩展 \Exception 和 \Error 类。可以用 Throwable 捕捉异常跟错误。\Exception 是所有PHP及用户异常的基类；\Error 是所有内部PHP错误的基类。
```php
$name = "Tony";
try {
    $name = $name->method();
} catch (\Error $e) {
    echo "出错消息 --- ", $e->getMessage(), PHP_EOL;
}

try {
    $name = $name->method();
} catch (\Throwable $e) {
    echo "出错消息 --- ", $e->getMessage(), PHP_EOL;
}

try {
    intdiv(5, 0);
} catch (\DivisionByZeroError $e) {
    echo "出错消息 --- ", $e->getMessage(), PHP_EOL;
}
```
* #### use 组合声明
use 组合声明可以减少 use 的输入冗余。
```php
use PHPGoodTaste\Utils\{
     Util,
     Form,
     Form\Validation,
     Form\Binding
 };
```
* #### 一次捕捉多种类型的异常 / 错误
PHP 7.1 新添加了捕获多种异常/错误类型的语法——通过竖杠“|”来实现。
```php
try {
      throw new LengthException("LengthException");
    //   throw new DivisionByZeroError("DivisionByZeroError");
    //   throw new Exception("Exception");
} catch (\DivisionByZeroError | \LengthException $e) {
    echo "出错消息 --- ", $e->getMessage(), PHP_EOL;
} catch (\Exception $e) {
    echo "出错消息 --- ", $e->getMessage(), PHP_EOL;
} finally {
    // ...
}
```
* #### 可见性修饰符的变化
PHP 7.1 之前的类常量是不允许添加可见性修饰符的，此时类常量可见性相当于 public。PHP 7.1 为类常量添加了可见性修饰符支持特性。总的来说，可见性修饰符使用范围如下所示：
* 函数/方法：public、private、protected、abstract、final
* 类：abstract、final
* 属性/变量：public、private、protected
* 类常量：public、private、protected
```php
class YourClass 
{
    const THE_OLD_STYLE_CONST = "One";

    public const THE_PUBLIC_CONST = "Two";
    private const THE_PRIVATE_CONST = "Three";
    protected const THE_PROTECTED_CONST = "Four";
}
```
* #### iterable 伪类型
PHP 7.1 引入了 iterable 伪类型。iterable 类型适用于数组、生成器以及实现了 Traversable 的对象，它是 PHP 中保留类名。
```php
$fn = function (iterable $it) : iterable {
    $result = [];
    foreach ($it as $value) {
        $result[] = $value + 1000;
    }
    return $result;
};

$fn([1, 2, 3, 4, 5, 6, 7, 8, 9, 10]);
```
* #### 可空类型（Nullable Type）
PHP 7.1 引入了可空类型。看看新兴的 Kotlin 编程语言的一个噱头特性就是可空类型。PHP 越来越像“强类型语言了”。对于同一类型的强制要求，可以设置其是否可空。
```php
$fn = function (?int $in) {
    return $in ?? "NULL";
};

$fn(null);
$fn(5);
$fn();  // TypeError: Too few arguments to function {closure}(), 0 passed in ...
```
* #### Void 返回类型
```php
function first(): void {
    // ...
}

function second(): void {
    // ...
    return;
}
```
* #### 性能提升了两倍
* #### 结合比较运算符 (<=>)
* #### 标量类型声明
* #### 返回类型声明
* #### try...catch 增加多条件判断，更多 Error 错误可以进行异常处理
* #### 匿名类，现在支持通过new class 来实例化一个匿名类，这可以用来替代一些“用后即焚”的完整类定义
[参考地址](http://php.net/manual/zh/migration70.new-features.php)

### 为什么PHP7比PHP5性能提升了
* 变量存储字节减小，减少内存占用，提升变量操作速度
* 改善数组结构，数组元素和 hash 映射表被分配在同一块内存里，降低了内存占用、提升了 cpu 缓存命中率
* 改进了函数的调用机制，通过优化参数传递的环节，减少了一些指令，提高执行效率

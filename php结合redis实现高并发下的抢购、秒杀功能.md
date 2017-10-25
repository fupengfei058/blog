抢购、秒杀是如今很常见的一个应用场景，主要需要解决的问题有两个：

1. 高并发对数据库产生的压力
2. 竞争状态下如何解决库存的正确减少（"超卖"问题）
对于第一个问题，已经很容易想到用缓存来处理抢购，避免直接操作数据库，例如使用Redis。
重点在于第二个问题

常规写法：

查询出对应商品的库存，看是否大于0，然后执行生成订单等操作，但是在判断库存是否大于0处，如果在高并发下就会有问题，导致库存量出现负数

```php
<?php  
$conn=mysql_connect("localhost","big","123456");    
if(!$conn){    
    echo "connect failed";    
    exit;    
}   
mysql_select_db("big",$conn);   
mysql_query("set names utf8");  
  
$price=10;  
$user_id=1;  
$goods_id=1;  
$sku_id=11;  
$number=1;  
  
//生成唯一订单  
function build_order_no(){  
    return date('ymd').substr(implode(NULL, array_map('ord', str_split(substr(uniqid(), 7, 13), 1))), 0, 8);  
}  
//记录日志  
function insertLog($event,$type=0){  
    global $conn;  
    $sql="insert into ih_log(event,type)   
    values('$event','$type')";    
    mysql_query($sql,$conn);    
}  
  
//模拟下单操作  
//库存是否大于0  
$sql="select number from ih_store where goods_id='$goods_id' and sku_id='$sku_id'";//解锁 此时ih_store数据中goods_id='$goods_id' and sku_id='$sku_id' 的数据被锁住(注3)，其它事务必须等待此次事务 提交后才能执行  
$rs=mysql_query($sql,$conn);  
$row=mysql_fetch_assoc($rs);  
if($row['number']>0){//高并发下会导致超卖  
    $order_sn=build_order_no();  
    //生成订单    
    $sql="insert into ih_order(order_sn,user_id,goods_id,sku_id,price)   
    values('$order_sn','$user_id','$goods_id','$sku_id','$price')";    
    $order_rs=mysql_query($sql,$conn);   
      
    //库存减少  
    $sql="update ih_store set number=number-{$number} where sku_id='$sku_id'";  
    $store_rs=mysql_query($sql,$conn);    
    if(mysql_affected_rows()){    
        insertLog('库存减少成功');  
    }else{    
        insertLog('库存减少失败');  
    }   
}else{  
    insertLog('库存不够');  
}  
?>
```
#### 优化方案1：将库存字段number字段设为unsigned，当库存为0时，因为字段不能为负数，将会返回false

```php
//库存减少  
$sql="update ih_store set number=number-{$number} where sku_id='$sku_id' and number>0";  
$store_rs=mysql_query($sql,$conn);    
if(mysql_affected_rows()){    
    insertLog('库存减少成功');  
}
```
#### 优化方案2：使用mysql的事务，锁住操作的行
```php
<?php  
$conn=mysql_connect("localhost","big","123456");    
if(!$conn){    
    echo "connect failed";    
    exit;    
}   
mysql_select_db("big",$conn);   
mysql_query("set names utf8");  
  
$price=10;  
$user_id=1;  
$goods_id=1;  
$sku_id=11;  
$number=1;  
  
//生成唯一订单号  
function build_order_no(){  
    return date('ymd').substr(implode(NULL, array_map('ord', str_split(substr(uniqid(), 7, 13), 1))), 0, 8);  
}  
//记录日志  
function insertLog($event,$type=0){  
    global $conn;  
    $sql="insert into ih_log(event,type)   
    values('$event','$type')";    
    mysql_query($sql,$conn);    
}  
  
//模拟下单操作  
//库存是否大于0  
mysql_query("BEGIN");   //开始事务  
$sql="select number from ih_store where goods_id='$goods_id' and sku_id='$sku_id' FOR UPDATE";//此时这条记录被锁住,其它事务必须等待此次事务提交后才能执行  
$rs=mysql_query($sql,$conn);  
$row=mysql_fetch_assoc($rs);  
if($row['number']>0){  
    //生成订单   
    $order_sn=build_order_no();   
    $sql="insert into ih_order(order_sn,user_id,goods_id,sku_id,price)   
    values('$order_sn','$user_id','$goods_id','$sku_id','$price')";    
    $order_rs=mysql_query($sql,$conn);   
      
    //库存减少  
    $sql="update ih_store set number=number-{$number} where sku_id='$sku_id'";  
    $store_rs=mysql_query($sql,$conn);    
    if(mysql_affected_rows()){    
        insertLog('库存减少成功');  
        mysql_query("COMMIT");//事务提交即解锁  
    }else{    
        insertLog('库存减少失败');  
    }  
}else{  
    insertLog('库存不够');  
    mysql_query("ROLLBACK");  
}  
?>
```

#### 优化方案3：使用非阻塞的文件排他锁
```php
<?php  
$conn=mysql_connect("localhost","root","123456");    
if(!$conn){    
    echo "connect failed";    
    exit;    
}   
mysql_select_db("big-bak",$conn);   
mysql_query("set names utf8");  
  
$price=10;  
$user_id=1;  
$goods_id=1;  
$sku_id=11;  
$number=1;  
  
//生成唯一订单号  
function build_order_no(){  
    return date('ymd').substr(implode(NULL, array_map('ord', str_split(substr(uniqid(), 7, 13), 1))), 0, 8);  
}  
//记录日志  
function insertLog($event,$type=0){  
    global $conn;  
    $sql="insert into ih_log(event,type)   
    values('$event','$type')";    
    mysql_query($sql,$conn);    
}  
  
$fp = fopen("lock.txt", "w+");  
if(!flock($fp,LOCK_EX | LOCK_NB)){  
    echo "系统繁忙，请稍后再试";  
    return;  
}  
//下单  
$sql="select number from ih_store where goods_id='$goods_id' and sku_id='$sku_id'";  
$rs=mysql_query($sql,$conn);  
$row=mysql_fetch_assoc($rs);  
if($row['number']>0){//库存是否大于0  
    //模拟下单操作   
    $order_sn=build_order_no();   
    $sql="insert into ih_order(order_sn,user_id,goods_id,sku_id,price)   
    values('$order_sn','$user_id','$goods_id','$sku_id','$price')";    
    $order_rs=mysql_query($sql,$conn);   
      
    //库存减少  
    $sql="update ih_store set number=number-{$number} where sku_id='$sku_id'";  
    $store_rs=mysql_query($sql,$conn);    
    if(mysql_affected_rows()){    
        insertLog('库存减少成功');  
        flock($fp,LOCK_UN);//释放锁  
    }else{    
        insertLog('库存减少失败');  
    }   
}else{  
    insertLog('库存不够');  
}  
fclose($fp);
```
#### 优化方案4：使用redis队列，因为pop操作是原子的，即使有很多用户同时到达，也是依次执行，推荐使用（mysql事务在高并发下性能下降很厉害，文件锁的方式也是）

先将商品库存如队列

```php
<?php  
$store=1000;  
$redis=new Redis();  
$result=$redis->connect('127.0.0.1',6379);  
$res=$redis->llen('goods_store');  
echo $res;  
$count=$store-$res;  
for($i=0;$i<$count;$i++){  
    $redis->lpush('goods_store',1);  
}  
echo $redis->llen('goods_store');  
?>
```

抢购、描述逻辑

```php
<?php  
$conn=mysql_connect("localhost","big","123456");    
if(!$conn){    
    echo "connect failed";    
    exit;    
}   
mysql_select_db("big",$conn);   
mysql_query("set names utf8");  
  
$price=10;  
$user_id=1;  
$goods_id=1;  
$sku_id=11;  
$number=1;  
  
//生成唯一订单号  
function build_order_no(){  
    return date('ymd').substr(implode(NULL, array_map('ord', str_split(substr(uniqid(), 7, 13), 1))), 0, 8);  
}  
//记录日志  
function insertLog($event,$type=0){  
    global $conn;  
    $sql="insert into ih_log(event,type)   
    values('$event','$type')";    
    mysql_query($sql,$conn);    
}  
  
//模拟下单操作  
//下单前判断redis队列库存量  
$redis=new Redis();  
$result=$redis->connect('127.0.0.1',6379);  
$count=$redis->lpop('goods_store');  
if(!$count){  
    insertLog('error:no store redis');  
    return;  
}  
  
//生成订单    
$order_sn=build_order_no();  
$sql="insert into ih_order(order_sn,user_id,goods_id,sku_id,price)   
values('$order_sn','$user_id','$goods_id','$sku_id','$price')";    
$order_rs=mysql_query($sql,$conn);   
  
//库存减少  
$sql="update ih_store set number=number-{$number} where sku_id='$sku_id'";  
$store_rs=mysql_query($sql,$conn);    
if(mysql_affected_rows()){    
    insertLog('库存减少成功');  
}else{    
    insertLog('库存减少失败');  
}
```

```
模拟5000高并发测试
webbench -c 5000 -t 60 http://192.168.1.198/big/index.php
ab -r -n 6000 -c 5000  http://192.168.1.198/big/index.php
```

上述只是简单模拟高并发下的抢购，真实场景要比这复杂很多，很多注意的地方

如抢购页面做成静态的，通过ajax调用接口

再如上面的会导致一个用户抢多个，思路：

需要一个排队队列和抢购结果队列及库存队列。高并发情况，先将用户进入排队队列，用一个线程循环处理从排队队列取出一个用户，判断用户是否已在抢购结果队列，如果在，则已抢购，否则未抢购，库存减1，写数据库，将用户入结果队列。

测试数据表
```
--  
-- 数据库: `big`  
--  
  
-- --------------------------------------------------------  
  
--  
-- 表的结构 `ih_goods`  
--  
  
  
CREATE TABLE IF NOT EXISTS `ih_goods` (  
  `goods_id` int(10) unsigned NOT NULL AUTO_INCREMENT,  
  `cat_id` int(11) NOT NULL,  
  `goods_name` varchar(255) NOT NULL,  
  PRIMARY KEY (`goods_id`)  
) ENGINE=MyISAM  DEFAULT CHARSET=utf8 AUTO_INCREMENT=2 ;  
  
  
--  
-- 转存表中的数据 `ih_goods`  
--  
  
  
INSERT INTO `ih_goods` (`goods_id`, `cat_id`, `goods_name`) VALUES  
(1, 0, '小米手机');  
  
-- --------------------------------------------------------  
  
--  
-- 表的结构 `ih_log`  
--  
  
CREATE TABLE IF NOT EXISTS `ih_log` (  
  `id` int(11) NOT NULL AUTO_INCREMENT,  
  `event` varchar(255) NOT NULL,  
  `type` tinyint(4) NOT NULL DEFAULT '0',  
  `addtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,  
  PRIMARY KEY (`id`)  
) ENGINE=MyISAM DEFAULT CHARSET=utf8 AUTO_INCREMENT=1 ;  
  
--  
-- 转存表中的数据 `ih_log`  
--  
  
  
-- --------------------------------------------------------  
  
--  
-- 表的结构 `ih_order`  
--  
  
CREATE TABLE IF NOT EXISTS `ih_order` (  
  `id` int(11) NOT NULL AUTO_INCREMENT,  
  `order_sn` char(32) NOT NULL,  
  `user_id` int(11) NOT NULL,  
  `status` int(11) NOT NULL DEFAULT '0',  
  `goods_id` int(11) NOT NULL DEFAULT '0',  
  `sku_id` int(11) NOT NULL DEFAULT '0',  
  `price` float NOT NULL,  
  `addtime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,  
  PRIMARY KEY (`id`)  
) ENGINE=InnoDB DEFAULT CHARSET=utf8 COMMENT='订单表' AUTO_INCREMENT=1 ;  
  
--  
-- 转存表中的数据 `ih_order`  
--  
  
  
-- --------------------------------------------------------  
  
--  
-- 表的结构 `ih_store`  
--  
  
CREATE TABLE IF NOT EXISTS `ih_store` (  
  `id` int(11) NOT NULL AUTO_INCREMENT,  
  `goods_id` int(11) NOT NULL,  
  `sku_id` int(10) unsigned NOT NULL DEFAULT '0',  
  `number` int(10) NOT NULL DEFAULT '0',  
  `freez` int(11) NOT NULL DEFAULT '0' COMMENT '虚拟库存',  
  PRIMARY KEY (`id`)  
) ENGINE=InnoDB  DEFAULT CHARSET=utf8 COMMENT='库存' AUTO_INCREMENT=2 ;  
  
--  
-- 转存表中的数据 `ih_store`  
--  
  
INSERT INTO `ih_store` (`id`, `goods_id`, `sku_id`, `number`, `freez`) VALUES  
(1, 1, 11, 500, 0);
```

链接：
http://blog.csdn.net/erjian666/article/details/58585181
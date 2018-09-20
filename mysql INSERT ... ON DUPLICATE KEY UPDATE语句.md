首先这个语法的目的是为了解决重复性，当数据库中存在某个记录时，执行这条语句会更新它，而不存在这条记录时，会插入它。<br>
相当于先判断一条记录是否存在，存在则update，否则insert。其语法是：
```
INSERT INTO tablename(field1,field2, field3, ...) VALUES(value1, value2, value3, ...) ON DUPLICATE KEY UPDATE field1=value1,field2=value2, field3=value3, ...;
```
tablename是表名，field1，field2，field3等是字段名称，value1，value2，value3等是字段值。<br>
例如：
```
INSERT INTO t_stock_chg(f_market, f_stockID, f_name) VALUES('SH', '600000', '白云机场') ON DUPLICATE KEY UPDATE f_market='SH', f_name='浦发银行';
```
这条语句如果记录不存在，插入该记录；存在将f_market改为SH，f_name改为浦发银行。现在有个问题是：这条语句判断该条记录是否存在的标准是什么？由于同一个值是可以同时出现在多个记录中的，所以必须有个字段是唯一不能重复的。

规则是这样的：如果你插入的记录导致一个UNIQUE索引或者primary key(主键)出现重复，那么就会认为该条记录存在，则执行update语句而不是insert语句，反之，则执行insert语句而不是更新语句。所以 ON DUPLICATE KEY UPDATE是不能写where条件的，例如如下语法是错误的：
```
INSERT INTO t_stock_chg(f_market, f_stockID, f_name) VALUES('SH', '600000', '白云机场') ON DUPLICATE KEY UPDATE f_market='SH', f_name='浦发银行' WHERE f_stockID='600000';
```
因为由UNIQUE索引或者主键保证唯一性，不需要WHERE子条件。所以上面的INSERT INTO t_stock_chg(f_market, f_stockID, f_name) VALUES('SH', '600000', '白云机场') ON DUPLICATE KEY UPDATE f_market='SH', f_name='浦发银行';中f_stockID就是唯一主键。

这里特别需要注意的是：如果行作为新记录被插入，则受影响行的值为1；如果原有的记录被更新，则受影响行的值为2，如果更新的数据和已有的数据一模一样，则受影响的行数是0，这意味着不会去更新，也就是说即使你有的时间戳是自动记录最后一次的更新时间，这个时间戳也不会变动。<br>
例如：
```
CREATE TABLE `t_stock_chg` (
  `f_market` varchar(64) NOT NULL COMMENT '市场',
  `f_stockID` varchar(10) NOT NULL DEFAULT '' COMMENT '股票代码',
  `f_updatetime` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP ON UPDATE CURRENT_TIMESTAMP COMMENT '插入时间戳',
  `f_name` varchar(16) DEFAULT NULL COMMENT '股票名称',
  PRIMARY KEY (`f_market`,`f_stockID`)
) ENGINE=MyISAM DEFAULT CHARSET=utf8
```
这里的字段f_updatetime每次在更新数据时会自动更新，但是如果记录中存在某条数据，后来又更新它，而更新的数据和原数据一模一样，那么这个字段也不会更新，仍然是上一次的时间。此时INSERT ... ON DUPLICATE KEY UPDATE影响行数是0。
关于 ON DUPLICATE KEY UPDATE的官方帮助说明在这里：[https://dev.mysql.com/doc/refman/5.5/en/insert-on-duplicate.html](https://dev.mysql.com/doc/refman/5.5/en/insert-on-duplicate.html)

与INSERT ... ON DUPLICATE KEY UPDATE类似的语句还有REPLACE INTO，REPLACE INTO首先尝试插入数据到表中，如果发现表中已经有此行数据（根据主键或者唯一索引判断）则先删除此行数据，然后插入新的数据。否则没有此行数据的话，直接插入新数据。

其实很多使用 REPLACE INTO 的场景，实际上需要的是 INSERT INTO … ON DUPLICATE KEY UPDATE，在正确理解 REPLACE INTO 行为和副作用的前提下，谨慎使用 REPLACE INTO。

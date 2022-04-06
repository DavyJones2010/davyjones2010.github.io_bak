---
layout: post
title: 记一次mysql unique index未生效原因排查
tags: [learn-from-failure, mysql, index, insert-on-duplicate]
lang: zh
---

# 背景描述
如下SQL: serial_number默认可空, 且为unique key的一部分:
```sql
CREATE TABLE `inventory` (
`stockid` int(11) unsigned NOT NULL AUTO_INCREMENT,
`productid` int(11) unsigned NOT NULL,
`factor1` int(11) unsigned NOT NULL,
`factor2` int(11) unsigned NOT NULL,
`factor3` int(11) unsigned NOT NULL,
`factor4` varchar(8) NOT NULL,
`factor5` decimal(10,2) NOT NULL,
`factor6` enum('A','B','C','D','NEW') NOT NULL,
`quantity` int(11) NOT NULL,
`stamp` timestamp NOT NULL DEFAULT CURRENT_TIMESTAMP,
`serial_number` varchar(11) DEFAULT NULL,
PRIMARY KEY (`stockid`),
UNIQUE KEY `serial_number` (`serial_number`),
UNIQUE KEY `productid_2` (`productid`,`factor1`,`factor2`,`factor3`,`factor4`,`factor5`,`factor6`,`serial_number`),
KEY `productid` (`productid`),
KEY `factor1` (`factor1`),
KEY `factor2` (`factor2`),
KEY `factor3` (`factor3`));

INSERT INTO inventory ( productid, factor1, factor2, factor3, factor4, factor5, factor6, quantity)
VALUES (242332,1,1,1,'V67',3.30,'NEW',10);

INSERT INTO inventory ( productid, factor1, factor2, factor3, factor4, factor5, factor6, quantity)
VALUES (242332,1,1,1,'V67',3.30,'NEW',10)
ON DUPLICATE KEY UPDATE `quantity` = VALUES(quantity) + quantity;
```

* 预期结果: 当执行第二条insert的时候, 应该会跟第一条duplicate, 从而执行更新操作.
* 实际结果: 当执行第二条insert的时候, 没有跟第一条duplicate, 也执行了插入操作. 从而出现了两条相同的结果如下:
![img.png](../../assets/img/2021-11-09-mysql-unique-index-not-working-1.png)


# 原因分析
上google上按照关键词搜了下, 发现很多人已经踩过类似的坑:
https://stackoverflow.com/questions/22156301/mysql-unique-key-not-working

> Mysql allows multiple NULLs in an unique constraint.
> 
> In your serial_number column replace NULL with a value and the constraint is triggered,see:
http://sqlfiddle.com/#!2/9dbd19/1
> 
> a UNIQUE index permits multiple NULL values for columns that can contain NULL


# 修改方案
修改ddl如下:
```sql
`serial_number` varchar(11) NOT NULL DEFAULT '',
```

# 最佳实践
* 方案1: 组成unique key的字段, 尽量不允许为空,  最好设置为: NOT NULL DEFAULT ''
* 方案2: 如果无法修改ddl, 那么最好在代码里做兼容, 把null值做个默认值映射, 填充进去.

# 其他
在线执行ddl, dml诊断的系统,  可以
1. 选择不同db类型&版本, 方便重现问题
2. 查看执行计划
3. 分享诊断的永久链接, 方便他人排查:
   http://sqlfiddle.com/
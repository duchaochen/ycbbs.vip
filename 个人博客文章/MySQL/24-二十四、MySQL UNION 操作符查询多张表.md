MySQL `UNION` 操作符用于连接两个以上的 `SELECT` 语句的结果组合到一个结果集合中

多个 SELECT 语句会删除重复的数据

### `UNION` 操作符语法 ###

MySQL `UNION` 操作符的语法格式如下

```
SELECT expression1, expression2, ... expression_n
FROM tables
[WHERE conditions]
UNION [ALL | DISTINCT]
SELECT expression1, expression2, ... expression_n
FROM tables
[WHERE conditions];
```

<table> 
 <thead> 
  <tr> 
   <th align="left">参数</th> 
   <th align="left">说明</th> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td align="left">expression1, expression2, ... expression_n</td> 
   <td align="left">要检索的列</td> 
  </tr> 
  <tr> 
   <td align="left">tables</td> 
   <td align="left">要检索的数据表</td> 
  </tr> 
  <tr> 
   <td align="left">WHERE conditions</td> 
   <td align="left">可选， 检索条件</td> 
  </tr> 
  <tr> 
   <td align="left">DISTINCT</td> 
   <td align="left">可选，删除结果集中重复的数据<br>默认情况下 UNION 操作符已经删除了重复数据<br>所以 DISTINCT 修饰符对结果没啥影响</td> 
  </tr> 
  <tr> 
   <td align="left">ALL</td> 
   <td align="left">可选，返回所有结果集，包含重复数据</td> 
  </tr> 
 </tbody> 
</table>

## 范例数据 ##

可以在 `mysql>` 命令行中运行以下语句填充范例数据

```
DROP TABLE IF EXISTS `tbl_language`;
DROP TABLE IF EXISTS `tbl_rank`;

CREATE TABLE IF NOT EXISTS `tbl_language`(
   `id` INT UNSIGNED AUTO_INCREMENT,
   `name` VARCHAR(64) NOT NULL,
   `url` VARCHAR(128) NOT NULL,
   `founded_at` DATE,
   PRIMARY KEY ( `id` )
)ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

CREATE TABLE IF NOT EXISTS `tbl_rank`(
   `id` INT UNSIGNED AUTO_INCREMENT,
   `name` VARCHAR(64) NOT NULL,
   `month` VARCHAR(7) NOT NULL,
   `rank` TINYINT NOT NULL,
   `rate` VARCHAR(32) NOT NULL,
   PRIMARY KEY ( `id` )
)ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

INSERT INTO `tbl_language` VALUES
    (1,'Python','https://www.ycbbs.vip','1991-2-20'),
    (2,'PHP','http://www.php.net','1994-1-1'),
    (3,'Ruby','https://www.ruby-lang.org/','1996-12-25');

INSERT INTO `tbl_rank` VALUES
    (1, 'Python','2018-04',4,'5.083%'),
    (2, 'PHP','2018-04',6,'4.218%'),
    (3, 'Ruby','2018-04',11,'2.018%'),
    (4, 'Java','2018-04',1,'15.777%'),
    (5, 'Python','2018-03',4,'5.869%'),
    (6, 'PHP','2018-03',7,'4.010%'),
    (7, 'Ruby','2018-03',12,'2.744%'),
    (8, 'Java','2018-03',1,'14.941'),
    (9, 'Python','2018-02',4,'5.168%'),
    (10, 'PHP','2018-02',7,'3.420%'),
    (11, 'Ruby','2018-02',10,'2.534%'),
    (12, 'Java','2018-02',1,'14.988%');
```

`tbl_language` 表中的数据如下

```
+----+--------+----------------------------+------------+
| id | name   | url                        | founded_at |
+----+--------+----------------------------+------------+
|  1 | Python | https://www.ycbbs.vip        | 1991-02-20 |
|  2 | PHP    | http://www.php.net         | 1994-01-01 |
|  3 | Ruby   | https://www.ruby-lang.org/ | 1996-12-25 |
+----+--------+----------------------------+------------+
```

`tbl_rank` 表中的数据如下

```
+----+--------+---------+------+---------+
| id | name   | month   | rank | rate    |
+----+--------+---------+------+---------+
|  1 | Python | 2018-04 |    4 | 5.083%  |
|  2 | PHP    | 2018-04 |    6 | 4.218%  |
|  3 | Ruby   | 2018-04 |   11 | 2.018%  |
|  4 | Java   | 2018-04 |    1 | 15.777% |
|  5 | Python | 2018-03 |    4 | 5.869%  |
|  6 | PHP    | 2018-03 |    7 | 4.010%  |
|  7 | Ruby   | 2018-03 |   12 | 2.744%  |
|  8 | Java   | 2018-03 |    1 | 14.941  |
|  9 | Python | 2018-02 |    4 | 5.168%  |
| 10 | PHP    | 2018-02 |    7 | 3.420%  |
| 11 | Ruby   | 2018-02 |   10 | 2.534%  |
| 12 | Java   | 2018-02 |    1 | 14.988% |
+----+--------+---------+------+---------+
```

## SQL UNION ##

下面的 SQL 语句从 `tbl_language` 和 `tbl_rank` 表中选取所有 **不同的** name（只有不同的值）

```
SELECT name FROM tbl_language UNION SELECT name FROM tbl_rank  ORDER BY name;
```

执行以上 SQL 输出结果如下

```
MariaDB [ycbbs]> SELECT name FROM tbl_language UNION SELECT name FROM tbl_rank  ORDER BY name;
+--------+
| name   |
+--------+
| Java   |
| PHP    |
| Python |
| Ruby   |
+--------+
4 rows in set (0.00 sec)
```

UNION 不能用于列出两个表中所有的 `name` ，UNION 只会选取不同的值

如果要选取重复的值，可以使用 `UNION ALL`

## SQL UNION ALL ##

下面的 SQL 语句从 `tbl_language` 和 `tbl_rank` 表中选取所有的 name（可能有重复的值）

```
SELECT name FROM tbl_language UNION ALL SELECT name FROM tbl_rank  ORDER BY name;
```

执行以上 SQL 输出结果如下

```
MariaDB [ycbbs]> SELECT name FROM tbl_language UNION ALL SELECT name FROM tbl_rank  ORDER BY name;
+--------+
| name   |
+--------+
| Java   |
| Java   |
| Java   |
| PHP    |
| PHP    |
| PHP    |
| PHP    |
| Python |
| Python |
| Python |
| Python |
| Ruby   |
| Ruby   |
| Ruby   |
| Ruby   |
+--------+
15 rows in set (0.00 sec)
```

## 带有 WHERE 的 SQL UNION ALL ##

下面的 SQL 语句从 `tbl_language` 和 `tbl_rank` 表中选取所有的 `Python`（可能有重复的值）

```
SELECT name FROM tbl_language WHERE name="Python" UNION ALL SELECT name FROM tbl_rank WHERE name="Python"  ORDER BY name;
```

执行以上 SQL 输出结果如下

```
MariaDB [ycbbs]> SELECT name FROM tbl_language WHERE name="Python" UNION ALL SELECT name FROM tbl_rank WHERE name="Python"  ORDER BY name;
+--------+
| name   |
+--------+
| Python |
| Python |
| Python |
| Python |
+--------+
4 rows in set (0.00 sec)
```

希望读者能够给小编留言，也可以点击[此处扫下面二维码关注微信公众号](https://www.ycbbs.vip/?p=28 "此处扫下面二维码关注微信公众号")
我们在创建表的时候可以让某个字段为空，比如下面的创建 `tbl_language` 的语句

```
CREATE TABLE IF NOT EXISTS `tbl_language`(
   `id` INT UNSIGNED AUTO_INCREMENT,
   `name` VARCHAR(64) NOT NULL,
   `url` VARCHAR(128) NOT NULL,
   `founded_at` DATE,
   PRIMARY KEY ( `id` )
)ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

因为 `founded_at` 没有明确指明 `NOT NULL` ，所以它是可以为空的

然后我们运行下面的语句插入数据

```
truncate `tbl_language`;
INSERT INTO `tbl_language` VALUES
    (1,'Python','https://www.ycbbs.vip','1991-2-20'),
    (2,'PHP','http://www.php.net','1994-1-1'),
    (3,'Ruby','https://www.ruby-lang.org/','1996-12-25'),
    (4,'Kotlin','http://kotlinlang.org/','2016-02-17');

INSERT INTO `tbl_language` (name,url) VALUES
    ('Perl','http://www.perl.org/'),
    ('Scala','http://www.scala-lang.org/');
```

使用不带 `WHERE` 的 `SELECT` 语句可以看到 6 条记录

```
MariaDB [ycbbs]> SELECT * FROM  `tbl_language`;
+----+--------+----------------------------+------------+
| id | name   | url                        | founded_at |
+----+--------+----------------------------+------------+
|  1 | Python | https://www.ycbbs.vip        | 1991-02-20 |
|  2 | PHP    | http://www.php.net         | 1994-01-01 |
|  3 | Ruby   | https://www.ruby-lang.org/ | 1996-12-25 |
|  4 | Kotlin | http://kotlinlang.org/     | 2016-02-17 |
|  5 | Perl   | http://www.perl.org/       | NULL       |
|  6 | Scala  | http://www.scala-lang.org/ | NULL       |
+----+--------+----------------------------+------------+
```

使用下面的语句查找 `founded_at` 为 `NULL` 的数据

```
SELECT * FROM  `tbl_language` WHERE founded_at = NULL;
```

运行结果显示 0 条记录

```
MariaDB [ycbbs]> SELECT * FROM  `tbl_language` WHERE founded_at = NULL;
Empty set (0.01 sec)
```

我们要怎么把 `founded_at` 为 `NULL` 的查找出来呢？

## MYSQL NULL ##

`WHERE` 子句中的 `NULL` 值时比较特殊的

我们不能使用 `= NULL` 或 `!= NULL` 在列中查找 `NULL` 值

MySQL 中，`NULL` 值与任何其它值的比较 ( 即使是 `NULL` ) 永远返回 false，即 `NULL = NULL` 返回 false

为了处理这种情况，MySQL 提供了三大运算符用来处理 `NULL` 值的情况

<table> 
 <thead> 
  <tr> 
   <th align="left">运算符</th> 
   <th align="left">说明</th> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td align="left">IS NULL</td> 
   <td align="left">当列的值是 NULL,此运算符返回 true</td> 
  </tr> 
  <tr> 
   <td align="left">IS NOT NULL</td> 
   <td align="left">当列的值不为 NULL, 运算符返回 true</td> 
  </tr> 
  <tr> 
   <td align="left"><=></td> 
   <td align="left">比较操作符（不同于 <code>=</code> 运算符），当比较的的两个值为 NULL 时返回 true</td> 
  </tr> 
 </tbody> 
</table>

我们经常使用的就是 `IS NULL` 和 `IS NOT NULL` 运算符

## 在命令提示符中使用 NULL 值 ##

既然我们已经掌握了 `MySQL` 中如何使用 `NULL`，那么查询 `founded_at` 为 `NULL` 的记录就非常简单了

```
SELECT * FROM  `tbl_language` WHERE founded_at is NULL;
```

运行结果如下

```
MariaDB [ycbbs]> SELECT * FROM  `tbl_language` WHERE founded_at is NULL;
+----+-------+----------------------------+------------+
| id | name  | url                        | founded_at |
+----+-------+----------------------------+------------+
|  5 | Perl  | http://www.perl.org/       | NULL       |
|  6 | Scala | http://www.scala-lang.org/ | NULL       |
+----+-------+----------------------------+------------+
2 rows in set (0.00 sec)
```

哇，查出来了，查出来了，有两条

然后可以使用下面的语句查找 `founded_at` 不是 `NULL` 的记录

```
SELECT * FROM  `tbl_language` WHERE founded_at is NOT NULL;
```

运行结果如下

```
MariaDB [ycbbs]> SELECT * FROM  `tbl_language` WHERE founded_at is NOT NULL;
+----+--------+----------------------------+------------+
| id | name   | url                        | founded_at |
+----+--------+----------------------------+------------+
|  1 | Python | https://www.ycbbs.vip        | 1991-02-20 |
|  2 | PHP    | http://www.php.net         | 1994-01-01 |
|  3 | Ruby   | https://www.ruby-lang.org/ | 1996-12-25 |
|  4 | Kotlin | http://kotlinlang.org/     | 2016-02-17 |
+----+--------+----------------------------+------------+
4 rows in set (0.00 sec)
```

> **注意：** 因为 `NULL` 只有简单的判断操作，所以在日常开发中，创建字段时应该使用 `NOT NULL` 来避免数据为 `NULL` 的情况

## 在 PHP 脚本中使用处理 NULL 值 ##

在 PHP 脚本中使用处理 NULL 值有两种方法：

1、  把所有的数据查找出来，然后一条一条去判断是否为 `null`
2、  在 SQL 语句中使用 `is NULL` 或者 `is NOT NULL` 来筛选

PHP 可以使用 `PDO::query()` 函数来查询某个表中的数据

#### PDO::query() 函数原型 ####

`PDO::query()` 有四个函数重载

```
PDOStatement PDO::query ( string $statement )

PDOStatement PDO::query ( string $statement , int $PDO::FETCH_COLUMN , int $colno )

PDOStatement PDO::query ( string $statement , int $PDO::FETCH_CLASS , string $classname , array $ctorargs )

PDOStatement PDO::query ( string $statement , int $PDO::FETCH_INTO , object $object )
```

如果成功，`PDO::query()` 返回 `PDOStatement` 对象，如果失败返回 FALSE

#### 参数 ####

<table> 
 <thead> 
  <tr> 
   <th align="left">参数</th> 
   <th align="left">说明</th> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td align="left">statement</td> 
   <td align="left">要被预处理和执行的 SQL 语句，查询中的数据应该被妥善地转义</td> 
  </tr> 
 </tbody> 
</table>

第二个参数有以下几个可选值，默认为 `PDO::FETCH_BOTH`

<table> 
 <thead> 
  <tr> 
   <th align="left">值</th> 
   <th align="left">说明</th> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td align="left">PDO::FETCH_ASSOC</td> 
   <td align="left">返回一个索引为结果集列名的数组</td> 
  </tr> 
  <tr> 
   <td align="left">PDO::FETCH_BOTH</td> 
   <td align="left">默认，返回一个索引为结果集列名和以0开始的列号的数组</td> 
  </tr> 
  <tr> 
   <td align="left">PDO::FETCH_BOUND</td> 
   <td align="left">返回 TRUE ，并分配结果集中的列值给 PDOStatement::bindColumn() 方法绑定的 PHP 变量</td> 
  </tr> 
  <tr> 
   <td align="left">PDO::FETCH_CLASS</td> 
   <td align="left">返回一个请求类的新实例，映射结果集中的列名到类中对应的属性名。如果 fetch_style 包含 PDO::FETCH_CLASSTYPE（例如：PDO::FETCH_CLASS |PDO::FETCH_CLASSTYPE），则类名由第一列的值决定</td> 
  </tr> 
  <tr> 
   <td align="left">PDO::FETCH_INTO</td> 
   <td align="left">更新一个被请求类已存在的实例，映射结果集中的列到类中命名的属性</td> 
  </tr> 
  <tr> 
   <td align="left">PDO::FETCH_LAZY</td> 
   <td align="left">结合使用 PDO::FETCH_BOTH 和 PDO::FETCH_OBJ，创建供用来访问的对象变量名</td> 
  </tr> 
  <tr> 
   <td align="left">PDO::FETCH_NUM</td> 
   <td align="left">返回一个索引为以0开始的结果集列号的数组</td> 
  </tr> 
  <tr> 
   <td align="left">PDO::FETCH_OBJ</td> 
   <td align="left">返回一个属性名对应结果集列名的匿名对象</td> 
  </tr> 
 </tbody> 
</table>

我们使用默认的 `PDO::FETCH_BOTH` 获取所有数据，其它方式请移步我们的 PHP 基础教程

```
<?php 

$dbh = new PDO('mysql:host=127.0.0.1;dbname=ycbbs', 'root', '');    


// 把所有的数据查出来
$sql= "SELECT * FROM `tbl_language`";

$stmt = $dbh->query($sql);

foreach($stmt as $row)
{
    // 判断是否为空，如果为空则输出数据
    if ( $row['founded_at'] === NULL ) {
        print_r($row);
    }
}

echo "\n------------------------------------------\n";


// 只查找 founded_at 为 NULL 的数据
$sql= "SELECT * FROM `tbl_language` WHERE founded_at is NULL";

$stmt = $dbh->query($sql);

foreach($stmt as $row)
{
    print_r($row);
}
```

输出结果如下

```
$ php main.php
Array
(
    [id] => 5
    [0] => 5
    [name] => Perl
    [1] => Perl
    [url] => http://www.perl.org/
    [2] => http://www.perl.org/
    [founded_at] => 
    [3] => 
)
Array
(
    [id] => 6
    [0] => 6
    [name] => Scala
    [1] => Scala
    [url] => http://www.scala-lang.org/
    [2] => http://www.scala-lang.org/
    [founded_at] => 
    [3] => 
)

------------------------------------------
Array
(
    [id] => 5
    [0] => 5
    [name] => Perl
    [1] => Perl
    [url] => http://www.perl.org/
    [2] => http://www.perl.org/
    [founded_at] => 
    [3] => 
)
Array
(
    [id] => 6
    [0] => 6
    [name] => Scala
    [1] => Scala
    [url] => http://www.scala-lang.org/
    [2] => http://www.scala-lang.org/
    [founded_at] => 
    [3] => 
)
```

两者数据的记录一模一样

希望读者能够给小编留言，也可以点击[此处扫下面二维码关注微信公众号](https://www.ycbbs.vip/?p=28 "此处扫下面二维码关注微信公众号")
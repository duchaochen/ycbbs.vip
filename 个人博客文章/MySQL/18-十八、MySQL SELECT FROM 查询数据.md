MySQL 使用 `SELECT FROM` SQL 语句来查询表中的数据

### `SELECT FROM` SQL 语句语法 ###

使用 `SELECT FROM` SQL 语句查询表中数据的语法格式如下

```
SELECT column_name,column_name
FROM table_name
[WHERE Clause]
[LIMIT N,M]
```

1、  查询语句中可以使用一个或者多个表，表之间使用逗号(,) 分隔，并使用 `WHERE` 语句来设定查询条件
    
```
    SELECT a.id,b.name FROM a,b WHERE a.id=b.id;
```
2、  SELECT 命令可以读取一条或者多条记录
3、  可以使用星号（`*`）来代替 `column_name` ，但这会返回表的所有字段数据
    
```
    SELECT * FROM tbl_language;
```
4、  可以使用 `WHERE` 子句来有条件的查询数据
    
```
    SELECT * FROM tbl_language WHERE name = 'Python';
```
5、  可以使用 `LIMIT` 子句来设定返回的记录数
    
```
    SELECT * FROM tbl_language LIMIT 1;
```
6、  可以通过 `LIMIT` 字句指定开始查询的数据偏移量
    
```
    SELECT * FROM tbl_language LIMIT 1,2;
```

> 偏移量从 0 开始计算， 0 表示第一条， 1 表示第二条
> 
> **注意：** LIMIT 有一个参数和两个参数的情况：
> 
> 1、  一个参数表示限制返回记录的条数
> 2、  两个参数，第一个表示数据偏移量，第二个表示返回记录的条数

## 插入范例数据 ##

可以在 `mysql>` 命令提示窗口中执行以下语句插入范例数据

```
truncate tbl_language;
INSERT INTO `tbl_language` VALUES
    (1,'Python','https://www.ycbbs.vip','1991-2-20'),
    (2,'PHP','http://www.php.net','1994-1-1'),
    (3,'Ruby','https://www.ruby-lang.org/','1996-12-25');
```

## 通过命令提示符获取数据 ##

可以在 `mysql>` 命令提示窗口中执行 `SELECT FROM` SQL 语句查询某个表中的数据

下面的代码使用 `SELECT FROM` SQL 语句查询表 `tbl_language` 中所有的数据

```
MariaDB [ycbbs]> SELECT * FROM tbl_language;
+----+--------+----------------------------+------------+
| id | name   | url                        | founded_at |
+----+--------+----------------------------+------------+
|  1 | Python | https://www.ycbbs.vip        | 1991-02-20 |
|  2 | PHP    | http://www.php.net         | 1994-01-01 |
|  3 | Ruby   | https://www.ruby-lang.org/ | 1996-12-25 |
+----+--------+----------------------------+------------+
3 rows in set (0.00 sec)
```

使用下面的语句查询表中 `name` 和 `url` 字段

```
MariaDB [ycbbs]> SELECT name,url FROM tbl_language;
+--------+----------------------------+
| name   | url                        |
+--------+----------------------------+
| Python | https://www.ycbbs.vip        |
| PHP    | http://www.php.net         |
| Ruby   | https://www.ruby-lang.org/ |
+--------+----------------------------+
3 rows in set (0.00 sec)
```

使用下面的语句查询表中 `name` 为 `Python` 的数据

```
MariaDB [ycbbs]> SELECT * FROM tbl_language WHERE name='Python';
+----+--------+---------------------+------------+
| id | name   | url                 | founded_at |
+----+--------+---------------------+------------+
|  1 | Python | https://www.ycbbs.vip | 1991-02-20 |
+----+--------+---------------------+------------+
1 row in set (0.01 sec)
```

使用下面的语句查询表 `tbl_language` 中所有的数据,但只返回 1 条数据

```
MariaDB [ycbbs]> SELECT * FROM tbl_language LIMIT 1;
+----+--------+---------------------+------------+
| id | name   | url                 | founded_at |
+----+--------+---------------------+------------+
|  1 | Python | https://www.ycbbs.vip | 1991-02-20 |
+----+--------+---------------------+------------+
1 row in set (0.01 sec)
```

使用下面的语句查询表 `tbl_language` 中所有的数据,但只返回第二条开始的 1 条数据，也就是第二条

> 注意偏移量从 0 开始计算， 0 表示第一条， 1 表示第二条

```
MariaDB [ycbbs]> SELECT * FROM tbl_language LIMIT 1,1;
+----+------+--------------------+------------+
| id | name | url                | founded_at |
+----+------+--------------------+------------+
|  2 | PHP  | http://www.php.net | 1994-01-01 |
+----+------+--------------------+------------+
1 row in set (0.00 sec)
```

## 使用 PHP 脚本获取数据 ##

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

/*
 * filename: main.php
 * author: 研发军团(www.ycbbs.vip)
 * 
 * Copyright © 2015-2065 www.ycbbs.vip. All rights reserved.
 */

$sql= "SELECT * FROM tbl_language;";

try {


    $dbh = new PDO('mysql:host=127.0.0.1;dbname=ycbbs', 'root', '');    

    $stmt = $dbh->query($sql);

    foreach($stmt as $row)
    {
        var_dump($row);
    }
}
catch (PDOException $e) 
{    
    echo "错误!: " , $e->getMessage() , "\n";  
}
```

输出结果如下

```
$ php main.php
array(8) {
  ["id"]=>
  string(1) "1"
  [0]=>
  string(1) "1"
  ["name"]=>
  string(6) "Python"
  [1]=>
  string(6) "Python"
  ["url"]=>
  string(19) "https://www.ycbbs.vip"
  [2]=>
  string(19) "https://www.ycbbs.vip"
  ["founded_at"]=>
  string(10) "1991-02-20"
  [3]=>
  string(10) "1991-02-20"
}
array(8) {
  ["id"]=>
  string(1) "2"
  [0]=>
  string(1) "2"
  ["name"]=>
  string(3) "PHP"
  [1]=>
  string(3) "PHP"
  ["url"]=>
  string(18) "http://www.php.net"
  [2]=>
  string(18) "http://www.php.net"
  ["founded_at"]=>
  string(10) "1994-01-01"
  [3]=>
  string(10) "1994-01-01"
}
array(8) {
  ["id"]=>
  string(1) "3"
  [0]=>
  string(1) "3"
  ["name"]=>
  string(4) "Ruby"
  [1]=>
  string(4) "Ruby"
  ["url"]=>
  string(26) "https://www.ruby-lang.org/"
  [2]=>
  string(26) "https://www.ruby-lang.org/"
  ["founded_at"]=>
  string(10) "1996-12-25"
  [3]=>
  string(10) "1996-12-25"
}
```


希望读者能够给小编留言，也可以点击[此处扫下面二维码关注微信公众号](https://www.ycbbs.vip/?p=28 "此处扫下面二维码关注微信公众号")
前面章节中我们已经了解到 `MySQL` 可以通过 `LIKE ...%` 子句来进行模糊匹配，但这都只是简单的模糊查询，也是速度最快的模糊查询

除此之外，`MySQL` 同样也支持其它正则表达式的匹配

`MySQL` 通过使用 `REGEXP` 操作符来进行正则表达式匹配

如果你了解过其它语言的正则表达式，比如 `PHP` 或 `Perl` 等，那么你会对 `MySQL` 的正则表达式元字符非常熟悉，因为它们都类似

`MySQL` `REGEXP` 操作符支持以下几种元子符

<table> 
 <thead> 
  <tr> 
   <th align="left">元字符</th> 
   <th align="left">描述</th> 
  </tr> 
 </thead> 
 <tbody> 
  <tr> 
   <td align="left">^</td> 
   <td align="left">匹配输入字符串的开始位置<br>如果设置了 <code>Multiline</code> 属性，^ 也匹配 '\n' 或 '\r' 之后的位置</td> 
  </tr> 
  <tr> 
   <td align="left">$</td> 
   <td align="left">匹配输入字符串的结束位置<br>如果设置了 <code>Multiline</code> 属性，$ 也匹配 '\n' 或 '\r' 之前的位置</td> 
  </tr> 
  <tr> 
   <td align="left">.</td> 
   <td align="left">匹配除 "\n" 之外的任何单个字符<br>如果要匹配包括 '\n' 在内的任何字符，请使用象 '[.\n]' 的模式</td> 
  </tr> 
  <tr> 
   <td align="left">[...]</td> 
   <td align="left">字符集合。匹配所包含的任意一个字符<br>例如， '[abc]' 可以匹配 "plain" 中的 'a'</td> 
  </tr> 
  <tr> 
   <td align="left">[^...]</td> 
   <td align="left">负值字符集合。匹配未包含的任意字符<br>例如， '[^abc]' 可以匹配 "plain" 中的'p'</td> 
  </tr> 
  <tr> 
   <td align="left">p1|p2|p3</td> 
   <td align="left">匹配 p1 或 p2 或 p3<br>例如，'z|food' 匹配 "z" 或 "food"。'(z|f)ood' 则匹配 "zood" 或 "food"</td> 
  </tr> 
  <tr> 
   <td align="left">*</td> 
   <td align="left">匹配前面的子表达式零次或多次<br>例如，zo<em> 能匹配 "z" 以及 "zoo"。</em> 等价于{0,}。</td> 
  </tr> 
  <tr> 
   <td align="left">+</td> 
   <td align="left">匹配前面的子表达式一次或多次<br>例如，'zo+' 能匹配 "zo" 以及 "zoo"，但不能匹配 "z"。+ 等价于 </td> 
  </tr> 
  <tr> 
   <td align="left">{n}</td> 
   <td align="left">n 是一个非负整数。匹配确定的 n 次<br>例如，'o{2}' 不能匹配 "Bob" 中的 'o'，但是能匹配 "food" 中的两个 o</td> 
  </tr> 
  <tr> 
   <td align="left">{n,m}</td> 
   <td align="left">m 和 n 均为非负整数，其中n <= m。最少匹配 n 次且最多匹配 m 次</td> 
  </tr> 
 </tbody> 
</table>

在 `MySQL` 中正则表达式用的不多，但也有那么几个时刻还是很有用处的

下面我们就拿几个伪需求来看看如何使用

> 说是伪需求，是因为除了全文检索，其实都可以用 LIKE 语句代替

## 测试数据 ##

首先运行下面的 SQL 语句准备测试数据

```
DROP TABLE IF EXISTS `tbl_language`;

CREATE TABLE IF NOT EXISTS `tbl_language`(
   `id` INT UNSIGNED AUTO_INCREMENT,
   `name` VARCHAR(64) NOT NULL,
   `url` VARCHAR(128) NOT NULL,
   `founded_at` DATE,
   PRIMARY KEY ( `id` )
)ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;

INSERT INTO `tbl_language` VALUES
    (1,'Python','https://www.ycbbs.vip','1991-2-20'),
    (2,'PHP','http://www.php.net','1994-1-1'),
    (3,'Ruby','https://www.ruby-lang.org/','1996-12-25'),
    (4,'Kotlin','http://kotlinlang.org/','2016-02-17');

INSERT INTO `tbl_language` (name,url) VALUES
    ('Perl','http://www.perl.org/'),
    ('Scala','http://www.scala-lang.org/');
```

使用 `SELECT * FROM tbl_language` 显示数据如下

```
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

## 范例 ##

1、  查找 `name` 字段中以 `Py` 为开头的所有 `name`
    
```
    SELECT name FROM tbl_language WHERE name REGEXP '^Py';
```
    
    运行结果如下
    
```
    +--------+
    | name   |
    +--------+
    | Python |
    +--------+
```
2、  查找 `url` 字段中以 `org/` 结尾的所有 `name`
    
```
    SELECT name FROM tbl_language WHERE url REGEXP 'org/$';
```
    
    运行结果如下
    
```
    +--------+
    | name   |
    +--------+
    | Ruby   |
    | Kotlin |
    | Perl   |
    | Scala  |
    +--------+
```
3、  查找 `url` 字段中包含 `lang` 字符串的所有 `name`
    
```
    SELECT name FROM tbl_language WHERE url REGEXP 'lang';
```
    
    运行结果如下
    
```
    +--------+
    | name   |
    +--------+
    | Ruby   |
    | Kotlin |
    | Scala  |
    +--------+
```
4、  来一个复杂的，查找 `url` 字段中包含 `-lan` 且以 `rg/` 结尾的所有 `name`
    
```
    SELECT name FROM tbl_language WHERE url REGEXP '-lan.*rg/$';
```
    
    运行结果如下
    
```
    +-------+
    | name  |
    +-------+
    | Ruby  |
    | Scala |
    +-------+
```

希望读者能够给小编留言，也可以点击[此处扫下面二维码关注微信公众号](https://www.ycbbs.vip/?p=28 "此处扫下面二维码关注微信公众号")
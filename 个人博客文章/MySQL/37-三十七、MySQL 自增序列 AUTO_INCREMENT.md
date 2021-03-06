`MySQL` 自增序列是一组整数：1, 2, 3, ...

一张数据表只能有一个自增主键

如果你想实现其它字段也实现自动增加，可以使用 `MySQL` 序列来实现

## AUTO\_INCREMENT ##

`MySQL` 定义序列最简单的方法就是使用 `AUTO_INCREMENT` 来定义列

比如我们前面创建 `tbl_language` 表的语句中就把 `id` 设定为一个自增主键

```
CREATE TABLE IF NOT EXISTS `tbl_language`(
   `id` INT UNSIGNED AUTO_INCREMENT,
   `name` VARCHAR(64) NOT NULL,
   `url` VARCHAR(128) NOT NULL,
   `founded_at` DATE,
   PRIMARY KEY ( `id` )
)ENGINE=InnoDB DEFAULT CHARSET=utf8mb4;
```

那么在插入数据时无需指定字段 `id` 的值，每插一条数据，它就会自增 1

```
MariaDB [ycbbs]> TRUNCATE tbl_language;
Query OK, 0 rows affected (0.02 sec)

MariaDB [ycbbs]> INSERT INTO `tbl_language` (name,url,founded_at) VALUES ('Python','https://www.ycbbs.vip','1991-2-20');
Query OK, 1 row affected (0.01 sec)

MariaDB [ycbbs]> INSERT INTO `tbl_language` (name,url,founded_at) VALUES ('PHP','http://www.php.net','1994-1-1');
Query OK, 1 row affected (0.01 sec)

MariaDB [ycbbs]> INSERT INTO `tbl_language` (name,url,founded_at) VALUES ('Ruby','https://www.ruby-lang.org/','1996-12-25');
Query OK, 1 row affected (0.01 sec)

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

可以看到每条数据的 `id` 值都会自增

## 重置序列 ##

如果删除了数据表中的多条记录，并希望对剩下数据的 `AUTO_INCREMENT` 列进行重新排列，那么可以通过删除自增的列，然后重新添加来实现

不过该操作要非常小心，如果在删除的同时又有新记录添加，有可能会出现数据混乱

```
MariaDB [ycbbs]> ALTER TABLE tbl_language DROP id;
MariaDB [ycbbs]> ALTER TABLE tbl_language
    -> ADD id INT UNSIGNED NOT NULL AUTO_INCREMENT FIRST,
    -> ADD PRIMARY KEY (id);
```

> 这个操作方法非常危险，我们不推荐使用，尤其是多表情况下，会造成数据混乱
> 
> 但是，处女座的我们，真的，真的是不能容忍有没有

## 设置序列的开始值 ##

默认情况下，`AUTO_INCREMENT` 列的默认初始值为 `1`

如果需要指定其它值，比如 1000, 那么可以通过添加 `auto_increment=1000` 来实现，就像下面这样

```
CREATE TABLE IF NOT EXISTS `tbl_language`(
   `id` INT UNSIGNED AUTO_INCREMENT,
   `name` VARCHAR(64) NOT NULL,
   `url` VARCHAR(128) NOT NULL,
   `founded_at` DATE,
   PRIMARY KEY ( `id` )
)ENGINE=InnoDB auto_increment=1000 DEFAULT CHARSET=utf8mb4;
```

或者可以在表创建成功后，通过运行下面的语句来实现

```
ALTER TABLE tbl_language AUTO_INCREMENT = 100;
```

## AUTO\_INCREMENT 保存在哪里 ? ##

`AUTO_INCREMENT` 是表的元数据，表的元数据都保存在 `information_schema.tables` 表里

我们使用下面的语句查看下 `information_schema.tables` 表有什么字段

```
MariaDB [ycbbs]> DESC information_schema.tables;
+-----------------+---------------------+------+---------|
| Field           | Type                | Null | Default |
+-----------------+---------------------+------+---------+
| TABLE_CATALOG   | varchar(512)        | NO   |         |
| TABLE_SCHEMA    | varchar(64)         | NO   |         |
| TABLE_NAME      | varchar(64)         | NO   |         |
| TABLE_TYPE      | varchar(64)         | NO   |         |
| ENGINE          | varchar(64)         | YES  | NULL    |
| VERSION         | bigint(21) unsigned | YES  | NULL    |
| ROW_FORMAT      | varchar(10)         | YES  | NULL    |
| TABLE_ROWS      | bigint(21) unsigned | YES  | NULL    |
| AVG_ROW_LENGTH  | bigint(21) unsigned | YES  | NULL    |
| DATA_LENGTH     | bigint(21) unsigned | YES  | NULL    |
| MAX_DATA_LENGTH | bigint(21) unsigned | YES  | NULL    |
| INDEX_LENGTH    | bigint(21) unsigned | YES  | NULL    |
| DATA_FREE       | bigint(21) unsigned | YES  | NULL    |
| AUTO_INCREMENT  | bigint(21) unsigned | YES  | NULL    |
| CREATE_TIME     | datetime            | YES  | NULL    |
| UPDATE_TIME     | datetime            | YES  | NULL    |
| CHECK_TIME      | datetime            | YES  | NULL    |
| TABLE_COLLATION | varchar(32)         | YES  | NULL    |
| CHECKSUM        | bigint(21) unsigned | YES  | NULL    |
| CREATE_OPTIONS  | varchar(2048)       | YES  | NULL    |
| TABLE_COMMENT   | varchar(2048)       | NO   |         |
+-----------------+---------------------+------+---------+
```

可以看到 `TABLE_NAME` 和 `AUTO_INCREMENT` 两列

我们使用下面的语句来查看下 `tbl_language` 表的元数据

```
SELECT * FROM information_schema.tables WHERE TABLE_NAME = 'tbl_language'\G;
```

运行结果如下

```
MariaDB [ycbbs]> SELECT * FROM information_schema.tables WHERE TABLE_NAME = 'tbl_language'\G;
*********************** 1. row ***********************
  TABLE_CATALOG: def
   TABLE_SCHEMA: ycbbs
     TABLE_NAME: tbl_language
     TABLE_TYPE: BASE TABLE
         ENGINE: InnoDB
        VERSION: 10
     ROW_FORMAT: Dynamic
     TABLE_ROWS: 3
 AVG_ROW_LENGTH: 5461
    DATA_LENGTH: 16384
MAX_DATA_LENGTH: 0
   INDEX_LENGTH: 0
      DATA_FREE: 0
 AUTO_INCREMENT: 4
    CREATE_TIME: 2018-04-09 08:58:12
    UPDATE_TIME: 2018-04-09 09:09:10
     CHECK_TIME: NULL
TABLE_COLLATION: utf8mb4_general_ci
       CHECKSUM: NULL
 CREATE_OPTIONS: 
  TABLE_COMMENT: 
1 rows in set (0.01 sec)
```

可以看到自增值为 4 ，而数据只有 3，哈哈，还记得创建表时默认为 1 吗？

它其实是先取值，然后再自增 +1


希望读者能够给小编留言，也可以点击[此处扫下面二维码关注微信公众号](https://www.ycbbs.vip/?p=28 "此处扫下面二维码关注微信公众号")
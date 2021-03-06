## 引言
`Java 8` 似乎也对 `java.text.SimpleDateFormat` 也不太满意，竟然重新创建了一个 `java.time.format` 包，该包下包含了几个类和枚举用于格式化日期时间。

`java.time.format` 包
`java.time.forma`t 包提供了以下几个类用于格式化日期时间

<table>
	<tr><td>类</td><td>说明</td></tr>
	<tr><td>DateTimeFormatter</td><td>用于打印和解析日期时间对象的格式化程序</td></tr>
	<tr><td>DateTimeFormatterBuilder</td><td>创建日期时间格式化样式的构建器</td></tr>
	<tr><td>DecimalStyle</td><td>日期和时间格式中使用的本地化十进制样式</td></tr>
</table>

`java.time.forma`t 包还提供了以下几个枚举，包含了常见的几种日期时间格式。
<table>
	<tr><td>枚举</td><td>说明</td></tr>
	<tr><td>FormatStyle</td><td>包含了本地化日期，时间或日期时间格式器的样式的枚举</td></tr>
	<tr><td>ResolverStyle</td><td>包含了解决日期和时间的不同方法的枚举</td></tr>
	<tr><td>SignStyle</td><td>包含了如何处理正/负号的方法的枚举</td></tr>
	<tr><td>TextStyle</td><td>包含了文本格式和解析的样式的枚举</td></tr>
</table>

## DateTimeFormatter 类
`DateTimeFormatter` 类格式化日期时间的最重要的类，该类是一个最终类，只能实例化，不能被扩展和继承。

`DateTimeFormatter` 类的定义如下

```public final class DateTimeFormatter extends Object```
`DateTimeFormatter` 类用于打印和解析日期时间对象的格式化器。此类提供打印和解析的主要应用程序入口点，并提供 `DateTimeFormatter` 的常见模式

使用预定义的常量，比如 `ISO_LOCAL_DATE`
使用模式字母，例如 `uuuu-MMM-dd`
使用本地化样式，例如 `long` 或 `medium`
所有的日期时间类，包括本地日期时间和包含时区的日期时间类，都提供了两个重要的方法

1、 一个用于格式化，`format(DateTimeFormatter formatter)`
2、 另一个用于解析，`parse(CharSequence text, DateTimeFormatter formatter)`

下面，我们写几个示例来演示下这两个方法，并演示下如和使用 `DateTimeFormatter` 类

### Java8Tester
```java
import java.time.ZonedDateTime;
import java.time.format.DateTimeFormatter;

public class Java8Tester {

   public static void main(String args[]) {
      Java8Tester tester = new Java8Tester();
      tester.run();
   }

   public void run() {

      ZonedDateTime now = ZonedDateTime.now();
      System.out.println("当前时间是: " + now);

      System.out.println("另一种表示形式:" + now.format(DateTimeFormatter.RFC_1123_DATE_TIME));
   }
}
//运行结果如下

[penglei@www.ycbbs.vip helloworld]$ javac Java8Tester.java && java Java8Tester
当前时间是: 2018-10-08T23:02:03.133357+08:00[Asia/Shanghai]
另一种表示形式:Mon, 8 Oct 2018 23:02:03 +0800
```
我们还可以调用 `DateTimeFormatter.ofPattern()` 方法创建自己的日期时间格式，例如

```java
import java.time.ZonedDateTime;
import java.time.format.DateTimeFormatter;

public class Java8Tester {

   public static void main(String args[]) {
      Java8Tester tester = new Java8Tester();
      tester.run();
   }

   public void run() {

      ZonedDateTime now = ZonedDateTime.now();
      System.out.println("当前时间是: " + now);

      DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy/MM/dd H:m:s");
      System.out.println("另一种表示形式:" + now.format(formatter));
   }
}
运行结果如下

[penglei@www.ycbbs.vip helloworld]$ javac Java8Tester.java && java Java8Tester
当前时间是: 2018-10-08T23:04:49.925018+08:00[Asia/Shanghai]
另一种表示形式:2018/10/08 23:4:49
```
当然了，我们可以调用 `LocalDateTime` 类的静态方法 `parse()` 将我们刚刚自定义的日期时间格式给解析回来

```java
import java.time.LocalDateTime;
import java.time.ZonedDateTime;
import java.time.format.DateTimeFormatter;

public class Java8Tester {

   public static void main(String args[]) {
      Java8Tester tester = new Java8Tester();
      tester.run();
   }

   public void run() {

      ZonedDateTime now = ZonedDateTime.now();
      System.out.println("当前时间是: " + now);

      DateTimeFormatter formatter = DateTimeFormatter.ofPattern("yyyy/MM/dd H:m:s");
      String text = now.format(formatter);
      System.out.println("另一种表示形式:" + text );

      LocalDateTime parsed = LocalDateTime.parse(text, formatter);
      System.out.println("解析后:" + parsed );
   }
}
//运行结果如下

[penglei@twww.ycbbs.vip helloworld]$ javac Java8Tester.java && java Java8Tester
当前时间是: 2018-10-08T23:10:00.979253+08:00[Asia/Shanghai]
另一种表示形式:2018/10/08 23:10:0
解析后:2018-10-08T23:10
```

写完了如果写得有什么问题，希望读者能够给小编留言，也可以点击[此处扫下面二维码关注微信公众号](https://www.ycbbs.vip/?p=28 "此处扫下面二维码关注微信公众号")
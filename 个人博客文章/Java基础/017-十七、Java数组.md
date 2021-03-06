## Java 数组

数组对于每一门编程语言来说都是重要的数据结构之一，当然不同语言对数组的实现及处理也不尽相同。

`Java` 语言中提供的数组是用来存储固定大小的同类型元素。

你可以声明一个数组变量，如 `numbers[100]` 来代替直接声明 100 个独立变量 `number0`，`number1`，`....`，`number99`。

现在将为大家介绍 `Java` 数组的声明、创建和初始化，并给出其对应的代码。

声明数组变量。

首先必须声明数组变量，才能在程序中使用数组。下面是声明数组变量的语法：
```java
dataType[] arrayRefVar; // 首选的方法
或
dataType arrayRefVar[]; // 效果相同，但不是首选方法。
```

> PS 建议使用 `dataType[] arrayRefVar` 的声明风格声明数组变量。 `dataType arrayRefVar[]` 风格是来自 `C/C++` 语言 ，在`Java`中采用是为了让 `C/C++` 程序员能够快速理解java语言。
```java
double[] myList0;//首选方式
double myList1[];//效果相同但不是首选方式
```

## 创建数组

`Java`语言使用`new`操作符来创建数组，

一、使用 `dataType[arraySize]` 创建了一个数组。

二、把新创建的数组的引用赋值给变量 `arrayRefVar`。

数组变量的声明，和创建数组可以用一条语句完成，如下所示：
```java
dataType[] arrayRefVar = new dataType[arraySize];
```

另外，你还可以使用如下的方式创建数组
```java
dataType[] arrayRefVar = {value0, value1, ..., valuek};
```

数组的元素是通过索引访问的。数组索引从 `0` 开始，所以索引值从 `0` 到 `arrayRefVar.length-1`。

下面的语句首先声明了一个数组变量 `myList`，接着创建了一个包含 `10` 个 `double` 类型元素的数组，并且把它的引用赋值给 `myList` 变量。
![](https://gitee.com/duchaochen/gongzhonghao/raw/master/3/17-1.jpg)
下面的图片描绘了数组 `myList`。这里 `myList` 数组里有 `10` 个 `double` 元素，它的下标从 `0` 到 `9`。
![](https://gitee.com/duchaochen/gongzhonghao/raw/master/3/17-2.jpg)

PS数组的元素类型和数组的大小都是确定的，所以当处理数组元素时候，我们通常使用基本循环或者 `foreach` 循环。

该实例完整地展示了如何创建、初始化和操纵数组：
![](https://gitee.com/duchaochen/gongzhonghao/raw/master/3/17-3.jpg)

也可以使用加强`for`循环数组如图
![](https://gitee.com/duchaochen/gongzhonghao/raw/master/3/17-4.jpg)

## 数组作为函数的参数

数组可以作为参数传递给方法。这里当作了解，后期会讲方法。

例如，下面的例子就是一个打印 `int` 数组中元素的方法:
![](https://gitee.com/duchaochen/gongzhonghao/raw/master/3/17-5.jpg)

## 多维数组

多维数组可以看成是数组的数组，比如二维数组就是一个特殊的一维数组，其每一个元素都是一个一维数组，例如：
```java
String str[][] = new String[3][4];
```

多维数组的动态初始化（以二维数组为例）
1. 直接为每一维分配空间，格式如下：

```java
type arrayName = new type[arraylenght1][arraylenght2];
```

`type` 可以为基本数据类型和复合数据类型，`arraylenght1` 和 `arraylenght2` 必须为正整数，`arraylenght1` 为行数，`arraylenght2` 为列数。
例如：

```java
int a[][] = new int[2][3];
```

解析：
二维数组 a 可以看成一个两行三列的数组。

2. 从最高维开始，分别为每一维分配空间，例如：

如图显示
![](https://gitee.com/duchaochen/gongzhonghao/raw/master/3/17-6.jpg)

解析：
`s[0]=new String[2]` 和 `s[1]=new String[3]` 是为最高维分配引用空间，也就是为最高维限制其能保存数据的最长的长度，然后再为其每个数组元素单独分配空间` s0=new String("Good")` 等操作。


写完了如果写得有什么问题，希望读者能够给小编留言，也可以点击[此处扫下面二维码关注微信公众号](https://www.ycbbs.vip/?p=28 "此处扫下面二维码关注微信公众号")
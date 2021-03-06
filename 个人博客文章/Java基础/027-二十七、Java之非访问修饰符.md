## 非访问修饰符


为了实现一些其他的功能，`Java` 也提供了许多非访问修饰符。

`static` 修饰符，用来修饰类方法和类变量。



`final` 修饰符，用来修饰类、方法和变量，`final` 修饰的类不能够被继承，修饰的方法不能被继承类重新定义，修饰的变量为常量，是不可修改的。



`abstract` 修饰符，用来创建抽象类和抽象方法。



`synchronized` 和 `volatile` 修饰符，主要用于线程的编程。



## static 修饰符


静态变量：`static` 关键字用来声明独立于对象的静态变量，无论一个类实例化多少对象，它的静态变量只有一份拷贝。 静态变量也被称为类变量。局部变量不能被声明为 `static` 变量。



静态方法：`static` 关键字用来声明独立于对象的静态方法。静态方法不能使用类的非静态变量。静态方法从参数列表得到数据，然后计算这些数据。



对类变量和方法的访问可以直接使用 类名称.`variablename` 和 类名称.`methodname` 的方式访问。
![](https://gitee.com/duchaochen/gongzhonghao/raw/master/4/27-1.jpg)

## final 修饰符


`final` 变量能被显式地初始化并且只能初始化一次。被声明为 `final` 的对象的引用不能指向不同的对象。但是 `final` 对象里的数据可以被改变。也就是说 `final` 对象的引用不能改变，但是里面的值可以改变。



`final` 修饰符通常和 `static` 修饰符一起使用来创建类常量。
![](https://gitee.com/duchaochen/gongzhonghao/raw/master/4/27-2.jpg)


写完了如果写得有什么问题，希望读者能够给小编留言，也可以点击[此处扫下面二维码关注微信公众号](https://www.ycbbs.vip/?p=28 "此处扫下面二维码关注微信公众号")
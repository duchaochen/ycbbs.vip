## 以下两个片段执行结果差异的原因是什么？
**片段一：**
```java
short s=1;
s=s+1;
```

**片段二：**
```java
short s=1;
s+=1;
```

可以自己组织一下答案，最后看结论

### 结论分析：
片段一自然是编译不通过的 ，提示损失精度 。

**那么片段二为什么能编译通过呢? **
隐式类型转换可以从小到大自动转，即byte->short->int->long，如果反过来会丢失精度，必须进行显示类型转换。

回到这一题来看，s+=1的意思与s = s+1不同，s=s+1这句先执行s+1然后把结果赋给s，由于1为int类型，所以s+1的返回值是int，编译器自动进行了隐式类型转换，所以将一个int类型赋给short就会出错。

而s+=1不同，由于他是使用+=操作符，在解析的时候s+=1就等价于s = (short)(s+1)，也就是s+=1 <=> s =  (s的类型)(s+1)。
>（最后结论引自百度知道，略有删改。
解答出处：https://zhidao.baidu.com/question/250880637.html）

### 扩展：
**基本类型数据及所占字节**

<table>
<tr>
<td>数据类型</td><td>所占字节</td>
</tr>
<tr>
<td>boolean</td><td>未定</td>
</tr>
<tr>
<td>byte</td><td>1字节</td>
</tr>
<tr>
<td>char</td><td>2字节</td>
</tr>
<tr>
<td>short</td><td>2字节</td>
</tr>
<tr><td>int</td><td>4字节</td></tr>
<tr><td>long</td><td>8字节</td></tr>
<tr><td>float</td><td>4字节</td></tr>
<tr><td>double</td><td>8字节</td></tr>
</table>

### 隐式转换与显示转换概念

 **隐式类型转换**

隐式转换也叫作自动类型转换, 由系统自动完成.

从存储范围小的类型到存储范围大的类型.

```java
byte ->short(char)->int->long->float->double
```
 **显示类型转换**

显示类型转换也叫作强制类型转换, 是从存储范围大的类型到存储范围小的类型.

当我们需要将数值范围较大的数值类型赋给数值范围较小的数值类型变量时，由于此时可能会丢失精度，因此，需要人为进行转换。我们称之为强制类型转换。

```java
double→float→long→int→short(char)→byte
```

### 基本数据类型之间的转换规则

1.在一个双操作数以及位运算等算术运算式中，会根据操作数的类型将低级的数据类型自动转换为高级的数据类型，分为以下几种情况：

- 1）只要两个操作数中有一个是double类型的，另一个将会被转换成double类型，并且结果也是double类型；

- 2）只要两个操作数中有一个是float类型的，另一个将会被转换成float类型，并且结果也是float类型；

- 3）只要两个操作数中有一个是long类型的，另一个将会被转换成long类型，并且结果也是long类型；

- 4）两个操作数（包括byte、short、int、char）都将会被转换成int类型，并且结果也是int类型。  

2. 如果低级类型为char型，向高级类型（整型）转换时，会转换为对应ASCII码值，再做其它类型的自动转换。

3. 对于byte,short,char三种类型而言，他们是平级的，因此不能相互自动转换，可以使用下述的强制类型转换。 如：

```java
short i=99 ;
char c=(char)i;
System.out.println("output:"+c);
```
4. 不能在布尔值和任何数字类型间强制类型转换；

5. 不同级别数据类型间的强制转换，可能会导致溢出或精度的下降。

6. 当字节类型变量参与运算，java作自动数据运算类型的提升，将其转换为int类型。

例如：

```java
byte b;
b=3;
b=(byte)(b*3);//必须声明byte。
```

希望读者能够给小编留言，也可以点击[此处扫下面二维码关注微信公众号](https://www.ycbbs.vip/?p=28 "此处扫下面二维码关注微信公众号")

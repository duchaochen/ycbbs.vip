### 91，什么是ORM？ 
对象关系映射（Object-Relational Mapping，简称ORM）是一种为了解决程序的面向对象模型与数据库的关系模型互不匹配问题的技术；

简单的说，ORM是通过使用描述对象和数据库之间映射的元数据（在Java中可以用XML或者是注解），将程序中的对象自动持久化到关系数据库中或者将关系数据库表中的行转换成Java对象，其本质上就是将数据从一种形式转换到另外一种形式。

### 92，Hibernate中SessionFactory是线程安全的吗？Session是线程安全的吗（两个线程能够共享同一个Session吗）？ 

SessionFactory对应Hibernate的一个数据存储的概念，它是线程安全的，可以被多个线程并发访问。SessionFactory一般只会在启动的时候构建。对于应用程序，最好将SessionFactory通过单例模式进行封装以便于访问。

Session是一个轻量级非线程安全的对象（线程间不能共享session），它表示与数据库进行交互的一个工作单元。Session是由SessionFactory创建的，在任务完成之后它会被关闭。Session是持久层服务对外提供的主要接口。

Session会延迟获取数据库连接（也就是在需要的时候才会获取）。为了避免创建太多的session，可以使用ThreadLocal将session和当前线程绑定在一起，这样可以让同一个线程获得的总是同一个session。

Hibernate 3中SessionFactory的getCurrentSession()方法就可以做到。

### 93，Session的save()、update()、merge()、lock()、saveOrUpdate()和persist()方法分别是做什么的？有什么区别？ 
Hibernate的对象有三种状态：瞬时态（transient）、持久态（persistent）和游离态（detached）。

瞬时态的实例可以通过调用save()、persist()或者saveOrUpdate()方法变成持久态；

游离态的实例可以通过调用 update()、saveOrUpdate()、lock()或者replicate()变成持久态。save()和persist()将会引发SQL的INSERT语句，而update()或merge()会引发UPDATE语句。

save()和update()的区别在于一个是将瞬时态对象变成持久态，一个是将游离态对象变为持久态。merge()方法可以完成save()和update()方法的功能，它的意图是将新的状态合并到已有的持久化对象上或创建新的持久化对象。

对于persist()方法，按照官方文档的说明：

1、persist()方法把一个瞬时态的实例持久化，但是并不保证标识符被立刻填入到持久化实例中，标识符的填入可能被推迟到flush的时间；

2、persist()方法保证当它在一个事务外部被调用的时候并不触发一个INSERT语句，当需要封装一个长会话流程的时候，persist()方法是很有必要的；

3、save()方法不保证第2条，它要返回标识符，所以它会立即执行INSERT语句，不管是在事务内部还是外部。至于lock()方法和update()方法的区别，update()方法是把一个已经更改过的脱管状态的对象变成持久状态；lock()方法是把一个没有更改过的脱管状态的对象变成持久状态。

### 94，阐述Session加载实体对象的过程。 
1、Session在调用数据库查询功能之前，首先会在一级缓存中通过实体类型和主键进行查找，如果一级缓存查找命中且数据状态合法，则直接返回； 
2、如果一级缓存没有命中，接下来Session会在当前NonExists记录（相当于一个查询黑名单，如果出现重复的无效查询可以迅速做出判断，从而提升性能）中进行查找，如果NonExists中存在同样的查询条件，则返回null； 
3、如果一级缓存查询失败查询二级缓存，如果二级缓存命中直接返回； 
4、如果之前的查询都未命中，则发出SQL语句，如果查询未发现对应记录则将此次查询添加到Session的NonExists中加以记录，并返回null； 
5、根据映射配置和SQL语句得到ResultSet，并创建对应的实体对象； 
6、将对象纳入Session（一级缓存）的管理； 
7、如果有对应的拦截器，则执行拦截器的onLoad方法； 
8、如果开启并设置了要使用二级缓存，则将数据对象纳入二级缓存； 
9、返回数据对象。

### 95，MyBatis中使用#和$书写占位符有什么区别？ 
#将传入的数据都当成一个字符串，会对传入的数据自动加上引号；
    $将传入的数据直接显示生成在SQL中。
注意：使用$占位符可能会导致SQL注射攻击，能用#的地方就不要使用$，写order by子句的时候应该用$而不是#。

### 96，解释一下MyBatis中命名空间（namespace）的作用。 
在大型项目中，可能存在大量的SQL语句，这时候为每个SQL语句起一个唯一的标识（ID）就变得并不容易了。为了解决这个问题，在MyBatis中，可以为每个映射文件起一个唯一的命名空间，这样定义在这个映射文件中的每个SQL语句就成了定义在这个命名空间中的一个ID。

只要我们能够保证每个命名空间中这个ID是唯一的，即使在不同映射文件中的语句ID相同，也不会再产生冲突了。

### 97、MyBatis中的动态SQL是什么意思？ 
对于一些复杂的查询，我们可能会指定多个查询条件，但是这些条件可能存在也可能不存在，如果不使用持久层框架我们可能需要自己拼装SQL语句，不过MyBatis提供了动态SQL的功能来解决这个问题。MyBatis中用于实现动态SQL的元素主要有： 
```html
- if    - choose / when / otherwise    - trim    - where    - set     - foreach
用法举例：
   <select id="foo" parameterType="Blog" resultType="Blog">
   select * from t_blog where 1 = 1
<if test="title != null">    
   and title = #{title}
</if>
<if test="content != null">    
   and content = #{content}
</if>
<if test="owner != null">    
   and owner = #{owner}
</if>
   </select>
```
### 98，JDBC编程有哪些不足之处，MyBatis是如何解决这些问题的？
1、JDBC：数据库链接创建、释放频繁造成系统资源浪费从而影响系统性能，如果使用数据库链接池可解决此问题。
MyBatis：在SqlMapConfig.xml中配置数据链接池，使用连接池管理数据库链接。

2、JDBC：Sql语句写在代码中造成代码不易维护，实际应用sql变化的可能较大，sql变动需要改变java代码。
MyBatis：将Sql语句配置在XXXXmapper.xml文件中与java代码分离。

3、JDBC：向sql语句传参数麻烦，因为sql语句的where条件不一定，可能多也可能少，占位符需要和参数一一对应。
MyBatis： Mybatis自动将java对象映射至sql语句。

4，JDBC：对结果集解析麻烦，sql变化导致解析代码变化，且解析前需要遍历，如果能将数据库记录封装成pojo对象解析比较方便。
MyBatis：Mybatis自动将sql执行结果映射至java对象。

### 99，MyBatis与Hibernate有哪些不同？
1、Mybatis和hibernate不同，它不完全是一个ORM框架，因为MyBatis需要程序员自己编写Sql语句，不过mybatis可以通过XML或注解方式灵活配置要运行的sql语句，并将java对象和sql语句映射生成最终执行的sql，最后将sql执行的结果再映射生成java对象。   

2、Mybatis学习门槛低，简单易学，程序员直接编写原生态sql，可严格控制sql执行性能，灵活度高，非常适合对关系数据模型要求不高的软件开发，例如互联网软件、企业运营类软件等，因为这类软件需求变化频繁，一但需求变化要求成果输出迅速。

但是灵活的前提是mybatis无法做到数据库无关性，如果需要实现支持多种数据库的软件则需要自定义多套sql映射文件，工作量大。

3、Hibernate对象/关系映射能力强，数据库无关性好，对于关系模型要求高的软件（例如需求固定的定制化软件）如果用hibernate开发可以节省很多代码，提高效率。

但是Hibernate的缺点是学习门槛高，要精通门槛更高，而且怎么设计O/R映射，在性能和对象模型之间如何权衡，以及怎样用好Hibernate需要具有很强的经验和能力才行。  

总之，按照用户的需求在有限的资源环境下只要能做出维护性、扩展性良好的软件架构都是好架构，所以框架只有适合才是最好。

（这里也可以结合自己的理解说，别说的收不住）

### 100，简单的说一下MyBatis的一级缓存和二级缓存？
Mybatis首先去缓存中查询结果集，如果没有则查询数据库，如果有则从缓存取出返回结果集就不走数据库。Mybatis内部存储缓存使用一个HashMap，key为hashCode+sqlId+Sql语句。

value为从查询出来映射生成的java对象

Mybatis的二级缓存即查询缓存，它的作用域是一个mapper的namespace，即在同一个namespace中查询sql可以从缓存中获取数据。二级缓存是可以跨SqlSession的。


希望读者能够给小编留言，也可以点击[此处扫下面二维码关注微信公众号](https://www.ycbbs.vip/?p=28 "此处扫下面二维码关注微信公众号")
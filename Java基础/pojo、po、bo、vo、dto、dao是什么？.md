# PO,BO,VO和POJO的区别

PO：persistent object 持久对象

1 ．有时也被称为Data对象，对应数据库中的entity，可以简单认为一个PO对应数据库中的一条记录。

2 ．在hibernate持久化框架中与insert/delet操作密切相关。

3 ．PO中不应该包含任何对数据库的操作。

------

POJO ：plain ordinary java object 无规则简单java对象

一个中间对象，可以转化为PO、DTO、VO。

1 ．POJO持久化之后==〉PO

（在运行期，由Hibernate中的cglib动态把POJO转换为PO，PO相对于POJO会增加一些用来管理数据库entity状态的属性和方法。PO对于programmer来说完全透明，由于是运行期生成PO，所以可以支持增量编译，增量调试。）

2 ．POJO传输过程中==〉DTO

3 ．POJO用作表示层==〉VO

PO 和VO都应该属于它。

------

BO：business object 业务对象

业务对象主要作用是把业务逻辑封装为一个对象。这个对象可以包括一个或多个其它的对象。

比如一个简历，有教育经历、工作经历、社会关系等等。我们可以把教育经历对应一个PO，工作经历对应一个PO，社会关系对应一个PO。

建立一个对应简历的BO对象处理简历，每个BO包含这些PO。

这样处理业务逻辑时，我们就可以针对BO去处理。

封装业务逻辑为一个对象（可以包括多个PO，通常需要将BO转化成PO，才能进行数据的持久化，反之，从DB中得到的PO，需要转化成BO才能在业务层使用）。

关于BO主要有三种概念

1 、只包含业务对象的属性；

2 、只包含业务方法；

3 、两者都包含。

在实际使用中，认为哪一种概念正确并不重要，关键是实际应用中适合自己项目的需要。

------

VO：value object 值对象 / view object 表现层对象

1 ．主要对应页面显示（web页面/swt、swing界面）的数据对象。

2 ．可以和表对应，也可以不，这根据业务的需要。

------

DTO（TO）：Data Transfer Object 数据传输对象

1 ．用在需要跨进程或远程传输时，它不应该包含业务逻辑。

2 ．比如一张表有100个字段，那么对应的PO就有100个属性（大多数情况下，DTO内的数据来自多个表）。但view层只需显示10个字段，没有必要把整个PO对象传递到client，这时我们就可以用只有这10个属性的DTO来传输数据到client，这样也不会暴露server端表结构。到达客户端以后，如果用这个对象来对应界面显示，那此时它的身份就转为VO。

------

DAO：data access object数据访问对象

1 ．主要用来封装对DB的访问（CRUD操作）。

2 ．通过接收Business层的数据，把POJO持久化为PO。

简易的关系图：

![这里写图片描述](https://img-blog.csdn.net/20180717104224284?watermark/2/text/aHR0cHM6Ly9ibG9nLmNzZG4ubmV0L3UwMTE4NzA1NDc=/font/5a6L5L2T/fontsize/400/fill/I0JBQkFCMA==/dissolve/70)





转自<https://blog.csdn.net/u011870547/article/details/81077153>


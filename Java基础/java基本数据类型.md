只有8种基本类型可以算.其他引用类型都是由java虚拟机决定的自己不能操作
byte 1字节
short 2字节
int 4字节
long 8字节
float 4字节
double 8字节
char 2字节
boolean 1字节或4字节（虽然定义了boolean这种数据类型，但是只对它提供了非常有限的支持。在Java虚拟机中没有任何供boolean值专用的字节码指令，Java语言表达式所操作的boolean值，在编译之后都使用Java虚拟机中的int数据类型来代替，而boolean数组将会被编码成Java虚拟机的byte数组，每个元素boolean元素占8位”。这样我们可以得出boolean类型占了单独使用是4个字节，在数组中又是1个字节。）





下边介绍一些在笔试面试中，经常遇到的问题。

1. short s1 = 1; s1 = s1 + 1;有什么错? short s1 = 1; s1 +=1;有什么错?

1) 对于short s1=1;s1=s1+1来说，在s1+1运算时会自动提升表达式的类型为int，那么将int赋予给short类型的变量s1会出现类型转换错误。

2) 对于short s1=1;s1+=1来说 +=是java语言规定的运算符，java编译器会对它进行特殊处理，因此可以正确编译。

2. Integer和int的区别

int是java的8种基本数据类型之一。Integer是Java为int类型提供的封装类。

int变量的默认值为0，Integer变量的默认值为null，这一点说明Integer可以区分出未赋值和值为0的区别，
比如说一名学生没来参加考试，另一名学生参加考试全答错了，那么第一名考生的成绩应该是null，第二名考生的成绩应该是0分。

Integer类内提供了一些关于整数操作的一些方法。

3. char类型变量能不能储存一个中文的汉字，为什么？

char类型变量是用来储存Unicode编码的字符的，unicode字符集包含了汉字，
所以char类型当然可以存储汉字的。

如果某个生僻字没有包含在unicode编码字符集中，那么char就不能存储该生僻字。

4. String是基本数据类型吗?

基本数据类型包括byte、short、int、char、long、float、double和boolean。
所以String不是基本数据类型。

5. switch语句能否作用在byte上，能否作用在long上，能否作用在string上？

在switch(expr1)中，expr1只能是一个整数表达式或者枚举常量，
整数表达式可以是int基本类型或Integer包装类型。

由于，byte,short,char都可以隐式转换为int，所以，这些类型以及这些类型的包装类型也是可以的。

long和String类型都不符合switch的语法规定，并且不能被隐式转换成int类型，
所以，它们不能作用于swtich语句中。

不过，在1.7版本之后switch就可以作用在string上了。


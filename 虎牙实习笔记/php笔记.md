LAMP:Linux+Apache+Mysql+PHP
一.第一个PHP脚本程序
1.PHP开发步骤
- 使用编辑器创建一个包含源代码的磁盘文件 <?php ... ?>
- 将文件上传到Web服务器上
- 通过浏览器访问Web服务器运行该文件

2.语法规范
- 下列情况使用两个空行：
	- 一个源代码的两个代码片段之间
	- 两个类的声明之间
- 下列情况使用一个空行
	- 两个函数的声明之间
	- 函数内的局部变量和函数的第一条语句之间
	- 块注释或单行注释之前
	- 一个函数内的两个逻辑代码段之间，用来提高可读性

3.语法
（1）变量
1)变量声明： $a=100;
2)变量范围：局部（函数内），全局（一个页面）
empty($var) 检查一个变量是否为空 ,isset($var) 检查变量是否设置,unset($var) 释放变量
可变变量：用其他变量值作为自己的变量名吗，eg:
$hi ="hello";
$$hi="world";
echo "$hi $hello"; //hello world
echo "$hi ${$hi}"; //hello world
3)变量的引用赋值
通常是传值赋值，引用赋值意味着两个变量相互影响，其中一个值改变会改变另外一个
eg: 
$foo='Bob';
$bar=&$foo;
注意：
- 不用将表达式、函数作为引用赋值
- PHP的引用并不是C语言的指针，而只是把各自的值关联起来而已，unsert($foo),$bar不会消失
4）变量类型
弱类型
boolean
integer
float,double
string:单引号，不转义和解析变量；双引号，解析变量和转义字符；定界符，中间放strng <<<EOT .....EOT;
array: $arr=array("foo"=>"bar",12=>true)；echo $arr["foo"]  key可以是integer或string，value可以是任何值
object: eg:
	<?php 
		class Person{
			var $name;
			
			function say(){
				echo "Doing foo."
			}
		}

		$p = new Person;
		$p->name="Tom";
		$p->say();
	
	?>
resource(资源)：
eg: $file_handle=fopen("info.txt","w");
$link_mysql=mysql_connect("localhost","root","123456");
NULL: 
- 将变量直接赋值为NULL
- 声明的变量尚未赋值
- 被销毁的变量
（2）常量
1）定义：
使用define(string name,mixed value[,bool case_insensitive]);
第一个参数是常量名，第二个是值，第三个是可选的区分大小写（默认是false）
eg:
define("CON_INT",11);
echo CON_INT; //11
defined("CONTANT")//false,检查常量是否存在
2）常量和变量：
- 常量前面没有$符号
- 常量只能用define函数定义，不能通过赋值语句
- 常量可以不用理会变量的范围规则而在任何地方定义和访问
- 常量一旦定义就不能被重新定义或取消定义，直到脚本运行结束自动释放
- 常量的值只能是标量（boolean，integer，float，string这四种之一）、
（3）运算符
1）字符变量运算：PHP沿袭了Perl的习惯，字符变量‘Z’+1=‘AA’，++‘a’='b'，只能递增不能递减
2）字符串运算：用“.”来拼接字符串
eg: 
$name = "Tom";
echo "我的名字是：".$name;
3)比较运算符
===：左边操作数等于右边操作数，包括类型
!===：左边操作数不等于右边操作数，或者类型不相同
4）逻辑运算符
xor：逻辑亦或
5）其他运算符
反引号 ``:PHP尝试将反引号中的内容作为外壳命令来执行，并将其输出信息返回，$a=`ls -al`
错误信息控制运算符@：将其放在表达式前，该表达式产生的任何错误都会被忽略 ，eg：@表达式
=>: 数组下标指定符号 键=>值
->:对象成员，方法访问，对象->成员
（4）函数
1)全局变量：在函数内无法直接使用全局变量
如果要使用需要加 global或$GLOBALS['name']
eg:
$name='123';
function test(){
	global $name;
	echo $name;
	echo $GLOBALS['name'];	
}

2)伪类型参数的函数；
mixed funName(mixed $args);
number funName(number $args);
3)引用参数的函数：
function funName(&$arg);
形参有&修饰的参数就需要传递参数，而不能传值
4）默认参数的函数
eg: function person($name="张三"，$age=20,$sex="男"){
}
5）可变个数参数的函数,使用func_get_args()(func_get_args($i))，返回参数数组
eg：
function more_args(){
	$args=func_get_args();
	for($i=0;$i<count($args);$i++){
		echo "第".$i."个参数".$args[$i]."<br>";
		//或者echo "第".$i."个参数".func_get_arg($i)."<br>";	
	}
}
5)回调函数
【1】.变量函数：将函数名作为变量的值，变量可以调用该函数
eg:
function one($a,$b){
	return $a+$b;
}
$result = "one";
echo $result(1,2);

【2】.将函数作为参数,利用变量函数
eg: function filter($fun){
	return $fun(1)+1;
	}
function fun($a){
return a;
}
filter("fun");
6)自定义函数库
导入函数库：require 'example.php' 或 include 'example.php'
对于include()语句来说每次执行文件时都要进行读取和评估
require()只处理一次。

 

4.数组与数据结构
（1）数组的分类：索引数组和关联数组
（2）数组的定义：
- 直接赋值：$contact1[0]=1;$contact2["hi"]='2313';
- 使用array()新建数组：$contact1=array(key1=>value,key2=>value); (array(value,value))
（3）数组的遍历：
- for:
eg： 
for($i=0;$i<count($contact);$i++){
	echo $contact[$i];
}
- foreach:
eg:
for($contact as $value){
...
}
or 
for($contact as $key=>$value){
...
}
(4)预定义数组
$_SERVER : 服务器相关的信息
$_ENV: 执行环境提交至脚本的变量
$_GET: 经由URL提交至脚本的变量
$_POST:经由POST方法提高至脚本的变量
$_REQUEST：经由get或post提交的变量
$_FILES:经由POST文件上传而提交至脚本的变量
$_COOKIE:cookies变量
$_SESSION:会话变量
$GLOBALS:包含一个引用指向每个当前脚本的全局范围内有效的变量

5.面向对象设计
属性用var修饰，一旦由其他关键字修饰就要去掉“var”
->访问属性和方法
$this
(1)构造方法(两个下划线)
function __construct(...)
(2)析构方法（销毁时调用）
function __destruct()(不能有参数)
(3)封装性
属性默认是public
__set(string name,mixed vaule)
__get(string name) //不需要调用，private属性 $object->attribute,自动调用__get方法
(4)继承性
单继承
extends
子类调用父类的方法，parent::方法名
(5)访问控制
public（公用的，默认）
private（私有的）
protected（保护的，子类可以访问）
(6)常见关键字和魔术方法
1）final
只能修饰类或方法
- 类不能被继承
- 方法不能被覆盖
2）static
修饰的属性方法通过:
类名::属性or方法
类内方法通过：
self::属性or方法
3）const关键字
使用方法和static类似，修饰类中的常量
4）克隆对象
eg：
$p1=new Person();
$p2=clone $p1;
5)__toString()
6)__call($functionName,$args):调用不存在的方法会自动调用该方法
7）自动加载类
如果找不到类的时候自动加载
eg:
<?php
function __autoload($className){
	include(strolower($className).".class.php");
}

$obj = new User(); //自动加载User.class.php文件
?>


6.字符串处理


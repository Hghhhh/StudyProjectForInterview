LAMP:Linux+Apache+Mysql+PHP
һ.��һ��PHP�ű�����
1.PHP��������
- ʹ�ñ༭������һ������Դ����Ĵ����ļ� <?php ... ?>
- ���ļ��ϴ���Web��������
- ͨ�����������Web���������и��ļ�

2.�﷨�淶
- �������ʹ���������У�
	- һ��Դ�������������Ƭ��֮��
	- �����������֮��
- �������ʹ��һ������
	- ��������������֮��
	- �����ڵľֲ������ͺ����ĵ�һ�����֮��
	- ��ע�ͻ���ע��֮ǰ
	- һ�������ڵ������߼������֮�䣬������߿ɶ���

3.�﷨
��1������
1)���������� $a=100;
2)������Χ���ֲ��������ڣ���ȫ�֣�һ��ҳ�棩
empty($var) ���һ�������Ƿ�Ϊ�� ,isset($var) �������Ƿ�����,unset($var) �ͷű���
�ɱ����������������ֵ��Ϊ�Լ��ı�������eg:
$hi ="hello";
$$hi="world";
echo "$hi $hello"; //hello world
echo "$hi ${$hi}"; //hello world
3)���������ø�ֵ
ͨ���Ǵ�ֵ��ֵ�����ø�ֵ��ζ�����������໥Ӱ�죬����һ��ֵ�ı��ı�����һ��
eg: 
$foo='Bob';
$bar=&$foo;
ע�⣺
- ���ý����ʽ��������Ϊ���ø�ֵ
- PHP�����ò�����C���Ե�ָ�룬��ֻ�ǰѸ��Ե�ֵ�����������ѣ�unsert($foo),$bar������ʧ
4����������
������
boolean
integer
float,double
string:�����ţ���ת��ͽ���������˫���ţ�����������ת���ַ�����������м��strng <<<EOT .....EOT;
array: $arr=array("foo"=>"bar",12=>true)��echo $arr["foo"]  key������integer��string��value�������κ�ֵ
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
resource(��Դ)��
eg: $file_handle=fopen("info.txt","w");
$link_mysql=mysql_connect("localhost","root","123456");
NULL: 
- ������ֱ�Ӹ�ֵΪNULL
- �����ı�����δ��ֵ
- �����ٵı���
��2������
1�����壺
ʹ��define(string name,mixed value[,bool case_insensitive]);
��һ�������ǳ��������ڶ�����ֵ���������ǿ�ѡ�����ִ�Сд��Ĭ����false��
eg:
define("CON_INT",11);
echo CON_INT; //11
defined("CONTANT")//false,��鳣���Ƿ����
2�������ͱ�����
- ����ǰ��û��$����
- ����ֻ����define�������壬����ͨ����ֵ���
- �������Բ����������ķ�Χ��������κεط�����ͷ���
- ����һ������Ͳ��ܱ����¶����ȡ�����壬ֱ���ű����н����Զ��ͷ�
- ������ֵֻ���Ǳ�����boolean��integer��float��string������֮һ����
��3�������
1���ַ��������㣺PHP��Ϯ��Perl��ϰ�ߣ��ַ�������Z��+1=��AA����++��a��='b'��ֻ�ܵ������ܵݼ�
2���ַ������㣺�á�.����ƴ���ַ���
eg: 
$name = "Tom";
echo "�ҵ������ǣ�".$name;
3)�Ƚ������
===����߲����������ұ߲���������������
!===����߲������������ұ߲��������������Ͳ���ͬ
4���߼������
xor���߼����
5�����������
������ ``:PHP���Խ��������е�������Ϊ���������ִ�У������������Ϣ���أ�$a=`ls -al`
������Ϣ���������@��������ڱ��ʽǰ���ñ��ʽ�������κδ��󶼻ᱻ���� ��eg��@���ʽ
=>: �����±�ָ������ ��=>ֵ
->:�����Ա���������ʣ�����->��Ա
��4������
1)ȫ�ֱ������ں������޷�ֱ��ʹ��ȫ�ֱ���
���Ҫʹ����Ҫ�� global��$GLOBALS['name']
eg:
$name='123';
function test(){
	global $name;
	echo $name;
	echo $GLOBALS['name'];	
}

2)α���Ͳ����ĺ�����
mixed funName(mixed $args);
number funName(number $args);
3)���ò����ĺ�����
function funName(&$arg);
�β���&���εĲ�������Ҫ���ݲ����������ܴ�ֵ
4��Ĭ�ϲ����ĺ���
eg: function person($name="����"��$age=20,$sex="��"){
}
5���ɱ���������ĺ���,ʹ��func_get_args()(func_get_args($i))�����ز�������
eg��
function more_args(){
	$args=func_get_args();
	for($i=0;$i<count($args);$i++){
		echo "��".$i."������".$args[$i]."<br>";
		//����echo "��".$i."������".func_get_arg($i)."<br>";	
	}
}
5)�ص�����
��1��.��������������������Ϊ������ֵ���������Ե��øú���
eg:
function one($a,$b){
	return $a+$b;
}
$result = "one";
echo $result(1,2);

��2��.��������Ϊ����,���ñ�������
eg: function filter($fun){
	return $fun(1)+1;
	}
function fun($a){
return a;
}
filter("fun");
6)�Զ��庯����
���뺯���⣺require 'example.php' �� include 'example.php'
����include()�����˵ÿ��ִ���ļ�ʱ��Ҫ���ж�ȡ������
require()ֻ����һ�Ρ�

 

4.���������ݽṹ
��1������ķ��ࣺ��������͹�������
��2������Ķ��壺
- ֱ�Ӹ�ֵ��$contact1[0]=1;$contact2["hi"]='2313';
- ʹ��array()�½����飺$contact1=array(key1=>value,key2=>value); (array(value,value))
��3������ı�����
- for:
eg�� 
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
(4)Ԥ��������
$_SERVER : ��������ص���Ϣ
$_ENV: ִ�л����ύ���ű��ı���
$_GET: ����URL�ύ���ű��ı���
$_POST:����POST����������ű��ı���
$_REQUEST������get��post�ύ�ı���
$_FILES:����POST�ļ��ϴ����ύ���ű��ı���
$_COOKIE:cookies����
$_SESSION:�Ự����
$GLOBALS:����һ������ָ��ÿ����ǰ�ű���ȫ�ַ�Χ����Ч�ı���

5.����������
������var���Σ�һ���������ؼ������ξ�Ҫȥ����var��
->�������Ժͷ���
$this
(1)���췽��(�����»���)
function __construct(...)
(2)��������������ʱ���ã�
function __destruct()(�����в���)
(3)��װ��
����Ĭ����public
__set(string name,mixed vaule)
__get(string name) //����Ҫ���ã�private���� $object->attribute,�Զ�����__get����
(4)�̳���
���̳�
extends
������ø���ķ�����parent::������
(5)���ʿ���
public�����õģ�Ĭ�ϣ�
private��˽�еģ�
protected�������ģ�������Է��ʣ�
(6)�����ؼ��ֺ�ħ������
1��final
ֻ��������򷽷�
- �಻�ܱ��̳�
- �������ܱ�����
2��static
���ε����Է���ͨ��:
����::����or����
���ڷ���ͨ����
self::����or����
3��const�ؼ���
ʹ�÷�����static���ƣ��������еĳ���
4����¡����
eg��
$p1=new Person();
$p2=clone $p1;
5)__toString()
6)__call($functionName,$args):���ò����ڵķ������Զ����ø÷���
7���Զ�������
����Ҳ������ʱ���Զ�����
eg:
<?php
function __autoload($className){
	include(strolower($className).".class.php");
}

$obj = new User(); //�Զ�����User.class.php�ļ�
?>


6.�ַ�������


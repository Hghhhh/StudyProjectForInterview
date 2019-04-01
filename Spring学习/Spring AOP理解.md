##为什么用AOP？

解决OOP的缺点，不用在特定业务的方法中显式调用与业务无关的代码。使用生成代理类的方式实现横切代码的调用。

##AOP的实现

aop有两种实现方式，一种是使用AspectJ在编译期静态生成代理类，另一种是使用CGLIB或JDK动态代理在运行期动态生成代理类。Spring AOP的选择方式是，如果要代理的类实现了接口，就使用JDK动态代理，否则使用CGLIB。

###AOP中的抽象概念
连接点（join point）：可以被拦截的目标方法。

切点（point cut）:指定的要进行切入的连接点。

通知（advice）：切点的前后或环绕要执行的动作。

切面（aspect）：由切点和通知组成，定义通知应用带哪个切点上。

织入（weaving）：把切面的代码应用到目标方法的过程。

###通知（advice）分为下面5种

- before 目标方法执行前通知，前置通知
- after 后置通知
- after throwing：目标方法抛出异常时执行，异常通知
- after returning：目标方法成功返回值时执行，后置返回通知。（after是不管是否成功返回都是执行）
- around：在目标函数执行中执行，可控制目标函数是否执行，环绕通知

<!--more-->

##AspectJ
AspectJ 是一个基于 Java 语言的 AOP 框架，提供了强大的 AOP 功能。AspectJ定义了AOP语法，所以它有一个专门的[编译器](https://baike.baidu.com/item/%E7%BC%96%E8%AF%91%E5%99%A8/8853067)ajc用来生成遵守Java字节编码规范的Class文件。

使用AspectJ需要安装插件，然后使用ajc编译器代替javac编译器，idea中安装AspectJ插件可以看[这篇文章](https://www.cnblogs.com/junzi2099/p/8275116.html)

原理：aspectJ编译的时候先编译切面类生成字节码文件，再编译连接点类，在编译连接点的时候切面的字节码织入，生成代理类的字节码文件。通过反编译生成的字节码可以看到代码中增加了切面的内容。

![aspectJ](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/aspectj.png)



##CGLIB动态代理的基本使用（摘自[五月的仓颉](https://www.cnblogs.com/xrq730/p/6661692.html)）

Cglib是一个强大的、高性能的**代码生成包**。

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/GCLIB.gif)

CGLIB使用了ASM这个直接操作字节码的框架。

使用CGLIB的步骤：

1.实现MethodInterceptor，在intercept(Object object, Method method, Object[] objects, MethodProxy proxy)方法中使用proxy.invokeSuper（object，objects）调用切点，在切点前后使用advice。

2.使用Enhancer创建代理对象，enhancer.setSupperClass(class)设置切点，enhancer.setCallback(Interceptor)设置拦截器（相当于切面），然后使用enhancer.create()创建代理对象。

3.代理对象调用方法。

**使用Cglib代码对类做代理**

下面演示一下Cglib代码示例----对类做代理。首先定义一个Dao类，里面有一个select()方法和一个update()方法：

```
public class Dao {
    
    public void update() {
        System.out.println("PeopleDao.update()");
    }
    
    public void select() {
        System.out.println("PeopleDao.select()");
    }
}
```

创建一个Dao代理，实现MethodInterceptor接口，目标是在update()方法与select()方法调用前后输出两句话：

```
public class DaoProxy implements MethodInterceptor {

    @Override
    public Object intercept(Object object, Method method, Object[] objects, MethodProxy proxy) throws Throwable {
        System.out.println("Before Method Invoke");
        proxy.invokeSuper(object, objects);
        System.out.println("After Method Invoke");
        
        return object;
    }
    
}
```

intercept方法的参数名并不是原生的参数名，我做了自己的调整，几个参数的含义为：

- Object表示要进行增强的对象
- Method表示拦截的方法
- Object[]数组表示参数列表，基本数据类型需要传入其包装类型，如int-->Integer、long-Long、double-->Double
- MethodProxy表示对方法的代理，invokeSuper方法表示对被代理对象方法的调用

写一个测试类：



```
public class CglibTest {

    @Test
    public void testCglib() {
        DaoProxy daoProxy = new DaoProxy();
        
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(Dao.class);
        enhancer.setCallback(daoProxy);
        
        Dao dao = (Dao)enhancer.create();
        dao.update();
        dao.select();
    }
    
}
```

这是使用Cglib的通用写法，setSuperclass表示设置要代理的类，setCallback表示设置回调即MethodInterceptor的实现类，使用create()方法生成一个代理对象，注意要强转一下，因为返回的是Object。最后看一下运行结果：

```
Before Method Invoke
PeopleDao.update()
After Method Invoke
Before Method Invoke
PeopleDao.select()
After Method Invoke
```

符合我们的期望。

 

**使用Cglib定义不同的拦截策略**

再扩展一点点，比方说在AOP中我们经常碰到的一种复杂场景是：**我们想对类A的B方法使用一种拦截策略、类A的C方法使用另外一种拦截策略**。

在本例中，即我们想对select()方法与update()方法使用不同的拦截策略，那么我们先定义一个新的Proxy：



```
public class DaoAnotherProxy implements MethodInterceptor {

    @Override
    public Object intercept(Object object, Method method, Object[] objects, MethodProxy proxy) throws Throwable {
        
        System.out.println("StartTime=[" + System.currentTimeMillis() + "]");
        method.invoke(object, objects);
        System.out.println("EndTime=[" + System.currentTimeMillis() + "]");
        return object;
    }
    
}
```



方法调用前后输出一下开始时间与结束时间。为了实现我们的需求，实现一下CallbackFilter：



```
public class DaoFilter implements CallbackFilter {

    @Override
    public int accept(Method method) {
        if ("select".equals(method.getName())) {
            return 0;
        }
        return 1;
    }
    
}
```



返回的数值表示顺序，结合下面的代码解释，测试代码要修改一下：



```
public class CglibTest {

    @Test
    public void testCglib() {
        DaoProxy daoProxy = new DaoProxy();
        DaoAnotherProxy daoAnotherProxy = new DaoAnotherProxy();
        
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(Dao.class);
        enhancer.setCallbacks(new Callback[]{daoProxy, daoAnotherProxy, NoOp.INSTANCE});
        enhancer.setCallbackFilter(new DaoFilter());
        
        Dao dao = (Dao)enhancer.create();
        dao.update();
        dao.select();
    }
    
}
```



意思是CallbackFilter的accept方法返回的数值表示的是顺序，顺序和setCallbacks里面Proxy的顺序是一致的。再解释清楚一点，Callback数组中有三个callback，那么：

- 方法名为"select"的方法返回的顺序为0，即使用Callback数组中的0位callback，即DaoProxy
- 方法名不为"select"的方法返回的顺序为1，即使用Callback数组中的1位callback，即DaoAnotherProxy

因此，方法的执行结果为：

```
StartTime=[1491198489261]
PeopleDao.update()
EndTime=[1491198489275]
Before Method Invoke
PeopleDao.select()
After Method Invoke
```

符合我们的预期，因为update()方法不是方法名为"select"的方法，因此返回1，返回1使用DaoAnotherProxy，即打印时间；select()方法是方法名为"select"的方法，因此返回0，返回0使用DaoProxy，即方法调用前后输出两句话。

这里要额外提一下，Callback数组中我特意定义了一个NoOp.INSTANCE，这表示一个空Callback，即如果不想对某个方法进行拦截，可以在DaoFilter中返回2，具体效果可以自己尝试一下。

 

**构造函数不拦截方法**

如果Update()方法与select()方法在构造函数中被调用，那么也是会对这两个方法进行相应的拦截的，现在我想要的是构造函数中调用的方法不会被拦截，那么应该如何做？先改一下Dao代码，加一个构造方法Dao()，调用一下update()方法：



```
public class Dao {
    
    public Dao() {
        update();
    }
    
    public void update() {
        System.out.println("PeopleDao.update()");
    }
    
    public void select() {
        System.out.println("PeopleDao.select()");
    }
}
```



如果想要在构造函数中调用update()方法时，不拦截的话，Enhancer中有一个setInterceptDuringConstruction(boolean interceptDuringConstruction)方法设置为false即可，默认为true，即构造函数中调用方法也是会拦截的。那么测试方法这么写：



```
public class CglibTest {

    @Test
    public void testCglib() {
        DaoProxy daoProxy = new DaoProxy();
        DaoAnotherProxy daoAnotherProxy = new DaoAnotherProxy();
        
        Enhancer enhancer = new Enhancer();
        enhancer.setSuperclass(Dao.class);
        enhancer.setCallbacks(new Callback[]{daoProxy, daoAnotherProxy, NoOp.INSTANCE});
        enhancer.setCallbackFilter(new DaoFilter());
        enhancer.setInterceptDuringConstruction(false);
        
        Dao dao = (Dao)enhancer.create();
        dao.update();
        dao.select();
    }
    
}
```

运行结果为：

```
PeopleDao.update()
StartTime=[1491202022297]
PeopleDao.update()
EndTime=[1491202022311]
Before Method Invoke
PeopleDao.select()
After Method Invoke
```

看到第一次update()方法的调用，即Dao类构造方法中的调用没有拦截，符合预期。

 

## JDK动态代理

jdk动态代理的步骤：

1.实现InvocationHandler,重写其中的invoke（Object proxy,Method method,Object[] objects）方法（相当于切面），其中proxy是代理对象引用，method是目标方法，objects是目标方法的参数

2.使用Proxy.newProxyInstance(ClassLoader, new Class[] , InvocationHandler)来生成代理对象。其中classLoader是目标类的类加载器，new Class[]是代理对象实需要实现的接口，InvocationHandler是我们自己实现的那个Handler。

3.代理对象调用目标方法。

贴一个简单的代码：

```java
public interface IHello {

	
	void sayHello();
}

public class Hello implements IHello{

	@Override
	public void sayHello() {
		// TODO Auto-generated method stub
		System.out.println("hello");
	}
}

public class MyHandler implements InvocationHandler {

	private Object obj =null;
	
	public MyHandler(Object o) {
		this.obj = o;
	}
	
	
	@Override
	public Object invoke(Object arg0, Method method, Object[] arg2) throws Throwable {
		System.out.println("before");
        //注意：如果在这里传的obj是arg0会引起循环调用，因为arg0的method就是这个参数methods
		//所以必须在实例化handler的时候传入子类的实例对象
        Object  o = method.invoke(obj, arg2);
		System.out.println("after");
		return o;
	}

    
    
}

public class Main {

	public static void main(String[] args) {
		IHello proxy = (IHello) Proxy.newProxyInstance(IHello.class.getClassLoader(), new Class[] {IHello.class}, new MyHandler(new Hello()));
		proxy.sayHello();
	}
}
运行结果：
before
hello
after
```

**动态代理机制：** 
当通过代理对象proxy调用方法时：proxy.sayHello(); 
都将会被重定向到InvocationProxyHandler类对象（调用处理器）的invoke（）函数。 

这里的Hello必须继承接口，使用接口IHello进行动态代理，否则会报错。

因为jdk动态代理是通过接口来生成代理对象的，如下图：

![](https://3116004636-1256103796.cos.ap-guangzhou.myqcloud.com/jdk%E5%8A%A8%E6%80%81%E4%BB%A3%E7%90%86.png)

####**为什么一定要使用接口吗？**

看一个反编译出来的代理类：

```sql
public final class $Proxy0 extends Proxy implements IHello {

    private static final long serialVersionUID = 5351158173626517207L;

    private static Method m1;
    private static Method m3;
    private static Method m0;
    private static Method m2;

    public $Proxy0(InvocationHandler paramInvocationHandler) {
        super(paramInvocationHandler);
    }

    public final boolean equals(Object paramObject) {
        try {
            return ((Boolean) this.h.invoke(this, m1, new Object[] { paramObject })).booleanValue();
        } catch (Error | RuntimeException localError) {
            throw localError;
        } catch (Throwable localThrowable) {
            throw new UndeclaredThrowableException(localThrowable);
        }
    }

    public final void sayHello(String paramString1) {
        try {
            this.h.invoke(this, m3, new Object[] { paramString1});
            return;
        } catch (Error | RuntimeException localError) {
            throw localError;
        } catch (Throwable localThrowable) {
            throw new UndeclaredThrowableException(localThrowable);
        }
    }

    public final int hashCode() {
        try {
            return ((Integer) this.h.invoke(this, m0, null)).intValue();
        } catch (Error | RuntimeException localError) {
            throw localError;
        } catch (Throwable localThrowable) {
            throw new UndeclaredThrowableException(localThrowable);
        }
    }

    public final String toString() {
        try {
            return (String) this.h.invoke(this, m2, null);
        } catch (Error | RuntimeException localError) {
            throw localError;
        } catch (Throwable localThrowable) {
            throw new UndeclaredThrowableException(localThrowable);
        }
    }

    static {
        try {
            m1 = Class.forName("java.lang.Object").getMethod("equals",
                    new Class[] { Class.forName("java.lang.Object") });
            m3 = Class.forName("IHello").getMethod("sayHello",
                    new Class[] { Class.forName("java.lang.String"), Class.forName("java.lang.String") });
            m0 = Class.forName("java.lang.Object").getMethod("hashCode", new Class[0]);
            m2 = Class.forName("java.lang.Object").getMethod("toString", new Class[0]);
        } catch (NoSuchMethodException localNoSuchMethodException) {
            throw new NoSuchMethodError(localNoSuchMethodException.getMessage());
        } catch (ClassNotFoundException localClassNotFoundException) {
            throw new NoClassDefFoundError(localClassNotFoundException.getMessage());
        }
    }
}
```

看到这个方法，就算破案了。这个就是jdk动态代理和为什么需要接口。因为
1.在需要继承proxy类获得有关方法和InvocationHandler构造方法传参的同时,java不能同时继承两个类，我们需要和想要代理的类建立联系，只能实现一个接口

2.需要反射获得代理类的有关参数，必须要通过某个类，反射获取有关方法，如本次测试用的 :printSomeThing

3.成功返回的是object类型，要获取原类，只能继承/实现，或者就是那个代理类

4.对具体实现的方法内部并不关心，这个交给InvocationHandler.invoke那个方法里去处理就好了，我只想根据你给我的接口反射出对我有用的东西。

5.考虑到设计模式，以及proxy编者编写代码的逻辑使然



关于CGLIB的原理我还没有弄懂，目前只知道CGLIB的代理类继承了目标类，以后有机会再了解了。

```
最后我们总结一下JDK动态代理和Cglib动态代理的区别：
1.JDK动态代理是实现了被代理对象的接口，Cglib是继承了被代理对象。
2.JDK和Cglib都是在运行期生成字节码，JDK是直接写Class字节码，Cglib使用ASM框架写Class字节码，Cglib代理实现更复杂，生成代理类比JDK效率低。
3.JDK调用代理方法，是通过反射机制调用，Cglib是通过FastClass机制直接调用方法，Cglib执行效率更高。
```


## 缓存

默认缓存127到-128的Integer对象，这个缓存可以通过jvm参数：`-XX:AutoBoxCacheMax=<size>`来设置缓存的最大值，而且设置的值必须大于等于127

```java
    private static class IntegerCache {
        static final int low = -128;
        static final int high;
        static final Integer cache[];

        static {
            // high value may be configured by property
            int h = 127;
            String integerCacheHighPropValue =
                sun.misc.VM.getSavedProperty("java.lang.Integer.IntegerCache.high");
            if (integerCacheHighPropValue != null) {
                try {
                    int i = parseInt(integerCacheHighPropValue);
                    i = Math.max(i, 127);
                    // Maximum array size is Integer.MAX_VALUE
                    h = Math.min(i, Integer.MAX_VALUE - (-low) -1);
                } catch( NumberFormatException nfe) {
                    // If the property cannot be parsed into an int, ignore it.
                }
            }
            high = h
            cache = new Integer[(high - low) + 1];  
            int j = low;
            for(int k = 0; k < cache.length; k++)
                cache[k] = new Integer(j++);

            // range [-128, 127] must be interned (JLS7 5.1.7)
            assert IntegerCache.high >= 127;
        }

        private IntegerCache() {}
    }
```

##自动装箱

这样一行代码就是Integer的自动装箱

```java
Integer a = 1;
```

通过javap分析字节码，发现自动装箱使用的是`Integer.valueOf(int value);`方法来创建这个Integer对象

valueOf源码：

```java
    public static Integer valueOf(int i) {
        //先去取缓存
        if (i >= IntegerCache.low && i <= IntegerCache.high)
            return IntegerCache.cache[i + (-IntegerCache.low)];
        //如果缓存没有就new一个
        return new Integer(i);
    }
```

再来看下下面的代码：

```java
Integer a = 1;
Integer b = new Integer(1);
Integer c = Integer.valueOf(1);
Integer d = 1111;
Integer e = 1111;
System.out.println(a==b);
System.out.println(a==c);
System.out.println(d==e);
结果是什么？
答案是：
false
true
false
```

原因看上面的源码就知道了吧。



## equals和==的区别

- ==：如果是int、float这种基本数据类型，==比较的是值是否相等。如果是Object对象，比较的是内存地址。

- equals：Integer重写了equals方法，比较是这样的：

```java
    public boolean equals(Object obj) {
        if (obj instanceof Integer) {
            return value == ((Integer)obj).intValue();
        }
        return false;
    }
    public int intValue() {
        return value;
    }
```

可以看到比较的是值是否相等。

接下来看下面的代码：

```java
package test;
public class Main {
	public static void main(String[] args) {
		Integer a = 1;
		Integer b = 2;
		Integer c = 3;
		Integer d = 1;
		System.out.println(d==a);
		System.out.println(c==(a+b));
		System.out.println(c.equals(a+b));
	}
}
结果是什么？
答案是：
true
true
true
```

为什么？把javap -c的字节码解析结果贴上来:

```class
Compiled from "Main.java"
public class test.Main {
  public test.Main();
    Code:
       0: aload_0
       1: invokespecial #8                  // Method java/lang/Object."<init>":()V
       4: return

  public static void main(java.lang.String[]);
    Code:
       0: iconst_1
       1: invokestatic  #16                 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
       4: astore_1
       5: iconst_2
       6: invokestatic  #16                 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
       9: astore_2
      10: iconst_3
      11: invokestatic  #16                 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      14: astore_3
      15: iconst_1
      16: invokestatic  #16                 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      19: astore        4
      21: getstatic     #22                 // Field java/lang/System.out:Ljava/io/PrintStream;
      24: aload         4
      26: aload_1
      27: if_acmpne     34
      30: iconst_1
      31: goto          35
      34: iconst_0
      35: invokevirtual #28                 // Method java/io/PrintStream.println:(Z)V
      38: getstatic     #22                 // Field java/lang/System.out:Ljava/io/PrintStream;
      41: aload_3
      42: invokevirtual #34                 // Method java/lang/Integer.intValue:()I
      45: aload_1
      46: invokevirtual #34                 // Method java/lang/Integer.intValue:()I
      49: aload_2
      50: invokevirtual #34                 // Method java/lang/Integer.intValue:()I
      53: iadd
      54: if_icmpne     61
      57: iconst_1
      58: goto          62
      61: iconst_0
      62: invokevirtual #28                 // Method java/io/PrintStream.println:(Z)V
      65: getstatic     #22                 // Field java/lang/System.out:Ljava/io/PrintStream;
      68: aload_3
      69: aload_1
      70: invokevirtual #34                 // Method java/lang/Integer.intValue:()I
      73: aload_2
      74: invokevirtual #34                 // Method java/lang/Integer.intValue:()I
      77: iadd
      78: invokestatic  #16                 // Method java/lang/Integer.valueOf:(I)Ljava/lang/Integer;
      81: invokevirtual #38                 // Method java/lang/Integer.equals:(Ljava/lang/Object;)Z
      84: invokevirtual #28                 // Method java/io/PrintStream.println:(Z)V
      87: return
}
```

可以看到：

- d==a使用了`if_acmpne`比较了内存地址；

- c==(a+b)，分别调用了Method java/lang/Integer.intValue拿到三个对象的int值，然后进行了if_icmpne比较了int的值是否相等；

-  c.equals(a+b)，对(a+b)使用java/lang/Integer.valueOf()方法进行了自动装箱，然后使用equals方法比较。

答案明了。
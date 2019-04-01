##一.借助数组进行分页

原理：进行数据库查询操作时，获取到数据库中所有满足条件的记录，保存在应用的临时数组中，再通过List的subList方法，获取到满足条件的所有记录。

实现：

首先在dao层，创建StudentMapper接口，用于对数据库的操作。在接口中定义通过数组分页的查询方法，如下所示：

```java
 List<Student> queryStudentsByArray();
```


方法很简单，就是获取所有的数据，通过list接收后进行分页操作。

创建StudentMapper.xml文件，编写查询的sql语句：

```java
<select id="queryStudentsByArray"  resultMap="studentmapper">
        select * from student
 </select>
```

可以看出再编写sql语句的时候，我们并没有作任何分页的相关操作。这里是查询到所有的学生信息。

接下来在service层获取数据并且进行分页实现：

定义IStuService接口，并且定义分页方法：

```List&lt;Student&gt; queryStudentsByArray(int currPage, int pageSize);java
List<Student> queryStudentsByArray(int currPage, int pageSize);
```



通过接收currPage参数表示显示第几页的数据，pageSize表示每页显示的数据条数。

创建IStuService接口实现类StuServiceIml对方法进行实现，对获取到的数组通过currPage和pageSize进行分页：

```java
 @Override
    public List<Student> queryStudentsByArray(int currPage, int pageSize) {
        List<Student> students = studentMapper.queryStudentsByArray();
//        从第几条数据开始
        int firstIndex = (currPage - 1) * pageSize;
//        到第几条数据结束
        int lastIndex = currPage * pageSize;
        return students.subList(firstIndex, lastIndex);
    }
```

通过subList方法，获取到两个索引间的所有数据。



## 二、借助Sql语句进行分页

在了解到通过数组分页的缺陷后，我们发现不能每次都对数据库中的所有数据都检索。然后在程序中对获取到的大量数据进行二次操作，这样对空间和性能都是极大的损耗。所以我们希望能直接在数据库语言中只检索符合条件的记录，不需要在通过程序对其作处理。这时，Sql语句分页技术横空出世。

实现：通过sql语句实现分页也是非常简单的，只是需要改变我们查询的语句就能实现了，即在sql语句后面添加limit分页语句。

然后在StudentMapper.xml文件中编写sql语句通过limiy关键字进行分页：

```java
 <select id="queryStudentsBySql" parameterType="map" resultMap="studentmapper">
        select * from student limit #{currIndex} , #{pageSize}
</select>
```



##三.拦截器分页

上面提到的数组分页和sql语句分页都不是我们今天讲解的重点，今天需要实现的是利用拦截器达到分页的效果。自定义拦截器实现了拦截所有以ByPage结尾的查询语句，并且利用获取到的分页相关参数统一在sql语句后面加上limit分页的相关语句，一劳永逸。不再需要在每个语句中单独去配置分页相关的参数了。。

首先我们看一下拦截器的具体实现，在这里我们需要拦截所有以ByPage结尾的所有查询语句，因此要使用该拦截器实现分页功能，那么再定义名称的时候需要满足它拦截的规则（以ByPage结尾），如下所示：

```java
package com.cbg.interceptor;

import org.apache.ibatis.executor.Executor;
import org.apache.ibatis.executor.parameter.ParameterHandler;
import org.apache.ibatis.executor.resultset.ResultSetHandler;
import org.apache.ibatis.executor.statement.StatementHandler;
import org.apache.ibatis.mapping.MappedStatement;
import org.apache.ibatis.plugin.*;
import org.apache.ibatis.reflection.MetaObject;
import org.apache.ibatis.reflection.SystemMetaObject;

import java.sql.Connection;
import java.util.Map;
import java.util.Properties;

/**
 * Created by chenboge on 2017/5/7.
 * <p>
 * Email:baigegechen@gmail.com
 * <p>
 * description:
 */

/**
 * @Intercepts 说明是一个拦截器
 * @Signature 拦截器的签名
 * type 拦截的类型 四大对象之一( Executor,ResultSetHandler,ParameterHandler,StatementHandler)
 * method 拦截的方法
 * args 参数
 */
@Intercepts({@Signature(type = StatementHandler.class, method = "prepare", args = {Connection.class, Integer.class})})
public class MyPageInterceptor implements Interceptor {

//每页显示的条目数
    private int pageSize;
//当前现实的页数
    private int currPage;

    private String dbType;


    @Override
    public Object intercept(Invocation invocation) throws Throwable {
        //获取StatementHandler，默认是RoutingStatementHandler
        StatementHandler statementHandler = (StatementHandler) invocation.getTarget();
        //获取statementHandler包装类
        MetaObject MetaObjectHandler = SystemMetaObject.forObject(statementHandler);

        //分离代理对象链
        while (MetaObjectHandler.hasGetter("h")) {
            Object obj = MetaObjectHandler.getValue("h");
            MetaObjectHandler = SystemMetaObject.forObject(obj);
        }

        while (MetaObjectHandler.hasGetter("target")) {
            Object obj = MetaObjectHandler.getValue("target");
            MetaObjectHandler = SystemMetaObject.forObject(obj);
        }

        //获取连接对象
        //Connection connection = (Connection) invocation.getArgs()[0];


        //object.getValue("delegate");  获取StatementHandler的实现类

        //获取查询接口映射的相关信息
        MappedStatement mappedStatement = (MappedStatement) MetaObjectHandler.getValue("delegate.mappedStatement");
        String mapId = mappedStatement.getId();

        //statementHandler.getBoundSql().getParameterObject();

        //拦截以.ByPage结尾的请求，分页功能的统一实现
        if (mapId.matches(".+ByPage$")) {
            //获取进行数据库操作时管理参数的handler
            ParameterHandler parameterHandler = (ParameterHandler) MetaObjectHandler.getValue("delegate.parameterHandler");
            //获取请求时的参数
            Map<String, Object> paraObject = (Map<String, Object>) parameterHandler.getParameterObject();
            //也可以这样获取
            //paraObject = (Map<String, Object>) statementHandler.getBoundSql().getParameterObject();

            //参数名称和在service中设置到map中的名称一致
            currPage = (int) paraObject.get("currPage");
            pageSize = (int) paraObject.get("pageSize");

            String sql = (String) MetaObjectHandler.getValue("delegate.boundSql.sql");
            //也可以通过statementHandler直接获取
            //sql = statementHandler.getBoundSql().getSql();

            //构建分页功能的sql语句
            String limitSql;
            sql = sql.trim();
            limitSql = sql + " limit " + (currPage - 1) * pageSize + "," + pageSize;

            //将构建完成的分页sql语句赋值个体'delegate.boundSql.sql'，偷天换日
            MetaObjectHandler.setValue("delegate.boundSql.sql", limitSql);
        }
//调用原对象的方法，进入责任链的下一级
        return invocation.proceed();
    }


    //获取代理对象
    @Override
    public Object plugin(Object o) {
    //生成object对象的动态代理对象
        return Plugin.wrap(o, this);
    }

    //设置代理对象的参数
    @Override
    public void setProperties(Properties properties) {
//如果项目中分页的pageSize是统一的，也可以在这里统一配置和获取，这样就不用每次请求都传递pageSize参数了。参数是在配置拦截器时配置的。
        String limit1 = properties.getProperty("limit", "10");
        this.pageSize = Integer.valueOf(limit1);
        this.dbType = properties.getProperty("dbType", "mysql");
    }
}
```

编写好拦截器后，需要注册到项目中，才能发挥它的作用。在mybatis的配置文件中，添加如下代码：

```java
<plugins>
    <plugin interceptor="com.cbg.interceptor.MyPageInterceptor">
        <property name="limit" value="10"/>
        <property name="dbType" value="mysql"/>
    </plugin>
</plugins>
```
如上所示，还能在里面配置一些属性，在拦截器的setProperties方法中可以获取配置好的属性值。如项目分页的pageSize参数的值固定，我们就可以配置在这里了，以后就不需要每次传入pageSize了，读取方式如下：

```java
 //读取配置的代理对象的参数
    @Override
    public void setProperties(Properties properties) {
        String limit1 = properties.getProperty("limit", "10");
        this.pageSize = Integer.valueOf(limit1);
        this.dbType = properties.getProperty("dbType", "mysql");

  }
```

##四.RowBounds实现分页

原理：通过RowBounds实现分页和通过数组方式分页原理差不多，都是一次获取所有符合条件的数据，然后在内存中对大数据进行操作，实现分页效果。只是数组分页需要我们自己去实现分页逻辑，这里更加简化而已。

存在问题：一次性从数据库获取的数据可能会很多，对内存的消耗很大，可能导师性能变差，甚至引发内存溢出。

适用场景：在数据量很大的情况下，建议还是适用拦截器实现分页效果。RowBounds建议在数据量相对较小的情况下使用。

简单介绍：这是代码实现上最简单的一种分页方式，只需要在dao层接口中要实现分页的方法中加入RowBounds参数，然后在service层通过offset（从第几行开始读取数据，默认值为0）和limit（要显示的记录条数，默认为java允许的最大整数：2147483647）两个参数构建出RowBounds对象，在调用dao层方法的时，将构造好的RowBounds传进去就能轻松实现分页效果了。

具体操作如下：

dao层接口方法：

```java
//加入RowBounds参数
public List<UserBean> queryUsersByPage(String userName, RowBounds rowBounds);
```

然后在service层构建RowBounds，调用dao层方法：

```java
  @Override
    @Transactional(isolation = Isolation.READ_COMMITTED, propagation = Propagation.SUPPORTS)
    public List<RoleBean> queryRolesByPage(String roleName, int start, int limit) {
        return roleDao.queryRolesByPage(roleName, new RowBounds(start, limit));

    }
```

RowBounds就是一个封装了offset和limit简单类，如下所示：

```java
public class RowBounds {
    public static final int NO_ROW_OFFSET = 0;
    public static final int NO_ROW_LIMIT = 2147483647;
    public static final RowBounds DEFAULT = new RowBounds();
    private int offset;
    private int limit;

    public RowBounds() {
        this.offset = 0;
        this.limit = 2147483647;
    }
    
    public RowBounds(int offset, int limit) {
        this.offset = offset;
        this.limit = limit;
    }
    
    public int getOffset() {
        return this.offset;
    }
    
    public int getLimit() {
        return this.limit;
    }

}
```



copy from https://blog.csdn.net/chenbaige/article/details/70846902
# Session工作原理

##1、tomcat中的session保存在哪里？
答：保存在tomcat本地的concurrentHashMap中，以sessionId为key

##2、tomcat是怎么追踪到请求时属于哪个session的？
答：通过cookie：产生会话时向浏览器发送存有sessionId的cookie，后续请求时带上这个cookie。

如果禁用了cookie，可以使用url重写的方法，调用response.encodeURL（url）将sessionId放在url中

##3、session是不是在用户登录时产生的？

答：不是，只要在后台调用了request.getSession，如果当前没有session，就会创建一个session，并将sessionId写到cookie。在jsp中默认开启了<%@ page session="true"%>，所以访问jsp页面的时候就创建了session
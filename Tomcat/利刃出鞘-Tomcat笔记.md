## 连接器 - Coyote

Coyote是Tomcat的连接器框架的名称，是Tomcat服务提供的供客户端访问的外部接口。客户端通过Coyote与服务器建立连接、发送请求并接收响应。

Coyote封装了底层的网络通信（Socket请求及响应处理），为Catalina容器提供了统一的接口，使Catalina容器与具体的协议及IO操作方式完全解耦。

Coyote作为独立的模块，只负责具体协议和IO相关操作，与Servlet规范实现没有直接关系，因此即便是Request与Response对象也并未实现Servlet规范对应的接口，而是在Catalina中将他们进一步封装为ServletRequest和ServletResponse。

### 连接器组件

![连接器组件](https://s2.ax1x.com/2020/03/01/32DYJe.png)

## 容器 - Catalina

Catalina是Servlet容器实现，包含了之前讲到的所有的容器组件，以及后续章节涉及到的安全、集群、管理等Servlet容器架构的各个方面。它通过松耦合的方式集成Coyote，以完成按照请求协议进行数据读写。同时，它还包括我们的启动入口、Shell程序等。

### Catalina架构

![Catalina架构](https://s2.ax1x.com/2020/03/01/32rKfg.png)

| 组件      | 职责                                                         |
| --------- | ------------------------------------------------------------ |
| Catalina  | 负责解析Tomcat的配置文件，以此来创建服务器Server组件，并根据命令来对其进行管理 |
| Server    | 服务器表示整个Catalina Servlet容器以及其他组件，负责组装并启动Servlet引擎，Tomcat连接器。Server通过实现LifeCycle接口，提供了一种优雅的启动和关闭整个系统的方式。 |
| Service   | 服务是Server内部组件，一个Server包含多个Service。它将若干个Connector组件绑定到一个Container（Engine）上 |
| Connector | 连接器，处理与客户端的通信，它负责接收客户请求，然后转给相关的容器处理1，最后向客户响应返回结果。 |
| Container | 容器，负责处理用户的Servlet请求，并返回对象给web用户的模块。 |

### Container结构

![Container结构图](https://s2.ax1x.com/2020/03/01/32sr8g.png)



## Tomcat启动流程

![启动时序图](https://s2.ax1x.com/2020/03/01/32yKMj.png)

### 源码解析

#### LifeCycle

核心方法：

- init ：初始化组件
- start：启动组件
- stop：停止组件
- destroy：销毁组件



## Tomcat请求处理流程

![tomcat请求处理流程图](https://s2.ax1x.com/2020/03/02/3WtCS1.png)

![源码解析](https://s2.ax1x.com/2020/03/02/3WUiVO.png)



## Jasper

Jasper模块是Tomcat的JSP核心引擎，JSP本质是一个Servlet。Tomcat使用Jasper对JSP语法进行解析，生成Servlet并生成Class字节码，用户在进行访问时，会访问Servlet，最终将返回的结果直接响应在浏览器。另外，在运行的时候，Jasper还会检测Jsp文件是否修改，如果修改，则会重新编译JSP文件。

### JSP编译方式

- 运行时编译

![JspServlet执行流程](https://s2.ax1x.com/2020/03/02/3fZFBt.png)

- 预编译：Tomcat提供一个Shell程序JspC，用于支持JSP预编译，而且在Tomcat安装目录下提供了一个catalina-tasks.xml文件声明了Tomcat支持的Ant任务，因此，我们很容易使用Ant来执行JSP预编译。

### JSP编译原理

![JSP编译](https://s2.ax1x.com/2020/03/02/3fmAfS.png)



## JVM配置

### JVM配置选项

windows平台（catalina.bat）

```shell
set JAVA_OPTS=-server -Xms2048m -Xmx2048m -XX:MetaspaceSize=256m -XX:MaxMetaspaeceSize=256m -XX:SurvivorRatio=8
```

linux平台（catalina.sh）

```powershell
JAVA_OPTS="-server -Xms2048m -Xmx2048m -XX:MetaspaceSize=256m -XX:MaxMetaspaeceSize=256m -XX:SurvivorRatio=8"
```

![参数说明](https://s2.ax1x.com/2020/03/02/3f3VLF.png)



## Tomcat集群

### Session共享方案

1、 ip_hash策略

2、 Session复制

3、 SSO单点登录

![单点登录](https://s2.ax1x.com/2020/03/03/3fJNo4.png)



## Tomcat安全

### 配置安全

![配置安全](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1583173153678.png)

### 应用安全

SpringSecurity、Shiro等

### 传输安全

配置Https

![https](https://s2.ax1x.com/2020/03/03/3fYcj0.png)



## Tomcat性能优化

### JVM参数调优

![JVM参数调优](https://s2.ax1x.com/2020/03/03/3fNPJJ.png)

### 垃圾收集器配置调整

![垃圾收集器配置](https://s2.ax1x.com/2020/03/03/3fUZ1s.png)

测试的时候可以打印垃圾收集信息

![GCDetail](https://s2.ax1x.com/2020/03/03/3fUBND.png)

### 连接器配置

![连接器配置](C:\Users\Administrator\AppData\Roaming\Typora\typora-user-images\1583175812783.png)




**vhost本质上是一个mini版的RabbitMQ服务器，拥有自己的队列、绑定、交换器和权限控制；**

**vhost通过在各个实例间提供逻辑上分离，允许你为不同应用程序安全保密地运行数据；**

vhost是AMQP概念的基础，必须在连接时进行指定，RabbitMQ包含了默认vhost：“/”；

当在RabbitMQ中创建一个用户时，用户通常会被指派给至少一个vhost，并且只能访问被指派vhost内的队列、交换器和绑定，vhost之间是绝对隔离的。

vhost操作：

rabbitmqctl add_vhost [vhost_name] #创建vhost

rabbitmqctl delete_vhost [vhost_name] #删除vhost

rabbitmqctl list_vhosts #查看

配置最大连接限制，0：表示不可用，-1：无限制

rabbitmqctl set_vhost_limits -p vhost_name '{"max-connections": 256}'

配置队列最大数，-1：无限制

rabbitmqctl set_vhost_limits -p vhost_name '{"max-queues": 1024}'

RabbitMQ is multi-tenant system: connections, exchanges, queues, bindings, user permissions, policies and some other things belong to virtual hosts, logical groups of entities.









作者：骑猪喝咖啡 
来源：CSDN 
原文：https://blog.csdn.net/hqwang4/article/details/81706090 
版权声明：本文为博主原创文章，转载请附上博文链接！
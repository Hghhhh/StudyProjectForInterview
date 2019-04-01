

root用户如果不能远程连接数据库

grant all on *.* to admin@'%' identified by '123456' with grant option; 
flush privileges;
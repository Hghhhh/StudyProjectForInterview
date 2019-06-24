简单命令：

- nginx -s reload
- nginx -s stop
- 启动：    ./usr/sbin/nginx
- access.log 和error.log 的地址 ：/var/log/nginx
- 主nginx.conf地址：/etc/nginx
- 自己配置*.conf可以放在/etc/nginx/conf.d



配置限流：

```conf
geo  $limited  { # the variable created is $limited
      default          1;
      127.0.0.1/32     0;
    }

map $limited $limit {
    1 $binary_remote_addr;
    0 "";
    }

limit_req_zone $binary_remote_addr zone=req_one:20m rate=12r/s;
limit_conn_zone $binary_remote_addr zone=addr:10m;
limit_conn_zone $server_name zone=perserver:20m;

server{

	listen 80;
	
	location / {
		proxy_pass http://13.11.123.31:80;
		limit_req zone=req_one burst=100 nodelay;
		limit_conn addr 15;
	}
	
}

```


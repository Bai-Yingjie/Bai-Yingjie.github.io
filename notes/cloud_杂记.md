- [nginx负载均衡](#nginx负载均衡)
  - [负载均衡mode](#负载均衡mode)
  - [server 权重](#server-权重)
  - [用DNS服务来更新server的IP](#用dns服务来更新server的ip)

# nginx负载均衡
nginx可以做URL的静态路由和负载均衡, 分到不同的应用微服务:
比如下面的配置使用round robin方式负载均衡到两个服务器
```
http {
    upstream backend {
        server backend1.example.com;
        server backend2.example.com;
        server 192.0.0.1 backup;
    }
    
    server {
        location / {
            proxy_pass http://backend;
        }
    }
}
```

## 负载均衡mode
默认是Round Robin, 还有least_conn, ip_hash, least_time, random等等, i.e.
```
upstream backend {
    least_conn;
    server backend1.example.com;
    server backend2.example.com;
}
```

## server 权重
可以配权重来反应不同server的性能配置, weight默认是1, 性能越好的, weight越高.
```
upstream backend {
    server backend1.example.com weight=5;
    server backend2.example.com;
    server 192.0.0.1 backup;
}
```

## 用DNS服务来更新server的IP
nginx会周期的跟10.0.0.1 check, 更新backend1.example.com和backend2.example.com的ip地址, 如果更换server了, 只要域名不变, ip可以run time变更, 不用重启.
```
http {
    resolver 10.0.0.1 valid=300s ipv6=off;
    resolver_timeout 10s;
    server {
        location / {
            proxy_pass http://backend;
        }
    }
    upstream backend {
        zone backend 32k;
        least_conn;
        # ...
        server backend1.example.com resolve;
        server backend2.example.com resolve;
    }
}
```
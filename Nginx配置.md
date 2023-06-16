# Nginx下载和安装

## windows

 [nginx下载](https://nginx.org/en/download.html)

## Linux

自行查找对应系统安装教程

以下默认安装路径为 `/usr/local/nginx`





# 配置部分

配置文件中的默认路径是你的安装路径，因此第一步建议创建配置文件用的文件夹。尽量避免堆放在nginx.conf文件中，难以管理

```bash
cd /usr/local/nginx
mkdir cert  #ssl证书存放文件夹
cd conf
mkdir conf.d  #配置文件夹
chmod 777 conf.d
```

接下来配置主配置文件nginx.conf，直接把其中的内容替换为以下内容

nginx.conf

```
# Nginx运行的用户和用户组
# user nobody;
# 工作进程：数目。根据硬件调整，通常等于CPU数量或者2倍于CPU。
worker_processes  1;

# #全局错误日志定义类型[ debug | info | notice | warn | error | crit ]
error_log  logs/error.log;

# 进程pid文件
pid        logs/nginx.pid;

events {
	# 每个工作进程的最大连接数量。
    worker_connections  1024;
}

# 设定http服务器，利用它的反向代理功能提供负载均衡支持
http {
	# 设定mime类型,类型由mime.type文件定义
    include       mime.types;
    # 默认文件类型
    default_type  application/octet-stream;
    
	# 日志格式设置。
    log_format  main  '$remote_addr - $remote_user [$time_local] "$request" '
                      '$status $body_bytes_sent "$http_referer" '
                      '"$http_user_agent" "$http_x_forwarded_for"';
	
	# 日志文件的存放路径
    access_log  logs/access.log  main;

	# #开启高效文件传输模式，sendfile指令指定nginx是否调用sendfile函数来输出文件，对于普通应用设为 on，如果用来进行下载等应用磁盘IO重负载应用，可设置为off，以平衡磁盘与网络I/O处理速度，降低系统的负载。注意：如果图片显示不正常把这个改成off。
    sendfile on;

    # keepalive超时时间
    keepalive_timeout 65;

	# 包含和关联各个域名配置文件
    include conf.d/*.conf;
}
```



配置好主配置文件后就可以在conf.d里添加域名的配置文件了，建议一个域名一个文件，方便管理

例如 xxx.com.conf

```
# 配置server
server{
	# 负载均衡策略，默认轮询
    upstream myServer {
		server 127.0.0.1:8001;
		server 127.0.0.1:8002;
	}
	
	# 监听端口
	listen 80;
	
	# 访问域名
    server_name *.xxx.com;
    
    return 301 https://$http_host$request_uri;
	
	# 日志文件的存放路径
    access_log  logs/xxx.com.access.log;
	error_log   logs/xxx.com.error.log;
  
	error_page  403 /403.html;
	error_page  404 /404.html;
	error_page  500 502 503 504  /500.html;

}

#负载均衡用的
upstream server_list{
	server localhost:8080;
	server localhost:8090;
}

server {
    listen 443 ssl http2;
    server_name *.xxx.com;

    ssl_certificate     /usr/local/nginx/cert/*.xxx/*.xxx.com.cer;  #还有可能是.pem文件
    ssl_certificate_key /usr/local/nginx/cert/*.xxx/*.xxx.com.key;  # 这里建议写绝对路径，保证不出错
    ssl_session_timeout 5m;
    ssl_protocols TLSV1 TLSv1.1 TLSv1.2;
    ssl_ciphers ECDHE-RSA-AES128-GCM-SHA256:ECDHE:ECDH:AES:HIGH:!NULL:!aNULL:!MD5:!ADH:!RC4;
    ssl_prefer_server_ciphers on;
    
    access_log logs/xxx.com.access.log;
    error_log  logs/xxx.com.error.log;
                
    location / {
        proxy_http_version 1.1;
        proxy_set_header Upgrade $http_upgrade;
        proxy_set_header Connection "upgrade";

        proxy_set_header X-Real-IP $remote_addr;
        proxy_set_header X-Forwarded-For $remote_addr;
        proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
        proxy_set_header Host $host;
        proxy_redirect off;

        proxy_pass http://server_list;
    }
    
    location ~* /.svn/ {
        deny all;
    }

    location ~* \.(tar|gz|zip|tgz|sh)$ {
        deny all;
    }
}

```



## 其他配置项

**负载均衡配置状态**

| 状态         | 概述                                                         |
| ------------ | ------------------------------------------------------------ |
| down         | 当前的server暂不参与负载均衡                                 |
| backup       | 预留的备份服务器，当其他服务器都挂掉时启用                   |
| max_fails    | 允许请求失败次数，如果请求失败次数超过限制，则经过fail_timeout时间后从虚拟服务池中kill掉该服务器 |
| fail_timeout | 经过max_fails失败后，服务器暂停时间，max_fails设置后必须设置此值 |
| max_conns    | 限制最大的连接数，用于服务器硬件配置不同的情况下             |

```
upstream node {
	server IP地址:8081 down;
	server IP地址:8082 backup;
	server IP地址:8081 max_fails=1 fail_timeout=10s;
}
```



**负载均衡调度策略**

| 调度算法     | 概述                                                         |
| ------------ | ------------------------------------------------------------ |
| 轮询         | 逐一轮询，默认方式                                           |
| weight       | 加权轮询，weight越大，分配几率越高                           |
| ip_hash      | 按照访问IP的hash结果分配，会导致来自同一IP的请求固定访问一台服务器 |
| url_hash     | 按照访问URL的hash结果分配                                    |
| least_conn   | 最少链接数，哪个服务器链接数少分配给谁                       |
| hash关键数值 | hash自定义的key                                              |

```
upstream node {
	server IP地址:8081 weight=10;
	server IP地址:8082 weight=20;
	server IP地址:8083 weight=30;
}
```

```
upstream node {
	hash $request_uri;
	server IP地址:8081;
	server IP地址:8082;
	server IP地址:8083;
}
```



**location匹配**

```
location [ = | ~ | ~* | ^~ ] uri{}
```

- `=`：用于不含正则表达式的uri前，要求请求字符串与uri严格匹配，匹配成功就停止向下搜索并立即处理该请求
- `~`：用于表示uri包含正则表达式，并且区分大小写
- `~*`：用于表示uri包含正则表达式，并且**不**区分大小写
- `^~`：用于不含正则表达式的uri前，要求Nginx服务器找到标识uri和请求，字符串匹配度最高的location后，立即使用此location处理请求，而不再使用location块中的正则uri和请求字符串做匹配

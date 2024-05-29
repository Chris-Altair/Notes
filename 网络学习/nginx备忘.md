nginx开启gzip

```nginx
gzip on;
gzip_types text/plain application/json;
```

设置成功后，response header会出现***Content-Encoding: gzip***

压缩前有300kb，压缩后只有30kb，可以说效果非常明显了



nginx校验host头，请求到nginx会校验head的Host，如果Host不符合要求会失败

```nginx
server {
    listen 443 ssl http2;
    server_name 192.168.1.32;
    if ($http_Host !~* ^192.168.1.32$){
        return 403;
    } 
}
```


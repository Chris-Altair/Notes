nginx开启gzip

```json
gzip on;
gzip_types text/plain application/json;
```

设置成功后，response header会出现***Content-Encoding: gzip***

压缩前有300kb，压缩后只有30kb，可以说效果非常明显了
https分为单向认证和双向认证

## 一、 https单向认证

调用单向认证的https接口，只需要获取证书的公钥即可。

注意：curl访问自定义证书要加参数 -k 不校验证书合法性。

证书可以通过浏览器导出base64格式的cer证书文件，然后可通过openssl或keytool提取证书中的公钥

```bash
# keytool提取公钥
keytool -import -v -alias test -file XXX.cer -keystore XXX.jks -storepass Aa123456 -noprompt
# openssl提取公钥
openssl x509 -in XXX.cer -pubkey  -noout > XXX.pem
```

## 二、nginx https单向证书配置（使用openssl）

### 1.  生成证书和密钥

```bash
# 1.创建服务器密钥server.key
openssl genrsa -out server.key 2048
# 2.创建服务器证书的申请文件server.csr
openssl req -new -key server.key -out server.csr # Common Name填ip或域名，其他随意，但不填浏览器会提示不安全
# 3.创建CA证书ca.crt(10年有效)，这个证书用来给自己的证书签名（输入参数需要跟上面保持一致）
openssl req -new -x509 -key server.key -out ca.crt -days 3650
# 4.创建自当前日期起有效期为期十年的服务器证书server.crt（server.crt和server.key就是你的nginx需要的证书文件和密钥）
openssl x509 -req -days 3650 -in server.csr -CA ca.crt -CAkey server.key -CAcreateserial -out server.crt
```

### 2.  nginx配置

```bash
server{
        listen 443 ssl http2; #必填,高版本nginx可选择是否配置http2
        ssl_certificate /home/***/cert/server.crt; #配置证书路径,从浏览器导出的证书本质上就是这个！！！（必填）
        ssl_certificate_key /home/***/cert/server.key; #配置密钥位置（必填）
        ssl_session_timeout  5m;
        ssl_protocols  SSLv2 SSLv3 TLSv1;
        ssl_ciphers  ALL:!ADH:!EXPORT56:RC4+RSA:+HIGH:+MEDIUM:+LOW:+SSLv2:+EXP;
        ssl_prefer_server_ciphers on;
        server_name localhost;
        location / {
            proxy_pass http://127.0.0.1:8888; #代理到本地8888端口
        }
}
```

> 参考：
> 1.https://www.jianshu.com/p/9523d888cf77 # 配置https单项证书nginx
> 2.https://www.jianshu.com/p/57066821b863 # 配置https双项证书nginx
> 3.https://juejin.im/post/6844903809068564493 # 扯一扯HTTPS单向认证、双向认证、抓包原理、反抓包策略
> 4.https://www.jianshu.com/p/dff237d75516 # 证书密钥转换
> 5.https://www.cnblogs.com/MomentsLee/p/10460832.html # 证书格式种类


[TOC]

jwt全称json web token，是一种无状态的认证令牌，分为三部分

- header：一般规定加密算法和类型

- payload：包括唯一标识，生成时间及过期时间等，可自定义属性

- sign：header和payload的签名，用于保证token的合法性

  构造方式：

```
jwt=base64url(header).base64url(payload).base64url(sign(base64url(header)+"."+base64url(payload),privateKey))
```

**注意：header和payload仅使用base64url编码，禁止存放敏感信息！**

### 1.header

```json
// alg：表示jwt使用的签名算法
// typ：表示token类型
{"alg":"RS256","typ":"JWT"}
```

### 2.payload

```json
// jti：令牌唯一标识
// ati：令牌生成时间，格式为unix时间戳
// exp：令牌过期时间，格式为unix时间戳
```
### 3.java实现jwt的构造和验证

```java
package cn.com.taiji.util;

import cn.hutool.core.codec.Base64;
import com.alibaba.fastjson.JSONObject;
import com.sun.jersey.api.client.Client;
import com.sun.jersey.api.client.ClientResponse;
import com.sun.jersey.api.client.WebResource;
import org.apache.commons.lang3.RandomStringUtils;
import org.apache.commons.lang3.StringUtils;
import org.slf4j.Logger;
import org.slf4j.LoggerFactory;
import org.springframework.beans.factory.annotation.Autowired;
import org.springframework.beans.factory.annotation.Value;
import org.springframework.stereotype.Component;

import javax.servlet.http.HttpServletRequest;
import javax.servlet.http.HttpSession;
import javax.ws.rs.core.MediaType;
import java.io.UnsupportedEncodingException;
import java.net.URLEncoder;
import java.security.*;
import java.security.interfaces.RSAPublicKey;
import java.security.spec.InvalidKeySpecException;
import java.security.spec.KeySpec;
import java.security.spec.PKCS8EncodedKeySpec;
import java.security.spec.X509EncodedKeySpec;
import java.util.HashMap;
import java.util.Map;

@Component
public class ClientUtils {
    private static Logger logger = LoggerFactory.getLogger(ClientUtils.class);

    @Autowired
    private Client client;

    @Value("${sso.am_url}")
    private String amUrl;
    @Value("${sso.client_id}")
    private String clientId;
    @Value("${sso.client_secret}")
    private String clientSecret;
    @Value("${sso.redirect_uri}")
    private String redirectUri;
    @Value("${sso.response_type}")
    private String responseType;
    @Value("${sso.grant_type}")
    private String grantType;
    @Value("${sso.scope}")
    private String scope;

    public String getState() {
        return RandomStringUtils.randomAlphabetic(6);
    }

    public String buildOauth2Req() {
        StringBuilder reqBuilder = new StringBuilder(amUrl);
        reqBuilder.append("/oauth/authorize?client_id=").append(clientId)
                .append("&redirect_uri=").append(redirectUri)
                .append("&response_type=").append(responseType)
//                .append("&state=" + getState())
//                .append("&client_secret=").append(clientSecret) //建议去掉，因为这个请求暴露在浏览器中
        ;
        return reqBuilder.toString();
    }

    private Map<String, String> fetchTokenMap(String code) throws UnsupportedEncodingException {
        StringBuilder accessTokenURLBuilder = new StringBuilder(amUrl);
        accessTokenURLBuilder.append("/oauth/token?client_id=").append(clientId)
                .append("&grant_type=").append(grantType)
                .append("&scope=").append(scope)
                .append("&redirect_uri=")
                .append(URLEncoder.encode(redirectUri, "UTF-8"))
                .append("&client_secret=").append(clientSecret)
                .append("&code=").append(code);
        logger.debug("获取访问令牌地址:" + accessTokenURLBuilder);
        WebResource clientResource = client.resource(accessTokenURLBuilder.toString());
        ClientResponse clientResponse = clientResource.accept(MediaType.APPLICATION_JSON).post(ClientResponse.class);
        String result = clientResponse.getEntity(String.class);
        logger.debug("result:{}", result);
        JSONObject json = JSONObject.parseObject(result);
        String accessToken = (String) json.get("access_token");
        String refreshToken = (String) json.get("refresh_token");
        if (StringUtils.isEmpty(accessToken) || StringUtils.isEmpty(refreshToken)) {
            logger.error("访问令牌为空！错误信息：{}", result);
            throw new RuntimeException("调用获取访问令牌接口为空！错误信息：" + result);
        }
        Map<String, String> tokenMap = new HashMap<>();
        tokenMap.put("accessToken", accessToken);
        tokenMap.put("refreshToken", refreshToken);
        return tokenMap;
    }

    /**
     * 根据code获取token
     *
     * @param code
     * @return
     */
    public String getUserInfo(String code, HttpServletRequest request) {
        HttpSession session = request.getSession();
        String accessToken = "";
        Map<String, String> tokenMap;
        try {
            tokenMap = fetchTokenMap(code);
            accessToken = tokenMap.get("accessToken");
            session.setAttribute("accessToken", tokenMap.get("accessToken"));
            session.setAttribute("refreshToken", tokenMap.get("refreshToken"));
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
            throw new RuntimeException("获取token失败");
        }
        StringBuilder userInfoURLBuilder = new StringBuilder(amUrl);
        userInfoURLBuilder.append("/api/user");
        WebResource clientResource = client.resource(userInfoURLBuilder.toString());
        ClientResponse clientResponse = clientResource
                .header("Authorization", "bearer " + accessToken)
                .accept(MediaType.APPLICATION_JSON).get(ClientResponse.class);
        String result = clientResponse.getEntity(String.class);
        logger.debug("result:{}", result);
        return result;
    }

    /**
     * 刷新令牌
     *
     * @param refreshToken
     * @return
     */
    public String refreshAsseccToken(String refreshToken) {
        StringBuilder refreshTokenURLBuilder = new StringBuilder(amUrl);
        refreshTokenURLBuilder.append("/oauth/token?")
                .append("grant_type=").append("refresh_token")
                .append("&refresh_token=").append(refreshToken);
        logger.debug("刷新访问令牌地址:" + refreshTokenURLBuilder);
        WebResource clientResource = client.resource(refreshTokenURLBuilder.toString());
        ClientResponse clientResponse = clientResource.accept(MediaType.APPLICATION_JSON).
                header("Authorization", "Basic " + Base64.encode(clientId + ":" + clientSecret)).
                post(ClientResponse.class);
        String result = clientResponse.getEntity(String.class);
        logger.debug("result:{}", result);
        JSONObject json = JSONObject.parseObject(result);
        String newAsseccToken = (String) json.get("access_token");
        if (StringUtils.isEmpty(newAsseccToken)) {
            logger.error("访问令牌为空！错误信息：{}", result);
            throw new RuntimeException("调用刷新访问令牌接口为空！错误信息：" + result);
        }
        return newAsseccToken;
    }

    /**
     * 检查jwt的sign是否合法
     * @param jwt
     * @return
     */
    public boolean checkJwtSign(String jwt) {
        StringBuilder tokenKeyURLBuilder = new StringBuilder(amUrl);
        tokenKeyURLBuilder.append("/oauth/token_key");
        WebResource clientResource = new Client().resource(tokenKeyURLBuilder.toString());
        ClientResponse clientResponse = clientResource.accept(MediaType.APPLICATION_JSON).get(ClientResponse.class);
        String result = clientResponse.getEntity(String.class);
        JSONObject json = JSONObject.parseObject(result);
        String alg = (String) json.get("alg");
        String pk = json.getString("value")
                .replace("-----BEGIN PUBLIC KEY-----", "")
                .replace("\n", "")
                .replace("-----END PUBLIC KEY-----", "");
        try {
            String[] jwts = jwt.split("\\.");
            String concatenated = jwts[0] + "." + jwts[1];
            byte[] data = concatenated.getBytes("UTF-8");
            byte[] sign = Base64.decode(jwts[2]);

            X509EncodedKeySpec keySpec = new X509EncodedKeySpec(Base64.decode(pk));
            KeyFactory keyFactory = KeyFactory.getInstance("RSA");
            PublicKey publicKey = keyFactory.generatePublic(keySpec);
            Signature signature = Signature.getInstance(alg);
            signature.initVerify(publicKey);
            signature.update(data);
            return signature.verify(sign);
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (SignatureException e) {
            e.printStackTrace();
        } catch (InvalidKeyException e) {
            e.printStackTrace();
        } catch (InvalidKeySpecException e) {
            e.printStackTrace();
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        return false;
    }

    /**
     * 构造jwt
     * base64url(header).base64url(payload).base64url(sign(base64url(header)+"."+base64url(payload),privateKey))
     * @param header
     * @param payload
     * @param encodedPrivateKey
     */
    private static String buildJwt(String header, String payload, String encodedPrivateKey) throws UnsupportedEncodingException, NoSuchAlgorithmException, SignatureException, InvalidKeySpecException, InvalidKeyException {
        String encodedHeader = Base64.encodeUrlSafe(header.getBytes("UTF-8"));
        String encodePayload = Base64.encodeUrlSafe(payload.getBytes("UTF-8"));
        String concatenated = encodedHeader + '.' + encodePayload;
        byte[] data = concatenated.getBytes("UTF-8");
        /* 获取私钥 */
        KeySpec keySpec = new PKCS8EncodedKeySpec(Base64.decode(encodedPrivateKey));
        KeyFactory keyFactory = KeyFactory.getInstance("RSA");
        PrivateKey privateKey = keyFactory.generatePrivate(keySpec);
        /* 使用私钥生成签名 */
        Signature signature = Signature.getInstance("SHA256withRSA");
        signature.initSign(privateKey);
        signature.update(data);
        byte[] sign = signature.sign();
        StringBuilder jwt = new StringBuilder(encodedHeader);
        jwt.append(".").append(encodePayload).append(".").append(Base64.encodeUrlSafe(sign));
        return jwt.toString();
    }

    public static void main(String[] args) {
        String header = "{\"alg\":\"RS256\",\"typ\":\"JWT\"}";
        String payload = "{\"aud\":[\"TGOA-RESOURCE\"],\"user_name\":\"user\",\"scope\":[\"all\"],\"exp\":1572604523,\"authorities\":[\"ROLE_USER\"],\"jti\":\"bb8474dd-1ca9-470b-97e8-bfedb7fa1a33\",\"client_id\":\"xzsp\"}";
        String privateK = "MIIBNgIBADANBgkqhkiG9w0BAQEFAASCASAwggEcAgEAAoGBAK2LcJearsAfQ0o6SCGywSmV1k/G2vX0AZjDGBhrtvQiUc7p6fbLO2MY+WO1pSDM6rp2VZtUtimChoukJA7jQmYH5vigC/9zetmJg61jMCHgmivSB5mGdTXEvuqCbdIb0AxQJZemXPJREx+WgDpDnuoAqj7hhNYDoZvE5GDh8xOLAgEAAoGAN5EGJASrH2jjKskuf1u07ZPEYxbQ1R+jwz30YR1cHx8+AnpzJ0o7YaeFcp+el7oFDl8FWg7tpKzeV6few8WQZIPHRgJUg2QSbwZdLhVbsmpq9JwIPVWuS6/VDeUiAJymkttnD7pDOd2zfnzLJBiSVQZ0OYFij6ys7bNX0fralYECAQACAQACAQACAQACAQA=";
        try {
            String jwt = buildJwt(header, payload, privateK);
            System.out.println("jwt = " + jwt);
            System.out.println("jwt.equals() = " + jwt.equals("eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsiVEdPQS1SRVNPVVJDRSJdLCJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImV4cCI6MTU3MjYwNDUyMywiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6ImJiODQ3NGRkLTFjYTktNDcwYi05N2U4LWJmZWRiN2ZhMWEzMyIsImNsaWVudF9pZCI6Inh6c3AifQ.IVO3TatyXcO0KZ4D8rpluTvK6OkWaoJbkeB6nJQ-zt2KFYkO7rDzsQ20vDXjHnHSX-3tC4sS0x6KWLMoQY7nFeTetvYFQdX5KFaJuip_ECo27u8iHRAsyK05vOZF0MF-fOHqX3efJTC_iOHW7e1SJPlllgKhwDgzFDOwEAqrcGM"));
        } catch (UnsupportedEncodingException e) {
            e.printStackTrace();
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (SignatureException e) {
            e.printStackTrace();
        } catch (InvalidKeySpecException e) {
            e.printStackTrace();
        } catch (InvalidKeyException e) {
            e.printStackTrace();
        }
        //        try {
//            String encodedHeader = Base64.encodeUrlSafe(header.getBytes("UTF-8"));
//            String encodePayload = Base64.encodeUrlSafe(payload.getBytes("UTF-8"));//结尾没==
//            String concatenated = encodedHeader + '.' + encodePayload;
//            byte[] data = concatenated.getBytes("UTF-8");
//            byte[] sign = Base64.decode("IVO3TatyXcO0KZ4D8rpluTvK6OkWaoJbkeB6nJQ-zt2KFYkO7rDzsQ20vDXjHnHSX-3tC4sS0x6KWLMoQY7nFeTetvYFQdX5KFaJuip_ECo27u8iHRAsyK05vOZF0MF-fOHqX3efJTC_iOHW7e1SJPlllgKhwDgzFDOwEAqrcGM");
//            boolean check = checkJwtSign(data,sign);
//            System.out.println("check = " + check);
//        } catch (UnsupportedEncodingException e) {
//            e.printStackTrace();
//        }

//        boolean check = checkJwtSign(
//                "eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsiVEdPQS1SRVNPVVJDRSJdLCJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImV4cCI6MTU3MjYwNDUyMywiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6ImJiODQ3NGRkLTFjYTktNDcwYi05N2U4LWJmZWRiN2ZhMWEzMyIsImNsaWVudF9pZCI6Inh6c3AifQ.IVO3TatyXcO0KZ4D8rpluTvK6OkWaoJbkeB6nJQ-zt2KFYkO7rDzsQ20vDXjHnHSX-3tC4sS0x6KWLMoQY7nFeTetvYFQdX5KFaJuip_ECo27u8iHRAsyK05vOZF0MF-fOHqX3efJTC_iOHW7e1SJPlllgKhwDgzFDOwEAqrcGM"
//        );
//        System.out.println("check = " + check);

    }
}

```

### 4.昔日笔记（待整理）

```
1.redirect_uri是在server的库中查询的吗？client端好像没传？（并不是，是客户端本身的，虽然不知道在哪配的）
2.security中addFilterBefore，addFilterAfter
理解


1 后端从前端的表单得到用户密码，包装成一个Authentication类的对象；
2 将Authentication对象传给“验证管理器”ProviderManager进行验证；
3 ProviderManager在一条链上依次调用AuthenticationProvider进行验证；
4 验证成功则返回一个封装了权限信息的Authentication对象（即对象的Collection<? extends GrantedAuthority>属性被赋值）；将此对象放入安全上下文SecurityContext中；需要时，可以将Authentication对象从SecurityContextHolder上下文中取出。

# 前面只是生成带有用户信息和权限信息的Authentication，而对指定resource用户是否可访问则是filter负责的
5 这里有两种方式判断用户是否有权限访问指定resource
	①针对方法级的aop
		开启方法级安全注解@EnableGlobalMethodSecurity(prePostEnabled = true)
		对要保护的方法上加入@PreAuthorize("hasRole('XXX')")
	②针对目录级的filter
		SecurityInterceptor简化授权和访问控制决定
		InvocationSecurityMetadataSource的getAttributes(Object object)这个方法获取fi对应的所有权限
        AccessDecisionManager的decide方法来校验用户的权限是否足够

问题 403是在哪配的？
jwt密钥是否唯一？


test:测试role是否跟client有关,删掉role_client中的sso1对应user的权限，观察user是否可单点登录
	emmm...还是可以登录进去，理论上ROLE_USER中已经没有sso1的权限了，为什么还能登进去？？？



update:解决前端无法删除应用的bug

6f063e6d-ae5b-44da-9e0d-6e8053c7a89d


oauth2访问顺序(授权码形式)

1.client 先发送	POST '/oauth/authorize' 若未登录则拦截到server登录页
2.server登录后若未授权则拦截到授权确认页(可以自定义授权页，也可通过autoapprove的值决定是否跳过授权页)	GET '/oauth/confirm_access'
3.确认授权后跳到	GET '/oauth/authorize?client_id=tdfOauth2SSO2v2&redirect_uri=http://localhost:8082/login&response_type=code'返回code
调用client	GET '/login?code=DatMHM&state=UVXQGh'
4. 根据code请求token:	GET|POST /oauth/token token_key check_token
5. 根据token请求用户信息


eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NjY5OTEzMTcsInVzZXJfbmFtZSI6ImFkbWluIiwiYXV0aG9yaXRpZXMiOlsiNWI2NmVjZjQ1ZDYzNDE1OWEwODQ2ODg5OGIxYjMyMTciXSwianRpIjoiY2U3NGRjYjYtN2RkMy00ZDAwLWE5MzMtNDBjNjc5NjU1NzM1IiwiY2xpZW50X2lkIjoidGRmT2F1dGgyU1NPMnYyIiwic2NvcGUiOlsiYWxsIl19.EyXhNftYNUwREZxUMFNqAKRJmKncrx-QuczSZv5PWbK9oGgJpqIjGqd_ZgCd0wDCiFrhaG7uDT4cQHnFopWXXoQql6G_bzbrZl1BosCf6QGcJnxJSueG1nkxBRuSk_6J_0pNJ3wCj4RbxROpsgvw6FIwIS6pyxKAu6gaROASk9KsoOv_6PLsrvaCBsLuIol9OgE0hEcJ0hrLDFAa4aSjE2SyKkfO4buTOxpefTI-Q9M8p3zNCgrBOoD6BDesnpBJvwxSEDOOF0uvI6Ai3GOJ9eSVDTdtiXTnY_nphWhJlGJdrsCA5Uk_dge1XQZk8bb2q5E5vUTB4NPqIAWJZuiHPw


security.oauth2.client.resource.user-info-uri


{"alg":"RS256","typ":"JWT"}.{"exp":1566992737,"user_name":"admin","authorities":["5b66ecf45d634159a08468898b1b3217"],"jti":"aa02ed58-1d5d-4287-8fbe-df52270f2671","client_id":"tdfOauth2SSO2v2","scope":["all"]}

jwt token格式
alg:加密算法
type：token类型

jti：令牌唯一标识
ati：令牌生成时间
exp：令牌过期时间

测试发现直接把登录token中的role_id（即用户所具有的角色）赋给jwt_token的auth...
cc377e1b32e74e71953ddcd595d5498b

在client的LoginSuccessListener中的参数authenticationSuccessEvent 这个参数可以获取jwt_token和user_token(毫无疑问user_token要么是直接来自server要么至少数据是来自server的，因为client根本没有)等信息

sessionId是cookie中的UISESSIONCOUPON

疑问：server是怎么生成token的(V)，client是怎么验证token的package cn.com.taiji.support.authorization;


重写了构造jwt_token的方法去掉了其中的authorities值，然而还是可以登录进去，好像不止传了jwt_token，连server的登录token也拿到了




client发送code请求token /oauth/token /token_key /check_token
根据token请求用户信息
在authentication server中
在resource server中/check_token 或filter

根据用户判断权限完成
这个退出和注销定向有点问题，url中多了一个oauth localhost:8080/oauth/logout


sso1以user单点登录后，在同一个浏览器sso2再以user单点登录后发现server的user已经是登录状态了

刷新后cookie成员相同

零、条件
	应用在同一ip，在同一浏览器中访问

一、现象（下面所有都与是否登录无关）
1.访问server页面后生成cookie：
	JSESSIONID	2F2DF2FC8F7DB8B154FE31FD9E19FDA2
	XSRF-TOKEN	293490cf-2f9e-4a3a-9340-e58a2852eaad

2.访问client1页面后生成cookie
	JSESSIONID	2F2DF2FC8F7DB8B154FE31FD9E19FDA2
	UISESSIONMEMBER	1369884CF4CEB4A37E794E90DA2CC66B
	XSRF-TOKEN	293490cf-2f9e-4a3a-9340-e58a2852eaad

3.刷新server页面后，发现UISESSIONMEMBER已经在cookie中

4.访问client2页面进入登录页生成cookie
	JSESSIONID	2F2DF2FC8F7DB8B154FE31FD9E19FDA2
	UISESSIONCOUPON	167547DEADA63388BB7DD275374B6436
	UISESSIONMEMBER	1369884CF4CEB4A37E794E90DA2CC66B
	XSRF-TOKEN	293490cf-2f9e-4a3a-9340-e58a2852eaad

5.刷新server页面后，发现UISESSIONCOUPON已经在cookie中
	JSESSIONID	2F2DF2FC8F7DB8B154FE31FD9E19FDA2
	UISESSIONCOUPON	167547DEADA63388BB7DD275374B6436
	UISESSIONMEMBER	1369884CF4CEB4A37E794E90DA2CC66B
	XSRF-TOKEN	293490cf-2f9e-4a3a-9340-e58a2852eaad

6.刷新client1或client2发现client1、client2和server的cookie一致，都是4个值

7.其他：
	①在同一浏览器中如果先打开client，再打开server刷新后发现他们的cookie还是一致的

	②将yml的session.cookie.name改成一致后发现两个client单点登录后，刷新另一个client会重新跳到认证页，即新的token使旧的token失效

	③日志中存在new session的情况（规律1可能有问题）

二、规律
1.在同一浏览器中，无论是先访问client还是server，也无论是否登录，初次链接都会生成JSESSIONID和XSRF-TOKEN，
并且后续其他应用的链接会使用初次产生的JSESSIONID和XSRF-TOKEN
（其中client1会多生成UISESSIONMEMBER、client2会多生成ISESSIONCOUPON）
，并且在刷新页面后所有打开的应用的cookie都是一致的

三、原因

浏览器的cookie有一个domain属性，该属性保存着这个cookie是哪个服务器使用的，这样当浏览器去请求不同服务器就可以选择合适的cookie带到后台了；
两个应用的所在同一个服务器即cookie的domain相同，所以会在同一个cookie中保存各自的sessionId信息，由于不同应用的yml文件修改的session的name，所以所有的应用信息都写到一个cookie中，所有应用都使用一个cookie
参考： https://blog.csdn.net/xiongmaodeguju/article/details/80498608

应用和服务器分离后,还是只需登录一次，另一个client就无需登录了，但各自cookie截然不同

client1
UISESSIONCOUPON	5DE0EAAAC97089A4F19AB9950E8894E4
XSRF-TOKEN	9ebbf37d-7133-4a87-94e0-65c63e264caf

client2
UISESSIONMEMBER	099DE6FCCBA01C8913FA232A5D0E2A27
XSRF-TOKEN	6b19c8f5-9ff4-46de-802d-ce79d47e48cd

server
JSESSIONID	oHnrWORl2FW5544pCzTN7GsUbKhooE1BGSdaOEfn
XSRF-TOKEN	f77cd834-09bc-46c4-9b47-aceecfb7e47a

http://localhost:9999/oauth/authorize?client_id=tdfOauth2SSO2v2&redirect_uri=http://localhost:8888/sso/login&response_type=${sso.response_type&state=VyJthf

问题二：server和client在同一个服务器下，并且没有修改cookiename，则server和client会操作同一个cookie

{"alg":"RS256","typ":"JWT"}{"exp":1569300403,"user_name":"admin","authorities":["5b66ecf45d634159a08468898b1b3217"],"jti":"cabdbc61-b183-4d06-90c4-6f686fe18e58","client_id":"tdfOauth2SSO1v2","scope":["all"]}
exp是过期时间，如果时间已经过期，会返回令牌过期的错误
http://localhost:9999/oauth/token?client_id=tdfOauth2SSO213&grant_type=authorization_code&scope=all&redirect_uri=http%3A%2F%2Flocalhost%3A8888%2Foauth2Callback&client_secret=123456&code=v3adnf

#根据code请求token(不需要cookie,可以认为是client端与server端的通信与浏览器无关，所以在开发者工具中观测不到)
#OAuth2不支持JSON访问令牌request.You可以检查这里说明 ..它需要application/x-www-form-urlencoded 参数不能采用json
#security框架中的/oauth/token的header中包含Authorization: Basic dGRmT2F1dGgyU1NPMXYyOjEyMzQ1Ng==就是--user
curl --user tdfOauth2SSO213:123456 \ # 这个参数可以省略，但如果有就必须是<client_id>:<client_secret>
http://localhost:9999/oauth/token -v -X POST \
-H "Content-Type:application/x-www-form-urlencoded" \
-d "client_id=tdfOauth2SSO213&grant_type=authorization_code&scope=all&redirect_uri=http%3A%2F%2Flocalhost%3A8888%2Foauth2Callback&client_secret=123456&code=nRT0lF"


#根据token请求用户信息(不需要cookie)
curl --header|-H "Authorization:bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NzE4ODkwMjAsInVzZXJfbmFtZSI6InVzZXIiLCJhdXRob3JpdGllcyI6WyJST0xFX1VTRVIiXSwianRpIjoiODJiMzgwOTAtYzcwZi00OGU5LWJkYzYtNjcxNjQ3MzdmNTA1IiwiY2xpZW50X2lkIjoidGRmT2F1dGgyU1NPMjEzIiwic2NvcGUiOlsiYWxsIiwicmVhZCJdfQ.R5urcftjVP6J2VBtQvXlFrvYyCfYk4fpx-UcXm_EUO7xg_K03mMGqKJRLPaRcMrXMcYMCtDvkjemXzAIi0atjBOTw3TEqWMDjQmD-rYkBMD6727IfuKz4YUZRxeA7_r0Fm1w9Uzp5oUucF00IiyrY_LHol4lAEvw5gmBWbwTNaJKPpfuOkislxluXTbskXSkDzM0H_DFMRkmeLRs284_fhuPhL8jrlZi9PaMSJLaheOtgWcSHxJcwCGgtMzwPXghFAiYRzVW0WY7en0NPgAVkecDylor29j2YdSm4775HmLtqM66MTv5tD6eqFYyVnPtuuWruBVDc9sz_Bwa3Hg0Rg" -v http://127.0.0.1:8201/me

#一定要设置Content-Type:application/x-www-form-urlencoded，且-d后的参数不能是json形（下面是错误示范）！！！！！
curl -H "Content-Type:application/json" --cookie "JSESSIONID=GPiMjhTkqHIV0THSrDM77eeMKt7qskW3oFcaGWZo; XSRF-TOKEN=e88ff300-a307-49f0-8be0-324beb0c7afb; CLIENTSESSION=19B25CE6E55124044039C2FD762091A7" -X POST -d '{"client_id":"tdfOauth2SSO213","grant_type":"authorization_code","scope":"all","redirect_uri":"http%3A%2F%2Flocalhost%3A8888%2Foauth2Callback","client_secret":"123456","code":"EY3eLk"}' "http://localhost:9999/oauth/token"



curl -H "Authorization:bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NzEzODg3NzEsInVzZXJfbmFtZSI6ImFkbWluIiwiYXV0aG9yaXRpZXMiOlsiNWI2NmVjZjQ1ZDYzNDE1OWEwODQ2ODg5OGIxYjMyMTciXSwianRpIjoiODdlYzlkM2MtODBjZC00N2Y1LWI2ZTMtNzRjNzJkYjMyMDZlIiwiY2xpZW50X2lkIjoidGRmT2F1dGgyU1NPMjEzIiwic2NvcGUiOlsiYWxsIl19.YgycvCuyx-KRiCGmb9fa42pcBHWytWYRT7MuCG7MH9LmmQO4KNbw1s6bKgJI3pKwviVS5UBK8xXNxzX3RuQ_XuaNljqtNgCjPaHDVkcTB2zjB_UJtHuy84dsgMcdPgFQv4pqOIX0o_oxjCVTesELKqQJCv1T9KfxXy_X4C2ZUJsTmzh75T3mBtnGYB4OWD40DQo8R4eNDrBmy0nJ_4v0FqZ1S3LxhKOiOa7CaYlaM2SepW09ZjHZ8MsOUjUzjZUBWoG8zqMUAsar6T2NendMcMDzeTpeLEjCp5lkdPlCkb3OQSaO1HnH5kYa2ZIqyu7q80W7yLtqL5eAWzAkxiBtQw" -v http://localhost:9999/test


http://localhost:8899/oauth/authorize?client_id=clientapp&client_secret=123&redirect_uri=http://127.0.0.1:8888/oauth2Callback&response_type=code


Idea-8306591a=2d23dccd-3912-4c5f-8059-a7177d431803; XSRF-TOKEN=40b0b328-9041-4138-8dd4-b9e686c47d31; CLIENTSESSION=DCCD72E3C98265530FD90D18C6DC77C4; JSESSIONID=XT-J1odKZ5pwlVdkZHyMbbKFFBm9d-RlmilKv_bM


curl -H "Content-Type:application/json" --cookie "Idea-8306591a=2d23dccd-3912-4c5f-8059-a7177d431803; XSRF-TOKEN=40b0b328-9041-4138-8dd4-b9e686c47d31; CLIENTSESSION=DCCD72E3C98265530FD90D18C6DC77C4; JSESSIONID=XT-J1odKZ5pwlVdkZHyMbbKFFBm9d-RlmilKv_bM" -X POST -d '{"client_id":"tdfOauth2SSO213","grant_type":"authorization_code","scope":"all","redirect_uri":"http%3A%2F%2Flocalhost%3A8888%2Foauth2Callback","client_secret":"123456","code":"EY3eLk"}' "http://localhost:9999/oauth/token"



curl -X POST -u "tdfOauth2SSO213:123456" http://127.0.0.1:8899/oauth/token -v  -H "Content-Type:application/x-www-form-urlencoded" -d "client_id=tdfOauth2SSO213&grant_type=authorization_code&scope=all&redirect_uri=http://127.0.0.1:8888/oauth2Callback&client_secret=123456&code=uDVzBQ"


oauth-server
{"alg":"HS256","typ":"JWT"}{"exp":1571669663,"user_name":"user","authorities":["ROLE_USER"],"jti":"8e28e195-81fb-4c2c-bc29-9c533357a8d2","client_id":"tdfOauth2SSO213","scope":["all"]}


/me是spring-security-oauth默认根据token请求用户信息的路径
请求token和请求userinfo是服务器之间的通讯，无需登录，但请求code需要登录


http://localhost:9999/oauth/authorize?client_id=tdfOauth2SSO213&http://127.0.0.1:8888/oauth2Callback&response_type=code&state=VyJthf

/login->
/custom/oauth/confirm_access->
/oauth/authorize


oauth_access_token和oauth_refresh_token是jdbcTokenStore需要的，不是JwtTokenStore需要的

keytool jdk自带工具 可以生成密钥
keytool -genkeypair -alias mytest -keyalg RSA -keypass mypass -keystore mytest.jks -storepass mypass


{"error":"invalid_token","error_description":"Access token expired: eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NzE3MzU0MjAsInVzZXJfbmFtZSI6InVzZXIiLCJhdXRob3JpdGllcyI6WyJST0xFX1VTRVIiXSwianRpIjoiMjE1MTBlYjUtMmEwMC00ODEzLThmNGItNzRkMTQ5ZmUyMzkxIiwiY2xpZW50X2lkIjoidGRmT2F1dGgyU1NPMjEzIiwic2NvcGUiOlsiYWxsIl19.J31t3mo_aubvo0sN-sB5va6Q2avl74llyvqeTHqaLnQf2noPrhNWbuMou8XremnD5sDR364MmTmgJOe-lzOyxkWeEpyKiH5QkHU1q9Dw1WWqWb-_zpXcUcd_2uf-V6liQ2XZdYwNY5RqeccETliEvx743fERWPmSCzqRQtTbm0dTC8Z-ujeOxNTZq_MFguok1uiEeNSj-idZUTaH9ozWds0VDvPYVBRtFpl3zl4BUwAd6u-ItXZrpkukMcj55pW1U3h8_5WT37O2aXTF08AXyriNm7p0F6Zi7KcOzEDNzP7Mep4f6E3UFgIKlRTklyCmSgSzvYjrptIKPZ0KYoCvuA"}


curl -H "Authorization:bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NzE4MzQ3MDgsInVzZXJfbmFtZSI6InVzZXIiLCJhdXRob3JpdGllcyI6WyJST0xFX1VTRVIiXSwianRpIjoiZWU5YzdjNzQtNjgzOS00NTZhLWFlMWUtMmMwMjRlYmJiZjkxIiwiY2xpZW50X2lkIjoidGRmT2F1dGgyU1NPMjEzIiwic2NvcGUiOlsiYWxsIiwicmVhZCJdfQ.JNYenq6EPsYRMdj9vKXO_7ahKgrl3Ry7u6nfeSjpTj9BmHB0C7iBYHTf_AiZ4GIcR3VvC1WL5GV1je0SOhcfbAbBkyh-UBbtIPFHqZnGzqe4W4v_Vfkc3q8UCBAflTIRkJ3qN3ib3tzsiaQC5VDV60FKuXVDD3T_dPrE8uRNKIr7v-8Brmw9D0uRDbgGWDQLOEi1hkcdea57wrVuZXwTbQhlxlxgXXHqAsUsYcdRl7OUWdPT5JpBWlM_EDY1AbkgsQxpXEuCgc5j7Yloed54qpm9H8-lrwCURvlKZmd0ggE7IleUkx7xyZAxSdlvJMzfOOkATlbr1gYi_9-VPoupQg" -v http://localhost:8201/read



{"access_token":"eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NzE3Mzk1NTcsInVzZXJfbmFtZSI6InVzZXIiLCJhdXRob3JpdGllcyI6WyJST0xFX1VTRVIiXSwianRpIjoiMWZkMjQwMzAtZmU5MC00MWM0LWIwNGYtMWM3MjIwOTU4MzY1IiwiY2xpZW50X2lkIjoidGRmT2F1dGgyU1NPMjEzIiwic2NvcGUiOlsiYWxsIl19.aHfWEXePuaVDxE2OIQPGIAw7hAr3cfOa5-1UycQxtXaD-N0DnB7rYyHbOdUfiIpU6NAw_J-khWb2kSmEN0EZSe9Xn-5c9507SyHgaaUVPLVJc2E8K7AL89CGVWXMv3VS3JkYNmoUq-aU_h77GIAoLTiT3SL5ohk4G4tJbguPBZMvXXZHflSiKk_AHH_qYgUvK5Zpcr-hx0iIJZkhNZSweqEutSjCheKXA39vUI3Z9Ho-KV_hFKUJEXTl_LyLYViwWcqQxKnvXuJ5cR9ThH8m-FZ5evY53y1gZzk_57urM_O1XoAx4CKWQJ6G0hpO-pI6Vih-YUAuHsw6X7oZ4u-7mQ",
"token_type":"bearer",
"refresh_token":"eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImF0aSI6IjFmZDI0MDMwLWZlOTAtNDFjNC1iMDRmLTFjNzIyMDk1ODM2NSIsImV4cCI6MTU3MTgyNTgzNywiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6IjBkYTIwMTQwLWNkODUtNDQyOC05NTkyLTY3YmEwYTYyOWFkNCIsImNsaWVudF9pZCI6InRkZk9hdXRoMlNTTzIxMyJ9.LKckawkzIa0b0xkhh7TCd5Tz8nXnmCaQMv_0kJNQjLZuWeoWia8y2jrJjtdnqyLFYyZt1-wBZVnhvBbia7-eA7mHiiiTezRgbKyvPc0cK9AHrBcWpPIzycRHLOtL1EWdb-QMMmbitsSVAVmpgdhdTKN2bQNKrmkP854RQGB72Fi9sBbFOsSWYodqxyQHDtRWoRtblOHGUFixhw29f_1vSw-W24WC5yZU1MpaiUqLylqhebJ8aFIzwn8NVWrBpR55p8MNn2Co05KS0V6fk4r71an7ura6d_gj0rMIT4xeM0V6CZkq_eZpRrb-wFh70CWlt35Mjfkld-_bTYosFqW3bw",
"expires_in":119,
"scope":"all",
"jti":"1fd24030-fe90-41c4-b04f-1c7220958365"}

# 根据refresh token换取token请求
curl -u tdfOauth2SSO213:123456 http://localhost:8201/oauth/token -d grant_type=refresh_token -d refresh_token=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImF0aSI6IjExNjhmOWZmLTYzNGEtNDFiMi04Y2FiLWNhZDU1NjU0NzE3ZSIsImV4cCI6MTU3MTg4MTY0MCwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6ImIzOTFkYjZjLTQ2ODEtNGVkOC1hMDRkLTBkZTVmZWUxZTAyOSIsImNsaWVudF9pZCI6InRkZk9hdXRoMlNTTzIxMyJ9.N8TNTZP3neRyrF0LfYDMa3fVDvzm4dLArzR1Vds8xXYHXlzARnOR5HUp7f5tAQifPjFm4dEdUu8e4fT0rgUZJX1jc29qyQDke5MroxhSwXHxq0OmewfTOr4JsANtoZMOqp_Wvu_u-k61PT4cAYD8HXEsQi3hg4Pl9KZmWamXCUUvb3QBQ_IXaWWOYK1ifhrIUKo1dSMKh0uV1UDjXfof9-gt5NTiQ2r3po25iYfdaRSsmZ4MoB4_j4c9ALslToVLSeD0hcVMojAlI0nfeT7MgbwZ3U8RVqBVjxx_LvsrLwo7hoiTGwyplibUOG3w2SuW63fbUen5F1-pniXFvE-u2g

################
第1次刷新token
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImF0aSI6IjExNjhmOWZmLTYzNGEtNDFiMi04Y2FiLWNhZDU1NjU0NzE3ZSIsImV4cCI6MTU3MTg4MTY0MCwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6ImIzOTFkYjZjLTQ2ODEtNGVkOC1hMDRkLTBkZTVmZWUxZTAyOSIsImNsaWVudF9pZCI6InRkZk9hdXRoMlNTTzIxMyJ9.N8TNTZP3neRyrF0LfYDMa3fVDvzm4dLArzR1Vds8xXYHXlzARnOR5HUp7f5tAQifPjFm4dEdUu8e4fT0rgUZJX1jc29qyQDke5MroxhSwXHxq0OmewfTOr4JsANtoZMOqp_Wvu_u-k61PT4cAYD8HXEsQi3hg4Pl9KZmWamXCUUvb3QBQ_IXaWWOYK1ifhrIUKo1dSMKh0uV1UDjXfof9-gt5NTiQ2r3po25iYfdaRSsmZ4MoB4_j4c9ALslToVLSeD0hcVMojAlI0nfeT7MgbwZ3U8RVqBVjxx_LvsrLwo7hoiTGwyplibUOG3w2SuW63fbUen5F1-pniXFvE-u2g
关键字段：
eyJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImF0aSI6IjExNjhmOWZmLTYzNGEtNDFiMi04Y2FiLWNhZDU1NjU0NzE3ZSIsImV4cCI6MTU3MTg4MTY0MCwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6ImIzOTFkYjZjLTQ2ODEtNGVkOC1hMDRkLTBkZTVmZWUxZTAyOSIsImNsaWVudF9pZCI6InRkZk9hdXRoMlNTTzIxMyJ9

使用refresh token请求token后生成的第二次refresh token
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImF0aSI6IjI5YjYxNmVmLWRlY2QtNDMzNC1iZjQ4LTQyZTg5NzViYzQ4ZSIsImV4cCI6MTU3MTg4MTY0MCwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6ImIzOTFkYjZjLTQ2ODEtNGVkOC1hMDRkLTBkZTVmZWUxZTAyOSIsImNsaWVudF9pZCI6InRkZk9hdXRoMlNTTzIxMyJ9.cnzFozJNFLJ7UfIxX4xK-8EK9srAHNQGqk-2UBz7j0X7beNlyXnwTfolKRiqA7uWdn8rchkxFAf93XLT0cpn86dFKDgfFjHiCNJe2C_6YurlnEFlvuFdBOWRkG876r7qYcHGt9W1HUatcsyVInP86CXpMJa39lrP9GKSIdkBOv9TuOMzB026SnsmHSfah9tOwrl_FW2XzrtmVNG_YYLG2rSshc3AhgTPW1hV2NyH2CtNHNP8u4Ylt6aDZ1SyWuAykNprWZWrjMd6PVZoQp8s9eY8Le66ahezMtU74SR8w7sSIEUEEGjHBmJNclXGRDOk5Sz9eWx7Kam7LLt0WBBrtw
关键字段：
eyJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImF0aSI6IjI5YjYxNmVmLWRlY2QtNDMzNC1iZjQ4LTQyZTg5NzViYzQ4ZSIsImV4cCI6MTU3MTg4MTY0MCwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6ImIzOTFkYjZjLTQ2ODEtNGVkOC1hMDRkLTBkZTVmZWUxZTAyOSIsImNsaWVudF9pZCI6InRkZk9hdXRoMlNTTzIxMyJ9

第1次refresh token关键信息
{"ati":"1168f9ff-634a-41b2-8cab-cad55654717e","exp":1571881640,"jti":"b391db6c-4681-4ed8-a04d-0de5fee1e029"}
第2次refresh token关键信息
{"ati":"29b616ef-decd-4334-bf48-42e8975bc48e","exp":1571881640,"jti":"b391db6c-4681-4ed8-a04d-0de5fee1e029"}
可以发现这两个refresh token只有ati(token创建时间，Unix时间戳格式)是不同的，其余信息均一样
################


当前时间2019/10/23 11:00左右
access_token
eyJleHAiOjE1NzE4NDI4NDQsInVzZXJfbmFtZSI6InVzZXIiLCJhdXRob3JpdGllcyI6WyJST0xFX1VTRVIiXSwianRpIjoiOWE4MGUzYzYtMDc0Ni00OTE4LTkwNGEtZWYxZTQ2MTYxYzA2IiwiY2xpZW50X2lkIjoidGRmT2F1dGgyU1NPMjEzIiwic2NvcGUiOlsiYWxsIl19
refresh_token
eyJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImF0aSI6IjlhODBlM2M2LTA3NDYtNDkxOC05MDRhLWVmMWU0NjE2MWMwNiIsImV4cCI6MTU3NDM5MTY0NCwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6ImRkZjgwMGE2LTM5ZjEtNDhkMC1hMjIxLTkyZjQzN2NmZDUwNiIsImNsaWVudF9pZCI6InRkZk9hdXRoMlNTTzIxMyJ9

security oauth jwt默认access token生存时间12h,refresh token生存时间1个月





"access_token":"eyJleHAiOjE1NzE4MDAyMjUsInVzZXJfbmFtZSI6InVzZXIiLCJhdXRob3JpdGllcyI6WyJST0xFX1VTRVIiXSwianRpIjoiYWU1YWY3NWYtMTQ4Ny00MDBkLTg5NDctNTY0NzA1NDg5MjI0IiwiY2xpZW50X2lkIjoidGRmT2F1dGgyU1NPMjEzIiwic2NvcGUiOlsiYWxsIl19",1571800225
"refresh_token":"eyJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImF0aSI6ImFlNWFmNzVmLTE0ODctNDAwZC04OTQ3LTU2NDcwNTQ4OTIyNCIsImV4cCI6MTU3MTgwMDQwNSwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6Ijg3NDBiYzU2LTFkMjEtNGVkZi04N2JhLTcxODM2NmIxZjExNiIsImNsaWVudF9pZCI6InRkZk9hdXRoMlNTTzIxMyJ9"1571800405

{"error":"invalid_client","error_description":"Bad client credentials"}
{"error":"invalid_grant","error_description":"Invalid authorization code: BXGf8V"}




jvBtqsGCOmnYzwe_-HvgOqlKk6HPiLEzS6uCCcnVkFXrhnkPMZ-uQXTR0u-7ZklF0XC7-AMW8FQDOJS1T7IyJpCyeU4lS8RIf_Z8RX51gPGnQWkRvNw61RfiSuSA45LR5NrFTAAGoXUca_lZnbqnl0td-6hBDVeHYkkpAsSck1NPhlcsn-Pvc2Vleui_Iy1U2mzZCM1Vx6Dy7x9IeP_rTNtDhULDMFbB_JYs-Dg6Zd5Ounb3mP57tBGhLYN7zJkN1AAaBYkElsc4GUsGsUWKqgteQSXZorpf6HdSJsQMZBDd7xG8zDDJ28hGjJSgWBndRGSzQEYU09Xbtzk-8khPuw




{"access_token":"eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NzE4MzM1ODIsInVzZXJfbmFtZSI6InVzZXIiLCJhdXRob3JpdGllcyI6WyJST0xFX1VTRVIiXSwianRpIjoiZTdlYmI2ZDItNmE1OC00YTNjLWFmZmItZmMyN2I4MmM5MjE2IiwiY2xpZW50X2lkIjoidGRmT2F1dGgyU1NPMjEzIiwic2NvcGUiOlsiYWxsIl19.ORVyJMyQ5_BnFriCVulkMQSCZY_ILCKvedxf4gW6ajyyzidtEBeNNwWef1yKWAi9xPforlmf7PZ5wmgza5ZYbcspg-bRSFtpbwykc3mwG_ziucqy5e-66jU9tIGV61HCdOJpxYdQ7JgBl4v6WzQYv7wN1uHwyHCPeDNvCt8mxlISSymjqi4Te9KFZePprOEs-clgrWPUUE1Ztm5J76D-AWdJtjYHoX6P7aUFbrOU8GqH_eE0xSkzqydy1C6IgXy6gyvUgigVAAnYsDMT78n8lxPoWSYqSzUE2zCWorXvYeD5Rh1nZjO9cHtbNmq67yuBH8SiVRTPZ6zpQlUtH7L1AA","token_type":"bearer","refresh_token":"eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImF0aSI6ImU3ZWJiNmQyLTZhNTgtNGEzYy1hZmZiLWZjMjdiODJjOTIxNiIsImV4cCI6MTU3MTg5ODM4MiwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6IjQyZjg5MjRlLTA4M2QtNDJlNS1hNDJmLTE0M2Q5MTVjNmFhYiIsImNsaWVudF9pZCI6InRkZk9hdXRoMlNTTzIxMyJ9.A1v1fYTfbc6hFlrcw6vHiRMaCKMmV_qEOCG_b1ibwzX_M1GPcF3yaDaowgkTdK2K-0luiPazQFtu3GDYLInn7pJWlWC9duJXqqGJEjmT0lBzcv2UPNNrFMVy5PFvaYjkLuJSzoKd1IAs-ofmTFU-Pm66--uSalqr6jJyv9DKRixfy90mzsREhvxQTozKoAxttHXJ2dKWhBlaPSNNOZF5bKBxBZ8pHudv1ShIfnolOA-DZ5ocVucq9cY5fdZGVtTWg842YXmYZ46hQaj8ACS69k4EXknXb3yMO6v_hIQZbtzmrZuS5V0vlM-ImdwFUgyERVzuhppRvbK1kP8DJ0fUDw","expires_in":7199,"scope":"all","jti":"e7ebb6d2-6a58-4a3c-affb-fc27b82c9216"}



curl http://localhost:8201/oauth/authorize?client_id=tdfOauth2SSO213&redirect_uri=http://127.0.0.1:8888/oauth2Callback&response_type=${sso.response_type&state=VyJthf

sDoBc5


curl --header "Authorization:bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJleHAiOjE1NzE4OTExNTYsInVzZXJfbmFtZSI6InVzZXIiLCJhdXRob3JpdGllcyI6WyJST0xFX1VTRVIiXSwianRpIjoiMzk0NGIwOGQtZTQzOC00YzAwLTk1YjktOGFhNDY1OGU2MGQ4IiwiY2xpZW50X2lkIjoidGRmT2F1dGgyU1NPMjEzIiwic2NvcGUiOlsicmVhZCJdfQ.AwcRJ7M--7919w7qMdh0KM79y5ApYd5b2WT3YJ6va77DxDJ6aWZn79sVZv2fJPshprrthDFqhhT_18F9Dk0-hdXkkJyo11cNQzf9Cy62fmuNCjX4fHMKnrTcmzrR896rb_lIuHdInQOqfqbq-HWGN9zX11b54HXBMsDD4bO3aBmjuYNtBtfeCp22ZONlayLg6YsFntmkSw9qm8xYorG3c7rc1Y0XgSODzedFp_MGelI7FDYAnE-ZQrV6YijbWY3KM7bYJ3FO3PAgBg8Z65vCMcdO60EWwrnt7mvzYy00vg5kuTPuUFTbzsq0IU7_7jNWAO8uU82eJqCrLBt18fpMsQ" -v http://127.0.0.1:8201/me

###############
关于oauth scope的观测
1.数据库存储client信息的scope为write,read,前端请求code的url中scope设置read
oauth登录用户方位这个client颁发的acces_token中的scope为"scope":["write","read"]
该access token可以访问/me和/read(需要read权限)和/write(需要write权限)

①可以发现只要client请求的url中scope是server中的client的scope的子集就可成功获取server端client的全部权限，不必全写
②经测试发现请求code的url中的scope的值无所谓（这里用的是all），但请求token的url中scope必须是server存储的client信息中的子集，否则会返回{"error":"invalid_scope","error_description":"Invalid scope: all","scope":"write read"}

2.数据库存储client信息的scope为read,前端请求code的url中scope设置read
access_token中scope只有read，自然可以访问/me和/read，但为什么能访问/write？？？
重启一下，还是能访问/write，推测@PreAuthorize("#oauth2.hasScope('write')")没生效，经测试这个注解确实没生效，需加入@EnableGlobalMethodSecurity(prePostEnabled=true)使其生效
这时/write果然访问失败，返回{"error":"access_denied","error_description":"不允许访问"}

3.数据库存储client信息的scope为all,前端请求code的url中scope设置read,返回{"error":"invalid_scope","error_description":"Invalid scope: read","scope":"all"}，可见all只是一个，并不是所有

4.数据库存储client信息的scope为all,前端请求code的url中scope设置all,自然可以获取token，但根据token访问不了/write返回{"error":"access_denied","error_description":"不允许访问"},可见all和write及read相互独立，并不是包含关系
总结：
	①访问server的请求中的scope必须属于server存储的client的scope(不必全有)，否则会返回Invalid scope；
access_token中会包装client的scope信息，并且该access_token只能访问对应scope可以访问的资源。
	②all并不包含read和write，它们是相互独立的
扩展：
	即使scope不同一样可以请求code，但在请求token时失败，而token中的scope和server存储的client一致，可以确定oauth是根据server存储的client来构造jwt，并且之前会比对请求和server存储的client信息是否一致，不一致会返回错误
问题：
	只要用户登录获取的token一定包含完整的scope，不会根据url来
###############


curl --header "Authorization:bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsiVEdPQS1SRVNPVVJDRSJdLCJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImV4cCI6MTU3MTk3NTUzOCwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6ImYwMzdkMmZlLTg5ZTUtNGRkOS1iZWZiLWQwZmIxMTQzN2JlMiIsImNsaWVudF9pZCI6InRkZk9hdXRoMlNTTzIxMyJ9.HpE8onlzjbMDNZhLrgj65Tz4-a5Qo_-RpLHjv5XIhJkjlIhztR9b08LX4pDoC34yyjgXcFsABmW9r35biuXITjORkT_JPcV2ntb7V7CDJtTqPRWt8Mmbh4XRwunwSyBfcpCHJJnmMKbmHP5UycWoW3UWVUE0VF03LCOqM42zBUqnacbCMHTuaHZ910yEqZXv2GxdeYm-kW7H2uiNDQqQapGhM7jBL_e-qSYdRrX8AJd90gxY5QkE3JYG1ZEMlNv8WE8phI4jQ3O_kjtWAkUHIGhIqmTaP4w_HgHf_DweePnGEO-rKJGHoUQam5sJYafIVTJwzOFMHO_baXmtcE6Nng" -v http://127.0.0.1:8201/api/user



###########
关于resource_id的思考(这是个什么东西？)
在数据库中的client信息加入resource_id（值为"api1"）后发现access_token的负载为
{"aud":["api1"],"user_name":"user","scope":["all"],"exp":1571908636,"authorities":["ROLE_USER"],"jti":"90233aaf-9e6e-4bcf-ba58-be61613e7a6c","client_id":"tdfOauth2SSO213"}
可以发现多了一个"aud":["api1"]
携带该token访问/me接口发现{"error":"access_denied","error_description":"Invalid token does not contain resource id (order)"}其中"order"来自resource server的resources.resourceId("order").stateless(true);
可以发现只有token中有resource server中配置的resource id，才能访问资源，个人感觉相当于较粗粒度的权限认证（scope是细粒度的），持有resource_id的client可以访问对应resource的资源

个人理解。。。
一个resource server有唯一的resource_id，持有该id的合法token可以根据这个id访问该resource的资源，根据token中的scope可以决定有没有权限访问resource中的资源

我总感觉oauth2里用户的概念很弱，client就相当于替代了用户。。。

############
关于client信息的authorities。。。(这又是个啥？？)
好像是client具有的权限


scope是client的权限，因为无论什么用户访问通过oauth登录client都会获得client的全部scope；
如果要对资源接口使用权限验证则可以考虑@PreAuthorize("hasAuthority('ROLE_USER')")
可参考https://stackoverflow.com/questions/35691051/difference-between-scope-and-authority-in-uaa
https://stackoverflow.com/questions/32092749/spring-oauth2-scope-vs-authoritiesroles


security中
@PreAuthorize("hasAuthority('ROLE_XXX')")与@PreAuthorize("hasRole('XXX')")相同



curl --header "Authorization:bearer eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsiVEdPQS1SRVNPVVJDRSJdLCJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImV4cCI6MTU3MTk5MjgyNywiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6IjkzZDdkMDZkLTRiZDgtNGFlMy05ZmMzLTdmZGY2ZjUyNThmMiIsImNsaWVudF9pZCI6InRkZk9hdXRoMlNTTzIxMyJ9.ef_PuVXD4Fem1NZqIwE8FfKq279mvU1Ccbj8NQU1txSTGNib8BS7ezrgawnChgSZIadB2yfz9PY6pGtDTAbs3SNCpz6ncjZvwEgOn5YsUFNi6G_YjwF_5IsiPYvloQcl07rPk_3g4hjQn9OfM2P1gxY3AZO3tiO8JpvQ0viE6wzWOYo_DbE24EanTIyuKMaESwf1mvCbdEpDo4U7XonM1M0PE1_-1VoNh3hUQveD5FYu4iQEXZqBvLI62BUqkfZAQAppiYfKPsVaG_x39_sYpvCEWx29RINxKTp6zxKBAMdA_eMB26o0be8eqUoMHkssv7kAIHzTgI-1_4t9oZFgkA" "http://127.0.0.1:8201/api/permissions?clientId=sso1&timestamp=123&sign=abc"




11-1
access
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9
eyJhdWQiOlsiVEdPQS1SRVNPVVJDRSJdLCJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImV4cCI6MTU3MjYwMTA2NywiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6ImIyZWRlODJiLWJkMjUtNDY2Ny04NzAyLWI4MDExMTI4OGVlZSIsImNsaWVudF9pZCI6Inh6c3AifQ


eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9
eyJhdWQiOlsiVEdPQS1SRVNPVVJDRSJdLCJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImV4cCI6MTU3MjYwMTE4NSwiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6IjVkZjFhZDZhLTY2NzMtNDI0MC1iYTk2LWJlMzIwNzIwYTc5ZCIsImNsaWVudF9pZCI6Inh6c3AifQ

refresh
eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsiVEdPQS1SRVNPVVJDRSJdLCJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImF0aSI6ImIyZWRlODJiLWJkMjUtNDY2Ny04NzAyLWI4MDExMTI4OGVlZSIsImV4cCI6MTU3MjY2NTg2NywiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6IjBiYmRiMDZiLTcyOGUtNGJlOS04YzY0LTA3ZmEwMGY1MjhjNCIsImNsaWVudF9pZCI6Inh6c3AifQ.XrCoyjgNBhOfsPXUWWs0Ed8eRSNxhmSuhahFJM89cMPow0foRb_XmKa3PiamhkhSNcEQvynzGJPinafP-8yYF9P-bQoZJEh4ZKEn-yxeWQ2Jl41vxJhC1Pvk1qOxaAr67D3wXD209077u32jCADFPb950rkBfkq7AnJMKlQs90l-bxMtNT63e8P-fU_2w66JFmAJeZKt2IiVEIC7A8iaIbCitXI8drTIJqvQJDPXExpx-JJFF07zYi1Qz57aX2jz6XfRx3gA12cx1urbZ7KUa2P38vrX4PhzZRaDO_cv6NIc8dUwaL_vlCOPCngGJmCvcdwbdBuOjJzl1Zt1nTqZ8w


{"aud":["TGOA-RESOURCE"],"user_name":"user","scope":["all"],"exp":1572599679,"authorities":["ROLE_USER"],"jti":"ad528be9-c643-4bc3-aca5-44be87a134c8","client_id":"xzsp"}

{"aud":["TGOA-RESOURCE"],"user_name":"user","scope":["all"],"ati":"ad528be9-c643-4bc3-aca5-44be87a134c8","exp":1572664479,"authorities":["ROLE_USER"],"jti":"4a22245d-3270-44e7-8db5-803c47ff8b36","client_id":"xzsp"}


curl -u xzsp:123456 -X POST http://localhost:8201/oauth/token -d grant_type=refresh_token -d refresh_token=eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsiVEdPQS1SRVNPVVJDRSJdLCJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImF0aSI6ImIyZWRlODJiLWJkMjUtNDY2Ny04NzAyLWI4MDExMTI4OGVlZSIsImV4cCI6MTU3MjY2NTg2NywiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6IjBiYmRiMDZiLTcyOGUtNGJlOS04YzY0LTA3ZmEwMGY1MjhjNCIsImNsaWVudF9pZCI6Inh6c3AifQ.XrCoyjgNBhOfsPXUWWs0Ed8eRSNxhmSuhahFJM89cMPow0foRb_XmKa3PiamhkhSNcEQvynzGJPinafP-8yYF9P-bQoZJEh4ZKEn-yxeWQ2Jl41vxJhC1Pvk1qOxaAr67D3wXD209077u32jCADFPb950rkBfkq7AnJMKlQs90l-bxMtNT63e8P-fU_2w66JFmAJeZKt2IiVEIC7A8iaIbCitXI8drTIJqvQJDPXExpx-JJFF07zYi1Qz57aX2jz6XfRx3gA12cx1urbZ7KUa2P38vrX4PhzZRaDO_cv6NIc8dUwaL_vlCOPCngGJmCvcdwbdBuOjJzl1Zt1nTqZ8w

更新密钥

eyJhbGciOiJSUzI1NiIsInR5cCI6IkpXVCJ9.eyJhdWQiOlsiVEdPQS1SRVNPVVJDRSJdLCJ1c2VyX25hbWUiOiJ1c2VyIiwic2NvcGUiOlsiYWxsIl0sImV4cCI6MTU3MjYwNDUyMywiYXV0aG9yaXRpZXMiOlsiUk9MRV9VU0VSIl0sImp0aSI6ImJiODQ3NGRkLTFjYTktNDcwYi05N2U4LWJmZWRiN2ZhMWEzMyIsImNsaWVudF9pZCI6Inh6c3AifQ.IVO3TatyXcO0KZ4D8rpluTvK6OkWaoJbkeB6nJQ-zt2KFYkO7rDzsQ20vDXjHnHSX-3tC4sS0x6KWLMoQY7nFeTetvYFQdX5KFaJuip_ECo27u8iHRAsyK05vOZF0MF-fOHqX3efJTC_iOHW7e1SJPlllgKhwDgzFDOwEAqrcGM

#jwt
base64url(header).base64url(payload).base64url(sign(base64url(header)+"."+base64url(payload),privateKey))

#重点
我之前可能理解错了oauth2的授权码模式，点击GET '/oauth2/authorize?XXX'跳到认证服务器登录页，输入用户名密码后最后携带code跳到回调地址，
注意：不是认证服务器而是浏览器调取请求服务器的回调地址！！！
因为经过测试如果认证服务器是公网，请求服务器是内网，这个流程仍然可以走通，如果是认证服务器调用请求服务器，外网ip是不可能访问内网ip的！！！
抓包进一步分析，认证服务器登录成功后会重新调用“/oauth2/authorize?XXX”会在response header的location中写入回调地址（其实就是重定向）；当浏览器接受到头信息中的 Location: xxxx 后，就会自动跳转到 xxxx 指向的URL地址，这点有点类似用 js 写跳转。但是这个跳转只有浏览器知道，不管体内容里有没有东西，用户都看不到
```


[TOC]

## 1. 问题背景

### ①使用openssl生成rsa2048密钥对

```bash
# 生成私钥
openssl genrsa -out rsa_private_key.pem 2048
# 从私钥中提取公钥
openssl rsa -in rsa_private_key.pem  -pubout -out rsa_public_key.pem
# 将私钥转换成pkcs8格式（因为java只能解析pkcs8格式）
openssl pkcs8 -topk8 -inform PEM -in rsa_private_key.pem -outform pem -nocrypt -out pkcs8.pem
```

### ②密钥存放yml文件

```yaml
key-pair:
  private-key: ***
  public-key: ***
```

### ③java加载密钥，构建密钥对

```java
@Data
@Component
@ConfigurationProperties(prefix = "key-pair")
@PropertySource(value = "classpath:login-key-pair.yml")
public class LoginKeyPairConfig {
    @Value("${private-key}")
    private String privateKey;
    @Value("${public-key}")
    private String publicKey;

    @Bean
    public KeyPair loginKeyPair() {
        byte[] publicKeyBuffer = Base64.getDecoder().decode(publicKey);
        byte[] privateKeyBuffer = Base64.getDecoder().decode(privateKey);
        X509EncodedKeySpec pubKeySpec = new X509EncodedKeySpec(publicKeyBuffer);
        PKCS8EncodedKeySpec priKeySpec = new PKCS8EncodedKeySpec(privateKeyBuffer);
        KeyPair keyPair = null;
        try {
            KeyFactory keyFactory = KeyFactory.getInstance("RSA");
            RSAPublicKey rsaPublicKey = (RSAPublicKey) keyFactory.generatePublic(pubKeySpec);
            RSAPrivateKey rsaPrivateKey = (RSAPrivateKey) keyFactory.generatePrivate(priKeySpec);
            keyPair = new KeyPair(rsaPublicKey, rsaPrivateKey);
        } catch (NoSuchAlgorithmException e) {
            e.printStackTrace();
        } catch (InvalidKeySpecException e) {
            e.printStackTrace();
        }
        return keyPair;
    }
}
```

### ④java加密解密方法（**有问题的版本**）

```java
	private static final String ENCRYPTION_ALGORITHM_NAME = "RSA";
    public String encryptStrByPublicKey(String plaintext) throws Exception {
        RSAPublicKey rsaPublicKey = (RSAPublicKey) loginKeyPair.getPublic();
        //注意这一句的参数
        Cipher cipher = Cipher.getInstance(ENCRYPTION_ALGORITHM_NAME);
        cipher.init(Cipher.ENCRYPT_MODE, rsaPublicKey);
        byte[] ciphertextBuffer = cipher.doFinal(plaintext.getBytes());
        String ciphertext = Base64.getEncoder().encodeToString(ciphertextBuffer);
        return ciphertext;
    }

    public String decryptStrByPrivateKey(String ciphertext) throws Exception {
        RSAPrivateKey privateKey = (RSAPrivateKey) loginKeyPair.getPrivate();
        //注意这一句的参数
        Cipher cipher = Cipher.getInstance(ENCRYPTION_ALGORITHM_NAME);
        cipher.init(Cipher.DECRYPT_MODE, privateKey);
        byte[] cipherTextBuffer = Base64.getDecoder().decode(ciphertext);
        String plaintext = new String(cipher.doFinal(cipherTextBuffer));
        return plaintext;
    }
```

### ⑤前端加密解密代码

```js
import JSEncrypt from 'jsencrypt'
let pubKey,priKey
let str = 'Alice'
let encryptor = new JSEncrypt()
let decrypt = new JSEncrypt()
encryptor.setPublicKey(pubKey)
decrypt.setPrivateKey(priKey)
let ciphertext = encryptor.encrypt(str)
let plaintext = decrypt.decrypt(ciphertext)
```

## 2. 问题描述

前端使用**公钥**加密，后端使用**私钥**解密，但测试发现**前端对同一字符串加密每次的结果都不同**，但**后端对同一字符串加密每次的结果却相同**

## 3. 问题原因

因为JSEncrypt支持的是openssl生成的pkcs1格式私钥，java需要pkcs8格式私钥，这两种方式的填充padding不同，所以需要后端修改填充方式

## 4. 解决方式

```java
//修改填充方式
private static final String ENCRYPTION_ALGORITHM_NAME = "RSA/None/PKCS1Padding";
//必须加new BouncyCastleProvider()
Cipher cipher = Cipher.getInstance(ENCRYPTION_ALGORITHM_NAME, new BouncyCastleProvider());
```

参考：https://blog.csdn.net/guyongqiangx/article/details/74930951
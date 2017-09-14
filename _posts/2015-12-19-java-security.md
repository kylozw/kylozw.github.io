---
layout: post
title: Java 安全算法
subtitle: 'Java, Algorithm, Security'
date: 2015-12-19T00:00:00.000Z
author: kylo
catalog: true
tags:
  - Java
  - Algorithm
  - Security
---

# 摘要

本文章的内容是对[慕课网的 moocer 老师对 Java 安全方面一些列课程的整理](http://www.imooc.com/u/315464/courses?sort=publish)，旨在对 Java 安全算法相关的知识进行梳理文中对部分相似内容进行了概括说明，并进行了一些比对，方便记忆以及日后回顾。

# Java安全组成

Java 安全主要由 **JCA** (Java Cryptography Architecture)、**JCE** (Java Cryptography Extension)、 **JSSE** (Java Secure Socket Extension)、 **JAAS** (Java Authentication and Authentication Service) 组成。

- JCA<br>
  提供基本的加密框架，如消息摘要、数字签名等；
- JCE<br>
  JCA 上的扩展，提供加密算法、消息摘要、密钥管理等，如 DES、 AES、 RSA 等，主要在 jdk 包；
- JSSE<br>
  提供基于 SSL 的安全套接字加密功能，主要用于网络传输；
- JAAS<br>
  提供 Java 系统的身份验证的功能。

JCA 和 JCE 只是提供接口，可以进行第三方扩展，通过配置第三方 Provider实现:

1. 修改 /jdk/jre/lib/security/java.security：

  ```java
  security.provider.11 = com.test.Provider
  ```

2. 另外还可以通过 Java 代码中调用 security 类的 addProvider() 方法：

  ```java
  Security.addProvider(Provider provider);
  ```

## 相关Java包、类

- java.security<br>
  为安全框架提供接口和类，仅能实现消息摘要；
- javax.crypto<br>
  用于安全消息摘要，消息认证（鉴别）码，能实现完整安全框架；
- java.net.ssl<br>
  网络加解密操作，HttpsURLConnection、SSLContext 类等。

## 第三方扩展

第三方扩展相对 jdk 的基础加密实现更加完善的高强度的算法。

- Bouncy Castle（以下简称BC）

- Commons Codec（以下简称CC）<br>
  Apache针对安全的支持，主要 Base64、二进制、十六进制、字符集编码、URL编码/解码。

# 算法实现

## Base64

Base64是一种基于64个可打印字符来表示二进制数据的表示方法,常用于在通常处理文本数据的场合，表示、传输、存储一些二进制数据。包括MIME的email、在XML中存储复杂数据。

- jdk 通过 BASE64Encoder类和 BASE64Decoder 类进行加密和解密：

  ```java
        BASE64Encoder encoder = new BASE64Encoder();
        String encode = encoder.encode(src.getBytes());
        System.out.println("encode : " + encode);

        BASE64Decoder decoder = new BASE64Decoder();
        System.out.println("decode : " + new String(decoder.decodeBuffer(encode)));
  ```

- BC 和 CC 都通过自己包中的 Base64 类的静态方法进行加解密：

  ```java
    //BC
    byte[] encodeBytes = Base64.encodeBase64(src.getBytes());
    System.out.println("encode : " + new String(encodeBytes));

    byte[] decodeBytes = Base64.decodeBase64(encodeBytes);
    System.out.println("decode : " + new String(decodeBytes));    

    //CC
    byte[] encodeBytes = org.bouncycastle.util.encoders.Base64.encode(src.getBytes());
    System.out.println("encode : " + new String(encodeBytes));

    byte[] decodeBytes = org.bouncycastle.util.encoders.Base64.decode(encodeBytes);
    System.out.println("decode : " + new String(decodeBytes));
  ```

## 消息摘要算法

消息摘要算法的主要作用是验证数据完整性、数字签名核心算法。主要有 MD (Message Digest) 消息摘要、 SHA (Secure Hash Algorithm) 安全散列、 MAC（Message Authentication Code） 消息认证码。

### MD (Message Digest)

MD 消息摘要在用户注册登录中的应用流程大致如下：

- 注册

  1. 用户注册
  2. 服务器对密码进行消息摘要
  3. 信息持久化（保存）
  4. 返回注册结果

- 登录

  1. 用户登录
  2. 服务器对密码进行消息摘要
  3. 通过用户名及摘要查询，对比两次摘要
  4. 返回登录结果

主要有 MD2、 MD4、 MD5 三种。

算法  | 摘要长度 | 实现方
--- | ---- | ---
MD2 | 128  | jdk
MD4 | 128  | BC
MD5 | 128  | jdk

- jdk 通过 MessageDigest 类实现：

  ```java
    MessageDigest md = MessageDigest.getInstance("MD5");
    byte[] md5Bytes = md.digest(src.getBytes());
    System.out.println("JDK MD5 : " + Hex.encodeHexString(md5Bytes));
  ```

- BC 通过 Digest 接口实现：

  ```java
    Digest digest = new MD5Digest();
    digest.update(src.getBytes(), 0, src.getBytes().length);
    byte[] md5Bytes = new byte[digest.getDigestSize()];
    digest.doFinal(md5Bytes, 0);
    System.out.println("BC MD5 : " + org.bouncycastle.util.encoders.Hex.toHexString(md5Bytes));
  ```

- CC 通过工具类 DigestUtils 实现：

  ```java
    System.out.println("CC MD5 : " + DigestUtils.md5Hex(src.getBytes()));
  ```

### SHA (Secure Hash Algorithm)

SHA 是固定长度的安全散列算法，与 MD 不同，不同明文的结果差异很大，主要有 SHA-1、 SHA-2 (SHA-224、SHA-256、SHA-384、SHA-512)。

SHA 消息摘要的应用流程大致如下：

1. 发送方公布消息摘要算法
2. 对待发布消息进行摘要处理
3. 发送摘要消息
4. 发送消息
5. 接收方消息鉴别

联合登录采用在原始信息中进行以下信息：

1. 约定Key
2. 增加时间戳
3. 排序

生成规定的字符，如：`http://**?msg=12jlgj32lj&timestamp=1309488734`。然后再对结果进行消息摘要。

算法      | 摘要长度 | 实现方
------- | ---- | ---
SHA-1   | 160  | jdk
SHA-224 | 224  | BC
SHA-256 | 256  | jdk
SHA-384 | 384  | jdk
SHA-512 | 512  | jdk

- jdk 通过 MessageDigest 类实现：

  ```java
    MessageDigest md = MessageDigest.getInstance("SHA");
    md.update(src.getBytes());
    System.out.println("jdk sha-1 : " + Hex.encodeHexString(md.digest()));
  ```

- bc 通过 Digest 接口实现：

  ```java
    Digest digest = new SHA224Digest();
    digest.update(src.getBytes(), 0, src.getBytes().length);
    byte[] sha224Bytes = new byte[digest.getDigestSize()];
    digest.doFinal(sha224Bytes, 0);
    System.out.println("bc sha-224 : " + org.bouncycastle.util.encoders.Hex.toHexString(sha224Bytes));
  ```

- CC 通过工具类 DigestUtils 实现：

  ```java
    System.out.println("cc sha1 - 1 :" + DigestUtils.sha1Hex(src.getBytes()));
    System.out.println("cc sha1 - 2 :" + DigestUtils.sha1Hex(src));
  ```

### MAC（Message Authentication Code）

MAC 兼容了 MD 和 SHA 算法的特点， 也称为HMAC (keyed-Hash Message Authentication Code) 含有密钥的散列函数算法。

MAC算法的消息传递：

1. 发送方公布消息摘要算法
2. 构建密钥
3. 发送密钥给接收方
4. 对待发消息进行摘要处理
5. 发送消息摘要
6. 发送消息
7. 接收方进行消息鉴别

包含 MD 和 SHA 两个系列：

- MD 系列：HmacMD2、HmacMD4、HmacMD5
- SHA 系列：HmacSHA1、HmacSHA224、HmacSHA256、HmacSHA384、HmacSHA512

算法         | 摘要长度 | 实现方
---------- | ---- | ---
HmacMD2    | 128  | BC
HmacMD4    | 128  | BC
HmacMD5    | 128  | jdk
HmacSHA1   | 160  | jdk
HmacSHA224 | 224  | BC
HmacSHA256 | 256  | jdk
HmacSHA384 | 384  | jdk
HmacSHA512 | 512  | jdk

- jdkHmacMD5实现

  有两种方法构建密钥：

  1. 通过 KeyGenerator 类：

    ```java
    KeyGenerator keyGenerator = KeyGenerator.getInstance("HmacMD5");// 初始化KeyGenerator
    SecretKey secretKey = keyGenerator.generateKey();// 产生密钥
    byte[] key = secretKey.getEncoded();
    ```

  2. CC 的 Hex.decodeHex() 方法：

    ```java
    byte[] key = Hex.decodeHex(new char[] {'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a', 'a'});
    ```

    对密钥进行还原，构建Mac实例，执行摘要：

    SecretKey restoreSecretKey = new SecretKeySpec(key, "HmacMD5");// 还原密钥 Mac mac = Mac.getInstance(restoreSecretKey.getAlgorithm());// 实例化MAC mac.init(restoreSecretKey);// 初始化Mac byte[] hmacMD5Bytes = mac.doFinal(src.getBytes());// 执行摘要 System.out.println("jdk hmacMD5 : " + Hex.encodeHexString(hmacMD5Bytes));

- bcHmacMD5 实现，直接使用 HMac 类

  ```java
    HMac hmac = new HMac(new MD5Digest());
    hmac.init(new KeyParameter(org.bouncycastle.util.encoders.Hex.decode("aaaaaaaaaa")));
    hmac.update(src.getBytes(), 0, src.getBytes().length);

    byte[] hmacMD5Bytes = new byte[hmac.getMacSize()];//执行摘要
    hmac.doFinal(hmacMD5Bytes, 0);

    System.out.println("bc hmacMD5 : " + org.bouncycastle.util.encoders.Hex.toHexString(hmacMD5Bytes));
  ```

## 对称加密算法

对称加密算法是一种初等的加密算法，其特点是加密密钥和解密密钥相同，主要有DES、3DES、AES、PBE、IDEA。

除了PBE之外，其他4种对称加密算法的实现主要由以下4步：

1. 生成Key
2. 转换Key
3. 加密
4. 解密

具体的代码如下：

- Key 由 keyGenerator 类生成：

  ```java
    KeyGenerator keyGenerator = KeyGenerator.getInstance("DES");
    keyGenerator.init(56);
    SecretKey secretKey = keyGenerator.generateKey();
    byte[] bytesKey = secretKey.getEncoded();
  ```

- 然后通过 SecretKeyFactory 类进行转换：

  ```java
    DESKeySpec desKeySpec = new DESKeySpec(bytesKey);
    SecretKeyFactory factory = SecretKeyFactory.getInstance("DES");
    Key convertSecretKey = factory.generateSecret(desKeySpec);
  ```

- Cipher 类进行加解密：

  ```java
    //加密
    Cipher cipher = Cipher.getInstance("DES/ECB/PKCS5Padding");
    cipher.init(Cipher.ENCRYPT_MODE, convertSecretKey);
    byte[] result = cipher.doFinal(src.getBytes());
    System.out.println("jdk des encrypt : " + Hex.encodeHexString(result));

    //解密
    cipher.init(Cipher.DECRYPT_MODE, convertSecretKey);
    result = cipher.doFinal(result);
    System.out.println("jdk des decrypt : " + new String(result));
  ```

  注意： IDEA 的 Cipher 实例获取方式略有不同：

  ```java
    Cipher cipher = Cipher.getInstance("IDEA/ECB/ISO10126Padding");
  ```

下面是 PBE 的加密，也是 4 步：

1. 初始化盐
2. 口令与密钥
3. 加密
4. 解密

具体的代码如下：

- 盐由 SecureRandom 类生成：

  ```java
    SecureRandom random = new SecureRandom();
    byte[] salt = random.generateSeed(8);
  ```

- SecretKeyFactory 对象使用 PBEKeySpec 对象作为参数生成密钥

  ```java
    String password = "imooc";
    PBEKeySpec pbeKeySpec = new PBEKeySpec(password.toCharArray());
    SecretKeyFactory factory = SecretKeyFactory.getInstance("PBEWITHMD5andDES");
    Key key = factory.generateSecret(pbeKeySpec);
  ```

- Cipher 对象使用 PBEParameterSpec 对象作为参数初始化，然后执行加解密操作

  ```java
    //加密
    PBEParameterSpec pbeParameterSpec = new PBEParameterSpec(salt, 100);
    Cipher cipher = Cipher.getInstance("PBEWITHMD5andDES");
    cipher.init(Cipher.ENCRYPT_MODE, key, pbeParameterSpec);
    byte[] result = cipher.doFinal(src.getBytes());
    System.out.println("jdk pbe encrypt : " + Base64.encodeBase64String(result));

    //解密
    cipher.init(Cipher.DECRYPT_MODE, key, pbeParameterSpec);
    result = cipher.doFinal(result);
    System.out.println("jdk pbe decrypt : " + new String(result));
  ```

## 非对称加密算法

相对于对称加密算法，非对称加密算法的加密密钥和解密密钥是不同的，这样能够达到双保险的效果。主要的算法有DH、RSA、ElGamal、ECC。

### DH

DH算法是非对称加密算法的起源，是密钥交换算法。该算法通过构建本地密钥解决密钥传递问题，双方密钥是对称的。

DH算法的实现有以下5步：

1. 初始化发送方密钥
2. 初始化接收方密钥
3. 密钥构建
4. 加密
5. 解密

具体的代码实现如下：

- 发送方的 senderKeyPairGenerator 和 senderKeyPair 对象进行初始化发送方密钥 senderPublicKeyEnc：

  ```java
    KeyPairGenerator senderKeyPairGenerator = KeyPairGenerator.getInstance("DH");
    senderKeyPairGenerator.initialize(512);
    KeyPair senderKeyPair = senderKeyPairGenerator.generateKeyPair();
    byte[] senderPublicKeyEnc = senderKeyPair.getPublic().getEncoded();//发送方公钥，发送给接收方（网络、文件。。。）
  ```

- 接收方由接收的公钥 senderPublicKeyEnc 作为参数，receiverKeyFactory 对象生成 receiverPublicKey 对象，receiverPublicKey 对象得到初始化参数 dhParameterSpec 对象，同样通过 receiverKeyPairGenerator 和 receiverKeypair 对象进行初始化接收方密钥 receiverPublicKeyEnc ，同时由 receiverKeypair 对象得到接收方的 receiverPrivateKey 对象：

  ```java
    KeyFactory receiverKeyFactory = KeyFactory.getInstance("DH");
    X509EncodedKeySpec x509EncodedKeySpec = new X509EncodedKeySpec(senderPublicKeyEnc);
    PublicKey receiverPublicKey = receiverKeyFactory.generatePublic(x509EncodedKeySpec);
    DHParameterSpec dhParameterSpec = ((DHPublicKey)receiverPublicKey).getParams();
    KeyPairGenerator receiverKeyPairGenerator = KeyPairGenerator.getInstance("DH");
    receiverKeyPairGenerator.initialize(dhParameterSpec);
    KeyPair receiverKeypair = receiverKeyPairGenerator.generateKeyPair();
    PrivateKey receiverPrivateKey = receiverKeypair.getPrivate();
    byte[] receiverPublicKeyEnc = receiverKeypair.getPublic().getEncoded();
  ```

- 接收方 receiverKeyAgreement 对象， 使用 receiverPrivateKey 和 receiverPublicKey 对象构建密钥 receiverDesKey ；发送方由接收的公钥 receiverPublicKeyEnc 作为参数，senderKeyFactory 对象生成 senderPublicKey 对象，senderKeyAgreement 对象使用 senderKeyPair.getPrivate() 和 senderPublicKey 对象构建密钥 senderDesKey 对象。如果 receiverDesKey 对象和 senderDesKey 对象相等，则进入加密阶段。

  ```java
    KeyAgreement receiverKeyAgreement = KeyAgreement.getInstance("DH");
    receiverKeyAgreement.init(receiverPrivateKey);
    receiverKeyAgreement.doPhase(receiverPublicKey, true);
    SecretKey receiverDesKey = receiverKeyAgreement.generateSecret("DES");

    KeyFactory senderKeyFactory = KeyFactory.getInstance("DH");
    x509EncodedKeySpec = new X509EncodedKeySpec(receiverPublicKeyEnc);
    PublicKey senderPublicKey = senderKeyFactory.generatePublic(x509EncodedKeySpec);
    KeyAgreement senderKeyAgreement = KeyAgreement.getInstance("DH");
    senderKeyAgreement.init(senderKeyPair.getPrivate());
    senderKeyAgreement.doPhase(senderPublicKey, true);
    SecretKey senderDesKey = senderKeyAgreement.generateSecret("DES");
    if (Objects.equals(receiverDesKey, senderDesKey)) {
        System.out.println("双方密钥相同");
    }
  ```

- Cipher 进行加解密操作：

  ```java
    // 加密
    Cipher cipher = Cipher.getInstance("DES");
    cipher.init(Cipher.ENCRYPT_MODE, senderDesKey);
    byte[] result = cipher.doFinal(src.getBytes());
    System.out.println("jdk dh encrypt : " + Base64.encodeBase64String(result));

    // 解密
    cipher.init(Cipher.DECRYPT_MODE, receiverDesKey);
    result = cipher.doFinal(result);
    System.out.println("jdk dh decrypt : " + new String(result));
  ```

### RSA

RSA 是唯一被广泛接受并实现的非对称算法，它用于数据加密和数字签名两个领域，也提供两种模式：

1. 公钥加密、私钥解密
2. 私钥加密、公钥解密

具体代码实现如下：

- 初始化密钥<br>
  KeyPairGenerator 和 KeyPair 初始化公钥和私钥，直接通过类型强制转化得到：

  ```java
    KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
    keyPairGenerator.initialize(512);
    KeyPair keyPair = keyPairGenerator.generateKeyPair();
    RSAPublicKey rsaPublicKey = (RSAPublicKey)keyPair.getPublic();
    RSAPrivateKey rsaPrivateKey = (RSAPrivateKey)keyPair.getPrivate();
    System.out.println("Public Key : " + Base64.encodeBase64String(rsaPublicKey.getEncoded()));
    System.out.println("Private Key : " + Base64.encodeBase64String(rsaPrivateKey.getEncoded()));
  ```

  私钥加密、公钥解密模式：

  ```java
    // 私钥加密、公钥解密——加密
    PKCS8EncodedKeySpec pkcs8EncodedKeySpec = new PKCS8EncodedKeySpec(rsaPrivateKey.getEncoded());
    KeyFactory keyFactory = KeyFactory.getInstance("RSA");
    PrivateKey privateKey = keyFactory.generatePrivate(pkcs8EncodedKeySpec);
    Cipher cipher = Cipher.getInstance("RSA");
    cipher.init(Cipher.ENCRYPT_MODE, privateKey);
    byte[] result = cipher.doFinal(src.getBytes());
    System.out.println("私钥加密、公钥解密——加密 : " + Base64.encodeBase64String(result));

    // 私钥加密、公钥解密——解密
    X509EncodedKeySpec x509EncodedKeySpec = new X509EncodedKeySpec(rsaPublicKey.getEncoded());
    keyFactory = KeyFactory.getInstance("RSA");
    PublicKey publicKey = keyFactory.generatePublic(x509EncodedKeySpec);
    cipher = Cipher.getInstance("RSA");
    cipher.init(Cipher.DECRYPT_MODE, publicKey);
    result = cipher.doFinal(result);
    System.out.println("私钥加密、公钥解密——解密：" + new String(result));
  ```

  公钥加密、私钥解密模式：

  ```java
    // 公钥加密、私钥解密——加密
    x509EncodedKeySpec = new X509EncodedKeySpec(rsaPublicKey.getEncoded());
    keyFactory = KeyFactory.getInstance("RSA");
    publicKey = keyFactory.generatePublic(x509EncodedKeySpec);
    cipher = Cipher.getInstance("RSA");
    cipher.init(Cipher.ENCRYPT_MODE, publicKey);
    result = cipher.doFinal(src.getBytes());
    System.out.println("公钥加密、私钥解密——加密 : " + Base64.encodeBase64String(result));

    // 公钥加密、私钥解密——解密
    pkcs8EncodedKeySpec = new PKCS8EncodedKeySpec(rsaPrivateKey.getEncoded());
    keyFactory = KeyFactory.getInstance("RSA");
    privateKey = keyFactory.generatePrivate(pkcs8EncodedKeySpec);
    cipher = Cipher.getInstance("RSA");
    cipher.init(Cipher.DECRYPT_MODE, privateKey);
    result = cipher.doFinal(result);
    System.out.println("公钥加密、私钥解密——解密：" + new String(result));
  ```

这两种模式实际上都使用 PKCS8EncodedKeySpec 对象生成私钥，使用 X509EncodedKeySpec 对象生成公钥， 只是在使用 Cipher 对象时，初始化采取 Cipher.ENCRYPT_MODE 还是 Cipher.DECRYPT_MODE 的不同。

### ElGamal

ElGamal 由 BC 实现，只提供公钥加密算法，其构建密钥对及加密数据传输过程如下：

1. 接收方构建密钥对
2. 向发送方公布密钥
3. 发送方使用公钥加密
4. 发送方发送加密数据
5. 接收方使用私钥解密数据

具体代码实现如下：

- AlgorithmParameterGenerator 对象生成参数 AlgorithmParameters 对象，由该参数获得DHParameterSpec对象，然后使用该对象初始化 KeyPairGenerator：

  ```java
    AlgorithmParameterGenerator algorithmParameterGenerator = AlgorithmParameterGenerator.getInstance("ElGamal");
    algorithmParameterGenerator.init(256);
    AlgorithmParameters algorithmParameters = algorithmParameterGenerator.generateParameters();
    DHParameterSpec dhParameterSpec = (DHParameterSpec)algorithmParameters.getParameterSpec(DHParameterSpec.class);
    KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("ElGamal");
    keyPairGenerator.initialize(dhParameterSpec, new SecureRandom());
    KeyPair keyPair = keyPairGenerator.generateKeyPair();
    PublicKey elGamalPublicKey = keyPair.getPublic();
    PrivateKey elGamalPrivateKey = keyPair.getPrivate();
    System.out.println("PublicKey: " + Base64.encodeBase64String(elGamalPublicKey));
    System.out.println("PrivateKey: " + Base64.encodeBase64String(elGamalPrivateKey));
  ```

- 接下来和RSA一样

### 数字签名算法

数字签名是带有密钥（公钥、私钥）的消息摘要算法，能验证数据完整性、认证数据来源、抗否认，采用的模式是：私钥签名、公钥验证。

数字签名的主要算法有 RSA、 DSA、 ECDSA，使用这些算法的步骤大同小异，有以下3步：

1. 初始化密钥
2. 执行签名
3. 验证签名

具体代码实现如下：

- 使用 KeyPair 对象构建公钥私钥

  ```java
    KeyPairGenerator keyPairGenerator = KeyPairGenerator.getInstance("RSA");
    keyPairGenerator.initialize(512);
    KeyPair keyPair = keyPairGenerator.generateKeyPair();
    RSAPublicKey rsaPublicKey = (RSAPublicKey)keyPair.getPublic();
    RSAPrivateKey rsaPrivateKey = (RSAPrivateKey)keyPair.getPrivate();
  ```

- 使用 Signature 对象执行签名和验证签名

  ```java
    // 执行签名
    PKCS8EncodedKeySpec pkcs8EncodedKeySpec = new PKCS8EncodedKeySpec(rsaPrivateKey.getEncoded());
    KeyFactory keyFactory = KeyFactory.getInstance("RSA");
    PrivateKey privateKey = keyFactory.generatePrivate(pkcs8EncodedKeySpec);
    Signature signature = Signature.getInstance("MD5withRSA");
    signature.initSign(privateKey);
    signature.update(src.getBytes());
    byte[] result = signature.sign();
    System.out.println("jdk rsa sign : " + Hex.encodeHexString(result));

    // 验证签名
    X509EncodedKeySpec x509EncodedKeySpec = new X509EncodedKeySpec(rsaPublicKey.getEncoded());
    keyFactory = KeyFactory.getInstance("RSA");
    PublicKey publicKey = keyFactory.generatePublic(x509EncodedKeySpec);
    signature = Signature.getInstance("MD5withRSA");
    signature.initVerify(publicKey);
    signature.update(src.getBytes());
    boolean bool = signature.verify(result);
    System.out.println("jdk rsa verify : " + bool);
  ```

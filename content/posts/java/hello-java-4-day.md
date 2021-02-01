---
title: "实现java的rsa加解密与签名并调用RPC微服务(微服务系列第四天)"
date: 2021-01-31T15:40:31+08:00
draft: false
toc: true
categories: 
  - java
images:
series:
  - hello java
  - 微服务
  - tars
tags: 
  - java
  - docker
  - mq
  - jms
  - tars
  - micro serivce
  - 微服务
---

## 前言
> 学习java的第4天 实现目标： 实现java的rsa加解密与签名并调用RPC微服务

## 实现RSA工具类
由于刚开始学习，本过程是边网上抄作业，边理解。
### JDK1.8 的 base64
src/main/java/com/example/rsaserver/utils/JavaBase64Util.java
```java
public class JavaBase64Util {
    public static final String UTF_8 = "UTF-8";
    public static Base64.Encoder encoder;
    public static Base64.Encoder urlEncoder;
    public static Base64.Decoder decoder;
    public static Base64.Decoder urlDecoder;

    static {
        encoder = Base64.getEncoder();
        urlEncoder = Base64.getUrlEncoder();
        decoder = Base64.getDecoder();
        urlDecoder = Base64.getUrlDecoder();
    }

    /**
     * encode
     * @param bytes
     * @return byte[]
     */
    public static byte[] encode(final byte[] bytes) {
        return encoder.encode(bytes);
    }

    /**
     * encode
     * @param string
     * @return
     */
    public static String encode(final String string) {
        final byte[] encode = encode(string.getBytes());
        try {
            return new String(encode, UTF_8);
        } catch (final UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * encode
     * @param bytes
     * @return
     */
    public static String encode2String(final byte[] bytes) {
        return encoder.encodeToString(bytes);
    }

    /**
     * encode
     * @param string
     * @return
     */
    public static byte[] encode2Byte(final String string) {
        return encode(string.getBytes());
    }

    /**
     * urlEncoder
     * @param bytes
     * @return
     */
    public static byte[] urlEncode(final byte[] bytes) {
        return urlEncoder.encode(bytes);
    }

    /**
     * urlEncoder
     * @param string
     * @return
     */
    public static String urlEncode(final String string) {
        final byte[] encode = urlEncode(string.getBytes());
        try {
            return new String(encode, UTF_8);
        } catch (final UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * urlEncoder
     * @param bytes
     * @return
     */
    public static String urlEncode2String(final byte[] bytes) {
        return urlEncoder.encodeToString(bytes);
    }

    /**
     * urlEncoder
     * @param string
     * @return
     */
    public static byte[] urlEncode2Byte(final String string) {
        return urlEncode(string.getBytes());
    }

    /**
     * decode
     * @param bytes
     * @return
     */
    public static byte[] decode(final byte[] bytes) {
        return decoder.decode(bytes);
    }

    /**
     * decode
     * @param string
     * @return
     */
    public static byte[] decode2Byte(final String string) {
        return decoder.decode(string.getBytes());
    }

    /**
     * decode
     * @param bytes
     * @return
     */
    public static String decode2String(final byte[] bytes) {
        try {
            return new String(decoder.decode(bytes), UTF_8);
        } catch (final UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * decode
     * @param string
     * @return
     */
    public static String decode(final String string) {
        final byte[] decode = decode(string.getBytes());
        try {
            return new String(decode, UTF_8);
        } catch (final UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * urlDecode
     * @param bytes
     * @return
     */
    public static byte[] urlDecode(final byte[] bytes) {
        return urlDecoder.decode(bytes);
    }

    /**
     * urlDecode
     * @param string
     * @return
     */
    public static byte[] urlDecode2Byte(final String string) {
        return urlDecode(string.getBytes());
    }

    /**
     * urlDecode
     * @param bytes
     * @return
     */
    public static String urlDecode2String(final byte[] bytes) {
        try {
            return new String(urlDecode(bytes), UTF_8);
        } catch (final UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        return null;
    }

    /**
     * urlDecode
     * @param string
     * @return
     */
    public static String urlDecode(final String string) {
        final byte[] decode = urlDecode(string.getBytes());
        try {
            return new String(decode, UTF_8);
        } catch (final UnsupportedEncodingException e) {
            e.printStackTrace();
        }
        return null;
    }
}
```
### 生成公私钥
src/main/java/com/example/rsaserver/utils/RSAUtils.java
```java
public class RSAUtils {

    /**
     * 加密算法RSA
     */
    public static final String KEY_ALGORITHM = "RSA";

    /**
     * 签名算法
     */
    public static final String SIGNATURE_ALGORITHM = "MD5withRSA";

    /**
     * 获取公钥的key
     */
    private static final String PUBLIC_KEY = "RSAPublicKey";

    /**
     * 获取私钥的key
     */
    private static final String PRIVATE_KEY = "RSAPrivateKey";

    /**
     * RSA最大加密明文大小
     */
    private static final int MAX_ENCRYPT_BLOCK = 117;

    /**
     * RSA最大解密密文大小
     */
    private static final int MAX_DECRYPT_BLOCK = 128;

    /**
     * RSA 位数 如果采用2048 上面最大加密和最大解密则须填写: 245 256
     */
    private static final int INITIALIZE_LENGTH = 1024;

    /**
     * <p>
     * 生成密钥对(公钥和私钥)
     * </p>
     * 
     * @return
     * @throws Exception
     */
    public static Map<String, Object> genKeyPair() throws Exception {
        KeyPairGenerator keyPairGen = KeyPairGenerator.getInstance(KEY_ALGORITHM);
        keyPairGen.initialize(INITIALIZE_LENGTH);
        KeyPair keyPair = keyPairGen.generateKeyPair();
        RSAPublicKey publicKey = (RSAPublicKey) keyPair.getPublic();
        RSAPrivateKey privateKey = (RSAPrivateKey) keyPair.getPrivate();
        Map<String, Object> keyMap = new HashMap<String, Object>(2);
        keyMap.put(PUBLIC_KEY, publicKey);
        keyMap.put(PRIVATE_KEY, privateKey);
        return keyMap;
    }

    /**
     * <p>
     * 获取私钥
     * </p>
     * 
     * @param keyMap 密钥对
     * @return
     * @throws Exception
     */
    public static String getPrivateKey(Map<String, Object> keyMap) throws Exception {
        Key key = (Key) keyMap.get(PRIVATE_KEY);
        return JavaBase64Util.encode2String(key.getEncoded());
    }

    /**
     * <p>
     * 获取公钥
     * </p>
     * 
     * @param keyMap 密钥对
     * @return
     * @throws Exception
     */
    public static String getPublicKey(Map<String, Object> keyMap) throws Exception {
        Key key = (Key) keyMap.get(PUBLIC_KEY);
        return JavaBase64Util.encode2String(key.getEncoded());
    }

}
```
### 测试 公私钥生成
src/test/java/com/example/rsaserver/RsaserverApplicationTests.java
```java
@SpringBootTest
class RsaserverApplicationTests {

	@Test
	void genKeyPair() {
		try {
			final Map<String, Object> keyPair = RSAUtils.genKeyPair();
			assertNotNull(keyPair);

			String publicKey = RSAUtils.getPublicKey(keyPair);
			String privateKey = RSAUtils.getPrivateKey(keyPair);
			assertNotNull(publicKey);
			assertNotNull(privateKey);

			System.out.printf("RSAPublicKey base64 string is %s\n", publicKey);
			System.out.printf("RSAPrivateKey base64 string is %s\n", privateKey);

			System.out.printf("RSAPublicKey byte is %s", keyPair.get("RSAPublicKey"));
			System.out.printf("RSAPublicKey byte is %s", keyPair.get("RSAPrivateKey"));

		} catch (Exception e) {
			fail(e);
		}
	}

}
```
运行测试代码
```sh
$ mvn test
067509788673[INFO] Tests run: 1, Failures: 0, Errors: 0, Skipped: 0, Time elapsed: 1.126 s - in com.example.rsaserver.RsaserverApplicationTests
```
测试通过，观看测试的输出，byte和base64的公私钥都已正常输出
### 基于 byte 的加解密
```java
	/**
	 * <P>
	 * 私钥解密
	 * </p>
	 * 
	 * @param encryptedData
	 *            已加密数据
	 * @param privateKey
	 *            私钥(BASE64编码)
	 * @return
	 * @throws Exception
	 */
	public static byte[] decryptByPrivateKey(byte[] encryptedData, String privateKey) throws Exception {
		byte[] keyBytes = JavaBase64Util.decode2Byte(privateKey);
		PKCS8EncodedKeySpec pkcs8KeySpec = new PKCS8EncodedKeySpec(keyBytes);
		KeyFactory keyFactory = KeyFactory.getInstance(KEY_ALGORITHM);
		Key privateK = keyFactory.generatePrivate(pkcs8KeySpec);
		Cipher cipher = Cipher.getInstance(keyFactory.getAlgorithm());
		cipher.init(Cipher.DECRYPT_MODE, privateK);
		int inputLen = encryptedData.length;
		ByteArrayOutputStream out = new ByteArrayOutputStream();
		int offSet = 0;
		byte[] cache;
		int i = 0;
		// 对数据分段解密
		while (inputLen - offSet > 0) {
			if (inputLen - offSet > MAX_DECRYPT_BLOCK) {
				cache = cipher.doFinal(encryptedData, offSet, MAX_DECRYPT_BLOCK);
			} else {
				cache = cipher.doFinal(encryptedData, offSet, inputLen - offSet);
			}
			out.write(cache, 0, cache.length);
			i++;
			offSet = i * MAX_DECRYPT_BLOCK;
		}
		byte[] decryptedData = out.toByteArray();
		out.close();
		return decryptedData;
	}
 
	/**
	 * <p>
	 * 公钥解密
	 * </p>
	 * 
	 * @param encryptedData
	 *            已加密数据
	 * @param publicKey
	 *            公钥(BASE64编码)
	 * @return
	 * @throws Exception
	 */
	public static byte[] decryptByPublicKey(byte[] encryptedData, String publicKey) throws Exception {
		byte[] keyBytes = JavaBase64Util.decode2Byte(publicKey);
		X509EncodedKeySpec x509KeySpec = new X509EncodedKeySpec(keyBytes);
		KeyFactory keyFactory = KeyFactory.getInstance(KEY_ALGORITHM);
		Key publicK = keyFactory.generatePublic(x509KeySpec);
		Cipher cipher = Cipher.getInstance(keyFactory.getAlgorithm());
		cipher.init(Cipher.DECRYPT_MODE, publicK);
		int inputLen = encryptedData.length;
		ByteArrayOutputStream out = new ByteArrayOutputStream();
		int offSet = 0;
		byte[] cache;
		int i = 0;
		// 对数据分段解密
		while (inputLen - offSet > 0) {
			if (inputLen - offSet > MAX_DECRYPT_BLOCK) {
				cache = cipher.doFinal(encryptedData, offSet, MAX_DECRYPT_BLOCK);
			} else {
				cache = cipher.doFinal(encryptedData, offSet, inputLen - offSet);
			}
			out.write(cache, 0, cache.length);
			i++;
			offSet = i * MAX_DECRYPT_BLOCK;
		}
		byte[] decryptedData = out.toByteArray();
		out.close();
		return decryptedData;
	}
 
	/**
	 * <p>
	 * 公钥加密
	 * </p>
	 * 
	 * @param data
	 *            源数据
	 * @param publicKey
	 *            公钥(BASE64编码)
	 * @return
	 * @throws Exception
	 */
	public static byte[] encryptByPublicKey(byte[] data, String publicKey) throws Exception {
		byte[] keyBytes = JavaBase64Util.decode2Byte(publicKey);
		X509EncodedKeySpec x509KeySpec = new X509EncodedKeySpec(keyBytes);
		KeyFactory keyFactory = KeyFactory.getInstance(KEY_ALGORITHM);
		Key publicK = keyFactory.generatePublic(x509KeySpec);
		// 对数据加密
		Cipher cipher = Cipher.getInstance(keyFactory.getAlgorithm());
		cipher.init(Cipher.ENCRYPT_MODE, publicK);
		int inputLen = data.length;
		ByteArrayOutputStream out = new ByteArrayOutputStream();
		int offSet = 0;
		byte[] cache;
		int i = 0;
		// 对数据分段加密
		while (inputLen - offSet > 0) {
			if (inputLen - offSet > MAX_ENCRYPT_BLOCK) {
				cache = cipher.doFinal(data, offSet, MAX_ENCRYPT_BLOCK);
			} else {
				cache = cipher.doFinal(data, offSet, inputLen - offSet);
			}
			out.write(cache, 0, cache.length);
			i++;
			offSet = i * MAX_ENCRYPT_BLOCK;
		}
		byte[] encryptedData = out.toByteArray();
		out.close();
		return encryptedData;
	}

```
### 基于字符串加解密 上面byte的包装
```java
    /**
     * 公钥加密
     */
    public static String encryptedDataOnJavaByPublicKey(String data, String PUBLICKEY) {
        try {
            data = JavaBase64Util.encode2String(encryptByPublicKey(data.getBytes(), PUBLICKEY));
        } catch (Exception e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
        return data;
    }

    /**
     * 私钥解密
     */
    public static String decryptDataOnJavaByPrivateKey(String data, String PRIVATEKEY) {
        String temp = "";
        try {
            byte[] rs = JavaBase64Util.decode2Byte(data);
            temp = new String(RSAUtils.decryptByPrivateKey(rs, PRIVATEKEY), "UTF-8");

        } catch (Exception e) {
            e.printStackTrace();
        }
        return temp;
    }

    /**
     * 私钥加密
     */
    public static String encryptedDataOnJavaByPrivateKey(String data, String PRIVATEKEY) {
        try {
            data = JavaBase64Util.encode2String(encryptByPrivateKey(data.getBytes(), PRIVATEKEY));
        } catch (Exception e) {
            // TODO Auto-generated catch block
            e.printStackTrace();
        }
        return data;
    }

    /**
     * 公钥解密
     */
    public static String decryptDataOnJavaByPublicKey(String data, String PUBLICKEY) {
        String temp = "";
        try {
            byte[] rs = JavaBase64Util.decode2Byte(data);
            temp = new String(RSAUtils.decryptByPublicKey(rs, PUBLICKEY), "UTF-8");

        } catch (Exception e) {
            e.printStackTrace();
        }
        return temp;
    }
``` 
### 测试 加密与解密
这里的字符串加密是byte加密的一个包装，所以测试字符串加密就都测试了。
```java
	final private String plain = "待加密字符串";

	@Test
	void encodeAndDecode() {
		try {
			final Map<String, Object> keyPair = RSAUtils.genKeyPair();
			String publicKey = RSAUtils.getPublicKey(keyPair);
			String privateKey = RSAUtils.getPrivateKey(keyPair);

			// 公钥加密，私钥解密
			String enData = RSAUtils.encryptedDataOnJavaByPublicKey(plain, publicKey);
			String deData = RSAUtils.decryptDataOnJavaByPrivateKey(enData, privateKey);
			assertEquals(deData, plain);

			// 私钥加密，公钥解密
			enData = RSAUtils.encryptedDataOnJavaByPrivateKey(plain, privateKey);
			deData = RSAUtils.decryptDataOnJavaByPublicKey(enData, publicKey);
			assertEquals(deData, plain);
		} catch (Exception e) {
			fail(e);
		}
	}
```
```sh
$ mvn test
```
测试通过
### 签名与验签
```java
    /**
     * <p>
     * 用私钥对信息生成数字签名，默认签名算法使用 MD5withRSA
     * </p>
     * 
     * @param data       已加密数据
     * @param privateKey 私钥(BASE64编码)
     * 
     * @return
     * @throws Exception
     */
    public static String sign(byte[] data, String privateKey) throws Exception {
        return sign(data, privateKey, SIGNATURE_ALGORITHM);
    }

    /**
     * 用私钥对信息生成数字签名
     * 
     * @param data
     * @param privateKey
     * @param algorithm
     * @return
     * @throws Exception
     */
    public static String sign(byte[] data, String privateKey, String algorithm) throws Exception {
        byte[] keyBytes = JavaBase64Util.decode2Byte(privateKey);
        PKCS8EncodedKeySpec pkcs8KeySpec = new PKCS8EncodedKeySpec(keyBytes);
        KeyFactory keyFactory = KeyFactory.getInstance(KEY_ALGORITHM);
        PrivateKey privateK = keyFactory.generatePrivate(pkcs8KeySpec);
        Signature signature = Signature.getInstance(algorithm);
        signature.initSign(privateK);
        signature.update(data);
        return JavaBase64Util.encode2String(signature.sign());
    }

    /**
     * <p>
     * 校验数字签名，默认使用 MD5withRSA
     * </p>
     * 
     * @param data      已加密数据
     * @param publicKey 公钥(BASE64编码)
     * @param sign      数字签名
     * 
     * @return
     * @throws Exception
     * 
     */
    public static boolean verify(byte[] data, String publicKey, String sign) throws Exception {
        return verify(data, publicKey, sign, SIGNATURE_ALGORITHM);
    }

    /**
     * 校验数字签名
     * 
     * @param data
     * @param publicKey
     * @param sign
     * @param algorithm
     * @return
     * @throws Exception
     */
    public static boolean verify(byte[] data, String publicKey, String sign, String algorithm) throws Exception {
        byte[] keyBytes = JavaBase64Util.decode2Byte(publicKey);
        X509EncodedKeySpec keySpec = new X509EncodedKeySpec(keyBytes);
        KeyFactory keyFactory = KeyFactory.getInstance(KEY_ALGORITHM);
        PublicKey publicK = keyFactory.generatePublic(keySpec);
        Signature signature = Signature.getInstance(algorithm);
        signature.initVerify(publicK);
        signature.update(data);
        return signature.verify(JavaBase64Util.decode2Byte(sign));
    }
```
### 测试 签名与验签
```java
	@Test
	void signAndVerify() {
		try {
			final Map<String, Object> keyPair = RSAUtils.genKeyPair();
			String publicKey = RSAUtils.getPublicKey(keyPair);
			String privateKey = RSAUtils.getPrivateKey(keyPair);

			// 正确的签名测试，签名内容与验证签名的内容相符
			String enSign = RSAUtils.sign(plain.getBytes(), privateKey);
			boolean ok = RSAUtils.verify(plain.getBytes(), publicKey, enSign);
			assertTrue(ok);

			// 错误的签名测试，签名内容与验证签名的内容不符合
			String errPlain = plain + "asdfasf";
			enSign = RSAUtils.sign(errPlain.getBytes(), privateKey);
			ok = RSAUtils.verify(plain.getBytes(), publicKey, enSign);
			assertFalse(ok);

			// 使用其它签名算法
			String algorithm = "SHA256withRSA";
			enSign = RSAUtils.sign(plain.getBytes(), privateKey,  algorithm);
			ok = RSAUtils.verify(plain.getBytes(), publicKey, enSign,  algorithm);
			assertTrue(ok);
		} catch (Exception e) {
			fail(e);
		}

	}
```
运行测试
```sh
$ mvn test
```
看到可以测试通过了。
### bouncycastle 包
网上查找资料，发现这个包针对加解密这一块有相当多的功能，但example很旧，很多API都有变化了，示例学习价值不大了，改天研究。
## 把RSA加入到tars项目中
### tars 接口
今天接口加入了枚举，可以比较有效的控制输出参数
src/main/resources/rsaserver.tars
```nginx
module rsaserver
{
        // 公钥类型，不同功能可能都需要到公钥，如果不想统一使用同一个公钥，可以添加多种类型，供使用。
        enum KeyType
        {
                PASSWORD,
                PLAIN,
                SIGN,
                JWT
        };

        // 签名类型
        enum SignType
        {
                MD5withRSA,
                SHA1withRSA,
                SHA256withRSA
        };

	interface Cipher
	{
                // 获取公钥 用于各应用需要公钥做加解密或签名使用
                // 为了网络安全，私钥是不允许外部获取的，只能调用远程服务间接使用私钥。
                bool getPublicKey(KeyType typ, out string pubKey);

                // 生成一对新的公私钥，客户端获取后用于其它用途
                // 默认生成的是长度1024 RSA的密钥对，有其它需要这个功能可以再加入参数就可以扩展了
                // 为了简单，目前就不加参数了
                bool genKey(out string priKey, out string pubKey);

                // 使用私钥加密
                bool encodeByPrivateKey(string data, out string enStr);

                // 使用公钥加密
                bool encodeByPublicKey(string data, out string enStr);

                // 使用私钥解密
                bool decodeByPrivateKey(string data, out string plain);

                // 使用公钥解密
                bool decodeByPublicKey(string data, out string plain);

                // 生成数字签名
                bool sign(string data, SignType typ, out string signStr);

                // 验证数字签名
                bool verify(string plain, string signStr, SignType typ);
	};
};
```
### pom.xml
#### 添加依赖
```xml
		<dependency>
			<groupId>com.tencent.tars</groupId>
			<artifactId>tars-spring-boot-starter</artifactId>
			<version>1.7.2</version>
		</dependency>
		<dependency>
			<groupId>org.apache.commons</groupId>
			<artifactId>commons-lang3</artifactId>
			<version>3.11</version>
		</dependency>
```
#### 添加插件
```xml
		<!--tars2java plugin -->
		<plugin>
			<groupId>com.tencent.tars</groupId>
			<artifactId>tars-maven-plugin</artifactId>
			<version>1.7.2</version>
			<configuration>
				<tars2JavaConfig>
					<!-- tars file location -->
					<tarsFiles>
						<tarsFile>${basedir}/src/main/resources/rsaserver.tars</tarsFile>
					</tarsFiles>
					<!-- Source file encoding -->
					<tarsFileCharset>UTF-8</tarsFileCharset>
					<!-- Generate server code -->
					<servant>true</servant>
					<!-- Generated source code encoding -->
					<charset>UTF-8</charset>
					<!-- Generated source code directory -->
					<srcPath>${basedir}/src/main/java</srcPath>
					<!-- Generated source code package prefix -->
					<packagePrefixName>com.example.rsaserver.service.</packagePrefixName>
				</tars2JavaConfig>
			</configuration>
		</plugin>
		<!--package plugin-->
```
#### 指定jar名称
```xml
		<finalName>rsaserver</finalName>
```
### 配置文件
这是我为了测试生成的一对公私钥，图方便直接把公私钥都放本地配置文件中进行测试。
src/main/resources/application.properties
```java
# rsa pub pri key config
test.key.public=MIGfMA0GCSqGSIb3DQEBAQUAA4GNADCBiQKBgQCOGU7lIbniqiI4WjoF3FmY9NAMr6VYrD+V0O/TShrSmaObYbYLHZ/NcIscCpV1ibdbhCylK77O+jEtBMAh+jR8lCVMcfNUtHBWkakiMXTMBnI5ddDOmU/J4jdQxhjlqfjvhjBzEMvUx0/16HuqfwEGolr1Zs6tBTjQ0TrsYDwSJQIDAQAB
test.key.private=MIICdgIBADANBgkqhkiG9w0BAQEFAASCAmAwggJcAgEAAoGBAI4ZTuUhueKqIjhaOgXcWZj00AyvpVisP5XQ79NKGtKZo5thtgsdn81wixwKlXWJt1uELKUrvs76MS0EwCH6NHyUJUxx81S0cFaRqSIxdMwGcjl10M6ZT8niN1DGGOWp+O+GMHMQy9THT/Xoe6p/AQaiWvVmzq0FONDROuxgPBIlAgMBAAECgYBic9h82twu1o/1GVaAPwZ4+o23bG8UO+umQmgXrY1eAwMfIhj+JJ1WurY3TIH3ON6ocrB4FBIU17YAqfzwzalU4doFrmwk5WCpLm6xgSlhAkAMj0TDOz0irinVpS8RcYIQLWKvnl391NNIP9l2cataCnDh4lA15yzrTiSVRWrzAQJBAPgQGYvwnWZUErk4f3QvqwFvxEJyPQCUfxvYp3B/aAS9fHVjFZOUUr2W/pcVJnWlPkeTdMRs7AHjJhPt2KFSf+UCQQCSpT/uSEqht6AafooDKHcHHLnRVflIfbFglf4sYHuERp6zT0pe52/7F28sXHBuztkeA7UvHVB2JVG7zsNqv6VBAkEAuxLpMS/0hAdDV4vUErsgK6UuTS3580YJ1eY94Ak1WN3Nznk6/GEPRQtqVGYO6woDPddmZ/v8wC+dt8nXZVHiQQJAJq+Tev/1OE5h3Ttum0CsjeLFHnVoyvfluE45fGmDjDS5HyKWwwyZHQtkl7ZXLtRAsMtXm/NGy7QyqLH2GY4vQQJAJagMRmIV0AlIr1UyZk2vVU9n3LDWO57a3VMxAMQ6Hi6R3yqZb2Stgo62djYCtjoqE25gAUAs2jJ8Bk7uyRbW/A==
```
### main
src/main/java/com/example/rsaserver/RsaserverApplication.java
```java
@SpringBootApplication
@EnableTarsServer
public class RsaserverApplication {

	public static void main(String[] args) {
		// 关闭 spring boot 自带的web服务 目前场景只用到了rpc服务
		SpringApplication app = new SpringApplication(RsaserverApplication.class);
		app.setWebApplicationType(WebApplicationType.NONE);
		app.run(args);

	}

}
```
### 配置类
src/main/java/com/example/tarsmqserver/domain/RSAConfig.java
```java
@Component
public class RSAConfig {
    @Value("${test.key.public}")
    private String publicKey;

    @Value("${test.key.private}")
    private  String privateKey;


    public String getPublicKey() {
        return publicKey;
    }

    public String getPrivateKey(){
        return privateKey;
    }

}
```
### tar2java 生成代码
```sh
$ mvn tars:tars2java
```
### 生成好的接口文件
src/main/java/com/example/rsaserver/service/rsaserver/CipherServant.java
```java
@Servant
public interface CipherServant {

	 boolean getPublicKey(@TarsMethodParameter(name="typ")int typ, @TarsHolder(name="pubKey") Holder<String> pubKey);

	 boolean genKey(@TarsHolder(name="priKey") Holder<String> priKey, @TarsHolder(name="pubKey") Holder<String> pubKey);

	 boolean encodeByPrivateKey(@TarsMethodParameter(name="data")String data, @TarsHolder(name="enStr") Holder<String> enStr);

	 boolean encodeByPublicKey(@TarsMethodParameter(name="data")String data, @TarsHolder(name="enStr") Holder<String> enStr);

	 boolean decodeByPrivateKey(@TarsMethodParameter(name="data")String data, @TarsHolder(name="plain") Holder<String> plain);

	 boolean decodeByPublicKey(@TarsMethodParameter(name="data")String data, @TarsHolder(name="plain") Holder<String> plain);

	 boolean sign(@TarsMethodParameter(name="data")String data, @TarsMethodParameter(name="typ")int typ, @TarsHolder(name="signStr") Holder<String> signStr);

	 boolean verify(@TarsMethodParameter(name="plain")String plain, @TarsMethodParameter(name="signStr")String signStr, @TarsMethodParameter(name="typ")int typ);
}
```
### tars 服务接口实现
src/main/java/com/example/rsaserver/service/rsaserver/impl/CipherServantImpl.java
```java
@TarsServant("cipherObj")
@Component
public class CipherServantImpl implements CipherServant {

    @Autowired
    private RSAConfig rsaConfig;

    private final static Logger RSA_LOGGER = LoggerFactory.getLogger("RSA_SERVER");

    @Override
    public boolean genKey(Holder<String> priKey, Holder<String> pubKey) {
        try {
            final Map<String, Object> keyPair = RSAUtils.genKeyPair();
            String publickey = RSAUtils.getPublicKey(keyPair);
            String privatekey = RSAUtils.getPrivateKey(keyPair);
            priKey.setValue(privatekey);
            pubKey.setValue(publickey);
        } catch (Exception e) {
            return exHandler("genKey", e);
        }

        RSA_LOGGER.info("genkey() successfully call");
        return true;
    }

    @Override
    public boolean getPublicKey(int typ, Holder<String> pubKey) {
        String publicKey = rsaConfig.getPublicKey();
        KeyType convert = KeyType.convert(typ);

        // TODO 这里目前没有往下扩展，后续可以扩展完善
        // 目前所有都返回统一的公钥
        switch (convert) {
            case PASSWORD:
                pubKey.setValue(publicKey);
                break;

            case PLAIN:
                pubKey.setValue(publicKey);
                break;

            case SIGN:
                pubKey.setValue(publicKey);
                break;

            case JWT:
                pubKey.setValue(publicKey);
                break;

            default:
                pubKey.setValue(null);
                break;
        }

        if (StringUtils.isBlank(pubKey.getValue())) {
            return false;
        }
        return true;
    }

    @Override
    public boolean encodeByPrivateKey(String data, Holder<String> enStr) {
        if (StringUtils.isBlank(data)) {
            return false;
        }

        try {
            String privateKey = rsaConfig.getPrivateKey();
            String enData = RSAUtils.encryptedDataOnJavaByPrivateKey(data, privateKey);
            enStr.setValue(enData);
        } catch (Exception e) {
            return exHandler("encodeByPrivateKey", e);
        }

        RSA_LOGGER.info("encodeByPrivateKey() successfully call");
        return true;
    }

    @Override
    public boolean encodeByPublicKey(String data, Holder<String> enStr) {
        if (StringUtils.isBlank(data)) {
            return false;
        }

        try {
            String publicKey = rsaConfig.getPublicKey();
            String enData = RSAUtils.encryptedDataOnJavaByPublicKey(data, publicKey);
            enStr.setValue(enData);
        } catch (Exception e) {
            return exHandler("encodeByPublicKey", e);
        }

        RSA_LOGGER.info("encodeByPublicKey() successfully call");
        return true;
    }

    @Override
    public boolean decodeByPrivateKey(String data, Holder<String> plain) {
        if (StringUtils.isBlank(data)) {
            return false;
        }

        try {
            String privateKey = rsaConfig.getPrivateKey();
            String deData = RSAUtils.decryptDataOnJavaByPrivateKey(data, privateKey);
            plain.setValue(deData);
        } catch (Exception e) {
            return exHandler("decodeByPrivateKey", e);
        }

        RSA_LOGGER.info("decodeByPrivateKey() successfully call");
        return true;
    }

    @Override
    public boolean decodeByPublicKey(String data, Holder<String> plain) {
        if (StringUtils.isBlank(data)) {
            return false;
        }

        try {
            String publicKey = rsaConfig.getPublicKey();
            String deData = RSAUtils.decryptDataOnJavaByPublicKey(data, publicKey);
            plain.setValue(deData);
        } catch (Exception e) {
            return exHandler("decodeByPublicKey", e);
        }

        RSA_LOGGER.info("decodeByPublicKey() successfully call");
        return true;
    }

    @Override
    public boolean sign(String data, int typ, Holder<String> signStr) {
        if (StringUtils.isBlank(data)) {
            return false;
        }

        try {
            // TODO 目前只使用了默认的 MD5withRSA 需要其它的功能,后续可以添加,通过入参typ来调用不同的方法
            String privateKey = rsaConfig.getPrivateKey();
            String enSign = RSAUtils.sign(data.getBytes(), privateKey);
            signStr.setValue(enSign);
        } catch (Exception e) {
            return exHandler("sign", e);
        }

        RSA_LOGGER.info("sign() successfully call");
        return true;
    }

    @Override
    public boolean verify(String plain, String signStr, int typ) {
        if (StringUtils.isBlank(plain) || StringUtils.isBlank(signStr)) {
            return false;
        }

        // TODO 目前只使用了默认的 MD5withRSA 需要其它的功能,后续可以添加,通过入参typ来调用不同的方法
        try {
            String publicKey = rsaConfig.getPublicKey();
            boolean ok = RSAUtils.verify(plain.getBytes(), publicKey, signStr);
            RSA_LOGGER.info("verify() successfully call");
            return ok;
        } catch (Exception e) {
            return exHandler("verify", e);
        }
    }

    private boolean exHandler(String funcname, Exception e) {
        e.printStackTrace();
        RSA_LOGGER.error("{}(): {}", funcname, e.toString());
        return false;

    }

}
```
### 测试代码
src/test/java/com/example/rsaserver/CipherServantImplTest.java
```java
@SpringBootTest
@Component
public class CipherServantImplTest {

    // 因CipherServantImpl中有 @Autowired 所以测试的时候不能使用new，
    // 必须使用 @Autowired来装载类才能读取到配置，要不然会出现空指针对错误
    @Autowired
    private CipherServantImpl imp;

    @Test
    void genKey() {
        Holder<String> priKey = new Holder<String>();
        Holder<String> pubKey = new Holder<String>();
        boolean ok = imp.genKey(priKey, pubKey);
        assertTrue(ok);
    }

    @Test
    public void getPublicKey() {
        KeyType convert = KeyType.convert(0);
        System.out.println(convert.toString());
        Holder<String> pubKey = new Holder<String>();
        boolean ok = imp.getPublicKey(0, pubKey);
        assertTrue(ok);
    }

    @Test
    public void encodeByPrivateKeyWithDecodeByPublicKey() {
        String data = "as65df4656a4sd6fa";
        Holder<String> enStr = new Holder<String>();
        boolean ok = imp.encodeByPrivateKey(data, enStr);
        assertTrue(ok);

        Holder<String> plain = new Holder<String>();
        imp.decodeByPublicKey(enStr.getValue(), plain);
        assertEquals(plain.value, data);

    }

    @Test
    public void encodeByPublicKeyWithDecodeByPrivateKey() {
        String data = "as65df4656a4sd6fa";
        Holder<String> enStr = new Holder<String>();
        boolean ok = imp.encodeByPublicKey(data, enStr);
        assertTrue(ok);

        Holder<String> plain = new Holder<String>();
        imp.decodeByPrivateKey(enStr.getValue(), plain);
        assertEquals(plain.value, data);
    }

    @Test
    public void signWithVerify() {
        String data = "as65df4656a4sd6fa";
        Holder<String> signStr = new Holder<String>();
        boolean ok = imp.sign(data, 0, signStr);
        assertTrue(ok);

        ok = imp.verify(data, signStr.getValue(), 0);
        assertTrue(ok);

    }
}
```
### tars 配置文件
src/main/resources/example.rsaserver.config.conf
```xml
<tars>
	<application>
		enableset=N
		setdivision=NULL
		<client>
			locator=
			sync-invoke-timeout=20000
			async-invoke-timeout=20000
			refresh-endpoint-interval=60000
			stat=tars.tarsstat.StatObj
			property=tars.tarsproperty.PropertyObj
			report-interval=60000
			modulename=rsaserver.cipherJavaServer
		</client>
		<server>
			node=
			app=rsaserver
			server=Cipher
			localip=127.0.0.1
			local=tcp -h 127.0.0.1 -p 18601 -t 3000
			basepath=.
			datapath=.
			logpath=.
			loglevel=DEBUG
			logsize=15M
			log=
			config=tars.tarsconfig.ConfigObj
			notify=tars.tarsnotify.NotifyObj
			mainclass=com.qq.tars.server.startup.Main
			jvmparams=-Xms1024m -Xmx1024m
			sessiontimeout=120000
			sessioncheckinterval=60000
			tcpnodelay=true
			udpbuffersize=8192
			charsetname=UTF-8
			<rsaserver.Cipher.cipherObjAdapter>
				allow
				endpoint=tcp -h 127.0.0.1 -p 18600 -t 60000
				handlegroup=rsaserver.Cipher.cipherObjAdapter
				maxconns=200000
				protocol=tars
				queuecap=10000
				queuetimeout=60000
				servant=rsaserver.Cipher.cipherObj
				shmcap=0
				shmkey=0
				threads=100
			</rsaserver.Cipher.cipherObjAdapter>
		</server>
	</application>
</tars>
```
### 运行测试
```sh
$ mvn test -Dconfig=src/main/resources/example.rsaserver.config.conf
```
或用我提供的Makefile来测试 [传送门](https://github.com/aomirun/demo/blob/main/day-4/rsaserver/Makefile)
```sh
$ make test
```
### 测试结果
```sh
[INFO] Tests run: 8, Failures: 0, Errors: 0, Skipped: 0
[INFO] 
[INFO] ------------------------------------------------------------------------
[INFO] BUILD SUCCESS
[INFO] ------------------------------------------------------------------------
```
### tars-web 上传及发布
```sh
$ make build-upload
```
或者手动上传
### 上传tars接口定义文件
手动上传`src/main/resources/rsaserver.tars`
手动测试
查看结果
一切正常
## 修改day2的mqserver项目
### 修改mqserver.tars
因tars插件多个tars文件,共有是否生成服务接口选项,本例中同是有服务端和客户端,需要分别生成才行
src/main/resources/mqserver.tars
```
module mqserver
{
	interface Message
	{
                // msg 字符串内容原样发送
                bool send(string msg);

                // 用私钥对字符串进行加密,返回加密后的字符串
                bool encode(string sign, out string enStr);

                // 用私钥对字符串进行加密,并发送加密后的内容
                bool encodeWithSend(string msg);

                // 具体字符串拼接可能自己定义,是否在字符串中传输是否加密字符,或者默认约定哪些服务发送密文,哪些服务发送明文
                // if isEncode = false then
                // 对发送的原内容进行签名,之后签名字符串与原内容字符串使用 | 拼接为一个字符串后发送
                // e.g: msg = abc, signStr = AF@FSDafe23 最后发送的字符串 msg = AF@FSDafe23|abc
                // if isEncode = true then
                // 对发送的原内容先进行签名并对原内容进行加密,之后使用 | 拼接为一个字符串后发送
                // e.g: msg = abc signStr = AF@FSDafe23 enStr = ASFDHFHSGGG233f2f2fadfAGAF 最后发送的字符串 msg = AF@FSDafe23|ASFDHFHSGGG233f2f2fadfAGAF
                bool signWithSend(string msg, bool isEncode);
	};
};
```
### 生成服务接口
```sh
$ mvn tars:tars2java
```
复制rsaserver.tars到mqserver项目中
### pom.xml 修改
```xml
			<!--tars2java plugin -->
			<plugin>
				<groupId>com.tencent.tars</groupId>
				<artifactId>tars-maven-plugin</artifactId>
				<version>1.7.2</version>
				<configuration>
					<tars2JavaConfig>
						<!-- tars file location -->
						<tarsFiles>
							<tarsFile>${basedir}/src/main/resources/mqserver.tars</tarsFile>
						</tarsFiles>
						<!-- Source file encoding -->
						<tarsFileCharset>UTF-8</tarsFileCharset>
						<!-- Generate server code -->
						<servant>true</servant>
						<!-- Generated source code encoding -->
						<charset>UTF-8</charset>
						<!-- Generated source code directory -->
						<srcPath>${basedir}/src/main/java</srcPath>
						<!-- Generated source code package prefix -->
						<packagePrefixName>com.example.tarsmqserver.service.</packagePrefixName>
					</tars2JavaConfig>
					<tars2JavaConfig>
						<!-- tars file location -->
						<tarsFiles>
							<tarsFile>${basedir}/src/main/resources/rsaserver.tars</tarsFile>
						</tarsFiles>
						<!-- Source file encoding -->
						<tarsFileCharset>UTF-8</tarsFileCharset>
						<!-- Generate server code -->
						<servant>false</servant>
						<!-- Generated source code encoding -->
						<charset>UTF-8</charset>
						<!-- Generated source code directory -->
						<srcPath>${basedir}/src/main/java</srcPath>
						<!-- Generated source code package prefix -->
						<packagePrefixName>com.example.tarsmqserver.service.</packagePrefixName>
					</tars2JavaConfig>
				</configuration>
			</plugin>
			<!--package plugin-->
```
生成tars客户端代码
```sh
$ mvn tars:tars2java
```
新生成的代码存放在
src/main/java/com/example/tarsmqserver/service/rsaserver
### 修改接口实现
src/main/java/com/example/tarsmqserver/service/mqserver/impl/MessageServantImpl.java
```java
@TarsServant("messageObj")
@Component
public class MessageServantImpl implements MessageServant {
    @Autowired
    private Producer producer;

    @Autowired
    private JmqConfig jmqConfig;

    @TarsClient("example.rsaserver.cipherObj")
    CipherPrx cipherPrx;

    private final static Logger MQSERVER_LOGGER = LoggerFactory.getLogger("MQSERVER");

    /**
     * 发送消息到IBM MQ
     */
    @Override
    public boolean send(String msg) {

        if (StringUtils.isBlank(msg)) {
            return false;
        }

        try {
            String queueName = jmqConfig.getSendQueueName();
            MQSERVER_LOGGER.info("send: {} -> {}", queueName, msg);
            return producer.sendMessage(queueName, msg);
                
        } catch (Exception e) {
            return exHandler("send", e);
        }
    }

    @Override
    public boolean encode(String msg, Holder<String> enStr) {
        if (StringUtils.isBlank(msg)) {
            return false;
        }

        try {
            boolean ok = cipherPrx.encodeByPrivateKey(msg, enStr);
            MQSERVER_LOGGER.info("encode: {} -> {}", msg, enStr.value);
            return ok;
        } catch (Exception e) {
            return exHandler("encode", e);
        }

    }

    @Override
    public boolean encodeWithSend(String msg) {
        if (StringUtils.isBlank(msg)) {
            return false;
        }

        try {
            Holder<String> enStr = new Holder<String>();
            boolean ok = encode(msg, enStr);
            if (!ok) {
                return false;
            }
            return send(enStr.value);

        } catch (Exception e) {
            return exHandler("encodeWithSend", e);
        }
    }

    @Override
    public boolean signWithSend(String msg, boolean isEncode) {
        if (StringUtils.isBlank(msg)) {
            return false;
        }

        try {
            // 原内容去生成签名
            Holder<String> signStr = new Holder<String>();
            boolean ok = cipherPrx.sign(msg, SignType.MD5withRSA.value(), signStr);
            MQSERVER_LOGGER.info("signWithSend: {} -> {}", msg, signStr.value);
            if (!ok) {
                return false;
            }
            String content = signStr.getValue() + "|";

            // 是否需要对发送的内容进行加密
            if (isEncode) {
                Holder<String> enStr = new Holder<String>();

                ok = encode(msg, enStr);
                // 加密失败处理逻辑,根据业务需求不同,可以采用返回失败或不加密,直接发送明文
                if (!ok) {
                    // 这里选择直接发送明文
                    content += msg;
                } else {
                    // 加密成功,发送密文
                    content += enStr.getValue();
                }

            } else {
                // 不加密,发送明文
                content += msg;
            }
            return send(content);
        } catch (Exception e) {
            return exHandler("signWithSend", e);
        }
    }

    /**
     * 错误处理器 类里面统一一下,有空再研究一下统一错误机制
     * @param funcname
     * @param e
     * @return
     */
    private boolean exHandler(String funcname, Exception e) {
        e.printStackTrace();
        MQSERVER_LOGGER.error("{}() fail call. cause: {}", funcname, e.toString());
        return false;

    }

}
```
### 优化消息监听者服务
```java
@Component
public class ConsumerListener {
    private final static Logger JMS_LISTENER = LoggerFactory.getLogger("JMS_LISTENER");

    /**
     * 使用JmsListener配置消费者监听的队列
     * 
     * @param receivedMsg 接收到的消息
     */

    // @JmsListener(destination = "DEV.QUEUE.1")
    @JmsListener(destination = "#{@conf.recvQueueName}")
    public void receiveQueue(String receivedMsg) {
        JMS_LISTENER.info("RECEIVE: {}", receivedMsg);
    }

    @Bean
    public DefaultJmsListenerContainerFactory jmsListenerContainerFactory(ConnectionFactory connectionFactory,
            ExampleErrorHandler errorHandler) {
        DefaultJmsListenerContainerFactory factory = new DefaultJmsListenerContainerFactory();
        factory.setConnectionFactory(connectionFactory);
        factory.setErrorHandler(errorHandler);
        return factory;
    }

    @Service
    public class ExampleErrorHandler implements ErrorHandler {
        @Override
        public void handleError(Throwable t) {
            // handle exception here
            JMS_LISTENER.error(t.toString());
        }
    }

    @Bean
    public JmqConfig conf() {
        return new JmqConfig();
    }
}
```
### 打包/上传/测试/观察
我使用 Makefile 打包上传 [Makefile 代码 传送门](https://github.com/aomirun/demo/blob/main/day-4/rsaserver/Makefile)
```sh
$ make build-upload
```
一切正常,加密和签名都测试成功了

## 源代码
[实现java的rsa加解密与签名并调用RPC微服务(微服务系列第四天)](https://github.com/aomirun/demo/tree/main/day-4)

## 结语
经过几天的折腾,虽然代码没写多少行,但碰到的问题确实很多,还好都基本上解决了.这次使用了一个服务调用另外一个服务的方式,体现了分布式微服务的一些开发与使用.
今天的rsaserver进展扩展,很快就可以弄出一个密钥服务中心,专门针对加解密进行处理,又单独保存了公私钥,扩展一下还可以搞密钥链.
私钥越少人掌握就越安全.

---
title: "AES加密和解密-CryptoJS和Java"

url: "https://wsk1103.github.io/"

tags:
  - 学习笔记
  - 算法
---


**AES**（Advanced Encryption Standard，高级加密标准）是一种对称加密算法，加密和解密使用相同的密钥

CryptoJS[：https://github.com/brix/crypto-js](https://github.com/brix/crypto-js)

# 使用16进制加密解密
## 1. 前端AES加密，后端AES解密
#### 前端
其中和加密相关的js从GitHub下载。
```
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>AES TEST</title>
</head>
<body>
<p>go</p>
<script src="core.js"></script>
<script src="cipher-core.js"></script>
<script src="mode-ecb.js"></script>
<script src="aes.js"></script>

<script>
    var key = CryptoJS.enc.Utf8.parse("1538663015386630");  
    var plaintText = 'wsk-test'; // 明文  
	var encrypted = "a551e4d54cfc396cc93c89dd55f6587c";
    var encryptedData = CryptoJS.AES.encrypt(plaintText, key, {  
        mode: CryptoJS.mode.ECB,  
        padding: CryptoJS.pad.Pkcs7  
    });  

	var a = plaintText + '';//加密前：
	var b = encryptedData + '';//加密后：
    console.log("加密前："+plaintText + '');  
    console.log("加密后："+encryptedData + '');  
 
    alert("加密前："+a);
    alert("加密后："+b);
</script>
</body>
</html>
```

#### 后台解密

```
    /**
     * PKCS5Padding -- Pkcs7 两种padding方法都可以
     *
     * @param content 69f23a1a98ca3d406692742ab7033f3b  16进制
     * @param key     1538663015386630
     * @return masget2019
     */
    public static String decryptAES(String content, String key) {
        try {
            SecretKeySpec skeySpec = new SecretKeySpec(key.getBytes(StandardCharsets.UTF_8), "AES");
            // "算法/模式/补码方式"
            Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
            cipher.init(Cipher.DECRYPT_MODE, skeySpec);
            //解密后是16进制
            return new String(cipher.doFinal(parseHexStr2Byte(content)));
        } catch (Exception e) {
            log.error(String.format("解密失败:，content：%s, key: %s", content, key));
        }
        return content;
    }
    
    /**
     * 将16进制转换为二进制
     *
     * @param hexStr
     * @return
     */
    private static byte[] parseHexStr2Byte(String hexStr) {
        if (hexStr.length() < 1) {
            return null;
        }
        byte[] result = new byte[hexStr.length() / 2];
        for (int i = 0; i < hexStr.length() / 2; i++) {
            int high = Integer.parseInt(hexStr.substring(i * 2, i * 2 + 1), 16);
            int low = Integer.parseInt(hexStr.substring(i * 2 + 1, i * 2 + 2), 16);
            result[i] = (byte) (high * 16 + low);
        }
        return result;
    }
```

## 后台AES加密，前端AES解密

#### 后台加密
```
    /**
     * 加密
     *
     * @param content
     * @param key
     * @return
     */
    public static String encryptAES(String content, String key) {
        try {
            SecretKeySpec skeySpec = new SecretKeySpec(key.getBytes(StandardCharsets.UTF_8), "AES");
            // "算法/模式/补码方式"
            Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
            cipher.init(Cipher.ENCRYPT_MODE, skeySpec);
            byte[] result = cipher.doFinal(content.getBytes(StandardCharsets.UTF_8));
            return parseByte2HexStr(result);
        } catch (Exception e) {
            log.error(String.format("加密失败:，content：%s, key: %s", content, key));
        }
        return content;
    }
    /**
     * 将二进制转换成16进制
     *
     * @param buf
     * @return
     */
    public static String parseByte2HexStr(byte[] buf) {
        StringBuilder sb = new StringBuilder();
        for (byte b : buf) {
            String hex = Integer.toHexString(b & 0xFF);
            if (hex.length() == 1) {
                hex = '0' + hex;
            }
            sb.append(hex.toUpperCase());
        }
        return sb.toString();
    }
```

#### 前端解密

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
 
</head>
 
<script src="core.js"></script>
<script src="enc-base64.js"></script>
<script src="cipher-core.js"></script>
<script src="mode-ecb.js"></script>
<script src="aes.js"></script>
<body>
<script>

    var key = CryptoJS.enc.Utf8.parse("1538663015386630"); //秘钥
 
	var wsk = "a551e4d54cfc396cc93c89dd55f6587c"; //后台返回的加密字符串
    var encryptedHexStr = CryptoJS.enc.Hex.parse(wsk);//a551e4d54cfc396cc93c89dd55f6587c
    var encryptedBase64Str = CryptoJS.enc.Base64.stringify(encryptedHexStr);
 
    var decryptedData = CryptoJS.AES.decrypt(encryptedBase64Str, key, {
        mode: CryptoJS.mode.ECB,
        padding: CryptoJS.pad.Pkcs7
    });
 
    var decryptedStr = decryptedData.toString(CryptoJS.enc.Utf8);
 
    console.log("解密后:"+decryptedStr);
</script>
</body>
</html>
```

注意，前端解密中的 **enc-base64.js** 不能导入到加密的html里面，否则加密后的结果不是 **16进制的形式**


# 使用base64加密解密

```
    /**
     * 加密，生成的16进制的字符串
     *
     * @param content
     * @param key
     * @return
     */
    public static String encryptAES(String content, String key) {
        try {
            SecretKeySpec skeySpec = new SecretKeySpec(key.getBytes(StandardCharsets.UTF_8), "AES");
            // "算法/模式/补码方式"
            Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
            cipher.init(Cipher.ENCRYPT_MODE, skeySpec);
            byte[] result = cipher.doFinal(content.getBytes(StandardCharsets.UTF_8));
            return parseByte2HexStr(result);
        } catch (Exception e) {
            log.error(String.format("加密失败:，content：%s, key: %s", content, key));
        }
        return content;
    }

    /**
     * 加密，生成的base64的字符串
     *
     * @param content
     * @param key
     * @return
     */
    public static String encryptAESWithBase64(String content, String key) {
        try {
            SecretKeySpec skeySpec = new SecretKeySpec(key.getBytes(StandardCharsets.UTF_8), "AES");
            // "算法/模式/补码方式"
            Cipher cipher = Cipher.getInstance("AES/ECB/PKCS5Padding");
            cipher.init(Cipher.ENCRYPT_MODE, skeySpec);
            byte[] result = cipher.doFinal(content.getBytes(StandardCharsets.UTF_8));
            return Base64.encodeBase64String(result);
        } catch (Exception e) {
            log.error(String.format("加密失败:，content：%s, key: %s", content, key));
        }
        return content;
    }
```

## 前端加密

```
<!doctype html>
<html lang="en">
<head>
    <meta charset="utf-8">
    <title>AES TEST</title>
</head>
<script src="http://raw.githack.com/brix/crypto-js/develop/src/core.js"></script>

<script src="http://raw.githack.com/brix/crypto-js/develop/src/enc-base64.js"></script>

<script src="http://raw.githack.com/brix/crypto-js/develop/src/cipher-core.js"></script>
<script src="http://raw.githack.com/brix/crypto-js/develop/src/mode-ecb.js"></script>
<script src="http://raw.githack.com/brix/crypto-js/develop/src/aes.js"></script>

<script
  src="http://code.jquery.com/jquery-2.2.4.min.js"
  integrity="sha256-BbhdlvQf/xTY9gja0Dq3HiwQF8LaCRTXxZKRutelT44="
  crossorigin="anonymous"></script>
  
<body>
  加密前：<div id="one"></div>
 <br/>

 秘钥：<div id="two"></div>
   <br/>
 加密后：<div id="three"></div>

<script>

	var tt = "1538663015386630";
    var key = CryptoJS.enc.Utf8.parse(tt);  //秘钥
    var plaintText = 'test'; // 明文  
    var encryptedData = CryptoJS.AES.encrypt(plaintText, key, {  
        mode: CryptoJS.mode.ECB,  
        padding: CryptoJS.pad.Pkcs7  
    });  

	var a = plaintText + '';//加密前：
	var b = encryptedData + '';//加密后：
	console.log("加密前："+plaintText + '');  
	console.log("加密后："+encryptedData + '');  
 
	//alert("加密前："+a);
	//alert("加密后："+b);
	$(document).ready(function(){
		$("#one").text(a);
	});
	$(document).ready(function(){
		$("#two").text(tt);
	});
	$(document).ready(function(){
		$("#three").text(b);
	});
</script>

</body>
</html>
```
## 前端解密

```
<!DOCTYPE html>
<html lang="en">
<head>
    <meta charset="UTF-8">
    <title>Title</title>
</head>
<script src="http://raw.githack.com/brix/crypto-js/develop/src/core.js"></script>
<script src="http://raw.githack.com/brix/crypto-js/develop/src/enc-base64.js"></script>
<script src="http://raw.githack.com/brix/crypto-js/develop/src/cipher-core.js"></script>
<script src="http://raw.githack.com/brix/crypto-js/develop/src/mode-ecb.js"></script>
<script src="http://raw.githack.com/brix/crypto-js/develop/src/aes.js"></script>
<script
  src="http://code.jquery.com/jquery-2.2.4.min.js"
  integrity="sha256-BbhdlvQf/xTY9gja0Dq3HiwQF8LaCRTXxZKRutelT44="
  crossorigin="anonymous"></script>
<body>
  解密前：<div id="one"></div>
 <br/>

 秘钥：<div id="two"></div>
   <br/>
 解密后：<div id="three"></div>
 
<script>
	var tt = "1538663015386630";
    var key = CryptoJS.enc.Utf8.parse(tt); //秘钥
 
	var wsk = "a551e4d54cfc396cc93c89dd55f6587c"; //后台返回的加密字符串
	//alert("解密前：" + wsk);
    var encryptedHexStr = CryptoJS.enc.Hex.parse(wsk);

    var encryptedBase64Str = CryptoJS.enc.Base64.stringify(encryptedHexStr);
 
    var decryptedData = CryptoJS.AES.decrypt(encryptedBase64Str, key, {
        mode: CryptoJS.mode.ECB,
        padding: CryptoJS.pad.Pkcs7
    });
 
    var decryptedStr = decryptedData.toString(CryptoJS.enc.Utf8);
	//alert("解密后：" + decryptedStr);
    console.log("解密后:"+decryptedStr);
	$(document).ready(function(){
		$("#one").text(wsk);
	});
	$(document).ready(function(){
		$("#two").text(tt);
	});
	$(document).ready(function(){
		$("#three").text(decryptedStr);
	});
 
</script>
</body>
</html>
```







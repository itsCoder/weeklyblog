  
> 建议先了解一下 RSA 原理，再看本文。推荐阮一峰老师的两篇文章：  
>* [RSA 算法原理（一）](http://www.ruanyifeng.com/blog/2013/06/rsa_algorithm_part_one.html)  
>* [RSA 算法原理（二）](http://www.ruanyifeng.com/blog/2013/07/rsa_algorithm_part_two.html)  

因为`php`支持的密钥格式是`.pem`所以`web`和`java`端都要将`pem`格式转换成其语言平台支持的格式。 比如，`java`支持的格式默认是`.asn`。   
### 密钥对的生成( php )
如果系统是`Linux`的话，可以使用`openssl`命令来生成。这里展示的是在`Windows`系统下的操作(因为当时做这个东西的时候是在 Windows 8.1 系统下完成的)，使用 [xampp](https://www.apachefriends.org/zh_cn/index.html) 集成好的`openssl`(其配置文件路径在xampp的安装路径下的php\extras\openssl\openssl.cnf，我们这次使用的就是这个文件)，下面展示用`php`来生成密钥对：(n 和 e 组成公钥，n 和 d 组成私钥)  
```php
<?php
//OPENSSL_CNF为 openssl.cnf 的路径
define('OPENSSL_CNF','<your openssl.cnf path >');
define('SHA',"sha512");
define('LENGTH',2048);

header("Content-type:text/html;charset=utf-8");
$configargs=array(
	'config'=> OPENSSL_CNF,
	"digest_alg" => SHA,
    "private_key_bits" => LENGTH,
    "private_key_type" => OPENSSL_KEYTYPE_RSA,);
$res=openssl_pkey_new($configargs);
openssl_pkey_export($res,$pri,null, $configargs);

/*Binary Data to Decimal Data
 * @param string $data
 * @return string $result
 * */
function hexTobin($hexString) 
    { 
        $hexLenght = strlen($hexString); 
        // only hexadecimal numbers is allowed 
        if ($hexLenght % 2 != 0 || preg_match("/[^\da-fA-F]/",$hexString)) return FALSE; 

        unset($binString); 
        for ($x = 1; $x <= $hexLenght/2; $x++) 
        { 
                $binString .= chr(hexdec(substr($hexString,2 * $x - 2,2))); 
        } 

        return $binString; 
    } 
 
/* Binary Data to Decimal Data
 * @param string $data
 * @return string $result
 **/
function binTodec($data)
	{
		$base = "256";
		$radix = "1";
		$result = "0";

		for($i = strlen($data) - 1; $i >= 0; $i--)
		{
			$digit = ord($data{$i});
			$part_res = bcmul($digit, $radix);
			$result = bcadd($result, $part_res);
			$radix = bcmul($radix, $base);
		}

		return $result;
	}

$details=openssl_pkey_get_details($res);

$n_hex=bin2hex($details['rsa']['n']);
$d_hex=bin2hex($details['rsa']['d']);
$e_hex=bin2hex($details['rsa']['e']);

$n_dec=binTodec($details['rsa']['n']);
$d_dec=binTodec($details['rsa']['d']);
$e_dec=binTodec($details['rsa']['e']);

openssl_pkey_free($res);

$pub=$details['key'];

$crtpath = "./test.crt"; //公钥文件路径  
$pempath = "./test.pem"; //私钥文件路径  

$n_hexpath = "./n_hex.key"; //n_hex文件路径  
$d_hexpath = "./d_hex.key"; //d_hex文件路径  
$e_hexpath = "./e_hex.key"; //e_hex文件路径 

$n_decpath = "./n_dec.key"; //n_dec文件路径  
$d_decpath = "./d_dec.key"; //d_dec文件路径  
$e_decpath = "./e_dec.key"; //e_dec文件路径 

//生成证书文件  
$fp = fopen($crtpath, "w");  
fwrite($fp, $pub);  
fclose($fp);  

$fp = fopen($pempath, "w");  
fwrite($fp, $pri);  
fclose($fp);  
//生成n_hex,d_hex,e_hex文件
$fp = fopen($n_hexpath, "w");  
fwrite($fp, $n_hex);  
fclose($fp);  

$fp = fopen($d_hexpath, "w");  
fwrite($fp, $d_hex);  
fclose($fp);  

$fp = fopen($e_hexpath, "w");  
fwrite($fp, $e_hex);  
fclose($fp); 
//生成n_dec,d_dec,e_dec文件
$fp = fopen($n_decpath, "w");  
fwrite($fp, $n_dec);  
fclose($fp);  

$fp = fopen($d_decpath, "w");  
fwrite($fp, $d_dec);  
fclose($fp);  

$fp = fopen($e_decpath, "w");  
fwrite($fp, $e_dec);  
fclose($fp); 

var_dump($pub);
echo "<br/>";
var_dump($pri);
$pu_key = openssl_pkey_get_public($pub);
print_r($pu_key);
echo "<br/>";
$data = 'plaintext data goes here.';

// Encrypt the data to $encrypted using the public key
openssl_public_encrypt($data, $encrypted,$pub,OPENSSL_PKCS1_PADDING);

// Decrypt the data using the private key and store the results in $decrypted
openssl_private_decrypt($encrypted, $decrypted, $pri,OPENSSL_PKCS1_PADDING);

echo $encrypted." ".strlen($encrypted)."   ".base64_encode($encrypted)."<br/>";
echo $decrypted;
?>
```  
### java
> 此处的核心代码是我从互联网上搜索到的，出处已经记不清是哪了，还请原作者见谅。  

```java
private String _key;
private KeyFormat _format;
private Cipher _decryptProvider;
private Cipher _encryptProvider;

public KeyWorker(String key) {
	this(key, KeyFormat.ASN);
}

public KeyWorker(String key, KeyFormat format) {
	this._key = key;
	this._format = format;
}

public String encrypt(String data) throws IllegalBlockSizeException,
BadPaddingException, InvalidKeyException, NoSuchAlgorithmException,
NoSuchPaddingException, InvalidKeySpecException, IOException, SAXException, ParserConfigurationException {
	this._makesureEncryptProvider();
	byte[] bytes = data.getBytes("UTF-8");
	bytes = this._encryptProvider.doFinal(bytes);
	return new BASE64Encoder().encode(bytes);
}

public String decrypt(String data) throws IOException,
IllegalBlockSizeException, BadPaddingException,
InvalidKeyException, NoSuchAlgorithmException,
NoSuchPaddingException, InvalidKeySpecException, SAXException, ParserConfigurationException {
	this._makesureDecryptProvider();
	
	byte[] bytes = new BASE64Decoder().decodeBuffer(data);
	bytes = this._decryptProvider.doFinal(bytes);
	return new String(bytes, "UTF-8");
}

private void _makesureDecryptProvider() throws NoSuchAlgorithmException,
NoSuchPaddingException, IOException, InvalidKeySpecException,
InvalidKeyException, SAXException, ParserConfigurationException {
if (this._decryptProvider != null)
	return;

Cipher deCipher = Cipher.getInstance("RSA/ECB/PKCS1Padding");
switch (this._format) {

case PEM: {
	this._key = this._key.replace("-----BEGIN PUBLIC KEY-----", "")
			.replace("-----END PUBLIC KEY-----", "")
			.replace("-----BEGIN PRIVATE KEY-----", "")
			.replace("-----END PRIVATE KEY-----", "")
			.replaceAll("\r\n", "");
}
case ASN:
default: {
	Boolean isPrivate = this._key.length() > 500;
	if (isPrivate) {
		PKCS8EncodedKeySpec spec = new PKCS8EncodedKeySpec(
				new BASE64Decoder().decodeBuffer(this._key));

		KeyFactory factory = KeyFactory.getInstance("RSA");
		RSAPrivateKey privateKey = (RSAPrivateKey) factory
				.generatePrivate(spec);
		deCipher.init(Cipher.DECRYPT_MODE, privateKey);
	} else {
		X509EncodedKeySpec spec = new X509EncodedKeySpec(
				new BASE64Decoder().decodeBuffer(this._key));

		KeyFactory factory = KeyFactory.getInstance("RSA");
		RSAPublicKey publicKey = (RSAPublicKey) factory
				.generatePublic(spec);
		deCipher.init(Cipher.DECRYPT_MODE, publicKey);
	}
}
	break;
}

this._decryptProvider = deCipher;
}

private void _makesureEncryptProvider() throws NoSuchAlgorithmException,
NoSuchPaddingException, IOException, InvalidKeySpecException,
InvalidKeyException, SAXException, ParserConfigurationException {
if (this._encryptProvider != null)
	return;

Cipher enCipher = Cipher.getInstance("RSA/ECB/PKCS1Padding");
switch (this._format) {

case PEM: {
	this._key = this._key.replace("-----BEGIN PUBLIC KEY-----", "")
			.replace("-----END PUBLIC KEY-----", "")
			.replace("-----BEGIN PRIVATE KEY-----", "")
			.replace("-----END PRIVATE KEY-----", "")
			.replaceAll("\r\n", "");
}
case ASN:
default: {
	Boolean isPrivate = this._key.length() > 500;
	if (isPrivate) {
		PKCS8EncodedKeySpec spec = new PKCS8EncodedKeySpec(
				new BASE64Decoder().decodeBuffer(this._key));

		KeyFactory factory = KeyFactory.getInstance("RSA");
		RSAPrivateKey privateKey = (RSAPrivateKey) factory
				.generatePrivate(spec);
		enCipher.init(Cipher.ENCRYPT_MODE, privateKey);

	} else {
		X509EncodedKeySpec spec = new X509EncodedKeySpec(
				new BASE64Decoder().decodeBuffer(this._key));

		KeyFactory factory = KeyFactory.getInstance("RSA");
		RSAPublicKey publicKey = (RSAPublicKey) factory
				.generatePublic(spec);
		enCipher.init(Cipher.ENCRYPT_MODE, publicKey);
	}
}
	break;
}
		this._encryptProvider = enCipher;
	}
```  
以上代码核心就是将其它密钥格式转换成`.pem`格式，和`php`后台那边适配。    
### web  
由于是前端，所以只做加密数据。然后必须先将密钥对 ’分解‘，成 ’n‘ , 'd' , 'e'。上代码：  
```html
<!DOCTYPE html>
<html>
<head>
<meta charset="UTF-8">
<title>JavaScript RSA Encryption </title>
</head>
 
<script language="JavaScript" type="text/javascript" src="./js/jsbn.js"></script>
<script language="JavaScript" type="text/javascript" src="./js/prng4.js"></script>
<script language="JavaScript" type="text/javascript" src="./js/rng.js"></script>
<script language="JavaScript" type="text/javascript" src="./js/rsa.js"></script>
<script language="JavaScript" type="text/javascript" src="./js/base64.js"></script>
<script language="JavaScript"> 
//publc key  and public length hex data
var public_key="b3a55f5100b87959ef1bd60508fca4f547af9a0617e8eea1a69a9f7f3f669ee01d89033a7521fa25c6437c6e4c4e0237afc23dbc3f1597b3a0a2181b45aae5effbb787cf6ced26fc042168bad462d916323246ce923c6fa22b6baf62f2a8f93a753b21b3fcd4c5789d89ca02badb88081452a5ecc12d88374475bfa409627e9014600d1b821b76b8b44e6d43bf28eb9fbb68a7b40e5c778d8ff63798764277c040432b9b27a682c8e1c202e95e8c3b826d5188c389716bb4a7278761a7b22ff39ede130c1b9022449f190a79846ea616fec3e1056e8f24a7b3b508ca734ea2c8a92f2c97adc4afd391a52e04504fd69553b4e048650a44ebdaa889701256d315";
var public_length="0010001";
function do_encrypt() {
  var before = new Date();
  var rsa = new RSAKey();
  rsa.setPublic(public_key, public_length);
  var res = rsa.encrypt(document.rsatest.plaintext.value);
  var after = new Date();
  if(res) {
    document.rsatest.ciphertext.value =res;
    document.rsatest.cipherb64.value = hex2b64(res);
    document.rsatest.status.value = "Time: " + (after - before) + "ms";
  }
}

</script>
 
<form name="rsatest" action="rsa.php" method="post">
Plaintext (string):<br>
<input name="plaintext" type="text" value="test" size=40>
<input type="button" value="encrypt" onClick="do_encrypt();"><p>
Ciphertext (hex):(Not used)<br>
<textarea name="ciphertext" rows=4 cols=70></textarea><p>
Ciphertext (base64):<br>
<textarea name="cipherb64" rows=3 cols=70></textarea><p>
Status:<br>
<input name="status" type="text" size=40><p>
<input type="submit" value="submit" />
</form>
  <body>
<html>
```  
  
##### 对应的 php 后台解密：  
```php
<?php 
//header("Content-type:text/javascript;charset=utf-8");
$encrypted=$_POST['cipherb64'];
$public_key = file_get_contents("../public.crt");
$private_key = file_get_contents("../private.pem");

//var_dump(base64_decode($encrypted));

$pu_key = openssl_pkey_get_public($public_key);//这个函数可用来判断公钥是否是可用的
$pi_key = openssl_pkey_get_private($private_key);//这个函数可用来判断私钥是否是可用的，可用返回资源id Resource id
//var_dump($pu_key);
//var_dump($pi_key);
openssl_private_decrypt(base64_decode($encrypted),$decrypted,$pi_key);//私钥解密
echo "\n";
var_dump($decrypted);
echo "\n"."共".strlen($decrypted)."个字节。";
?>
 ```
### 源代码分享
以上全部代码在我的 Github 托管。[传送门](https://github.com/ZetaoYang/RSA)




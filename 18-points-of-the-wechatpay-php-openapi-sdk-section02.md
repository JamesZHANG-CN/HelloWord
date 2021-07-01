---
title: "微信支付PHP开发对接18讲——02: RSA-OAEP非对称加解密重构"
date: 2021-06-29T14:50:11+08:00
lastmod: 2021-07-01T13:18:59+08:00
keywords: ["微信支付", "WeChatPay PHP SDK", "GuzzleHttp", "PHPStan Level8"]
description: "有开发者反馈，先前的加解密`Util\\SensitiveInfoCrypto`实现，用法看似简单，其实用起来'坑'蛮多的。'坑'点在于：初始化所需的`私钥`和`公钥(证书)`，在业务场景下是`非配对`的！`公钥(证书)`加密时，所用的`公钥(证书)`是`平台证书(公钥)`，而解密时所需的`私钥`，是`商户私钥`。并且，加解密稍不注意就会干扰到业务处理(初始化参数以及切换`stage`稍微繁琐)。遂重构一遍，命名为 `Crypto\\Rsa` 类。"
tags: []
toc: false
categories: [php, wechat]
summary: "有开发者反馈，先前的加解密`Util\\SensitiveInfoCrypto`实现，用法看似简单，其实用起来'坑'蛮多的。'坑'点在于：初始化所需的`私钥`和`公钥(证书)`，在业务场景下是`非配对`的！`公钥(证书)`加密时，所用的`公钥(证书)`是`平台证书(公钥)`，而解密时所需的`私钥`，是`商户私钥`。并且，加解密稍不注意就会干扰到业务处理(初始化参数以及切换`stage`稍微繁琐)。遂重构一遍，命名为 `Crypto\\Rsa` 类。"
author: "James"
---

这个 `Crypto\Rsa` 类，是对之前的一个实现 `Util\SensitiveInfoCrypto` 重构。上一版实现是这么用的：

```php
<?php
// Encrypt usage:
$encryptor = new SensitiveInfoCrypto(
    PemUtil::loadCertificate('/downloaded/pubcert.pem')
);
$json = json_encode(['name' => $encryptor('Alice')]);
// That's simple!

// Decrypt usage:
$decryptor = new SensitiveInfoCrypto(
    null,
    PemUtil::loadPrivateKey('/merchant/key.pem')
);
$decrypted = $decryptor->setStage('decrypt')(
    'base64 encoding message was given by the payment plat'
);
// That's simple too!

// Working both Encrypt and Decrypt usages:
$crypto = new SensitiveInfoCrypto(
    PemUtil::loadCertificate('/merchant/cert.pem'),
    PemUtil::loadPrivateKey('/merchant/key.pem')
);
$encrypted = $crypto('Carol');
$decrypted = $crypto->setStage('decrypt')($encrypted);
// Having fun with this!
```

有开发者反馈，上述用法看似简单，其实用起来"坑"蛮多的。稍微分析一下，确实是的。"坑"点在于：初始化所需的`私钥`和`公钥(证书)`，在业务场景下是`非配对`的！`公钥(证书)`加密时，所用的`公钥(证书)`是`平台证书(公钥)`，而解密时所需的`私钥`，是`商户私钥`。并且，加解密稍不注意就会干扰到业务处理(初始化参数以及切换`stage`稍微繁琐)。

是的，这个`SensitiveInfoCrypto`类过度设计了。

所以，在新包内，这个是必须要被重写一遍实现的。

## Crypto\Rsa::preCondition 前置条件检测

检测当前`ext-openssl`扩展，是否支持`SHA256`哈希散列，为了更清晰地区别传统`Hash`散列算法，这里用到了算法别名即`sha256WithRSAEncryption`。代码块如下：

```php
<?php
const sha256WithRSAEncryption = 'sha256WithRSAEncryption';
private static function preCondition(): void
{
    if (!in_array(sha256WithRSAEncryption, openssl_get_md_methods(true))) {
        throw new RuntimeException('It looks like the ext-openssl extension missing the `sha256WithRSAEncryption` digest method.');
    }
}
```

**小技巧:** 这里用到了命名空间下常量功能(`PHP7`开始支持)，定义了一个同名的 `sha256WithRSAEncryption` 哈希别名常量，RSA下的`SHA256`哈希散列别名，这个检测其实是多余的，在未来的某个版本，可以安全地移除掉。

## Crypto\Rsa::encrypt 公钥加密

既然是要重写，首先要考虑易用，那静态方法其实比实例化后使用方便得多，代码块如下：

```php
<?php
/**
 * Encrypts text with `OPENSSL_PKCS1_OAEP_PADDING`.
 *
 * @param string $plaintext - Cleartext to encode.
 * @param \OpenSSLAsymmetricKey|\OpenSSLCertificate|object|resource|string|mixed $publicKey - A PEM encoded public key.
 *
 * @return string - The base64-encoded ciphertext.
 * @throws UnexpectedValueException
 */
public static function encrypt(string $plaintext, $publicKey): string
{
    if (!openssl_public_encrypt($plaintext, $encrypted, $publicKey, OPENSSL_PKCS1_OAEP_PADDING)) {
        throw new UnexpectedValueException('Encrypting the input $plaintext failed, please checking your $publicKey whether or nor correct.');
    }

    return base64_encode($encrypted);
}
```

函数接受两个参数，同时对返回值做了类型签名，所接受的第二参数 `$publicKey` 是透传给 `openssl_public_encrypt` 函数的，所以可以接受的类型范围比较广。 这里捎带提一下，`PHP8` 有许多改进，尤其是把`OpenSSL`相关的原`资源`类型，现在定义成`对象`了，即代码注释上的: `\OpenSSLAsymmetricKey|\OpenSSLCertificate`，这俩是`PHP8`上才有的。

在加入`PHPStan`代码静态分析工具后，这里就稍显尴尬了，因为本SDK最低版本要兼容至`PHP7.2`，迭代过程中，前后兼容`PHP8`是个挑战，遂加入了 `phpstan-baseline.neon` 基线，特意区分开了 `phpstan-php7.neon` 及 `phpstan.neon.dist` 各两个配置文件，静态分析从4级(`level3`)提升至6级(`level5`)再至7级(`level6`)，以至最高级别(`level8`/`max`)做了大量的代码注释修正以及代码优化。 目前看到的即是`最高等级`静态分析的代码。

**小技巧**: 这里同样用到了`PHP7`命名空间下声明使用常量功能，即 `use const OPENSSL_PKCS1_OAEP_PADDING;`。所以在中间代码块上，可以不用再特别注意 `FQN`,可以安全使用。

我们用测试用例来覆盖一下：

```php
<?php
const BASE64_EXPRESSION = '#^[a-zA-Z0-9][a-zA-Z0-9\+/]*={0,2}$#';
/**
 * @return array<string,array{string,string|resource|mixed,resource|mixed}>
 */
public function keysProvider(): array
{
    $privateKey = openssl_pkey_new([
        'digest_alg' => 'sha256',
        'default_bits' => 2048,
        'private_key_bits' => 2048,
        'private_key_type' => OPENSSL_KEYTYPE_RSA,
        'config' => dirname(__DIR__) . DS . 'fixtures' . DS . 'openssl.conf',
    ]);

    while ($msg = openssl_error_string()) {
        'cli' === PHP_SAPI && fwrite(STDERR, 'OpenSSL ' . $msg . PHP_EOL);
    }

    ['key' => $publicKey] = $privateKey ? openssl_pkey_get_details($privateKey) : [];

    return [
        'plaintext, publicKey and privateKey' => ['hello wechatpay 你好 微信支付', $publicKey, $privateKey]
    ];
}
/**
 * @dataProvider keysProvider
 * @param string $plaintext
 * @param object|resource|mixed $publicKey
 */
public function testEncrypt(string $plaintext, $publicKey): void
{
    $ciphertext = Rsa::encrypt($plaintext, $publicKey);
    self::assertIsString($ciphertext);
    self::assertNotEquals($plaintext, $ciphertext);

    if (method_exists($this, 'assertMatchesRegularExpression')) {
        $this->assertMatchesRegularExpression(BASE64_EXPRESSION, $ciphertext);
    } else {
        self::assertRegExp(BASE64_EXPRESSION, $ciphertext);
    }
}
```

`BASE64_EXPRESSION` 是个命名空间常量，是 `base64` 字符串的一个正则匹配规则，相较于`Formatter`类内置的 `bas64` 检测规则，这里做了调整，加入来**必须是字母或数字开头**规则。

有人可能会问，这里为什么不用`\w\d`代替呢？答案是：按照`base64`规范，只能出现**字母或数字或加号或斜线**，`\w` 是 `\word` 的简写， `\word` 存在语言适配表现不一致情况，即在法语系内，部分字符也是匹配到了 `\w` 内，这是其一；其二就是 `\d` 按照PHP官方文档介绍，是`decial digit`的简写，`decial`可能会带入点号(`.`)及逗号(`,`)，不严谨，遂还是按照`base64`规范来。

另外，这里的数据供给器`keysProvider`函数，调试调整了一段时间，思考如下：

1. 相较于传统使用文件`fixtures`来提供`RSA`私钥/公钥，使是函数生成，是为了更安全的被使用在测试场景中；
2. 这里尝试更范的场景覆盖，每轮生成的`私钥`、`公钥`理论上不一样，覆盖会更广；

在`数据供给器`生成环节，检测出一个问题就是，在windows上，`PHP7.2/7.3`与`7.4+`表现不一致，内置的 `openssl_pkey_new` 函数在`7.2/7.3`上不工作。这真是“意外”中的意外。

在翻了PHP源码以及百谷歌度之后，最后从PHP手册上找到了线索如下：

> **Note: Note to Win32 Users**
>
>> Additionally, if you are planning to use the key generation and certificate signing functions, you will need to install a valid `openssl.cnf` file on your system. 

随后又翻了下PHP的变更历史，`PHP7.4.0`对windows环境做了优化，C++代码做了自动搜索`openssl.cnf`文件并取默认值。前向兼容方案遂如上述代码，在`私钥`生成时，指定配置文件即可。

**小技巧:**

1. `ext-openssl`在工作时，会在各个阶段把异常信息打入堆栈中，可以通过 `openssl_error_string` 获取到堆栈信息；
2. 在测试环境下，本测试供给器函数，把这些“错误”信息，使用了 `fwrite` 直接写入至 `STDERR` 管道，仅在`CLI`模式下有效；
3. 数组`Array`解构，除了用`list`顺序解构(`PHP7+`)之外，还可以通过键值`key`来解构，即 `['key' => $publicKey] = []` 形式来解构;

## Crypto\Rsa::decrypt 私钥解密

对应地，私钥解密也变得用起来简单得多了，型参类型签名，返回值类型签名，代码块如下：

```php
<?php
/**
 * Decrypts base64 encoded string with `privateKey` with `OPENSSL_PKCS1_OAEP_PADDING`.
 *
 * @param string $ciphertext - Was previously encrypted string using the corresponding public key.
 * @param \OpenSSLAsymmetricKey|\OpenSSLCertificate|resource|string|mixed $privateKey - A PEM encoded private key.
 *
 * @return string - The utf-8 plaintext.
 * @throws UnexpectedValueException
 */
public static function decrypt(string $ciphertext, $privateKey): string
{
    if (!openssl_private_decrypt(base64_decode($ciphertext), $decrypted, $privateKey, OPENSSL_PKCS1_OAEP_PADDING)) {
        throw new UnexpectedValueException('Decrypting the input $ciphertext failed, please checking your $privateKey whether or nor correct.');
    }

    return $decrypted;
}
```

如前所属，每轮测试的数据供给是不一样的，所以得从加密开始，测试用例如下：

```php
<?php
/**
 * @dataProvider keysProvider
 * @param string $plaintext
 * @param object|resource|mixed $publicKey
 * @param object|resource|mixed $privateKey
 */
public function testDecrypt(string $plaintext, $publicKey, $privateKey): void
{
    $ciphertext = Rsa::encrypt($plaintext, $publicKey);
    self::assertIsString($ciphertext);
    self::assertNotEquals($plaintext, $ciphertext);

    if (method_exists($this, 'assertMatchesRegularExpression')) {
        $this->assertMatchesRegularExpression(BASE64_EXPRESSION, $ciphertext);
    } else {
        self::assertRegExp(BASE64_EXPRESSION, $ciphertext);
    }

    $mytext = Rsa::decrypt($ciphertext, $privateKey);
    self::assertIsString($mytext);
    self::assertEquals($plaintext, $mytext);
}
```

这里有个知识点需要补充一下，即，`publicKey` 公钥和 `privateKey` 私钥是配对的，公钥可以从私钥提取、也可以从私钥签发的证书提取。当前测试用例是从私钥提取的，后边再讲从`证书`提取。

## Crypto\Rsa::sign 私钥签名

顾名思义，`私钥`理应是私密的，用来做签名，具有不可篡改特性。签名封装代码如下:

```php
<?php
/**
 * Creates and returns a `base64_encode` string that uses `sha256WithRSAEncryption`.
 *
 * @param string $message - Content will be `openssl_sign`.
 * @param \OpenSSLAsymmetricKey|\OpenSSLCertificate|object|resource|string|mixed $privateKey - A PEM encoded private key.
 *
 * @return string - The base64-encoded signature.
 * @throws UnexpectedValueException
 */
public static function sign(string $message, $privateKey): string
{
    static::preCondition();

    if (!openssl_sign($message, $signature, $privateKey, sha256WithRSAEncryption)) {
        throw new UnexpectedValueException('Signing the input $message failed, please checking your $privateKey whether or nor correct.');
    }

    return base64_encode($signature);
}
```

测试代码如下：

```php
<?php
/**
 * @dataProvider keysProvider
 * @param string $plaintext
 * @param object|resource|mixed $publicKey
 * @param object|resource|mixed $privateKey
 */
public function testSign(string $plaintext, $publicKey, $privateKey): void
{
    $signature = Rsa::sign($plaintext, $privateKey);

    self::assertIsString($signature);
    self::assertNotEquals($plaintext, $signature);

    if (method_exists($this, 'assertMatchesRegularExpression')) {
        $this->assertMatchesRegularExpression(BASE64_EXPRESSION, $signature);
    } else {
        self::assertRegExp(BASE64_EXPRESSION, $signature);
    }
}
```

因为使用了同一套`数据供给器`代码，所以这个测试用例上，第二参数`$publicKey`还得加上（虽然没用）。

## Crypto\Rsa::verify 公钥验签

这个验签逻辑，可以用来理解`非对称加密技术`。如上一小结，`私钥数据签名`的数据，一般私钥是需要严密保存的，基本不会对外分发。那问题来了，收到加密数据的接收方，应该如何验证数据签名来自`预期的数据签名方`呢？`公钥验签`就是来解决这个`数据`及`数据签名`真伪的一种方式。

```php
<?php
/**
 * Verifying the `message` with given `signature` string that uses `sha256WithRSAEncryption`.
 *
 * @param string $message - Content will be `openssl_verify`.
 * @param string $signature - The base64-encoded ciphertext.
 * @param \OpenSSLAsymmetricKey|\OpenSSLCertificate|object|resource|string|mixed $publicKey - A PEM encoded public key.
 *
 * @return boolean - True is passed, false is failed.
 * @throws UnexpectedValueException
 */
public static function verify(string $message, string $signature, $publicKey): bool
{
    static::preCondition();

    if (($result = openssl_verify($message, base64_decode($signature), $publicKey, sha256WithRSAEncryption)) === false) {
        throw new UnexpectedValueException('Verified the input $message failed, please checking your $publicKey whether or nor correct.');
    }

    return $result === 1;
}
```

小知识：上一小结提到，`私钥`和`公钥`是配对出现的，`公钥`含在`私钥`及`证书`里，所以验签逻辑的`公钥`输入，可以是源`私钥`，也可以是源`私钥`签发的`证书`，即代码注释里的`\OpenSSLAsymmetricKey`及`\OpenSSLCertificate`。

至此，01章节`格式化请求参数`及`格式化响应参数`提到的两个关键函数 `Rsa::sign` 及 `Rsa::verify` 也讲解完了，微信支付APIv3的核心部件，通过这两个静态类，共计10余个函数就抽象完成了。

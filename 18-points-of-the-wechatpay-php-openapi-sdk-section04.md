---
title: "微信支付PHP开发对接18讲——04: AesGcm AES-GCM加解密"
date: 2021-07-02T11:39:11+08:00
lastmod: 2021-07-02T11:45:33+08:00
keywords: ["微信支付", "WeChatPay PHP SDK", "GuzzleHttp", "PHPStan Level8"]
description: "官方`APIv3`做了安全加强，对于敏感信息及回调通知信息，才有了`AES-GCM`加密，依赖商户平台配置`APIv3密钥`。PHP自7.1开始支持`GCM`模式。上一讲提到了`协变`（covariant）设计规则，这个类的实现就是对`AesInterface`做了方法入参扩展，分别如下。"
tags: []
toc: false
categories: [php, wechat]
summary: "官方`APIv3`做了安全加强，对于敏感信息及回调通知信息，才有了`AES-GCM`加密，依赖商户平台配置`APIv3密钥`。PHP自7.1开始支持`GCM`模式。上一讲提到了`协变`（covariant）设计规则，这个类的实现就是对`AesInterface`做了方法入参扩展，分别如下。"
author: "James"
---

官方`APIv3`做了安全加强，对于敏感信息及回调通知信息，才有了`AES-GCM`加密，依赖商户平台配置`APIv3密钥`。PHP自7.1开始支持`GCM`模式。上一讲提到了`协变`（covariant）设计规则，这个类的实现就是对`AesInterface`做了方法入参扩展，分别如下。

## Crypto\AesGcm::preCondition 前置条件检测

这个方法如同`Rsa::preCondition`类似，检测当前`ext-openssl`扩展，是否支持`aes-256-gcm`加解密算法，均可在未来的版本中安全删除。

```php
<?php
/**
 * Detect the ext-openssl whether or nor including the `aes-256-gcm` algorithm
 *
 * @throws RuntimeException
 */
private static function preCondition(): void
{
    if (!in_array(static::ALGO_AES_256_GCM, openssl_get_cipher_methods())) {
        throw new RuntimeException('It looks like the ext-openssl extension missing the `aes-256-gcm` cipher method.');
    }
}
```

扩展知识：`static::ALGO_AES_256_GCM` 是`PHP7`中的`延迟静态绑定(late static bindings)`，其作用域是要看运行时的上下文类，更多知识参阅PHP官方文档。

## Crypto\AesGcm::encrypt 加密

这个方法，在官方的`wechatpay-guzzle-middleware`没有实现，这是新包新增的。从这个我们能窥出一丢丢官方接口设计上的一些`分歧点`。

官方文档上的`敏感信息加解密`，上下行均才有RSA证书加密模式，RSA是`非对称加解密`，公钥加密/私钥解密，可以提供极佳的安全体验。而在`证书及回调通知`时，却采用的是`对称加解密`方案，加密均由平台方来完成，商户侧仅需在收到报文进行解密即可。

为什么在`敏感信息加解密`使用`非对称加解密`，而在`证书及回调通知`使用`对称加解密`，唯一合理的解释就是为了`安全`，“良苦用心”没有明说，然这个没明说却给对接`APIv3`带出了许多“难以理解”；而不提供对称加密函数，这又让人不得不把问题上升到哲学层面（我认为你不需要，所以我不提供了）唉。。。

实现这个函数，也就几行代码如下：

```php
<?php
/**
 * Encrypts given data with given key, iv and aad, returns a base64 encoded string.
 *
 * @param string $plaintext - Text to encode.
 * @param string $key - The secret key, 32 bytes string.
 * @param string $iv - The initialization vector, 16 bytes string.
 * @param string $aad - The additional authenticated data, maybe empty string.
 *
 * @return string - The base64-encoded ciphertext.
 */
public static function encrypt(string $plaintext, string $key, string $iv = '', string $aad = ''): string
{
    static::preCondition();

    $ciphertext = openssl_encrypt($plaintext, static::ALGO_AES_256_GCM, $key, OPENSSL_RAW_DATA, $iv, $tag, $aad, static::BLOCK_SIZE);

    if (false === $ciphertext) {
        throw new UnexpectedValueException('Encrypting the input $plaintext failed, please checking your $key and $iv whether or nor correct.');
    }

    return base64_encode($ciphertext . $tag);
}
```

测试代码如下:

```php
<?php
const BASE64_EXPRESSION = '#^[a-zA-Z0-9\+/]+={0,2}$#';

/**
 * @return array<string,array{string,string,string,string}>
 */
public function dataProvider(): array
{
    return [
        'random key and iv' => [
            'hello wechatpay 你好 微信支付',
            Formatter::nonce(AesGcm::KEY_LENGTH_BYTE),
            Formatter::nonce(AesGcm::BLOCK_SIZE),
            ''
        ],
        'random key, iv and aad' => [
            'hello wechatpay 你好 微信支付',
            Formatter::nonce(AesGcm::KEY_LENGTH_BYTE),
            Formatter::nonce(AesGcm::BLOCK_SIZE),
            Formatter::nonce(AesGcm::BLOCK_SIZE)
        ],
    ];
}

/**
 * @dataProvider dataProvider
 * @param string $plaintext
 * @param string $key
 * @param string $iv
 * @param string $aad
 */
public function testEncrypt(string $plaintext, $key, $iv, $aad): void
{
    $ciphertext = AesGcm::encrypt($plaintext, $key, $iv, $aad);
    self::assertIsString($ciphertext);
    self::assertNotEquals($plaintext, $ciphertext);

    if (method_exists($this, 'assertMatchesRegularExpression')) {
        $this->assertMatchesRegularExpression(self::BASE64_EXPRESSION, $ciphertext);
    } else {
        self::assertRegExp(self::BASE64_EXPRESSION, $ciphertext);
    }
}
```

目前看的这个测试用例，是修正后的，期间翻了一次车。。。因由是由`BASE64_EXPRESSION`类常量的定义引起的。

测试用例覆盖，采用了与`RsaTest`类似的动态生成`数据供给`方案，为验证加密后的字符串是`base64`而不是其他，所以需要个`规则`来判断字符串是不是`base64`，这里采用了正则表达式。正是这个表达式，翻了车了。

上一版的正则表达式为`#^[a-zA-Z0-9][a-zA-Z0-9\+/]*={0,2}$#`，我们来回溯一些测试样本数据：

```
('hello wechatpay 你好 微信支付', 'RSXrQ0bANKaUGdbvWwPENFNjhftB6EYs', 'XxG5mkSo7DBiGxSN', '') -> '/bXfSUzxl3dcrGBbduG6Jh9vd269iRzO91qSRnzzLl+RPxH6fVPS6hKPlC3hADltDKuU'
('hello wechatpay 你好 微信支付', '0hVcffnbHcx9zpyi9bgmbGtDZHOXuq6V', 'In4deshcFFOhdyTs', 'lUY1Dm04bXRhj4Z1') -> '+RwaCGnFNMJPezHifxSBjEuJR3LBYndNLZHO1gV9cj5/hlL55hwlcNzpAOr/1Vm42hp8'
('hello wechatpay 你好 微信支付', '0ZehDc6SnHPcEzqXv18Qiikz0syFvUoO', 'urJE0OMNEuVYwY9Y', '') -> '/LtXN1bbvlxubbgypv23QKsdIw14RAhsL1GNUHAfwfEBBNp2elvcy7mw8D8KUOJ4VIUC'
```

`base64字符串+/`也可以出现在起始位，翻新了我对`base64`的认知。。。

## Crypto\AesGcm::decrypt 解密

解密函数对官方源版做了部分调整，调整点是对`auth_tag`长度判断上进行判断。按照php官方手册上说`openssl_decrypt`在出来`GCM`模式密文时，调用方要自行判断`auth_tag`的长度。实现代码如下：

```php
<?php
/**
 * Takes a base64 encoded string and decrypts it using a given key, iv and aad.
 *
 * @param string $ciphertext - The base64-encoded ciphertext.
 * @param string $key - The secret key, 32 bytes string.
 * @param string $iv - The initialization vector, 16 bytes string.
 * @param string $aad - The additional authenticated data, maybe empty string.
 *
 * @return string - The utf-8 plaintext.
 */
public static function decrypt(string $ciphertext, string $key, string $iv = '', string $aad = ''): string
{
    static::preCondition();

    $ciphertext = base64_decode($ciphertext);
    $authTag = substr($ciphertext, intval(-static::BLOCK_SIZE));
    $tagLength = strlen($authTag);

    /* Manually checking the length of the tag, because the `openssl_decrypt` was mentioned there, it's the caller's responsibility. */
    if ($tagLength > static::BLOCK_SIZE || ($tagLength < 12 && $tagLength !== 8 && $tagLength !== 4)) {
        throw new RuntimeException('The inputs `$ciphertext` incomplete, the bytes length must be one of 16, 15, 14, 13, 12, 8 or 4.');
    }

    $plaintext = openssl_decrypt(substr($ciphertext, 0, intval(-static::BLOCK_SIZE)), static::ALGO_AES_256_GCM, $key, OPENSSL_RAW_DATA, $iv, $authTag, $aad);

    if (false === $plaintext) {
        throw new UnexpectedValueException('Decrypting the input $ciphertext failed, please checking your $key and $iv whether or nor correct.');
    }

    return $plaintext;
}
```

测试代码如下：

```php
<?php
/**
 * @return array<string,array{string,string,string,string}>
 */
public function dataProvider(): array
{
    return [
        'random key and iv' => [
            'hello wechatpay 你好 微信支付',
            Formatter::nonce(AesGcm::KEY_LENGTH_BYTE),
            Formatter::nonce(AesGcm::BLOCK_SIZE),
            ''
        ],
        'random key, iv and aad' => [
            'hello wechatpay 你好 微信支付',
            Formatter::nonce(AesGcm::KEY_LENGTH_BYTE),
            Formatter::nonce(AesGcm::BLOCK_SIZE),
            Formatter::nonce(AesGcm::BLOCK_SIZE)
        ],
    ];
}

/**
 * @dataProvider dataProvider
 * @param string $plaintext
 * @param string $key
 * @param string $iv
 * @param string $aad
 */
public function testDecrypt(string $plaintext, $key, $iv, $aad): void
{
    $ciphertext = AesGcm::encrypt($plaintext, $key, $iv, $aad);
    self::assertIsString($ciphertext);
    self::assertNotEquals($plaintext, $ciphertext);

    if (method_exists($this, 'assertMatchesRegularExpression')) {
        $this->assertMatchesRegularExpression(self::BASE64_EXPRESSION, $ciphertext);
    } else {
        self::assertRegExp(self::BASE64_EXPRESSION, $ciphertext);
    }

    $mytext = AesGcm::decrypt($ciphertext, $key, $iv, $aad);
    self::assertIsString($mytext);
    self::assertEquals($plaintext, $mytext);
}
```

至此，`APIv3`上的包括证书及回调所需的函数，均封装完毕，下一讲就对接`HttpClient`，来驱动请求响应。

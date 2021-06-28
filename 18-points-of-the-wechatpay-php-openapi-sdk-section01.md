---
title: "微信支付PHP开发对接18讲——01: Formatter 从格式化参数说起"
date: 2021-06-27T17:00:11+08:00
lastmod: 2021-06-28T12:11:48+08:00
keywords: ["微信支付", "WeChatPay PHP SDK", "GuzzleHttp", "PHPStan Level8"]
description: "SDK目标是优先`APIv3`版，当然也需要考虑当下，`APIv2`还在并行运行。两者之间有共性也有特性，把共性部分抽象出来，当属`格式化`参数部分。`随机字符串`首当其冲，那就从这个函数实现开始吧。"
tags: []
toc: false
categories: [php, wechat]
summary: "SDK目标是优先`APIv3`版，当然也需要考虑当下，`APIv2`还在并行运行。两者之间有共性也有特性，把共性部分抽象出来，当属`格式化`参数部分。`随机字符串`首当其冲，那就从这个函数实现开始吧。"
author: "James"
---

SDK目标是优先`APIv3`版，当然也需要考虑当下，`APIv2`还在并行运行。两者之间有共性也有特性，把共性部分抽象出来，当属`格式化`参数部分。`随机字符串`首当其冲，那就从这个函数实现开始吧。

## Formatter::nonce 随机字符串产生器

本博发布之日，这个函数已经迭代了一个版本，并且加入了测试用例覆盖，最终形态如下：

```php
<?php
/**
 * Generate a random ASCII string aka `nonce`, similar as `random_bytes`.
 *
 * @param int $size - Nonce string length, default is 32.
 *
 * @return string - base62 random string.
 */
public static function nonce(int $size = 32): string
{
    return array_reduce(range(1, $size), static function(string $char) {
        return $char .= 'abcdefghijklmnopqrstuvwxyz0123456789ABCDEFGHIJKLMNOPQRSTUVWXYZ'[mt_rand(0, 61)];
    }, '');
}
```

本来是可以一行显示的，为了阅读起来方便，特意做了格式化，这个函数使用了PHP内置的三个函数`array_reduce`, `range` 及 `mt_rand`，并且使用了 `Closure` `Static Anonymous Functions`。
此函数的设计思路是：根据入参`$size`，先构建一个堆栈，然后从 `Base62` 字符串内随机取个数，填充堆栈并合并返回。

上一版是这么实现的：

```php
<?php
const BASE62_CHARS = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';

public static function nonce(int $size = 32): string
{
    return preg_replace_callback('#0#', static function() {
        return BASE62_CHARS[rand(0, 61)];
    }, str_repeat('0', $size));
}
```

使用了 `preg_replace_callback` `rand` 及 `str_repeat` 三个内置函数，同样的设计思路，不过有个缺陷，即 `$size` 型参必须大于0，这个是受限 `str_repeat` 型参要求。

两个函数其实都备注有与PHP内置的 `random_bytes` 函数相似，而 `random_bytes` 也存在型参 `$size` 必须大于0的要求，其返回值是 `Base36` 的。

最终选择 `array_reduce+range+mt_rand` 的组合，这里是做了性能测试的，测试代码如下：

```php
// file: cli.php, run: php -f cli.php
<?php
const BASE62_CHARS = '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ';

$start = microtime(true);
$a = '';
for ($i = 0; $i < 50; $i++) {
    $a .= BASE62_CHARS[mt_rand(0, 61)];
}
printf(
    '[%30s] Time: %.7f s  %-32.32s %s', 'preg_replace_callback',
    microtime(true) - $start, $a, PHP_EOL
);

$start = microtime(true);
$a = preg_replace_callback('#0#', static function() {
    return BASE62_CHARS[rand(0, 61)];
}, str_repeat('0', 50));
printf(
    '[%30s] Time: %.7f s  %-32.32s %s', 'preg_replace_callback',
    microtime(true) - $start, $a, PHP_EOL
);

$start = microtime(true);
$a = array_reduce(range(1, 50), static function(string $c) {
    return $c .= '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ'[random_int(0, 61)];
}, '');
printf(
    '[%30s] Time: %.7f s  %-32.32s %s', 'preg_replace_callback',
    microtime(true) - $start, $a, PHP_EOL
);

$start = microtime(true);
$a = array_reduce(range(1, 50), static function(string $c) {
    return $c .= '0123456789abcdefghijklmnopqrstuvwxyzABCDEFGHIJKLMNOPQRSTUVWXYZ'[mt_rand(0, 61)];
}, '');
printf(
    '[%30s] Time: %.7f s  %-32.32s %s', 'preg_replace_callback',
    microtime(true) - $start, $a, PHP_EOL
);

$start = microtime(true);
$a = substr(bin2hex(random_bytes(50)), 0, 50);
printf(
    '[%30s] Time: %.7f s  %-32.32s %s', 'preg_replace_callback',
    microtime(true) - $start, $a, PHP_EOL
);
```

跑上四圈的结果如下：

```
[                      for loop] Time: 0.0004230 s  twqdlCDCyMknrFMJEjPfl94SdFK2a2Km
[         preg_replace_callback] Time: 0.0035770 s  aTb7yrpFVitQju7sKAgofTuiSsiw1Top
[       array_reduce/random_int] Time: 0.0001869 s  9IkLR7HEryOy8zlzuxpVIBpttpprlNib
[          array_reduce/mt_rand] Time: 0.0000098 s  7j2F2stRXiHvAxQ4j2IhplHXCHMGtl9j
[   substr/bin2hex/random_bytes] Time: 0.0000100 s  e6e33fd212b3083bfc4ebb29a13710b1


[                      for loop] Time: 0.0000420 s  UppY2CoJOVSmHE0EkynhyOeqaWdvSGJ5
[         preg_replace_callback] Time: 0.0002630 s  s64pxnpKALGq16HxnbCbKh0NGgRq8Ou2
[       array_reduce/random_int] Time: 0.0001841 s  7yYaGsWoBudf5PKlz2OkdodGrN05OkQo
[          array_reduce/mt_rand] Time: 0.0000188 s  0ruErA844vf9CifmfhEiyjoUVcbgv3yy
[   substr/bin2hex/random_bytes] Time: 0.0000091 s  db57e8c90751d00b76da5d553f54c950


[                      for loop] Time: 0.0000441 s  0hZgmb4EeYuL14FDd0UIkbjiNyjGjZxy
[         preg_replace_callback] Time: 0.0002651 s  BMoSsbN2N7ObEDmqpgtTKKtGdMiUuV4U
[       array_reduce/random_int] Time: 0.0001869 s  7cGZKBUGiZL6v594o5k6wtQTmm5I7IYq
[          array_reduce/mt_rand] Time: 0.0000200 s  leXlVr5aJ8zmhbn9kk27D3bpy4FJ6Mhy
[   substr/bin2hex/random_bytes] Time: 0.0000100 s  7c9de4e1ee533aa3c9ddecf78667afaf


[                      for loop] Time: 0.0000479 s  OpqudO593AKmRROllDX8h0tjuBzLXPQZ
[         preg_replace_callback] Time: 0.0002871 s  oXoJEBRt0Qpy5jXUNDnAapblbXhpBFI3
[       array_reduce/random_int] Time: 0.0002170 s  SPdIjbbiI3wKLrMCnAiurpBdnxeJsip6
[          array_reduce/mt_rand] Time: 0.0000169 s  bKygHiTtIj6LsnmWxRKXTtSpr1oIdkBJ
[   substr/bin2hex/random_bytes] Time: 0.0000169 s  f8c4105678ba0e97f41fbb9197332f28
```

其中 `preg_replace_callback` 组合是最慢的， `array_reduce/mt_rand` 组合与最快的 `random_bytes` 很接近。

`array_reduce/mt_rand` 接受的入参可以是负数，也可以是正数，从严谨性上来取舍，遂选择了这个组合。
[测试用例](https://github.com/TheNorthMemory/wechatpay-php/blob/5fb1c64a154cc63bbff730e59c0632c0f6e45043/tests/FormatterTest.php)如下：

```php
<?php
public function nonceRulesProvider(): array
{
    return [
        'default $size=32'       => [32,  '/[a-zA-Z0-9]{32}/'],
        'half-default $size=16'  => [16,  '/[a-zA-Z0-9]{16}/'],
        'hundred $size=100'      => [100, '/[a-zA-Z0-9]{100}/'],
        'one $size=1'            => [1,   '/[a-zA-Z0-9]{1}/'],
        'zero $size=0'           => [0,   '/[a-zA-Z0-9]{2}/'],
        'negative $size=-1'      => [-1,  '/[a-zA-Z0-9]{3}/'],
        'negative $size=-16'     => [-16, '/[a-zA-Z0-9]{18}/'],
        'negative $size=-32'     => [-32, '/[a-zA-Z0-9]{34}/'],
    ];
}
/**
 * @dataProvider nonceRulesProvider
 */
public function testNonce(int $size, string $pattern): void
{
    $nonce = Formatter::nonce($size);

    self::assertIsString($nonce);

    self::assertTrue(strlen($nonce) === ($size > 0 ? $size : abs($size - 2)));

    if (method_exists($this, 'assertMatchesRegularExpression')) {
        self::assertMatchesRegularExpression($pattern, $nonce);
    } else {
        self::assertRegExp($pattern, $nonce);
    }
}
```

适应性上堪称完美。

**一个重要提示**： 按照PHP手册上提示，`mt_rand`是密码学不安全的函数，这里也做一并提示 `Formatter::nonce()` 也是密码学不安全实现。本类库在使用时，仅当`salt`（盐）使用，扩展使用时，请注意使用场景。

## Formatter::timestamp 时间戳

这个函数是对`time()`的一个及其简单的一个封装。之所以要封装，其实是有一点点说法的。按照微信支付官方开发文档说明，时间戳是自1970年1月1日起的`Unix timesamp`，即 Epoch timesamp。PHP内置`time()`函数就是这个值。其他平台，有见到把`timesamp`翻译成`yyyy-MM-dd HH:mm:ss`格式的字符串，做这么个封装的原因：

1. 对函数返回值做类型签名，严格区分其他平台的翻译；
2. PHP的命名空间`namespace`，存在`FQN`引用，自`PHP7`开始，在命名空间下可以`use function`，为让代码通俗易懂，在一个地方引用内置函数，比多次引用要直观；

代码块如下

```php
<?php
/**
 * Retrieve the current `Unix` timestamp.
 *
 * @return int - Epoch timestamp.
 */
public static function timestamp(): int
{
    return time();
}
```

[测试用例](https://github.com/TheNorthMemory/wechatpay-php/blob/5fb1c64a154cc63bbff730e59c0632c0f6e45043/tests/FormatterTest.php)如下：

```php
<?php
public function testTimestamp(): void
{
    $timestamp = Formatter::timestamp();
    $pattern = '/^1[0-9]{9}/';

    self::assertIsInt($timestamp);

    $timestamp = strval($timestamp);

    self::assertTrue(strlen($timestamp) === 10);

    if (method_exists($this, 'assertMatchesRegularExpression')) {
        self::assertMatchesRegularExpression($pattern, $timestamp);
    } else {
        self::assertRegExp($pattern, $timestamp);
    }
}
```

1开头的10位纯数字，记住了，这就是 `Unix timestamp` 。

### Formatter::authorization 认证值

这个函数，官方文档说的很详细了，但是内涵相当多的知识点，没有明说(哲学：我认为你应该知道所以我不讲了)，这个函数就是其中之一。翻[MDN](https://developer.mozilla.org/zh-CN/docs/Web/HTTP/Headers/Authorization)，引申有说明，[RFC 7235, section 4.2: Authorization](https://tools.ietf.org/html/rfc7235#section-4.2)：只要符合规范，厂商可自主实现`<type> <credentials>`。

微信支付 `APIv3` 即以 `WECHATPAY2-SHA256-RSA2048` 声明 `type` 变量，`mchid="%s",nonce_str="%s",signature="%s",timestamp="%s",serial_no="%s"` 组合实现 `credentials` 声明。

代码块如下：

```php
<?php
/**
 * Formatting for the heading `Authorization` value.
 *
 * @param string $mchid - The merchant ID.
 * @param string $nonce - The Nonce string.
 * @param string $signature - The base64-encoded `Rsa::sign` ciphertext.
 * @param string $timestamp - The `Unix` timestamp.
 * @param string $serial - The serial number of the merchant public certification.
 *
 * @return string - The APIv3 Authorization `header` value
 */
public static function authorization(string $mchid, string $nonce, string $signature, string $timestamp, string $serial): string
{
    return sprintf(
        'WECHATPAY2-SHA256-RSA2048 mchid="%s",nonce_str="%s",signature="%s",timestamp="%s",serial_no="%s"',
        $mchid, $nonce, $signature, $timestamp, $serial
    );
}
```

那些官方没有明说的知识点就来了：

1. 商户号 `mchid` 是1至32字符的`base62`字符串，当前绝大部分商户号是纯数字；
2. 请求随机串 `nonce_str` 是至少16字符的`base62`字符串，上限不清楚，没测试过（不敢，怕封号)；
3. 签名值 `signature` 是`base64`字符串，base64末尾有可能有0个、1个或者2个`=`号，这都是正常的`base64`字符串；
4. 时间戳 `timestamp` 是 `unix timestamp`，如 `Formatter::timestamp()` 封装所述;
5. 商户API证书 `serial_no` 是8至40字符的`【0-9A-Z]`全大写字符串；
6. 认证值有严格的顺序要求，即 `mchid,nonce,signature,timestamp,serial` 非字典序，值的组合用半角逗号(,)分隔，严格没有空格；

我们用测试用例覆盖来校验如下：

```php
<?php
public function testAuthorization(): void
{
    $value = Formatter::authorization('1001', Formatter::nonce(), 'mock', (string) Formatter::timestamp(), 'mockmockmock');

    self::assertIsString($value);

    self::assertStringStartsWith('WECHATPAY2-SHA256-RSA2048 ', $value);
    self::assertStringEndsWith('"', $value);

    $pattern = '/^WECHATPAY2-SHA256-RSA2048 '
        . 'mchid="[0-9A-Za-z]{1,32}",'
        . 'nonce_str="[0-9A-Za-z]{16,}",'
        . 'signature="[0-9A-Za-z\+\/]+={0,2}",'
        . 'timestamp="1[0-9]{9}",'
        . 'serial_no="[0-9A-Z]{8,40}"$/';

    if (method_exists($this, 'assertMatchesRegularExpression')) {
        self::assertMatchesRegularExpression($pattern, $value);
    } else {
        self::assertRegExp($pattern, $value);
    }
}
```

PS: 官方文档上，特意说了 认证类型 `<type>`，目前为 `WECHATPAY2-SHA256-RSA2048`，畅想应该还会有其他值。所以在`APIv3`开发的时候，建议还是要**多看看官方文档/公告说明**，以免 `我的代码没动过啊，为什么现在不行了` 这类问题产生。

## Formatter::request 请求字符串

顾名思义，这个函数就是用来格式化请求参数的，按照官方开发文档介绍，输入参数是需要按照"\n"(char`0x0A`)做排列合并，而`0x0A`是个非打印字符，文本描述起来比较困难，另外对于空请求串情况，占位行不能省略，顾做一个纵向封装，以程序语言(函数型参)来描述请求串，代码块如下：

```php
<?php
/**
 * Formatting this `HTTP::request` for `Rsa::sign` input.
 *
 * @param string $method - The HTTP verb, must be the uppercase sting.
 * @param string $uri - Combined string with `URL::pathname` and `URL::search`.
 * @param string $timestamp - The `Unix` timestamp, should be the one used in `authorization`.
 * @param string $nonce - The `Nonce` string, should be the one used in `authorization`.
 * @param string $body - The playload string, HTTP `GET` should be an empty string.
 *
 * @return string - The content for `Rsa::sign`
 */
public static function request(string $method, string $uri, string $timestamp, string $nonce, string $body = ''): string
{
    return static::joinedByLineFeed($method, $uri, $timestamp, $nonce, $body);
}
```

函数入参接受5部分，其中`$body`可为空，内置驱动 `joinedByLineFeed` 做参数合并并返回字符串。这里有个点就是对入参 `$timestamp` 的类型定义，这里用了字符串定义，原因是：合并后的输出是个字符串，输入端就做了*妥协*，当然在非严格限制模式行，用纯数字输入也是可以的。

测试用例覆盖如下:

```php
<?php
const LINE_FEED = "\n";

public function requestPhrasesProvider(): array
{
    return [
        'DELETE root(/)' => ['DELETE', '/', ''],
        'DELETE root(/) with query' => ['DELETE', '/?hello=wechatpay', ''],
        'GET root(/)' => ['GET', '/', ''],
        'GET root(/) with query' => ['GET', '/?hello=wechatpay', ''],
        'POST root(/) with body' => ['POST', '/', '{}'],
        'POST root(/) with body and query' => ['POST', '/?hello=wechatpay', '{}'],
        'PUT root(/) with body' => ['PUT', '/', '{}'],
        'PUT root(/) with body and query' => ['PUT', '/?hello=wechatpay', '{}'],
        'PATCH root(/) with body' => ['PATCH', '/', '{}'],
        'PATCH root(/) with body and query' => ['PATCH', '/?hello=wechatpay', '{}'],
    ];
}

/**
 * @dataProvider requestPhrasesProvider
 */
public function testRequest(string $method, string $uri, string $body): void
{
    $value = Formatter::request($method, $uri, (string) Formatter::timestamp(), Formatter::nonce(), $body);

    self::assertIsString($value);

    self::assertStringStartsWith($method, $value);
    self::assertStringEndsWith(LINE_FEED, $value);
    self::assertLessThanOrEqual(substr_count($value, LINE_FEED), 5);

    $pattern = '#^' . $method . LINE_FEED
        .  preg_quote($uri) . LINE_FEED
        . '1[0-9]{9}' . LINE_FEED
        . '[0-9A-Za-z]{32}' . LINE_FEED
        . preg_quote($body) . LINE_FEED
        . '$#';

    if (method_exists($this, 'assertMatchesRegularExpression')) {
        self::assertMatchesRegularExpression($pattern, $value);
    } else {
        self::assertRegExp($pattern, $value);
    }
}
```

测试用例说明一下，共计十种情形，函数返回值，必须以所请求的 `$method` 开头，并且至少含有5个`LINE_FEED`常量，其中一个`LINE_FEED`在末尾。 用例的数据供给，含了已知的`APIv3` HTTP verbs，即：`DELETE`, `GET`, `POST`, `PUT` 及 `PATCH`。 按照 [rfc3986](https://tools.ietf.org/html/rfc3986) 规范，`DELETE`及`GET`是不带请求`$body`的。

## Formatter::response 响应字符串

这个函数是`APIv3`开发文档上验签逻辑的一个封装，直白代码如下：

```php
<?php
/**
 * Formatting this `HTTP::response` for `Rsa::verify` input.
 *
 * @param string $timestamp - The `Unix` timestamp, should be the one from `response::headers[Wechatpay-Timestamp]`.
 * @param string $nonce - The `Nonce` string, should be the one from `response::headers[Wechatpay-Nonce]`.
 * @param string $body - The response payload string, HTTP status(`201`, `204`) should be an empty string.
 *
 * @return string - The content for `Rsa::verify`
 */
public static function response(string $timestamp, string $nonce, string $body = ''): string
{
    return static::joinedByLineFeed($timestamp, $nonce, $body);
}
```

测试用例覆盖如下：

```php
<?php
public function responsePhrasesProvider(): array
{
    return [
        'HTTP 200 STATUS with body' => ['{}'],
        'HTTP 200 STATUS with no body' => [''],
        'HTTP 202 STATUS with no body' => [''],
        'HTTP 204 STATUS with no body' => [''],
        'HTTP 301 STATUS with no body' => [''],
        'HTTP 301 STATUS with body' => ['<html></html>'],
        'HTTP 302 STATUS with no body' => [''],
        'HTTP 302 STATUS with body' => ['<html></html>'],
        'HTTP 307 STATUS with no body' => [''],
        'HTTP 307 STATUS with body' => ['<html></html>'],
        'HTTP 400 STATUS with body' => ['{}'],
        'HTTP 401 STATUS with body' => ['{}'],
        'HTTP 403 STATUS with body' => ['<html></html>'],
        'HTTP 404 STATUS with body' => ['<html></html>'],
        'HTTP 500 STATUS with body' => ['{}'],
        'HTTP 502 STATUS with body' => ['<html></html>'],
        'HTTP 503 STATUS with body' => ['<html></html>'],
    ];
}

/**
 * @dataProvider responsePhrasesProvider
 */
public function testResponse(string $body): void
{
    $value = Formatter::response((string) Formatter::timestamp(), Formatter::nonce(), $body);

    self::assertIsString($value);

    self::assertStringEndsWith(LINE_FEED, $value);
    self::assertLessThanOrEqual(substr_count($value, LINE_FEED), 3);

    $pattern = '#^1[0-9]{9}' . LINE_FEED
        . '[0-9A-Za-z]{32}' . LINE_FEED
        . preg_quote($body) . LINE_FEED
        . '$#';

    if (method_exists($this, 'assertMatchesRegularExpression')) {
        self::assertMatchesRegularExpression($pattern, $value);
    } else {
        self::assertRegExp($pattern, $value);
    }
}
```

知识点如下：

1. HTTP请求返回内容，均是纯文本格式，即使是头部内容，解析出来也是字符串形式；
2. 与请求串格式化函数很像，这里接受3个参数，并且使用内置抽象函数`joinedByLineFeed`做数据合并；
3. 官方文档上，仅描述了`204`状态码时的返回内容为空，其实API接口，也有可能产生`202`状态码返回，内容也是空的；
4. `301/302/307/403/404/502/503`等状态码时，返回的内容有可能不是预期的`json`字符串，而是`html`串；
5. `$timestamp`/`$nonce` 是从HTTP HEADES上取，对应的key是`Wechatpay-Timestamp`及`Wechatpay-Nonce`，其实还有3个key非常有用，后边再讲;

## Formatter::joinedByLineFeed

## Formatter::ksort

## Formatter::queryStringLike

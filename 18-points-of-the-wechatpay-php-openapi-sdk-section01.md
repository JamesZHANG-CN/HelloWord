---
title: "微信支付PHP开发对接18讲——01: Formatter 从格式化参数说起"
date: 2021-06-27T17:00:11+08:00
keywords: ["微信支付", "WeChatPay PHP SDK", "GuzzleHttp", "PHPStan Level8"]
description: "SDK目标是优先`APIv3`版，当然也需要考虑当下，`APIv2`还在并行运行。两者之间有共性也有特性，把共性部分抽象出来，当属`格式化`参数部分。`随机字符串`首当其冲，那就从这个函数实现开始吧。"
tags: []
toc: false
categories: [php, wechat]
summary: "SDK目标是优先`APIv3`版，当然也需要考虑当下，`APIv2`还在并行运行。两者之间有共性也有特性，把共性部分抽象出来，当属`格式化`参数部分。`随机字符串`首当其冲，那就从这个函数实现开始吧。"
author: "James"
---

# 01: Formatter 从格式化参数说起

SDK目标是优先`APIv3`版，当然也需要考虑当下，`APIv2`还在并行运行。两者之间有共性也有特性，把共性部分抽象出来，当属`格式化`参数部分。`随机字符串`首当其冲，那就从这个函数实现开始吧。

## Formatter::nonce 随机字符串产生器

本博发布之日，这个函数已经迭代了一个版本，并且加入了测试用例覆盖，最终形态如下：

```php
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

## Formatter::timestamp

## Formatter::authorization

## Formatter::request

## Formatter::response

## Formatter::joinedByLineFeed

## Formatter::ksort

## Formatter::queryStringLike

---
title: "香港及海外接入点，如何使用wechatpay-php支付开发包概览"
date: 2021-06-25T11:49:15+08:00
lastmod: 2021-06-27T10:52:06+08:00
keywords: ["微信支付", "WeChatPay PHP SDK", "GuzzleHttp", "offshore usage"]
description: "简述非中国大陆区使用 wechatpay/wechatpay 开发包的step1.2.3."
tags: []
toc: false
categories: [php, wechat]
summary: "简述非中国大陆区使用 wechatpay/wechatpay 开发包的step1.2.3."
author: "James"
---

## 前置条件

已有商户号、商户私钥、商户证书、APIV3密钥，通过此包内置的 平台证书下载器，已取得平台证书。

## 安装软件

`composer require wechatpay/wechatpay`

## WeChatPay\Builder::factory 工厂方法


在工厂方法内，只需传入指定的接入点`base_uri`即可支持，例如：

```php
$instance = \WeChatPay\Builder::factory([
    'base_uri' => 'https://api.mch.weixin.qq.com/hk/',
    'mchid' => '1234567',
    'serial' => '7654321',
    'privateKey' => \WeChatPay\Util\PemUtil::loadPrivateKey('/path/to/apiclient_key.pem'),
    'certs' => [
        '7890123' => \WeChatPay\Util\PemUtil::loadCertificate('/path/to/wechatpay_7890123.pem'),
    ]
]);
```

## 融合钱包-请求下单

```php
$instance->v3->pay->transactions->micropay
->postAsync(['json' => [
    'mchid' => '1234567',
    'appid' => 'wx8888888888888888',
    'out_trade_no' => 'your_out_trade_no',
    'trade_type' => 'MICROPAY',
    'description' => 'QQ Doll',
    'amount' => [
        'total' => 528800,
        'currency' => 'HKD',
    ]
]])
->then(function(\Psr\Http\Message\ResponseInterface $response) {
    // do some successful procedure
})
->otherwise(function(\Psr\Http\Message\ResponseInterface $response) {
    // do some fail procedure
});
```

## 融合钱包-查单

```php
$instance->v3->pay->transactions->transaction_id->'{transaction_id}'
->getAsync([
    'transaction_id' => 'your_achived_transaction_id',
])
->then(function(\Psr\Http\Message\ResponseInterface $response) {
    // do some successful procedure
})
->otherwise(function(\Psr\Http\Message\ResponseInterface $response) {
    // do some fail procedure
});
```

OR

```php
$instance->v3->pay->transactions->outTradeNo->'{out_trade_no}'
->getAsync([
    'out_trade_no' => 'your_out_trade_no',
])
->then(function(\Psr\Http\Message\ResponseInterface) {
    // do some successful procedure
})
->otherwise(function(\Psr\Http\Message\ResponseInterface) {
    // do some fail procedure
});
```

## 融合钱包-撤销订单

```php
$instance->v3->pay->transactions->transaction_id->'{transaction_id}'->reverse
->postAsync([
    'transaction_id' => 'your_achived_transaction_id',
])
->then(function(\Psr\Http\Message\ResponseInterface $response) {
    // do some successful procedure
})
->otherwise(function(\Psr\Http\Message\ResponseInterface $response) {
    // do some fail procedure
});
```

OR

```php
$instance->v3->pay->transactions->outTradeNo->'{out_trade_no}'->reverse
->postAsync([
    'out_trade_no' => 'your_out_trade_no',
])
->then(function(\Psr\Http\Message\ResponseInterface $response) {
    // do some successful procedure
})
->otherwise(function(\Psr\Http\Message\ResponseInterface $response) {
    // do some fail procedure
});
```

## 融合钱包-退款

```php
$instance->v3->refunds->postAsync(['json' => [
    'mchid' => '1234567',
    'appid' => 'wx8888888888888888',
    'out_trade_no' => 'your_out_trade_no',
    'out_refund_no' => 'your_out_refund_no',
    'trade_type' => 'MICROPAY',
    'description' => 'QQ Doll',
    'amount' => [
        'refund' => 528800,
        'total' => 528800,
        'currency' => 'HKD',
    ]
]])
->then(function(\Psr\Http\Message\ResponseInterface $response) {
    // do some successful procedure
})
->otherwise(function(\Psr\Http\Message\ResponseInterface $response) {
    // do some fail procedure
});
```

## 融合钱包-查询单笔退款

```php
$instance->v3->refunds->transaction_id->'{transaction_id}'
->getAsync([
    'transaction_id' => 'your_achived_transaction_id',
])
->then(function(\Psr\Http\Message\ResponseInterface $response) {
    // do some successful procedure
})
->otherwise(function(\Psr\Http\Message\ResponseInterface $response) {
    // do some fail procedure
});
```

OR

```php
$instance->v3->refunds->outTradeNo->'{out_trade_no}'
->getAsync([
    'out_trade_no' => 'your_out_trade_no',
])
->then(function(\Psr\Http\Message\ResponseInterface $response) {
    // do some successful procedure
})
->otherwise(function(\Psr\Http\Message\ResponseInterface $response) {
    // do some fail procedure
});
```

## 融合钱包-查询所有退款

```php
$instance->v3->refunds->getAsync(['query' => [
    'mchid' => '1234567',
    'offset' => 0,
    'count' => 20,
]])
->then(function(\Psr\Http\Message\ResponseInterface $response) {
    // do some successful procedure
})
->otherwise(function(\Psr\Http\Message\ResponseInterface $response) {
    // do some fail procedure
});
```

## 最后

对照着文档，写代码就是如此简单。。。

Repo: https://github.com/TheNorthMemory/wechatpay-php 欢迎 Star .

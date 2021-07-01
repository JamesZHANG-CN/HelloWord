---
title: "微信支付PHP开发对接18讲——从起步到跑顺路"
date: 2021-06-27T16:00:11+08:00
lastmod: 2021-06-29T14:58:50+08:00
keywords: ["微信支付", "WeChatPay PHP SDK", "GuzzleHttp", "PHPStan Level8"]
description: "用时18天，18个文件，2021年6月18日正式发布，带你了解一下「可链式调用的微信支付PHPSDK」研发细节。"
tags: []
toc: false
categories: [php, wechat]
summary: "用时18天，18个文件，2021年6月18日正式发布，带你了解一下「可链式调用的微信支付PHPSDK」研发细节。"
author: "James"
---

用时18天，18个文件，2021年6月18日正式发布，带你了解一下「可链式调用的微信支付PHPSDK」研发细节。

在QQ群、微信群以及微信开发者社区内，看多了对接开发的千奇百怪的问题，也尝试回答了许多开发者的问题，终究敌不过新晋问题的产生，与其这样反反复复地回答几乎相同的问题，不如“造”一把，从易用性上下手，着重把 `wechatpay-guzzle-middle` 官方PHP包给翻新一下。

设定目标： `GuzzleHttp` 升级至 `7` ，引申地，PHP最低版本兼容至 `7.2.5`。 `PHP7` 上许多特性即可用起来，比如 变量类型签名、函数入参及返回值签名，这些均是安全地、受控地使用SDK的必要因素。对于习惯了动态语言编程的开发者来说，限定不是为了限制使用，恰恰相反，新的SDK更具创新性地把链式调用带入了PHP语言中。关于链式可参考 `wechatpay-axios-plugin` 这款nodejs包的使用说明。 PHP上做了部分语言相关适配，会在后期的讲解中展开。

另外，`GuzzleHttp` 带入了 `PromiseA+` 的一个实现，把`异步编程`带入到了PHP语言中， `wechatpapy-php` 把链式及异步揉合在了一起，开发对接写起代码来，可以自由流畅地在顺序代码中使用异步模式编程。其优势即：`resolve`模型对应的是正确逻辑，`reject`即异常逻辑，很自然地就把业务逻辑区分开了，编码效率及健壮性有了很大提升。

链式解决了一个问题就是，代码看起来就像是内置的（其实是动态的），无任何IDE辅助的情况下，程序即可自我解释「我在干什么」，这里简单提一句先，后边再展开讲“链”抽象的切入点。

下面步入正文，一个文件一个文件的讲解「研发细节」。

## 01: Formatter 从格式化参数说起

[Formatter 从格式化参数说起](/post/18-points-of-the-wechatpay-php-openapi-sdk-section01/)

## 02: Crypto\Rsa RSA-OAEP非对称加解密重构

[Crypto\Rsa RSA-OAEP非对称加解密重构](/post/18-points-of-the-wechatpay-php-openapi-sdk-section02/)

## 03: Crypto\AesInterface AES抽象接口

## 04: Crypto\AesGcm AES-GCM加解密函数重构

## 05: ClientDecorator HTTPClient的魔幻装饰器

## 06: ClientDecoratorInterface 抽象装饰器接口

## 07: ClientJsonTrait APIv3抽象可复用代码块

## 08: Builder 工厂模式创建链式组合

## 09: BuilderTrait 可链式复用方法代码块

## 10: bin/CertificateDownloader 重写平台证书下载器

## 11: Util\MediaUtil 媒体文件上传封装文件适配PHP7+

## 12: README.md We're ready for APIv3 developing

## 13: ClientXmlTrait APIv2抽象可复用代码块

## 14: Crypto\Hash Hash散列数据签名

## 15: Crypto\AesEcb AES-ECB加解密

## 16: Transformer Xml2Array2Xml转换器

## 17: Exception\WeChatPayException 异常处理接口

## 18: 跑顺路，就是这么简单

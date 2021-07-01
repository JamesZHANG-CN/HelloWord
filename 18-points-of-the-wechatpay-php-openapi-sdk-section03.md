---
title: "微信支付PHP开发对接18讲——03: AesInterface AES抽象接口"
date: 2021-07-01T19:19:11+08:00
lastmod: 2021-07-01T19:38:26+08:00
keywords: ["微信支付", "WeChatPay PHP SDK", "GuzzleHttp", "PHPStan Level8"]
description: "1. `PHP7` 可以在接口文件中定义公共类常量; 2. 阐述高等级加密`AES`的两个基础知识点; 3. 抽象公共加/解密函数接口，引述程序设计中的`协变`设计规则;"
tags: []
toc: false
categories: [php, wechat]
summary: "`AES` 加解密的块大小，固定是16字节的，提供加密算法常量 `ALGO_AES_256_GCM` 及 `ALGO_AES_256_ECB`，从字面量上给`类实现`提供了可以直接使用的公共加/解密算法。另外3点：1. `PHP7` 可以在接口文件中定义公共类常量; 2. 阐述高等级加密`AES`的两个基础知识点; 3. 抽象公共加/解密函数接口，引述程序设计中的`协变`设计规则;"
author: "James"
---

之所以把这个`接口文件`单独拿出来讲，主要是为了阐明三件事情：

1. `PHP7` 可以在接口文件中定义公共类常量;
2. 阐述高等级加密`AES`的两个基础知识点;
3. 抽象公共加/解密函数接口，引述程序设计中的`协变`设计规则;

`AES` 加解密的块大小，固定是16字节的，遂定义了 `Crypto\AesInterface::BLOCK_SIZE` 这个常量；`GCM`模式的认证标签`auth_tag`，在`OpenSSL`中的可能值是`16, 15, 14, 13, 12, 8 or 4`的其中一个，官方文档在阐述敏感信息加解密的时候，定义了`AUTH_TAG_LENGTH_BYTE`常量，严格意义上来说，这个值应该是个数组，只是在实现过程中，取了最大值，即等于`AES`的块大小，这个在程序代码中标记成`即将废弃`功能（因为不准确），预计在下一个大版本上会移除。

接口上同时定义了两个加密算法常量 `ALGO_AES_256_GCM` 及 `ALGO_AES_256_ECB`，从字面量上给`类实现`提供了可以直接使用的公共加/解密算法。

接口同时定义了抽象公共方法 加密：`encrypt(string $plaintext, string $key, string $iv = ''): string;` 及解密： `decrypt(string $ciphertext, string $key, string $iv = ''): string;` 的参数名称/顺序及类型，返回值类型，给`类实现`圈定基础参数。

这个接口类就没啥说的类，下一讲，我们来说说APIv3的敏感信息加解密`AES-GCM`实现。

---
title: "这很可能是NodeJS中用来开发微信支付APIv3的顶级SDK之一"
date: 2020-06-19T11:05:37+08:00
keywords: ["微信支付V3", "Wechatpay APIv3 SDK", "Axios Plugin"]
description: "Most probably one of the topest SDK in the NodeJS for developing Wechatpay APIv3. 使用NodeJS原生方法完整实现了微信支付APIv3的请求应答工作，HTTP客户端采用成熟的Axios，通过向Axios注册拦截器完整实现微信支付APIv3上行数据签名，下行数据验签。包括收单、媒体文件上传、发核券以及账单下载解析功能，同时提供官方应答证书命令行下载工具。"
tags: []
categories: ["wechat"]
summary: "微信支付APIv3使用了许多成熟且牛逼的接口设计（RESTful API with JSON over HTTP），数据交换使用非对称（RSA）加/解密方案，上行数据采用（RSA）私钥证书签名，下行数据采用（RSA）公钥证书验签。本开发包使用NodeJS原生方法完整实现了微信支付APIv3的请求应答工作，HTTP客户端采用成熟的Axios，通过向Axios注册拦截器完整实现微信支付APIv3上行数据签名，下行数据验签。包括收单、媒体文件上传、发核券以及账单下载解析功能，同时提供官方应答证书命令行下载工具。"
author: "James"
---

## 缘起

微信支付官方提供了两个包，一个是`Java`版的，另一个是`PHP`版的。继上一篇无侵入地向官方`PHP`版提供了媒体文件上传功能之后，我就在想，`NodeJS`都已经到14.4版本了，应该有一个流行的js版的开发包吧。搜了一圈儿，发现几乎没有，遂动手花了大约两周闲暇时间，完整实现了微信支付APIv3的几乎所有功能，特别地对账单下载及处理，也算是填补并加强了官方包暂未实现的功能之一吧。有了这些成熟开发包的覆盖，包括官方包在内，现在是时候全面切换至微信支付APIv3版了。

项目以MIT许可证开源，使用请保留许可证说明。

## 主要功能

- [x] 使用Node原生代码实现微信支付APIv3的AES加/解密功能(`aes-256-gcm` with `aad`)
- [x] 使用Node原生代码实现微信支付APIv3的RSA加/解密、签名、验签功能(`sha256WithRSAEncryption` with `RSA_PKCS1_OAEP_PADDING`)
- [x] 大部分微信支付APIv3的HTTP GET/POST应该能够正常工作，依赖 `Axios`
- [x] 支持微信支付APIv3的媒体文件上传(图片/视频)功能，可选依赖 `form-data`
- [x] 支持微信支付APIv3的应答证书下载功能，依赖 `commander`, 使用手册如下
- [x] 支持微信支付APIv3的帐单下载及解析功能

## 安装

`$ npm install wechatpay-axios-plugin`

## 系统要求

`NodeJS`的原生`crypto`模块，自v12.9.0在 `publicEncrypt` 及 `privateDecrypt` 增加了 `oaepHash` 入参选项，本项目显式声明入参，本人不确定其在v12.9.0以下是否正常工作。所以Node的最低版本要求应该是v12.9.0.

## 万里长征第一步

微信支付APIv3使用了许多成熟且牛逼的接口设计（RESTful API with JSON over HTTP），数据交换使用非对称（RSA）加/解密方案，对上行数据要求（RSA）签名，对下行数据要求（RSA）验签。API上行所需的`商户RSA私钥证书`，可以由商户的`超级管理员`在`微信支付商户平台`生成并获取到，然而，API下行所需的`平台RSA公共证书`只能从`/v3/certificates`接口获取（应答证书还经过了AES对称加密，得用`APIv3密钥`才能解密 :+1: ）。本项目也提供了命令行下载工具，使用手册如下：

`$ ./bin/certificateDownloader -h`

```
Usage: certificateDownloader [options]

Options:
  -V, --version              output the version number
  -m, --mchid <string>       The merchant's ID, aka mchid.
  -s, --serialno <string>    The serial number of the merchant's public certificate aka serialno.
  -f, --privatekey <string>  The path of the merchant's private key certificate aka privatekey.
  -k, --key <string>         The secret key string of the merchant's APIv3 aka key.
  -o, --output [string]      Path to output the downloaded wechatpay's public certificate(s) (default: "/tmp")
  -h, --help                 display help for command
```

**注：** 像其他通用命令行工具一样，`-h` `--help` 均会打印出帮助手册，说明档里的`<string>`指 必选参数，类型是字符串； `[string]`指 可选字符串参数，默认值是 `/temp`（系统默认临时目录）


`$ ./bin/certificateDownloader -m NUMERICAL -s HEXADECIAL -f apiclient_key.pem -k YOURAPIV3SECRETKEY -o .`

```
Wechatpay Public Certificate#0
  serial=HEXADECIALHEXADECIALHEXADECIAL
  notBefore=Wed, 22 Apr 2020 01:43:19 GMT
  notAfter=Mon, 21 Apr 2025 01:43:19 GMT
  Saved to: wechatpay_HEXADECIALHEXADECIALHEXADECIAL.pem
You should verify the above infos again even if this library already did(by rsa.verify):
    openssl x509 -in wechatpay_HEXADECIALHEXADECIALHEXADECIAL.pem -noout -serial -dates

```
**注：** 提供必选参数且运行后，屏幕即打印出如上信息，提示`证书序列号`及`起、止格林威治(GMT)时间`以及证书下载保存位置。

接口通讯要用`商户RSA私钥证书`签名及`平台RSA公共证书`验签，只有获取到了`平台RSA公共证书`，后续的其他接口才能正常应答验签，所谓“万里长征第一步”就在这里。本下载工具也无例外对应答内容，做了验签处理，技法“剑(qí)走(zhāo)偏(yín)锋(jì)“而已，即：用Axios的拦截器把下载的证书(AES解密)处理完后，立即用于验签。

得到证书之后，开发者需要把所得`serial`及`wechatpay_HEXADECIALHEXADECIALHEXADECIAL.pem`（文件流或者文本内容）组成 {key:value} 对，key为证书序列号，value为证书内容，传入以下的构造函数的`certs`字段里。 其他接口使用就基本上没有啥问题了。

文档及示例在github及npm上README都有，也都是基本用法，没啥花活儿，祝开心。 :smile:

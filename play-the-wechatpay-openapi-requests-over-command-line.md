---
title: "真香：一行命令即可体验「微信支付」全系接口能力"
date: 2021-03-28T19:59:13+08:00
keywords: [wechatpay openapi cli 微信支付接口CLI工具]
description: "Play the WeChatPay OpenAPI Requests Over Command Line，以命令行方式与OpenAPI网关交互——花式快速体验支付开放能力"
tags: []
toc: false
categories: [javascript, wechat]
summary: "以命令行方式与微信支付接口交互，旨在提供一种简洁高效的方式，让开发者可以快速对接，希望能对开发有所帮助。"
author: "James"
---

你没看错，这款唯二的「Play the OpenAPI requests over command line」——「以命令行方式与微信支付接口交互」姗姗来了，旨在提供一种简洁高效的方式，让开发者可以快速对接，希望能对开发有所帮助。

nodejs版的 `wechatpay-axios-plugin` 已经进入`v0.5`版，在这一版上，有如下改变：

- 全部代码（包括265条测试用例）使用 `eslint-config-airbnb-base` 代码风格校验；
- 使用 `yargs` 包替换了 `commander`，重构了「平台证书下载器」工具，使其降级为此命令行一个特殊方法；
- 降级 `form-data` 及 `yargs` 包为 `peerDependencies`，没有这俩包，90%+ 的接口是可以正常工作的，缩减依赖；
- 新增 `bin/cli.js`，开发可以仅用一条命令就能跑得欢实不要不要的了；

## 使用

开始使用前，需要开发者自行安装`cli`模式依赖的npm包，即 `npm i yargs`

### 帮助手册

`./node_modules/.bin/wxpay --help`

```
wxpay <command>

Commands:
  wxpay crt        The WeChatPay APIv3s Certificate Downloader
  wxpay req <uri>  Play the OpenAPI requests over command line

Options:
      --version  Show version number  [boolean]
      --help     Show help  [boolean]
  -u, --baseURL  The baseURL  [string] [default: "https://api.mch.weixin.qq.com/"]

for more information visit "https://github.com/TheNorthMemory/wechatpay-axios-plugin"
```

`./node_modules/.bin/wxpay crt --help`

```
wxpay crt

The WeChatPay APIv3s Certificate Downloader

cert
  -m, --mchid       The merchants ID.  [string] [required]
  -s, --serialno    The serial number.  [string] [required]
  -f, --privatekey  Path of the merchants private key certificate.  [string] [required]
  -k, --key         The secret key string of the merchants APIv3.  [string] [required]
  -o, --output      Path to output the platform certificate(s)  [string] [default: "/tmp"]

Options:
      --version  Show version number  [boolean]
      --help     Show help  [boolean]
  -u, --baseURL  The baseURL  [string] [default: "https://api.mch.weixin.qq.com/"]
```


`./node_modules/.bin/wxpay req --help`

```
wxpay req <uri>

Play the WeChatPay OpenAPI requests over command line

request <uri>
  -c, --config   The configuration  [required]
  -b, --binary   Point out the response as `arraybuffer`  [boolean]
  -m, --method   The request HTTP verb  [default: "POST"]
  -h, --headers  Special request HTTP header(s)
  -d, --data     The request HTTP body
  -p, --params   The request HTTP query parameter(s)

Options:
      --version  Show version number  [boolean]
      --help     Show help  [boolean]
  -u, --baseURL  The baseURL  [string] [default: "https://api.mch.weixin.qq.com/"]
```

### 证书下载

`wxpay crt -m N -s S -f F.pem -k K -o .`

```
The WeChatPay Platform Certificate#0
  serial=HEXADECIAL
  notBefore=Wed, 22 Apr 2020 01:43:19 GMT
  notAfter=Mon, 21 Apr 2025 01:43:19 GMT
  Saved to: wechatpay_HEXADECIAL.pem
You may confirm the above infos again even if this library already did(by Rsa.verify):
    openssl x509 -in wechatpay_HEXADECIAL.pem -noout -serial -dates
```

### v3版Native下单

```
wxpay req v3/pay/transactions/native \
  -c.mchid 1230000109 \
  -c.serial HEXADECIAL \
  -c.privateKey /path/your/merchant/mchid.key \
  -c.certs.HEXADECIAL /path/the/platform/certificates/HEXADECIAL.pem \
  -d.appid wxd678efh567hg6787 \
  -d.mchid 1230000109 \
  -d.description 'Image形象店-深圳腾大-QQ公仔' \
  -d.out_trade_no '1217752501201407033233368018' \
  -d.notify_url 'https://www.weixin.qq.com/wxpay/pay.php' \
  -d.amount.total 100 \
  -d.amount.currency CNY
```

### v3版查询订单

```
wxpay req v3/pay/transactions/id/1217752501201407033233368018 \
  -c.mchid 1230000109 \
  -c.serial HEXADECIAL \
  -c.privateKey /path/your/merchant/mchid.key \
  -c.certs.HEXADECIAL /path/the/platform/certificates/HEXADECIAL.pem \
  -m get \
  -p.mchid 1230000109
```

### v3版关闭订单

```
wxpay req v3/pay/transactions/out-trade-no/1217752501201407033233368018 \
  -c.mchid 1230000109 \
  -c.serial HEXADECIAL \
  -c.privateKey /path/your/merchant/mchid.key \
  -c.certs.HEXADECIAL /path/the/platform/certificates/HEXADECIAL.pem \
  -d.mchid 1230000109
```

### v3版申请对账单

```
wxpay req v3/bill/tradebill \
  -c.mchid 1230000109 \
  -c.serial HEXADECIAL \
  -c.privateKey /path/your/merchant/mchid.key \
  -c.certs.HEXADECIAL /path/the/platform/certificates/HEXADECIAL.pem \
  -m get \
  -p.bill_date '2021-02-12' \
  -p.bill_type 'ALL'
```

### v2版付款码付

```
wxpay req v2/pay/micropay \
  -c.mchid 1230000109 \
  -c.serial any \
  -c.privateKey any \
  -c.certs.any \
  -c.secret your_merchant_secret_key_string \
  -d.appid wxd678efh567hg6787 \
  -d.mch_id 1230000109 \
  -d.device_info 013467007045764 \
  -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS \
  -d.detail 'Image形象店-深圳腾大-QQ公仔' \
  -d.spbill_create_ip 8.8.8.8 \
  -d.out_trade_no '1217752501201407033233368018' \
  -d.total_fee 100 \
  -d.fee_type CNY \
  -d.auth_code 120061098828009406
```

`auth_code` 输入配合扫码枪，体验就飞起来了～

### v2版付款码查询openid

```
wxpay req v2/tools/authcodetoopenid \
  -c.mchid 1230000109 \
  -c.serial any \
  -c.privateKey any \
  -c.certs.any \
  -c.secret your_merchant_secret_key_string \
  -d.appid wxd678efh567hg6787 \
  -d.mch_id 1230000109 \
  -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS \
  -d.auth_code 120061098828009406
```

## 设计

主程序 `bin/cli.js` 上，加入了一个中间件，代码如下：

```javascript
.middleware((argv) => {
  if (argv.c && argv.c.privateKey && argv.c.privateKey !== 'any') {
    /* eslint-disable-next-line no-param-reassign */
    argv.config.privateKey = readFileSync(argv.c.privateKey);
  }
  if (argv.c && argv.c.certs && Object.keys(argv.c.certs)[0] !== 'any') {
    /* eslint-disable-next-line no-return-assign, no-param-reassign, no-sequences */
    argv.config.certs = Object.entries(argv.config.certs).reduce((o, [k, v]) => (o[k] = readFileSync(v), o), {});
  }
  if (argv.c && argv.c.merchant) {
    if (argv.c.merchant.cert && argv.c.merchant.cert !== 'any') {
      /* eslint-disable-next-line no-param-reassign */
      argv.config.merchant.cert = readFileSync(argv.c.merchant.cert);
    }
    if (argv.c.merchant.key && argv.c.merchant.key !== 'any') {
      /* eslint-disable-next-line no-param-reassign */
      argv.config.merchant.key = readFileSync(argv.c.merchant.key);
    }
    if (argv.c.merchant.pfx && argv.c.merchant.pfx !== 'any') {
      /* eslint-disable-next-line no-param-reassign */
      argv.config.merchant.pfx = readFileSync(argv.c.merchant.pfx);
    }
  }
}, true)
```

其作用就是，把sdk所需的证书，从文件形式自动加载，供后续 `cli/request.js`方法直接使用，`req`方法核心代码也就10来行！

```javascript
handler(argv) {
  const {
    baseURL, uri, config, method, data, params, headers,
  } = argv;
  const responseType = argv.binary ? 'arraybuffer' : undefined;
  const structure = [{ params, headers, responseType }];

  if (data) { structure.unshift(data); }

  (new Wechatpay({ baseURL, ...config }))[uri][method](...structure)
    /* eslint-disable-next-line no-console */
    .then(console.info).catch(console.error);
},
```

代码以 MIT 开放源码公开在 https://github.com/TheNorthMemory/wechatpay-axios-plugin ，如果喜欢，就来个 **Star** 。

## 后记

Q: 哇！这么简单？

A: 不就是为了简单么。。。

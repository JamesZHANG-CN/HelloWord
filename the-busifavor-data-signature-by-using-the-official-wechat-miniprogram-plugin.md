---
title: "微信支付官方「小程序发券插件」(≤1.1.3) 数据签名概要"
date: 2020-07-21T13:09:13+08:00
keywords: ["微信支付", "官方小程序发券插件发券", "数据签名"]
description: "The Busifavor Data Signature by Using the Official Wechat Miniprogram Plugin 微信支付官方小程序发券插件的商家券数据签名，单独抽离出数据转换函数，配合使用云开发NodeJS环境，让签名不再麻烦。"
tags: []
categories: ["wechat"]
summary: "微信支付官方「小程序发券插件」(≤1.1.3) 的商家券数据签名，单独抽离出数据转换函数，配合使用云开发NodeJS环境，让签名不再麻烦。"
author: "James"
---

先来看看「[小程序发券插件](https://pay.weixin.qq.com/wiki/doc/apiv3/wxpay/marketing/miniprogram/chapter3_1.shtml)」所需参数，yaml格式如下：

```yaml
type: object
required:
  - send_coupon_params
  - sign
  - send_coupon_merchant
properties:
  bindcustomevent:
    type: string
    description: 自定义事件
  send_coupon_params:
    type: array
    description: 发券参数
    items:
      type: object
      required:
        - stock_id
        - out_request_no
      properties:
        stock_id:
          type: string
          description: 批次号
          example: abc123
        out_request_no:
          type: string
          description: 发券凭证
          example: '1234567'
  sign:
    type: string
    description: 签名
    example: 9A0A8659F005D6984697E2CA0A9CF3B79A0A8659F005D6984697E2CA0A9CF3B7
  send_coupon_merchant:
    type: string
    description: 发券商户号
    example: '10016226'
```

转换成数据结构即：

```js
{
  bindcustomevent: 'getcoupon',
  send_coupon_params: [
    {out_request_no:'1234567',stock_id:'abc123'}
  ],
  send_coupon_merchant: '10016226'
}
```

签名时，文档说明是需要把上述参数，以`key`下标从0开始拉平成1维对象，即：

```js
{
  out_request_no0: '1234567',
  stock_id0: 'abc123',
  send_coupon_merchant: '10016226'
}
```

参数`key`下标以`index`记录了顺序，最大支持十张券刚好是0-9，有点儿 `Array.prototype.flatMap()` 的味道，但不完全是。

转换函数如下:

```js
// 可单独抽离成模块
exports.busiFavorFlat = ({send_coupon_params, send_coupon_merchant = []} = {}) => {
  return {
    send_coupon_merchant,
    ...(send_coupon_params || []).reduce((des, row, idx) => (Object.keys(row).map(one => des[`${one}${idx}`] = row[one]), des), {}),
  }
}
```

使用如下:

```js
const responder = {
  send_coupon_params: [
    {out_request_no:'1234567',stock_id:'abc123'},
    {out_request_no:'7654321',stock_id:'321cba'},
  ],
  send_coupon_merchant: '10016226'
}

const waitToSign = busiFavorFlat(responder)
```

以 [v2版nodejs签名代码为例](https://developers.weixin.qq.com/community/develop/doc/000c6040f00c0088deaab14cb5bc00?pass_ticket=SM4hJgkhUua61lLujzVuWtkblmILCKb2NEN4JxNpy8vfixixqolzA7VI+2PaAZ48&jumpto=comment&commentid=0000407aa40280d9dfaa2503d5b8)，把以上输出，代入签名，大功告成，如下～

```js
// let key = 'exposed_your_key_here_have_risks'
responder.sign = signHmacSha256(toSignString(ksort(waitToSign), key), key)
// console.info(responder.sign)
// 20FB971D442A119C85CA1A49DBDEA61A14944D98AFC1D2B040030C5C8792A83A
// 签名及数据结构，以JSON串发送给小程序端
// server.response.toMiniProgram(JSON.stringify(responder))
```

文章首发于[微信开发者社区](https://developers.weixin.qq.com/community/develop/article/doc/000cae6e244d38bbeeaa32dd25bc13)。

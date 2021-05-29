---
title: "免充值产品测试验收用例的wechatpay-axios-plugin教学帖"
date: 2021-04-21T21:13:20+08:00
lastmod: 2021-05-10T18:54:06+08:00
keywords: [微信支付, 免充值产品, 测试验收, wechatpay-axios-plugin]
description: "微信开发者社区内大佬们都有贡献 「免充值代金券」 的测试用例实现，这篇文章也来凑一下热闹，顺带帮你把一下为啥要做这个 「测试验收」 以及 「验收注意」 细节。"
tags: []
toc: false
categories: [javascript, wechat]
summary: "微信开发者社区内大佬们都有贡献 「免充值代金券」 的测试用例实现，这篇文章也来凑一下热闹，顺带帮你把一下为啥要做这个 「测试验收」 以及 「验收注意」 细节。"
author: "James"
---

社区内大佬们都有贡献 **「免充值代金券」** 的测试用例实现，这篇文章也来凑一下热闹，顺带帮你把一下为啥要做这个 **「测试验收」** 以及 **「验收注意」** 细节。

## 引言

[官方用例文档链接](https://pay.weixin.qq.com/wiki/doc/api/download/mczyscsyl.pdf) 一共25页，非常详细，照着描一步步做，很快就能验收完。本篇作为「教学帖」，试着让友商们理解，这个验收的重要性，如果希望能够获取帮助直接验收，请阅读[代金券接口升级验收脚本 用例组合1001+1002+1003+1004+1005](https://developers.weixin.qq.com/community/pay/article/doc/0002e82b060c3028230c915f150813)。

## 安装nodejs sdk

本篇以 `wechatpay-axios-plugin` 这款开发包展开，详细介绍可阅读 [从APIv3到APIv2再到企业微信，这款微信支付开发包的README你应该来读一读](https://developers.weixin.qq.com/community/develop/article/doc/000ca44ae3cff894e9fbb46ba5b413)。

```js
npm install wechatpay-axios-plugin@"v0.5.5"
```

v0.6系列做了`返回数据签名强校验`，以下示例代码需要做特殊处理，本篇以v0.5.5展开。

## 获取沙箱密钥

先要理解，沙箱环境是个仿真环境，不是生产环境，友商朋友们应该做环境隔离。起步需要商户使用生产环境的 `API秘钥`，去获取`沙箱秘钥`，后续所有`沙箱环境`操作都要使用由`沙箱秘钥`生成的`数据签名sign`。

```js
const { Wechatpay, Formatter } = require('wechatpay-axios-plugin');

const mchid = '你的商户号';
const secret = '你的32字节的`API密钥`字符串';
const appid = '你的APPID字符串';

const mch_id = mchid;
const noop = {serial: 'any', privateKey: 'any', certs: {any: undefined}};

// 实例化一个对象
let wxpay = new Wechatpay({ secret, mchid, ...noop });

const { data: { sandbox_signkey } } = await wxpay.v2.sandboxnew.pay.getsignkey({ mch_id, nonce_str: Formatter.nonce() });
```

## 1001 付款码(刷卡)支付

订单金额 `5.01` 元，其中 `0.01` 元使用免充值券，用户实际支付 `5.00` 元。验证商户具备正确解析及识别免充值代金券字段的能力。

### 1001.1 请求支付

```js
// 重新实例化一个沙箱环境的对象
wxpay = new Wechatpay({ secret: sandbox_signkey, mchid, ...noop });

// 模拟一个商户订单号
let out_trade_no = `SD${+new Date()}_501`;

const { data: {
  coupon_fee,
  settlement_total_fee,
  total_fee
} } = await wxpay.v2.sandboxnew.pay.micropay({
    appid, mch_id, nonce_str: Formatter.nonce(), out_trade_no,
    body: 'dummybot',
    total_fee: 501,
    spbill_create_ip: '127.0.0.1',
    auth_code: '120061098828009406'
});

console.table({
    代金券金额: coupon_fee,
    应结订单金额: settlement_total_fee,
    订单金额: total_fee
});
```

打印日志应如下：

```
┌──────────────┬────────┐
│   (index)    │ Values │
├──────────────┼────────┤
│  代金券金额    │  '1'   │
│ 应结订单金额   │ '500'  │
│   订单金额    │ '501'  │
└──────────────┴────────┘
```

### 1001.2 获取支付结果

```js
const { data: {
  settlement_total_fee,
  total_fee, coupon_fee,
  coupon_fee_0,
  coupon_type_0,
  coupon_count,
} } = await wxpay.v2.sandboxnew.pay.orderquery({
    appid, mch_id, nonce_str: Formatter.nonce(), out_trade_no,
});

console.table({
    代金券金额: coupon_fee,
    应结订单金额: settlement_total_fee,
    订单金额: total_fee,
    单个代金券支付金额: coupon_fee_0,
    代金券类型: coupon_type_0,
    代金券使用数量: coupon_count
});
```

打印日志应如下：

```
┌────────────────────┬───────────┐
│      (index)       │  Values   │
├────────────────────┼───────────┤
│     代金券金额       │    '1'    │
│    应结订单金额      │   '500'   │
│      订单金额       │   '501'   │
│ 单个代金券支付金额    │    '1'    │
│     代金券类型       │ 'NO_CACH' │
│   代金券使用数量     │    '1'    │
└────────────────────┴───────────┘
```

## 1002 付款码(刷卡)支付退款

订单金额 `5.02` 元，其中 `0.01` 元使用免充值代金劵，实际支付 `5.01` 元，退款查询升级。

### 1002.1 请求支付

```js
//模拟重置一个商户订单号
out_trade_no = `SD${+new Date()}_502`;

const { data: {
  coupon_fee,
  settlement_total_fee,
  total_fee,
} } = await wxpay.v2.sandboxnew.pay.micropay({
    appid, mch_id, nonce_str: Formatter.nonce(), out_trade_no,
    body: 'dummybot',
    total_fee: 502,
    spbill_create_ip: '127.0.0.1',
    auth_code: '120061098828009406'
});

console.table({
  代金券金额: coupon_fee,
  应结订单金额: settlement_total_fee,
  订单金额: total_fee
});
```

打印日志应如下：

```
┌──────────────┬────────┐
│   (index)    │ Values │
├──────────────┼────────┤
│  代金券金额   │  '1'   │
│ 应结订单金额   │ '501'  │
│   订单金额    │ '502'  │
└──────────────┴────────┘
```

### 1002.2 获取支付结果

```js
const { data: {
  settlement_total_fee,
  total_fee,
  coupon_fee,
  coupon_fee_0,
  coupon_type_0,
  coupon_count
} } = await wxpay.v2.sandboxnew.pay.orderquery({
    appid, mch_id, nonce_str: Formatter.nonce(), out_trade_no,
});

console.table({
    商户订单号: out_trade_no,
    代金券金额: coupon_fee,
    应结订单金额: settlement_total_fee,
    订单金额: total_fee,
    单个代金券支付金额: coupon_fee_0,
    代金券类型: coupon_type_0,
    代金券使用数量: coupon_count
});
```

打印日志应如下：

```
┌────────────────────┬───────────────────────┐
│      (index)       │        Values         │
├────────────────────┼───────────────────────┤
│     商户订单号     │ 'SD1618966329677_502' │
│     代金券金额     │          '1'          │
│    应结订单金额    │         '501'         │
│      订单金额      │         '502'         │
│ 单个代金券支付金额 │          '1'          │
│     代金券类型     │       'NO_CACH'       │
│   代金券使用数量   │          '1'          │
└────────────────────┴───────────────────────┘
```

### 1002.3 请求退款

```js
const { data: {
  cash_refund_fee,
  cash_fee,
  refund_fee,
  total_fee
} } = await wxpay.v2.sandboxnew.pay.refund({
    appid, mch_id, nonce_str: Formatter.nonce(), out_trade_no,
    out_refund_no: `RD${out_trade_no}`,
    total_fee: 502,
    refund_fee: 501,
});

console.table({
    退款金额: refund_fee,
    标价金额: total_fee,
    现金支付金额: cash_fee,
    现金退款金额: cash_refund_fee,
});
```

打印日志应如下：

```
┌────────────────────┬─────────────┐
│      (index)       │   Values    │
├────────────────────┼─────────────┤
│     退款金额        │   '502'     │
│     标价金额        │    '502'    │
│    现金支付金额      │    '501'    │
│    现金退款金额      │    '501'    │
└────────────────────┴─────────────┘
```

### 1002.4 获取退款结果

```js
const { data: {
    settlement_total_fee,
    total_fee,
    cash_fee,
    settlement_refund_fee,
    coupon_refund_fee_0,
    coupon_type_0_0,
    coupon_refund_fee_0_0,
    refund_fee_0,
    coupon_refund_count_0,
} } = await wxpay.v2.sandboxnew.pay.refundquery({
    appid, mch_id, out_trade_no, nonce_str: Formatter.nonce(),
});

console.table({
    应结订单金额: settlement_total_fee,
    订单金额: total_fee,
    现金支付金额: cash_fee,
    退款金额: settlement_refund_fee,
    总代金券退款金额: coupon_refund_fee_0,
    代金券类型: coupon_type_0_0,
    总代金券退款金额: coupon_refund_fee_0_0,
    申请退款金额: refund_fee_0,
    退款代金券使用数量: coupon_refund_count_0,
});
```

## 1003 JSAPI/APP/Native支付

订单金额 `5.51` 元，其中 `0.01` 元使用免充值券，实际支付 `5.50` 元。 验证正常支付流程，商户使用免充值代金券支付。

### 1003.1 统一下单

```js
//模拟重置一个商户订单号
out_trade_no = `SD${+new Date()}_551`;
const {data: { prepay_id } } = await wxpay.v2.sandboxnew.pay.unifiedorder({
  appid, mch_id, out_trade_no, nonce_str: Formatter.nonce(),
  body: 'dummybot',
  total_fee: 551,
  notify_url: 'https://www.weixin.qq.com/wxpay/pay.php',
  spbill_create_ip: '127.0.0.1',
  trade_type: 'JSAPI'
});

console.table({ 预支付交易会话标识: prepay_id });
```

### 1003.2 获取支付结果

```js
const { data: {
  out_trade_no,
  total_fee,
  cash_fee,
  coupon_fee,
  coupon_count,
  coupon_fee_0,
  coupon_type_0,
  coupon_fee_0
} } = wxpay.v2.sandboxnew.pay.orderquery({
    appid, mch_id, out_trade_no, nonce_str: Formatter.nonce(),
});

console.table({
  out_trade_no,
  total_fee,
  cash_fee,
  coupon_fee,
  coupon_count,
  coupon_fee_0,
  coupon_type_0,
  coupon_fee_0
});
```

## 1004 JSAPI/APP/Native支付退款

订单金额 `5.52` 元，其中 `0.01` 元使用免充值券，实际支付 `5.51` 元。

### 1004.1 统一下单

```js
//模拟重置一个商户订单号
out_trade_no = `SD${+new Date()}_551`;
const {data: { prepay_id } } = await wxpay.v2.sandboxnew.pay.unifiedorder({
  appid, mch_id, out_trade_no, nonce_str: Formatter.nonce(),
  body: 'dummybot',
  total_fee: 552,
  notify_url: 'https://www.weixin.qq.com/wxpay/pay.php',
  spbill_create_ip: '127.0.0.1',
  trade_type: 'JSAPI'
});

console.table({ 预支付交易会话标识: prepay_id });
```

### 1004.2 获取支付结果

```js
const { data: {
  out_trade_no,
  total_fee,
  cash_fee,
  coupon_fee,
  coupon_count,
  coupon_fee_0,
  coupon_type_0,
  coupon_fee_0
} } = wxpay.v2.sandboxnew.pay.orderquery({
    appid, mch_id, out_trade_no, nonce_str: Formatter.nonce(),
});

console.table({
  out_trade_no,
  total_fee,
  cash_fee,
  coupon_fee,
  coupon_count,
  coupon_fee_0,
  coupon_type_0,
  coupon_fee_0
});
```

### 1004.3 请求退款

```js
const { data: { cash_refund_fee, cash_fee, refund_fee, total_fee } } = await wxpay.v2.sandboxnew.pay.refund({
    appid, mch_id, nonce_str: Formatter.nonce(), out_trade_no,
    out_refund_no: `RD${out_trade_no}`,
    total_fee: 552,
    refund_fee: 551,
});

console.table({
    退款金额: refund_fee, 标价金额: total_fee, 现金支付金额: cash_fee, 现金退款金额: cash_refund_fee,
});
```

打印日志应如下：

```
┌─────────────────┬──────────┐
│      (index)    │  Values  │
├─────────────────┼──────────┤
│     退款金额     │  '552'   │
│     标价金额     │   '552'  │
│    现金支付金额   │  '551'  │
│    现金退款金额   │  '551'  │
└─────────────────┴──────────┘
```

### 1004.4 获取退款结果

```js
const { data: {
    settlement_total_fee,
    total_fee,
    cash_fee,
    settlement_refund_fee,
    coupon_refund_fee_0,
    coupon_type_0_0,
    coupon_refund_fee_0_0,
    refund_fee_0,
    coupon_refund_count_0,
} } = await wxpay.v2.sandboxnew.pay.refundquery({
    appid, mch_id, out_trade_no, nonce_str: Formatter.nonce(),
});

console.table({
    应结订单金额: settlement_total_fee
    订单金额: total_fee
    现金支付金额: cash_fee,
    退款金额: settlement_refund_fee
    总代金券退款金额: coupon_refund_fee_0
    代金券类型: coupon_type_0_0,
    总代金券退款金额: coupon_refund_fee_0_0
    申请退款金额: refund_fee_0
    退款代金券使用数量: coupon_refund_count_0,
});
```

## 1005 交易对账单下载

使用了免充值券的订单，免充值券部分的金额不计入结算金额。验证商户对账能正确理解到这一点，对账无误。这里预期会返回 `1269` 条明细数据。

汇总结果：总交易单数,应结订单总金额,退款总金额,充值券退款总金额,手续费总金额,订单总金额,申请退款金额。

这里数据应为：

```
1269, `10.79, `5.93, `0.24, `0.0,`11.27, `6.37
```

```js
const { data } = await wxpay.v2.sandboxnew.pay.downloadbill({
  appid,
  mch_id,
  bill_type: 'ALL',
  bill_date: (new Date(+new Date() + (8 - 48)*3600*1000)).toISOString().slice(0, 10),
  nonce_str: Formatter.nonce(),
}, { transformResponse: [function csvCastor(data) { return data.toString() }] });

console.table(data);
```

打印日志形如：

```
交易时间,公众账号ID,商户号,子商户号,设备号,微信订单号,商户订单号,用户标识,交易类型,交易状态,付款银行,货币种类,应结订单金额,代金券金额,微信退款单号,商户退款单号,退款金额,充值券退款金额,退款类型,退款状态,商品名称,商户数据包,手续费,费率,订单金额,申请退款金额,费率备注
`2016-05-04  02:18:18,`wxf7c30a8258df4208,`10014843,`0,`harryma007,`4.00123E+27,`autotest_20160501030456_45023,`oT2kauIMXH398DZBeJ4m22CuSDQ0,`NATIVE,`REFUND,`PAB_DEBIT,`CNY,`0,`0,`2.00123E+27,`REF4001232001201605015390231647,`0.01,`0,`ORIGINAL,`PROCESSING,`body中文测试,`attach中文测试,`0,`0.60%,`0,`0.01,`
`2016-05-04  02:18:18,`wxf7c30a8258df4208,`10014843,`0,`harryma007,`4.00123E+27,`autotest_20160501060418_79156,`oT2kauIMXH398DZBeJ4m22CuSDQ0,`NATIVE,`REFUND,`PAB_DEBIT,`CNY,`0,`0,`2.00123E+27,`REF4001232001201605015391766944,`0.01,`0,`ORIGINAL,`PROCESSING,`body中文测试,`attach中文测试,`0,`0.60%,`0,`0.01,`
...
14:51:51,`wxf7c30a8258df4208,`10014843,`0,`harryma8888,`4.00968E+27,`wxautotest1462344441,`oT2kauGtJag902bjdvevrJbpGuxo,`NATIVE,`SUCCESS,`CMBC_CREDIT,`CNY,`0.05,`0.01,`0,`0,`0,`0,`中文[body],`测试中文[attach],`0,`0.60%,`0.05,`0,`
总交易单数,应结订单总金额,退款总金额,充值券退款总金额,手续费总金额,订单总金额,申请退款总金额
1269,`10.79,`5.93,`0.24,`0,`11.27,`6.37
```

##	一铲到底

以下是一键验收全流程的一个`魔性 chain`到底的实现，感谢您阅读至此。本文如果对你开通 **「免充代金券」** 功能有帮助，那就来个赞呗。

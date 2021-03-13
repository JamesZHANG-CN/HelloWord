---
title: "再造一遍微信支付v2版的nodejs版SDK"
date: 2020-09-12T15:31:43+08:00
keywords: ["微信支付", "Wechatpay SDK", "Axios Plugin"]
description: "在2020这个时间节点，之所以还要再造一遍微信支付v2(相对于APIv3来说)的SDK轮子，实属是无奈之举，线下交易场景常见的付款码支付及退款功能，官方当下还没有开放出来v3版的，只能借助v2接口来处理； wechatpay-axios-plugin 从一开始目标就是为云原生而设计，遂再造一遍轮子，也抽出一些共生方法，为v3而用。"
tags: []
categories: [javascript, wechat]
summary: "在2020这个时间节点，之所以还要再造一遍微信支付v2(相对于APIv3来说)的SDK轮子，实属是无奈之举，线下交易场景常见的付款码支付及退款功能，官方当下还没有开放出来v3版的，只能借助v2接口来处理； wechatpay-axios-plugin 从一开始目标就是为云原生而设计，遂再造一遍轮子，也抽出一些共生方法，为v3而用。"
author: "James"
---

在2020这个时间节点，之所以还要再造一遍微信支付v2(相对于APIv3来说)的SDK轮子，实属是无奈之举，线下交易场景常见的付款码支付及退款功能，官方当下还没有开放出来v3版的，只能借助v2接口来处理；`wechatpay-axios-plugin` 从一开始目标就是为云原生而设计，遂再造一遍轮子，也抽出一些共生方法，为v3而用。

## 设计思路

此核心部件是利用了Axios的transform功能，数据在类库内部流转过程中，经过 `transformRequest` 及 `transformResponse` 处理，通过类库在本身功能，自定义这两个transformer，从而达到 HTTP 请求/响应 处理。

### Transform.request

方法返回值是个数组，含两个方法 `[signer, toXml]`，字面意思即，对输入数据签名，然后转换成xml；

### Transform.response

方法返回值是个数组，含两个方法 `[toObject, verifier]`，字面意思即，返回值做数据转换为对象，然后校验签名；

## 证书设置

凡是涉及资金变动的接口，均需要商户证书，此实现同时支持 `pem` 及 `p12` 格式的证书，使用方法见随包README：

```javascript
const {Wechatpay, Formatter: fmt} = require('wechatpay-axios-plugin')
const client = Wechatpay.xmlBased({
  secret: 'your_merchant_secret_key_string',
  merchant: {
    cert: '-----BEGIN CERTIFICATE-----' + '...' + '-----END CERTIFICATE-----',
    key: '-----BEGIN PRIVATE KEY-----' + '...' + '-----END PRIVATE KEY-----',
    // or
    // passphrase: 'your_merchant_id',
    // pfx: fs.readFileSync('/your/merchant/cert/apiclient_cert.p12'),
  },
})
```

实力化一个 `client` 的最小参数为 `secret`，即所谓的 **密钥**，字符串形式，32字节长度。

## 自定义打印日志

```javascript
//在格式转换完后，打印日志
client.defaults.transformRequest.push(data => (console.log(data), data))
//在请求返回，先行打印日志
client.defaults.transformResponse.unshift(data => (console.log(data), data))
```

## 使用示例

实例化对象 `secret` 所对应的商户类型，可以是服务商、普通商户、特约商户，入参按照官方文档，手捋填入即可，以下几个方法，均测试过，正常运转。

### 申请退款

```javascript
client.post('/secapi/pay/refund', {
  appid: 'wx8888888888888888',
  mch_id: '1900000109',
  out_trade_no: '1217752501201407033233368018',
  out_refund_no: '1217752501201407033233368018',
  total_fee: 100,
  refund_fee: 100,
  refund_fee_type: 'CNY',
  nonce_str: fmt.nonce(),
}).then(res => console.info(res.data)).catch(({response}) => console.error(response))
```

```javascript
//log输入
{
  return_code: 'SUCCESS',
  return_msg: 'OK',
  appid: 'wx8888888888888888',
  mch_id: '1365319302',
  nonce_str: 'X8bpYtUJbPHK0Fyd',
  sign: '12BDC0390958455875108947AD51D897',
  result_code: 'SUCCESS',
  transaction_id: '4200000684202009114087736848',
  out_trade_no: '1217752501201407033233368018',
  out_refund_no: '1217752501201407033233368018',
  refund_id: '50300005642020091102621479983',
  refund_channel: '',
  refund_fee: '100',
  coupon_refund_fee: '0',
  total_fee: '100',
  cash_fee: '100',
  coupon_refund_count: '0',
  cash_refund_fee: '100'
}
```

### 付款码支付

```javascript
client.post('/pay/micropay', {
  appid: 'wx8888888888888888',
  mch_id: '1900000109',
  nonce_str: fmt.nonce(),
  sign_type: 'HMAC-SHA256',
  body: 'image形象店-深圳腾大-QQ公仔',
  out_trade_no: '1217752501201407033233368018',
  total_fee: 888,
  fee_type: 'CNY',
  spbill_create_ip: '8.8.8.8',
  auth_code: '120061098828009406',
}).then(res => console.info(res.data)).catch(({response}) => console.error(response))
```

### 现金红包

```javascript
client.post('/mmpaymkttransfers/sendredpack', {
  nonce_str: fmt.nonce(),
  mch_billno: '10000098201411111234567890',
  mch_id: '10000098',
  wxappid: 'wx8888888888888888',
  send_name: '鹅企支付',
  re_openid: 'oxTWIuGaIt6gTKsQRLau2M0yL16E',
  total_amount: 1000,
  total_num: 1,
  wishing: 'HAPPY BIRTHDAY',
  client_ip: '192.168.0.1',
  act_name: '回馈活动',
  remark: '会员回馈活动',
  scene_id: 'PRODUCT_4',
}).then(res => console.info(res.data)).catch(({response}) => console.error(response))
```

### 企业付款

```javascript
client.post('/mmpaymkttransfers/promotion/transfers', {
  appid: 'wx8888888888888888',
  mch_id: '1900000109',
  partner_trade_no: '10000098201411111234567890',
  openid: 'oxTWIuGaIt6gTKsQRLau2M0yL16E',
  check_name: 'FORCE_CHECK',
  re_user_name: '王小王',
  amount: 10099,
  desc: '理赔',
  spbill_create_ip: '192.168.0.1',
  nonce_str: fmt.nonce(),
}).then(res => console.info(res.data)).catch(({response}) => console.error(response))
```

## 和v3一起用

```javascript
const wxpay = new Wechatpay({
  mchid: 'your_merchant_id',
  serial: 'serial_number_of_your_merchant_public_cert',
  privateKey: '-----BEGIN PRIVATE KEY-----' + '...' + '-----END PRIVATE KEY-----',
  certs: {
    'serial_number': '-----BEGIN CERTIFICATE-----' + '...' + '-----END CERTIFICATE-----',
  }
})
```

### Native下单

```javascript
wxpay.v3.pay.transactions.native
  .post({/*文档参数放这里就好*/})
  .then(({data: {code_url}}) => console.info(code_url))
  .catch(({response: {status, statusText, data}}) => console.error(status, statusText, data))
```

### 查询订单

```javascript
wxpay.v3.pay.transactions.id['{transaction_id}']
  .withEntities({transaction_id: '1217752501201407033233368018'})
  .get({params: {mchid: '1230000109'}})
  .then(({data}) => console.info(data))
  .catch(({response: {status, statusText, data}}) => console.error(status, statusText, data))
```

### 关单

```javascript
wxpay.v3.pay.transactions.outTradeNo['1217752501201407033233368018']
  .post({mchid: '1230000109'})
  .then(({status, statusText}) => console.info(status, statusText))
  .catch(({response: {status, statusText, data}}) => console.error(status, statusText, data))
```

### 创建商家券

```javascript
wxpay.v3.marketing.busifavor.stocks
  .post({/*商家券创建条件*/})
  .then(({data}) => console.info(data))
  .catch(({response: {status, statusText, data}}) => console.error(status, statusText, data))
```

### 查询用户单张券详情

```javascript
;(async () => {
  try {
    const {data: detail} = await wxpay.v3.marketing.busifavor.users.$openid$.coupons['{coupon_code}'].appids['wx233544546545989']
      .withEntities({openid: '2323dfsdf342342', coupon_code: '123446565767'})
      .get()
    console.info(detail)
  } catch({response: {status, statusText, data}}) {
    console.error(status, statusText, data)
  }
}
```

### 上传图片

```javascript
const FormData = require('form-data')
const {createReadStream} = require('fs')

const imageMeta = {
  filename: 'hellowechatpay.png',
  // easy calculated by the command `sha256sum hellowechatpay.png` on OSX
  // or by require('wechatpay-axios-plugin').Hash.sha256(filebuffer)
  sha256: '1a47b1eb40f501457eaeafb1b1417edaddfbe7a4a8f9decec2d330d1b4477fbe',
}

const imageData = new FormData()
imageData.append('meta', JSON.stringify(imageMeta), {contentType: 'application/json'})
imageData.append('file', createReadStream('./hellowechatpay.png'))

Wechatpay.client.post('/v3/marketing/favor/media/image-upload', imageData, {
  meta: imageMeta,
  headers: imageData.getHeaders()
}).then(res => {
  console.info(res.data.media_url)
}).catch(error => {
  console.error(error)
})
```

### 下载账单并格式化

```javascript
const assert = require('assert')
const {Hash: {sha1}} = require('wechatpay-axios-plugin')

Wechatpay.client.get('/v3/bill/tradebill', {
  params: {
    bill_date: '2020-06-01',
    bill_type: 'ALL',
  }
}).then(({data: {download_url, hash_value}}) => client.get(download_url, {
    signed: hash_value,
    responseType: 'arraybuffer',
})).then(res => {
  assert(sha1(res.data) === res.config.signed, 'verify the SHA1 digest failed.')
  console.info(fmt.castCsvBill(res.data))
}).catch(error => {
  console.error(error)
})
```

### 委托营销

```javascript
(async () => {
  try {
    const res = await Wechatpay.client.post(`/v3/marketing/partnerships/build`, {
      partner: {
        type,
        appid
      },
      authorized_data: {
        business_type,
        stock_id
      }
    }, {
      headers: {
        [`Idempotency-Key`]: 12345
      }
    })
    console.info(res.data)
  } catch (error) {
    console.error(error)
  }
})()
```

### 查询投诉信息并解密

```javascript
;(async () => {
  try {
    const res = await Wechatpay.client.get('/v3/merchant-service/complaints', {params: {
      limit: 50,
      offset: 0,
      begin_date: (new Date(+new Date - 29*86400*1000)).toJSON().slice(0, 10),
      end_date: (new Date).toJSON().slice(0, 10),
    }})
    // decrypt the `Sensitive Information`
    res.data.data.map(row => (row.payer_phone = rsa.decrypt(row.payer_phone, merchantPrivateKey), row))
    console.info(res.data)
  } catch({response: {status, statusText, data, headers}, request, config}) {
    console.error(status, statusText, data)
  }
})()
```

## TODO

v2版的`AES-256-ECB/PKCS7Padding`未做封装，这个不难，npm上也有许多优秀的类库可用，暂且先这样。

## 写到最后

MIT开放源码@[npm](https://www.npmjs.com/package/wechatpay-axios-plugin), [github](https://github.com/TheNorthMemory/wechatpay-axios-plugin) ，可用于企业商业用途。

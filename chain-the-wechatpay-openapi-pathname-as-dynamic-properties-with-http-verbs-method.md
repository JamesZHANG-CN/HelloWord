---
title: "链式动态参数化的微信支付APIv2&v3' Smart Develop Kit"
date: 2021-02-28T17:30:37+08:00
keywords: [WeChatPay APIv2 APIv3 微信支付SDK]
description: "由一个issue引起的重构，使用相同语法糖(method chain)请求微信支付接口"
tags: []
toc: false
categories: [javascript, wechat]
author: "James"
summary: "由一个issue引起的重构，使用相同语法糖(method chain)请求微信支付接口。"
---

好久没有写博客了，这两天刚把 [wechatpay-axios-plugin](https://github.com/TheNorthMemory/wechatpay-axios-plugin) 做了一遍更新，记录一下“暴力”重构过程。

上一版本的SDK，集合了XML交换介质的APIv2接口，对v2的接口调用，存在诸多不便，一直想优化来着，懒病缠身。。。一直没啥动力；近期有小伙伴给提了个 [#10 在进行订单查询的时候，订单号不能更新](https://github.com/TheNorthMemory/wechatpay-axios-plugin/issues/10) 问题，为了解决这个链式调用动态参数赋值污染问题，顺手就给把APIv2也给完美地链了:

目标接口`pathname`映射成资源树如下:

```
v2
├── mmpaymkttransfers
│   └── sendredpack
├── pay
│   ├── micropay
│   └── refundquery
├── secapi
│   ├── mch
│   │   └── querysubdevconfig
│   └── pay
│       └── refund
v3
├── certificates
├── bill
│   └── tradebill
├── ecommerce
│   ├── fund
│   │   └── withdraw
│   └── profitsharing
│       └── orders
└── marketing
    └── busifavor
        └── users
            └── {openid}
                └── coupons
                    └── {coupon_code}
                        └── appids
                            └── {appid}
```

比较上一版的SDK，这一版本去除了 `entities` 及 `withEntities` 对象，直接使用 `Functon.name` 作为存贮并延展资源树节点的上下级标识，这个`Function`同时默认为`HTTP POST`方法对象，并且隐式绑定了`HTTP DELETE/GET/POST/PUT/PATCH` 方法，使实例化的资源树结构更简洁（洁癖），去除调试信息上的 `[Function: ]` label 之后，几乎就和上述资源树一模一样了！

**console.info(wxpay)**

```js
[Function (anonymous)] {
  v2: [Function: /v2] {
    pay: [Function: /v2/pay] { micropay: [Function: /v2/pay/micropay] },
    secapi: [Function: /v2/secapi] {
      pay: [Function: /v2/secapi/pay] {
        refund: [Function: /v2/secapi/pay/refund]
      }
    },
    mmpaymkttransfers: [Function: /v2/mmpaymkttransfers] {
      sendredpack: [Function: /v2/mmpaymkttransfers/sendredpack],
      promotion: [Function: /v2/mmpaymkttransfers/promotion] {
        transfers: [Function: /v2/mmpaymkttransfers/promotion/transfers]
      }
    }
  },
  v3: [Function: /v3] {
    pay: [Function: /v3/pay] {
      transactions: [Function: /v3/pay/transactions] {
        native: [Function: /v3/pay/transactions/native],
        id: [Function: /v3/pay/transactions/id] {
          '{transaction_id}': [Function: /v3/pay/transactions/id/{transaction_id}]
        },
        outTradeNo: [Function: /v3/pay/transactions/out-trade-no] {
          '1217752501201407033233368018': [Function: /v3/pay/transactions/out-trade-no/1217752501201407033233368018]
        }
      },
      partner: [Function: /v3/pay/partner] {
        transactions: [Function: /v3/pay/partner/transactions] {
          native: [Function: /v3/pay/partner/transactions/native]
        }
      }
    },
    marketing: [Function: /v3/marketing] {
      busifavor: [Function: /v3/marketing/busifavor] {
        stocks: [Function: /v3/marketing/busifavor/stocks],
        users: [Function: /v3/marketing/busifavor/users] {
          '$openid$': [Function: /v3/marketing/busifavor/users/{openid}] {
            coupons: [Function: /v3/marketing/busifavor/users/{openid}/coupons] {
              '{coupon_code}': [Function: /v3/marketing/busifavor/users/{openid}/coupons/{coupon_code}] {
                appids: [Function: /v3/marketing/busifavor/users/{openid}/coupons/{coupon_code}/appids] {
                  wx233544546545989: [Function: /v3/marketing/busifavor/users/{openid}/coupons/{coupon_code}/appids/wx233544546545989]
                }
              }
            }
          }
        }
      },
      favor: [Function: /v3/marketing/favor] {
        media: [Function: /v3/marketing/favor/media] {
          imageUpload: [Function: /v3/marketing/favor/media/image-upload]
        },
        stocks: [Function: /v3/marketing/favor/stocks] {
          '$stock_id$': [Function: /v3/marketing/favor/stocks/{stock_id}] {
            useFlow: [Function: /v3/marketing/favor/stocks/{stock_id}/use-flow]
          }
        }
      },
      partnerships: [Function: /v3/marketing/partnerships] {
        build: [Function: /v3/marketing/partnerships/build]
      }
    },
    combineTransactions: [Function: /v3/combine-transactions] {
      jsapi: [Function: /v3/combine-transactions/jsapi]
    },
    bill: [Function: /v3/bill] { tradebill: [Function: /v3/bill/tradebill] },
    billdownload: [Function: /v3/billdownload] {
      file: [Function: /v3/billdownload/file]
    },
    smartguide: [Function: /v3/smartguide] {
      guides: [Function: /v3/smartguide/guides] {
        '$guide_id$': [Function: /v3/smartguide/guides/{guide_id}] {
          assign: [Function: /v3/smartguide/guides/{guide_id}/assign]
        }
      }
    },
    merchantService: [Function: /v3/merchant-service] {
      complaints: [Function: /v3/merchant-service/complaints]
    },
    merchant: [Function: /v3/merchant] {
      media: [Function: /v3/merchant/media] {
        video_upload: [Function: /v3/merchant/media/video_upload]
      }
    }
  }
}
```

编码书写方式有如下约定：

1. 请求 `URI` 作为级联对象，可以轻松构建请求对象，例如 `/v3/pay/transactions/native` 即自然翻译成 `v3.pay.transactions.native`;
2. 每个 `URI` 所支持的 `HTTP METHOD`，即作为 请求对象的末尾执行方法，例如: `v3.pay.transactions.native.post({})`;
3. 每个 `URI` 有中线(dash)分隔符的，可以使用驼峰`camelCase`风格书写，例如: `merchant-service`可写成 `merchantService`，或者属性风格，例如 `v3['merchant-service']`;
4. 每个 `URI`.pathname 中，若有动态参数，例如 `business_code/{business_code}` 可写成 `business_code.$business_code$` 或者属性风格书写，例如 `business_code['{business_code}']`，抑或直接按属性风格，直接写参数值也可以，例如 `business_code['2000001234567890']`;
5. 建议 `URI` 按照 `PascalCase` 风格书写, `TS Definition` 已在路上(还有若干问题没解决)，将是这种风格，代码提示将会很自然;
6. SDK内置的 `/v2` 对象，其特殊标识为APIv2级联对象，之后串接切分后的`pathname`，如 `/v2/pay/micropay` 即以XML形式请求远端接口；
7. 每个级联对象默认为HTTP`POST`函数，其同时隐式内置`GET/POST/PUT/PATCH/DELETE` 操作方法链，支持全大写及全小写(未来有可能会删除)两种编码方式，说明见`变更历史`;

### 初始化

```js
const {Wechatpay, Formatter} = require('wechatpay-axios-plugin')
const wxpay = new Wechatpay({
  mchid: 'your_merchant_id',
  serial: 'serial_number_of_your_merchant_public_cert',
  privateKey: '-----BEGIN PRIVATE KEY-----' + '...' + '-----END PRIVATE KEY-----',
  certs: {
    'serial_number': '-----BEGIN CERTIFICATE-----' + '...' + '-----END CERTIFICATE-----',
  },
  // APIv2参数 >= 0.4.0 开始支持
  secret: 'your_merchant_secret_key_string',
  // 注： 如果不涉及资金变动，如仅收款，merchant参数可选，仅需 `secret` 一个参数，注意其为v2版的。
  merchant: {
    cert: '-----BEGIN CERTIFICATE-----' + '...' + '-----END CERTIFICATE-----',
    key: '-----BEGIN PRIVATE KEY-----' + '...' + '-----END PRIVATE KEY-----',
    // or
    // passphrase: 'your_merchant_id',
    // pfx: fs.readFileSync('/your/merchant/cert/apiclient_cert.p12'),
  },
})
```

### 付款码(刷卡)支付

```js
wxpay.v2.pay.micropay({
  appid: 'wx8888888888888888',
  mch_id: '1900000109',
  nonce_str: Formatter.nonce(),
  sign_type: 'HMAC-SHA256',
  body: 'image形象店-深圳腾大-QQ公仔',
  out_trade_no: '1217752501201407033233368018',
  total_fee: 888,
  fee_type: 'CNY',
  spbill_create_ip: '8.8.8.8',
  auth_code: '120061098828009406',
})
.then(res => console.info(res.data))
.catch(({response: {status, statusText, data}}) => console.error(status, statusText, data))
```

### 关单API
```js
wxpay.v3.pay.transactions.outTradeNo.$out_trade_no$
  .post({mchid: '1230000109'}, {out_trade_no: '1217752501201407033233368018'})
  .then(({status, statusText}) => console.info(status, statusText))
  .catch(({response: {status, statusText, data}}) => console.error(status, statusText, data))
```

核心构造类去掉注释，也就几十行代码如下:

```js
const Decorator = require('./decorator')

const CLIENT = Symbol('CLIENT')

class Wechatpay {
  static get client() { return this[CLIENT] }

  static normalize(str) {
    return (str || '')
      .replace(/^[A-Z]/, w => w.toLowerCase())
      .replace(/[A-Z]/g, w => `-${w.toLowerCase()}`)
      .replace(/^\$(.*)\$$/, `{$1}`)
  }

  static compose(prefix = '', suffix = '') {
    const name = (prefix || suffix) ? `${prefix}/${suffix}` : `${suffix}`

    return new Proxy(this.chain(name), this.handler)
  }

  static chain(pathname) {
    const client = this[CLIENT]

    return ['POST', 'PUT', 'PATCH', /*alias*/'post', 'put', 'patch'].reduce((resource, method) => {
      return Object.defineProperty(resource, method, {value: {[method](data, config) {
        return client.request(pathname, method, data, config)
      }}[method]})
    }, ['DELETE', 'GET', /*alias*/'delete', 'get'].reduce((resource, method) => {
      return Object.defineProperty(resource, method, {value: {[method](config) {
        return client.request(pathname, method, undefined, config)
      }}[method]})
    }, {[pathname](data, config) {
      return client.request(pathname, 'POST', data, config)
    }}[pathname]))
  }

  static get handler() {
    return {
      get: (target, property) => {
        if (typeof property === `symbol` || property === `inspect`) {
          return target
        }

        if (!Object.prototype.hasOwnProperty.call(target, property)) {
          target[property] = this.compose(target.name, this.normalize(property))
        }

        return target[property]
      },
    }
  }

  constructor(config = {}) {
    return Object.defineProperty(this.constructor, CLIENT, {value: new Decorator(config)}).compose()
  }
}
```

好吧，这可能是我所写的最“暴力”的代码了。。。打个记号～嘿嘿～

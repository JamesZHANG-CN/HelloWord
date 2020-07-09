---
title: "微信支付APIv3的Nodejs版SDK，让开发变得简单不再繁琐"
date: 2020-07-09T13:33:33+08:00
keywords: ["Wechatpay APIv3", "wechatpay-axios-plugin"]
description: "以树形构型方式实现微信支付APIv3的Nodejs版SDK，支持API弹性扩容，让v3开发简单高效。"
tags: []
toc: false
categories: ["wechat"]
summary: "以树形构型方式实现微信支付APIv3的Nodejs版SDK，支持API弹性扩容，让v3开发简单高效。"
author: "James"
---

在向云端推送这个 `wechatpay-axios-plugin` 业务实现时，发现0.1系列还不够好用，还需要进行更多层级的包裹包装，遂再次做了重大更新，让SDK使用起来更简单、飘逸。

先看官方文档，每一个接口，文档都至少标示了`请求URL` `请求方式` `请求参数` `返回参数` 这几个要素，`URL` 可以拆分成 `Base` 及 `URI`，按照这种思路，封装SDK其实完全就可以不用动脑，即，对`URI`资源的 `POST` 或 `GET` 请求(条件带上`参数`)，取得`返回参数`。

更近一步，我们设想一下，如果把众多接口的`URI`按照斜线(`/` `slash`)分割，然后组织在一起，是不是就可以构建出一颗树，这颗树的每个节点(实体`Entity`)都存在有若干个方法(`HTTP METHODs`)，这是不是就能把接口`SDK实现`更简单化了?!

例如：

- /v3/certificates
- /v3/bill/tradebill
- /v3/ecommerce/fund/withdraw
- /v3/ecommerce/profitsharing/orders
- /v3/marketing/busifavor/users/{openid}/coupons/{coupon_code}/appids/{appid}

树形化即：

```
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

按照这种树形构想，我们来看下需要做的`封装实现`工作：

1. 把实体对象，按照实体的排列顺序，映射出请求的URI；
2. 每个对象实体，包含有若干操作方法，其中可选带参数发起RPC请求；
3. 随官方放出更多的接口，SDK需要能够弹性扩容；

wechatpay-axios-plugin~0.2.0 版本实现了上述这3个目标，我们用伪代码来校验看一下这个`封装实现`：

```javascript
require('util').inspect.defaultOptions.depth = 10;

const { Wechatpay } = require('wechatpay-axios-plugin');

const wxpay = new Wechatpay({mchid: '1', serial: '2', privateKey: '3', certs: {'4': '5'}});

wxpay.v3.certificates;
wxpay.v3.bill.tradebill;
wxpay.v3.ecommerce.fund.withdraw;
wxpay.v3.marketing.busifavor.users['{openid}'].coupons.$coupon_code$.appids['wx233544546545989'];

console.info(wxpay);
//以下是输出内容
{
  entities: [],
  withEntities: [Function: withEntities],
  get: [AsyncFunction: get],
  post: [AsyncFunction: post],
  upload: [AsyncFunction: upload],
  v3: {
    entities: [ 'v3' ],
    withEntities: [Function: withEntities],
    get: [AsyncFunction: get],
    post: [AsyncFunction: post],
    upload: [AsyncFunction: upload],
    certificates: {
      entities: [ 'v3', 'certificates' ],
      withEntities: [Function: withEntities],
      get: [AsyncFunction: get],
      post: [AsyncFunction: post],
      upload: [AsyncFunction: upload]
    },
    bill: {
      entities: [ 'v3', 'bill' ],
      withEntities: [Function: withEntities],
      get: [AsyncFunction: get],
      post: [AsyncFunction: post],
      upload: [AsyncFunction: upload],
      tradebill: {
        entities: [ 'v3', 'bill', 'tradebill' ],
        withEntities: [Function: withEntities],
        get: [AsyncFunction: get],
        post: [AsyncFunction: post],
        upload: [AsyncFunction: upload]
      }
    },
    ecommerce: {
      entities: [ 'v3', 'ecommerce' ],
      withEntities: [Function: withEntities],
      get: [AsyncFunction: get],
      post: [AsyncFunction: post],
      upload: [AsyncFunction: upload],
      fund: {
        entities: [ 'v3', 'ecommerce', 'fund' ],
        withEntities: [Function: withEntities],
        get: [AsyncFunction: get],
        post: [AsyncFunction: post],
        upload: [AsyncFunction: upload],
        withdraw: {
          entities: [ 'v3', 'ecommerce', 'fund', 'withdraw' ],
          withEntities: [Function: withEntities],
          get: [AsyncFunction: get],
          post: [AsyncFunction: post],
          upload: [AsyncFunction: upload]
        }
      }
    },
    marketing: {
      entities: [ 'v3', 'marketing' ],
      withEntities: [Function: withEntities],
      get: [AsyncFunction: get],
      post: [AsyncFunction: post],
      upload: [AsyncFunction: upload],
      busifavor: {
        entities: [ 'v3', 'marketing', 'busifavor' ],
        withEntities: [Function: withEntities],
        get: [AsyncFunction: get],
        post: [AsyncFunction: post],
        upload: [AsyncFunction: upload],
        users: {
          entities: [ 'v3', 'marketing', 'busifavor', 'users' ],
          withEntities: [Function: withEntities],
          get: [AsyncFunction: get],
          post: [AsyncFunction: post],
          upload: [AsyncFunction: upload],
          '{openid}': {
            entities: [ 'v3', 'marketing', 'busifavor', 'users', '{openid}' ],
            withEntities: [Function: withEntities],
            get: [AsyncFunction: get],
            post: [AsyncFunction: post],
            upload: [AsyncFunction: upload],
            coupons: {
              entities: [
                'v3',
                'marketing',
                'busifavor',
                'users',
                '{openid}',
                'coupons'
              ],
              withEntities: [Function: withEntities],
              get: [AsyncFunction: get],
              post: [AsyncFunction: post],
              upload: [AsyncFunction: upload],
              '$coupon_code$': {
                entities: [
                  'v3',
                  'marketing',
                  'busifavor',
                  'users',
                  '{openid}',
                  'coupons',
                  '{coupon_code}'
                ],
                withEntities: [Function: withEntities],
                get: [AsyncFunction: get],
                post: [AsyncFunction: post],
                upload: [AsyncFunction: upload],
                appids: {
                  entities: [
                    'v3',
                    'marketing',
                    'busifavor',
                    'users',
                    '{openid}',
                    'coupons',
                    '{coupon_code}',
                    'appids'
                  ],
                  withEntities: [Function: withEntities],
                  get: [AsyncFunction: get],
                  post: [AsyncFunction: post],
                  upload: [AsyncFunction: upload],
                  wx233544546545989: {
                    entities: [
                      'v3',
                      'marketing',
                      'busifavor',
                      'users',
                      '{openid}',
                      'coupons',
                      '{coupon_code}',
                      'appids',
                      'wx233544546545989'
                    ],
                    withEntities: [Function: withEntities],
                    get: [AsyncFunction: get],
                    post: [AsyncFunction: post],
                    upload: [AsyncFunction: upload]
                  }
                }
              }
            }
          }
        }
      }
    }
  }
}
```

**注：** API树实体节点，存储在每个 `entities` 属性上，方便后续的`get`, `post` 抑或 `upload` 方法调用调用前，反构成最终请求的`URI`；特别地，对于动态树实体节点来说，每个实体节点均提供了 `withEntities` 方法，用来在最终请求前，把动态实体节点替换成实际的值。

正常用法示例如下：

```javascript
const {Wechatpay} = require('wechatpay-axios-plugin');
const wxpay = new Wechatpay({/*初始化参数，README有*/}, {/*可选调整axios的参数*/});

//拿证书
wxpay.v3.certificates.get();

//带参申请交易账单
wxpay.v3.bill.tradebill.get({params: {bill_date}});

//带参发起账户余额提现
wxpay.v3.ecommerce.fund.withdraw.post({sub_mchid, out_request_no, amount, remark, bank_memo});

//查询用户单张券详情
wxpay.v3.marketing.busifavor.users['{openid}'].coupons.$coupon_code$.appids['wx233544546545989'].withEntities({openid, coupon_code}).get();
```

请求APIv3是不是就“*丧心病狂*”般的简单了？！

详细功能说明及用法示例，npmjs及github的README均有。

文章首发于[微信开发者社区](https://developers.weixin.qq.com/community/pay/article/doc/000cc48a370008b8ff9a64a3c51013)，如果喜欢，就给来个 **赞** 及 **Star** 吧。

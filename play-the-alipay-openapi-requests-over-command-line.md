---
title: "命令行方式与支付宝OpenAPI网关交互"
date: 2020-10-20T15:21:06+08:00
keywords: [alipay openapi cli 支付宝开放平台接口CLI工具]
description: "Play the Alipay OpenAPI Requests Over Command Line，以命令行方式与OpenAPI网关交互——花式快速体验支付开放能力"
tags: []
categories: [javascript, alipay]
author: "James"
summary: "以命令行方式与OpenAPI网关交互——花式快速体验支付宝开放能力"
---

以命令行方式与OpenAPI网关交互——花式快速体验支付开放能力

这其实是此款[nodejs SDK](https://github.com/TheNorthMemory/whats-alipay)的用法展示，同时一并补充官方「在线调试」用法，让输入输出JSON更加直观，加速官方开放能力体验。


安装包


`npm install whats-alipay`

执行命令查看帮助信息

`./node_modules/.bin/whatsCli -h`

```
Usage: cli.js [options]
Options:
  -k, --privateKey  The privateKey pem file path                                                              [required]
  -p, --publicCert  The publicCert pem file path                                                              [required]
  -m, --method      The method, eg: alipay.trade.query                                                        [required]
  -s, --search      The search parameters, eg: search.app_id=2088                                             [required]
  -b, --biz         The biz_content, eg: biz.out_trade_no=abcd1234                                            [required]
  -d, --log         Turn on the request trace log                                                              [boolean]
  -u, --baseURL     The OpenAPI gateway, eg: https://openapi.alipaydev.com/gateway.do
  -h, --help        Show help                                                                                  [boolean]
  -V, --version     Show version number                                                                        [boolean]
Examples:
  cli.js -k merchant.key -p alipay.pub -m alipay.trade.pay      The Face2Face barCode scenario
  -s.app_id=2088 -b.subject=HelloKitty
  -b.out_trade_no=Kitty0001 -b.scene=bar_code
  -b.total_amount=0.01 -b.auth_code=
  cli.js -k merchant.key -p alipay.pub -m alipay.trade.refund   The trade refund scenario
  -s.app_id=2088 -b.refund_amount=0.01 -b.refund_currency=CNY
  -b.out_trade_no=Kitty0001
  cli.js -d -u https://openapi.alipaydev.com/gateway.do -k      The trade query scenario over the sandbox environment
  merchant.key -p alipay.pub -m alipay.trade.query              with trace logging
  -s.app_id=2088 -b.out_trade_no=Kitty0001
```

快速体验

面对面支付产品能力

`
./node_modules/.bin/whatsCli -k merchant.key -p alipay.pub -m alipay.trade.pay -s.app_id=2088 -b.subject=HelloKitty -b.out_trade_no=Kitty0001 -b.scene=bar_code -b.total_amount=0.01 -b.auth_code=
`

配合扫描枪，可以让如上auth_code从手动切换成快速输入并提交，体验更佳。

通用退款能力


`./node_modules/.bin/whatsCli -k merchant.key -p alipay.pub -m alipay.trade.refund -s.app_id=2088 -b.refund_amount=0.01 -b.refund_currency=CNY -b.out_trade_no=Kitty0001`


付款单查询能力

`./node_modules/.bin/whatsCli -k merchant.key -p alipay.pub -m alipay.trade.query -s.app_id=2088 -b.out_trade_no=Kitty0001`

沙箱环境

`./node_modules/.bin/whatsCli -u https://openapi.alipaydev.com/gateway.do -k merchant.key -p alipay.pub -m alipay.trade.query -s.app_id=2088 -b.out_trade_no=Kitty0001`

开启查询交互日志

`./node_modules/.bin/whatsCli --log -u https://openapi.alipaydev.com/gateway.do -k merchant.key -p alipay.pub -m alipay.trade.query -s.app_id=2088 -b.out_trade_no=Kitty0001`

```javascript
[
  'https://openapi.alipaydev.com/gateway.do',
  'post',
  { out_trade_no: 'Kitty0001' },
  {
    format: 'JSON',
    charset: 'utf-8',
    sign_type: 'RSA2',
    version: '1.0',
    app_id: 2088,
    method: 'alipay.trade.query'
  }
]
biz_content=%7B%22out_trade_no%22%3A%22Kitty0001%22%7D&sign=emhZqmbUNkGWoCwxcalzr9gF9ZQ6IjqdbStC32S4DnJw4Z15omMDghCs58LF%2Bb1alNeOKrS5YIH2ISx23ZuD50GeCIWy3nXUaaouwdIih38dtFKb6jqkBfhiiFs1V1%2FGg91gjc83PboBQB3thmmll2zILkWuPYQoz964wnR%2FJ04Wx%2FBsIHlzD0Tr2bhur%2B5lE0Ldg2EzYm%2FyLN7yKUaIAHmjpHMbWwQ2EQrEsic6qpRNqjHJ7Tmp9k6kGfndkT06r1Mpe2WxSh6fabDi%2Beh1CX%2BXnS8KX4Umeg%2F0swfaAEb9GbnKgeLgp43eUj9S0KbtG8wvFSA%2FqUkhWfXoh8Cicw%3D%3D
{"alipay_trade_query_response":{"code":"40002","msg":"Invalid Arguments","sub_code":"isv.invalid-app-id","sub_msg":"无效的AppID参数"}}
{
  alipay_trade_query_response: {
    code: '40002',
    msg: 'Invalid Arguments',
    sub_code: 'isv.invalid-app-id',
    sub_msg: '无效的AppID参数'
  }
}
```

以上体验，已随 whats-aliapy nodejs sdk v0.0.11 发布，cli核心代码如下：

```javascript
const fs = require('fs')
const {Alipay, Decorator} = require('..')
const whats = new Alipay({
  privateKey: fs.readFileSync(argv.privateKey),
  publicCert: fs.readFileSync(argv.publicCert),
})

argv.method.split('.').reduce((f, i) => f[i], whats)(argv.biz, argv.search)
  .catch(({response: {data}}) => data)
  .then(({data}) => data)
  .then(console.log)
```

欢迎体验，欢迎Star, 项目地址 https://github.com/TheNorthMemory/whats-alipay

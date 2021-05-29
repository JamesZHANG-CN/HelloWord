---
title: "简单bash组合CLI能力，练习接入/使用「微信支付v2」沙箱环境"
date: 2021-04-30T21:20:07+08:00
keywords: [wechatpay, apiv2, sandbox, cli]
description: "前两天有一开发者，一直在捣鼓签名的事儿，我建议其使用一下CLI工具，发现还是得再做个“教程”引导贴，遂就拿沙箱环境来练习吧。"
tags: []
toc: false
categories: [javascript, wechat]
summary: "前两天有一开发者，一直在捣鼓签名的事儿，我建议其使用一下CLI工具，发现还是得再做个“教程”引导贴，遂就拿沙箱环境来练习吧。"
author: "James"
---

前两天有一开发者，一直在捣鼓签名的事儿，我建议其使用一下CLI工具，发现还是得再做个“教程”引导贴，遂就拿沙箱环境来练习吧。

## 引言

接续的是上一贴[免充值产品测试验收用例的教学帖](https://developers.weixin.qq.com/community/develop/article/doc/000ea011304e3852750cd611456c13)，本篇以纯shell结合CLI模式运转，windows用户建议使用WSL/WSL2来练习。


## 环境准备

1. 空建一个文件夹，例如: ~/wxpay_sandbox (熟npm用户可忽略)
2. 增加两字节的内容是`{}`的`package.json`文件 (熟npm用户可忽略)
3. 安装依赖软件包 `npm install wechatpay-axios-plugin yargs --save`
4. 简单shell套个壳，例如 `~/wxpay_sandbox/practice.sh`
5. 更新文件为可执行 `chmod +x ~/wxpay_sandbox/practice.sh`

```shell
#!/bin/bash

# filename practice.sh

appid="你的appid"
mchid="你的商户号"

./node_modules/.bin/wxpay -c.appid $appid -c.mchid $mchid -c.serial any -c.privateKey any -c.certs.any $@
```

## 获取沙箱密钥

切换工作目录 `cd ~/wxpay_sandbox/`，然后执行如下命令：

```shell
./practice.sh v2.sandboxnew.pay.getsignkey -b -b -c.secret 你商户的密钥 -d.mch_id 你的商户号 -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS
```

记录一下屏幕打印的 `沙箱密钥` sandbox_signkey 值，后续练习均需要这个值。

## 1001 付款码(刷卡)支付

订单金额 `5.01` 元，其中 `0.01` 元使用免充值券，用户实际支付 `5.00` 元。验证商户具备正确解析及识别免充值代金券字段的能力。

### 1001.1 请求支付

```shell
./practice.sh v2.sandboxnew.pay.micropay \
  -c.secret 沙箱密钥 \
  -d.appid 你的appid \
  -d.mch_id 你的商户号 \
  -d.out_trade_no sandbox20210430200000_501 \
  -d.body dummybot \
  -d.total_fee 501 \
  -d.spbill_create_ip 127.0.0.1 \
  -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS \
  -d.auth_code 120061098828009406
```

注意查看屏幕打印的 coupon_fee 代金券金额, settlement_total_fee 应结订单金额, total_fee 订单金额


### 1001.2 获取支付结果

```shell
./practice.sh v2.sandboxnew.pay.orderquery \
  -c.secret 沙箱密钥 \
  -d.appid 你的appid \
  -d.mch_id 你的商户号 \
  -d.out_trade_no sandbox20210430200000_501 \
  -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS
```

注意查看屏幕打印，这里不累述了

## 1002 付款码(刷卡)支付退款

订单金额 `5.02` 元，其中 `0.01` 元使用免充值代金劵，实际支付 `5.01` 元，退款查询升级。

### 1002.1 请求支付

```shell
./practice.sh v2.sandboxnew.pay.micropay \
  -c.secret 沙箱密钥 \
  -d.appid 你的appid \
  -d.mch_id 你的商户号 \
  -d.body dummybot \
  -d.total_fee 502 \
  -d.spbill_create_ip 127.0.0.1 \
  -d.out_trade_no sandbox20210430200000_502 \
  -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS \
  -d.auth_code 120061098828009406
```

### 1002.2 获取支付结果

```shell
./practice.sh v2.sandboxnew.pay.orderquery \
  -c.secret 沙箱密钥 \
  -d.appid 你的appid \
  -d.mch_id 你的商户号 \
  -d.out_trade_no sandbox20210430200000_502 \
  -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS
```

### 1002.3 请求退款

```shell
./practice.sh v2.sandboxnew.pay.refund \
  -c.secret 沙箱密钥 \
  -d.appid 你的appid \
  -d.mch_id 你的商户号 \
  -d.out_trade_no sandbox20210430200000_502 \
  -d.out_refund_no RD_sandbox20210430200000_502 \
  -d.total_fee 502 \
  -d.refund_fee 501 \
  -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS
```

### 1002.4 获取退款结果

```shell
./practice.sh v2.sandboxnew.pay.refundquery \
  -c.secret 沙箱密钥 \
  -d.appid 你的appid \
  -d.mch_id 你的商户号 \
  -d.out_trade_no sandbox20210430200000_502 \
  -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS
```

## 1003 JSAPI/APP/Native支付

订单金额 `5.51` 元，其中 `0.01` 元使用免充值券，实际支付 `5.50` 元。 验证正常支付流程，商户使用免充值代金券支付。

### 1003.1 统一下单

```shell
./practice.sh v2.sandboxnew.pay.unifiedorder \
  -c.secret 沙箱密钥 \
  -d.appid 你的appid \
  -d.mch_id 你的商户号 \
  -d.out_trade_no sandbox20210430200000_551 \
  -d.body dummybot \
  -d.total_fee 551 \
  -d.notify_url https://www.weixin.qq.com/wxpay/pay.php \
  -d.spbill_create_ip 127.0.0.1 \
  -d.trade_type JSAPI \
  -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS
```

### 1003.2 获取支付结果

```shell
./practice.sh v2.sandboxnew.pay.orderquery \
  -c.secret 沙箱密钥 \
  -d.appid 你的appid \
  -d.mch_id 你的商户号 \
  -d.out_trade_no sandbox20210430200000_551 \
  -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS
```

## 1004 JSAPI/APP/Native支付退款

订单金额 `5.52` 元，其中 `0.01` 元使用免充值券，实际支付 `5.51` 元。

### 1004.1 统一下单

```shell
./practice.sh v2.sandboxnew.pay.unifiedorder \
  -c.secret 沙箱密钥 \
  -d.appid 你的appid \
  -d.mch_id 你的商户号 \
  -d.out_trade_no sandbox20210430200000_552 \
  -d.body dummybot \
  -d.total_fee 552 \
  -d.notify_url https://www.weixin.qq.com/wxpay/pay.php \
  -d.spbill_create_ip 127.0.0.1 \
  -d.trade_type JSAPI \
  -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS
```

### 1004.2 获取支付结果

```shell
./practice.sh v2.sandboxnew.pay.orderquery \
  -c.secret 沙箱密钥 \
  -d.appid 你的appid \
  -d.mch_id 你的商户号 \
  -d.out_trade_no sandbox20210430200000_552 \
  -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS
```

### 1004.3 请求退款

```shell
./practice.sh v2.sandboxnew.pay.refund \
  -c.secret 沙箱密钥 \
  -d.appid 你的appid \
  -d.mch_id 你的商户号 \
  -d.out_trade_no sandbox20210430200000_552 \
  -d.out_refund_no RD_sandbox20210430200000_552 \
  -d.total_fee 552 \
  -d.refund_fee 551 \
  -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS
```

### 1004.4 获取退款结果

```shell
./practice.sh v2.sandboxnew.pay.refundquery \
  -c.secret 沙箱密钥 \
  -d.appid 你的appid \
  -d.mch_id 你的商户号 \
  -d.out_trade_no sandbox20210430200000_552 \
  -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS
```

## 1005 交易对账单下载


```shell
./practice.sh v2.sandboxnew.pay.downloadbill \
  -b -b -b \
  -c.secret 沙箱密钥 \
  -d.appid 你的appid \
  -d.mch_id 你的商户号 \
  -d.bill_type ALL \
  -d.bill_date 2021-04-30 \
  -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS
```


## 最后

就等你来练习咯～

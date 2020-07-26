---
title: "微信支付APIv3接口文档及开发者工具"
date: 2020-07-23T09:41:32+08:00
keywords: [wechatpay apiv3 openapi yaml specifications]
description: "共计收录微信支付APIv3官方文档118+18个开放接口定义，SPA单页应用，适配移动端，支持标签检索。"
tags: []
toc: false
categories: [wechat]
summary: "官方文档侧重点是业务介绍，感觉是顺带把接口释义也做了下，对开发来说，请求方法、URI地址、数据结构、数据类型 这些才是关键点，本项目只摘录这些点，期望能给开发带来便利；另外这个项目其实是附产，本是为了生成代码，结果*悲伤*有一大筐，先就开源介绍一下这个项目吧。"
author: "James"
---

## 项目地址

整理自[微信支付官方文档](https://pay.weixin.qq.com/wiki/doc/apiv3/wxpay/pages/Overview.shtml) [WechatPay-API-v3](https://wechatpay-api.gitbook.io/wechatpay-api-v3/)，SPA单页应用[戳这里体验](https://thenorthmemory.github.io/wechatpay-openapi/)，共计收录118+18个开放接口，适配了移动端体验，支持标签检索。有如下约定：

1. 对于类型定义是 **string($MediaId)**， 需要用 */通用接口* 媒体(图片/视频)上传获取到返回值；
2. 对于类型定义是 **string($MediaUrl)**，需要用 */运营工具/通用接口* 营销图片上传接口取得返回值；
3. 对于类型定义是 **string($rfc3339)**，请使用时间带时区字符串；
4. 对于类型定义是 **string($rfc2397)**，官方文档说是可直接用于img标签，不确定是否符合RFC2397规范；
5. 对于类型定义是 **string($numeric)**，请留意其值传递实质是字符串类型，类型校对需要使用numeric校对；
6. 对于类型定义是 **string($jsonArrayLike)**，请使用数组对象转字符串；
7. 对于类型定义是 **string($date)**，请留意使用 *东八区* 日期格式；
8. 对于类型定义是 **string($url)**，请使用以https开头的URL；
9. 对于类型定义是 **number($uint64)**，请注意返回值JSON解码时，有可能整型溢出；
10. 对于对象属性 **x-is-sensitive**，请根据场景需要，在请求体内需要 *rsa.encrypt*, 在返回体需要 *rsa.decrypt* ；

Q: 有官方文档，这个项目有啥用？

A. 官方文档侧重点是业务介绍，感觉是顺带把接口释义也做了下，对开发来说，**请求方法、URI地址、数据结构、数据类型**这些才是关键点，本项目只摘录这些点，期望能给开发带来便利；另外这个项目其实是附产，本是为了生成代码，结果*悲伤*有一大筐，先就开源介绍一下这个项目吧。


## SPEC文件化

- ./spec/-notify.yaml
- ./spec/apply4sub/sub_merchants/{sub_mchid}/modify-settlement.yaml
- ./spec/apply4sub/sub_merchants/{sub_mchid}/settlement.yaml
- ./spec/apply4subject/applyment.yaml
- ./spec/apply4subject/applyment/merchants/{sub_mchid}/state.yaml
- ./spec/apply4subject/applyment/{applyment_id}/cancel.yaml
- ./spec/apply4subject/applyment/{business_code}/cancel.yaml
- ./spec/applyment4sub/applyment/applyment_id/{applyment_id}.yaml
- ./spec/applyment4sub/applyment/business_code/{business_code}.yaml
- ./spec/applyment4sub/applyment/stub.yaml
- ./spec/bill/fundflowbill.yaml
- ./spec/bill/tradebill.yaml
- ./spec/billdownload/file.yaml
- ./spec/certificates.yaml
- ./spec/combine-transactions/app.yaml
- ./spec/combine-transactions/h5.yaml
- ./spec/combine-transactions/jsapi.yaml
- ./spec/combine-transactions/native.yaml
- ./spec/combine-transactions/out-trade-no/{combine_out_trade_no}.yaml
- ./spec/combine-transactions/out-trade-no/{combine_out_trade_no}/close.yaml
- ./spec/ecommerce/applyments/out-request-no/{out_request_no}.yaml
- ./spec/ecommerce/applyments/stub.yaml
- ./spec/ecommerce/applyments/{applyment_id}.yaml
- ./spec/ecommerce/fund/balance/{sub_mchid}.yaml
- ./spec/ecommerce/fund/enddaybalance/{sub_mchid}.yaml
- ./spec/ecommerce/fund/withdraw.yaml
- ./spec/ecommerce/fund/withdraw/{withdraw_id}.yaml
- ./spec/ecommerce/profitsharing/finish-order.yaml
- ./spec/ecommerce/profitsharing/orders.yaml
- ./spec/ecommerce/profitsharing/receivers/add.yaml
- ./spec/ecommerce/profitsharing/receivers/delete.yaml
- ./spec/ecommerce/profitsharing/returnorders.yaml
- ./spec/ecommerce/refunds/apply.yaml
- ./spec/ecommerce/refunds/id/{refund_id}.yaml
- ./spec/ecommerce/refunds/out-refund-no/{out_refund_no}.yaml
- ./spec/ecommerce/subsidies/cancel.yaml
- ./spec/ecommerce/subsidies/create.yaml
- ./spec/ecommerce/subsidies/return.yaml
- ./spec/goldplan/merchants/changecustompagestatus.yaml
- ./spec/goldplan/merchants/changegoldplanstatus.yaml
- ./spec/marketing/busifavor/callbacks.yaml
- ./spec/marketing/busifavor/coupons/associate.yaml
- ./spec/marketing/busifavor/coupons/disassociate.yaml
- ./spec/marketing/busifavor/coupons/use.yaml
- ./spec/marketing/busifavor/coupons/{card_id}/send.yaml
- ./spec/marketing/busifavor/stocks.yaml
- ./spec/marketing/busifavor/stocks/{stock_id}.yaml
- ./spec/marketing/busifavor/stocks/{stock_id}/couponcodes.yaml
- ./spec/marketing/busifavor/users/{openid}/coupons.yaml
- ./spec/marketing/busifavor/users/{openid}/coupons/{coupon_code}/appids/{appid}.yaml
- ./spec/marketing/favor/callbacks.yaml
- ./spec/marketing/favor/coupon-stocks.yaml
- ./spec/marketing/favor/media/image-upload.yaml
- ./spec/marketing/favor/stocks.yaml
- ./spec/marketing/favor/stocks/{stock_id}.yaml
- ./spec/marketing/favor/stocks/{stock_id}/items.yaml
- ./spec/marketing/favor/stocks/{stock_id}/merchants.yaml
- ./spec/marketing/favor/stocks/{stock_id}/pause.yaml
- ./spec/marketing/favor/stocks/{stock_id}/refund-flow.yaml
- ./spec/marketing/favor/stocks/{stock_id}/restart.yaml
- ./spec/marketing/favor/stocks/{stock_id}/start.yaml
- ./spec/marketing/favor/stocks/{stock_id}/use-flow.yaml
- ./spec/marketing/favor/users/{openid}/coupons.yaml
- ./spec/marketing/favor/users/{openid}/coupons/{coupon_id}.yaml
- ./spec/marketing/partnerships/build.yaml
- ./spec/marketing/partnerships/terminate.yaml
- ./spec/marketing/paygiftactivity/activities.yaml
- ./spec/marketing/paygiftactivity/activities/{activity_id}.yaml
- ./spec/marketing/paygiftactivity/activities/{activity_id}/goods.yaml
- ./spec/marketing/paygiftactivity/activities/{activity_id}/merchants.yaml
- ./spec/marketing/paygiftactivity/activities/{activity_id}/merchants/add.yaml
- ./spec/marketing/paygiftactivity/activities/{activity_id}/merchants/delete.yaml
- ./spec/marketing/paygiftactivity/activities/{activity_id}/terminate.yaml
- ./spec/marketing/paygiftactivity/unique-threshold-activity.yaml
- ./spec/merchant-service/complaint-notifications.yaml
- ./spec/merchant-service/complaints.yaml
- ./spec/merchant/fund/balance/{account_type}.yaml
- ./spec/merchant/fund/dayendbalance/{account_type}.yaml
- ./spec/merchant/fund/withdraw.yaml
- ./spec/merchant/fund/withdraw/bill-type/{bill_type}.yaml
- ./spec/merchant/fund/withdraw/out-request-no/{out_request_no}.yaml
- ./spec/merchant/media/upload.yaml
- ./spec/merchant/media/video_upload.yaml
- ./spec/pay/partner/transactions/app.yaml
- ./spec/pay/partner/transactions/h5.yaml
- ./spec/pay/partner/transactions/id/{transaction_id}.yaml
- ./spec/pay/partner/transactions/jsapi.yaml
- ./spec/pay/partner/transactions/native.yaml
- ./spec/pay/partner/transactions/out-trade-no/{out_trade_no}.yaml
- ./spec/pay/partner/transactions/out-trade-no/{out_trade_no}/close.yaml
- ./spec/pay/transactions/app.yaml
- ./spec/pay/transactions/h5.yaml
- ./spec/pay/transactions/id/{transaction_id}.yaml
- ./spec/pay/transactions/jsapi.yaml
- ./spec/pay/transactions/native.yaml
- ./spec/pay/transactions/out-trade-no/{out_trade_no}.yaml
- ./spec/pay/transactions/out-trade-no/{out_trade_no}/close.yaml
- ./spec/payscore/serviceorder.yaml
- ./spec/payscore/serviceorder/direct-complete.yaml
- ./spec/payscore/serviceorder/{out_order_no}/cancel.yaml
- ./spec/payscore/serviceorder/{out_order_no}/complete.yaml
- ./spec/payscore/serviceorder/{out_order_no}/modify.yaml
- ./spec/payscore/serviceorder/{out_order_no}/pay.yaml
- ./spec/payscore/serviceorder/{out_order_no}/sync.yaml
- ./spec/payscore/user-service-state.yaml
- ./spec/payscore/users/{openid}/permissions/{service_id}/terminate.yaml
- ./spec/smartguide/guides.yaml
- ./spec/smartguide/guides/{guide_id}.yaml
- ./spec/smartguide/guides/{guide_id}/assign.yaml

**注**: 有俩异形，这里就不表了，官方后期会修，等修完了我再更新同步。

## 后记

其实吧，两周前，这个工具的高级用法已经做完了，即，可以作为官方接口开发者工具使用，思来想去还是不开源这个用法了，这里仅提供思路，供有需要的同学自行钻研吧。

1. 修改 `swagger.yaml` 中的 `baseURL` 为同域地址，并转发 `/v3` 至代理工具；
2. 可选修改 `schemas` 中的 `https` 为 `http`;
3. 做个代理工具（nodejs大概40行代码就搞完了），把界面提交的数据做签名，代理至官方接口地址，返回结果给界面；

然后就没有然后了。
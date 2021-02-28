---
title: "支付宝page类网关调用，不得不注意的charset参数赋值位置"
date: 2020-10-22T08:21:26+08:00
keywords: [charset alipay openapi cli page类网关接口应该注意charset参数位置]
description: "The Charset Key Point of the Alipay's OpenAPI Gateway Requests, 调用OpenAPI page类网关接口应该注意charset参数位置"
tags: []
categories: [javascript]
author: "James"
summary: "调用OpenAPI page类网关接口应该注意charset参数位置以及官方可改进增强的点"
---

以测试用例覆盖来看，调用OpenAPI page类网关接口应该注意charset参数位置以及官方可改进增强的点

是的，你没看错，这个仍旧是 [nodejs SDK](/posts/yet-another-alipay-openapi-smart-development-kit-in-nodejs-env/) 的一篇展开文章，page类的测试用例源码可见[alipay.trade.wap.pay.test.js](https://github.com/TheNorthMemory/whats-alipay/blob/master/tests/openapi/alipay.trade.wap.pay.test.js)。

在做这个用例覆盖的时候，一开始以为这是官方的一个BUG，后来翻腾了好几遍文档，最后从 [这里](https://opensupport.alipay.com/support/helpcenter/192/201602472810) 翻到一句话如下：

> 请确认charset参数放在了URL查询字符串中且各参数值使用charset参数指示的字符集编码

豁然明白，官方同学是知道有这么个问题的，只是我的这个用例展开不是验签出错，而是在支付确认页，中文subject显示成了"昆金铐"——编码错乱的标志。

我们来看下此用例发生条件及过程：

1. 网关地址是: `https://openapi.alipay.com/gateway.do`；
2. 接口数据以表单形式以 `POST` + `utf-8` 编码提交；
3. 网关 `302` 至 `https://mclient.alipay.com/cashier/mobilepay.htm` 承接页；
4. 承接页 做唤醒支付宝APP尝试，不成功自动转入 `/h5Continue.htm` 登录状态校验页；
5. `h5Continue` 页做完校验后，已登录则转入支付确认页；

这个时候，在第五步的时候，即出现了著名的"昆金铐"，原则上来说，如果是眼浅出错，则在第三步的时候，即转入 `40002:invalid-signature` 页面，不会进入 承接页及登录校验页，说明这个时候，其实服务端验签按`utf-8`已经是验签通过了，然而在第四转第五步时，却出现了编码错乱，这个一方面是因为官方默认的编码是gbk，另一方面我们不妨深挖一下这个转入支付确认页逻辑。

我们通过此 nodejs SDK，以命令行的形式，来提交这个page类表单看看，承接页的源码(片段)是个什么样：

```javascript
'<div class="am-content">\r\n' +
'    <input type="hidden" name="params" value="{&quot;server_param&quot;:&quot;emlkPTQ5O25kcHQ3ZmMwOTtjYz15&quot;}" />\r\n' +
'    <div class="result">\r\n' +
'        <div class="result-logo"></div>\r\n' +
'        <div class="result-title">正在尝试打开支付宝客户端</div>\r\n' +
'                  <div class="result-tips">1.如果未安装支付宝APP，请先 <a class="J-h5Download" href="#">点这里下载支付宝APP</a> 并完成安装，再点击「使用支付宝APP付款」；</div>\r\n' +
'          <div class="result-tips">2.如果无法打开支付宝APP，请点击「继续浏览器付款」；</div>\r\n' +
'          <div class="result-tips" style="margin-bottom: 40px;">3.如果你已完成付款，请点击「已完成付款」；</div>\r\n' +
'          <div class="result-botton"><a class="J-startapp am-button am-button-blue" href="#">使用支付宝APP付款</a></div>\r\n' +
'          <div class="result-botton"><a class="J-h5pay am-button am-button-white" href="#">继续浏览器付款</a></div>\r\n' +
'                <div class="result-botton"><a class="J-complete am-button am-button-white" href="#">已完成付款</a></div>\r\n' +
'    </div>\r\n' +
'</div>\r\n' +
"    $('.J-h5pay').on('click', function(e){\r\n" +
"      var _url = '/h5Continue.htm?h5_route_token=RZ52N4oP3VgS4pukmk2jPbXlflxVsXmobilecashierRZ54&awid=RZ54RWrYV33dZbX9cIbtVw8KL6sUY3mobileclientgwRZ54';\r\n" +
'      if(window.json_ua){\r\n' +
"        _url += '&ua=' + encodeURIComponent(window.json_ua + '');\r\n" +
'      }\r\n' +
'      window.location.href = _url\r\n' +
'    })\r\n' +
'            Zepto.ajax({\r\n' +
"                type: 'post',\r\n" +
"                url: '/h5/h5RoutePayResultQuery.json?operateType=1&h5_route_token=RZ52N4oP3VgS4pukmk2jPbXlflxVsXmobilecashierRZ54',\r\n" +
'                data:{\r\n' +
"                    '_input_charset': 'utf-8',\r\n" +
"                    'params': $('input[name=params]').val(),\r\n" +
"                    'session': 'RZ52N4oP3VgS4pukmk2jPbXlflxVsXmobilecashierRZ54',\r\n" +
"                    'r': Math.floor(Math.random()*1e13)\r\n" +
'                },\r\n' +
'                timeout: 30000,\r\n' +
"                dataType: 'json',\r\n" +
```

页面逻辑大致上是说，以`alipays://`, `alipay://` schema的形式如果唤醒了APP，这个页面即停在这个承接页上，页面每30秒做异步轮询查询付款状态；如果没唤醒或者6秒计时结束，则转入 `/h5Continue.htm` 页面。

这里就有些值得细品的地方了，即，每30秒查询付款状态时，查询参数带入了 `_input_charset: 'utf-8'` 这么个参数，字面意思理解就是，输入编码格式是`utf-8`，拔特，转入 `/h5Continue.htm` 却没带上这个参数，以至于后续页面落入了 `gbk` 默认编码这个锅里。

本人没有仔细分析为什么没有带入这个参数，还是回到一开始 「签名验签」常见问题上，官方应该已经注意到了这个问题，给予开发者的建议是，需要把 `charset` 带入至 `URL` 中，那。。。就这么办吧～

所以，当你阅读至此，应该明白了： 在调用支付宝网关page类服务时，有一个参数是当前「必须」带入至URL上的，即`charset`，这个目前是解决编码错乱问题的不二法门。

稍微展开了说，`mobilepay.htm`既然已经感知到了输入编码是`utf-8`了，就应该加强一下转入`/h5Continue.htm`及后续页面的编码一致性问题，这是嫩给官方的建议。同时为减少广大脑力劳动者脱发风险，请收下俺的膝盖 Orz...

`whats-aliapy` nodejs sdk 已发布至 v0.1.0，同时做了如下三项改进：

  - 调整 `同步应答验签` 逻辑，遵从本SDK约定，只要能从应答返回中解析出有效负载，即仅返回负载；
  - 新增 `异步通知消息` 验签文档示例函数；
  - 不再可选依赖 `form-data`，以内置 `Form` 类为主；

`cli` 及 `sdk` 本身，对网关返回的应用级异常JSON输出，数据感知能力变得更流畅。欢迎体验，欢迎 **Star** 项目地址 https://github.com/TheNorthMemory/whats-alipay

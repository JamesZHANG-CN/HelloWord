---
title: "AES-GCM在Node10解密消息时崩溃问题研究及解决办法"
date: 2021-03-13T21:14:44+08:00
keywords: [wechatpay nodejs aes-256-gcm auth-tag]
description: "AES-GCM在Node10解密消息时有可能崩溃 Invalid Aes Gcm Decipher auth_tag Makes Crash on Node10"
tags: []
categories: [javascript, wechat]
author: "James"
summary: "AES-GCM在Node10解密消息时有可能崩溃，使用云开发环境的同学需要进来看一下"
---

## 背景知识

`AES-GCM`是微信支付APIv3的加解密方案之一，定义可见[rfc5116](https://tools.ietf.org/html/rfc5116)，v3使用的是`aead_aes_256_gcm`。稍微补充一个`aead`的的描述，`aead`加密方式与其他对称加密方式主要不同的地方就是：

**它每一段密文必定有对应的校验码，通过核对校验码来判断密文是否完整**。

[APIv3回调通知和平台证书下载](https://pay.weixin.qq.com/wiki/doc/apiv3/wechatpay/wechatpay4_2.shtml)文档上有介绍`AES-GCM`的使用场景。nodejs原生`crypto`模块，在处理`GCM`模式解密时，从变更历史上看，`Node11`加入了强制校验`auth_tag`(authentication tag)长度规则，`Node10`目前全系列还没有合并这个向前兼容规则，详情可见 https://github.com/nodejs/node/pull/20039 。

## 测试代码

先上一段测试用js代码，来复现 nodejs#20039 上连带反馈的问题：

```js
const crypto = require('crypto')
const decrypt = (ciphertext, key, iv, aad = '') => {
  const buf = Buffer.from(ciphertext, 'base64')
  const tag = buf.slice(-16)
  const payload = buf.slice(0, -16)

  const decipher = crypto.createDecipheriv(
    'aes-256-gcm', key, iv
  ).setAuthTag(tag).setAAD(Buffer.from(aad))

  return Buffer.concat([
    decipher.update(payload, 'hex'),
    decipher.final()
  ]).toString('utf8')
}

const mockupIv = 'abcdef0123456789'
const mockupKey = 'abcdef0123456789abcdef0123456789'

try {
  decrypt('', mockupKey, mockupIv)
} catch {}
```

上述代码，在node10.15-10.24，均抛出如下不可捕获的错误(fatal error)，程序会直接挂掉，在12-15之间，可以正常运行。

## 错误日志
类似如下:

```
node[97219]: ../src/node_crypto.cc:3047:CipherBase::UpdateResult node::crypto::CipherBase::Update(const char *, int, unsigned char **, int *): Assertion `MaybePassAuthTagToOpenSSL()' failed.
 1: 0x100d69661 node::Abort() (.cold.1) [/Users/james/.nvm/versions/node/v10.24.0/bin/node]
 2: 0x10003aeb4 node_module_register [/Users/james/.nvm/versions/node/v10.24.0/bin/node]
 3: 0x100039fb9 node::AddEnvironmentCleanupHook(v8::Isolate*, void (*)(void*), void*) [/Users/james/.nvm/versions/node/v10.24.0/bin/node]
 4: 0x100112fae node::StringBytes::InlineDecoder::Decode(node::Environment*, v8::Local<v8::String>, v8::Local<v8::Value>, node::encoding) [/Users/james/.nvm/versions/node/v10.24.0/bin/node]
 5: 0x1001119dc node::crypto::CipherBase::Update(v8::FunctionCallbackInfo<v8::Value> const&) [/Users/james/.nvm/versions/node/v10.24.0/bin/node]
 6: 0x1002386c3 v8::internal::FunctionCallbackArguments::Call(v8::internal::CallHandlerInfo*) [/Users/james/.nvm/versions/node/v10.24.0/bin/node]
 7: 0x100237bae v8::internal::MaybeHandle<v8::internal::Object> v8::internal::(anonymous namespace)::HandleApiCallHelper<false>(v8::internal::Isolate*, v8::internal::Handle<v8::internal::HeapObject>, v8::internal::Handle<v8::internal::HeapObject>, v8::internal::Handle<v8::internal::FunctionTemplateInfo>, v8::internal::Handle<v8::internal::Object>, v8::internal::BuiltinArguments) [/Users/james/.nvm/versions/node/v10.24.0/bin/node]
 8: 0x10023728a v8::internal::Builtin_Impl_HandleApiCall(v8::internal::BuiltinArguments, v8::internal::Isolate*) [/Users/james/.nvm/versions/node/v10.24.0/bin/node]
 9: 0x37d3d8d5bf3d
10: 0x37d3d8d118d5
11: 0x37d3d8d0a5c3
12: 0x37d3d8d118d5
13: 0x37d3d8d0a5c3
[1]    97218 abort      npm test
```

上述错误日志，发生在我本地的`Node10`环境中。我花了几个小时，翻了好几遍github issues，最后找到了 nodejs#20039 pull requests，通读下来并反复测试了10.15-10.24版本，均无法正常捕获，这应该是上述pr没合并至`Node10`系列所致。

## 产生条件

稍微分析一下，可能产生致命错误的条件：

**密文为空字符串时，程序会崩**

密文为 `Cg==`(base64空字符串) CLI会有 **Warning DEP0090** 弹出

 > (node:987) [DEP0090] DeprecationWarning: Permitting authentication tag lengths of 1 bytes is deprecated. Valid GCM tag lengths are 4, 8, 12, 13, 14, 15, 16.

[微信支付官方文档在解密示例代码](https://pay.weixin.qq.com/wiki/doc/apiv3/wechatpay/wechatpay4_2.shtml) 常量定义了这个`auth_tag`长度为128位16字节，匹配rfc5116规范并且取的是最大值。

这下问题来了，万一无法正常获取到待解密字符串或者获取到的是空字符串，`GCM`模式校验码位又必须是16字节，业务逻辑又强依赖解密后字符串（**验签证书是v3通讯强依赖**）这崩掉了，着急上火的可真就是摊上事儿了！

## 向前兼容方案

找到问题关键点，那就打个业务逻辑补丁：**应用端，对输入待解密字符串，做长度校验，长度为0的，不进入解密函数**；或者可以采用如下**向前兼容js patch补丁**：

```diff
-    ).setAuthTag(tag).setAAD(Buffer.from(aad))
+    )
+
+    // Restrict valid GCM tag length, patches for Node < 11.0.0
+    // more @see https://github.com/nodejs/node/pull/20039
+    const tagLen = tag.length
+    if (tagLen > 16 || (tagLen < 12 && tagLen != 8 && tagLen != 4)) {
+      let backport = new TypeError(`Invalid authentication tag length: ${tagLen}`)
+      backport.code = 'ERR_CRYPTO_INVALID_AUTH_TAG'
+      throw backport
+    }
+    decipher.setAuthTag(tag).setAAD(Buffer.from(aad))
```

上述代码取自 [wechatpay-axios-plugin@aa36a56](https://github.com/TheNorthMemory/wechatpay-axios-plugin/commit/aa36a56d80dfb8f5eb0c45d91b352c0d30ae9852)，也已随源码用例覆盖`Node10`-`Node15`版本，均达预期，可安全使用。

## 可能的影响面

小程序云开发标配目前是`Node10`，不清楚云开发团队在处理`消息通知及关键信息解密`时，是否采用的是轻量化如nodejs原生`crypto`这样的解决方案，这个就需要云产品团队相关的同学进来看看，评估一下有无风险点了。

对自主对接云开发的开发者来说，**建议尽快给打下业务逻辑补丁或者程序解密补丁**，避免不可预期的错误发生(**虽然极小概率，但支付的事，可真不是小事儿**)。

## 题外话

建议云开发平台，能够升级一下`Node10`至最新`lts`运行时，一并建议能同时支持`Node12`、`Node14`运行时。

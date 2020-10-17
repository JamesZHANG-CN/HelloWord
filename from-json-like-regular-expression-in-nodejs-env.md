---
title: "Using RegExp to parse the text(JSON like) in Nodejs Env"
date: 2020-09-22T12:21:06+08:00
lastmod: 2020-10-17T15:45:06+08:00
keywords: ["fromJsonLike", "RegExp"]
description: "不同开发语言对unicode及slash等JSON字符串处理上会存在差异，以下方法是使用正则表达式，按JSON部分规范来抽取所用载核，目前可完美兼容转译后到源数据。"
tags: []
categories: ["javascript"]
author: "James"
summary: "不同开发语言对unicode及slash等JSON字符串处理上会存在差异，以下方法是使用正则表达式，按JSON部分规范来抽取所用载核，目前可完美兼容转译后到源数据。"
---

支付宝OpenAPI返回的是JSON字符串，不同开发语言对unicode及slash等处理上会存在差异。nodejs在处理这个JSON时，用`JSON.stringify(JSON.parese(data))​`方法可能奏效，不过优解方案是对源字符串做处理获取， 使用正则表达式，按JSON部分规范来抽取所用载核，对于pretty格式化美化后的JSON，同样奏效。

```javascript
var fromJsonLike = (source, placeholder = '(?<ident>[a-z](?:[a-z_])+_response)') => {
  const maybe = `(?:[\\r|\\n|\\s|\\t]*)`
  const pattern = new RegExp(
    `^\\{${maybe}"${placeholder}"${maybe}:${maybe}"?(?<payload>.*?)"?${maybe}` +
    `(?:,)?${maybe}(?:"sign"${maybe}:${maybe}"(?<sign>[^"]+)"${maybe})?\\}${maybe}$`,
    'm'
  )
  const {groups: {ident, payload, sign}} = (source || '').match(pattern) || {groups: {}}

  return {ident, payload, sign}
}
```

Case 1 input: JSON 编码器，对URL的slash(/)做了转译 \/，测试样本数据如下：

```javascript
var test = '{"alipay_offline_material_image_upload_response":{"code":"10000","msg":"Success","image_id":"Zp1Nm6FDTZaEuSSniGd5awAAACMAAQED","image_url":"http:\\/\\/oalipay-dl-django.alicdn.com\\/rest\\/1.0\\/image?fileIds=Zp1Nm6FDTZaEuSSniGd5awAAACMAAQED&zoom=original"},"sign":"P8xrBWqZCUv11UrEBjhQ4Sk3hyj4607qehO2VbKIS0hWa4U+NeLlOftqTyhGv+x1lzfqN590Y/8CaNIzEEg06FiNWJlUFM/uEFJLzSKGse4MjHbblpiSzI3eCV5RzxH26wZbEd9wyVYYi0pHFBf35UrBva47g7b5EuKCHfoVA95/zin9fAyb3xhhiHhmfGaWIDV/1LmE2vtqtOHQnISbY/deC71U614ySZ3YB97ws8npCcCJ+tgZvhHPkMRGvmyYPCRDB/aIN/sKDSLtfPp0u8DxE8pHLvCHm3wR84MQxqNbKgpd8NTKNvH+obELsbCrqPhjW7qI48634qx6enDupw=="}'
```

调用函数：

```javascript
console.info(fromJsonLike(test))
```

输出如下，可见，对转译的slash未做更新变化，而 `JSON.stringify(JSON.parese(data))​` 会抹掉这个转译。

```javascript
{
  ident: 'alipay_offline_material_image_upload_response',
  payload: '{"code":"10000","msg":"Success","image_id":"Zp1Nm6FDTZaEuSSniGd5awAAACMAAQED","image_url":"http:\\/\\/oalipay-dl-django.alicdn.com\\/rest\\/1.0\\/image?fileIds=Zp1Nm6FDTZaEuSSniGd5awAAACMAAQED&zoom=original"}',
  sign: 'P8xrBWqZCUv11UrEBjhQ4Sk3hyj4607qehO2VbKIS0hWa4U+NeLlOftqTyhGv+x1lzfqN590Y/8CaNIzEEg06FiNWJlUFM/uEFJLzSKGse4MjHbblpiSzI3eCV5RzxH26wZbEd9wyVYYi0pHFBf35UrBva47g7b5EuKCHfoVA95/zin9fAyb3xhhiHhmfGaWIDV/1LmE2vtqtOHQnISbY/deC71U614ySZ3YB97ws8npCcCJ+tgZvhHPkMRGvmyYPCRDB/aIN/sKDSLtfPp0u8DxE8pHLvCHm3wR84MQxqNbKgpd8NTKNvH+obELsbCrqPhjW7qI48634qx6enDupw=='
}
```

Case 2 input: pretty格式化美化后的JSON字符串，测试样本如下：

```javascript
var test = `{"alipay_offline_material_image_upload_response"
  :
  {"code":"10000","msg":"Success","image_id":"akGwYYaFTai3r1uB0ww-1QAAACMAAQQD","image_url":"https:\\/\\/oalipay-dl-django.alicdn.com\\/rest\\/1.0\\/image?fileIds=akGwYYaFTai3r1uB0ww-1QAAACMAAQQD&zoom=original"},
  "sign"    :    "PAb3IueJzGe/pU/fRAPjwIs543Kc/A6D0rz03AMejwr8h8rDc6FhDJDnz3fGLDdQP7ctjtQwwJW3pmdZZcGmp4lb/5YYgtoK6McjnRr4ER/raJLYn1IbpzkowhGow2esA/XeDblIAYUbZjU6ts0IqNncrZrCknDWHpaZXwGuaU7CUBk74xBeMeja7rEEkFlm9MRtiQNYnum/cGVtcDv/aQ8KkPyAD58oJiAzoXv0R6jFhlZtAWv+M0SaOlhTpZh1K6wlP+1Umiqdvqbc1oWdfpv75a+lGTkGHMy8K7/bnAGm20IRsisSv1B5rpJyeGfrVf6tb4MZ7vG4w0rS0c2hfA=="}`;
```

调用函数：

```javascript
console.info(fromJsonLike(test))
```

输出同样奏效

```javascript
{
  ident: 'alipay_offline_material_image_upload_response',
  payload: '{"code":"10000","msg":"Success","image_id":"akGwYYaFTai3r1uB0ww-1QAAACMAAQQD","image_url":"https:\\/\\/oalipay-dl-django.alicdn.com\\/rest\\/1.0\\/image?fileIds=akGwYYaFTai3r1uB0ww-1QAAACMAAQQD&zoom=original"}',
  sign: 'PAb3IueJzGe/pU/fRAPjwIs543Kc/A6D0rz03AMejwr8h8rDc6FhDJDnz3fGLDdQP7ctjtQwwJW3pmdZZcGmp4lb/5YYgtoK6McjnRr4ER/raJLYn1IbpzkowhGow2esA/XeDblIAYUbZjU6ts0IqNncrZrCknDWHpaZXwGuaU7CUBk74xBeMeja7rEEkFlm9MRtiQNYnum/cGVtcDv/aQ8KkPyAD58oJiAzoXv0R6jFhlZtAWv+M0SaOlhTpZh1K6wlP+1Umiqdvqbc1oWdfpv75a+lGTkGHMy8K7/bnAGm20IRsisSv1B5rpJyeGfrVf6tb4MZ7vG4w0rS0c2hfA=='
}
```

Case 3 input: 手工拼接的JSON串，对 `sign` 做了断行处理，样本如下：

```javascript
var test = `{"alipay_offline_material_image_upload_response":{"code":"10000","msg":"Success","image_id":"akGwYYaFTai3r1uB0ww-1QAAACMAAQQD","image_url":"https:\\/\\/oalipay-dl-django.alicdn.com\\/rest\\/1.0\\/image?fileIds=akGwYYaFTai3r1uB0ww-1QAAACMAAQQD&zoom=original"},
  "sign":"PAb3IueJzGe/pU/fRAPjwIs543Kc/A6D0rz03AMejwr8h8rDc6FhDJDnz3fGLDdQP7ctjtQwwJW3pmdZZcGmp4lb/5YYgtoK6McjnRr4ER/raJLYn1IbpzkowhGow2esA/XeDblIAYUbZjU6ts0IqNncrZrCknDWHpaZXwGuaU7CUBk74xBeMeja7rEEkFlm9MRtiQNYnum/cGVtcDv/aQ8KkPyAD58oJiAzoXv0R6jFhlZtAWv+M0SaOlhTpZh1K6wlP+1Umiqdvqbc1oWdfpv75a+lGTkGHMy8K7/bnAGm20IRsisSv1B5rpJyeGfrVf6tb4MZ7vG4w0rS0c2hfA=="}`;
```

调用函数：

```javascript
console.info(fromJsonLike(test))
```

输出同样奏效

```javascript
{
  ident: 'alipay_offline_material_image_upload_response',
  payload: '{"code":"10000","msg":"Success","image_id":"akGwYYaFTai3r1uB0ww-1QAAACMAAQQD","image_url":"https:\\/\\/oalipay-dl-django.alicdn.com\\/rest\\/1.0\\/image?fileIds=akGwYYaFTai3r1uB0ww-1QAAACMAAQQD&zoom=original"}',
  sign: 'PAb3IueJzGe/pU/fRAPjwIs543Kc/A6D0rz03AMejwr8h8rDc6FhDJDnz3fGLDdQP7ctjtQwwJW3pmdZZcGmp4lb/5YYgtoK6McjnRr4ER/raJLYn1IbpzkowhGow2esA/XeDblIAYUbZjU6ts0IqNncrZrCknDWHpaZXwGuaU7CUBk74xBeMeja7rEEkFlm9MRtiQNYnum/cGVtcDv/aQ8KkPyAD58oJiAzoXv0R6jFhlZtAWv+M0SaOlhTpZh1K6wlP+1Umiqdvqbc1oWdfpv75a+lGTkGHMy8K7/bnAGm20IRsisSv1B5rpJyeGfrVf6tb4MZ7vG4w0rS0c2hfA=='
}
```

Case 4 input: 手工拼接的，类JSON字符串，末尾多了逗号(,)

```javascript
var test = `{"error_response":{"code":"40002","msg":"Invalid Arguments","sub_code":"isv.code-invalid","sub_msg":"授权码code无效"},}`
```

调用函数：


```javascript
console.info(fromJsonLike(test))
```

输出同样奏效

```javascript
{
  ident: 'error_response',
  payload: '{"code":"40002","msg":"Invalid Arguments","sub_code":"isv.code-invalid","sub_msg":"授权码code无效"}',
  sign: undefined
}
```

Case 5 input: 手工拼接的，类JSON字符串，末尾多了逗号(,)并且有回车断行

```javascript
var test = `{"error_response":
  {"code":"40002","msg":"Invalid Arguments","sub_code":"isv.code-invalid","sub_msg":"授权码code无效"},
  }`
```

```javascript
console.info(fromJsonLike(test))
```

```javascript
{
  ident: 'error_response',
  payload: '{"code":"40002","msg":"Invalid Arguments","sub_code":"isv.code-invalid","sub_msg":"授权码code无效"}',
  sign: undefined
}
```

Case 6 input: 标准JSON对象

```javascript
var test = `{"error_response":{"code":"40002","msg":"Invalid Arguments","sub_code":"isv.code-invalid","sub_msg":"授权码code无效"}}`
```

```javascript
console.info(fromJsonLike(test))
```

```javascript
{
  ident: 'error_response',
  payload: '{"code":"40002","msg":"Invalid Arguments","sub_code":"isv.code-invalid","sub_msg":"授权码code无效"}',
  sign: undefined
}
```

Case 7 input: 标准JSON对象，返回的内容是aes加密后的base64密文

```javascript
var test = '{"alipay_open_auth_app_aes_set_response":"4AOYHE0rpPnRnghunsGo+mY02DzANFLwNJJCiHfrNh2oaB2pn33PwOEOvH8mjhkE3Wh/jR+3jHM9nvoFvOsY/SqZbZzamRg9Eh3VkRqOhSM=","sign":"abcde="}'
```

```javascript
console.info(fromJsonLike(test))
```

仅取到密文部分，不包括前、后双引号

```javascript
{
  ident: 'alipay_open_auth_app_aes_set_response',
  payload: '4AOYHE0rpPnRnghunsGo+mY02DzANFLwNJJCiHfrNh2oaB2pn33PwOEOvH8mjhkE3Wh/jR+3jHM9nvoFvOsY/SqZbZzamRg9Eh3VkRqOhSM=',
  sign: 'abcde='
}
```

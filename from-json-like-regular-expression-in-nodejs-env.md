---
title: "Alipay OpenAPI同步/异步验签，请注意编程语言对JSON处理的差异性"
date: 2020-09-22T12:21:06+08:00
lastmod: 2020-10-23T15:37:06+08:00
keywords: ["fromJsonLike", "RegExp"]
description: "不同开发语言对unicode及slash等JSON字符串处理上会存在差异，以下方法是使用正则表达式，按JSON部分规范来抽取所用载核，目前可完美兼容转译后到源数据。"
tags: []
categories: ["javascript"]
author: "James"
summary: "不同开发语言对unicode及slash等JSON字符串处理上会存在差异，以下方法是使用正则表达式，按JSON部分规范来抽取所用载核，目前可完美兼容转译后到源数据。"
---

支付宝OpenAPI返回的是JSON字符串，不同开发语言对unicode及slash等处理上会存在差异。按标准JSON规格文档来说，严格意义上对负载有歧义的字符是需要做转义，如斜杠(`/`)，不同编程语言对这个规格的处理上，有稍许差异，nodejs在处理这个JSON时，用`JSON.stringify(JSON.parese(data))​`方法可能奏效，不过优解方案是对源字符串做处理获取， 使用正则表达式，按JSON部分规范来抽取所用载核，对于pretty格式化美化后的JSON，同样奏效。例如从如下两个接口，返回的样本数据可从 [alipay.offline.material.image.upload.json](https://github.com/TheNorthMemory/whats-alipay/blob/master/tests/fixtures/alipay.offline.material.image.upload.json) 及 [alipay.offline.market.shop.category.query.json](https://github.com/TheNorthMemory/whats-alipay/blob/master/tests/fixtures/alipay.offline.market.shop.category.query.json) 来看，转义斜杠(`/`)是平台标准做法。

这转义斜杠如果不注意呀，很容易造成验签失败，因为`javascript`对JSON负载内斜杠不转义。。。所以啊，得老老实实对返回的源字符串做截取有效负载及签名值处理！不能任性的对源字符串，做 `JSON.parse` 然后取值再用去验签，偶尔验签真就能过，那是运气好；验签不通过那是大概率事件。

以正则表达式，对源返回字符串匹配获取，[原版函数](https://github.com/TheNorthMemory/whats-alipay/blob/master/lib/formatter.js#L77-L95)带注释如下：

```javascript
/**
 * Parse the `source` with given `placeholder`.
 *
 * @param {string} source - The inputs string.
 * @param {string} [placeholder] - The payload pattern.
 *
 * @returns {object} - `{ident, payload, sign}` object
 */
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

翻译下来就是，从 开头（允许有空格、换行、TAB等单个或多个字符存在），以有效负载标识规则 (`xxx_response`格式)，匹配至 `"sign"`位或者末位`}`，同时兼容负载是`aes`加密串的无感获取，正则规则中的 `(?<Name>x)` 是 `Named capturing group` 语法块，即对匹配到的内容做命名，以上函数包括型参`placeholder`默认值中的 `<ident>`，一共分组命名3个，即返回值对象，`payload`为有效负载，`sign`即验签串, 然后从这个函数里，拿这俩值去验签，`99.999%` 应该就没问题了(极小概率见如下`缺陷`)。

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

测试用例覆盖也开见 [formatter.test.js#L92-L120](https://github.com/TheNorthMemory/whats-alipay/blob/master/tests/lib/formatter.test.js#L92-L120)，重点覆盖了 `带转义斜杠的JSON`、`美化后JSON`，及 `非标准JSON`，应该没啥问题。

这里有个缺陷，就是，当返回的字符串签名标识`sign`不是在有效负载之后，那么就会取不到了；不过从源码阅读官方`easysdk`上来看，这种情形基本上不会出现，因为官方包对返回的类JSON字符串处理逻辑也是按照 `方法标识符(x_response)` + `签名标识(sign)` 处理的，可以安全使用。

最后，这个正则表达式，应该很容易翻译成其他开发语言版的，如果有同学翻译了，欢迎再此留个言，俺也就欣慰了。

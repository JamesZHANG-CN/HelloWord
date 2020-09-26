---
title: "Using RegExp to parse the text(JSON like) in Nodejs Env"
date: 2020-09-22T12:21:06+08:00
keywords: ["fromJsonLike", "RegExp"]
description: "Using RegExp to parse the text(JSON like) to standard JSON in Nodejs Env"
tags: []
categories: ["javascript"]
author: "James"
summary: "Practice of parse the text(JSON like) to standard JSON."
---

A castor to parse the text (JSON like) to standard JSON.

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

Case 1 input: JSON TEXT with escaped dash(/)

```javascript
var test = '{"alipay_offline_material_image_upload_response":{"code":"10000","msg":"Success","image_id":"Zp1Nm6FDTZaEuSSniGd5awAAACMAAQED","image_url":"http:\\/\\/oalipay-dl-django.alicdn.com\\/rest\\/1.0\\/image?fileIds=Zp1Nm6FDTZaEuSSniGd5awAAACMAAQED&zoom=original"},"sign":"P8xrBWqZCUv11UrEBjhQ4Sk3hyj4607qehO2VbKIS0hWa4U+NeLlOftqTyhGv+x1lzfqN590Y/8CaNIzEEg06FiNWJlUFM/uEFJLzSKGse4MjHbblpiSzI3eCV5RzxH26wZbEd9wyVYYi0pHFBf35UrBva47g7b5EuKCHfoVA95/zin9fAyb3xhhiHhmfGaWIDV/1LmE2vtqtOHQnISbY/deC71U614ySZ3YB97ws8npCcCJ+tgZvhHPkMRGvmyYPCRDB/aIN/sKDSLtfPp0u8DxE8pHLvCHm3wR84MQxqNbKgpd8NTKNvH+obELsbCrqPhjW7qI48634qx6enDupw=="}'
```

```javascript
console.info(fromJsonLike(test))
```

```javascript
{
  ident: 'alipay_offline_material_image_upload_response',
  payload: '{"code":"10000","msg":"Success","image_id":"Zp1Nm6FDTZaEuSSniGd5awAAACMAAQED","image_url":"http:\\/\\/oalipay-dl-django.alicdn.com\\/rest\\/1.0\\/image?fileIds=Zp1Nm6FDTZaEuSSniGd5awAAACMAAQED&zoom=original"}',
  sign: 'P8xrBWqZCUv11UrEBjhQ4Sk3hyj4607qehO2VbKIS0hWa4U+NeLlOftqTyhGv+x1lzfqN590Y/8CaNIzEEg06FiNWJlUFM/uEFJLzSKGse4MjHbblpiSzI3eCV5RzxH26wZbEd9wyVYYi0pHFBf35UrBva47g7b5EuKCHfoVA95/zin9fAyb3xhhiHhmfGaWIDV/1LmE2vtqtOHQnISbY/deC71U614ySZ3YB97ws8npCcCJ+tgZvhHPkMRGvmyYPCRDB/aIN/sKDSLtfPp0u8DxE8pHLvCHm3wR84MQxqNbKgpd8NTKNvH+obELsbCrqPhjW7qI48634qx6enDupw=='
}
```

Case 2 input: JSONLike TEXT with escaped dash(/) and spaces(seprate manually)

```javascript
var test = `{"alipay_offline_material_image_upload_response"
  :
  {"code":"10000","msg":"Success","image_id":"akGwYYaFTai3r1uB0ww-1QAAACMAAQQD","image_url":"https:\\/\\/oalipay-dl-django.alicdn.com\\/rest\\/1.0\\/image?fileIds=akGwYYaFTai3r1uB0ww-1QAAACMAAQQD&zoom=original"},
  "sign"    :    "PAb3IueJzGe/pU/fRAPjwIs543Kc/A6D0rz03AMejwr8h8rDc6FhDJDnz3fGLDdQP7ctjtQwwJW3pmdZZcGmp4lb/5YYgtoK6McjnRr4ER/raJLYn1IbpzkowhGow2esA/XeDblIAYUbZjU6ts0IqNncrZrCknDWHpaZXwGuaU7CUBk74xBeMeja7rEEkFlm9MRtiQNYnum/cGVtcDv/aQ8KkPyAD58oJiAzoXv0R6jFhlZtAWv+M0SaOlhTpZh1K6wlP+1Umiqdvqbc1oWdfpv75a+lGTkGHMy8K7/bnAGm20IRsisSv1B5rpJyeGfrVf6tb4MZ7vG4w0rS0c2hfA=="}`;
```

```javascript
console.info(fromJsonLike(test))
```

```javascript
{
  ident: 'alipay_offline_material_image_upload_response',
  payload: '{"code":"10000","msg":"Success","image_id":"akGwYYaFTai3r1uB0ww-1QAAACMAAQQD","image_url":"https:\\/\\/oalipay-dl-django.alicdn.com\\/rest\\/1.0\\/image?fileIds=akGwYYaFTai3r1uB0ww-1QAAACMAAQQD&zoom=original"}',
  sign: 'PAb3IueJzGe/pU/fRAPjwIs543Kc/A6D0rz03AMejwr8h8rDc6FhDJDnz3fGLDdQP7ctjtQwwJW3pmdZZcGmp4lb/5YYgtoK6McjnRr4ER/raJLYn1IbpzkowhGow2esA/XeDblIAYUbZjU6ts0IqNncrZrCknDWHpaZXwGuaU7CUBk74xBeMeja7rEEkFlm9MRtiQNYnum/cGVtcDv/aQ8KkPyAD58oJiAzoXv0R6jFhlZtAWv+M0SaOlhTpZh1K6wlP+1Umiqdvqbc1oWdfpv75a+lGTkGHMy8K7/bnAGm20IRsisSv1B5rpJyeGfrVf6tb4MZ7vG4w0rS0c2hfA=='
}
```

Case 3 input: JSONLike TEXT with escaped dash(/) and CRLF(seprate manually)

```javascript
var test = `{"alipay_offline_material_image_upload_response":{"code":"10000","msg":"Success","image_id":"akGwYYaFTai3r1uB0ww-1QAAACMAAQQD","image_url":"https:\\/\\/oalipay-dl-django.alicdn.com\\/rest\\/1.0\\/image?fileIds=akGwYYaFTai3r1uB0ww-1QAAACMAAQQD&zoom=original"},
  "sign":"PAb3IueJzGe/pU/fRAPjwIs543Kc/A6D0rz03AMejwr8h8rDc6FhDJDnz3fGLDdQP7ctjtQwwJW3pmdZZcGmp4lb/5YYgtoK6McjnRr4ER/raJLYn1IbpzkowhGow2esA/XeDblIAYUbZjU6ts0IqNncrZrCknDWHpaZXwGuaU7CUBk74xBeMeja7rEEkFlm9MRtiQNYnum/cGVtcDv/aQ8KkPyAD58oJiAzoXv0R6jFhlZtAWv+M0SaOlhTpZh1K6wlP+1Umiqdvqbc1oWdfpv75a+lGTkGHMy8K7/bnAGm20IRsisSv1B5rpJyeGfrVf6tb4MZ7vG4w0rS0c2hfA=="}`;
```

```javascript
console.info(fromJsonLike(test))
```

```javascript
{
  ident: 'alipay_offline_material_image_upload_response',
  payload: '{"code":"10000","msg":"Success","image_id":"akGwYYaFTai3r1uB0ww-1QAAACMAAQQD","image_url":"https:\\/\\/oalipay-dl-django.alicdn.com\\/rest\\/1.0\\/image?fileIds=akGwYYaFTai3r1uB0ww-1QAAACMAAQQD&zoom=original"}',
  sign: 'PAb3IueJzGe/pU/fRAPjwIs543Kc/A6D0rz03AMejwr8h8rDc6FhDJDnz3fGLDdQP7ctjtQwwJW3pmdZZcGmp4lb/5YYgtoK6McjnRr4ER/raJLYn1IbpzkowhGow2esA/XeDblIAYUbZjU6ts0IqNncrZrCknDWHpaZXwGuaU7CUBk74xBeMeja7rEEkFlm9MRtiQNYnum/cGVtcDv/aQ8KkPyAD58oJiAzoXv0R6jFhlZtAWv+M0SaOlhTpZh1K6wlP+1Umiqdvqbc1oWdfpv75a+lGTkGHMy8K7/bnAGm20IRsisSv1B5rpJyeGfrVf6tb4MZ7vG4w0rS0c2hfA=='
}
```

Case 4 input: invalid JSON TEXT

```javascript
var test = `{"error_response":{"code":"40002","msg":"Invalid Arguments","sub_code":"isv.code-invalid","sub_msg":"授权码code无效"},}`
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

Case 5 input: invalid JSON TEXT with CRLF(seprate manually)

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

Case 6 input: JSON TEXT

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

Case 7 input: JSON value as BASE64-CIPHER-TEXT

```javascript
var test = '{"alipay_open_auth_app_aes_set_response":"4AOYHE0rpPnRnghunsGo+mY02DzANFLwNJJCiHfrNh2oaB2pn33PwOEOvH8mjhkE3Wh/jR+3jHM9nvoFvOsY/SqZbZzamRg9Eh3VkRqOhSM=","sign":"abcde="}'
```

```javascript
console.info(fromJsonLike(test))
```

```javascript
{
  ident: 'alipay_open_auth_app_aes_set_response',
  payload: '4AOYHE0rpPnRnghunsGo+mY02DzANFLwNJJCiHfrNh2oaB2pn33PwOEOvH8mjhkE3Wh/jR+3jHM9nvoFvOsY/SqZbZzamRg9Eh3VkRqOhSM=',
  sign: 'abcde='
}
```

---
title: "加强版的微信支付APIv3媒体文件上传无依赖版NodeJS实现"
date: 2021-05-29T14:34:09+08:00
keywords: []
description: "NodeJS上无依赖版媒体文件上传 rfc1867/rfc2388 协议 multipart/form-data 实现。"
tags: []
toc: false
categories: [javascript, wechat]
description: "NodeJS上无依赖版媒体文件上传 rfc1867/rfc2388 协议 multipart/form-data 实现。"
author: "James"
---

## 设计思路

没啥思路，纯体力活，就是用NodeJS来实现一把媒体文件上传的 rfc1867/rfc2388 协议。 NodeJS上可以搜罗到的 `multipart` 实现，大部分是解释器，装载器引用最多的是`form-data`这个包，接续[上一版](https://developers.weixin.qq.com/community/develop/article/doc/000c24f0390ff8b5d91b2489059413)的实现，广度支持 nodejs http client，不挑食了。

`node-fetch`用来探测请求体是不是`FormData`函数片段：

```js
//refer: https://github.com/node-fetch/node-fetch/blob/master/src/utils/is.js#L47-L67

/**
 * Check if `obj` is a spec-compliant `FormData` object
 *
 * @param {*} object
 * @return {boolean}
 */
export function isFormData(object) {
    return (
        typeof object === 'object' &&
        typeof object.append === 'function' &&
        typeof object.set === 'function' &&
        typeof object.get === 'function' &&
        typeof object.getAll === 'function' &&
        typeof object.delete === 'function' &&
        typeof object.keys === 'function' &&
        typeof object.values === 'function' &&
        typeof object.entries === 'function' &&
        typeof object.constructor === 'function' &&
        object[Symbol.toStringTag] === 'FormData'
    );
}
```

`axios`用来探测代码片段:

```js
//refer: https://github.com/axios/axios/blob/master/lib/utils.js#L50-L58
/**
 * Determine if a value is a FormData
 *
 * @param {Object} val The value to test
 * @returns {boolean} True if value is an FormData, otherwise false
 */
function isFormData(val) {
  return (typeof FormData !== 'undefined') && (val instanceof FormData);
}
```

这个函数在浏览器环境上是可以的，但在NodeJS上是有问题的，`FormData`不是全局对象，探测失败，退而求其次，使用如下探测函数:

```js
// https://github.com/axios/axios/blob/master/lib/utils.js#L161-L169
/**
 * Determine if a value is a Stream
 *
 * @param {Object} val The value to test
 * @returns {boolean} True if value is a Stream, otherwise false
 */
function isStream(val) {
  return isObject(val) && isFunction(val.pipe);
}
```

## 实现

在coding过程中，遇到了一个比较大的挑战就是对 `delete` 的实现。按照MDN的文档介绍，delete是要删除同名所有值，这里啃了好久，也妥妥的搞定了。总体实现就是在[上一版](https://developers.weixin.qq.com/community/develop/article/doc/000c24f0390ff8b5d91b2489059413)的同步代码模型上，增加了10几个方法，覆盖住了[MDN FormData](https://developer.mozilla.org/zh-CN/docs/Web/API/FormData)上罗列的所有方法，并且支持了 stream 流动模式。

核心代码就是以下4个函数，均是对内置的`data` BufferList 进行编排：

```js
  append(name, value, filename = '') {
    const {
      data, dashDash, boundary, CRLF, indices,
    } = this;

    data.splice(...(data.length ? [-2, 1] : [0, 0, dashDash, boundary, CRLF]));
    indices.push([name, data.push(...this.formed(name, value, filename)) - 1]);
    data.push(CRLF, dashDash, boundary, dashDash, CRLF);

    return this;
  }

  formed(name, value, filename = '') {
    const { mimeTypes, CRLF, EMPTY } = this;
    const isBufferOrStream = Buffer.isBuffer(value) || (value instanceof ReadStream);
    return [
      Buffer.from(`Content-Disposition: form-data; name="${name}"${filename && isBufferOrStream ? `; filename="${filename}"` : ''}`),
      CRLF,
      ...(
        filename || isBufferOrStream
          ? [Buffer.from(`Content-Type: ${mimeTypes[extname(filename).substring(1).toLowerCase()] || 'application/octet-stream'}`), CRLF]
          : [EMPTY, EMPTY]
      ),
      CRLF,
      isBufferOrStream ? value : Buffer.from(String(value)),
    ];
  }

  set(name, value, filename = '') {
    if (this.has(name)) {
      this.indices.filter(([field]) => field === name).forEach(([field, index]) => {
        this.data.splice(index - 5, 6, ...this.formed(field, value, filename));
      });
    } else {
      this.append(name, value, filename);
    }

    return this;
  }

  delete(name) {
    this.indices = Object.values(this.indices.filter(([field]) => field === name).reduceRight((mapper, [, index]) => {
      this.data.splice(index - 8, 10);
      Reflect.deleteProperty(mapper, `${index}`);
      Object.entries(mapper).filter(([fixed]) => +fixed > index).forEach(([fixed, [field, idx]]) => {
        Reflect.set(mapper, `${fixed}`, [field, idx - 10]);
      });
      return mapper;
    }, this.indices.reduce((des, [field, value]) => {
      Reflect.set(des, value, [field, value]);
      return des;
    }, {})));

    if (!this.indices.length) {
      this.data.splice(0, this.data.length);
    }

    return this;
  }
```

## 应用

### 同步模式

```js
(new Multipart())
  .append('a', 1)
  .append('b', '2')
  .append('c', Buffer.from('31'))
  .append('d', JSON.stringify({}), 'any.json')
  .append('e', require('fs').readFileSync('/path/your/file.jpg'), 'file.jpg')
  .getBuffer();
```

### 异步(流动)模式

```js
(new Multipart())
  .append('f', require('fs').createReadStream('/path/your/file2.jpg'), 'file2.jpg')
  .pipe(require('fs').createWriteStream('./file3.jpg'));
```

### 示例上传代码

```js
const {readFileSync} = require('fs')

const {Wechatpay, Multipart, Hash: { sha256 }} = require('wechatpay-axios-plugin');

const wxpay = new Wechatpay({ mchid, secret, serial, privateKey, certs });

const file = readFileSync('./hellowechatpay.png');
const meta = {filename: 'hellowechatpay.png', sha256: sha256(file)};

const form = new Multipart();
form.append('file', file, 'hellowechatpay.png');
form.append('meta', JSON.stringify(meta), 'meta.json');

wxpay
  .v3.marketing.favor.media.imageUpload(form, { meta, headers: form.getHeaders() })
  .then(({data: { media_id }}) => media_id)
  .then(console.info)
  .catch(console.error);
```

更高级的stream流动模式下，一条龙从浏览器客户端至微信服务端的媒体直接上传，用js来实现，就变得很有可能了，这个留给喜欢钻研的同学搞一搞了。

## 最后

附上 [源码地址](https://github.com/TheNorthMemory/wechatpay-axios-plugin/blob/master/lib/multipart.js) 及 [测试用例地址](https://github.com/TheNorthMemory/wechatpay-axios-plugin/blob/master/tests/lib/multipart.test.js)，如果用着还不放心，可以再装一下 `form-data`包，通过   `new FormData` 来使用。

希望本文能解决你的困惑，如果喜欢，欢迎 **Star**。

## 应用

- [用了一天终于实现，云函数实现V3图片上传](https://developers.weixin.qq.com/community/pay/article/doc/00088ebe95036026e22c31c6f56c13)

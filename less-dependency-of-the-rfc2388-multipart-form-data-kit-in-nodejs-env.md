---
title: "rfc2388表单上传文件NodeJS无依赖版本实现"
date: 2020-10-17T14:26:34+08:00
lastmod: 2021-02-28T16:54:06+08:00
keywords: [RFC2388 multipart/form-data ES2015 file uploader pure javascript]
description: "受微信支付官方技术助手伙伴在某次问答亲测有效代码启发，以下ES2015代码为微信支付APIv3上传文件文档补充实现而做，供其他开发语言参考实现。"
tags: []
categories: []
categories: ["javascript"]
author: "James"
summary: "上传媒体文件API的是基于RFC2388协议，圈定的范围是，有大小限制的图片(2M)或者视频(5M)类型单文件，受微信支付官方技术助手伙伴在某次问答亲测有效代码启发，以下ES2015代码为微信支付APIv3图片、视频上传类接口文档补充nodejs纯语言无依赖版参考实现。"
---

上传媒体文件API的是基于 [RFC2388](https://www.ietf.org/rfc/rfc2388.html) 协议，圈定的范围是，有大小限制的图片(2M)或者视频(5M)类型单文件，按API接口标准，需要对文件的meta{filename,sha256}信息做数据签名，并且这个meta还须随请求载核一同发送给服务端，难就难在这里了，唉嘘～

官方文档小分队犯了一个错误，就是试图用文本语言来表达非字符内容，整得一众开发者迷途了，社区反馈波澜滔滔。这支小分队应该每人扣一个长鹅抱宠，捐给像俺这样努力帮扶开发者的贡献者（😄）。其实rfc2388标准上有写，这个协议就是扩展HTTP文本协议，用来传输非字符内容的，引述如下：

> multipart/form-data can be used for forms that are presented using representations other than HTML (spreadsheets, Portable Document Format, etc), and for transport using other means than electronic mail or HTTP. This document defines the representation of form values independently of the application for which it is used.

上述引述内容，提到3种文件，HTML文件还算是字符型文件，表格及PDF文件就已经算是二进制文件了，文件内容人类得借助专用软件翻译，才能转成可被识别内容（肉眼能直接识别的是大牛，不再此列）。受官方技术助手伙伴在某次问答亲测有效代码截图启示，特意又研读了几遍RFC协议，国庆档给抽出成无依赖ES2015版本，已内置于另一款著名支付产品SDK包中，亲测可用，以下版本是微信支付社区特供，单文件、无依赖，适合云开发集成使用。

废话说了一箩筐，还是上代码吧:


```javascript
const {extname} = require('path')
/**
 * Simple and lite of `multipart/form-data` implementation, most similar to `form-data`
 *
 * ```js
 * (new Form)
 *   .append('a', 1)
 *   .append('b', '2')
 *   .append('c', Buffer.from('31'))
 *   .append('d', JSON.stringify({}), 'any.json')
 *   .append('e', require('fs').readFileSync('/path/your/file.jpg'), 'file.jpg')
 *   .getBuffer()
 * ```
 */
class Form {
  /**
   * Create a `multipart/form-data` buffer container for the file uploading.
   *
   * @constructor
   */
  constructor() {
    Object.defineProperties(this, {
      /**
       * built-in mime-type mapping
       * @type {Object<string,string>}
       */
      mimeTypes: {
        value: {
          bmp: `image/bmp`,
          gif: `image/gif`,
          png: `image/png`,
          jpg: `image/jpeg`,
          jpe: `image/jpeg`,
          jpeg: `image/jpeg`,
          mp4: `video/mp4`,
          mpeg: `video/mpeg`,
          json: `application/json`,
        },
        configurable: false,
        enumerable: false,
        writable: true,
      },

      /**
       * @type {Buffer}
       */
      dashDash: {
        value: Buffer.from(`--`),
        configurable: false,
        enumerable: false,
        writable: false,
      },

      /**
       * @type {Buffer}
       */
      boundary: {
        value: Buffer.from(`${`-`.repeat(26)}${`0`.repeat(24).replace(/0/g, () => Math.random()*10|0)}`),
        configurable: false,
        enumerable: false,
        writable: false,
      },

      /**
       * @type {Buffer}
       */
      CRLF: {
        value: Buffer.from(`\r\n`),
        configurable: false,
        enumerable: false,
        writable: false,
      },

      /**
       * The Form's data storage
       * @type {array<Buffer>}
       */
      data: {
        value: [],
        configurable: false,
        enumerable: true,
        writable: true,
      },

      /**
       * The entities' value indices whose were in `this.data`
       * @type {Object<string, number>}
       */
      indices: {
        value: {},
        configurable: false,
        enumerable: true,
        writable: true,
      },
    })
  }

  /**
   * To retrieve the `data` buffer
   *
   * @return {Buffer} - The payload buffer
   */
  getBuffer() {
    return Buffer.concat([
      this.dashDash, this.boundary, this.CRLF,
      ...this.data.slice(0, -2),
      this.boundary, this.dashDash, this.CRLF,
    ])
  }

  /**
   * To retrieve the `Content-Type` multipart/form-data header
   *
   * @return {Object<string, string>} - The `Content-Type` header With `this.boundary`
   */
  getHeaders() {
    return {
      'Content-Type': `multipart/form-data; boundary=${this.boundary}`
    }
  }

  /**
   * Append a customized mime-type(s)
   *
   * @param {Object<string,string>} things - The mime-type
   *
   * @return {Form} - The `Form` class instance self
   */
  appendMimeTypes(things) {
    Object.assign(this.mimeTypes, things)

    return this
  }

  /**
   * Append data wrapped by `boundary`
   *
   * @param  {string} field - The field
   * @param  {string|Buffer} value - The value
   * @param  {String} [filename] - Optional filename, when provided, then append the `Content-Type` after of the `Content-Disposition`
   *
   * @return {Form} - The `Form` class instance self
   */
  append(field, value, filename = '') {
    const {data, dashDash, boundary, CRLF, mimeTypes, indices} = this

    data.push(Buffer.from(`Content-Disposition: form-data; name="${field}"${filename && Buffer.isBuffer(value) ? `; filename="${filename}"` : ``}`))
    data.push(CRLF)
    if (filename || Buffer.isBuffer(value)) {
      data.push(Buffer.from(`Content-Type: ${mimeTypes[extname(filename).substring(1).toLowerCase()] || `application/octet-stream`}`))
      data.push(CRLF)
    }
    data.push(CRLF)
    indices[field] = data.push(Buffer.isBuffer(value) ? value : Buffer.from(String(value)))
    data.push(CRLF)
    data.push(dashDash)
    data.push(boundary)
    data.push(CRLF)

    return this
  }
}

module.exports = Form
module.exports.default = Form
```

测试用例如下:

```
lib/form
  ✓ should be class `Form`
  new Form
    ✓ should instanceOf Form and have properties `data` and `indices`
    ✓ The `mimeTypes` property should be there and only allowed append(cannot deleted)
    ✓ The `dashDash` Buffer property should be there and cannot be deleted/modified
    ✓ The `boundary` Buffer property should be there and cannot be deleted/modified
    ✓ The `CRLF` Buffer property should be there and cannot be deleted/modified
    ✓ The `data` property should be instanceOf Array and cannot deleted
    ✓ The `indices` property should be instanceOf Object and cannot deleted
    ✓ Method `getBuffer()` should returns a Buffer instance and had fixed length(108) default
    ✓ Method `getHeaders()` should returns a Object[`Content-type`] with `multipart/form-data; boundary=`
    ✓ Method `appendMimeTypes()` should returns the Form instance
    ✓ Method `appendMimeTypes({any: 'mock'})` should returns the Form instance, and affected `form.data` property
    ✓ Method `append()` should returns the Form instance, and affected `form.data` property
    ✓ Method `append()` should append name="undefined" disposition onto the `form.data` property
    ✓ Method `append({}, 1)` should append name="[object Object]" disposition onto the `form.data` property
    ✓ Method `append('meta', JSON.stringify({}), 'meta.json')` should append a `Content-Type: application/json` onto the `form.data` property
    ✓ Method `append('image_content', Buffer.from('R0lGODlhAQABAAAAACH5BAEKAAEALAAAAAABAAEAAAICTAEAOw==', 'base64'), 'demo.gif')` should append a `Content-Type: image/gif` onto the `form.data` property
```

API文档如下

Simple and lite of `multipart/form-data` implementation, most similar to `form-data`

```js
(new Form)
  .append('a', 1)
  .append('b', '2')
  .append('c', Buffer.from('31'))
  .append('d', JSON.stringify({}), 'any.json')
  .append('e', require('fs').readFileSync('/path/your/file.jpg'), 'file.jpg')
  .getBuffer()
```

* [Form](#Form)
    * [new Form()](#new_Form_new)
    * [.getBuffer()](#Form+getBuffer) ⇒ `Buffer`
    * [.getHeaders()](#Form+getHeaders) ⇒ `Object.<string, string>`
    * [.appendMimeTypes(things)](#Form+appendMimeTypes) ⇒ `Form`
    * [.append(field, value, [filename])](#Form+append) ⇒ `Form`

- new Form()

  - Create a `multipart/form-data` buffer container for the file uploading.

- form.getBuffer() ⇒ `Buffer`

  - To retrieve the `data` buffer
  - **Kind**: instance method of [`Form`](#Form)
  - **Returns**: `Buffer` - - The payload buffer

- form.getHeaders() ⇒ `Object.<string, string>`

  - To retrieve the `Content-Type` multipart/form-data header
  - **Kind**: instance method of [`Form`](#Form)
  - **Returns**: `Object.<string, string>` - - The `Content-Type` header With `this.boundary`

- form.appendMimeTypes(things) ⇒ [`Form`](#Form)

  - Append a customized mime-type(s)
  - **Kind**: instance method of [`Form`](#Form)
  - **Returns**: [`Form`](#Form) - - The `Form` class instance self
  - Param: things
  - Type: `Object.<string, string>`
  - Description: The mime-type


- form.append(field, value, [filename]) ⇒ [`Form`](#Form)

  - Append data wrapped by `boundary`
  - **Kind**: instance method of [`Form`](#Form)
  - **Returns**: [`Form`](#Form) - - The `Form` class instance self
  - Param: field
  - Type: `string`
  - Description: The field
  - Param: value
  - Type: `string` | `Buffer`
  - Description: The value
  - Param: [filename]
  - Type: `string` | `Buffer`
  - Description: Optional filename, when provided, then append the `Content-Type` after of the `Content-Disposition`

- mimeTypes : `Object.<string, string>`

  - built-in mime-type mapping
  - **Kind**: global variable

- dashDash : `Buffer`

  - **Kind**: global variable

- boundary : `Buffer`

  - **Kind**: global variable

- CRLF : `Buffer`

  - **Kind**: global variable

- data : `array.<Buffer>`

  - The Form's data storage
  - **Kind**: global variable

- indices : `Object.<string, number>`

  - The entities' value indices whose were in `this.data`


此类内置常用的几种文件类型（`append`第三入参以文件名后缀比对），已经够用了，视频仅内置了两种，对于 官方接口支持的`avi`, `wmv`, `mov`, `mkv`, `flv`, `f4v`, `m4v`, `rmvb`，开发者可用透过 `appendMimeTypes` 方法，自行扩展以符合 RFC2388 规范。

图片上传接口，用法

```javascript
const form = new Form
form.append(
  'file',
  require('fs').readFileSync('/path/your/file.jpg'), 'file.jpg'
  )
  .append('meta', JSON.stringify({
    filename:'file.jpg',
    sha256:'779a563f99f824975b3651bfd8597555e69fb135925e460dae3996d47c415fb0'
  }), 'meta.json')
```

整个需要发送的表单体就准备妥当了，然后按照v3开发规范，该数据签名的签名，想用什么`client`提交就用什么`client`，然后就没然后了。。。

以我习惯用的 `axios` 为例，数据提交类似如下：

```javascript
const axios = require('axios')

//伪代码, baseURL 要按实际接口地址赋值, Authorization 要按v3规范赋值
axios.create({baseURL}).post(
  form.getBuffer(),
  {headers: {
      Authorization: 'WECHATPAY2-SHA256-RSA2048 ',
      Accept: 'application/json',
      ...form.getHeaders(),
  }}
)
  .catch(({response: {headers, data}}) => ({headers, data}))
  .then(({headers, data}) => ({headers, data}))
  .then(console.log)
```

写到最后

Form类文件以MIT开源，文章转载请注明出处「[来自微信开发者社区](https://developers.weixin.qq.com/community/develop/article/doc/000c24f0390ff8b5d91b2489059413)」。

---
title: "Pkcs7 Padding Unpadding Research in Nodejs Env"
date: 2020-09-17T18:24:20+08:00
keywords: ["PKCS7", "rfc2315", "NodeJS"]
description: "PKCS7/Padding implementation on ES2015+, just using the Native `Buffer` library to do the right things."
tags: []
categories: ["wechat"]
author: "James"
summary: "Pure javascript codes to implement the `PKCS7/Padding` specification, regarding by rfc2315. It just using the Native `Buffer` library to do the right things."
---
It's a part of the `wechatpay-axios-plugin` codes, regarding by the [rfc2315](https://tools.ietf.org/html/rfc2315#section-10.3), the `PKCS7/Padding` spec was post a notes there <quote>The padding can be removed unambiguously since all input is padded and no padding string is a suffix of another. This padding method is well-defined if and only if k < 256; methods for larger k are an open issue for further study.</quote>

For other developing language, there were more than one librares did the right thing. In Javascript, there were also lots of librares too. The following codes is another implementation on ES2015+ style. It just using the Native `Buffer` library to do the right things.

```javascript
class Aes {
  /**
   * @property {integer} BLOCK_SIZE - The `aes` block size
   */
  static get BLOCK_SIZE() { return 16 }

  /**
   * @property {object} pkcs7 - The PKCS7 padding/unpadding container
   */
  static get pkcs7() {
    const {BLOCK_SIZE} = this

    return {
      /**
       * padding, 32 bytes/256 bits `secret key` may optional need the last block.
       * @see https://tools.ietf.org/html/rfc2315#section-10.3
       * <quote>
       * The padding can be removed unambiguously since all input is
       *     padded and no padding string is a suffix of another. This
       *     padding method is well-defined if and only if k < 256;
       *     methods for larger k are an open issue for further study.
       * </quote>
       *
       * @param {string|Buffer} thing - The input
       * @param {boolean} [optional = true] - The flag for the last padding
       *
       * @return {Buffer} - The PADDING tailed payload
       */
      padding: (thing, optional = true) => {
        const payload = Buffer.from(thing)
        const pad = BLOCK_SIZE - payload.length % BLOCK_SIZE

        if (optional && pad === BLOCK_SIZE) {
          return payload
        }

        return Buffer.concat([payload, Buffer.alloc(pad, pad)])
      },

      /**
       * unpadding
       *
       * @param  {string|Buffer} thing - The input
       * @return {Buffer} - The PADDING wiped payload
       */
      unpadding: thing => {
        const byte = Buffer.alloc(1)
        const payload = Buffer.from(thing)
        payload.copy(byte, 0, payload.length - 1)
        const pad = byte.readUInt8()

        if (!~~Buffer.compare(Buffer.alloc(pad, pad), payload.slice(-pad))) {
          return payload.slice(0, -pad)
        }

        return payload
      },
    }
  }
}
```

Usage manual

```javascript
console.log(Aes.pkcs7.padding(''))
```
```
<Buffer >
```
```javascript
console.log(Aes.pkcs7.padding('', false))
```
```
<Buffer 10 10 10 10 10 10 10 10 10 10 10 10 10 10 10 10>
```
```javascript
console.log(Aes.pkcs7.padding('1234567890'))
```
```
<Buffer 31 32 33 34 35 36 37 38 39 30 06 06 06 06 06 06>
```
```javascript
console.log(Aes.pkcs7.unpadding(Aes.pkcs7.padding('', false)).toString())
```
```
''
```
```javascript
console.log(Aes.pkcs7.unpadding(Aes.pkcs7.padding('1234567890')).toString())
```
```
1234567890
```

That's it!

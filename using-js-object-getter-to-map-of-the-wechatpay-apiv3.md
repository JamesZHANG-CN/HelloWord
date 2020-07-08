---
title: "使用Getter构建基于HTTP传输协议的面向对象编程模式SDK"
date: 2020-07-08T22:40:12+08:00
keywords: ["微信支付V3", "Wechatpay APIv3 SDK", "Axios Plugin", "wechatpay-axios-plugin npm"]
description: "使用Object Getter方法，构建可动态弹性扩容的微信支付APIv3 SDK"
tags: []
categories: ["wechat"]
summary: "按照HTTP协议，从暴露出的URL根级开始，客户端均可对URI发起 `GET` `POST` `PATCH` `DELETE` 等请求。这里按照以斜线拆分 `URL.pathname` 的串型javascript实体对象模型，对微信支付APIv3，使用Object Getter方法，构建可动态弹性扩容的SDK"
author: "James"
---

有待于微信支付APIv3良好的接口设计，让基于getter模型构建可弹性扩容的SDK成为可能。RFC3986 已经规范化了 HTTP 协议，我们将要操作的API资源以及方法，均在规范范围之内，基于此，让我们来设计一款符合规范的SDK。

## 设计目标

1. `URI` 是要操作的资源标识，通过斜线`/`(slash)对实体做了分割；
2. `HTTP METHOD` `GET`/`POST` 是对URI资源的操作方法；
3. URI 是可变化的，将要实施的SDK应该具备随 URI 伸缩而弹性扩容或者缩容功能；

## 之前的实践

很早就内置在 javascript 内核中的 `Proxy` 是一款好东西，详细介绍可参考 MDN ，这里不详述。我们回溯在18年中就已经实践过用 `Proxy` 来代理了基于 `Hessian2` 的 `WebService`，使二进制协议可以转换成了 JSON RPC，至今稳定运行且持续提供服务中。对于微信支付APIv3来说，我们少了一层协议转换，却多了一层资源弹性扩缩容功能，这个难度不是一丢丢，相当于要重新设计之前的设计。

## 设计历程

### #1 URI转换成javascript资源对象

这个相对来说，最简单，只须按照 `URL.pathname` 去掉开头斜线，中间部分按斜线分割就好，每个分割后的字符串，记之为实体Entity，例如

`/v3/ecommerce/profitsharing/orders` 即分割成 `v3` `ecommerce` `profitsharing` `orders` 4个实体，依次序即构成各个实体对象标识即 `v3.ecommerce.profitsharing.orders` .

### #2 对javscript资源对象绑定 HTTP METHODs

按照HTTP协议，从暴露出的根级开始，客户端均可以发起 `GET` `POST` `PATCH` `DELETE` 等请求。那么按照 #1 的拆分方法，串型的每一级实体，实际均须绑定上这些 `METHODs`，还好微信支付APIv3多见 `GET`和`POST`请求，那么每一级对象实体，就包含有至少这两个方法；

特殊的，基于openAPI规范，对于动态path参数来说，某个实体有可能是个变量，在客户端最终发起请求之前，需要提供一个方法进行path参数替换，这就要求每个实体上，均需有个转换方法了；

另外，微信支付APIv3的`URI` 中的实体资源标识，存在中线(`- dash`)分割单词的情形，虽然javascript对象可以按照属性方式进行获取，然则不好看呐。。。这里要做少许约定，即对中线分割的单词，可以按照驼峰式(`camelCase`)进行单词组合，这样就使资源实体紧凑化了；

### #3 随URI弹性扩容javascript实体资源对象

这是本类库的精华部分之一了，类似其他开发语言一样，`javascript`中的 `Object`对象，均含有 `getter` 及 `setter` ，详细介绍可翻 MDN， 这里不垒述。这里我们结合使用 `Proxy` 类，对每个`URI`实体对象绑定代理上一个`getter`方法，这样就无限可扩容本`SDK`要操作的实体资源标识。

## 约定

此编程模型，在开发书写过程中，有如下约定：

1. 请求 `URI` 作为级联对象，可以轻松构建请求对象，例如 `/v3/pay/transactions/native` 即自然翻译成 `v3.pay.transactions.native`;
2. 每个 `URI` 所支持的 `HTTP METHOD`，即作为 请求对象的末尾执行方法，例如: `v3.pay.transactions.native.post({})`;
3. 每个 `URI` 有中线(dash)分隔符的，可以使用驼峰`camelCase`风格书写，例如: `merchant-service`可写成 `merchantService`，或者属性风格，例如 `v3['merchant-service']`;
4. 每个 `URI`.pathname 中，若有动态参数，例如 `business_code/{business_code}` 可写成 `business_code.$business_code$` 或者属性风格书写，例如 `business_code['{business_code}']`，抑或直接按属性风格，直接写参数值也可以，例如 `business_code['2000001234567890']`;
5. 建议 `URI` 按照 `PascalCase` 风格书写, `TS Definition` 已在路上(还有若干问题没解决)，将是这种风格，代码提示将会很自然;

All of the remote's `URI.pathname` is mapping as the `SDK`'s `Object` which contains the HTTP `get` `post` and `upload` (named for `multipart/form-data`) methods. Specially, while the `Object`(named as `container`) with dynamic veriable parameter(s) (`uri_template`), those may be transformed by the `container`'s `withEntities` method.

With this style, the following conversions should be known:

1. Each of the `URI`, eg: `/v3/pay/transactions/native` is mapping to `v3.pay.transactions.native`;
2. Each of the `URI`'s `HTTP METHOD`，are mapping to the `container`'s executor, eg: `v3.pay.transactions.native.post({})`;
3. While the `URI` contains the `dash-split-words`, the entity can be wrote as `camelCase` style, eg: `merchant-service` to `merchantService` or by the `Object.property`'s style as `v3['merchant-service']`;
4. While the `URI` contains the `dynamic_veriable_parameter`, eg: `business_code/{business_code}`, it can be `business_code.$business_code$` or `business_code['{business_code}']` or directly replaced with actual value eg: `business_code['2000001234567890']`;
5. Recommend writing the `URI` entities with `PascalCase` style. Because the `TS Definition` is on the road, those shall be this style;

## 最终形态

```js
const axios = require('axios')
const interceptor = require('./interceptor')

/**
 * A Wechatpay APIv3's amazing client.
 *
 * ```js
 * const {Wechatpay} = require('wechatpay-axios-plugin')
 * const wxpay = new Wechatpay({
 *   mchid,
 *   serial,
 *   privateKey: '-----BEGIN PRIVATE KEY-----' + '...' + '-----END PRIVATE KEY-----',
 *   certs: {
 *     'serial_number': '-----BEGIN CERTIFICATE----- ... -----END CERTIFICATE-----'
 *   }
 * })
 *
 * wxpay.V3.Marketing.Busifavor.Stocks.post({})
 *   .then(({data}) => console.info(data))
 *   .catch(({response: {data}}) => console.error(data))
 *
 * wxpay.V3.Pay.Transactions.Native.post({})
 *   .then(({data: {code_url}}) => console.info(code_url))
 *   .catch(({response: {data}}) => console.error(data))
 *
 * ;(async () => {
 *   try {
 *     const {data: detail} = await wxpay.V3.Pay.Transactions.Id.$transaction_id$
 *       .withEntities({transaction_id: '1217752501201407033233368018'})
 *       .get({params: {mchid: '1230000109'}})
 *     // or simple like this
 *     // const {data: detail} = await wxpay.V3.Pay.Transactions.Id['{transaction_id}']
 *     //   .withEntities({transaction_id: '1217752501201407033233368018'})
 *     //   .get({params: {mchid: '1230000109'}})
 *     // or simple like this
 *     // const {data: detail} = await wxpay.v3.pay.transactions.id['121775']
 *     //   .get({params: {mchid: '1230000109'}})
 *     console.info(detail)
 *   } catch({response: {status, statusText, data}}) {
 *     console.error(status, statusText, data)
 *   }
 * })()
 * ```
 */
class Wechatpay {
  /**
   * @property {import('axios').AxiosInstance} client - The axios instance
   */
  static client;

  /**
   * @property {RegExp} URI_ENTITY - The URI entity which's split by slash
   */
  /*eslint-disable-next-line*/
  static URI_ENTITY = /^\{([^\}]+)\}$/;

  /**
   * Compose the `URL`.pathname based on the container's entities
   * @param {array} entities - Each `container` of `entities`
   * @returns {string} - The `URL`.pathname
   */
  static pathname(entities = []) {
    return `/${entities.join('/')}`
  }

  /**
   * Normalize the `str` by the rules:
   *  `PascalCase` -> `camelCase`
   * & `camelCase` -> `camel-case`
   * & `$dynamic$` -> `{dynamic}`
   *
   * @param {string} str - The string waiting for normalization
   * @returns {string} - The transformed string
   */
  static normalize(str) {
    return (str || '')
      // PascalCase` to `camelCase`
      .replace(/^[A-Z]/, w => w.toLowerCase())
      // `camelCase` to `camel-case`
      .replace(/[A-Z]/g, w => `-${w.toLowerCase()}`)
      // `$dynamic_variable$` to `{dynamic_variable}`
      .replace(/^\$/, `{`).replace(/\$$/, `}`)
  }

  /**
   * @property {object} container - Client side the URIs' entity mapper
   */
  static container = {
    /**
     * @property {string[]} entities - The URI entities
     */
    entities: [],

    /**
     * @property {function} withEntities - Replace the `uri` with real entities' value
     * @param {string[]} list - The real entities' mapping
     * @returns {object} - the container's instance
     */
    withEntities: function(list) {
      this.entities.forEach((one, index, src) => {
        if (Wechatpay.URI_ENTITY.test(one)) {
          const sign = one.replace(Wechatpay.URI_ENTITY, `$1`)
          src[index] = list[sign] ? list[sign] : one
        }
      })

      return this
    },

    /**
     * @property {function} get - The alias of the HTTP `GET` request
     * @param {...any} arg - The request arguments
     * @returns {PromiseLike} - The `AxiosPromise`
     */
    get: async function(...arg) {
      return Wechatpay.client.get(Wechatpay.pathname(this.entities), ...arg)
    },

    /**
     * @property {function} post - The alias of the HTTP `POST` request
     * @param {...any} arg - The request arguments
     * @returns {PromiseLike} - The `AxiosPromise`
     */
    post: async function(...arg) {
      return Wechatpay.client.post(Wechatpay.pathname(this.entities), ...arg)
    },

    /**
     * @property {function} upload - The alias of the HTTP 'multipart/form-data' request
     * @param {...any} arg - The request arguments
     * @returns {PromiseLike} - The `AxiosPromise`
     */
    upload: async function(...arg) {
      //TODO: wrap the FormData and define the headers['content-type']
      return this.post(...arg)
    },
  };

  /**
   * @property {object} handler - Handler of the container instance's `getter`
   */
  static handler = {
    /**
     * @property {function} get - Object's `getter` handler
     * @param {object} target - The object
     * @param {string} property - The property
     * @returns {object} - An object or object's property
     */
    get: (target, property) => {
      if (!property || typeof property === `symbol` || property === `inspect`) {
        return target
      }
      if (!(property in target)) {
        /*eslint-disable-next-line*/
        target[property] = new Proxy({...Wechatpay.container}, Wechatpay.handler)
        if (`entities` in target) {
          target[property].entities = [...target.entities, Wechatpay.normalize(property)]
        }
      }

      return target[property]
    },
  };

  /**
   * Constructor of the magic APIv3 container
   * @param {object} wxpayConfig - @see {apiConfig}
   * @param {object} axiosConfig - @see {import('axios').AxiosRequestConfig}
   * @constructor
   * @returns {Proxy} - The magic APIv3 container
   */
  constructor(
  	wxpayConfig = {},
  	axiosConfig = {baseURL: 'https://api.mch.weixin.qq.com'}
  ) {
    Wechatpay.client = Wechatpay.client
      || interceptor(axios.create(axiosConfig), wxpayConfig)

    /*eslint-disable-next-line*/
    return new Proxy({...Wechatpay.container}, Wechatpay.handler)
  }
}
```

## 结束语

本类库遵循`CMD`模块设计，`class`上，仅且唯一向外暴露出构造函数，其他方法及属性均封装成静态`类`属性/方法，保证对象实例化之后全局均是操作同一对象/属性，然而这也同时是一把双刃剑，对持久进程多租户/商户来说，可能存有资源相互干扰情况。这一点上，请使用此类库的开发者特别注意。对于有此需求的开发者，我的建议是，在这之上，用容器技术(`docker`)进行租/商户环境唯一化，简单暴力。

以上，已随 `wechatpay-axios-plugin` npm包，开源在GitHub上，喜欢就给点个 Star 吧。

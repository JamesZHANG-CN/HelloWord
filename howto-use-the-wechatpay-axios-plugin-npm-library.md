---
title: "wechatpay-axios-plugin 开发系列之「起步」"
date: 2021-03-30T10:15:16+08:00
keywords: [wechatpay-axios-plugin nodejs development]
description: "如何起步 wechatpay-axios-plugin SDK开发包。Howto Use the Wechatpay Axios Plugin Npm Library"
tags: []
toc: false
categories: [javascript wechat]
summary: "nodejs及云开发环境，如何起步 wechatpay-axios-plugin SDK开发包。"
author: "James"
---

从github issues可以感知到，本SDK已有案例被应用到云开发及自主开发环境，从开发者角度来看，此类库还有些难以架弩，本系列将作为开发对接指导，希望能对开发有所帮助。

## 起步

说起开发，最忌讳的是急急火火的先堆代码，作为一款有性格的SDK，源码全程是英文注释，README我可以用母语来书写，有些同学可能对此不适应，编码讲究的是效率，这里就不展开了。

这里还有一些思考，是关于「起步」，即：如何快速的对接「微信支付」接口，下面我们来展开一下

## 前置条件

1. 注册微信支付商户号
2. 登录商户平台，设置`API密钥`
3. 使用官方专用证书生成工具，生成商户证书
4. 设置API密钥及`APIv3密钥`
5. 有一个nodejs开发环境，版本最低要求是10.15.0，windows、macos、linux操作系统均支持
6. 懂得基本的npm包管理

## 开始

基于一些考虑，如云开发生产环境，依赖项决定了启动速度，这里需要区分一下生产环境及开发环境。面向生产环境，此款类库仅严格依赖两个包：`axios`及`node-xml2js`，衍生总计仅需6个包；面向开发环境需要额外增加`yargs`包。此文以开发环境展开。

### 准备环境

准备一个工作目录，使用 `npm init` 先构建一个基础环境，如已有环境，则略过此步。

### 安装软件

基础包

`npm install wechatpay-axios-plugin --save`

CLI依赖包

`npm install yargs --save-dev`

如需媒体上传，如「制券」及「拓展商户」，则需再安装 `form-data` 依赖

`npm install form-data --save`

### 平台证书

APIv3开发多了一个步骤，就是需要自行下载「平台证书」，本类库提供有下载器，执行命令如下：


`./node_modules/.bin/wxpay crt -m {商户号} -s {商户证书序列号} -f {商户私钥证书路径} -k {APIv3密钥(32字节)} -o {保存地址}`

花括弧内为变量，按需自行替换，执行结果类似如下：

```
The WeChatPay Platform Certificate#0
  serial=HEXADECIAL
  notBefore=Wed, 22 Apr 2020 01:43:19 GMT
  notAfter=Mon, 21 Apr 2025 01:43:19 GMT
  Saved to: wechatpay_HEXADECIAL.pem
You may confirm the above infos again even if this library already did(by Rsa.verify):
    openssl x509 -in wechatpay_HEXADECIAL.pem -noout -serial -dates

```

### 快速体验

接续[上一篇: 一行命令即可体验「微信支付」全系接口能力](/post/play-the-wechatpay-openapi-requests-over-command-line/)，现在就可以在开发环境上，验证及调试官方接口了，额外地， v0.5.1做了优化，命令行现在可以这么执行了：

```
./node_modules/.bin/wxpay v2/pay/micropay \
  -c.mchid 1230000109 \
  -c.serial any \
  -c.privateKey any \
  -c.certs.any \
  -c.secret your_merchant_secret_key_string \
  -d.appid wxd678efh567hg6787 \
  -d.mch_id 1230000109 \
  -d.device_info 013467007045764 \
  -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS \
  -d.detail 'Image形象店-深圳腾大-QQ公仔' \
  -d.spbill_create_ip 8.8.8.8 \
  -d.out_trade_no '1217752501201407033233368018' \
  -d.total_fee 100 \
  -d.fee_type CNY \
  -d.auth_code 120061098828009406
```

以上述命令行为例，此命令请求的是v2版的「付款码支付」接口，`any` 为CLI模式保留字，意思即无需此参数（不能不给），`1230000109` 替换为你的商户号，`your_merchant_secret_key_string` 替换为你的`API密钥`，这条命令就可以验证支付能力了。屏幕打印的日志类似如下：

```javascript
{
  status: 200,
  statusText: 'OK',
  headers: {
    server: 'nginx',
    date: 'Tue, 30 Mar 2021 00:49:30 GMT',
    'content-type': 'text/plain',
    'content-length': '105',
    connection: 'keep-alive',
    'keep-alive': 'timeout=8',
    'request-id': '089AEB89830610A10518B9BF8C5820EA442882F601-47002, 089AEB89830610F6041896C2EEA30620F98004289CF805-0',
    'mmlas-verifyresult': 'CAA='
  },
  config: {
    url: '/pay/micropay',
    method: 'post',
    data: '<xml><appid>wxd678efh567hg6787</appid><mch_id>1230000109</mch_id><device_info>013467007045764</device_info><nonce_str>5K8264ILTKCH16CQ2502SI8ZNMTM67VS</nonce_str><detail>Image形象店-深圳腾大-QQ公仔</detail><spbill_create_ip>8.8.8.8</spbill_create_ip><out_trade_no>1217752501201407033233368018</out_trade_no><total_fee>100</total_fee><fee_type>CNY</fee_type><auth_code>120061098828009406</auth_code><sign>C5524E8CEB11D1060A9FF0E696F52F51</sign></xml>',
    headers: {
      Accept: 'application/json, text/plain, */*',
      'Content-Type': 'text/xml; charset=utf-8',
      'User-Agent': 'wechatpay-axios-plugin/0.5.1 axios/0.21.1 node/14.14.0 darwin/x64',
      'Content-Length': 458
    },
    baseURL: 'https://api.mch.weixin.qq.com/',
    transformRequest: [ [Function: signer], [Function: toXml] ],
    transformResponse: [ [Function: toObject], [Function: verifier] ],
    timeout: 0,
    adapter: [Function: httpAdapter],
    responseType: 'text',
    xsrfCookieName: 'XSRF-TOKEN',
    xsrfHeaderName: 'X-XSRF-TOKEN',
    maxContentLength: -1,
    maxBodyLength: -1,
    httpsAgent: Agent {
      _events: [Object: null prototype],
      _eventsCount: 2,
      _maxListeners: undefined,
      defaultPort: 443,
      protocol: 'https:',
      options: [Object],
      requests: {},
      sockets: {},
      freeSockets: [Object],
      keepAliveMsecs: 1000,
      keepAlive: true,
      maxSockets: Infinity,
      maxFreeSockets: 256,
      scheduling: 'fifo',
      maxTotalSockets: Infinity,
      totalSocketCount: 0,
      maxCachedSessions: 100,
      _sessionCache: [Object],
      [Symbol(kCapture)]: false
    },
    validateStatus: [Function: validateStatus],
    mchid: 1230000109,
    serial: 'any',
    privateKey: 'any',
    certs: { any: true },
    secret: 'your_*********************tring'
  },
  request: <ref *1> ClientRequest {
    _events: [Object: null prototype] {
      socket: [Function (anonymous)],
      abort: [Function (anonymous)],
      aborted: [Function (anonymous)],
      connect: [Function (anonymous)],
      error: [Function (anonymous)],
      timeout: [Function (anonymous)],
      prefinish: [Function: requestOnPrefinish]
    },
    _eventsCount: 7,
    _maxListeners: undefined,
    outputData: [],
    outputSize: 0,
    writable: true,
    destroyed: true,
    _last: false,
    chunkedEncoding: false,
    shouldKeepAlive: true,
    _defaultKeepAlive: true,
    useChunkedEncodingByDefault: true,
    sendDate: false,
    _removedConnection: false,
    _removedContLen: false,
    _removedTE: false,
    _contentLength: null,
    _hasBody: true,
    _trailer: '',
    finished: true,
    _headerSent: true,
    socket: TLSSocket {
      _tlsOptions: [Object],
      _secureEstablished: true,
      _securePending: false,
      _newSessionPending: false,
      _controlReleased: true,
      secureConnecting: false,
      _SNICallback: null,
      servername: 'api.mch.weixin.qq.com',
      alpnProtocol: false,
      authorized: true,
      authorizationError: null,
      encrypted: true,
      _events: [Object: null prototype],
      _eventsCount: 9,
      connecting: false,
      _hadError: false,
      _parent: null,
      _host: 'api.mch.weixin.qq.com',
      _readableState: [ReadableState],
      _maxListeners: undefined,
      _writableState: [WritableState],
      allowHalfOpen: false,
      _sockname: null,
      _pendingData: null,
      _pendingEncoding: '',
      server: undefined,
      _server: null,
      ssl: [TLSWrap],
      _requestCert: true,
      _rejectUnauthorized: true,
      parser: null,
      _httpMessage: null,
      timeout: 0,
      [Symbol(res)]: [TLSWrap],
      [Symbol(verified)]: true,
      [Symbol(pendingSession)]: null,
      [Symbol(async_id_symbol)]: -1,
      [Symbol(kHandle)]: [TLSWrap],
      [Symbol(kSetNoDelay)]: false,
      [Symbol(lastWriteQueueSize)]: 0,
      [Symbol(timeout)]: null,
      [Symbol(kBuffer)]: null,
      [Symbol(kBufferCb)]: null,
      [Symbol(kBufferGen)]: null,
      [Symbol(kCapture)]: false,
      [Symbol(kBytesRead)]: 0,
      [Symbol(kBytesWritten)]: 0,
      [Symbol(connect-options)]: [Object],
      [Symbol(RequestTimeout)]: undefined
    },
    _header: 'POST /pay/micropay HTTP/1.1\r\n' +
      'Accept: application/json, text/plain, */*\r\n' +
      'Content-Type: text/xml; charset=utf-8\r\n' +
      'User-Agent: wechatpay-axios-plugin/0.5.1 axios/0.21.1 node/14.14.0 darwin/x64\r\n' +
      'Content-Length: 458\r\n' +
      'Host: api.mch.weixin.qq.com\r\n' +
      'Connection: keep-alive\r\n' +
      '\r\n',
    _keepAliveTimeout: 0,
    _onPendingData: [Function: noopPendingOutput],
    agent: Agent {
      _events: [Object: null prototype],
      _eventsCount: 2,
      _maxListeners: undefined,
      defaultPort: 443,
      protocol: 'https:',
      options: [Object],
      requests: {},
      sockets: {},
      freeSockets: [Object],
      keepAliveMsecs: 1000,
      keepAlive: true,
      maxSockets: Infinity,
      maxFreeSockets: 256,
      scheduling: 'fifo',
      maxTotalSockets: Infinity,
      totalSocketCount: 0,
      maxCachedSessions: 100,
      _sessionCache: [Object],
      [Symbol(kCapture)]: false
    },
    socketPath: undefined,
    method: 'POST',
    maxHeaderSize: undefined,
    insecureHTTPParser: undefined,
    path: '/pay/micropay',
    _ended: true,
    res: IncomingMessage {
      _readableState: [ReadableState],
      _events: [Object: null prototype],
      _eventsCount: 3,
      _maxListeners: undefined,
      socket: null,
      httpVersionMajor: 1,
      httpVersionMinor: 1,
      httpVersion: '1.1',
      complete: true,
      headers: [Object],
      rawHeaders: [Array],
      trailers: {},
      rawTrailers: [],
      aborted: false,
      upgrade: false,
      url: '',
      method: null,
      statusCode: 200,
      statusMessage: 'OK',
      client: [TLSSocket],
      _consuming: false,
      _dumped: false,
      req: [Circular *1],
      responseUrl: 'https://api.mch.weixin.qq.com/pay/micropay',
      redirects: [],
      [Symbol(kCapture)]: false,
      [Symbol(RequestTimeout)]: undefined
    },
    aborted: false,
    timeoutCb: null,
    upgradeOrConnect: false,
    parser: null,
    maxHeadersCount: null,
    reusedSocket: false,
    host: 'api.mch.weixin.qq.com',
    protocol: 'https:',
    _redirectable: Writable {
      _writableState: [WritableState],
      _events: [Object: null prototype],
      _eventsCount: 2,
      _maxListeners: undefined,
      _options: [Object],
      _ended: true,
      _ending: true,
      _redirectCount: 0,
      _redirects: [],
      _requestBodyLength: 458,
      _requestBodyBuffers: [],
      _onNativeResponse: [Function (anonymous)],
      _currentRequest: [Circular *1],
      _currentUrl: 'https://api.mch.weixin.qq.com/pay/micropay',
      [Symbol(kCapture)]: false
    },
    [Symbol(kCapture)]: false,
    [Symbol(kNeedDrain)]: false,
    [Symbol(corked)]: 0,
    [Symbol(kOutHeaders)]: [Object: null prototype] {
      accept: [Array],
      'content-type': [Array],
      'user-agent': [Array],
      'content-length': [Array],
      host: [Array]
    }
  },
  data: { return_code: 'FAIL', return_msg: '缺少参数' }
}
```

当你遇到困难，需要社区帮助的时候，类库也做了安全规范，把`secret`做了脱敏处理，上述日志你可以无脑地直接拷贝粘贴过来（再无脑也需要注意格式，以`javascript`代码形式粘贴，会方便阅读）。


## 海外主体

同样适用，在CLI驱动时，仅需在给一个 `--baseURL` 参数即可，如香港：

下载平台证书

```
./node_modules/.bin/wxpay crt \
  --baseURL 'https://apihk.mch.weixin.qq.com' \
  -m {商户号} \
  -s {商户证书序列号} \
  -f {商户私钥证书路径} \
  -k {APIv3密钥(32字节)} \
  -o {保存地址}
```

付款码支付

```
./node_modules/.bin/wxpay v2/pay/micropay \
  --baseURL 'https://apihk.mch.weixin.qq.com' \
  -c.mchid 1230000109 \
  -c.serial any \
  -c.privateKey any \
  -c.certs.any \
  -c.secret your_merchant_secret_key_string \
  -d.appid wxd678efh567hg6787 \
  -d.mch_id 1230000109 \
  -d.device_info 013467007045764 \
  -d.nonce_str 5K8264ILTKCH16CQ2502SI8ZNMTM67VS \
  -d.detail 'Image形象店-深圳腾大-QQ公仔' \
  -d.spbill_create_ip 8.8.8.8 \
  -d.out_trade_no '1217752501201407033233368018' \
  -d.total_fee 100 \
  -d.fee_type CNY \
  -d.auth_code 120061098828009406
```

官方接口至少70%以上的接口，均可以在CLI模式上做快速体验，验证入参出参、调试，甚至可以作为CI工具集，希望能为你的开发带来帮助。

## 下一回

正统WEB编程。

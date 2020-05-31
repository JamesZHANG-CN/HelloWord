---
title: "使用Guzzle标准包，向微信支付V3官方中间件添加媒体上传类"
date: 2020-05-31T10:54:14+08:00
draft: true
keywords: ["微信支付V3", "GuzzleHttp", "MediaUtils"]
description: "通过使用 GuzzlHttp\\Psr7\\Fnstream 修饰 GuzzlHttp\\Psr7\\MultipartStream 类，使微信支付官方wechatpay-guzzle-middleware支持媒体文件上传 。 同时在处理文件上传时，优化了业务代码获取文件二进制内容及对内容做 sha256 计算。 不侵入官方包，使用起来相当简单，仅在需要上传媒体文件时引入并实例化即可。"
tags: []
categories: ["wechat"]
summary: "通过使用 GuzzlHttp\\Psr7\\Fnstream 修饰 GuzzlHttp\\Psr7\\MultipartStream 类，使微信支付官方wechatpay-guzzle-middleware支持媒体文件上传 。 同时在处理文件上传时，优化了业务代码获取文件二进制内容及对内容做 sha256 计算。 不侵入官方包，使用起来相当简单，仅在需要上传媒体文件时引入并实例化即可。"
author: "James"
---

既然是给中间件做新功能加强，自然就需要熟悉中间件的实现机理，`wechatpay-guzzle-middleware` (以下简称middleware) PHP包解决了 微信支付APIV3(以下简称APIV3) 的通信协议签名、返回值验签等工作，基本实现了 APIV3 的规范要求，唯有媒体(图片、视频)上传功能未加强，社区有同学使用通过注册metaJson头方案实现了媒体上传(本人尝试过，方案可行)，然则不是一个优解方案。

理想方案应该是：

- 符合 APIV3 规范
- 不增加、不调整基础包实现(脱耦)
- 使用起来应该足够简单

实现上述三个标准，这挑战其实蛮大的，还好， `Guzzle` 包经过了社区检验，是一套功能完善的libary包。下面我们动手来分析，如何仅用 `Guzzle` 包，来解决上述三项挑战。

## 协议分析

1. 通过HTTP协议上传文件，传输体一定是 `multipart/form-data` 类型。APIV3 规范说明指出，通过不同的 `boundary` 体，传输媒体文件二进制流及文件流 `meta{filename,sha256}` JSON结构数据，HTTP头部签名是对 `meta` 数据做签名；

2. middleware 的HTTP头部签名是在 `WechatPay\GuzzleMiddleware\Auth\WechatPay2Credentials::buildMessage` 实现，其通过 `(string) $request->getBody()` 获取待签数据，在本案中，即需要返回 `meta` JSON数据，稍后再表；

3. `GuzzleHttp\Client` 在处理文件上传时，是通过 `GuzzleHttp\Handler\CurlFactory::applyBody` 向 `GuzzleHttp\Handler\CurlHandler`
传递最终 `$request` 对象，由 `GuzzleHttp\Handler\CurlHandler::__invoke` 发起请求；这里 `applyBody` 在处理文件上传时，源码注释上说，小于1M的文件，使用 `CURLOPT_POSTFIELDS`，其获取 `body` 数据的方法与 middleware 一样，都是`(string) $request->getBody()`，前面已提到，这里预期返回的是 `meta` JSON数据，非真正传输的混合二进制文件内容，无法达到要求；继续分析代码块，`applyBody` 方法的 `CURLOPT_UPLOAD` 逻辑区域，是通过 `CURLOPT_READFUNCTION` 从流中获取需要上传的字节流长度，程序设计上就需要让 `applyBody` 走到这块逻辑处理单元，其判断条件是 `$request->getBody()->getSize()` 为null或者传输内容大于1M，并且 `$options['_body_as_string']` 为空或者未设置，(注：给读者留个作业，`Guzzle` 内部是如何获取上传文件`字节流长度`及`字节串`的）这条路可行；


## 实现

通过上述协议及框架能力分析，`$request->getBody()` 获取到的 `Stream` 实例，其应有 `MultipartStream` 的所有功能需求，并且至少须有两个方法即 `__toString` 及 `getSize` 返回特定数据串。 `Guzzle` 包提供了 `GuzzlHttp\Psr7\Fnstream` 这个类，允许开发者修饰定义 15 种 `Stream` 的接口函数，`__toString` 及 `getSize` 就包含再内，实现起来仅需分别向 `WechatPay2Credentials::buildMessage` 及 `CurlFactory::applyBody` 输出其对应所需数据即可

- 传输内容需是 `multipart/form-data; boundary=Boundary` 结构体， 那就直接用 `GuzzlHttp\Psr7\MultipartStream` 来构造好了；

- 考虑需要足够简单，构造 `MultipartStream` 入参时，使用 `GuzzleHttp\Psr7\UploadedFile` 直接 `Lazy` 读文件;

- 计算文件内容的 `sha256` 摘要，直接使用 `GuzzleHttp\Psr7\hash` 完成；

- 通过 `GuzzlHttp\Psr7\Fnstream::decorate` 修饰两个方法 `__toString` 及 `getSize`，对应的使 `buildMessage` 及 `applyBody` 两个方法分别获取其相对应的数据；

- 最后还需要向 `GuzzleHttp\Client::request` 显示声明 `Content-Type` 头，其值来自于 `MultipartStream`；

### 完整代码如下

```php
<?php
/**
 * MediaUtil
 * PHP version 5
 *
 * @category Class
 * @package  WechatPay
 * @author   WeChatPay Team
 * @link     https://pay.weixin.qq.com
 */

namespace WechatPay\GuzzleMiddleware\Util;

use GuzzleHttp\Psr7\UploadedFile;
use GuzzleHttp\Psr7\MultipartStream;
use GuzzleHttp\Psr7\FnStream;

/**
 * Util for Media(image or video) uploading.
 *
 * @package  WechatPay
 * @author   James Zhang(https://github.com/TheNorthMemory)
 */
class MediaUtil {

    /**
     * local file path
     *
     * @var string
     */
    private $filepath;

    /**
     * upload meta json
     *
     * @var string
     */
    private $json;

    /**
     * upload contents stream
     *
     * @var MultipartStream
     */
    private $multipart;


    /**
     * multipart stream wrapper
     *
     * @var FnStream
     */
    private $stream;

    /**
     * Constructor
     *
     * @param string $filepath The media file path,
     *                         should be one of the
     *                         images(jpg|bmp|png)
     *                         or
     *                         video(avi|wmv|mpeg|mp4|mov|mkv|flv|f4v|m4v|rmvb)
     */
    public function __construct($filepath)
    {
        $this->filepath = $filepath;
        $this->composeStream();
    }

    /**
     * Compose the GuzzleHttp\Psr7\FnStream
     */
    private function composeStream()
    {
        $basename = \basename($this->filepath);
        $uploader = new UploadedFile(
            $this->filepath,
            0,
            UPLOAD_ERR_OK,
            $basename,
            \GuzzleHttp\Psr7\mimetype_from_filename($this->filepath)
        );
        $stream = $uploader->getStream();

        $json = \GuzzleHttp\json_encode([
            'filename' => $basename,
            'sha256'   => \GuzzleHttp\Psr7\hash($stream, 'sha256'),
        ]);
        $this->meta = $json;

        $multipart = new MultipartStream([
            [
                'name'     => 'meta',
                'contents' => $json,
                'headers'  => [
                    'Content-Type' => 'application/json',
                ],
            ],
            [
                'name'     => 'file',
                'filename' => $basename,
                'contents' => $stream,
            ],
        ]);
        $this->multipart = $multipart;

        $this->stream = FnStream::decorate($multipart, [
             // for signature
            '__toString' => function () use ($json) {
                return $json;
            },
             // let the `CURL` to use `CURLOPT_UPLOAD` context
            'getSize' => function () {
                return null;
            },
        ]);
    }

    /**
     * Get the `meta` of the multipart data string
     */
    public function getMeta()
    {
        return $this->meta;
    }

    /**
     * Get the `GuzzleHttp\Psr7\FnStream` context
     */
    public function getStream()
    {
        return $this->stream;
    }

    /**
     * Get the `Content-Type` of the `GuzzleHttp\Psr7\MultipartStream`
     */
    public function getContentType()
    {
        return 'multipart/form-data; boundary=' . $this->multipart->getBoundary();
    }
}
```

### 使用方法

```php
<?php
// 引入 `MediaUtil` 正常初始化，无额外条件
use WechatPay\GuzzleMiddleware\Util\MediaUtil;
// 实例化一个媒体文件流，注意文件后缀名需符合接口要求
$media = new MediaUtil('/your/file/path/with.extension');
// POST 语法糖
$resp = $client->post('merchant/media/upload', [
    'body'    => $media->getStream(),
    'headers' => [
        'Accept'       => 'application/json',
        'content-type' => $media->getContentType(),
    ]
]);
```

结案。

## 结束语

通过使用 `GuzzlHttp\Psr7\Fnstream` 修饰 `GuzzlHttp\Psr7\MultipartStream` 的两个方法，从而让 `wechatpay-guzzle-middleware` 可以内置消化媒体文件上传需求。

同时，这里也用到了 `Guzzle` 处理文件上传的类，优化了业务代码对文件的操作步骤（获取文件二进制内容及对内容做 `sha256` 计算）。 使用起来相当简单，同时也与基础包完全解耦，仅在需要上传媒体文件时引入并实例化即可。

正文的作业，有写完的同学，欢迎 `comment` 交流。
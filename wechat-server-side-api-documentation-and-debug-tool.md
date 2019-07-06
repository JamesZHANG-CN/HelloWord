---
title: "微信服务端接口文档及工具"
summary: "微信官方服务端接口文档以表格及样本数据形式展示的有瑕疵，本开源项目本着造福开发者的目的，收录了670+服务端API入口及参数描述定义。"
date: 2019-06-24T23:07:16+08:00
keywords: [the wechat server side api documentation and debug tool]
description: "the wechat server side api documentation is based on the swagger aka openapi specification"
tags: [wechat swagger]
categories: [wechat]
author: ""
---

截止2019/6/24日，本文档已收录670+个微信服务端接口文档，配合简单反向代理服务器，即可完美诠释「软件交付」——即是接口文档，又可作Debug工具，一遍书写，造福大众。官方文档里，很多地方不明不白的，尤其对传输的数据结构描述，table表格形式根本就表达不出来层级关系，本着造福开发者的初衷，一边工作，一边整理，一边校对校准，部分接口因未使用到，仅做了入口定义，希望能对后来者有所帮助。

代码托管在 [github](https://github.com/thenorthmemory/wechat-swagger) 上，文档参考可直接访问 [这里](https://thenorthmemory.github.io/wechat-swagger/)。

举个栗子，微信公众号卡券，是个超级入口，一共可以创建 截止目前共十一种类型的券 `GROUPON`, `CASH`, `DISCOUNT`, `GIFT`, `GENERAL_COUPON`, `GENERAL_CARD`, `MEETING_TICKET`, `SCENIC_TICKET`, `MOVIE_TICKET`, `BOARDING_PASS`, `MEMBER_CARD`，官方文档做得也是没得说，至少说明了适用场景及基本入参出参结构，但是这里有个小坑，唯有 `Try it out` 才能知道其基本结构，详细可翻阅 [这里](https://github.com/TheNorthMemory/wechat-swagger/commit/688a3d0b319c07e0422035310b834799b8dd801e)。

另外，为了搞定中文TAG跳转，也给 swagger-ui 贡献了[PR #4921](https://github.com/swagger-api/swagger-ui/pull/4921)，也是在捋问题文档的时候，入参非得要求是 String 类型的 `JSON` 数据，可参阅 [这里](https://github.com/TheNorthMemory/wechat-swagger/commit/73cf2302841f690364f63dfc1e389472a35c1372#diff-352d6ce365e31f5c164b14463f8266dbR142)，swagger-ui 本应有能力直接提供 `XML CDATA`, 也提了 [PR #4919](https://github.com/swagger-api/swagger-ui/pull/4919)，至今未合并，只能 [自造轮子](https://www.npmjs.com/package/@thenorthmemory/swagger-ui-dist) 来合理显示并支持 `CDATA` 数据结构。

总而言之，微信服务端接口，还是很易用的，就是开发时 `试错` 工具太少，官方提供的也debug工具也有限，造个轮子，手工 `cmd+c`/`cmd+v` 弄了好久，部分不明不白的地方俺也稀里糊涂就先捋上，有机会再试错了。

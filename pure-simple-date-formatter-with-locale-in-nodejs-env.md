---
title: "使用NodeJS内置Intl.DateTimeFormat方法，实现按时区格式化时间日期字符串"
date: 2020-09-20T12:29:04+08:00
lastmod: 2021-02-28T16:54:06+08:00
keywords: ["DateTimeFormat", "RegExp", "NodeJS", "yyyy-MM-dd HH:mm:ss"]
description: "在手写支付宝OpenAPI SDK的时候，其timestamp要求是yyyy-MM-dd HH:mm:ss 格式，以下函数仅使用内置 Date 及 Intl，尝试对不同时区的时间标准化输出，低依赖。"
tags: []
categories: ["javascript"]
author: "James"
summary: "在手写支付宝OpenAPI SDK的时候，其timestamp要求是yyyy-MM-dd HH:mm:ss 格式，以下函数仅使用nodejs内置的Date及Intl类，来实现对不同时区的时间格式化处理。"
---

As of a special project, there were mentioned that the `Date` should always as `yyyy-MM-dd HH:mm:ss` format. But there should have a `timezone` potential risk while the program were running on the place globally. Especially under the `BaaS` or `FaaS` AKA `Serverless` environment.

I'd researched some resources such as BCP47, rfc3339 about the `DateTimeFormat`. Those specifications It's not easier to scope and use that `options` for me. So sad so bad.

As of the `Moment.js` was under LTS situation, here's a pure simple way to do the formatting date and time with locale, just using the build-in `Date` and `Intl.DateTimeFormat` classes.

Generally, the `en-GB` locale which was `dd/MM/yyyy, HH:mm:ss` format and it's similar to `yyyy-MM-dd HH:mm:ss`. Here's just need put the `source` in that particular order. Codes below:

v1 版

```javascript
var localeDateTime = thing => new Intl.DateTimeFormat(
    'en-GB',
    {
        dateStyle:'short',
        timeStyle:'medium',
        timeZone: 'Asia/Shanghai',
    }
).format(new Date(thing)).replace(
    /^(\d{2})\/(\d{2})\/(\d{4}),\s(.*)$/,
    (_, dd, MM, yyyy, partialTime) => `${yyyy}-${MM}-${dd} ${partialTime}`
)
```

v2 版

```javascript
var localeDateTime = thing => new Intl.DateTimeFormat(
    'en-GB',
    {
        dateStyle:'short',
        timeStyle:'medium',
        timeZone: 'Asia/Shanghai',
    }
).format(new Date(thing)).replace(
    /^(\d{2})\/(\d{2})\/(\d{4}),\s(\d{2}):(\d{2}):(\d{2})$/,
    (_, dd, MM, yyyy, HH, mm, ss) => `${yyyy}-${MM}-${dd} ${HH}:${mm}:${ss}`
)
```

As of the `ES2018` RegExp named capture groups(since nodejs v10.8.0) available, the codes should more readable with v3:

v3 版

```javascript
var localeDateTime = (thing, timeZone = 'Asia/Shanghai') => new Intl.DateTimeFormat(
    'en-GB',
    {
        dateStyle:'short',
        timeStyle:'medium',
        timeZone,
    }
).format(new Date(thing)).replace(
    /^(?<dd>\d{2})\/(?<MM>\d{2})\/(?<yyyy>\d{4}),\s(?<HH>\d{2}):(?<mm>\d{2}):(?<ss>\d{2})$/,
    '$<yyyy>-$<MM>-$<dd> $<HH>:$<mm>:$<ss>'
)
```

v4 版

```javascript
var localeDateTime = (thing = Date.now(), timeZone = 'Asia/Shanghai') => Intl.DateTimeFormat.call(
    null,
    'en-GB',
    {
        dateStyle:'short',
        timeStyle:'medium',
        timeZone,
    }
).format(new Date(thing)).replace(
    /^(?<dd>\d{2})\/(?<MM>\d{2})\/(?<yyyy>\d{4}),\s(?<HH>\d{2}):(?<mm>\d{2}):(?<ss>\d{2})$/,
    '$<yyyy>-$<MM>-$<dd> $<HH>:$<mm>:$<ss>'
)
```

v5 版本

*Notes*: NodeJS v10.15.3,v12.18.0,v14.5.0 had strange behavior on `Intl.DateTimeFormat`, see `hourCycle:h23` comments below:

```javascript
var localeDateTime = (when = Date.now(), timeZone = 'Asia/Shanghai') {

    // short `when` like `Aug 9, 1995` is dependend on the `TZ` env
    process.env.TZ = timeZone

    let data = new Intl.DateTimeFormat(
      'en',
      {
        calendar: 'gregory', hourCycle: 'h11',
        year: 'numeric', month: '2-digit', day: '2-digit',
        hour: 'numeric', minute: '2-digit', second: '2-digit',
        timeZone,
      }
    ).formatToParts(
      new Date(when)
    ).filter(({type}) => type != 'literal').reduce(
      (des, {type, value}) => (des[type.toLowerCase()] = value.toUpperCase(), des),
      {}
    )

    // hourCycle:h23 fix(12 AM to 0, 12 PM to 0)
    data.hour = data.hour == '12' ? `0` : data.hour

    if (data.dayperiod == 'PM') {
      // hourCycle:h23 add(1 PM to 13)
      data.hour = `${12 + data.hour * 1}`
    } else if (data.dayperiod == 'AM') {
      // hourCycle:h23 fix(2-digit)
      data.hour = data.hour.length > 1 ? data.hour : `0${data.hour}`
    }

    return `${data.year}-${data.month}-${data.day} ${data.hour}:${data.minute}:${data.second}`
  }
```

Usage samples:

```javascript
console.info(localeDateTime())
```

```
'2020-09-20 08:38:54'
```

```javascript
console.info(localeDateTime(1600538598726))
```

```
'2020-09-20 02:03:18'
```

```javascript
console.info(localeDateTime('January 1, 1970, 00:00:00 UTC'))
```

```
'1970-01-01 08:00:00'
```

```javascript
console.info(localeDateTime(Date.now(), 'UTC'))
```

```
'2020-09-20 00:39:01'
```

```javascript
console.info(localeDateTime('2019-01-01'))
```

```
'2019-01-01 08:00:00'
```

```javascript
console.info(localeDateTime('2020-11-11 00:00'))
```

```
'2020-11-11 00:00:00'
```

```javascript
console.info(localeDateTime('Aug 9, 1995'))
```

```
'1995-08-09 00:00:00'
```

```javascript
console.info(localeDateTime(Date.UTC(2012, 11, 20, 3, 0, 0), 'America/Los_Angeles'))
```

```
'2012-12-19 19:00:00'
```

Ok~ It's enough.

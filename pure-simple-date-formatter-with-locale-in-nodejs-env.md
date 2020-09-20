---
title: "Pure Simple DateTime Formatter With Locale in Nodejs Env"
date: 2020-09-20T12:29:04+08:00
keywords: []
description: "Pure Simple Date Formatter With Locale in Nodejs Env"
tags: []
categories: ["javascript"]
author: "James"
summary: "It's a practice of the `Date` and `Intl.DateTimeFormat` usage regarding by BCP47 and rfc3339, Formatting the datetime in `yyyy-MM-dd HH:mm:ss` style."
---

It's a practice of the `Date` and `Intl.DateTimeFormat` usage sample. I'd researched some resources such as BCP47, rfc3339 about the `DateTimeFormat`. It's not easier to scope and use that `options` for me. So sad so bad.

In a special project, there were mentioned the `Date` should always as `yyyy-MM-dd HH:mm:ss` format. But that `key point` maybe have some `timezone` potential risks while the codes were running under the `BaaS` or `FaaS` AKA `Serverless` environment.

As of the `Moment.js` was under LTS situation, here's a pure simple way to do the formatting date and time with locale.

The `DateTimeFormat` of the `en-GB` locale was `dd/MM/yyyy, HH:mm:ss`, it's similar to `yyyy-MM-dd HH:mm:ss`. Here's need convert the `date` only. Codes below:

v1

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

v2

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

Great features of the `ES2018` RegExp named capture groups(since nodejs v10.8.0 available), the codes should should more readable with v3:

v3

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

final version 4

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

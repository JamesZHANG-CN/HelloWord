---
title: "A snowflake-style id generator for postgresql"
date: 2019-07-06T16:06:57+08:00
keywords: ["postgresql", "plpgql", "snowflake"]
description: "a pretty elegant, twitter snowflake style id_generator for postgresql"
tags: ["postgres"]
categories: ["数据库"]
author: "<a href=\"https://rob.conery.io/about-me/\">Rob Conery</a>"
summary: "很有参考意义的一个 plpgql id_generator 解决方案，可能后期会应用到pg数据库上。"
---

在各种群里潜水，看到一条消息说解决分布式数据库ID自增解决方案——twitter snowflake，遂搜索并学习了一下，发现真的挺好，ID算法能保证使用69年。。。够用了。。。够用了！

snowflake 算法原版见 [这里](https://github.com/twitter-archive/snowflake/blob/snowflake-2010/src/main/scala/com/twitter/service/snowflake/IdWorker.scala)，相当精妙的算法。

以下是摘录自 Rob Conery 的[英文博文](https://rob.conery.io/2014/05/28/a-better-id-generator-for-postgresql/)，实现得也相当赞，记录下来，以备不时之需。

```sql
create schema shard_1;
create sequence shard_1.global_id_sequence;

CREATE OR REPLACE FUNCTION shard_1.id_generator(OUT result bigint) AS $$
DECLARE
    our_epoch bigint := 1314220021721;
    seq_id bigint;
    now_millis bigint;
    -- the id of this DB shard, must be set for each
    -- schema shard you have - you could pass this as a parameter too
    shard_id int := 1;
BEGIN
    SELECT nextval('shard_1.global_id_sequence') % 1024 INTO seq_id;

    SELECT FLOOR(EXTRACT(EPOCH FROM clock_timestamp()) * 1000) INTO now_millis;
    result := (now_millis - our_epoch) << 23;
    result := result | (shard_id << 10);
    result := result | (seq_id);
END;
$$ LANGUAGE PLPGSQL;

select shard_1.id_generator();
```

```sql
create table shard_1.users(
  id bigint not null default id_generator(),
  email varchar(255) not null unique,
  first varchar(50),
  last varchar(50)
)
```

---
title: "Yet another alibaba canal tool in alpine"
date: 2019-10-16T20:55:42+08:00
keywords: [another mysql binlog synchronism tool]
description: "Yet another alibaba canal tool in alpine, less size of the deployer&adapter together. Easy bootstrap and usage."
tags: []
categories: [alibaba]
author: "James"
summary: "docker image of the mysql binlog synchronism tool in alpine"
---

## Why

The official docker containers should have a `PID 1` issue there, [this prject](https://github.com/TheNorthMemory/canal-docker) was made for try to solve this problem.

## Bootstrap

```bash
#!/bin/sh

JAVA_OPTS_EXT="-Djava.awt.headless=true -Djava.net.preferIPv4Stack=true -Dfile.encoding=UTF-8"
case ${@} in
  -a | a | -adapter | adapter)
    exec java ${JAVA_OPTS} ${JAVA_OPTS_EXT} com.alibaba.otter.canal.adapter.launcher.CanalAdapterApplication
  ;;
  -d | d | -deployer | deployer)
    exec java ${JAVA_OPTS} ${JAVA_OPTS_EXT} com.alibaba.otter.canal.deployer.CanalLauncher
  ;;
  -v | v | -version | version)
    echo "canal version: 1.1.4 d53bfd7 on java-${JAVA_VERSION}"
  ;;
  *)
    echo "usage: [[-][a[dapter]]|[d[eployer]]|[v[ersion]]]"
    echo ""
    echo "  adapter: launch the canal adapter, the workdir must be located in /alibaba/canal-adapter"
    echo "  deployer: launch the canal deployer, the workdir must be located in /alibaba/canal-deployer"
    echo "  version: display this build version"
    echo ""
    echo "shown this usage manual default"
  ;;
esac
```

## Example usage in k8s

it was commented on [here](https://github.com/alibaba/canal/issues/2114#issuecomment-527171994), copied here for record.

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: k8s-canal
data:
  canal.properties: |
    canal.zkServers = zookeeper:2181
    canal.withoutNetty = false
    canal.serverMode = tcp
    canal.destinations = sys_info
    canal.auto.scan = false
    canal.conf.dir = .
    canal.instance.binlog.format = ROW,STATEMENT,MIXED
    canal.instance.binlog.image = FULL,MINIMAL,NOBLOB
    canal.instance.tsdb.enable = true
    canal.instance.tsdb.dir = ${canal.conf.dir}/tsdb
    canal.instance.tsdb.url = jdbc:h2:${canal.instance.tsdb.dir}/h2;CACHE_SIZE=1000;MODE=MYSQL;
    canal.instance.tsdb.dbUsername = canal
    canal.instance.tsdb.dbPassword = canal
    canal.instance.tsdb.spring.xml = classpath:spring/tsdb/h2-tsdb.xml
    canal.instance.global.mode = spring
    canal.instance.global.lazy = false
    canal.instance.global.spring.xml = classpath:spring/default-instance.xml
  sys_info-instance.properties: |
    canal.instance.gtidon = false
    canal.instance.master.address = __RDS__:3306
    canal.instance.master.journal.name =
    canal.instance.master.position =
    canal.instance.dbUsername = reader
    canal.instance.dbPassword = "readER!"
    canal.instance.defaultDatabaseName = sys_info
    canal.instance.filter.regex = sys_info\\.jdp_tb_item
  application.yml: |
    server:
      port: 8081
    logging:
      level:
        org.springframework: WARN
        com.alibaba.otter.canal.client.adapter.rdb: WARN
    spring:
      jackson:
        date-format: yyyy-MM-dd HH:mm:ss
        time-zone: GMT+8
        default-property-inclusion: non_null
    canal.conf:
      zookeeperHosts: zookeeper:2181
      batchSize: 500
      syncBatchSize: 1000
      retries: 2
      mode: tcp
      canalAdapters:
      - instance: sys_info
        groups:
        - groupId: g1
          outerAdapters:
          - name: rdb
            key: pg_52_50
            properties:
              jdbc.url: jdbc:postgresql://__PG__:5432/analytics
              jdbc.username: writer
              jdbc.password: "writER!"
              threads: 10
              commitSize: 3000
  sys_info-jdp_tb_item.yml: |
    destination     : sys_info
    outerAdapterKey : sys_info
    concurrent      : true
    dbMapping:
      database    : sys_info
      table       : jdp_tb_item
      targetTable : sys_info.jdp_tb_item
      targetColumns:
        num_iid        : num_iid
        nick           : nick
        approve_status : approve_status
        has_showcase   : has_showcase
        created        : created
        modified       : modified
        cid            : cid
        has_discount   : has_discount
        jdp_hashcode   : jdp_hashcode
        jdp_response   : jdp_response
        jdp_delete     : jdp_delete
        jdp_created    : jdp_created
        jdp_modified   : jdp_modified
      targetPk:
        num_iid: num_iid
---
apiVersion: v1
kind: Service
metadata:
  name: zookeeper
  labels:
    app: zk
spec:
  ports:
  - port: 2181
    name: client
  clusterIP: None
  selector:
    app: zk
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  labels:
    app: ak
  name: zookeeper
spec:
  replicas: 1
  selector:
    matchLabels:
      app: zk
  template:
    metadata:
      name: zk
      labels:
        app: zk
    spec:
      volumes:
      - name: localtime
        hostPath:
          path: /usr/share/zoneinfo/Asia/Shanghai
      containers:
      - name: zookeeper
        image: zookeeper:3.3
        volumeMounts:
        - name: localtime
          mountPath: /etc/localtime
          readOnly: true
---
apiVersion: v1
kind: Service
metadata:
  name: deployer
  labels:
    app: canal-deployer
spec:
  ports:
  - port: 11111
    name: deployer
  - port: 11112
    name: metrics
  clusterIP: None
  selector:
    app: canal-deployer
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  labels:
    app: canal-deployer
  name: deployer
spec:
  minReadySeconds: 10
  selector:
    matchLabels:
      app: canal-deployer
  template:
    metadata:
      name: canal-deployer
      labels:
        app: canal-deployer
    spec:
      volumes:
      - name: localtime
        hostPath:
          path: /usr/share/zoneinfo/Asia/Shanghai
      - name: conf
        configMap:
          name: k8s-canal
      - name: tsdb
        hostPath:
          path: /opt/canal-h2-tsdb
      containers:
      - name: deployer
        image: thenorthmemory/canal:latest-alpine
        volumeMounts:
        - mountPath: /etc/localtime
          name: localtime
          readOnly: true
        - mountPath: /alibaba/canal-deployer/conf/canal.properties
          name: conf
          subPath: canal.properties
          readOnly: true
        - mountPath: /alibaba/canal-deployer/conf/sys_info/instance.properties
          name: conf
          subPath: sys_info-instance.properties
          readOnly: true
        - mountPath: /alibaba/canal-deployer/tsdb
          name: tsdb
        env:
        - name: TZ
          value: Asia/Shanghai
        - name: JAVA_OPTS
          value: "-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -Xmn138m -Xms256m -Xmx256m"
        args:
        - deployer
        ports:
        - containerPort: 11111
          protocol: TCP
        - containerPort: 11112
          protocol: TCP
        workingDir: /alibaba/canal-deployer
---
kind: Deployment
apiVersion: extensions/v1beta1
metadata:
  labels:
    app: canal-adapter
  name: adapter
spec:
  minReadySeconds: 10
  selector:
    matchLabels:
      app: canal-adapter
  template:
    metadata:
      name: canal-adapter
      labels:
        app: canal-adapter
    spec:
      volumes:
      - name: localtime
        hostPath:
          path: /usr/share/zoneinfo/Asia/Shanghai
      - name: conf
        configMap:
          name: k8s-canal
      containers:
      - name: adapter
        image: thenorthmemory/canal:latest-alpine
        volumeMounts:
        - mountPath: /etc/localtime
          name: localtime
          readOnly: true
        - mountPath: /alibaba/canal-adapter/conf/application.yml
          name: conf
          subPath: application.yml
          readOnly: true
        - mountPath: /alibaba/canal-adapter/conf/rdb/mytest_user.yml
          name: conf
          subPath: sys_info-jdp_tb_item.yml
          readOnly: true
        env:
        - name: TZ
          value: Asia/Shanghai
        - name: JAVA_OPTS
          value: "-XX:+UnlockExperimentalVMOptions -XX:+UseCGroupMemoryLimitForHeap -Xmn138m -Xms256m -Xmx256m"
        args:
        - adapter
        ports:
        - containerPort: 8081
          protocol: TCP
        workingDir: /alibaba/canal-adapter
```

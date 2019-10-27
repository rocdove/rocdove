---
title: Migration FAQ
categories: [blueking, migration]
date: 2019-10-24 09:09:19
tags: ["blueking","blueking_migration"]
draft: fasle
---
# 迁移后排障

## 服务状态异常

### status all 和 status module 状态不一致

./bkcec status all 中显示某个服务为exit状态，
./bkcec status module 中显示服务正常，而服务所在IP不同
举例：

    # ./bkcec status all
    [172.21.18.10] consul: RUNNING
    [172.21.18.10] zk: RUNNING
    ...
    [172.21.18.8] fta: EXIT
    ...

    # ./bkcec status fta
    # ./bkcec status fta
    [172.21.18.7] fta         common:apiserver                 RUNNING   pid 2472, uptime 0:08:12
    [172.21.18.7] fta         common:jobserver                 RUNNING   pid 2470, uptime 0:08:12
    [172.21.18.7] fta         common:logging                   RUNNING   pid 2475, uptime 0:08:12
    [172.21.18.7] fta         common:polling0                  RUNNING   pid 2471, uptime 0:08:12
    [172.21.18.7] fta         common:qos                       RUNNING   pid 2468, uptime 0:08:12
    [172.21.18.7] fta         common:scheduler0                RUNNING   pid 2469, uptime 0:08:12
    [172.21.18.7] fta         common:webserver                 RUNNING   pid 2473, uptime 0:08:12
    [172.21.18.7] fta         fta:collect0                     RUNNING   pid 2466, uptime 0:08:12
    [172.21.18.7] fta         fta:converge0                    RUNNING   pid 2463, uptime 0:08:12
    [172.21.18.7] fta         fta:job                          RUNNING   pid 2465, uptime 0:08:12
    [172.21.18.7] fta         fta:match_alarm0                 RUNNING   pid 2467, uptime 0:08:12
    [172.21.18.7] fta         fta:poll_alarm                   RUNNING   pid 2462, uptime 0:08:12
    [172.21.18.7] fta         fta:solution                     RUNNING   pid 2464, uptime 0:08:12

原因：
迁移后原先老机器上的/data/bkcec/.installed_module 文件中未去除已迁走的 module

解决方式：
登陆到老机器上，删除/data/bkcec/.installed_module 文件中已迁走的 module

## 域名解析失败

部署完成后发现gse.service.consul无法解析
查看是否有gse.json文件，如果没有手动添加： [重新同步渲染安装 consul]

    # cat /data/bkce/etc/consul.d/gse.json
    {
        "service": {
            "id": "gse-1",
            "checks": [
                {
                    "service_id": "gse-1",
                    "interval": "10s",
                    "script": "/data/bkce/bin/health_check/check_proc_exists -m gse"
                }
            ],
            "name": "gse",
            "enableTagOverride": false,
            "address": "10.1.11.14"    //修改成具体的ip地址
        }
    }

## consul 报错

服务迁移后，consul中对应的服务配置未删除，导致consul对迁移前的服务check失败。

解决方式：
在迁移前的机器上，移除不需要的配置。

- config_pah: $INSTALL_PATH/etc/consul.d/_service_.json

## kafka 迁移后无法启动

    [2019-10-12 18:25:12,117] INFO starting (kafka.server.KafkaServer)
    [2019-10-12 18:25:12,118] INFO Connecting to zookeeper on zk.service.consul:2181/common_kafka (kafka.server.KafkaServer)
    [2019-10-12 18:25:12,244] INFO Created zookeeper path /common_kafka (kafka.server.KafkaServer)
    [2019-10-12 18:25:12,698] INFO Cluster ID = uNx3OnVSQE2_OEYETtHb0A (kafka.server.KafkaServer)
    [2019-10-12 18:25:12,720] FATAL Fatal error during KafkaServer startup. Prepare to shutdown (kafka.server.KafkaServer)
    kafka.common.InconsistentBrokerIdException: Configured broker.id 1 doesn't match stored broker.id 2 in meta.properties. If you moved your data, make sure your configured broker.id matches. If you intend to create a new broker, you should remove all data in your data directories (log.dirs).
            at kafka.server.KafkaServer.getBrokerId(KafkaServer.scala:689)
            at kafka.server.KafkaServer.startup(KafkaServer.scala:194)
            at kafka.server.KafkaServerStartable.startup(KafkaServerStartable.scala:39)
            at kafka.Kafka$.main(Kafka.scala:67)
            at kafka.Kafka.main(Kafka.scala)
    [2019-10-12 18:25:12,722] INFO shutting down (kafka.server.KafkaServer)
    [2019-10-12 18:25:12,744] INFO shut down completed (kafka.server.KafkaServer)
    [2019-10-12 18:25:12,745] FATAL Fatal error during KafkaServerStartable startup. Prepare to shutdown (kafka.server.KafkaServerStartable)
    kafka.common.InconsistentBrokerIdException: Configured broker.id 1 doesn't match stored broker.id 2 in meta.properties. If you moved your data, make sure your configured broker.id matches. If you intend to create a new broker, you should remove all data in your data directories (log.dirs).
            at kafka.server.KafkaServer.getBrokerId(KafkaServer.scala:689)
            at kafka.server.KafkaServer.startup(KafkaServer.scala:194)
            at kafka.server.KafkaServerStartable.startup(KafkaServerStartable.scala:39)
            at kafka.Kafka$.main(Kafka.scala:67)
            at kafka.Kafka.main(Kafka.scala)
    [2019-10-12 18:25:12,746] INFO shutting down (kafka.server.KafkaServer)

问题原因：

    /data/bkce/service/kafka/config/server.properties中
    现在kafka配置的broker.id和原先值不一致。

    迁移kafka后kafka在install.conf中的顺序有变动，从新render kafka 会导致重新分配broker.id,
    致使配置的id和数据的id不一致，导致kafka无法启动。

解决办法：

    - 方式1： 修改为原先配置的id
    - 方式2： 将数据转移到正确的id下
    - 迁移kafka时，配置install.conf，保持为原先得IP顺序。来保证render kafka不会改变原先配置的broker.id

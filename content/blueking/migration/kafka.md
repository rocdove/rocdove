---
title: "Migration Kafka"
categories: [blueking, migration]
date: 2019-10-26T13:42:43+08:00
tags: ["blueking","blueking_migration"]
draft: fasle
---
# kafka 迁移

- kafka 为集群部署，一台一台迁移不影响使用

## stop

    ./bkcec stop kafka

## 删除安装标记

    ssh $KAFKA_IP "source /data/install/utils.fc; sed -i '/kafka/d' $INSTALL_PATH/.installed_module; grep kafka $INSTALL_PATH/.installed_module"
    sed -i '/kafka/d' /data/install/.bk_install.step; grep kafka /data/install/.bk_install.step"

    # 删除consul对应服务配置
    ssh ${KAFKA_IP} "source /data/install/utils.fc; rm -f $INSTALL_PATH/etc/consul.d/kafka.json"

## 配置install.config

- 注意配置时新加节点的位置最好紧挨着要替换掉的kafka机器IP。保持原有kafka顺序，使render后kafka的broker与原先一致
- 删除原ip所在行的 kafka
- 新增一行 $ip kafka ip为待迁移的机器IP
- 更新env: source utils.fc

## 新机器配置好中控机的 ssh 免密登陆

    ./configure_ssh_without_pass

## 同步文件

    ./bkcec sync common
    ./bkcec sync consul
    ./bkcec sync kafka

## 安装consul，并重启服务

    ./bkcec install consul
    
    ./bkcec stop consul
    ./bkcec start consul

## 安装 kafka

    ./bkcec install kafka
    ./bkcec initdata kafka

## 给新机器授予mysql权限（非新机器时无需执行）

    ./bkcec initdata mysql

## 重新渲染相关模块配置，并重启服务（注意启动顺序）

    ./bkcec render kafka

    ./bkcec start kafka

## 在CMDB面板中转移，迁移涉及到的机器到空闲机

转移到空闲机是为了在安装gse_agent时，能从新按照新的模块配置重新注册到CMDB。
为了能让服务器与模块的管理关系与实际保持一致，在从新安装gse_agent之前需要把
配置变动的机器置于蓝鲸业务的空闲机中。

## 重新安装gse-agent

    # 给新机器添加gse_agent，给模块配置变动的机器从新部署gse_agent.并注册到CMDB。
    ./bkcec install gse_agent

## 检查服务是否正常

- bkmonitor上是否正常采集到数据
- 节点管理上添加新节点agent

## 排障

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

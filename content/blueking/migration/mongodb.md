---
title: "Migration MongoDB"
categories: [blueking, migration]
date: 2019-10-26T13:42:43+08:00
tags: ["blueking","blueking_migration"]
draft: fasle
---
# mongodb 迁移

- 数据库服务的迁移需要同步原有数据
- 依赖 mongodb 的服务需要从新渲染配置：
  - cmdb

## 停止相关服务

    ./bkcec stop cmdb

## 备份数据库，并停止 mongodb

    auth=($(grep cmdb /data/install/.app.token))
    # 压缩备份（数据量大）
    # /data/src/service/mongodb/bin/mongodump -h $MONGODB_IP:$MONGODB_PORT -u ${auth[0]} -p${auth[1]} -d cmdb --gzip --archive=/tmp/mongodb_migration_bak.gz
    # 不压缩备份（数据量小）
    /data/src/service/mongodb/bin/mongodump -h $MONGODB_IP:$MONGODB_PORT -u ${auth[0]} -p${auth[1]} -d cmdb -o /tmp/mongodb_migration_bak
    ./bkcec stop mongodb

## 删除安装标记

    ssh $MONGODB_IP "source /data/install/utils.fc; sed -i '/mongodb/d' $INSTALL_PATH/.installed_module; grep mongodb $INSTALL_PATH/.installed_module"
    sed -i '/mongodb/d' /data/install/.bk_install.step; grep mongodb /data/install/.bk_install.step"

    # 删除consul对应服务配置
    ssh ${MONGODB_IP} "source /data/install/utils.fc; rm -f $INSTALL_PATH/etc/consul.d/mongodb.json"

## 配置install.config

- 删除原ip所在行的 mongodb
- 新增一行 $ip mongodb ip为待迁移的机器IP
- 更新env: source utils.fc

## 新机器配置好中控机的 ssh 免密登陆

    ./configure_ssh_without_pass

## 同步文件

    ./bkcec sync common
    ./bkcec sync consul
    ./bkcec sync mongodb

## 安装consul，并重启服务

    ./bkcec install consul
    
    ./bkcec stop consul
    ./bkcec start consul

## 安装 mongodb

    ./bkcec install mongodb

## 给新机器授予mysql权限（非新机器时无需执行）

    ./bkcec initdata mysql

## 初始化 mongodb ,生成mongod.key等文件

    ./bkcec initdata mongodb

## 重新渲染相关模块配置，并重启服务（注意启动顺序）

    ./bkcec render mongodb
    ./bkcec render cmdb

    ./bkcec start mongodb
    # 导入压缩备份
    # /data/src/service/mongodb/bin/mongorestore -h $MONGODB_IP:$MONGODB_PORT -u ${auth[0]} -p${auth[1]} --gzip --archive=/tmp/mongodb_migration_bak.gz
    # 导入备份数据
    /data/src/service/mongodb/bin/mongorestore -h $MONGODB_IP:$MONGODB_PORT -u ${auth[0]} -p${auth[1]} -d cmdb --dir /tmp/mongodb_migration_bak/cmdb
    
    ./bkcec start cmdb

## 在CMDB界面上，进入业务拓扑，把公共组件下的mysql模块下的老机器转移到空闲机

转移到空闲机是为了在安装gse_agent时，能从新按照新的模块配置重新注册到CMDB。
为了能让服务器与模块的管理关系与实际保持一致，在从新安装gse_agent之前需要把
配置变动的机器置于蓝鲸业务的空闲机中。

## 重新安装gse-agent

    # 给新机器添加gse_agent，给模块配置变动的机器从新部署gse_agent.并注册到CMDB。
    ./bkcec install gse_agent

## 检查服务是否正常

- bkmonitor上是否正常采集到数据
- 节点管理上添加新节点agent

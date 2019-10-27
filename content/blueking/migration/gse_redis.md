---
title: "Migration GSE and Redis"
categories: [blueking, migration]
date: 2019-10-26T13:42:43+08:00
tags: ["blueking","blueking_migration"]
draft: fasle
---
# gse and redis 迁移

- gse 与 redis 需要部署在同一台机器上
- gse 若需要跨云支持， gse 所在机器必须有外网 IP

- 依赖 gse and redis 服务需要从新渲染配置：
  - job
  - cmdb
- 依赖 gse and redis 但不需要渲染配置
  - bkdata

## 停止相关服务

    # 至少要停bkdata dataapi，不然bkdata dataapi 下的 update_cc_cache也会因为访问api时因cmdb停止无法鉴权通过而stop
    ./bkcec stop bkdata
    ./bkcec stop job
    ./bkcec stop cmdb
    ./bkcec stop gse
    ./bkcec stop redis

## 删除安装标记

    ssh $GSE_IP "source /data/install/utils.fc; sed -i '/gse/d' $INSTALL_PATH/.installed_module; grep gse $INSTALL_PATH/.installed_module"
    sed -i '/gse/d' /data/install/.bk_install.step; grep gse /data/install/.bk_install.step

    ssh $REDIS_IP "source /data/install/utils.fc; sed -i '/redis/d' $INSTALL_PATH/.installed_module; grep redis $INSTALL_PATH/.installed_module"
    sed -i '/redis/d' /data/install/.bk_install.step; grep redis /data/install/.bk_install.step

    # 删除consul对应服务配置
    ssh ${GSE_IP} "source /data/install/utils.fc; rm -f $INSTALL_PATH/etc/consul.d/{gse,redis}.json"

## 配置install.config

- 删除原ip所在行的 gse,redis
- 新增一行 $ip gse,redis ip为待迁移的机器IP
- 更新env: source utils.fc

## 新机器配置好中控机的 ssh 免密登陆

    ./configure_ssh_without_pass

## 同步文件

    ./bkcec sync common
    ./bkcec sync consul
    ./bkcec sync redis
    ./bkcec sync gse

## 安装consul，并重启服务

    ./bkcec install consul
    
    ./bkcec stop consul
    ./bkcec start consul

## 安装 redis, gse

    ./bkcec install redis
    ./bkcec install gse

## 给新机器授予mysql权限（非新机器时无需执行）

    ./bkcec initdata mysql

## 重新渲染相关模块配置，并重启服务（注意启动顺序）

    ./bkcec render cmdb
    ./bkcec render job
    ./bkcec render gse
    ./bkcec render redis

    ./bkcec start redis
    ./bkcec start gse
    ./bkcec start job
    ./bkcec start cmdb
    ./bkcec start bkdata

## 在CMDB面板中转移，迁移涉及到的机器到空闲机

转移到空闲机是为了在安装gse_agent时，能从新按照新的模块配置重新注册到CMDB。
为了能让服务器与模块的管理关系与实际保持一致，在从新安装gse_agent之前需要把
配置变动的机器置于蓝鲸业务的空闲机中。

## 重新安装gse-agent

    # 给新机器添加gse_agent，给模块配置变动的机器从新部署gse_agent.并注册到CMDB。
    ./bkcec install gse_agent

    # 所有proxy和pagent都需要检查状态，重新部署

## 检查服务是否正常

- bkmonitor上是否正常采集到数据
- 节点管理上添加新节点agent

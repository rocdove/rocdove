---
title: "Migration License"
categories: [blueking, migration]
date: 2019-10-26T13:42:43+08:00
tags: ["blueking","blueking_migration"]
draft: fasle
---
# license 迁移

- 依赖 license 服务不需要从新渲染配置：
  - paas
  - cmdb
  - job
  - paas-agent
- 需要先获取迁移目标地址的 mac
  - [获取证书](https://bk.tencent.com/download_ssl/)

## 停止相关服务

    # ./bkcec stop appo
    # ./bkcec stop appt
    # ./bkcec stop job
    # ./bkcec stop cmdb
    # ./bkcec stop paas
    ./bkcec stop license

## 备份原有证书, 并解压新证书

    cd /data/src/
    tar zcf cert_bk.tar.gz cert/
    cd /data/
    tar xvf ssl_certificates.tar.gz -C src/cert/

## 删除安装标记

    ssh $LICENSE_IP "source /data/install/utils.fc; sed -i '/license/d' $INSTALL_PATH/.installed_module; grep license $INSTALL_PATH/.installed_module"
    sed -i '/license/d' /data/install/.bk_install.step; grep license /data/install/.bk_install.step"

    # 删除consul对应服务配置
    ssh ${LICENSE_IP} "source /data/install/utils.fc; rm -f $INSTALL_PATH/etc/consul.d/license.json"

## 配置install.config

- 删除原ip所在行的 nginx
- 新增一行 $ip nginx ip为待迁移的机器IP
- 更新env: source utils.fc

## 新机器配置好中控机的 ssh 免密登陆

    ./configure_ssh_without_pass

## 同步文件

    ./bkcec sync common
    ./bkcec sync consul
    ./bkcec sync license

## 安装consul，并重启服务

    ./bkcec install consul
    
    ./bkcec stop consul
    ./bkcec start consul

## 安装 license

    ./bkcec install license

    # 重新安装GSE，否则会导致gse_agent无法正常安装     
    ./bkcec install gse 1

## 给新机器授予mysql权限（非新机器时无需执行）

    ./bkcec initdata mysql

## 重新渲染相关模块配置，并重启服务（注意启动顺序）

    # ./bkcec render appo
    # ./bkcec render appt
    # ./bkcec render job
    # ./bkcec render cmdb
    # ./bkcec render paas
    ./bkcec render license

    ./bkcec start license
    # ./bkcec start paas
    # ./bkcec start cmdb
    # ./bkcec start job
    # ./bkcec start appo
    # ./bkcec start appt

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

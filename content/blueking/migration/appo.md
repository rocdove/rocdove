---
title: "Migration APPO"
categories: [blueking, migration]
date: 2019-10-26T13:42:43+08:00
tags: ["blueking","blueking_migration"]
draft: false
---
# paas_agent (appt, appo) 迁移

## appo

- 依赖于 appo 需要重新部署的服务
  - saas-o
- appo 迁移需要更改nginx配置：
  - nginx

### 停止相关服务

    ./bkcec stop saas-o
    ./bkcec stop appo

### 删除安装标记

    ssh $APPO_IP "source /data/install/utils.fc; sed -i '/appo/d' $INSTALL_PATH.installed_module; grep appo $INSTALL_PATH.installed_module"
    sed -i '/appo/d' /data/install/.bk_install.step; grep appo /data/install/.bk_install.step

    # 删除consul对应服务配置
    ssh $APPO_IP "source /data/install/utils.fc; rm -f $INSTALL_PATH/etc/consul.d/appo.json"

### 配置install.config

- 删除原ip所在行的 appo
- 新增一行 $ip appo ip为待迁移的机器IP
- 更新env: source utils.fc

### 新机器配置好中控机的 ssh 免密登陆

    ./configure_ssh_without_pass

### 同步文件

    ./bkcec sync common
    ./bkcec sync consul
    ./bkcec sync appo

### 安装consul，并重启服务

    ./bkcec install consul
    
    ./bkcec stop consul
    ./bkcec start consul

### 给新机器授予mysql权限（非新机器时无需执行）

    ./bkcec initdata mysql

### 安装 appo

    ./bkcec install appo

### 删除老的 appo 信息，并初始化新部署的 appo

    # 去开发者中心-》服务器信息 ，删除老的信息
    ./bkcec initdata appo

注： 不预先删除老的appo信息，会遇到如下类似信息：
“api response: {"msg": "prod环境已有一台服务器10.1.16.4处于激活中, 无法再激活10.1.16.10"}”

### 重新渲染相关模块配置，并重启服务（注意启动顺序）(appo 需要 activate 操作)

    ./bkcec render appo
    ./bkcec render nginx
    ./bkcec stop nginx
    ./bkcec start nginx
    ./bkcec start appo
    ./bkcec activate appo

### 重新部署 saas-o

    ./bkcec install saas-o

### 在CMDB面板中转移，迁移涉及到的机器到空闲机

转移到空闲机是为了在安装gse_agent时，能从新按照新的模块配置重新注册到CMDB。
为了能让服务器与模块的管理关系与实际保持一致，在从新安装gse_agent之前需要把
配置变动的机器置于蓝鲸业务的空闲机中。

### 重新安装gse-agent

    # 给新机器添加gse_agent，给模块配置变动的机器从新部署gse_agent.并注册到CMDB。
    ./bkcec install gse_agent

### 检查服务是否正常

- bkmonitor上是否正常采集到数据
- 节点管理上添加新节点agent

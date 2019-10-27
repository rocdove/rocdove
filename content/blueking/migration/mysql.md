---
title: "Migration MySql"
categories: [blueking, migration]
date: 2019-10-26T13:42:43+08:00
tags: ["blueking","blueking_migration"]
draft: fasle
---
# MySQL服务迁移

- 数据库服务的迁移需要同步原有数据
- 依赖mysql的服务需要从新渲染配置：
  - paas
  - job
  - bkdata
  - fta
  - saas-o：重新部署已部署saas服务

## 停止相关服务

    ./bkcec stop saas-o
    ./bkcec stop fta
    ./bkcec stop bkdata
    ./bkcec stop job
    ./bkcec stop paas

## 备份数据库，并停止mysql

    mysqldump -h $MYSQL_IP -P $MYSQL_PORT -u $MYSQL_USER -p$MYSQL_PASS -A > /tmp/mysql_migration_bak.sql
    ./bkcec stop mysql

## 删除安装标记

    ssh $MYSQL_IP "source /data/install/utils.fc; sed -i '/mysql/d' $INSTALL_PATH/.installed_module; grep mysql $INSTALL_PATH/.installed_module"
    sed -i '/mysql/d' /data/install/.bk_install.step; grep mysql /data/install/.bk_install.step"

    # 删除consul对应服务配置
    ssh ${MYSQL_IP} "source /data/install/utils.fc; rm -f $INSTALL_PATH/etc/consul.d/mysql.json"

## 配置install.config

- 删除原ip所在行的mysql
- 新增一行 $ip mysql ip为待迁移的机器IP
- 更新env: source utils.fc

## 新机器配置好中控机的 ssh 免密登陆

    ./configure_ssh_without_pass

## 同步文件

    ./bkcec sync common
    ./bkcec sync consul
    ./bkcec sync mysql

## 安装consul，并重启服务

    ./bkcec install consul
    
    ./bkcec stop consul
    ./bkcec start consul

## 安装mysql，给新机器授予mysql权限（非新机器时无需执行）

    ./bkcec install mysql
    ./bkcec start mysql
    # initdata要在导入数据之前做，不然中控机无法远程访问mysql服务
    ./bkcec initdata mysql
    mysql -h $MYSQL_IP -P $MYSQL_PORT -u $MYSQL_USER -p$MYSQL_PASS < /tmp/mysql_migration_bak.sql

## 重新渲染相关模块配置，并重启服务（注意启动顺序）

    ./bkcec render mysql
    ./bkcec render paas
    ./bkcec render job
    ./bkcec render bkdata
    ./bkcec render fta

    ./bkcec start paas
    ./bkcec start job
    ./bkcec start bkdata
    ./bkcec start fta

## 在CMDB界面上，进入业务拓扑，把公共组件下的mysql模块下的老机器转移到空闲机

转移到空闲机是为了在安装gse_agent时，能从新按照新的模块配置重新注册到CMDB。
为了能让服务器与模块的管理关系与实际保持一致，在从新安装gse_agent之前需要把
配置变动的机器置于蓝鲸业务的空闲机中。

## 重新安装gse-agent

    # 给新机器添加gse_agent，给模块配置变动的机器从新部署gse_agent.并注册到CMDB。
    ./bkcec install gse_agent

## 从新安装saas-o，或者重新部署所有saas服务

    # 若saas使用了蓝鲸服务的mysql，则需要从新部署，来变更配置
    ./bkcec install saas-o

## 检查服务是否正常

- bkmonitor上是否正常采集到数据
- 节点管理上添加新节点agent

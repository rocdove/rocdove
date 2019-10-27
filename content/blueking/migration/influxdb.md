---
title: "Migration InfluxDB"
categories: [blueking, migration]
date: 2019-10-26T13:42:43+08:00
tags: ["blueking","blueking_migration"]
draft: fasle
---
# influxdb 迁移

- 数据库服务的迁移需要同步原有数据
- 依赖 influxdb 的服务需要从新渲染配置：
  - bkdata
  - gse

## 停止相关服务

```bash
./bkcec stop bkdata
./bkcec stop influxdb
```

## 备份数据库

```bash
rsync -av $INFLUXDB_IP:$INSTALL_PATH/public/influxdb /tmp/influxdb_migration_bak/
```

## 删除安装标记

```bash
ssh $INFLUXDB_IP "source /data/install/utils.fc; sed -i '/influxdb/d' $INSTALL_PATH/.installed_module; grep influxdb $INSTALL_PATH/.installed_module"
sed -i '/influxdb/d' /data/install/.bk_install.step; grep influxdb /data/install/.bk_install.step"

# 删除consul对应服务配置
ssh ${INFLUXDB_IP} "source /data/install/utils.fc; rm -f $INSTALL_PATH/etc/consul.d/influxdb.json"
```

## 配置install.config

- 删除原ip所在行的 influxdb
- 新增一行 $ip influxdb ip为待迁移的机器IP
- 更新env: source utils.fc

## 新机器配置好中控机的 ssh 免密登陆

```bash
./configure_ssh_without_pass
```

## 同步文件

```bash
./bkcec sync common
./bkcec sync consul
./bkcec sync influxdb
```

## 安装consul，并重启服务

```bash
./bkcec install consul

./bkcec stop consul
./bkcec start consul
```

## 安装 influxdb

```bash
./bkcec install influxdb
```

## 存入备份数据

```bash
source utils.fc
rsync -av /tmp/influxdb_migration_bak/influxdb $INFLUXDB_IP:$INSTALL_PATH/public/
ssh $INFLUXDB_IP "source /data/install/utils.fc; chown -R influxdb:influxdb $INSTALL_PATH/public/influxdb; ls -l $INSTALL_PATH/public/influxdb"
```

## 给新机器授予mysql权限（非新机器时无需执行）

```bash
./bkcec initdata mysql
```

## 重新渲染相关模块配置，并重启服务（注意启动顺序）

```bash
./bkcec render influxdb
./bkcec render bkdata

./bkcec start influxdb
./bkcec start bkdata
```

## 在CMDB界面上，进入业务拓扑，把公共组件下的mysql模块下的老机器转移到空闲机

转移到空闲机是为了在安装gse_agent时，能从新按照新的模块配置重新注册到CMDB。
为了能让服务器与模块的管理关系与实际保持一致，在从新安装gse_agent之前需要把
配置变动的机器置于蓝鲸业务的空闲机中。

## 重新安装gse-agent

```bash
# 给新机器添加gse_agent，给模块配置变动的机器从新部署gse_agent.并注册到CMDB。
./bkcec install gse_agent
```

## 检查服务是否正常

- bkmonitor上是否正常采集到数据
- 节点管理上添加新节点agent

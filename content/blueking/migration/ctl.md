---
title: "Migration CTL"
categories: [blueking, migration]
date: 2019-10-24 09:09:19
tags: ["blueking","blueking_migration"]
draft: true
---
# 中控机迁移

## 迁移到新机器(没有蓝鲸组件部署的机器)

1. 停掉所有服务
./bkcec stop all
2. 备份数据
tar zcvf data.tar.gz /data
3. 节点互信配置
4. 拷贝数据到新节点
scp data.tar.gz $ip:/
scp /root/.bkrc
5. 在新节点解压data.tar.gz到根目录下
tar zxvf data.tar.gz
然后替换IP:
cd /data
for i in `grep -rl "$(cat $CTRL_DIR/.controller_ip)" * --exclude-dir=$INSTALL_DIR/logs --exclude-dir=$INSTALL_DIR/public`;do sed -i 's/10.1.13.12/10.1.13.10/g' $i;done
6. consul
./bkcec install consul 1
./bkcec start consul
7. render
echo consul zk kafka mysql mongodb redis redis_cluster influxdb beanstalk rabbitmq es license gse nginx appt appo bkdata paas job cmdb | xargs -n1 ./bkcec render
8. upgrade
echo nginx appt bkdata| xargs -n1 ./bkcec upgrade
9. mysql
./bkcec start mysql
./bkcec initdata mysql
10. start

## zk报错启动不了:

```bash
rm -rf /data/bkce/public/zk/*
./bkcec install zk 1
./bkcec start zk
```

## es报错启动不了:

```bash
_add_user es /data/bkce/service/es/
chown -R es:es /data/bkce/service/es/*
chown -R es:es /data/bkce/logs/es/

echo consul zk kafka mongodb redis redis_cluster influxdb beanstalk rabbitmq es license gse nginx appt appo bkdata all | xargs -n1 ./bkcec start
```

11. 重新安装gse_agent

## 迁移到已部署有蓝鲸组件的机器

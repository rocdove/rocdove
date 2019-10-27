---
title: "Blueking Install"
categories: [blueking, install]
tags: ["blueking"]
date: 2019-10-26T12:35:01+08:00
draft: false
---

## 注意项

1. zk,kafka,consul：这三个服务要配置奇数个节点（3个节点）
2. gse,redis：这两个服务要放在同一个服务器上
3. bkdata：这个服务需要放置在高配置的服务器上（4c16G+）
4. nginx,gse,appo: 如果需要跨公网访问，这三个服务需要配置公网访问策略

## 安装步骤

### 配置 install.conf

配置IP，确认注意事项

### 配置 globals.env

* 域名
* 密码
* HTTP_SCHEMA
* PYPI_SOURCE='mirrors.cloud.tencent.com'
* PYPI_SOURCE='mirrors.tencentyun.com'

### 配置 pip 源

```bash
# 公网访问
cat <<EOF > /data/src/.pip/pip.conf
[global]
index-url = http://mirrors.cloud.tencent.com/pypi/simple
trusted-host = mirrors.cloud.tencent.com
EOF

# 腾讯内部访问
cat <<EOF > /data/src/.pip/pip.conf
[global]
index-url = http://mirrors.tencentyun.com/pypi/simple
trusted-host = mirrors.tencentyun.com
EOF
```

### 安装命令

```bash
./configure_ssh_without_pass
./precheck.sh -r

./bk_install paas
./bk_install cmdb
./bk_install job
./bk_install app_mgr
./bk_install bkdata
./bk_install fta
./bkcec install gse_agent
./bkcec install saas-o
```

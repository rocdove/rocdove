---
title: "Blueking Install ENV"
tags: ["blueking"]
date: 2019-10-26T12:31:42+08:00
draft: false
---

# 基础环境准备

## base

```bash
cat /etc/centos-release

setenforce 0

systemctl stop firewalld
systemctl disable firewalld

cat <<EOF > /etc/security/limits.d/99-nofile.conf
root soft nofile 204800
root hard nofile 204800
EOF

umask

echo "$http_proxy" "$https_proxy"
```

## 域名解析处理

```bash
cat <<EOF > /etc/hosts
127.0.0.1   localhost localhost.localdomain localhost4 localhost4.localdomain4
::1         localhost localhost.localdomain localhost6 localhost6.localdomain6

EOF

cat <<EOF >> /etc/sysconfig/network-scripts/ifcfg-eth0
NM_CONTROLLED="no"
PEERDNS="no"
EOF

cat <<EOF > /etc/resolv.conf
nameserver 127.0.0.1
nameserver 114.114.114.114
EOF
```

## yum 源处理

```bash

mkdir /etc/yum.repos.d/backup_roc
mv /etc/yum.repos.d/*.repo /etc/yum.repos.d/backup_roc

wget -O /etc/yum.repos.d/CentOS-Base.repo http://mirrors.cloud.tencent.com/repo/centos7_base.repo
wget -O /etc/yum.repos.d/epel.repo http://mirrors.cloud.tencent.com/repo/epel-7.repo
# 处理为腾讯内部源域名
sed -i 's/mirrors.cloud.tencent.com/mirrors.tencentyun.com/g' /etc/yum.repos.d/*.repo
```

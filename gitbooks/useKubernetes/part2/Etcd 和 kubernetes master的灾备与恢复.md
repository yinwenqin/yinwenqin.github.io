---
title: "Etcd和kubernetes master的灾备与恢复"
date: 2020/01/01 17:49:16
tags: 
- Kubernetes



---

# Etcd和kubernetes master的灾备与恢复

## 背景说明

问题：假设某台带有etcd的k8s master节点完全故障，彻底无法恢复

方案：新启动一台主机，配置为故障master主机相同的ip和主机名，并尝试原地恢复，顶替原故障master节点



## Etcd恢复

#### 参考官方文档：

https://github.com/etcd-io/etcd/blob/master/Documentation/op-guide/recovery.md#restoring-a-cluster

**注意**：从官方文档可得知，备份快照时会改变cluster id(改变cluster id不影响客户端，客户端是使用ip来连接访问的)，因此，需要etcd cluster的**每个成员**都参与备份与恢复。



####  检查

```shell
~# export ENDPOINTS="https://10.90.1.238:2379,https://10.90.1.239:2379,https://10.90.1.240:2379"

~# etcdctl --endpoints=$ENDPOINTS --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem --cacert=/etc/etcd/ssl/ca.pem  member list
316dbd21b17d1b4f, started, NODE238, https://10.90.1.238:2380, https://10.90.1.238:2379, false
8a206cf1ed53b6f4, started, NODE240, https://10.90.1.240:2380, https://10.90.1.240:2379, false
e444265665d3bd32, started, NODE239, https://10.90.1.239:2380, https://10.90.1.239:2379, false

~# etcdctl endpoint health --endpoints=$ENDPOINTS --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem
https://10.90.1.239:2379 is healthy: successfully committed proposal: took = 16.882627ms
https://10.90.1.240:2379 is healthy: successfully committed proposal: took = 17.046735ms
https://10.90.1.238:2379 is healthy: successfully committed proposal: took = 19.395533ms

~ # kubectl get nodes
NAME        STATUS   ROLES    AGE    VERSION
ubuntu238   Ready    master   123d   v1.15.3
ubuntu239   Ready    master   123d   v1.15.3
ubuntu240   Ready    master   123d   v1.15.3
```

确认正常无误

### 开始备份

#### 关停客户端

```shell
# 3台分别执行,二进制安装的关停相应组件服务
service kubelet stop
service docker stop
```

#### 备份快照

在每一台etcd节点上分别执行：

```shell
# 修改为对应节点的ip
export EP="https://10.90.1.238:2379"

export ETCDCTL_API=3

mkdir -p /var/lib/etcd_bak

etcdctl --endpoints=$EP --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem --cacert=/etc/etcd/ssl/ca.pem   snapshot save  /var/lib/etcd_bak/`hostname`-etcd_`date +%Y%m%d%H%M`.db

systemctl stop etcd

mv /var/lib/etcd /var/lib/etcd_old

```

拷贝文件：

```bash
scp -r /etc/etcd/ssl/* NEW_HOST
scp -r /etc/systemd/system/etcd.service  NEW_HOST
scp  /var/lib/etcd_bak/ubuntu238-etcd_202001011000.db  NEW_HOST
```



将备份出的快照文件、ssl认证文件、systemd启动脚本，拷贝到新主机上，同时，在新主机上安装好docker、kubelet、kubectl、etcd服务，安装步骤这里省略，由于这几个组件都是go语言的二进制程序，因此可以直接将原主机的执行文件scp到新主机上相同路径中，不必花费精力去找相应版本的rpm包。

### 恢复备份

在新加入的一台替换主机，和保持不变的另外2台主机上执行：

**必须每台成员主机全部执行，因为etcd协商的cluster id等一些元数据发生了改变**

```bash
# 这3个环境变量相应修改
export NODE_NAME=NODE238
export NODE_IP=10.90.1.238
export SNAPSHOT=ubuntu238-etcd_202001011000.db

export ETCDCTL_API=3
export ETCD_NODES="NODE238"=https://10.90.1.238:2380,"NODE239"=https://10.90.1.239:2380,"NODE240"=https://10.90.1.240:2380
export ENDPOINTS="https://10.90.1.238:2379,https://10.90.1.239:2379,https://10.90.1.240:2379"

cd /var/lib/etcd_bak/

etcdctl snapshot restore  $SNAPSHOT \
--name=$NODE_NAME \
--cert=/etc/etcd/ssl/etcd.pem \
--key=/etc/etcd/ssl/etcd-key.pem \
--cacert=/etc/etcd/ssl/etcd-ca.pem \
--initial-advertise-peer-urls=https://$NODE_IP:2380 \
--initial-cluster-token=etcd-cluster-0 \
--initial-cluster=$ETCD_NODES \
--data-dir=/var/lib/etcd/

```

完成以上步骤后，3台etcd主机上一起启动etcd并检查状态：

```bash
~ # service etcd start
~ # etcdctl endpoint health --endpoints=$ENDPOINTS --cacert=/etc/etcd/ssl/ca.pem --cert=/etc/etcd/ssl/etcd.pem --key=/etc/etcd/ssl/etcd-key.pem
https://10.90.1.240:2379 is healthy: successfully committed proposal: took = 16.112222ms
https://10.90.1.238:2379 is healthy: successfully committed proposal: took = 16.126138ms
https://10.90.1.239:2379 is healthy: successfully committed proposal: took = 17.345037ms
```

etcd恢复成功！

## K8s master恢复

如果是原主机不动，则无需操作，如果是替换为新主机，则需要将如下文件拷贝至新主机上(适用于kubeadm安装的k8s,二进制需要对应修改)：

- /etc/kubernetes/目录下的所有文件(证书，manifest文件)
- 用户(root)主目录下 /root/.kube/config文件(kubectl连接认证)
- /var/lib/kubelet/目录下所有文件(plugins、容器连接认证)
- /etc/systemd/system/kubelet.service.d/10-kubeadm.conf(kubelet systemd 启动文件)

复制完成后，启动服务:

```shell
service docker start
service kubelet start
```

检查：

```bash
kubectl get nodes
NAME        STATUS   ROLES    AGE    VERSION
ubuntu238   Ready    master   123d   v1.15.3
ubuntu239   Ready    master   123d   v1.15.3
ubuntu240   Ready    master   123d   v1.15.3
```

不出意外的话，一分钟左右master就恢复了。

## 总结

master恢复还是很简单的，没什么好多说的，etcd的恢复比较麻烦一些，需要全部成员重新initial一次，一定要按步骤来，仔细一些不要错。

**提示**：操作需要中断客户端写入，即停止k8s服务，风险高，生产慎用！
---
title: "Etcd 集群整体宕机的恢复方案"
date: "2025-05-11"
description: ""
categories: ["云平台"]
tags: ["etcd"]
series: ["灾备方案"]
ShowToc: true
TocOpen: false
---
 
# 背景

Kubernetes 集群目前并没有提供关机重启的选项，因此维护 etcd 集群的稳定至关重要，在生产环境下我推荐把 etcd 集群放在单独的区域中，并通过 systemd 管理。但即使这样，也有可能遇到极端事件导致集群整体宕机后无法启动。

那么，如果解决这个问题呢？本文将通过一个现实的案例，给出一个可行的 etcd 集群恢复方案。

# 案例

核心组件版本：

```.yaml 
Kubernetes: v1.31.7
etcd Version: 3.5.19
```

在我的个人 PC 中，[利用 Kuberspray 在 vm 中搭建了 3 节点的 Kubernetes 高可用集群]({{< relref "posts/cloud/kubernetes/kubespray.md" >}})，其中 etcd 模拟生产环境使用 systemd 管理。但由于频繁的关机、休眠，非常容易出现 etcd 集群整体不可用、Raft 状态错误或 WAL 损坏等情况。

这里我们要分两种情况：

### 第一种：还有 etcd 服务能够正常运行

针对这种就比较好办，可以从健康节点导出快照，并通过 peer 自动恢复即可，此处不做赘述。

### 第二种：所有节点 etcd 均无法正常运行

这时要做以下几件事。

1. 确认所有 etcd 服务关闭，并关闭 kubelet 和 kube-apiserver 服务避免干扰。
```.bash
systemctl stop etcd
systemctl stop kubelet
systemctl stop kube-apiserver
```

2. 备份原始 etcd 数据并清空数据目录，比如备份到 `/var/lib/etcd.bak`，并清空 `/var/lib/etcd/` 下的数据。

3. 在 每个节点 执行以下命令查看快照信息。
```.bash
etcdutl snapshot status /var/lib/etcd.bak/member/snap/db
```

输出结果：
```.bash
334118b6, 1768088, 3563, 18 MB
```

上面的输出分别表示：快照的哈希值、快照对应的 etcd revision（版本号）、快照中存储的 key 数量、快照文件大小。我们应选择 revision 最大的快照来执行恢复，最大程度减少数据损失。

5. 选定快照后，将该快照文件同步到其他机器对应目录下，并分别执行以下命令执行恢复：
```.bash
etcdutl snapshot restore /var/lib/etcd.bak/member/snap/db --data-dir /var/lib/etcd --name <当前节点的etcd的名称标识> \
    --initial-cluster etcd1=https://192.168.0.150:2380,etcd2=https://192.168.0.151:2380,etcd3=https://192.168.0.152:2380 \
    --initial-advertise-peer-urls https://<当前节点的IP地址>:2380 \
    --skip-hash-check=true
```

执行 `etcdutl snapshot restore` 后，通常会：
* 重建全新的 `--data-dir`，包括新的 `wal/` 和 `snap/` 目录
* 解析指定的快照文件，重建数据
* 按照 `--name`，`--initial-cluster`，`--initial-advertise-peer-urls` 参数生成新的 Raft 集群配置（确保参数和之前一致）

6. 接下来通过 systemd 重启服务。
```.bash
systemctl start etcd
```

7. 检查集群健康状态后，并顺序恢复 kubelet、kube-apiserver 等服务。
```.bash
etcdctl --endpoints=https://192.168.0.150:2379,https://192.168.0.151:2379,https://192.168.0.152:2379 \
  --cacert=/etc/ssl/etcd/ssl/ca.pem \
  --cert=/etc/ssl/etcd/ssl/admin-kube3.pem \
  --key=/etc/ssl/etcd/ssl/admin-kube3-key.pem \
  endpoint health
```

输出状态：
```.bash
https://192.168.0.150:2379 is healthy: successfully committed proposal: took = 17.176498ms
https://192.168.0.152:2379 is healthy: successfully committed proposal: took = 18.531198ms
https://192.168.0.151:2379 is healthy: successfully committed proposal: took = 19.581746ms
```

8. 顺序恢复其他服务。
```.bash
systemctl start kubelet
systemctl start kube-apiserver
```
# K3s 备份和恢复

K3s 的备份和恢复方式取决于使用的数据存储类型：

- 使用嵌入式 SQLite 数据存储进行备份和恢复
- 使用外部数据存储进行备份和恢复
- 使用嵌入式 etcd 数据存储进行备份和恢复

## 使用嵌入式 SQLite 数据存储进行备份和恢复

#### 方式1：备份/恢复数据目录

- 备份
```
# cp -rf /var/lib/rancher/k3s/server/db /opt/db
```

- 恢复
```
# systemctl stop k3s
# rm -rf /var/lib/rancher/k3s/server/db
# cp -rf /opt/db /var/lib/rancher/k3s/server/db
# systemctl start k3s
```

#### 方式2：通过 SQLite cli

- 备份
```
# sqlite3 /var/lib/rancher/k3s/server/db/state.db
SQLite version 3.22.0 2018-01-22 18:45:57
Enter ".help" for usage hints.
sqlite> .backup "/opt/kine.db"
sqlite> .exit
```

- 恢复
```
# systemctl stop k3s

# sqlite3 /var/lib/rancher/k3s/server/db/state.db
SQLite version 3.22.0 2018-01-22 18:45:57
Enter ".help" for usage hints.
sqlite> .restore '/opt/kine.db'
sqlite> .exit

# systemctl start k3s
```

## 使用外部数据存储进行备份和恢复

当使用外部数据存储时，备份和恢复操作是在 K3s 之外处理的。数据库管理员需要对外部数据库进行备份，或者从快照或转储中进行恢复。我们建议将数据库配置为执行定期快照。

- 备份

```
# mysqldump -uroot -p --all-databases --master-data > k3s-dbdump.db
```

- 恢复

停止K3s服务
```
# systemctl stop k3s
```

恢复mysql数据
```
mysql -uroot -p  < k3s-dbdump.db
```

启动K3s服务
```
# systemctl start k3s
```

## 使用嵌入式 etcd 数据存储进行备份和恢复

- 创建快照

K3s 默认启用快照。快照目录默认为 `/var/lib/rancher/k3s/server/db/snapshots`。要配置快照间隔或保留的快照数量，请参考：

| 参数                            | 描述                                                                                                                           |
| :------------------------------ | :----------------------------------------------------------------------------------------------------------------------------- |
| `--etcd-disable-snapshots`      | 禁用自动 etcd 快照                                                                                                             |
| `--etcd-snapshot-schedule-cron` | 以 Cron 表达式的形式配置触发定时快照的时间点，例如：每 5 小时触发一次`* */5 * * *`，默认值为每 12 小时触发一次：`0 */12 * * *` |
| `--etcd-snapshot-retention`     | 保留的快照数量，默认值为 5。                                                                                                   |
| `--etcd-snapshot-dir`           | 保存数据库快照的目录路径。(默认位置：`${data-dir}/db/snapshots`)                                                               |
| `--cluster-reset`               | 忘记所有的对等体，成为新集群的唯一成员，也可以通过环境变量`[$K3S_CLUSTER_RESET]`进行设置。                                     |
| `--cluster-reset-restore-path`  | 要恢复的快照文件的路径                                                                                                         |

- 从快照恢复集群

当 K3s 从备份中恢复时，旧的数据目录将被移动到`/var/lib/rancher/k3s/server/db/etcd-old/`。然后 K3s 会尝试通过创建一个新的数据目录来恢复快照，然后从一个带有一个 etcd 成员的新 K3s 集群启动 etcd。

要从备份中恢复集群，运行 K3s 时，请使用`--cluster-reset`选项运行 K3s，同时给出`--cluster-reset-restore-path`，如下：

```shell
./k3s server \
  --cluster-reset \
  --cluster-reset-restore-path=<PATH-TO-SNAPSHOT>
```

**结果:** 日志中出现一条信息，**Etcd 正在运行，现在需要在没有 `--cluster-reset` 标志的情况下重新启动。备份和删除每个对等 etcd 服务器上的 ${datadir}/server/db 并重新加入节点**

### S3 兼容 API 支持

K3s 支持向具有 S3 兼容 API 的系统写入 etcd 快照和从系统中恢复 etcd 快照。S3 支持按需和计划快照。

下面的参数已经被添加到 `server` 子命令中。这些标志也存在于 `etcd-snapshot` 子命令中，但是 `--etcd-s3` 部分被删除以避免冗余。

```
k3s etcd-snapshot \
  --s3 \
  # --s3-endpoint minio.kingsd.top:9000 \
  --s3-bucket=<S3-BUCKET-NAME> \
  --s3-access-key=<S3-ACCESS-KEY> \
  --s3-secret-key=<S3-SECRET-KEY>
```

要从 S3 中执行按需的 etcd 快照还原，首先确保 K3s 没有运行。然后运行以下命令：

```
k3s server \
  --cluster-init \
  --cluster-reset \
  --etcd-s3 \
  # --etcd-s3-endpoint minio.kingsd.top:9000 \
  --cluster-reset-restore-path=<SNAPSHOT-NAME> \
  --etcd-s3-bucket=<S3-BUCKET-NAME> \
  --etcd-s3-access-key=<S3-ACCESS-KEY> \
  --etcd-s3-secret-key=<S3-SECRET-KEY>
```
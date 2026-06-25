# ETCD 故障处理：解决 "etcdserver: no space" 写入报错
## 1. 问题描述
在 ETCD 运行过程中，遇到无法写入数据的故障，关联日志报错信息如下：
```
etcdserver: no space, 类似字眼基本上可以认定是etcd数据存储到达阈值无法写入
```
### 原因分析
ETCD 默认配置了 2GB 的存储配额（Quota）。当数据库文件大小超过该阈值时，为防止数据丢失和性能急剧下降，ETCD 会触发 NOSPACE 告警并自动进入只读模式，拒绝所有新的写入请求。


## 2. 处理步骤
### 步骤 0： 确认集群状态与告警
首先检查 ETCD 端点状态，确认是否存在 NOSPACE 告警。
``` bash
etcdctl --endpoints=http://etcd:2379 --write-out=table endpoint status

```
**预期输出**： 在 ALARM 列中显示 NOSPACE


### 步骤 1： 压缩键值历史版本 (Compact)
ETCD 保存了键值对的所有历史版本，压缩操作可清除旧版本数据，为碎片整理做准备。

**方法 A：分步执行**
1. 获取当前 Revision:
```bash
etcdctl --endpoints=http://etcd:2379 endpoint status --write-out="json"
```
2. 执行压缩:
```bash
# 将 [$revision] 替换为上一步获取的实际 revision 值
etcdctl --endpoints=http://etcd:2379 compact [$revision]
```

**方法 B：一键执行（推荐）**
通过命令组合自动获取最新 revision 并执行压缩
```bash
etcdctl --endpoints="http://etcd:2379" compact $(etcdctl --endpoints=http://etcd:2379 endpoint status --write-out="json" | egrep -o '"revision":[0-9]*' | egrep -o '[0-9].*')
```

### 步骤 2：碎片整理 (Defrag) 
压缩操作仅标记旧数据为删除状态，并未释放物理磁盘空间。必须执行碎片整理才能真正释放空间。
```bash
# 对每个 endpoint 分别执行
etcdctl --endpoints=http://etcd1:2379 defrag
etcdctl --endpoints=http://etcd2:2379 defrag
etcdctl --endpoints=http://etcd3:2379 defrag
```

### 步骤 3：解除告警并恢复写入
碎片整理完成且空间释放后，手动关闭 NOSPACE 告警以恢复集群写入能力
```bash
etcdctl --endpoints=http://etcd:2379 alarm disarm
```
执行后再次检查 endpoint status，确认 ALARM 列为空，即可验证服务恢复正常


## 3. 后续优化建议
| 优化项 | 说明 |
| :--- | :--- |
| 调整存储配额 | 根据业务增长预估，适当调大 `--quota-backend-bytes`（如设置为 4GB/8GB），避免频繁触顶 |
| 启用自动压缩 | 配置 `--auto-compaction-retention` 参数，让 ETCD 自动定期压缩历史版本 |
| 监控告警 | 接入 Prometheus + Grafana，对 `etcd_mvcc_db_total_size_in_bytes` 指标设置预警阈值（如达到配额的 80% 即告警） |
| 定期巡检 | 将 `endpoint status` 纳入日常巡检脚本，提前发现空间隐患 |

## 4. 参考链接
- [ETCD 官方文档 - Maintenance](https://etcd.io/docs/v3.6/op-guide/maintenance/)

- [ETCD Space Quota Configuration](https://etcd.io/docs/v3.6/dev-guide/limit/)
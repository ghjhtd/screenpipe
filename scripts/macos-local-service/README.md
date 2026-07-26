# macOS 常驻状态监控

这套轻量 App 用于在不运行官方桌面客户端的情况下，常驻启动本地
`screenpipe record` 引擎、显示菜单栏健康状态并执行有限度的自动恢复。

## 权限策略

- 状态监控 App 本身不申请录屏或麦克风权限，真正采集数据的是 `screenpipe` 引擎。
- 子进程始终设置 `SCREENPIPE_MANAGED_MODE=1`，禁止自动启动时弹出原生授权窗口或打开系统设置。
- 引擎缺少录屏或麦克风权限时以 `EX_NOPERM`（77）退出；监控器收到后停止自动重启，等待用户授权后手动重试。
- macOS 15 及以上只使用无副作用的预检，不调用旧式 `CGWindowListCreateImage` 权限探测。

## 外部配置

运行参数从 `~/.screenpipe/monitor/engine-arguments.json` 读取，格式参考
[`engine-arguments.example.json`](./engine-arguments.example.json)。修改参数后只需从菜单栏选择“重启录制引擎”，不需要重新编译或重新签名 App。

## 本机编译

```bash
swiftc -O scripts/macos-local-service/ScreenpipeLocalService.swift \
  -o ScreenpipeLocalService \
  -framework AppKit \
  -framework CoreGraphics \
  -framework Foundation
```

部署时应保持 `CFBundleIdentifier` 和签名证书稳定，避免 macOS TCC 将重新构建后的程序识别成新的权限主体。

## NAS 完整归档

[`screenpipe-nas-archive`](./screenpipe-nas-archive) 不只同步媒体文件，还会：

- 用 SQLite online backup API 生成可恢复的一致性 `db.sqlite`；
- 维护不会随本机清理而丢失路径的 `archive-catalog.sqlite`；
- 在 NAS 的 Btrfs 上保留 latest/previous 两代数据库快照；
- 仅在三类数据都成功落盘后写入 `archive_complete_through` 回执。

将该回执通过 `--retention-archive-receipt` 交给录制引擎后，本机清理只会处理回执已经覆盖的时间范围。NAS 离线或数据库备份失败时，清理会自动停住，不会为了满足保留天数而牺牲尚未归档的数据。

媒体和映射目录默认每 15 分钟增量同步；累计数据库每 6 小时生成一次一致性快照，避免每轮都写入数 GB 临时文件而消耗本机 SSD。回执时间取三类归档中最慢的一项，因此这一优化不会放宽本机清理的安全边界。

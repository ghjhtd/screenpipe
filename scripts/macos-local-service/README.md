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

## NAS 增量长期归档

[`screenpipe-nas-archive`](./screenpipe-nas-archive) 配合素渊 `memory-sync.py` 回传新增原始记录、后续修改和媒体。不再每六小时复制累计数据库；NAS 以一次性历史基线和永久增量保留历史，不周期新建全量 generation/月锚点。

记录由现有十分钟 LaunchAgent 回传，媒体与小型 catalog 每十五分钟补传。修改日志在 NAS 事务确认后才清退；丢回执安全重放，断网/关机后恢复续传。NAS 不随 Mac 删除原件。

`archive_complete_through` 需验证目标记录水位、修改序号与最旧未完成媒体，不把元数据成功等同于媒体全部完成。录制引擎仍通过 `--retention-archive-receipt` 限制本地清理；无法验证新水位则继续保留最近成功值。

安装需同时切换素渊 helper、Mac 脚本、NAS 维护代码与 `_service/state/archive-policy.json` 的 `archive_mode=incremental`。永久增量库属于备份原件，不能当普通缓存删除。既有全量历史保留，A/B 继续按 OPC 日程备份 NAS。

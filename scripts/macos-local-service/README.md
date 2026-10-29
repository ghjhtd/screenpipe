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

`archive_complete_through` 需验证目标记录水位、修改序号与最旧未完成媒体，不把元数据成功等同于媒体全部完成。录制引擎使用 `--retention-days 0`，本机七天记录和媒体由素渊 `memory-retention.py` 在归档成功后统一清理：NAS 永久基线和增量逐批内容核验、媒体 SHA-256 一致后才删除本机副本。未确认数据与近期引用的旧锚点保留，离线时允许暂时超过七天。新库启用增量空页回收；已有累计大库只能通过一次性核验迁移缩小，不能直接删除。

引擎使用 `--disable-snapshot-compaction` 保留原始 JPEG，避免已发现的压缩后尾帧关联缺失；已有 MP4 继续可读。恢复压缩须先独立修复并验证。部署必须同时安装素渊 `deploy/screenpipe-memory/` 下的 `memory_retention.py`、`memory-retention.py`、新版 `memory_incremental.py`、`memory_transport.py` 与 `catalog-sync.py`，NAS 安装只读核验 helper；缺少 helper 时归档仍保留数据，清理失败待修，不能回落到旧 media-only 清理。

安装需同时切换素渊 helper、Mac 脚本、NAS 维护代码与 `_service/state/archive-policy.json` 的 `archive_mode=incremental`。永久增量库属于备份原件，不能当普通缓存删除。既有全量历史保留，A/B 继续按 OPC 日程备份 NAS。

## 2026-10-29 回收闭环修复

此处归档入口保持与素渊 `deploy/screenpipe-memory/screenpipe-nas-archive` 的版本化副本一致；回收 helper 的唯一源码在素渊同目录。本轮素渊修复提交为 `a733465`。

回收使用完整待传 ID，先校验并处理过期媒体，再处理记录；新上传失败不再饿死已有 NAS 证明的回收。媒体定位的历史差异仅在有原件证明时兼容，缺证的单独记录保留并重算其余候选。上传、rsync 与核验复用同一个私有 SSH 控制连接，保持已有目标和身份校验。

Mac 需同时安装 `memory_transport.py`；NAS 需先更新 `memory_incremental.py` 和 `memory_retention.py`，再更新本机 helper 和本入口。78 项相关回归、Python 静态检查与 zsh 语法检查通过。完整水位仍受早期缺图约束，不把分批清理成功说成历史原件全部完整。

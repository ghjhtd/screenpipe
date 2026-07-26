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

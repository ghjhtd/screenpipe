# Screenpipe 归档控制台

面向个人长期数字记忆的 Tailnet 内网控制台。它将 Screenpipe 在 NAS 上的长期归档 API 映射成三个主要工作面：

- **回溯**：按时间、内容类型和关键词检索屏幕、音频与交互证据。
- **询问**：以证据检索为核心组织问题和可追溯来源；当前不伪装成已接入大模型。
- **档案**：查看跨年归档密度、不可变数据库分代、媒体与索引健康状态。

其余页面覆盖会议、记忆、Pipes、连接和系统能力。浏览器只访问同源 BFF；NAS Bearer Token 只保存在服务端，不会下发到前端。

## 本地验证

```bash
bun install
bun run build
bun test
SCREENPIPE_ARCHIVE_API_KEY_FILE=/path/to/api-key bun run serve
```

默认监听 `127.0.0.1:3035`。生产部署通过 `CONSOLE_HOST`、`CONSOLE_PORT` 和 `SCREENPIPE_UPSTREAM` 覆盖。

## Destiny 部署合同

- 服务地址：`http://100.64.0.7:3035`
- 上游归档：`http://100.64.0.1:3030`
- 密钥文件：`/opt/screenpipe-archive-console/shared/api-key`
- Node 运行时：`/opt/screenpipe-archive-console/runtime/node`（从 Destiny 已验证的 Node 22 固化，避免依赖用户 Home 下的 nvm 路径）
- 网络边界：只发布到 Destiny 的 Tailscale 地址，不绑定公网网卡
- 运行方式：主路径为 `screenpipe-archive-console.service`；`compose.destiny.yaml` 保留为容器化备选

构建前必须先运行 `bun run build && bun test`。部署时将本目录同步到一个不可变 release 目录，只安装生产依赖，再令 `/opt/screenpipe-archive-console/current` 指向该 release 并重启 systemd 服务。若 Docker Hub 可用，也可以从 current 执行 Compose；两种方式使用相同的只读前端构建、BFF 和服务端密钥文件。

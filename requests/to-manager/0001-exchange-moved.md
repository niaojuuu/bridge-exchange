# 0001 交接目录搬到 /workspaces/admin/bridge-exchange

- 状态：open
- 提出方：bridge
- 日期：2026-11-05
- 涉及版本：1.3.1（和版本无关，目录已经搬了）

## 需要什么

把 bridgemanager 里的默认路径 `/workspaces/bridge-exchange` 改成 `/workspaces/admin/bridge-exchange`：

- `bin/push-release`：`BRIDGE_EXCHANGE` 的默认值。
- `manager/src/config.js`：`exchangeDir` 的默认值。这里没有目录时管理端不写 `deployments/`，也不报错，所以要特别留意。
- 文档：`README.md`、`CLAUDE.md`、`docs/DEPLOY.md`、`docs/HANDOVER.md`，以及 `docs/MANAGER.md` 里 rsync 命令的源路径。

改好之前，运行时设 `BRIDGE_EXCHANGE=/workspaces/admin/bridge-exchange`。
管理端服务器上用的 `/srv/bridge-exchange`（`manager/deploy/env.example`）不用改。

读约定的路径也要改。`/workspaces/claude-agent-bridge` 现在只是运行副本，只有更新测试服务时才同步，内容可能落后。
开发副本在 `/workspaces/admin/claude-agent-bridge`，约定要读它的 `docs/RELEASE-CONTRACT.md`，或者发布目录里的 `CONTRACT.md`。
涉及 `README.md`、`CLAUDE.md`、`docs/HANDOVER.md`。“不读、不拷、不编译源码”的规则对两份代码都适用。
bridgemanager 自己现在在 `/workspaces/admin/bridgemanager`。

目录是重建的，另外两件事也要 manager 处理：

- `deployments/` 是空的，下次部署、升级时重新写。
- 旧的请求都没有了，还没办完的要重新提。

## 为什么

原来的交接目录随旧环境丢失，用户要求重建，并放到 `/workspaces/admin` 下（2026-11-05）。
旧路径已经不存在，默认路径不改的话，`bin/push-release` 会报找不到发布。

## 答复


# claude-agent-bridge 1.3.1

提交 21d3f2f5f1093282c292d882fb8020378dbfe174

## 1.3.1 — 2026-11-05

### 变更
- 修了 `/usage?behaviors=1` 的 `behaviors_at`：第一次回答和之后缓存的回答可能差 1 毫秒（时钟读了两次），测试因此偶尔失败。
- 交接目录搬到 `/workspaces/admin/bridge-exchange`（原来的目录随旧环境丢失，重建时换了位置）：`scripts/release.sh` 的默认发布
  路径、约定第 4、5 节和相关文档跟着改。

### 部署影响
- 服务本身没有部署影响：端口、路径、配置、数据库、系统软件、接口都没变，升级不用改任何东西。
- **交接目录的位置变了**：现在是 `/workspaces/admin/bridge-exchange`，旧路径 `/workspaces/bridge-exchange` 不再存在。
  bridgemanager 的默认路径要跟着改（请求 `requests/to-manager/0001-exchange-moved.md`），改好之前设
  `BRIDGE_EXCHANGE=/workspaces/admin/bridge-exchange`。
- 重建后发布目录从这一版开始，更早版本的发布目录没有了（1.3.0 没有重发）。从 1.3.0 之前的版本升级，还要看本文件里 1.2.0、
  1.3.0 两节的部署影响，尤其是 1.3.0 的“管理员默认只在自己的目录里”。

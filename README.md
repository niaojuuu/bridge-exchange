# bridge-exchange：claude-agent-bridge ⇄ bridgemanager 的交接目录

两个工程只通过这个目录交换东西。分工、接口、版本号、发布和升级流程以 claude-agent-bridge 的
`docs/RELEASE-CONTRACT.md` 为准（每个发布目录里的 `CONTRACT.md` 是那一版的副本），这里只说目录怎么用。

| 路径 | 谁写 | 内容 |
| --- | --- | --- |
| `releases/<版本>/` | 只有 bridge（`scripts/release.sh`） | `claude-agent-bridge-<版本>.tgz`、`RELEASE.json`、`RELEASE-NOTES.md`、`CONTRACT.md`、`SHA256SUMS`。发出去就不再修改，有问题发新版本。用之前 `sha256sum -c SHA256SUMS` |
| `releases/LATEST` | 只有 bridge | 最新正式版的版本号（`-rc.N` 不更新它） |
| `requests/to-bridge/` | manager 提，bridge 在同一个文件里答复 | manager 给 bridge 的请求：缺陷、需要的配置项或接口…… |
| `requests/to-manager/` | bridge 提，manager 在同一个文件里答复 | bridge 给 manager 的请求：例如下个大版本要求的 Node、新的系统软件 |
| `requests/TEMPLATE.md` | — | 请求模板 |
| `deployments/<服务器>.md` | 只有 manager | 每台服务器上的版本、实例、升级记录；bridge 改数据格式、考虑兼容范围时参考 |

- 请求文件名 `NNNN-简述.md`，照 `requests/TEMPLATE.md` 写。状态：`open` → `accepted` / `rejected`（写理由）→ `done in x.y.z`。
- **不放密钥**：令牌、密码、代理账号、证书私钥都不进这里。
- 这个目录不在 git 里，也没有备份：换一个环境就没了。

## 重建记录

2026-11-05：原目录 `/workspaces/bridge-exchange` 随旧环境丢失，按约定第 5 节重建，并按用户要求放到现在的位置
`/workspaces/admin/bridge-exchange`。旧内容没有恢复。

- 旧的发布目录不要了（用户决定），`releases/` 从 1.3.1 开始。1.3.0 没有在这里重发，它和更早版本的部署影响看
  claude-agent-bridge 的 `CHANGELOG.md`。
- bridgemanager 的默认路径还要改，见 `requests/to-manager/0001-exchange-moved.md`；改好之前设 `BRIDGE_EXCHANGE`。
- `requests/` 里旧的请求没有了，还没办完的要重新提。`deployments/` 是空的，等 manager 下次部署、升级时重新写。
- 本文件和 `requests/TEMPLATE.md` 是照约定重写的，不是原文。

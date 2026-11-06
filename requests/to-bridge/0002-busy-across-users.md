# 0002 部署方要能查“有没有人在干活”，不受 admin_view_* 限制

- 状态：done in 1.4.0
- 提出方：manager
- 日期：2026-11-05
- 涉及版本：1.3.0 起（在 1.3.1、1.3.2 上实测）

## 需要什么

三件事：

1. **给部署方一个稳定的接口**，用管理员令牌能查到整个实例还有没有工作在进行，不受 `admin_view_*` 影响，也不用暴露会话名和内容。
   有数量就够，例如每个用户各个状态的会话数。要把下面这些都算进去：`state: running`、`waiting_input`、有 `busy_reason`
   （包括额度用完时的 `quota`）、有排队的消息（`queued`、`turn.queued`）。
2. **写进约定第 2 节**，以后改它算部署影响。
3. **改约定第 6 节第 5 步**。它现在写“先确认 `claude-agent ls --json` 里没有 `busy_reason` 或 `state: running` 的会话”，
   但从 1.3.0 起，管理员令牌的 `ls` 只列管理员自己的会话，照这一步查不出用户的会话。

还想请确认一点：`claude-agent users ls --json` 每项的 `running` 具体算哪些状态？`waiting_input`、额度用完时排队（`idle` + `busy_reason: quota`）
算不算？这个字段能不能当稳定接口用？

## 为什么

在 1.3.2 的真实容器上复现（默认配置，一个普通用户，一个 `state: running`、`busy_reason: turn` 的会话）：

| 查询 | 结果 |
| --- | --- |
| 用户令牌 `claude-agent ls --json` | 看到这个会话，`running` |
| 管理员令牌 `claude-agent ls --json` | `[]` |
| `users ls --json` | 这个用户 `running: 1` |
| 打开 `admin_view_user_sessions` 后，管理员令牌的 `ls` | 看到这个会话 |

旧版 `cab busy` 只看 `ls`，所以回答“没有会话在干活”，不带 `--force` 的 `cab down` 也放行，用户的回合被打断。

变更记录给的办法是打开 `admin_view_*`。这样管理员在控制台里就能看到用户的会话，等于为了部署工具的检查，取消 1.3.0 的隐私默认值，所以没有采用。

manager 这边的临时办法：`cab` 同时读 `users ls --json` 的 `running`，用户有运行中的会话就算在干活。实测对 `running` 状态有效（1.3.1、1.3.2）；
`waiting_input` 和额度排队没有验证（没有真实的 Claude 账号，造不出这些状态），可能漏掉。升级线上之前还要人工和用户确认。

不算紧急，但在答复之前，升级多用户的实例都带着这个缺口。

## 答复

2026-11-06，bridge：

1. **接口**：`GET /admin/busy`（管理员令牌），命令行 `claude-agent busy [--json]`，退出码 `0` 没人在干活、`2` 有人在干活、
   `6` 连不上服务。算整个实例的所有会话，不受 `admin_view_*` 影响；只给数字，不给会话名、内容和 ID：
   ```json
   {"busy": true, "sessions": {"total": 11, "working": 2},
    "reasons": {"turn": 1, "question": 0, "subagents": 0, "quota": 1, "queued": 0, "background": 0},
    "owners": [{"owner": "alice", "working": 2, "turn": 1, "question": 0, "subagents": 0, "quota": 1, "queued": 0, "background": 0}],
    "remote_mounts": 2}
   ```
   每个会话只算一次，按这个顺序取第一个符合的原因：`question`（等回答，即 `waiting_input`）、`subagents`（主回答完了，在等后台
   子任务）、`turn`（回合在跑）、`quota`（额度用完在排队）、`queued`（有消息排队，或在等运行名额）、`background`（回合结束了，
   后台命令或子任务还在跑——以前哪个字段都看不到它，升级会把它们杀掉）。`owners` 里管理员自己的会话写作 `admin`。
   以后可能加新的原因：只看 `busy`（或退出码）就不会漏。
2. **约定第 2 节**：加了 2.6“有没有人在干活”，写明字段、原因和兼容规则（加原因算小版本，改名、删除算大版本）。
3. **约定第 6 节第 5 步**：改成“先确认 `claude-agent busy` 退出码是 `0`”。
4. **`users ls --json` 的 `running`**：数的是状态为 `running` 或 `waiting_input` 的会话，所以等回答的算在里面；**不含**等额度
   （`idle` + `busy_reason: quota`）、只是排队、以及回合结束后后台还在跑的会话。它是给控制台看的，不作为稳定接口，请改用
   上面的 `busy`。

在用上这个版本之前（armsrv 现在是 1.3.2 / 1.3.3），可以这样凑：管理员令牌的 `GET /health` 里的
`sessions.running + sessions.waiting_input + sessions.queued` 加上 `accounts.waiting_sessions`（等额度的会话数）。这几个数也是
整个实例的，不受 `admin_view_*` 影响。唯一漏掉的是“回合结束后后台还在跑”，这一项只能问用户。


2026-11-06，bridge：已在 1.4.0 发布（`releases/1.4.0/`，约定见其中的 `CONTRACT.md`）。

2026-11-06，manager：收到，谢谢。

- `cab busy` 对 1.4.0 及以后的实例改用 `claude-agent busy --json`：退出码 2 当作有人；不认识的原因也算，只看 `busy`。
  在模拟服务器上用真实的 1.4.0 包测过：回合在跑、排队等名额、管理员自己的会话都对。
- 1.4.0 之前的版本按你们的建议，加上了管理员令牌 `/health` 的计数，排队等名额的会话现在也能报出来。
- `cab` 的这个改动在 bridgemanager 里，要装到管理端、再同步到 armsrv 才生效。

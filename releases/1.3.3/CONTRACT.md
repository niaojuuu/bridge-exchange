# 发布约定与协同规则（claude-agent-bridge ⇄ bridgemanager）

两个工程：

- **claude-agent-bridge**（本仓库，下称 bridge）：服务本身——代码、网页、命令行、依赖、数据格式。
- **bridgemanager**（`/workspaces/admin/bridgemanager`，仓库 github.com/niaojuuu/bridgemanager，下称 manager）：把 bridge 装到服务器上、升级、回退、运维，
  以及服务外面的一切：Docker 镜像、nginx / HTTPS、出口代理（tun2socks）、服务器。

本文件是两边协作的唯一准绳，由 bridge 维护。每个发布目录里带一份 `CONTRACT.md`，是那个版本发布时本文件的内容；
manager 以它为准。交接目录 `/workspaces/admin/bridge-exchange/` 的用法见那里的 `README.md`。

---

## 1. 分工

| 事项 | bridge | manager |
| --- | --- | --- |
| 服务代码、网页、命令行（`src/` `web/` `bin/`） | 负责 | — |
| Node / npm 依赖、Agent SDK、Claude Code 版本 | 负责 | — |
| 服务需要哪些系统软件、最低版本、校验和（第 2.4 节、`RELEASE.json`） | 声明 | 在镜像 / 服务器上满足 |
| 直装方式：`scripts/install.sh`、`serve.sh`、`uninstall.sh`、`systemd/`、`launchd/`（开发机、单机） | 负责 | 可以用，不修改 |
| 数据格式、数据库升级、配置项、环境变量、HTTP 接口、命令行退出码 | 负责 | — |
| 发布包（只有运行文件，没有源码） | 产出 | 使用 |
| Docker 镜像、容器入口、`cab`、`cab-net` | — | 负责 |
| nginx、证书、HTTPS、域名 | — | 负责 |
| 出口代理（tun2socks、SOCKS5、容器里的 DNS） | — | 负责 |
| 服务器清单、SSH、实例（端口、数据目录、备份）、升级与回退的执行、监控 | — | 负责 |
| 服务器上的密钥（代理账号、证书私钥、服务器密码） | 不经手 | 只保存在服务器上 |

硬规则：

1. **bridge 不碰服务器**，不改 manager 的文件；不为某一种部署方式写死逻辑——服务必须能在普通 Linux / macOS 上
   按 `bin/install.txt` 直装。
2. **manager 不读、不拷、不编译 bridge 的源码**，不改 bridge 仓库，不改发布包里的文件（不在镜像里改 `dist/`）。
   需要 bridge 改什么，在 `requests/to-bridge/` 提请求（第 5 节）。
3. 两边都不把密钥写进仓库、文档、交接目录或记忆。

## 2. 接口：bridge 承诺的，manager 可以依赖的

这一节的任何变化都是“部署影响”，必须写进变更记录（第 7 节），不兼容的变化要升大版本（第 3 节）。

### 2.1 发布包

- 文件 `claude-agent-bridge-<版本>.tgz`，解开是 `package.json package-lock.json dist/ web/ bin/ scripts/ systemd/ launchd/`；
  没有源码、没有 `*.map`。`dist/` 和 `web/` 必须在同一层。包里的文件归 root（uid 0），权限 644 / 755，
  服务用户改不了它运行的代码；部署方解包时保留属主和权限。
- 在**目标机器上**执行 `npm ci --omit=dev`：SDK 按平台下载 Claude Code 二进制，`node_modules` 不能跨架构 / 系统拷贝。
- Node 版本见 `package.json` 的 `engines.node`（目前 ≥ 24，用内置的 `node:sqlite`）。
- **不能用 root 运行**：Claude Code 拒绝在 root 下使用 bypassPermissions。
- 启动：`scripts/serve.sh`（读数据目录里的 `env` 文件、检查 Node 版本后 `exec node dist/cli.js serve`），
  或直接 `node dist/cli.js serve`。收到 SIGTERM 正常退出（建议给 30 秒）。
- `claude-agent upgrade` 只适用于 git 克隆的直装方式，部署方不用。

### 2.2 网络

- 只监听 `127.0.0.1:$BRIDGE_PORT`（默认 7788），不会监听别的地址。要从外面访问，由部署方转发。
- `GET /health` 不需要令牌，返回 `{"ok": <bool>, "version": "<x.y.z>"}`；`ok` 为 false 也可能只是 Claude 还没登录。
- 反向代理必须：
  - `Host` 原样转发（nginx：`proxy_set_header Host $http_host`）。服务只接受本机名（127.0.0.1 / localhost / [::1]，
    任意端口）和 `config.json` 的 `allowed_hosts` 里的名字，并要求 `Origin` 和 `Host` 同源。
  - HTTPS 时带 `X-Forwarded-Proto: https`，登录 cookie 才会带 `Secure`。
  - 不缓冲响应（SSE 实时推送）；`/mounts/connect`、`/agent/connect` 是 WebSocket（转发 `Upgrade` / `Connection`）。
  - 长连接超时大于 `mount_heartbeat_s`（默认 15 秒），建议 1 小时；上传不限大小、不缓冲请求体。
- 鉴权：每个请求都要令牌（Bearer 或登录 cookie），不信任来源地址，所以本机转发不会降低安全性。

### 2.3 数据与配置

- 数据目录 `$BRIDGE_HOME`（Linux 默认 `~/.claude-agent-bridge`）：`token`（管理员令牌，第一次启动时随机生成，
  之后不变）、`env`（0600，服务和子进程的环境，例如 `CLAUDE_CODE_OAUTH_TOKEN`）、`config.json`、`guardrails.json`、
  `users.json`（用户和令牌哈希）、`accounts.json`（Claude 账号）、`bridge.db`、`uploads/`、`logs/`、`serve.pid`。
- 备份：除 `logs/`、`serve.pid` 之外整个目录都是状态。`bridge.db` 是 WAL 模式（旁边有 `-wal`、`-shm`），服务运行时
  不能直接拷，要用 `sqlite3 bridge.db ".backup …"`，或者停服务后三个文件一起拷；恢复时先删掉旧的 `-wal`、`-shm`。
- Claude 账号的登录：`~/.claude` 和旁边的 `~/.claude<N>` 等目录。备份 / 迁移时和数据目录一起处理。
- 多用户目录：`config.json` 的 `users_root`（默认 `/workspaces`）。
- 远程目录在服务器磁盘上的副本：`<mounts_root>/.cache/<用户>/<副本 id>/`（`mounts_root` 默认 `<users_root>/.remote`，
  所以默认在 `users_root` 所在的盘上）。每个共享过的文件夹最多 `mount_disk_cache_mb`（默认 2048 MB），
  `mount_disk_cache_days`（默认 7）天没再共享就删掉；所在的盘剩余不到 10% 时不再写。是可以丢的缓存：不用备份，
  删掉只是下次重新从用户电脑取。用户文件的副本，部署方要按用户数据对待（不外传）。
- 部署方可以设的环境变量：`BRIDGE_PORT` `BRIDGE_HOME` `BRIDGE_LOG_DIR` `BRIDGE_USERS_ROOT` `BRIDGE_SSHFS`
  `BRIDGE_MAX_RUNNING` `BRIDGE_MAX_LIVE` `BRIDGE_IDLE_TTL_MIN` `BRIDGE_QUESTION_TIMEOUT_S` `BRIDGE_WAIT_FOR_SUBAGENTS`
  `BRIDGE_LOG_LEVEL`。
- 部署方常用的 `config.json` 字段：`allowed_hosts`、`users_root`、`user_sandbox`、`remote_mounts`、`mounts_root`、
  `sshfs_path`、`mount_heartbeat_s`、`mount_cache_mb`、`mount_disk_cache_mb`、`mount_disk_cache_days`、`local_agent`、
  `admin_root`（管理员自己的目录，默认 `<users_root>/admin`，和用户目录平级；启动时自动建；包含用户目录的设置会让服务起不来）、
  `admin_view_elsewhere`、`admin_view_user_dirs`、`admin_view_user_sessions`（管理员在控制台里能看什么，默认都是
  false；只能在 `config.json` 里改）、`admin_sandbox`（管理员的会话也按这三项进沙箱，默认 false；打开后要 bubblewrap，
  会话连不上本机端口和 Docker、不能 sudo）。管理员在网页上给单个用户设的远程目录副本大小、天数存在 `users.json` 里，覆盖前两项默认值。没写的字段用默认值；服务自己保存配置时会写出完整的字段。
- 数据库在启动时自动升级。**同一个大版本内只加表、加列**，所以回退到同一大版本里的旧版本可以直接用现有数据；
  跨大版本回退要用升级前的备份。

### 2.4 系统软件

- 必需：`git`、`curl`。
- 推荐（缺了只是某个功能不可用）：
  - `ps`（procps）：接管终端里还开着的会话之前做检测。
  - Linux 多用户沙箱：`bubblewrap`、`socat`，并且能创建用户命名空间（Ubuntu 23.10 起要
    `kernel.apparmor_restrict_unprivileged_userns=0`）。缺了普通用户的会话起不来，管理员不受影响。
  - Linux 远程目录：`/dev/fuse`、setuid root 的 `fusermount3`（fuse3）、`sshfs` ≥ 3.7.6（修了 CVE-2026-47187；
    源码包的 sha256 见 `RELEASE.json`，和 `scripts/install.sh` 一致）。
- 确切的最低版本和校验和以每个版本的 `RELEASE.json` → `system_deps` 为准。

### 2.5 命令行

- `claude-agent health | doctor | web | ls | status | …`，用法见 `claude-agent --help`。
- 退出码：`0` 完成 · `2` 还在运行 · `3` Claude 在提问 · `4` 失败 · `5` 找不到 · `6` 连不上服务 · `7` 被拒绝 · `1` 其他。
- `claude-agent doctor` 只检查 bwrap 装没装，不检查能不能用；部署方要自己实际试一次（manager 的 `cab` 已经这样做）。

## 3. 版本号

语义化版本 `x.y.z`：`package.json` 的 `version`，git tag `v<x.y.z>`。

- **z（补丁）**：修 bug，部署方什么都不用做。
- **y（小版本）**：新功能。可以新增可选的配置项、环境变量、可选的系统软件（缺了只是新功能不可用）；数据库只加不改。
  部署方不改任何东西也能升级。
- **x（大版本）**：需要部署方配合的变化——新的必需系统软件或提高最低版本、Node 主版本要求变化、删除或改名环境变量 /
  配置项 / 命令 / HTTP 接口、端口 / 路径 / 数据目录变化、数据库不兼容（回退要用备份）、反向代理要求变化。
- 预发布 `x.y.z-rc.N`：只给测试实例用，不更新 `LATEST`。
- **发布后不可修改**：同一个版本号的发布目录不再改动；发现问题就发新版本。

## 4. 发布流程（bridge 做）

1. **平时**：每做一个有部署影响的改动，当时就写进 `CHANGELOG.md` 的 `[未发布]` → `部署影响`，不要留到发布时再回忆。
2. **定版本**：按第 3 节决定版本号；把 `## [未发布]` 改成 `## x.y.z — YYYY-MM-DD`（上面再放一个空的 `[未发布]`），
   改 `package.json` 的 `version`；`npm test` 全过；提交。
3. **发布**：`scripts/release.sh`。它会检查工作区干净、版本号没用过、变更记录里有这一版且有“部署影响”；
   跑 `npm test`；用全新编译的 `dist/` 打包；写出发布目录；打 tag `v<x.y.z>`（推送 tag 由用户决定）。
   `scripts/release.sh --dry-run` 只试做，不打 tag、不写交接目录。
4. **发布目录** `/workspaces/admin/bridge-exchange/releases/<x.y.z>/`：
   - `claude-agent-bridge-<x.y.z>.tgz`——发布包（第 2.1 节）
   - `RELEASE.json`——版本、提交、构建时间、Node 要求、Agent SDK 和 Claude Code 版本、默认端口、数据目录、系统软件要求
   - `RELEASE-NOTES.md`——变更记录里这一版的内容
   - `CONTRACT.md`——本文件此时的内容
   - `SHA256SUMS`
   正式版同时更新 `releases/LATEST`。
5. **通知**：发布目录出现就是通知；用户也会转告 manager。安全修复等在变更记录里标“紧急”。

## 5. 协作渠道：`/workspaces/admin/bridge-exchange/`

- `releases/`：只有 bridge 写。
- `requests/to-bridge/`：manager 给 bridge 的请求（缺陷、需要的配置项或接口…）。
  `requests/to-manager/`：bridge 给 manager 的（例如“下个大版本要求 Node 26”“需要新的系统软件”）。
  文件名 `NNNN-简述.md`，照 `requests/TEMPLATE.md` 写；对方在同一文件里答复并改状态：
  `open` → `accepted` / `rejected`（写理由）→ `done in x.y.z`。
- `deployments/<服务器>.md`：只有 manager 写，记录每台服务器上的版本、实例、升级记录。
  bridge 改数据格式、考虑兼容范围时参考它。
- 双方都不在这里放密钥。

## 6. 升级流程（manager 做）

1. 读新版本 `RELEASE-NOTES.md` 的“部署影响”，并和当前版本的 `CONTRACT.md` 对比；跨了几个版本就逐个看。
2. `sha256sum -c SHA256SUMS` 校验。
3. 需要时改镜像（系统软件等）；构建**带版本号的镜像**，保留上一版镜像用于回退。
4. **先升测试实例**并检查：`/health`、`claude-agent doctor`、沙箱自检、出口 IP、HTTPS、实际跑一个 Claude 回合、SSE 实时推送。
5. **再升正式实例**：先确认 `claude-agent ls --json` 里没有 `busy_reason` 或 `state: running` 的会话（或按约定时间），
   备份实例数据（第 2.3 节的数据目录和 Claude 登录目录），升级，再做第 4 步的检查。
   正在跑的会话会被打断，之后发消息即可接着做；用户共享中的远程目录会断开，页面会自动重连。
6. 更新 `deployments/<服务器>.md`。
7. 出问题：回退到上一版镜像（同一大版本内数据直接可用；跨大版本用第 5 步的备份），并在 `requests/to-bridge/` 提请求。
   不要在服务器上改 bridge 的文件来临时解决。

## 7. 变更记录（`CHANGELOG.md`）

每个版本一节，最新的在上：

```markdown
## x.y.z — YYYY-MM-DD

### 变更
- 用户能感知的变化（新功能、修复）

### 部署影响
- 无
  <!-- 或者逐条写：系统软件 / Node / 环境变量 / 配置项 / 端口与路径 / 数据库（能否回退）/ 反向代理要求 /
       部署方要做的事。大版本必须写清楚怎么迁移。紧急修复在这里写“紧急”。 -->
```

## 8. 本文件的修改

- 由 bridge 修改。改了第 2 节就是部署影响，写进变更记录，并按第 3 节定版本号。
- manager 对约定有意见或需要新的保证，在 `requests/to-bridge/` 提。

---
layout: default
title: DSH 部署与通知配置记录
---

# DSH 部署与通知配置记录

> 记录时间：2026-10-28
> 服务器：阿里云 ECS（内网 `172.21.x.x`），Ubuntu，systemd 249，内存 1.7 GB
> DSH 版本：`0.2.1-rc.2`，profile：`web`

---

## 一、最终成果

| 项目 | 状态 |
|---|---|
| DSH 常驻运行（脱离 SSH） | ✅ systemd 用户服务，崩溃自愈 + 开机自启 |
| 登录 token 固定 | ✅ `<YOUR_FIXED_TOKEN>` |
| 企微任务通知 | ✅ 已验证（`push.log` 显示 `status=200`） |

**固定访问地址（可收藏）：**

```
http://127.0.0.1:3080/?token=<YOUR_FIXED_TOKEN>
```

前提是隧道开着：

```bash
ssh -N -L 3080:127.0.0.1:3080 root@<YOUR_SERVER_IP>
```

---

## 二、整体架构

```
Mac iTerm ──SSH 隧道──► 阿里云服务器
                          └─ systemd --user (dsh.service)
                               └─ node dsh web  (127.0.0.1:3080)
                                    ├─ dsh-push ──HTTPS──► 企业微信群机器人
                                    └─ dsh-fixed-pairing-token
```

关键点：DSH 只监听 `127.0.0.1`（回环地址），**必须靠 SSH 隧道才能访问**。这是有意的安全设计——DSH 的 Web 界面能读写文件、执行命令，暴露公网等于送出 root 权限。

---

## 三、systemd 常驻配置

### 3.1 为什么需要

原本 DSH 跑在 SSH 会话下：

```
sshd → bash → npm exec dsh web → node dsh web
```

父进程是 SSH 登录的 bash，**Mac 一断线（关机/合盖/网络抖动），SSH 断开 → SIGHUP → DSH 被杀**。定时任务、长任务、通知全部中断。

改造后父进程变成 `systemd --user`，彻底脱离 SSH 会话。

### 3.2 前置：开启 linger（关键）

```bash
loginctl enable-linger root
loginctl show-user root | grep Linger    # 应显示 Linger=yes
```

**不开 linger 的话**，用户级服务会在你登出后被系统清理掉，等于白配。

### 3.3 服务文件

路径：`/root/.config/systemd/user/dsh.service`

```ini
[Unit]
Description=DeepSeek Harness Web UI
Documentation=https://github.com/deepseek-ai/deepseek-harness
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
WorkingDirectory=/root/my_space_1026
Environment=DSH_HOME=/root/.dsh
Environment=HOME=/root
# 直接用 node 跑入口，绕开 npx（npx 每次会联网检查版本，网络抖动会导致启动失败）
ExecStart=/usr/bin/node /root/.npm/_npx/1e7f6d9597241db0/node_modules/@deepseek-ai/dsh/lib/bin.js web --no-open

# 崩溃自动重拉：这是"常驻"的核心
Restart=always
RestartSec=5

# 优雅停止：等同 Ctrl-C，给 dsh 机会把会话日志写完整
KillSignal=SIGINT
TimeoutStopSec=30

StandardOutput=journal
StandardError=journal

[Install]
WantedBy=default.target
```

设计取舍：

| 参数 | 为什么这么写 |
|---|---|
| `ExecStart` 用 `node` 直接跑入口 | 不用 `npx`，避免每次启动联网检查版本；网络抖动会导致起不来 |
| `Restart=always` + `RestartSec=5` | 崩溃 5 秒后自动恢复 |
| `KillSignal=SIGINT` | 等同 Ctrl-C，让 DSH 把会话日志（zstd）写完整再退出 |
| `DSH_HOME=/root/.dsh` | 显式指定，systemd 环境变量与登录 shell 不同 |

### 3.4 启用

```bash
systemctl --user daemon-reload
systemctl --user enable dsh.service      # 开机自启
systemctl --user start dsh.service
systemctl --user status dsh.service
```

### 3.5 常用命令

```bash
systemctl --user status dsh        # 状态
systemctl --user restart dsh       # 重启
systemctl --user stop dsh          # 停止
journalctl --user -u dsh -f        # 实时日志
journalctl --user -u dsh -n 100    # 最近 100 行
```

### 3.6 ⚠️ 踩坑：重启会杀掉当前会话

**这是本次最大的坑。** 如果用 AI 助手（比如在 DSH 里让 Agent 帮你重启），会出现：

```
Agent 执行 systemctl restart dsh
  → systemd 杀掉旧 node 进程
  → Agent 自己就跑在这个进程里
  → Agent 被一起杀掉，验证代码根本执行不到
```

表现：每次重启后对话中断，验证脚本没有输出。

**正确做法**：重启操作由人手动执行，或者用 `setsid` 脱离进程树：

```bash
setsid nohup bash -c 'systemctl --user restart dsh' </dev/null >/dev/null 2>&1 &
```

但即使这样，**验证脚本也会被打断**（因为验证要等重启完成后才能跑）。所以重启后的验证最好由人在外部终端做。

---

## 四、固定登录 Token

### 4.1 为什么需要

DSH 默认每次启动生成**随机 token**：

```
dsh web: http://127.0.0.1:3080/?token=<random-base64url>...
```

token 一变，收藏的书签就失效，要重新从日志找。

### 4.2 插件

`dsh-fixed-pairing-token`（来自 GitHub：`Xinglei-dev/dsh-plugin`）

它在启动时覆盖 `connection.browserAuth.launchToken`。

### 4.3 配置位置

token **不写在插件包里**（避免包里带凭据），而是写在 profile 的 patch 层：

文件：`/root/.dsh/profiles/web/cordis.patch.yml`

```yaml
- id: fixed-pairing-token
  name: "dsh-fixed-pairing-token"
  config:
    token: "<YOUR_FIXED_TOKEN>"
```

也可以改用环境变量：`DSH_PAIRING_TOKEN`。

### 4.4 ⚠️ 踩坑：token 字符集限制

插件校验规则（源码注释）：

> Everything outside this set would not survive a round trip through `URLSearchParams` byte for byte, which would make the printed address disagree with the configured token.

即：

| 规则 | 值 |
|---|---|
| 允许字符 | `A-Z` `a-z` `0-9` `-` `_` |
| 最小长度 | 16 字符 |

**原始想用的 token 形如 `<name>@<date>`，因含 `@` 被拒。** 因为 `@` 在 URL 里会被转义成 `%40`，导致地址栏的 token 和配置的不一致。

改用 `<YOUR_FIXED_TOKEN>`（`@` → `-`）。

**该插件的安全设计**（值得称赞）：没配 token 时它**拒绝启动**，而不是悄悄退回随机 token——

```
fixed-pairing-token: TypeError: no launch token configured; 
set config.token or DSH_PAIRING_TOKEN. 
Refusing to start with the random token, because the bookmarked address 
would silently stop being fixed
```

避免"以为固定了其实没固定"的隐性故障。

> 安全提醒：token 明文出现在 URL 中。若用「姓名+日期」构成，可预测性较强。如果隧道将来对他人开放，建议加随机后缀，如 `<your-name>-<date>-<random-suffix>`。

---

## 五、企微通知（dsh-push）

### 5.1 企微 Webhook 是怎么来的

**你提供的 URL 结构：**

```
https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=<YOUR_WECOM_WEBHOOK_KEY>
└──────────┬──────────┘└────────┬────────┘      └──────────────┬──────────────┘
      腾讯官方域名         固定路径              群机器人唯一标识（密钥）
```

**生成方式**（企业微信内部群）：

```
企业微信 App / 电脑客户端
  → 「消息」标签（不是"通讯录"、不是"管理后台"）
  → 进入某个【内部群聊】
  → 右上角「···」/ 群设置
  → 「群机器人」
  → 添加机器人 → 起名 → 复制 Webhook 地址
```

**三个前提，缺一个入口就不出现：**

1. **必须在「群聊」里**——团队/组织设置、管理后台 admin 里都没有这个入口
2. **必须是「内部群」**——成员都得是企业通讯录成员。如果你是通过「邀请微信好友」拉的人，群会变成**外部群**，永远没有机器人入口
3. **群成员 ≥ 2 人**（只有自己时入口不出现）

### 5.2 ⚠️ 踩坑：个人团队找不到群机器人入口

本次卡了很久。实际排查过程：

| 尝试 | 结果 |
|---|---|
| 个人注册企业微信 | 可以注册 |
| 创建团队 | 可以 |
| 拉 1 个人进群（看起来是通讯录成员） | 仍无机器人入口 |

**结论：未做企业认证的个人团队，功能受限，群机器人入口不开放。**

> 说明：这一条是实测结论，未能从官方文档完全确证具体限制条款。如果你的团队后来认证了，可以再试一次。

### 5.3 替代方案对比（均已实测连通性）

> 那次你说「我已经有企微链接了」并把 URL 发过来，问题即解决。
> 以下为当时准备的备选，留作参考：

| 服务 | 服务器连通性 | 推到哪 | 个人可用 |
|---|---|---|---|
| **PushPlus** | ✅ 200 / 0.19s | 微信服务号 | 微信扫码即可 |
| **Server酱** | ✅ 200 / 0.29s | 微信服务号 | 微信扫码即可 |
| **Bark** | ✅ 200 / 0.11s | iOS 系统通知 | 装 App，免登录 |
| **Qmsg酱** | ✅ 200 / 0.27s | QQ 消息 | 加机器人 QQ 好友 |
| **WxPusher** | ✅ 200 / 0.17s | 微信 | 需建应用 |
| 企业微信 webhook | ✅ 200 | 企业微信 | 需内部群 |

### 5.4 插件安装

```bash
npx -y @deepseek-ai/dsh plugin --profile web add dsh-push
systemctl --user restart dsh
```

装完 `dsh-push` 会自动加进 `package.json` 的 `dsh.profile.bundles`（因为它声明了 `dsh.bundle`）。

### 5.5 配置文件

路径：`/root/.dsh/dsh-push/config.json`（权限 `600`，含密钥）

```json
{
  "minDurationSec": 30,
  "timeoutMs": 10000,
  "shutdownTimeoutMs": 3000,
  "dedupeMs": 5000,
  "includeSubagents": false,
  "needInputTools": ["ask_user_question", "exit_plan_mode"],
  "events": {
    "idle": true,
    "interrupted": true,
    "needInput": true,
    "exit": false
  },
  "titleTemplate": "DSH {{status}}",
  "template": "{{status}} · {{reason}}\n项目：{{project}}\n耗时：{{duration}}\n时间：{{time}}",
  "channels": [
    {
      "type": "wecom",
      "name": "企业微信",
      "url": "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=<YOUR_WECOM_WEBHOOK_KEY>",
      "timeoutMs": 10000
    }
  ],
  "debug": false
}
```

**支持的渠道类型**：`bark` / `feishu` / `wecom` / `dingtalk` / `ntfy` / `webhook`（可配多个，并行推送）

**可用模板变量**：`{{event}}` `{{status}}` `{{reason}}` `{{cwd}}` `{{project}}` `{{duration}}` `{{session}}` `{{model}}` `{{host}}` `{{time}}` `{{date}}` `{{title}}` `{{text}}`

**字段说明：**

| 字段 | 作用 |
|---|---|
| `minDurationSec` | 只通知超过 N 秒的任务。设 0 = 每轮都通知 |
| `events.idle` | 任务完成 |
| `events.interrupted` | 出错/被取消 |
| `events.needInput` | **DSH 等你回答时**（最实用） |
| `dedupeMs` | 去重窗口 |
| `includeSubagents` | 是否包含子智能体（默认排除，避免噪音） |

改配置**不用重启**——插件按文件 mtime 校验缓存，会自动重载。

### 5.6 ⚠️ 踩坑：dsh-push 0.1.1 加载失败（最难的坑）

**现象**：插件装了、配置也对，但 `push.log` 永远不生成，企微收不到任何自动通知。

**日志里（开启 journald 持久化后才看到）**：

```
dsh-push (dsh-push): Error: cannot get property "agents" without inject
    at new apply (.../dsh-push/lib/index.js:1207:10)
dsh: warning: 1 entry did not activate
```

**根因有三层，逐层排查：**

**第一层：插件缺 `inject` 声明**

插件代码直接访问 `ctx.agents`：

```js
function apply(ctx) {
  if (ctx.agents === void 0) throw new Error("dsh-push requires the dsh-agent event graph");
```

但 Cordis 框架里，**访问未声明的服务会直接抛错**。插件必须导出 `inject` 声明依赖。

**第二层：`inject` 写成了对象**

错误写法：
```js
export const inject = { required: ["agents"], optional: ["commands"] };
```

Cordis 的格式是**普通数组**（参考 `dsh-time-context`）：
```js
const inject = ["agents", "sessionProjections"];
```

**第三层（真正的根因）：`default` 导出覆盖了 `inject`**

Cordis 的 `unwrapExports` 实现：

```js
unwrapExports(exports) {
  if (isNullable(exports)) return exports;
  exports = exports.default ?? exports;   // ← 优先取 default！
  if (!exports.__esModule) return exports;
  return exports.default ?? exports;
}
```

插件原本导出 `export { apply, apply as default }`。因为**同时提供了 `default`**，`unwrapExports` 拿到的就是那个纯函数 `apply`，模块上的**具名 `inject` 被整个丢弃**。

对比能正常工作的 `dsh-time-context`：

```js
export { Config, apply, inject, name };   // ← 没有 default
```

**最终修复**（改 `/root/.dsh/profiles/web/node_modules/dsh-push/lib/index.js` 末尾）：

```js
// 修复前
export { apply, apply as default };

// 修复后
export const inject = ["agents", "commands"];
export { apply };
```

即：**补上 `inject` 具名导出，同时删掉 `default` 导出。**

**验证修复生效：**

```bash
journalctl _SYSTEMD_USER_UNIT=dsh.service --since "1 minute ago" | grep dsh-push
# 无输出 = 加载成功
```

修复后日志干净，`push.log` 出现：

```
2026-10-28T02:57:40.314Z [idle] 企业微信 ok status=200 (159ms)
2026-10-28T02:58:13.682Z [idle] 企业微信 ok status=200 (159ms)
```

> ⚠️ **注意**：这是直接改了 `node_modules` 里的文件。以后重装或升级 `dsh-push` 会被覆盖，需要重新打补丁。更稳妥的做法是给作者提 issue，或 fork 后发布修复版。

### 5.7 ⚠️ 踩坑：journald 没有持久化日志

排查插件问题时发现：

```bash
journalctl --user -u dsh
# No journal files were found.
```

系统默认没有 `/var/log/journal` 目录，日志只在内存里，**服务重启后旧日志全部丢失**，根本无法排错。

**修复：**

```bash
mkdir -p /var/log/journal
systemd-tmpfiles --create --prefix /var/log/journal
systemctl restart systemd-journald
```

然后改用这个命令查用户服务日志（`--user` 有时查不到）：

```bash
journalctl _SYSTEMD_USER_UNIT=dsh.service -n 50 --no-pager
```

---

## 六、验证方法

### 6.1 确认 DSH 已脱离 SSH

```bash
pid=$(pgrep -f "bin.js web" | head -1)
ps -o pid=,ppid=,cmd= -p $pid
# 父进程（PPID）应该是 systemd --user，而不是 sshd/bash
```

### 6.2 确认崩溃自愈

```bash
systemctl --user show dsh.service -p NRestarts --value
# 数字 > 0 说明触发过自动重启
```

### 6.3 确认通知链路

```bash
tail -5 /root/.dsh/dsh-push/push.log
# 出现 "[idle] 企业微信 ok status=200" 即成功
```

### 6.4 手动测企微通道

```bash
curl -s -X POST "https://qyapi.weixin.qq.com/cgi-bin/webhook/send?key=<YOUR_WECOM_WEBHOOK_KEY>" \
  -H "Content-Type: application/json" \
  -d '{"msgtype":"text","text":{"content":"测试消息"}}'
# 期望返回 {"errcode":0,"errmsg":"ok"}
```

---

## 七、能力的边界（重要）

### ✅ 现在能做到

- DSH 常驻运行，Mac 关机/断网不影响
- 服务器重启后自动恢复
- 任务完成 / 出错 / 需要你回答时，企业微信收到通知

### ❌ 现在做不到

| 需求 | 原因 |
|---|---|
| 在企微里回复消息给 DSH | 企微群机器人 webhook 是**单向推送钩子**，没有回传通道 |
| 在企微里 @ 它下达任务 | 同上，需要双向通道 |
| 从任何设备访问 | DSH 只绑 `127.0.0.1`，每台设备都要自己开 SSH 隧道 |

### 如果想要双向对话

需要「聊天通道」类插件，而不是推送类：

| 方案 | 能力 | 门槛 |
|---|---|---|
| `@tencent-connect/dsh-qqbot` | 双向、群里 @ 下任务 | 腾讯官方包，手机 QQ 扫码即可，**无需企业认证** |
| `dsh-bot-gateway` | QQ/Telegram/飞书/钉钉 | 需配置，QQ 部分需 NapCat |
| `dsh-qq-onebot` / `dsh-onebot-qq` | QQ 群聊 | 需 NapCat（个人号协议，有封号风险） |

> 已核实 `@tencent-connect/dsh-qqbot@0.5.3`：腾讯官方维护（仓库 `github.com/tencent-connect/dsh-qqbot`），peer 依赖与本机 `0.2.1-rc.2` 全部匹配，机制是手机 QQ 扫码绑定，**绕过 QQ 开放平台的实名/企业认证**。这是目前想上 QQ 通道的最优解。

---

## 八、其他踩过的坑（备查）

### 8.1 第三方包 `dsh-plugin-market` 打包缺陷

曾有需求装插件市场，误装了**第三方同名包** `dsh-plugin-market@1.3.0`（非官方 `@deepseek-ai/dsh-plugin-market`），报错：

```
"@deepseek-ai/cordis@workspace:^" is in the dependencies 
but no package named "@deepseek-ai/cordis" is present in the workspace
```

**原因**：作者从自己的 monorepo 直接 publish，没有把 pnpm 的 `workspace:^` / `link:..\..\vendor\...` 协议替换成真实版本号（路径里还带 Windows 反斜杠）。

**结论**：这是那个包的发布缺陷，装多少遍都会失败。官方包名是 `@deepseek-ai/dsh-plugin-market`。而且 DSH 已内置插件市场（`dsh-experimental-plugin-market-bundle`），**根本不需要额外装**。

### 8.2 pnpm 未安装

插件市场报错：

```
Command failed with ENOENT: pnpm view dsh-market ...
spawn pnpm ENOENT
```

**原因**：服务器没装 pnpm。DSH 的插件管理直接 `spawn` pnpm。

```bash
npm install -g pnpm --registry=https://registry.npmmirror.com/
pnpm -v
```

### 8.3 `dsh-pet` 桌宠在你的架构下上不了桌面

`dsh-pet` 声明 `dsh.client.platform: "web"`，且桌面模式依赖**宿主进程**去下载 Electron 并拉起窗口。你的 DSH 宿主在**无图形界面的云服务器**上，Electron 下了也开不了窗。

想在 Mac 桌面上养宠物，需要用专门做「客户端本地运行」的包（如 `dsh-pet-live2d-desktop-darwin-arm64`），且隧道必须常开。

### 8.4 DSH 没有「删除会话」功能

只有归档，没有删除。官方明确：

- `dsh-client-ui-workspace`：「No Session deletion」
- `dsh-session-persistence-jsonl`：「Nothing deletes session files … the seam has no deletion API」

物理删除只能从外部操作：

| 内容 | 路径 |
|---|---|
| 会话日志 | `~/.dsh/sessions/--<cwd>--/<session-id>/session.vN.jsonl.zstd` |
| 投影缓存 | `~/.dsh/storages/session_projcache/sessions/<id>.json` |
| 工作区注册表 | `~/.dsh/storages/workspace.json` |

⚠️ 删除前**必须先停 DSH**：它持有会话写锁（`session.lock` 上的 flock），且 `workspace.json` 是**整份内存状态覆盖写**，不监听文件变化。运行中改文件，DSH 下次写入时会把旧状态写回去，"复活"已删的工作区。

工作区里保留了脚本 `dsh-session-prune.py`，支持按树结构浏览和选择性删除。

---

## 九、关键文件清单

| 文件 | 说明 |
|---|---|
| `/root/.config/systemd/user/dsh.service` | systemd 服务定义 |
| `/root/.dsh/dsh-push/config.json` | 通知配置（权限 600） |
| `/root/.dsh/dsh-push/push.log` | 推送日志 |
| `/root/.dsh/profiles/web/cordis.patch.yml` | profile 补丁层（含固定 token） |
| `/root/.dsh/profiles/web/package.json` | profile 的 bundles 与依赖 |
| `/root/.dsh/profiles/web/node_modules/dsh-push/lib/index.js` | **已打补丁**（inject + 去掉 default） |
| `/root/my_space_1026/dsh-session-prune.py` | 会话清理脚本 |

---

## 十、备忘

- **DSH 入口路径写死在 systemd 里**：`/root/.npm/_npx/1e7f6d9597241db0/...`。这个哈希目录在 npm 缓存清理或升级后可能变化，届时服务会起不来。想更稳妥可以 `npm i -g @deepseek-ai/dsh` 后把 `ExecStart` 改成 `/usr/bin/dsh web --no-open`。
- **服务器内存偏紧**：1.7 GB 总量，DSH 约占 400 MB，可用约 840 MB。想跑 NapCat（Linux QQ）会不够用。
- **每个设备需各自建隧道**：`ssh -N -L 3080:127.0.0.1:3080 root@<YOUR_SERVER_IP>`。可写进 `~/.ssh/config` 简化：
  ```
  Host dsh
      HostName <YOUR_SERVER_IP>
      User root
      LocalForward 3080 127.0.0.1:3080
      ServerAliveInterval 60
  ```
  以后只需 `ssh -N dsh`。

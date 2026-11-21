# instabox-keepalive

让 InstaBox 免费机**不被自动休眠**、也**不撞 2 小时强制挂起**的 Cloudflare Worker。

不用自己的机器常开，一个 Worker 能带多个账号，免费额度用不到 3%。

---

## 一、它对付的是哪两条限制

InstaBox 免费机有两条硬规矩，续租只能解决第一条：

| 平台限制 | 后果 | 这个 Worker 怎么做 | 要配吗 |
| --- | --- | --- | --- |
| 空闲 30 分钟 | 自动休眠 | cron 每 5 分钟续一次活动租约（TTL 900 秒） | 不用，部署即生效 |
| 连续跑满 **2 小时（120 分钟）** | 强制挂起 + 冷却**约 14 分钟** | 剩 20 分钟时自己 `suspend → resume` 一次，把计时清零 | 不用，默认开 |

第二条是重点，也是最容易搞错的地方（很多地方写 8 小时，是错的）：

- **上限是 2 小时**。实测：`runStartedAt` 之后正好 **120 分 00 秒**被挂，冷却 **14 分 11 秒**，期间续租和 resume 全部返回 `409 machine_cooldown`。
- **保活靠的是"提前量"兜底，不是那几个整点**。默认 `INSTABOX_CYCLE_MARGIN_MIN=20`，即每 **100 分钟**必轮一次（实测间隔稳定在 5995–6001 秒）。
- `INSTABOX_CYCLE_AT` 那四个点只是把轮转时刻**锚定**在固定整点，让你能预知什么时候会断 2–3 秒。删了机器也不会死，只是断线时刻开始漂移。

> ⚠️ `INSTABOX_CYCLE=0` **不是**"只保活不轮转"，而是**关掉整个轮转**（定时点和提前量都不触发）。
> 机器会一路跑到 120 分钟被平台挂起，冷却约 14 分钟才恢复，前后停机十几分钟。想减少轮转次数请调 `INSTABOX_CYCLE_MARGIN_MIN`，别关 `INSTABOX_CYCLE`。

## 二、部署（网页面板，5 分钟）

不用装 wrangler。

1. **建 Worker**：面板 → `Workers & Pages` → `Create` → `Worker` → 起名 → `Deploy`
2. **贴代码**：`Edit code` → 把 `worker/worker.js` 全文覆盖进去 → `Deploy`
3. **加 key**：`Settings` → `Variables and Secrets`，加一条 **Secret**
   `INSTABOX_API_KEY` = 你的 `ibk_...`
4. **加定时**：`Settings` → `Triggers` → `Cron Triggers`，填 `*/5 * * * *`
   > `*/5` 是按钟表对齐的（`:00/:05/:10…`），不是"上次跑完 +5 分钟"。
5. **（建议）加口令**：再存一条 Secret `TOKEN` = 随机串（`openssl rand -hex 16`）。不加接口对所有人开放。
6. **（可选）绑 KV 看日志**：`Storage & Databases` → `KV` → 建 namespace，回 Worker 的 `Bindings` 加绑定，**变量名必须叫 `LOG`**。不绑也能跑，只是 `/log` 是空的。

## 三、验证

```sh
curl "https://<你的worker>.workers.dev/status?token=<你的TOKEN>"
```

看到这三样就对了：

- `"state": "running"`
- `"remainingMinutes": 120` —— 离 2 小时上限还剩几分钟，每轮 cron 掉 5 左右
- `"nextCycleAt"` 有值 —— 轮转已经排上了

想看它到底干过什么：

```sh
curl "https://<你的worker>.workers.dev/log?token=<你的TOKEN>"
```

## 四、一次轮转发生了什么

以 `19:00` 那次为例（抖动默认全关）：

```text
19:00:0x  cron 触发 → 判定：到点了 + 机器在跑 + 这个点之后还没重启过
19:00:0x  POST /suspend
19:00:1x  确认 suspended
19:00:1x  POST /resume（失败自动用 /start 兜底）
19:00:2x  确认 running
19:00:2x  PUT 活动租约（生命周期操作会让旧租约失效，必须补一次）
```

几个要点：

- **停多久**：每台 10–30 秒。默认 `CYCLE_HOLD_SEC=0`，就是"确认睡下立刻叫醒"。进程通常保留（`runtime.processesPreserved`）
- **多台错峰**：一次 cron 只轮转一台（一轮约 20 个子请求，免费版单次上限 50）。3 台就是 19:00 / 19:05 / 19:10
- **兜底**：某个点没赶上（正在部署、Worker 抖动），剩余时间掉到 20 分钟时会立刻补一次
- **手动触发**：`GET /?token=xxx` 走同一套判断，想立刻轮不用等 cron
- **同一个点只转一次**：靠 `runStartedAt` 和点比较，不需要额外存状态

## 五、想改才看

### 只续租、不轮转

```
INSTABOX_CYCLE = 0
```

**这是关掉全部轮转的开关**（定时点和提前量都不触发）：机器会一路跑到 120 分钟 → 平台挂起 → 冷却约 14 分钟 → 自动恢复。前后停机十几分钟，只有"宁可停机也不许机器重启"的场景才这么用。

### 换轮转时间点

```
INSTABOX_CYCLE_AT = "01:30,08:00,13:00,19:00"   # 本地时间，逗号分隔
INSTABOX_CYCLE_TZ = "Asia/Shanghai"
```

**前提变了**：上限只有 2 小时，所以任何"一天只放几个点"的方案都盖不住整个周期 ——
保活永远靠 `MARGIN` 兜底（每 100 分钟一轮），这些点只决定"额外的轮转落在哪个整点"。

| 配置 | 实际效果 |
| --- | --- |
| `01:30,08:00,13:00,19:00`（默认） | 每天 4 次锚定 + 兜底，一天约 14 次轮转，断线时刻可预期 |
| `04:00`（只留一个） | 只在凌晨锚定一次，其余全交给兜底，断线时刻基本漂移 |
| 留空 `""` | 回落到默认四个点（空串不会真的"不定时"） |

想彻底减少轮转次数，改的是 `INSTABOX_CYCLE_MARGIN_MIN`（提前量越小轮得越少、但越贴近上限），
不是删时间点。

懒得算就只报高峰时段，让它自己排：

```
INSTABOX_PEAK_HOURS = "10-12,14-18,20-23"
```

会排成 `00:00,06:00,12:00,18:00`（全在高峰外、间隔 6 小时）。`/accounts` 里能看到排出来的结果和来源（`cycleAtSource: "auto"`）。

### 多账号

**第 N 个账号 = 变量名加 `_N` 后缀**，加账号就是多存一个 Secret：

| 1 号 | 2 号 |
| --- | --- |
| `INSTABOX_API_KEY` | `INSTABOX_API_KEY_2` |
| `INSTABOX_MACHINE` | `INSTABOX_MACHINE_2` |
| `INSTABOX_CYCLE_AT` | `INSTABOX_CYCLE_AT_2` |

规则：

- 带后缀的优先，没写的继承全局值
- **`INSTABOX_MACHINE` 也会继承** —— 两个账号机器名不同时，记得单独写 `_2`
- 编号可以不连续（只配 `_2` 也能跑），`_1` 等同不带后缀
- 同一把 key 配两次只续一次；某个账号 key 失效不影响其它

三把 key、其余全默认，面板上就这几条：

| 变量 | 类型 |
| --- | --- |
| `INSTABOX_API_KEY` / `_2` / `_3` | Secret |
| `TOKEN` | Secret |

`/accounts?token=...` 看到 `#1` `#2` `#3` 三行（key 打码）就说明都认到了。

### 安全

| 变量 | 默认 | 作用 |
| --- | --- | --- |
| `TOKEN` | 空 | 配了之后所有请求带 `?token=`，错了返回 `401` |
| `INSTABOX_ANON` | `open` | **只在没配 `TOKEN` 时生效**：`open` 全放行（旧版行为）、`readonly` 只放 `/status` `/accounts` `/log`、`closed` 全 `403` |

不配 `TOKEN` 也能跑（和旧版一致），但任何人知道域名就能触发续租/释放、刷光 KV 日志额度。建议配上。

## 六、接口

```text
GET /?token=xxx            所有账号各续租一次（cron 跑的就是这个逻辑）
GET /status?token=xxx      机器状态
GET /accounts?token=xxx    配了哪些账号（key 打码）
GET /log?token=xxx         最近 48 条日志
GET /release?token=xxx     放掉租约
```

都支持 `&account=2` 只看 2 号账号、`&format=text` 输出纯文本。

`/status` 每台机器带这些字段：

| 字段 | 含义 |
| --- | --- |
| `state` | `running` / `suspended` / `stopped` |
| `runStartedAt` | 本轮连续运行的起点。**轮转成功会跳变，这是验证轮转是否真生效的关键** |
| `remainingMinutes` | 离 2 小时上限还剩几分钟（上限 120） |
| `cooldownUntil` | 冷却到什么时候（没冷却为 `null`） |
| `nextCycleAt` | 下次轮转时间 |

`/` 的结果里，被轮转的机器带 `"cycled": true`、`"trigger": "slot"`（定时）或 `"limit"`（兜底）。

## 七、完整变量表

日常不用看，要微调时查。

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `INSTABOX_API_KEY` | — | **必填**，`ibk_...`，存 Secret |
| `INSTABOX_MACHINE` | 空 | 空 = 该 key 名下全部机器；填名字/逗号分隔/id 可限定 |
| `INSTABOX_LEASE` | `keepalive` | 租约名，换名字等于换租约 |
| `INSTABOX_LEASE_TTL` | `900` | 租约有效期（秒）。**必须大于 cron 间隔**，`*/5` 配 900 有 3 倍余量 |
| `INSTABOX_AUTO_RESUME` | `1` | 收到 409 自动 `resume`（stopped 用 `start`）再续租；设 0 只报错 |
| `INSTABOX_LOG_MODE` | `hourly` | `hourly`（每小时最多 1 条）/ `every` / `off` |
| `INSTABOX_API_URL` | `https://api.instabox.sh` | 换私有网关才需要改 |
| `INSTABOX_CYCLE` | `1` | 轮转开关，**默认开** |
| `INSTABOX_CYCLE_AT` | `01:30,08:00,13:00,19:00` | 固定时间点，`CYCLE=0` 时不生效 |
| `INSTABOX_PEAK_HOURS` | 空 | 自动排时间表，`CYCLE_AT` 优先于它 |
| `INSTABOX_CYCLE_TZ` | `Asia/Shanghai` | 时区 |
| `INSTABOX_CYCLE_WINDOW_MIN` | `30` | 到点后多少分钟内允许触发 |
| `INSTABOX_CYCLE_MARGIN_MIN` | `20` | 兜底：剩余多少分钟强制轮转；0 = 关 |
| `INSTABOX_CYCLE_JITTER_MIN` | `0-0` | 开始时间抖动，如 `0-15` 表示到点后晚 0–15 分钟 |
| `INSTABOX_CYCLE_HOLD_SEC` | `0-0` | 停机时长，如 `600-900`。**超过 600 秒必须绑 KV** |
| `INSTABOX_WAKE_JITTER_MIN` | `0-0` | 冷却结束后再随机等几分钟 |

## 八、常见问题

**为什么不能直接把自动休眠关掉？**

2026-10-18 实测：账号级 `fixedAutoSuspendSeconds: 1800`、`manualLifecycle: false`，服务端把空闲阈值锁死。对 `PATCH /v1/machines/{id}` 传 `autoSuspendAfterSeconds: 0`（含官方 CLI 和 SDK）一律返回 200，但值不变 —— 静默忽略。所以只能走租约这条官方口子。

**为什么不在机器里跑定时任务？**

机器一 suspend，里面的 cron / systemd timer 全冻住，叫不醒自己。续租必须来自**外部**，这就是放在 Cloudflare Worker 上的原因。

**免费额度够吗？**

- **子请求**：每次调用上限 50 个。一个账号约「1 次列机器 + 每台 1 次续租」，一次轮转约 20 个。3 账号各 1 台用不到 10 个
- **KV 写**：每天 1000 次。默认 `hourly` 每天约 24 次，占 2.4%；多账号也是一条日志装全部
- cron 别改成 `* * * * *`（1440 次/天），会把 KV 写额度刷光

**机器上的任务必须连续跑超过 2 小时？**

免费版做不到 —— 120 分钟是硬上限。要么拆任务（每 100 分钟断 2–3 秒，任务能续跑就不怕），
要么升级付费版（`maxRunMinutes = -1`，不限）。
`INSTABOX_CYCLE=0` 不是解法，那只会让机器每 2 小时被平台挂 14 分钟。

**机器变多了要改什么？**

不用改 —— 提前量会自己加宽。因为每轮 cron 只轮转一台，N 台同时进窗口时排在后面的要等：

| 账号/机器数 | 实际提前量 | 最后一台轮完时离上限还剩 |
| --- | --- | --- |
| 1–3 台 | 20 分钟（不变） | 10 分钟以上，安全 |
| 4 台 | 自动升到 25 | 5 分钟，够 |
| 5 台 | 自动升到 30 | 5 分钟，够 |

公式是 `max(你配的 MARGIN, 10 + (N-1)×5)`。所以加到 4、5 台都不用动配置，
但**别手动把 `INSTABOX_CYCLE_MARGIN_MIN` 调小**，那会抵消这个自动加宽。

**改了 cron 间隔要注意什么？**

`LEASE_TTL` 必须跟着调大，否则两次续租之间会空窗。`*/5` 配 900 有余量；拉到 15 分钟以上就要同步改 TTL。

## 九、CLI 部署（可选）

```sh
cd worker
npx wrangler secret put INSTABOX_API_KEY
npx wrangler secret put TOKEN
npx wrangler deploy
```

`wrangler.toml` 里已设好 `crons = ["*/5 * * * *"]` 和 `LEASE_TTL = "900"`，轮转相关变量默认注释（不写就走默认行为）。

## 免责声明

仅用于管理你自己账号下的 InstaBox 机器。Worker 能读到这些 key，能读到它们的人也能操作你的机器。

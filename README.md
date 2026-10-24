# instabox-keepalive

InstaBox 的免费机器有两条限制：

1. **空闲 30 分钟自动休眠** —— 靠官方 `activity lease` 续租就能一直压住
2. **连续跑满 8 小时强制挂起，再冷却 10 分钟** —— 续租压不住，只能到点前自己重启一次

这个仓库是一个 **Cloudflare Worker**：cron 每 5 分钟续一次租（解决问题 1），需要时可以按固定时间点主动 `suspend → resume` 把连续运行计时清零（解决问题 2）。

不需要你自己的机器常开，一个 Worker 能同时带多个账号，免费额度用不到 3%。

| 问题 | Worker 怎么处理 | 要配置吗 |
| --- | --- | --- |
| 空闲 30 分钟被休眠 | 每 5 分钟续一次租（TTL 900 秒） | 不用，部署完就生效 |
| 机器被挂起/停掉 | 续租收到 `409 machine_not_running` 时自动 `resume` | 不用，默认开 |
| 跑满 8 小时被强制挂起 + 冷却 | 到点前自己 `suspend → resume`，计时清零 | 要，默认关，设 `INSTABOX_CYCLE=1` 才开 |

代码只有两个文件：`worker/worker.js`（全部逻辑）、`worker/wrangler.toml`（CLI 部署用）。

---

## 一、部署（网页面板，5 分钟）

不用装 wrangler。

1. **建 Worker**：面板 → `Workers & Pages` → `Create` → `Worker` → 起名（如 `instabox-keepalive`）→ `Deploy`
2. **贴代码**：`Edit code` → 把 `worker/worker.js` 全文覆盖进去 → `Deploy`
3. **加变量**：`Settings` → `Variables and Secrets`，只加这两条就能跑：

   | 变量 | 类型 | 值 |
   | --- | --- | --- |
   | `INSTABOX_API_KEY` | **Secret（加密）** | 你的 `ibk_...`（多账号见第四节） |
   | `TOKEN` | **Secret** | 自己起一串随机值，例如 `openssl rand -hex 16` 的输出 |

4. **加定时**：`Settings` → `Triggers` → `Cron Triggers` → `Add Cron Trigger`，填 `*/5 * * * *` → 保存

   > 这就是保活开关。`*/5` 表示每小时 `:00/:05/:10…` 各跑一次，是**按钟表对齐**的，不是"上次跑完 +5 分钟"。

5. **（可选）绑 KV 看日志**：`Storage & Databases` → `KV` → `Create namespace`，再回到 Worker 的 `Settings` → `Bindings` 加一个 KV 绑定，**变量名必须填 `LOG`**。不绑也能用，只是 `/log` 是空的、长停机抖动降级为 600 秒。

## 二、验证

```sh
curl "https://<你的worker>.workers.dev/status?token=<你的TOKEN>"
```

看到 `"state": "running"`、`"remainingMinutes": 480` 就对了。`remainingMinutes` 是离 8 小时上限还剩几分钟，每轮 cron 会掉 5 左右。

`TOKEN` 不填也能跑，但任何知道你 Worker 域名的人都能触发续租/释放，**强烈建议填**。

## 三、配置项

只有第一组是必填的，其余留空就是默认行为。**所有变量都支持 `_N` 后缀**做多账号（见第四节）。

### 必填

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `INSTABOX_API_KEY` | — | `ibk_...`，存 Secret |
| `TOKEN` | — | 访问口令，存 Secret，所有请求带 `?token=` |

### 保活相关（一般不用改）

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `INSTABOX_MACHINE` | 空 | 空 = 该 key 名下**全部机器**；填 `qqq` 只保一台，多台用逗号分隔，也可填机器 id |
| `INSTABOX_LEASE` | `keepalive` | 租约名，同一台机器换名字等于换租约 |
| `INSTABOX_LEASE_TTL` | `900` | 租约有效期（秒）。必须大于 cron 间隔：`*/5` 配 `300` 正好卡边界，`900` 有 3 倍余量 |
| `INSTABOX_AUTO_RESUME` | `1` | 收到 409 时自动 `resume`（stopped 用 `start` 兜底）再续租；设 `0` 只报错不唤醒 |
| `INSTABOX_LOG_MODE` | `hourly` | 写 KV 日志的频次：`hourly`（每小时最多 1 条）/ `every`（每次）/ `off` |
| `INSTABOX_API_URL` | `https://api.instabox.sh` | 只有换私有网关时才需要改 |

### 定时检修相关（可选，默认全关）

| 变量 | 默认 | 说明 |
| --- | --- | --- |
| `INSTABOX_CYCLE` | 空（关） | **默认关**。只有显式设 `1`（或 `on` / `true` / `yes`）才启用轮转；不设就只做「续租 + 409 自动唤醒」 |
| `INSTABOX_CYCLE_AT` | `01:30,08:00,13:00,19:00` | 固定时间点（本地时间）。**只在 `INSTABOX_CYCLE=1` 时生效**；开了又不填就用这套默认值，填了按你写的走 |
| `INSTABOX_PEAK_HOURS` | 空 | 只给高峰时段（如 `10-12,14-18,20-23`），自动排重启点。`CYCLE_AT` 优先于它 |
| `INSTABOX_CYCLE_TZ` | `Asia/Shanghai` | 时间点按哪个时区算 |
| `INSTABOX_CYCLE_WINDOW_MIN` | `30` | 到点后多少分钟内允许触发（`*/5` → 6 次机会） |
| `INSTABOX_CYCLE_MARGIN_MIN` | `20` | 兜底：剩余多少分钟时强制轮转一次。设 `0` 关掉兜底 |
| `INSTABOX_CYCLE_JITTER_MIN` | `0-0` | 开始时间抖动，如 `0-15` 表示到点后随机晚 0–15 分钟 |
| `INSTABOX_CYCLE_HOLD_SEC` | `0-0` | 停机时长，如 `600-900` 表示停 10–15 分钟再唤醒 |
| `INSTABOX_WAKE_JITTER_MIN` | `0-0` | 平台冷却结束后再随机等几分钟唤醒 |

## 四、多账号

**第 N 个账号 = 单账号变量名加 `_N` 后缀**，加一个账号就是多存一个 Secret：

| 1 号账号 | 2 号账号 | 说明 |
| --- | --- | --- |
| `INSTABOX_API_KEY` | `INSTABOX_API_KEY_2` | 必填，Secret |
| `INSTABOX_MACHINE` | `INSTABOX_MACHINE_2` | 可选，留空 = 该账号全部机器 |
| `INSTABOX_LEASE` | `INSTABOX_LEASE_2` | 可选 |
| `INSTABOX_LEASE_TTL` | `INSTABOX_LEASE_TTL_2` | 可选 |
| `INSTABOX_AUTO_RESUME` | `INSTABOX_AUTO_RESUME_2` | 可选 |
| `INSTABOX_CYCLE` 及后续所有 `INSTABOX_CYCLE_*` / `INSTABOX_PEAK_HOURS` / `INSTABOX_WAKE_JITTER_MIN` | 同左加 `_2` | 可选 |

规则：

- 带后缀的优先；没写的自动继承不带后缀的全局值
- `INSTABOX_MACHINE` 也会继承 —— 两个账号机器名不同时，记得单独写 `INSTABOX_MACHINE_2`
- 编号可以不连续（只配 `_2` 也能跑），`INSTABOX_API_KEY_1` 等同 `INSTABOX_API_KEY`
- 同一把 key 配两次只续一次；某个账号 key 失效不影响其它账号
- 每个账号的机器列表分开缓存、分开续租

**三把 key、其余全用默认值**，面板上最终就这几条：

| 变量 | 类型 | 值 |
| --- | --- | --- |
| `INSTABOX_API_KEY` | Secret | `ibk_aaa…` |
| `INSTABOX_API_KEY_2` | Secret | `ibk_bbb…` |
| `INSTABOX_API_KEY_3` | Secret | `ibk_ccc…` |
| `TOKEN` | Secret | 自己起的随机串 |
| `INSTABOX_LEASE_TTL` | Text | `900`（不写也是 900） |
| `INSTABOX_LOG_MODE` | Text | `hourly`（不写也是 hourly） |

部署完打开 `/accounts?token=...`，看到 `#1` `#2` `#3` 三行（key 打码）就说明都认到了。

## 五、8 小时上限与定时检修（可选）

免费机 `maxRunMinutes = 480`（8 小时，账号级固定）。跑满后平台强制挂起，并进入冷却（`cooldownUntil`，默认 10 分钟）——**冷却期内 `resume` / `start` / `wake` 全部返回 `409 machine_cooldown`**，只能等。

### 三种策略

| 策略 | 怎么配 | 结果 |
| --- | --- | --- |
| **定时检修** | `INSTABOX_CYCLE=1`（可再写 `INSTABOX_CYCLE_AT=...`） | 在时间点停 10–30 秒，永远不进冷却 |
| **不干预**（默认） | 什么都不加 | 每 8 小时停 10 分钟 + 最多一个 cron 间隔 |
| **不干预但保留配置** | 不设 `INSTABOX_CYCLE` + 保留 `CYCLE_AT` | 同上，配置留着方便以后再开 |

> **升级注意**：现在轮转默认**关**。要定时检修必须显式加 `INSTABOX_CYCLE=1`；只写 `INSTABOX_CYCLE_AT` 不会生效（`/status` 里会给出 `cycleWarning` 提示）。

### 开法 A：固定时间点

先设 `INSTABOX_CYCLE=1` 打开轮转。此时 `INSTABOX_CYCLE_AT` 不写就用默认值 `01:30,08:00,13:00,19:00`（间隔 6.5/5/6/6.5 小时，都小于 8 小时上限）。想换成别的时间点：

```toml
INSTABOX_CYCLE_AT = "01:30,08:00,13:00,19:00"   # 本地时间，逗号分隔
INSTABOX_CYCLE_TZ = "Asia/Shanghai"
INSTABOX_CYCLE_WINDOW_MIN = "30"                # 到点后 30 分钟内触发
```

**两个时间点之间必须小于 8 小时**，否则中间还是会撞上上限：

| 时间点 | 最大间隔 | 余量 | 说明 |
| --- | --- | --- | --- |
| `01:30,08:00,13:00,19:00` | 6.5 小时 | 1.5 小时 | **推荐**，避开高峰 10-12 / 14-18 / 20-23 |
| `02:00,08:00,14:00,20:00` | 6 小时 | 2 小时 | 间隔最均匀 |
| `00:00,06:00,12:00,18:00` | 6 小时 | 2 小时 | 整点党 |
| `03:00,10:00,17:00,00:00` | 7 小时 | 1 小时 | 三次一天，间隔不均 |
| `04:00`（一天一次） | 24 小时 | ❌ | 不行，8 小时就被挂了 |

同一个时间点只会转一次：判断靠 `runStartedAt` 和点比较，不需要额外存储。

### 开法 B：只给高峰时段，自动排

不想自己算时间点，就把高峰告诉它：

```toml
INSTABOX_PEAK_HOURS = "10-12,14-18,20-23"
INSTABOX_CYCLE_TZ = "Asia/Shanghai"
# INSTABOX_CYCLE_AT 留空
```

会自动排成 `00:00, 06:00, 12:00, 18:00`（全在高峰外、间隔 6 小时）。支持 `10-12`、`10:30-12:00`、跨零点 `23-02`。

`/accounts` 能看到排出来的结果和来源：

```json
{ "cycleAt": "00:00,06:00,12:00,18:00", "cycleAtSource": "auto", "nextCycleAt": "2026-10-23T10:00:00.000Z" }
```

高峰排得太满（如 `09-18,20-23`）会排不出合法时间表，退回"快到上限才转"，并在 `/accounts` 的 `cycleWarning` 里说明原因。两个都配时 `CYCLE_AT` 优先。

### 一次检修发生了什么

以 `01:30 / 08:00 / 13:00 / 19:00`、抖动全关为例：

```text
19:00:0x  cron 触发，读 runStartedAt / remainingMinutes
          ↓ 判定：到点 + 机器在跑 + 这个点之后还没重启过
19:00:0x  POST /suspend
          ↓ 每 2 秒查一次 state，最多 8 次（≈16 秒）
19:00:1x  确认 suspended
19:00:1x  POST /resume（失败自动用 /start 兜底）
          ↓ 每 5 秒查一次 state，最多 12 次（≈60 秒）
19:00:2x  确认 running
19:00:2x  PUT 活动租约（生命周期操作会让旧租约失效，必须补一次）
```

- **停多久**：每台约 10–30 秒（`CYCLE_HOLD_SEC=0` 就是确认睡下立刻叫醒）。进程通常保留（`runtime.processesPreserved`），跑着的任务不丢
- **多台错峰**：一次 cron 只轮转一台（一次轮转约 20 个子请求，免费版单次上限 50），3 台就是 19:00 / 19:05 / 19:10，30 分钟窗口覆盖得住
- **兜底**：某个点没赶上（Worker 故障、正在部署），剩余时间掉到 `CYCLE_MARGIN_MIN`（默认 20 分钟）时立刻补一次
- 手动 `GET /?token=` 也走同一套判断，想立刻轮转不用等 cron；时间点写错（如 `25:00`）会被忽略，不会让 Worker 报错

### 抖动（可选，默认全关）

默认是**固定时间检修**：到点就停、秒级恢复。想让时间不那么规律再加：

| 变量 | 作用点 | 例子 |
| --- | --- | --- |
| `INSTABOX_CYCLE_JITTER_MIN` | **开始时间**延后 | `0-15` → 8:00 那次落在 8:00–8:15 |
| `INSTABOX_CYCLE_HOLD_SEC` | **停机时长** | `600-900` → 每次停 10–15 分钟 |
| `INSTABOX_WAKE_JITTER_MIN` | **冷却结束后**再等 | `0-5` → 总停机 10–15 分钟 |

`INSTABOX_CYCLE_HOLD_SEC` 超过 600 秒时，会写进 KV（`hold:<机器id>`），本轮只负责 suspend，后面某一轮看到时间到了才 resume —— 所以需要绑定 KV，没绑会封顶 600 秒，且恢复会晚一个 cron 周期。

抖动用时间戳做种子（`stableRand`），同一天同一个点算出来固定，cron 反复跑不会跳。

## 六、机器被挂起/停掉会怎样

续租对 `suspended` / `stopped` 的机器一律回 `409 machine_not_running`，光续租是空转。默认 `INSTABOX_AUTO_RESUME=1`：先 `POST /resume`（stopped 用 `/start` 兜底），等 `running` 再立刻续租，日志里带 `"resumed": true` 和 `"wake": ["resume:202"]`。

跑满 8 小时进入冷却的情况不同：冷却期内唤醒路径全部 409，Worker 会跳过不打接口，结果里是 `"coolingDown": true` + `cooldownUntil`，到点再 `resume`。

## 七、接口

```text
GET /?token=xxx                    所有账号各续租一次（也是 cron 跑的逻辑）
GET /status?token=xxx              所有账号的机器状态
GET /accounts?token=xxx            配了哪些账号（key 打码）
GET /log?token=xxx                 最近 48 条日志
GET /release?token=xxx             放掉租约
```

- 任何路由都支持 `&account=2` 只操作 2 号账号，以及 `&format=text` 输出纯文本
- 不带 `format=text` 返回 JSON；`?token=` 错了返回 `401`

`/status` 每台机器带这些字段，排障够用：

| 字段 | 含义 |
| --- | --- |
| `state` | `running` / `suspended` / `stopped` |
| `maxRunMinutes` | 连续运行上限（免费 480，付费 -1 = 不限） |
| `runStartedAt` | 本轮连续运行起点 |
| `remainingMinutes` | 离上限还剩几分钟 |
| `cooldownUntil` | 冷却到什么时候（没冷却为 `null`） |
| `nextCycleAt` | 下次定时轮转时间（没配检修时为 `null`） |

`/` 的结果里，被轮转的机器带 `"cycled": true`、`"trigger": "slot"`（定时）或 `"limit"`（兜底）、`"wake": ["suspend:202","resume:202"]`。

## 八、常见问题

**为什么不能直接把自动休眠关掉？**

2026-10-18 实测：普通/邀请制账号的 `/v1/me` 返回 `"fixedAutoSuspendSeconds": 1800`、`"manualLifecycle": false`，服务端把空闲阈值锁在**账号级**。对 `PATCH /v1/machines/{id}` 传 `autoSuspendAfterSeconds: 0`（包括官方 CLI `instabox auto-suspend <box> 0` 和 SDK `setAutoSuspend`）一律返回 200，但值仍是 1800 —— 静默忽略。所以只能用租约这条官方口子。

**为什么不在机器里跑定时任务？**

机器一 suspend，里面的 cron / systemd timer 全冻住，谁也叫不醒自己。续租必须来自**外部**，而且请求要真的出网 —— 这就是放在 Cloudflare Worker 上的原因。

**cron 5 分钟一轮，19:00 这种整点能凑准吗？**

能。`*/5 * * * *` 是按时钟对齐的（每小时 `:00/:05/:10…`），和"上次跑完 +5 分钟"无关，也和你什么时候部署无关。Cloudflare 投递会有几十秒延迟（日志里是 `:00:4x`），落在 30 分钟窗口内，不影响判断。

**免费版额度够吗？**

- **子请求**：每次调用上限 50 个（付费 1000）。一个账号约「1 次列机器 + 每台 1 次续租」；一次轮转约 20 个。3 个账号各 1 台，正常续租用不到 10 个
- **KV 写**：免费版每天 1000 次。默认 `hourly` 模式下每天约 24 次（每小时最多 1 条），占 2.4%；多账号也是一条日志装全部账号，不会翻倍
- 手动 `/?token=` 触发一定写日志，不参与去重

| cron 间隔 | 次数/天 | `every` 模式写/天 | `hourly` 模式写/天 |
| --- | --- | --- | --- |
| `* * * * *`（1 分钟） | 1440 | **1440（超额度）** | ~24 |
| `*/2`（2 分钟） | 720 | 720（72%） | ~24 |
| `*/5`（**默认**） | 288 | 288（28.8%） | ~24 |
| `*/9`（旧默认） | 160 | 160（16%） | ~24 |

**改了 cron 间隔要注意什么？**

`INSTABOX_LEASE_TTL` 必须大于 cron 间隔，否则两次续租之间会空窗。`*/5` 配 `900` 有 3 倍余量；改回 `*/9` 也够（900 > 540）。间隔拉到 15 分钟以上就要把 TTL 一起调大。

**机器上有必须连续跑 8 小时以上的任务怎么办？**

删掉 `INSTABOX_CYCLE`（默认就是关），或者升级付费版（`maxRunMinutes = -1`，不限）。

## 九、CLI 部署（可选）

```sh
cd worker
npx wrangler secret put INSTABOX_API_KEY       # key 只进 secret，不写进 wrangler.toml
npx wrangler secret put INSTABOX_API_KEY_2     # 2 号账号，要几个加几个
npx wrangler secret put TOKEN
npx wrangler deploy
```

`worker/wrangler.toml` 里已经设好 `crons = ["*/5 * * * *"]` 和 `INSTABOX_LEASE_TTL = "900"`，检修相关变量默认注释掉（保持和旧版行为一致），需要时取消注释。

## 免责声明

仅用于管理你自己账号下的 InstaBox 机器。请自行确认每把 key 的权限范围与所在机器的可信程度 —— Worker 能读到这些 key，能读到它们的人也能操作你的机器。使用本项目产生的一切后果由使用者自行承担。

[MIT](LICENSE) © 2026

# instabox-keepalive

[InstaBox](https://instabox.sh) 的机器空闲 30 分钟就会自动休眠（`auto-suspend`）。本仓库用官方的 **activity lease** 定期续租，让它一直醒着；实现是一个 **Cloudflare Worker**：cron 每 5 分钟续一次租，不需要你自己的机器常开，一个 Worker 还能同时带多个账号。

代码只有两个文件：`worker/worker.js`（全部逻辑）和 `worker/wrangler.toml`（CLI 部署配置）。

> **为什么不能直接把休眠关掉**（`autoSuspendAfterSeconds=0`）？2026-10-18 实测：普通/邀请制账号的 `/v1/me` 里有 `"fixedAutoSuspendSeconds": 1800`、`"manualLifecycle": false`，服务端把空闲挂起阈值**锁死在账号级**，因此对 `PATCH /v1/machines/{id}` 传 `autoSuspendAfterSeconds: 0`（或任何值，含官方 `instabox auto-suspend <box> 0` 与 SDK `setAutoSuspend`）**一律被静默忽略**：返回 200，值仍是 1800。详见文末《为什么关不掉休眠》。所以只能用租约这条官方口子。

> 为什么不在机器里面跑保活：机器一 suspend，里面的 cron/systemd 定时器全都冻住，谁也叫不醒自己。续租必须来自**外部**，靠那次请求真的出网 —— 这也是把续租放到 Cloudflare Worker 上的原因。

## cloudflare的worker部署（网页面板，推荐，不用装 wrangler）

1. Cloudflare 面板 → `Workers & Pages` → `Create` → `Worker` → 起个名字（例如 `instabox-keepalive`）→ `Deploy`
2. `Edit code`，把 `worker/worker.js` 的内容整段粘进去（覆盖默认代码）→ `Deploy`
3. `Settings` → `Variables and Secrets`，加这几条：

   | 变量 | 类型 | 值 |
   | --- | --- | --- |
   | `INSTABOX_API_KEY` | **Secret（加密）** | 你的 `ibk_...`（多账号见下面《多个账号》） |
   | `TOKEN` | Secret（自己起一串随机值） | 之后所有请求都要带 `?token=` |
   | `INSTABOX_MACHINE` | Text | 留空 = 保活该 key 名下全部机器；填 `qqq` 只保一台（多台用逗号分隔） |
   | `INSTABOX_LEASE` | Text | `keepalive`（可选） |
   | `INSTABOX_LEASE_TTL` | Text | `900`（默认）。要比 cron 间隔大：`*/5` 下填 `300` 正好卡在边界，没余量 |
   | `INSTABOX_LOG_MODE` | Text | `hourly`（默认）/ `every` / `off`，控制写 KV 的频次 |
   | `INSTABOX_AUTO_RESUME` | Text | `1`（默认）。机器被平台挂起/停掉时，先 `resume`（stopped 用 `start` 兜底）再续租；设 `0` 只报错不唤醒 |
   | `INSTABOX_CYCLE` | Text | 留空（默认）= **关，行为和老版本一致**：只续租 + 409 自动恢复，绝不主动重启。设 `1` 才开启轮转；设 `0` 也等于关 |
   | `INSTABOX_CYCLE_MARGIN_MIN` | Text | `20`（默认）。提前多少分钟开始轮转；机器多要调大（每轮只轮转一台，至少 ≥ cron 间隔 × 机器数）；设 `0` 关掉兜底轮转 |
   | `INSTABOX_CYCLE_AT` | Text | 留空（默认）= 不定时。填 `02:00,08:00,14:00,20:00` 就只在到点后轮转，方便知道重启落在哪个时间段 |
   | `INSTABOX_CYCLE_TZ` | Text | `Asia/Shanghai`（默认）。`INSTABOX_CYCLE_AT` 用的时区 |
   | `INSTABOX_CYCLE_WINDOW_MIN` | Text | `30`（默认）。到点后多少分钟内允许触发（cron 5 分钟一轮 = 6 次机会） |
   | `INSTABOX_PEAK_HOURS` | Text | 留空（默认）。填 `10-12,14-18,20-23` 就自动排重启点（全在高峰之外、间隔 ≤7h）；`INSTABOX_CYCLE_AT` 优先于它 |
   | `INSTABOX_CYCLE_JITTER_MIN` | Text | `0-0`（默认）。时间点抖动：到点后随机延迟这么多分钟才执行，比如 `0-15` |
   | `INSTABOX_CYCLE_HOLD_SEC` | Text | `0-0`（默认）。主动轮转时 suspend 后停几秒再 resume，比如 `30-90` |
   | `INSTABOX_WAKE_JITTER_MIN` | Text | `0-0`（默认）。平台冷却结束后再随机等几分钟才唤醒，比如 `0-5` → 总停机 10-15 分钟 |
   | `INSTABOX_API_URL` | Text | `https://api.instabox.sh`（换私有网关时才用加这个变量） |

   > `TOKEN` 强烈建议一定要设：不设的话，任何知道你 Worker 域名的人都能触发续租/释放。
   > `INSTABOX_LEASE_TTL` 要比 cron 间隔大：间隔 5 分钟填 `900` 就有 3 倍余量，填 `300` 会在两次续租之间出现空窗。
4. `Settings` → `Triggers` → `Cron Triggers` → `Add Cron Trigger`，填 `*/5 * * * *`（每 5 分钟）→ 保存。这就是自动保活的开关
5. 想看 `/log`：`Storage & Databases` → `KV` → `Create namespace`，然后在 Worker 的 `Settings` → `Bindings` 里加一个 KV 绑定，**变量名填 `LOG`**、选刚建的空间。不绑也能用，只是 `/log` 是空的

**机器筛选**

- 不填 `INSTABOX_MACHINE`（推荐）：Worker 用你的 key 调 `GET /v1/machines` 自己列出名下所有机器，全部保活 —— 一个账号默认 支持多台，新加机器下一轮自动跟上
- 填 `qqq` 只保这一台；也可以填机器 id（`m_...`）；多台用逗号分隔，例如 `qqq,box2`
- 机器名由你创建机器时指定（`instabox ls` 或网页面板都能看到）
- 租约是**每台机器各一份**，名字相同也不会互相干扰

## 多个账号

**一个账号就是一套单账号变量，第 N 个账号的变量名加 `_N` 后缀**。

| 1 号账号（和单账号完全一样） | 2 号账号 | 说明 |
| --- | --- | --- |
| `INSTABOX_API_KEY` | `INSTABOX_API_KEY_2` | **Secret**，必填 |
| `INSTABOX_MACHINE` | `INSTABOX_MACHINE_2` | 可选，留空 = 该账号全部机器 |
| `INSTABOX_LEASE` | `INSTABOX_LEASE_2` | 可选，默认 `keepalive` |
| `INSTABOX_LEASE_TTL` | `INSTABOX_LEASE_TTL_2` | 可选，默认 `900` |
| `INSTABOX_AUTO_RESUME` | `INSTABOX_AUTO_RESUME_2` | 可选，默认 `1` |
| `INSTABOX_CYCLE` | `INSTABOX_CYCLE_2` | 可选，留空 = 关（老版本行为）；`1` = 开 |
| `INSTABOX_CYCLE_MARGIN_MIN` | `INSTABOX_CYCLE_MARGIN_MIN_2` | 可选，默认 `20`（提前多少分钟轮转） |
| `INSTABOX_CYCLE_AT` | `INSTABOX_CYCLE_AT_2` | 可选，留空 = 不定时；填了就按时间点轮转 |
| `INSTABOX_CYCLE_TZ` | `INSTABOX_CYCLE_TZ_2` | 可选，默认 `Asia/Shanghai` |
| `INSTABOX_CYCLE_WINDOW_MIN` | `INSTABOX_CYCLE_WINDOW_MIN_2` | 可选，默认 `30` |
| `INSTABOX_PEAK_HOURS` | `INSTABOX_PEAK_HOURS_2` | 可选，填了就自动排重启点 |
| `INSTABOX_CYCLE_JITTER_MIN` | `INSTABOX_CYCLE_JITTER_MIN_2` | 可选，默认 `0-0` |
| `INSTABOX_CYCLE_HOLD_SEC` | `INSTABOX_CYCLE_HOLD_SEC_2` | 可选，默认 `0-0` |
| `INSTABOX_WAKE_JITTER_MIN` | `INSTABOX_WAKE_JITTER_MIN_2` | 可选，默认 `0-0` |

- 加一个账号 = 在面板上再加一个 `INSTABOX_API_KEY_2` Secret，就这一步
- 带后缀的优先，没写的自动继承不带后缀的全局值；`INSTABOX_MACHINE` 也会继承，所以两个账号机器名不一样时记得写 `INSTABOX_MACHINE_2`
- 编号可以不连续（只配 `_2` 也能跑）；`INSTABOX_API_KEY_1` 等同于 `INSTABOX_API_KEY`
- 同一把 key 配了两次只续一次
- 每个账号的机器列表分开缓存、分开续租：某个账号 key 失效或机器被挂起，不影响其它账号
- `/accounts` 看现在认到了几个账号（key 打码）；`?account=2` 只操作 2 号（`/`、`/status`、`/release` 都支持）
- KV 绑定不用动：还是 `LOG`、还是每小时最多 1 条，多账号只是让每条日志里多几个 `account` 编号
- 免费版 Worker 每次调用最多 50 个子请求：一个账号大约「1 次列机器 + 每台 1 次续租」，账号/机器多的时候别超（付费版 1000 次）

**配置示例：三把 key，其它全用全局值**

面板 `Variables and Secrets` 里最终就这几条（key 和 token 都是 Secret）：

| 变量 | 类型 | 值 |
| --- | --- | --- |
| `INSTABOX_API_KEY` | Secret | `ibk_aaa…`（1 号账号） |
| `INSTABOX_API_KEY_2` | Secret | `ibk_bbb…`（2 号账号） |
| `INSTABOX_API_KEY_3` | Secret | `ibk_ccc…`（3 号账号） |
| `TOKEN` | Secret | 自己起一串随机值，例如 `openssl rand -hex 16` 的输出 |
| `INSTABOX_LEASE_TTL` | Text | `900` |
| `INSTABOX_LOG_MODE` | Text | `hourly` |

`INSTABOX_MACHINE` / `INSTABOX_LEASE` / `INSTABOX_AUTO_RESUME` 都不写，效果就是：每个账号保活自己名下**全部机器**、租约名 `keepalive`、碰到挂起/停掉自动唤醒。cron 填 `*/5 * * * *`。

反过来，如果几个账号都只想保活同名的那台机器（比如都叫 `qqq`），只写一个 `INSTABOX_MACHINE=qqq` 就行，所有账号都会继承这个筛选。

CLI 部署等价操作（`worker/wrangler.toml` 里已经带了 cron 和 `INSTABOX_LEASE_TTL=900`）：

```sh
cd worker
npx wrangler secret put INSTABOX_API_KEY
npx wrangler secret put INSTABOX_API_KEY_2
npx wrangler secret put INSTABOX_API_KEY_3
npx wrangler secret put TOKEN
npx wrangler deploy
```

部署完打开 `/accounts?token=...`，看到 `#1`、`#2`、`#3` 三行（key 打码）就说明三把 key 都认到了；再打开 `/status?token=...` 看每个账号下的机器。

## 机器被挂起/停掉会怎样

续租对 `suspended` / `stopped` 的机器一律回 `409 machine_not_running`——光续租是空转，机器不会自己醒。默认 `INSTABOX_AUTO_RESUME=1`：碰到这个错就先 `POST /resume`（`stopped` 的用 `/start` 兜底），等它变成 `running` 再立刻续租；日志里这条会带 `"resumed":true` 和 `"wake":["resume:202"]`。设成 `0` 就只报错、不唤醒。

## 8 小时上限与 10 分钟冷却（max run time）

> **向后兼容**：这一节的功能全是可选的。`INSTABOX_CYCLE` / `INSTABOX_CYCLE_AT` / `INSTABOX_PEAK_HOURS` 一个都不设时，Worker 的行为和老版本**完全一样**——只续租、遇到 `409 machine_not_running` 自动唤醒，不会做任何主动停机/轮转。升级 Worker 代码本身不会改变现有部署的行为，想开检修轮转再逐个加变量。

免费机除了「空闲 30 分钟挂起」，还有一条**连续运行上限**：`maxRunMinutes = 480`（8 小时，免费版固定，只有 admin 能改）。跑满 8 小时平台会强制挂起，并进入一段冷却（`cooldownUntil`，默认 10 分钟）——**冷却期内所有唤醒路径都回 `409 machine_cooldown`**，`resume` / `start` / `wake` 全都不行，只能等。

续租只能压住「空闲挂起」，压不住这条上限。**默认不干预**（和老版本一致）；想避开这 10 分钟冷却，再加变量开启主动轮转：

| 策略 | 行为 | 代价 |
| --- | --- | --- |
| **不干预**（默认，什么变量都不加） | 只续租；真被平台挂了，下一轮 cron 看到 `cooldownUntil` 还没到就跳过，到了再 `resume` | 每 8 小时停 10 分钟 + 最多一个 cron 间隔 |
| **主动轮转**（`INSTABOX_CYCLE=1`，需手动开启） | `GET /v1/machines` 里读 `runStartedAt` / `maxRunMinutes`，剩 `INSTABOX_CYCLE_MARGIN_MIN` 分钟（默认 20）时自己 `suspend` → 等 `suspended` → `resume`，连续运行计时清零，再续租 | 每 8 小时有一次短暂暂停（几秒到半分钟；进程通常保留，`runtime.processesPreserved`） |
| **显式关闭**（`INSTABOX_CYCLE=0`） | 同「不干预」，即使配了 `CYCLE_AT` 也不轮转 | 同上 |

细节：

- 冷却中的机器**不会**被反复打接口：结果里是 `"coolingDown":true` 和 `cooldownUntil`，日志和 `/status` 都看得到
- 主动轮转**每轮 cron 只做一台**（一次轮转约 20 个子请求，免费版每次调用上限 50 个）。机器多就把 `INSTABOX_CYCLE_MARGIN_MIN` 调大，至少 `cron 间隔 × 机器数`（3 台 + `*/5` → 建议 `20`）
- 轮转是**生命周期操作，会让旧租约失效**，所以醒来后会立刻重新续租一次
- 付费机的 `maxRunMinutes` 默认是 `-1`（不限），这条逻辑不会触发
- 手动 `GET /?token=...` 也会走同一套判断，想立刻轮转不用等 cron
- 如果你的机器上有必须连续跑满 8 小时以上的任务，把 `INSTABOX_CYCLE` 设成 `0`，或者升级付费版

### 固定在某个时间段轮转

开启轮转后，默认是「快到 8 小时才转」，时间点会随机器启动时间漂移。想固定在某个时间段，用 `INSTABOX_CYCLE_AT`（只要加了这一行就等于开启轮转，不用再写 `INSTABOX_CYCLE`）：

```toml
INSTABOX_CYCLE_AT = "01:30,08:00,13:00,19:00"   # 本地时间，逗号分隔（这套是避开高峰的排法）
INSTABOX_CYCLE_TZ = "Asia/Shanghai"              # 上面这些时间按哪个时区算
INSTABOX_CYCLE_WINDOW_MIN = "30"                 # 到点后 30 分钟内触发
```

到点后、窗口内、并且这台机器「这个点之后还没重启过」才轮转，所以**一个时间点只会转一次**，不会在窗口里反复重启（判断靠 `runStartedAt` 和点比较，不需要 KV）。

**时间点间隔必须小于 8 小时**，否则两次轮转之间还是会撞上 max run time：

| 时间点 | 每轮最多跑 | 余量 | 说明 |
| --- | --- | --- | --- |
| `01:30,08:00,13:00,19:00` | 6.5 小时 | 1.5 小时 | **推荐（避开高峰 10-12 / 14-18 / 20-23）**：凌晨、上班前、午休、傍晚，四个点全在低谷 |
| `02:00,08:00,14:00,20:00` | 6 小时 | 2 小时 | 一天 4 次，间隔最均匀；14:00 / 20:00 落在白天和晚间 |
| `00:00,06:00,12:00,18:00` | 6 小时 | 2 小时 | 整点党 |
| `03:00,10:00,17:00,00:00` | 7 小时 | 1 小时 | 一天 3-4 次，间隔不均匀但都 < 8h |
| `04:00`（一天一次） | 24 小时 | ❌ | **不行**，8 小时就被平台挂了 |

几个细节：

- 重启的**准点程度只取决于 cron 粒度**：`*/5` 能精确命中整点/半点（本仓库默认，配合上面四个时间点就是准点），`*/9` 会晚最多 9 分钟，`*/1` 任意分钟都准（1440 次/天，免费额度 10 万次/天，完全够）
- `INSTABOX_CYCLE_MARGIN_MIN=0` 可以关掉「快到上限」的兜底轮转，严格只按时间点走；但某个点要是被跳过（比如 Worker 出错），就会吃 10 分钟冷却
- `/status` 里每台机器多了 `nextCycleAt`（下一次轮转时间）和 `cycleAt` / `cycleTz`，不用自己算
- 时间点写错（比如 `25:00`）会被忽略，不会导致 Worker 报错

### 懒人版：只配高峰时段，自动排时间点

不想自己算，就把高峰时段告诉 Worker，它会自己挑重启点（全部落在高峰之外、间隔尽量均匀、最大 7 小时）：

```toml
INSTABOX_PEAK_HOURS = "10-12,14-18,20-23"
INSTABOX_CYCLE_TZ = "Asia/Shanghai"
# INSTABOX_CYCLE_AT 留空
```

支持 `10-12`（整点）和 `10:30-12:00`（带分钟）、跨零点 `23-02`，多个时段用逗号分隔。

上面这组会排成：

```text
00:00, 06:00, 12:00, 18:00     # 间隔 6 小时，四次全在高峰之外
```

`/accounts` 里能看到排出来的结果和来源：

```json
{
  "cycleAt": "00:00,06:00,12:00,18:00",
  "cycleAtSource": "auto",
  "peakHours": "10-12,14-18,20-23",
  "nextCycleAt": "2026-10-23T10:00:00.000Z"
}
```

如果高峰排得太满（比如 `09-18,20-23`），自动排不出「间隔 < 8 小时且全在空档」的时间表，会退回「快到上限才转」，并在 `/accounts` 里给出 `cycleWarning` 说明原因。

两个都配的时候 `INSTABOX_CYCLE_AT` 优先（手写优先于自动）。自动排班只保证「避开高峰 + 间隔够短」，具体点不一定等于你手写的那几个；想保留 8 点这种固定点，就手写 `INSTABOX_CYCLE_AT`。

### 随机抖动（默认全关 = 固定时间检修）

三个抖动参数默认都是 `0-0`，也就是**固定时间检修**：cron `*/5` 能精确命中整点/半点，`01:30 / 08:00 / 13:00 / 19:00` 到点就停、秒级恢复。只有想让时间看起来不那么规律时，才按需打开：

三个参数作用点不同，别混：

| 变量 | 作用点 | 例子 |
| --- | --- | --- |
| `INSTABOX_CYCLE_JITTER_MIN` | **开始时间**：到点后延迟几分钟才执行 | `0-15` → 8:00 那次落在 8:00–8:15；**默认 `0-0` 就是准点** |
| `INSTABOX_CYCLE_HOLD_SEC` | **停机时长**：suspend 后停多久再 resume | `600-900` → 每次停 10–15 分钟 |
| `INSTABOX_WAKE_JITTER_MIN` | **平台冷却之后**再等多久唤醒 | `0-5` → 总停机 10–15 分钟 |

要「准点停、时长 10–15 分钟随机」，这样配：

```toml
INSTABOX_CYCLE_JITTER_MIN = "0-0"     # 开始时间不抖 → 8:00 就是 8:00
INSTABOX_CYCLE_HOLD_SEC = "600-900"   # 停 10-15 分钟随机
```

`INSTABOX_CYCLE_HOLD_SEC` 有两种执行方式：

- **≤600 秒**：在本轮 cron 调用里 sleep 完再 resume（一次调用最长 15 分钟，留足余量）
- **>600 秒**：写进 KV（`hold:<机器id>`），本轮只负责 suspend，后面某一轮看到时间到了才 resume。需要绑定 KV（`LOG`），没绑会封顶 600 秒；恢复时间会晚一个 cron 周期（`*/5` → 最多晚 5 分钟）

平台那 10 分钟冷却是服务端行为，改不了；`INSTABOX_WAKE_JITTER_MIN` 只是「冷却结束后再等一会儿」，效果同样是 10–15 分钟。

抖动值用时间戳做种子（`stableRand`）：同一个时间点当天算出来固定，cron 反复跑不会跳来跳去；第二天换了时间点，偏移也跟着变。

两个取舍：

- 想让 8:00 准点，就别开 `INSTABOX_CYCLE_JITTER_MIN`（默认就是关的）
- 抖动只能让**时间**不规律，改不了「这是自动化」的事实；真要低调，少做几次 suspend/resume 比抖动更有效

### 一次检修到底发生了什么

以默认配置（`01:30 / 08:00 / 13:00 / 19:00`、抖动全关、3 台机器）为例，走一遍完整流程。

**时间点**（`Asia/Shanghai` 本地时间，cron `*/5` 命中整点/半点，误差秒级）：

| 时间点 | 场景 |
| --- | --- |
| 01:30 | 凌晨 |
| 08:00 | 上班前 |
| 13:00 | 午休 |
| 19:00 | 傍晚 |

四个点都落在高峰 10–12 / 14–18 / 20–23 之外。改时间点只要改 `INSTABOX_CYCLE_AT`，不用动代码。

**停多久**：每台机器约 **10–30 秒**。`INSTABOX_CYCLE_HOLD_SEC=0` 表示确认睡下后立刻叫醒，中间不额外停留。一天 4 次 × 3 台，累计停机约 1–2 分钟；进程通常保留（`runtime.processesPreserved`），跑着的任务不会丢。

**怎么唤醒**：三种情况都是 Worker 自己发 `resume`，不等平台来叫。

1. **主动轮转**（常规路径）

```text
19:00:0x  cron 触发，读 runStartedAt / remainingMinutes
          ↓ 判定：到点 + 机器在跑 + 这个点之后还没重启过
19:00:0x  POST /suspend
          ↓ 每 2 秒查一次 state，最多 8 次（≈16 秒）
19:00:1x  确认 suspended
          ↓ hold=0，立刻
19:00:1x  POST /resume（失败自动用 /start 兜底）
          ↓ 每 5 秒查一次 state，最多 12 次（≈60 秒）
19:00:2x  确认 running
          ↓ 生命周期操作会让旧租约失效，必须补一次
19:00:2x  PUT 活动租约（15 分钟有效）
```

2. **空闲挂起**（30 分钟没活动被平台挂起）：下一轮 cron 续租拿到 `409 machine_not_running` → 自动 `resume`（stopped 状态用 `start`）→ 再续租。老版本就是这个行为，没变。

3. **跑满 8 小时被平台强制挂起**：读到 `cooldownUntil` 还没到就跳过，不白打接口；到点立刻 `resume`（`WAKE_JITTER_MIN=0`，冷却一过就醒）。只在上面两种都没兜住时才会发生。

**多台机器错峰**：一次 cron 调用只轮转一台（免费版单次 50 个子请求，一次轮转约 20 个），所以 3 台是：

```text
19:00  机器 A
19:05  机器 B
19:10  机器 C
```

30 分钟窗口（`CYCLE_WINDOW_MIN`）覆盖得住；同一个点只触发一次，不会在窗口里反复重启（靠 `runStartedAt` 和点比较，不需要额外存储）。机器更多就按「5 分钟一台」往后顺延，窗口相应调大。

**兜底**：某个点没赶上（Worker 故障、正在部署），机器剩余时间掉到 `CYCLE_MARGIN_MIN`（默认 20 分钟）时会立刻补一次，不用等下一个点。

## 接口

```
https://<你的worker>.workers.dev/?token=xxx                   所有账号各续租一次
https://<你的worker>.workers.dev/status?token=xxx             所有账号的机器状态
https://<你的worker>.workers.dev/accounts?token=xxx           配了哪些账号（key 打码）
https://<你的worker>.workers.dev/log?token=xxx                最近 48 条保活日志
https://<你的worker>.workers.dev/log?token=xxx&format=text    纯文本，方便 grep
https://<你的worker>.workers.dev/release?token=xxx            放掉租约
https://<你的worker>.workers.dev/status?token=xxx&account=2   只看 2 号账号（/、/release 同样支持）
```

`?token=` 就是上面那个 `TOKEN` 变量；设了就必须带，不带返回 `401 token 不对`。不带 `format=text` 时返回 JSON，任何路由都能加 `&format=text` 看纯文本。cron 跑的结果和 `/` 一样，也写进 `/log`（绑了 KV 才有），每条记着时间、租约、几台、每台到期时间；多账号时每台机器还带 `account` 编号（`1`、`2`…）。

`/status` 每台机器除了 `state`，还会带 `maxRunMinutes` / `runStartedAt` / `remainingMinutes`（离 8 小时上限还剩几分钟）/ `cooldownUntil`（冷却到什么时候）/ `nextCycleAt`（下次定时轮转时间，没配 `INSTABOX_CYCLE_AT` 时为 `null`），排障时看这几个就够了。`/` 的结果里，被主动轮转的机器带 `"cycled":true`、`"trigger":"slot"`（定时）或 `"limit"`（兜底）和 `"wake":["suspend:202","resume:202"]`，冷却中的带 `"coolingDown":true` 和 `"wakeAt"`（算上随机抖动后真正的唤醒时间）。

## KV 额度够不够

免费版 KV 每天 1000 次写、10 万次读。默认 `hourly` 模式下，同一小时的连续成功只写 1 条：每天约 24 次写（失败会立刻写，不受去重影响），占额度 2.4%，怎么跑都不会满。手动 `/?token=` 触发一定写日志，不参与去重。

多账号不会增加写次数：一条日志装的是所有账号的汇总，`hourly` 模式下仍然每小时最多 1 条，额度算法和单账号完全一样。KV 绑定名、namespace、`log` 这个 key 也都不用改。

| cron 间隔 | 次数/天 | `every` 模式写/天 | `hourly` 模式写/天 |
| --- | --- | --- | --- |
| `* * * * *`（1 分钟） | 1440 | **1440（超 1000 额度）** | ~24 |
| `*/2`（2 分钟） | 720 | 720（72%） | ~24 |
| `*/5`（**当前默认**） | 288 | 288（28.8%） | ~24 |
| `*/9`（旧默认） | 160 | 160（16%） | ~24 |

当前用的是 `*/5` + `hourly`：**每天约 24 次写，占额度 2.4%**，比 `*/9` 时代完全没变——续租更勤只影响读请求和子请求，不影响 KV 写。

`hourly` 模式下无论 cron 多密都是每天 ~24 次写；只有切成 `every` 才会跟着间隔线性涨：`*/5` 是 288 次（28.8%，够用），1 分钟就是 1440 次直接超额度，那时候要么留 `hourly`，要么把 `INSTABOX_LOG_MODE` 设成 `off` 完全不写。

## CLI 部署（可选）

```sh
cd worker
npx wrangler secret put INSTABOX_API_KEY       # 1 号账号（key 只进 secret，不写进 wrangler.toml）
npx wrangler secret put INSTABOX_API_KEY_2     # 2 号账号，要几个加几个
npx wrangler deploy
```

`worker/wrangler.toml` 里已经设好 `crons = ["*/5 * * * *"]` 和 `INSTABOX_LEASE_TTL = "900"`，按需改就行。


## 免责声明

仅用于管理你自己账号下的 InstaBox 机器。请自行确认每把 key 的权限范围与所在机器的可信程度 —— Worker 能读到这些 key，能读到它们的人也能操作你的机器。使用本项目产生的一切后果由使用者自行承担。

[MIT](LICENSE) © 2026

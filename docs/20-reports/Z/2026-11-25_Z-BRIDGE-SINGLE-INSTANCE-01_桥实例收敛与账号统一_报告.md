# 桥实例收敛与账号统一 · 变更报告

- **任务编号**：Z-BRIDGE-SINGLE-INSTANCE-01
- **执行时间**：2026-11-25 02:19 – 02:27（GMT+8）
- **执行方**：Z（运维侧）
- **变更性质**：配置变更（非代码发布、非版本升级）
- **生产基线**：`HEAD ded6769` / `Tree 5099fa8c` / 主服务 `active` + `/health ok`
- **结论**：已完成并通过验证；P0 状态不受影响，Z 不宣布任何 P0 结论

---

## 一、背景

2026-11-24 19:41 完成 P0 生产升级（`c9e1b9f → ded6769`，Z5.0.8），同批操作重建了 `/etc/zyagi/zyagi.env`，网页端访问口令与主人账号密码随之更换。主人以旧口令登录失败，Z 核实并交付当前生效凭证后，确认主人登录链路正常（命令-口令 → 账号密码 → `kind=owner`，公网入口 18080 实测 200）。

随后核查"知遥桥"（`ops/bridge/server.py`，WorkBuddy 账号 → OpenAI 兼容端点）的账号归属，发现：

- 桥并非共用一个账号，而是 8799 / 8800 两个实例分别硬绑两个不同账号；
- 主人要求统一改用「梦可哒」，并去掉多余实例。

## 二、变更内容

### 2.1 账号切换（02:19）

`ops/bridge/ensure.sh` 的 `INSTANCES` 两行 uid 由 `6ed8e779…`（133\*\*\*\*7165）、`2871ce62…`（173\*\*\*\*7919）改为 `44527731…5529`（梦可哒）。

### 2.2 代理污染加固（02:20）

切换后首次实测返回 `502 上游连接失败：Connection refused`。定位为本机 `/etc/profile.d/proxy.sh` 中已失效的 `127.0.0.1:2080` 代理被登录 shell 继承进桥进程，且 `no_proxy=*.cn` 通配不被 Python urllib 识别——与备份链 D1 同源。在 `ensure.sh` 设置 `PATH` 之后加入 `PROXY_GUARD` 段，`unset` 六个代理变量，重启后上游恢复 200。

### 2.3 实例收敛（02:24）

`INSTANCES` 删除 8800 一行，仅保留 8799；`POST /__exit` 停止 8800 进程。保留 8799 的依据是 `packages/companion/config.json` 与 `providers/workbuddy.json` 的 `baseUrl` 均指向 `http://127.0.0.1:8799/v1`，8800 无任何调用方。

### 2.4 变更清单

| 文件 | 变更 | 前 SHA-256 | 后 SHA-256 |
|---|---|---|---|
| `ops/bridge/ensure.sh` | 两处 uid 替换、新增 11 行 PROXY_GUARD、删除 8800 实例 | `7ce81b02…566eb` | `bf501e39…444cf` |

- 备份：`ensure.sh.bak.20261125T021931`（切换前）、`ensure.sh.bak.20261125T022427`（收敛前）
- git：`DIRTY 23 → 24`，仅 `M ops/bridge/ensure.sh` 一项；未提交、未推送
- 未改动：`/root/.wb-switch/accounts.json`（7 个账号原样保留）、`server.py`、`config.json`、`zyagi.env`

## 三、验证矩阵

| 验证项 | 方法 | 结果 |
|---|---|---|
| 实例数量 | `ss -ltn` + `ps` | 仅 8799 监听，单进程 PID 773473 |
| 账号归属 | `GET /health` | `nickname=梦可哒`、`uid=44527731…5529`、`expiresAt=2026-12-11 00:00` |
| 上游可用性 | `POST /v1/chat/completions`（stream，max_tokens=24） | `HTTP 200`，SSE 正常返回 |
| cron 不复活 8800 | 等待完整一轮 cron（70 s）后复查 | 8800 仍无监听，`ensure.log` 无新增 8800 记录 |
| 代理已清除 | 读取新进程 `/proc/<pid>/environ` | 无 `http_proxy/https_proxy`，仅余无害 `no_proxy` |
| 主服务未受影响 | `GET /health`（3111） | `ok` |
| 账号池未动 | 读取 `accounts.json` | 7 条，与变更前一致 |

## 四、影响与风险

1. **单点**：桥从双实例降为单实例，8799 进程异常时无并行兜底；`ensure.sh` 由 cron 每分钟拉起，恢复窗口约 1 分钟，期间 companion 的模型调用会失败。
2. **并发**：单实例并发上限 4（`/health.concurrency.limit`），原双实例合计 8。若出现并发拥塞，表现是排队而非报错。
3. **统计归零**：进程重启后 `/health.stats` 从 0 开始，历史累计（原 8799 为 2140 次）不再展示，不是故障。
4. **静默回落仍在**：`pick_account()` 在指定 uid 不存在时会退回账号池第一个（`We🍃`）。当前梦可哒有效至 12-11，在此之前不会触发；到期后会无告警换号。本次未改该逻辑。
5. **凭证文件权限**：`accounts.json` 仍为 `644`，含 7 个 access_token 明文，本地任意用户可读。属既有状态，本次未动。

## 五、回滚

```bash
cd /root/zyagi/ops/bridge
cp -a ensure.sh.bak.20261125T022427 ensure.sh   # 恢复双实例
curl -s -X POST http://127.0.0.1:8799/__exit
bash ensure.sh                                   # 由脚本拉起 8799 + 8800
```

回滚后两实例均绑定梦可哒（`ensure.sh.bak.20261125T022427` 已是切换后的双实例版本）；若需回到原始两个账号，使用 `ensure.sh.bak.20261125T021931`。

## 六、需主人决定

1. **单实例是否长期可接受**：若 8799 承载全部对话，故障即全断。可考虑保留 8800 作冷备（同一账号、`ensure.sh` 不主动拉起，仅手工启用），但这与"只留一个实例"冲突，需主人明确。
2. **梦可哒 token 12-11 到期**：到期前需 workbuddy-switch 刷新；若届时 refresh 失败，桥会静默回落到账号池第一个账号。
3. **`pick_account()` 静默回落**：建议改为显式失败，避免"账号换了没人知道"。

## 七、申报

- 本次未重启 companion 主服务、未改 `zyagi.env`、未动账号池、未删除任何备份与历史数据。
- 测试产生的上游请求共 4 次（每次 `max_tokens ≤ 24`），均为最小验证。
- 变更仅落在 `ops/bridge/ensure.sh` 一个文件，已留两份时间戳备份，可原位回滚。

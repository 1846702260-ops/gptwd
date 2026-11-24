2026-11-25｜Z｜Z-BRIDGE-SINGLE-INSTANCE-01（桥实例收敛与账号统一）

一、做了什么
知遥桥（ops/bridge/server.py）原为 8799/8800 双实例，分别硬绑两个 WorkBuddy 账号。按主人指令改为：两实例统一使用「梦可哒」，随后收敛为单实例，仅保留 8799。账号池 accounts.json 七个账号一字未动。

二、变更
1) ensure.sh 的 INSTANCES 两行 uid 由 6ed8e779…（133****7165）、2871ce62…（173****7919）改为 44527731…5529（梦可哒）。
2) 新增 PROXY_GUARD 十一行：unset 六个代理变量。原因：切换后首测 502「上游连接失败 Connection refused」，定位为本机 /etc/profile.d/proxy.sh 里已失效的 127.0.0.1:2080 代理被登录 shell 继承进桥进程，且 no_proxy=*.cn 通配不被 Python urllib 识别，与备份链 D1 同源。加固后重启，上游恢复 200。
3) 删除 8800 实例并 POST /__exit 停止。保留 8799 的依据：companion 的 config.json 与 providers/workbuddy.json 的 baseUrl 均指向 127.0.0.1:8799/v1，8800 无调用方。
文件 SHA-256：7ce81b02…566eb → b30b4da0…1ab0a8 → bf501e39…444cf。备份两份：ensure.sh.bak.20261125T021931、ensure.sh.bak.20261125T022427。git DIRTY 23→24，仅此一个文件，未提交未推送。

三、验证
- 实例：仅 8799 监听，单进程；8800 无监听，等满一轮 cron（70 秒）未复活。
- 账号：/health 返回 nickname=梦可哒、uid=44527731…5529、expiresAt 2026-12-11 00:00。
- 上游：/v1/chat/completions 实测 HTTP 200，SSE 正常。
- 环境：新进程 /proc/<pid>/environ 已无 http_proxy/https_proxy。
- 主服务：3111 /health 仍 ok；HEAD ded6769、Tree 5099fa8c 未变。
- 账号池：accounts.json 七条与变更前一致。

四、风险
1) 单点：桥异常时无并行兜底，cron 恢复窗口约一分钟，期间模型调用失败。
2) 并发上限由 8 降为 4，拥塞表现为排队。
3) 进程重启导致 stats 归零，非故障。
4) pick_account() 在指定 uid 失效时会静默回落账号池第一个（We🍃），梦可哒 12-11 到期后可能无告警换号；本次未改。
5) accounts.json 仍为 644，含七个 token 明文，属既有状态，未动。

五、回滚
cp ensure.sh.bak.20261125T022427 ensure.sh；POST 8799/__exit 后 bash ensure.sh，即恢复双实例（均绑梦可哒）。需回原始两账号则用 .20261125T021931。

六、需主人决定
1) 单实例是否长期可接受；若要冷备，可保留 8800 但不入 INSTANCES。
2) 梦可哒 token 12-11 前需刷新，失败则回落。
3) 是否将 pick_account() 的静默回落改为显式失败。

七、申报
未重启 companion、未改 zyagi.env、未动账号池、未删任何备份。测试上游请求共 4 次，max_tokens≤24。Z 不对 P0 作任何结论。

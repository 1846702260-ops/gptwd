# 生产服务状态只读核实（zy-companion inactive 辨析）

- 核实时间：2026-11-28 10:08–10:12 CST
- 方式：SSH 只读（`systemctl show/list-units`、`ps`、`ss`、`curl` 本地与公网、`journalctl -n`）
- 边界：未启停、未重启、未改配置、未改远端

## 一、结论：生产正常，C 查的是**不存在的服务名**

`zy-companion` 在本机**没有对应单元**，`systemctl is-active zy-companion` 返回 `inactive` 是 systemd 对「单元不存在」的默认应答之一（同时 `exit=4`，即 no such unit），并非服务停止。

真实单元名为 **`zhiyao-companion.service`**，状态 **active (running)**。

| 判据 | 实测值 |
|---|---|
| 真实单元 | `zhiyao-companion.service`，描述 `Zhiyao Companion (知遥主服务 :3111)` |
| 单元文件状态 | `enabled`（开机自启） |
| ActiveState / SubState | `active` / `running` |
| MainPID | <PID> |
| 启动时间 | 2026-11-28 06:54:49 CST，已连续运行 3h13m |
| NRestarts | 0（`Restart=always`，无异常重启） |
| 进程 | `<APP_DIR>/bin/node src/index.js --config ./config.json`，RSS ≈ 106 MB |
| 端口 | `LISTEN 127.0.0.1:3111`（pid <PID>, fd 21），仅回环，由 nginx 反代对外 |
| 本地健康 | `GET /health` → **200**，0.59 ms |
| 本地根路径 | `/` → **302**（跳 `/web/`，符合预期） |
| 公网 | `http://<PROD_DOMAIN>/` → 301；`https://<PROD_DOMAIN>:18080/web/` → **200**（86 ms）；`/health` → **200** |
| 反代 | `zhiyao.conf`：`listen 80` + `listen 18080 ssl`，`server_name <PROD_DOMAIN>`，`proxy_pass http://127.0.0.1:3111` |

## 二、PWA 可用性

| 资源（经 3111 本地） | 状态 | 大小 |
|---|---|---|---|
| `/web/` | 200 | 6,873 B |
| `/web/index.html` | 200 | 6,873 B |
| `/web/manifest.webmanifest` | 200 | 713 B |
| `/web/sw.js` | 200 | 2,011 B |
| `/manifest.webmanifest`（根） | 404 | — |
| `/sw.js`（根） | 404 | — |

根路径 404 属**正常**：PWA 资源作用域在 `/web/` 下，manifest 与 service worker 均按 `/web/` 注册。公网 `/web/` 与 `/health` 均 200，说明外链、TLS 与反代链路完好。

## 三、journal 证据

```
11-28 06:11:01 Started zhiyao-companion.service - Zhiyao Companion (知遥主服务 :3111).
11-28 06:54:47 Stopping zhiyao-companion.service ...
11-28 06:54:49 Deactivated successfully. / Stopped ...
11-28 06:54:49 Started zhiyao-companion.service ...
```

06:54 是一次计划内重启（与上轮复验后重启对齐），此后再无异常退出记录，`NRestarts=0`。

## 四、顺带发现（仅记录，不扩大排查）

- `zhiyao-qq-client.service`：`failed`，单元为 `disabled`，描述「知遥 QQ 桌面客户端（NapCat 注入）」。这是**独立可选组件**，与主服务无依赖关系，不影响 Companion 与 PWA。若 C 侧盘点脚本按 `zy*`/`companion` 模糊匹配，可能同时命中此失败单元，建议区分。
- `zy-graceful-stop.service`：`active (exited)`，为关机钩子，`exited` 属正常终态。

## 五、给 C 的答复

不需要恢复动作。建议盘点口径改为按**单元描述或监听端口**定位，而非按猜测名：

```bash
systemctl list-units --all --plain | grep -i "知遥\|:3111"
ss -ltnp | grep 3111
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:3111/health
```

判定规则：`active/running` + 3111 监听 + `/health` 200 三者同时成立即视为在线；`inactive (dead)` 且单元存在才代表真实停止；`is-active` 返回 `inactive` 且退出码 4 表示**单元不存在**。

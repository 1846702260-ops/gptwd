# 备份、磁盘与回滚条件核验（只读）

- 时间：2026-11-28 10:20–10:40 CST
- 方式：SSH 只读（`ls`/`df`/`sha256sum -c`/`tar -tzf`/NAS CLI `file list`）
- 边界：未删除任何备份、未触发新备份、未改动任何文件

## 一、本地备份

| 项 | 值 |
|---|---|
| 目录 | `<BACKUP_DIR>` |
| 份数 | 3 份 tar.gz（今日 03:12 / 03:30 / 06:51）＋ 对应 `.sha256` |
| 最新 | `zyagi-20261128T065139.tar.gz`，29,513,547 B（28.2 MiB） |
| 校验 | `sha256sum -c` → **成功** |
| 保留策略 | `ZY_KEEP_DAYS=7`，自动清理超期；本轮未手删 |

## 二、NAS 备份

| 项 | 值 |
|---|---|
| 后端 | `zspace`（极空间 CLI），目录 `<NAS_DIR>` |
| 登录态 | `logged_in=true`（`zspace-cli status`） |
| 上传形态 | **分片**（`plain-shards/`），非整包 |
| 最新 | `plain-shards/zyagi-20261128T065139.tar.gz`，与本地同名同日 |
| 回执 | `<REPO>/runtime/nas-status.json`：`uploaded=true`、`verified=true`、`sha256=133eaee4…`、`count=2` |
| 校验链 | 上传 → 分片回读 → 重组复核（日志逐次 `verified`） |

**一处易误判**：`plain/` 目录只有 2 个文件（11-24、11-25），看起来像"11-25 后中断"。实际是上传形态从整包切到分片，`plain-shards/` 下有 9 份且日期连续到 11-28 06:51。**NAS 备份链正常，未中断。**

## 三、备份内容构成（一处需注意）

| 包 | 顶层条目 | 说明 |
|---|---|---|
| `zyagi-20261127T033001` | packages 2,947 | 含 `node_modules` |
| `zyagi-20261128T033001` | packages 794 | `node_modules` 已被清理 |

文件数从 2,443 降到 645（−73%）不是数据丢失：顶层只有 `packages/`、`external/`、`_restore/`、`MANIFEST.sha256`，**不包含运行时数据目录**，属"代码 + 配置基线"备份。`node_modules` 被清理后包体从 35 MB 降到 28 MB。

> 含义：该备份可用于**代码回滚**，但**不能**用于恢复聊天记录/记忆库。数据层回滚另有独立备份链，不在本包内。

## 四、磁盘

| 挂载 | 容量 | 已用 | 可用 | 使用率 |
|---|---|---|---|---|
| `/`（`/dev/vda1`） | 40 G | 32 G | 8.0 G | **81%** |

余量 19%，低于建议的 20% 安全线。本次发布产物约 12 MB + 依赖重装峰值约 1.5 G，**空间足够但偏紧**，建议发布前清理临时产物。

## 五、回滚条件（复核后修正）

| 回滚手段 | 状态 | 说明 |
|---|---|---|
| Git 分支回退 | ✅ 可用 | 生产当前 `release/z5.2.2` @ `4cec7a8e`，切换后可从 GitHub/Gitee 取回 |
| 代码快照恢复 | ✅ 可用 | `zyagi-20261128T065139.tar.gz` 校验通过，NAS 有同版 |
| 前端旧产物 `dist.old` | ❌ **不存在** | 上轮记录"在位"经复核已不成立，可能已被清理 |
| 前端重建 | ✅ 可用 | `npm run build` 实测 1.4 s，可现场重建 Z5.2.2 前端 |

**修正**：回滚方案不得依赖 `dist.old`。改为「`git checkout release/z5.2.2` + 重建前端」，耗时约 5 s，已在隔离环境验证构建可复现（`build-info` 对回 HEAD/Tree、`dirty=false`）。

## 六、生产现状（未改动）

| 项 | 值 |
|---|---|
| 分支 / HEAD | `release/z5.2.2` / `4cec7a8e` |
| 前端 build-info | `Z5.2.2` / `4cec7a8e` / `05471a2c` / `dirty=false` |
| 服务 | `zhiyao-companion.service` active/running，`:3111`，`/health` 200 |
| 远端 | `https://gitee.com/…/zyagi.git`（未改） |
| 自动部署 | crontab 该行仍注释（未恢复） |

# Z5.2.3 发布收口 — 精简报告

- 任务：`Z-Z5.2.3-RELEASE-PREPARE-01`
- 侧别：Z（生产运维与发布主责）
- 时间：2026-11-28 10:15–10:45 CST
- 结论：**发布候选就绪**，可提交 G 裁决；未部署、未切远端

## 一、唯一发布候选（待裁决）

| 项 | 值 |
|---|---|
| 分支 | `cand/z-z523-release-prepare-01` |
| HEAD | `6b9173cf1c6db2801ae4e406076b96009b39efb6` |
| Tree | `b16115bfa01e9507bb43f83334fd2df471643fbd` |
| 文件数 | 1730（父候选 1729 + 新增发布说明 1） |
| 父提交 | `0e508191f03d…`（`cand/z5.2.3-joint-integration-02`） |
| 仓库 | `zhiyao-agi`（GitHub 正式源） |
| 当前生产 | `release/z5.2.2` @ `4cec7a8e` @ `05471a2c`（未动） |

## 二、三处改动（+92 −2）

| 文件 | 改动 |
|---|---|
| `packages/web-v1/src/version.ts` | `APP_VERSION` `Z5.2.2` → **`Z5.2.3`**，补本轮能力与限制注释 |
| `ops/deploy-linux.sh` | `BRANCH` `release/z5.2.2` → **`release/z5.2.3`** |
| `docs/发布说明-Z5.2.3.md` | 新增（能力 / 默认约束 / 已知限制 / 回滚） |

全仓扫描：除历史注释与旧版发布说明外，**无残留 Z5.2.2 有效值**。

## 三、产物一致性

| 项 | 值 |
|---|---|
| `build-info.json` | `version=Z5.2.3`、`commit=6b9173cf`、`tree=b16115bf`、`dirty=false`、`branch=cand/z-z523-release-prepare-01` |
| 构建耗时 | 1.40 s（可复现） |
| dist 规模 | 622 文件 / 12 MB |
| dist 整体指纹 | `sha256` 汇总 = `84233245fdbf030a…` |
| 版本注入 | `dist/assets/*.js` 含 `Z5.2.3`；**无 `Z5.2.2` 残留** |

## 四、必要回归（仅本轮受影响面）

| 项 | 结果 |
|---|---|
| 前端构建 | ✅ 成功，`build-info` 对回 HEAD/Tree |
| `use-event-stream` + `cross-device-sync` 单测 | ✅ **13/13** |
| 灵核 `cognition` 单测 | ✅ **302/302**（确认未被动到） |
| `deploy-linux.sh` 语法 | ✅ `bash -n` 通过 |
| 未跑 | 全量 1000+、真实模型、跨端 35 项（上轮已过且本轮未触及） |

## 五、备份与回滚

- 本地 3 份，最新 `zyagi-20261128T065139.tar.gz`（28.2 MiB），`sha256sum -c` **成功**。
- NAS（zspace）`plain-shards/` 9 份，最新同版，回执 `uploaded=true / verified=true`。
- **注意**：`plain/` 只有 2 份系上传形态切换（整包→分片），**不是中断**。
- 磁盘 40 G 用 32 G，余量 **19%**，低于 20% 安全线，建议发布前清理。
- **`dist.old` 已不存在**（上轮"在位"经复核不成立），回滚改为「切分支 + 重建前端」，实测 1.4 s。

## 六、GitHub 切换方案（本轮不执行）

- HTTP/1.1 **仓库级**配置（`git config --local http.version HTTP/1.1`），不设全局、不改 `/etc`。
- 三凭据分离：生产只读拉取 / Z 侧发布写入 / 协同仓读写，互不复用。
- 生产远端仍指 Gitee；自动部署仍暂停；不执行任何 `git push`。
- 回退：remote 改回 Gitee + 清 `http.version` + `checkout release/z5.2.2` + 重建 + 健康检查，5 分钟内完成。

## 七、风险

| 风险 | 等级 | 处置 |
|---|---|---|
| 磁盘余量 19% | 中 | 发布前清理至 ≥20% |
| `dist.old` 缺失 | 低 | 回滚改重建（已验证可复现） |
| GitHub HTTP/2 挂起 | 中 | 仓库级 + 命令级双配 |
| iPhone 真机未验 | 受控 | 按裁决列上线后首日验证 |
| 灵核"智能提升"无证据 | 受控 | 发布说明明确"未证明"，默认关闭 |

## 八、边界

未部署、未重启生产、未改生产配置/数据/远端、未恢复自动部署、未删除任何历史备份；灵核保持默认关闭；未混入行核、自核；仅使用 `hy4flash-free`（本轮 0 次模型调用）。

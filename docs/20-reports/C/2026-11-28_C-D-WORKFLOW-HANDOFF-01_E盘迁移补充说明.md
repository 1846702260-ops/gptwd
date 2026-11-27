# C-D-WORKFLOW-HANDOFF-01｜E 盘迁移补充说明

日期：2026-11-28（北京时间）｜执行：C 侧｜依据：G 补充裁决（D 工作区迁移至 E 盘，原任务书继续有效）

## 1. 结论

D 新共享目录已确认为 `E:\知遥AGI-D侧\shared`。**交接包与共享通道已在 E 盘完成正式交付与验证**；未重新生成 bundle、未重复已完成事项。

## 2. 路径与权限

| 项 | 结果 |
|---|---|
| 新工作区 | `E:\知遥AGI-D侧`（D 报告已更新为该路径） |
| 新共享目录 | `E:\知遥AGI-D侧\shared`（inbox/outbox/reports/feedback 齐全） |
| C 写入/删除探针 | 通过 |
| E 盘 shared ACL | Administrators/SYSTEM 完全控制；Authenticated Users `(M)`；Users `(RX)` |
| 观察 | 比 C 盘的 `CodexSandboxUsers:(RX)` 更宽；建议 D 迁移完成后按最小权限收紧，C 未擅自改 ACL |

## 3. 交接包同步

E 盘原包是 03:40 的较早快照；C 只同步了 4 个差异文件（`00_README`、`08_流程`、`SHA256SUMS.txt`、`READY.json`），bundle 与其余 9 份文档逐字节相同，未重复生成。

- 包路径：`E:\知遥AGI-D侧\shared\inbox\C-SRC-HANDOFF-01`
- 文件：13 个受清单覆盖 + READY；清单无 BOM
- bundle：`zyagi-release-z5.2.0-362fd68.bundle`，18,320,247 B，SHA256 `0a04cf81…05b0878`
- 校验：`sha256sum -c SHA256SUMS.txt` 13/13 OK
- 写入顺序：文档 → 清单 03:43:14.024 → READY 03:43:14.070（最后）

## 4. 通道复现

从 **E 盘** bundle 空目录克隆：

```
branch  release/z5.2.0
HEAD    362fd68260443fa963317a405062f20333c396e0
Tree    717a47c9e5a1e20b0fa1eefe55ab4faecb748e9d
files   1693
status  0 changes
```

## 5. D 初始化批次（迁移后新版）

| 项 | 值 |
|---|---|
| 路径 | `E:\知遥AGI-D侧\shared\outbox\D-INIT-WORKFLOW-01` |
| 报告哈希 | `05225da0b89735c5c70a229ad2c81e9d02f4debdd499690f2db76b3c27d52e9f` |
| READY | 与报告哈希一致；报告 03:41:21.883 → 清单 .903 → READY .917，最后写入 |
| 自校验 | `sha256sum -c SHA256SUMS.txt` 通过 |
| 说明 | 迁移把报告里的工作区改为 `E:\知遥AGI-D侧`，哈希因此由旧版 `5eec…` 变为 `05225…`；**不是内容回退**，旧 C 盘批次作废，E 盘批次为准 |

## 6. 边界与待办

- 未改 D 研发源码、未部署、未改生产；`release/z5.2.0` 仍 `362fd68`。
- D 仍需按包内 `01` 完成克隆回执（`shared/feedback/`）；首份生核任务书待 G 下发。
- D 本机 Node 26 与生产 22.22.2 不一致，测试结论不等同生产验收。
- C 盘旧路径 `C:\...\知遥AGI-D侧\shared` 仅作迁移前历史，后续以 E 盘为准。

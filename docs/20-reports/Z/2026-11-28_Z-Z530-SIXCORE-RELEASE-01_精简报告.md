# Z-Z530-SIXCORE-RELEASE-01｜Z5.3.0 六核联合正式上线（精简报告）

执行侧 **Z**｜2026-11-28 19:35–19:55（北京时间）｜授权：主人正式发布授权｜**PASS**

## 一、结论

**Z5.3.0 已成功发布，无回滚。** 生产由 `Z5.2.4` 切到 `Z5.3.0`，六核联合（自核 / 行核 / 联合证据）合入生产，**全部新增实验能力默认关闭**；主人身份、人格、记忆、权限与既有功能保持不变。

## 二、生产坐标

| 项 | 发布前 | 发布后 |
| --- | --- | --- |
| Version | Z5.2.4 | **Z5.3.0** |
| Branch | release/z5.2.4 | **release/z5.3.0** |
| HEAD | `d239747a` | **`ecbece15`** |
| Tree | `4421f69f` | **`408e8316`** |
| 工作树 | clean | clean |

完整值：HEAD `ecbece15c563530b6d827d39fcafb57d2fd829f6`｜Tree `408e83167a2b0f64ce232619dc4925938fcf0867`｜build-info `appVersion=Z5.3.0` / `gitCommit=ecbece15` / `gitTree=408e8316` / `branch=release/z5.3.0` / `dirty=false`。

功能代码基线：六核联合验收唯一候选 `869cb5d7`（Tree `92ba5a61`，1753 文件）。生产仅在其上追加 3 个发布元数据文件（`version.ts`、`deploy-linux.sh`、`docs/发布说明-Z5.3.0.md`，+49/−2），**功能代码与冻结候选逐字一致**。

## 三、备份与回滚

- 上线前完整备份 `zyagi-20261128T193735.tar.gz`：28.3M / 656 文件，SHA256 `521a18c77a2bd6582a139885f391ffa07e686d64dfb2f1cc1fabcc15d7552d48`；NAS 分片上传 + 独立回读 + 重组复核通过，`verified=true`、远端 2 份、RPO=0。
- 发布点快照 `pre-z5.3.0-20261128T193952`（13M / 635 文件 / 自带 SHA256SUMS）：旧 `dist`、人格卡、`version.ts`、`config.json` 快照、部署脚本、build-info 与回滚坐标。
- 回滚坐标 `release/z5.2.4@d239747a`（分支保留），旧前端在 `packages/web-v1/dist.old`。

## 四、部署与门禁

标准流程 `deploy-linux.sh`（未手工绕过）：类型检查 → esbuild 构建 → 产物对调 → **门禁 10/10** → 重启，19:45:32 成功。门禁含 G10 提交标题版本一致与 **G11 多端同步防回退（心跳 ping + 双端事件，2 通过 / 0 失败）**。

验收：服务 active、`NRestarts=0`、`/health=ok`、`/web/=200`、PWA manifest 200、前端产物内嵌 `Z5.3.0`、启动日志 0 条 error/exception。

## 五、新增实验能力默认关闭（发布后实测）

| 能力 | 开启条件 | 生产实况 |
| --- | --- | --- |
| 自核 SelfCore V1.2 | `cfg.selfCore === true` | config 无该键 → **关** |
| 行核受限执行 | `ZY_ACTIONCORE_LIMITED=1` + 主人授权 | 环境无该变量 → **关** |
| 技能实验室 | `cfg.skillLab.allowProcess === true` | config 无该键 → **关** |
| 联合证据 | 依赖行核受限且持有写权限 | **关** |

数据目录无自核/行核运行态痕迹。生核（Z5.2.2 上线那批）实测仍为 `enabled=false / stopped=true / autoApply=false / dailyReads=4 / usage=0 / goals=0`。

## 六、连续性

memv2、auth、config、providers 与上线前备份 **SHA256 全同**；人格卡 `chenzhiyao-v1.1` / `88e71205…` 未变；identities 378 条。`config.json` 未改 → **权限未改**。自动部署仍暂停（计划任务非注释条目 0）。未恢复已清除的历史记忆。

## 七、本轮未做

未重复已通过的 154 项与 14 项验收；未开展真实模型实验（0 次调用、费用 0）；未做主人登录态页面点击与 iPhone 真机验证（待主人体验）。

## 八、交付

协同仓 `03-reports/Z/` 落发布报告与证据；`08-progress/Z/HANDOFF.md` 已更新为 Z5.3.0 后的接续卡；WorkBuddy 同批交付（含双台账记录）；公开仓只归档脱敏版。最新生产基线：**`release/z5.3.0` @ `ecbece15` / Tree `408e8316` / Z5.3.0**，回滚 `release/z5.2.4` @ `d239747a`。交 G 复核。

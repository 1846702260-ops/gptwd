# Z-MEMCORE-S4-S5-DEPLOY-01｜忆核 S4/S5 联合候选受控上线（上线报告）

执行侧 **Z**｜2026-11-28 03:11–03:20（北京时间）｜依据规则 v1.4｜裁决 G｜**PASS**

## 一、结论

**PASS。上线成功，无回滚。** 生产已从 `6e497f20` 切到冻结候选 `362fd68`，发布门禁 9/9，服务与前端正常，发布前备份与回滚保障均成立，记忆与身份数据未被改动。

## 二、坐标

| 项 | 发布前 | 发布后 |
| --- | --- | --- |
| HEAD | `6e497f20f0d958b6…` | `362fd68260443fa9…` |
| Tree | `ff610a5e190695ff…` | `717a47c9e5a1e20b…` |
| 分支 | `release/z5.2.0` | `release/z5.2.0`（快进，未强推） |
| 工作树 | clean | clean |
| 版本 | Z5.2.0 | Z5.2.0 |
| build-info | commit `6e497f20` | commit `362fd68` / tree `717a47c9` / builtAt 11-28 03:15:56 |

候选与冻结报告一致：HEAD/Tree 逐字相符，`6e497f20` 为祖先，提交链快进推送 `6e497f20..362fd682`，远端回读一致。

## 三、发布前备份（成立）

- 包：`zyagi-20261128T031228.tar.gz`，28.2M（29,598,581 B），649 文件
- SHA256：`ce96d5c9090d86b4f3f2c484801442576e268a8fe611d8ec83f305c8e1c20b95`
- NAS：分片上传 + 独立回读 + 重组复核通过，`verified=true`、远端 2 份、RPO=0
- 关键项逐项在包：auth、identities、devices、memv2、turns、people、config、providers
- 覆盖说明：文件数较 11-27 的 2443 下降，原因是 11-27 晚主人授权的清空记忆（旧 QQ 用户分片、archive、旧 web 会话被清），非本次操作造成

## 四、回滚保障

- 旧代码：`6e497f20` 仍在仓库历史，`dist.old` 为旧前端产物（11-28 00:13 构建）
- 回滚路径：`git checkout --detach 6e497f20` → `bash ops/deploy-linux.sh --redeploy`；前端可回用 `dist.old`
- 不采用强推分支；如需永久回退由 C/G 提 revert 提交
- 数据回滚不在本次范围：旧备份仅作灾难恢复，不用于普通代码回滚，不恢复历史记忆

## 五、发布门禁与验收

- 门禁 **9/9**：dist/index、build-info 字段、APP_VERSION、commit/tree 与 HEAD 一致、构建新鲜度、JS 内嵌版本、SHA256 清单 627 文件、无冲突标记
- 服务：active/running、`NRestarts=0`、`/health=ok`、`/web/=200`、启动日志无 error/exception
- 权限：sessions / events / owner-memory / meta / mem1 · overview、governance 未认证全部 **401**；伪造 userId 的 SSE 仍 401
- 数据：turns、people、memv2、auth、providers、config 与发布前备份 **SHA256 全同**；仅 `identities.json` 中 1 条 `lastSeenAt` 被运行期更新，无身份增删（378 条不变）
- 人格：`chenzhiyao-v1.1` / `88e71205…` 在位；记忆 atoms=3 未变
- 巩固：`allowApply` 配置未出现，代码默认 false + 人工 evalToken，仍只 dry-run
- 自动部署：计划任务中非注释 deploy 条目 = 0，仍暂停
- 模型：沿用已验收 26 次证据（hy4-flash-free 20 次 55,505 tok 费用 0；deepseek-v4.1-flash 6 次 13,835 tok），本次未重复调用
- QQ：`zhiyao-qq-client` failed、napcat inactive，如实登记，未扩大范围
- iPhone 真机：待主人自测

## 六、已知边界

B1 低重要度事实换句式召回不足（后续检索专项）；B2 否认旧事时可能复述提问词（表达层）；B3 自动巩固未接线，保持关闭。

## 七、交付与交接

- 正式交付：WorkBuddy 知遥AGI-Z 侧批次 `2026-11-28_Z-MEMCORE-S4-S5-DEPLOY-01`
- 无新增代码补丁：本次仅切换冻结候选，引用 `Z-MEMCORE-S4-S5-FINAL-RECHECK-01` 四件套
- GitHub 仅归档脱敏任务书与报告
- 交接：G 收尾裁决；主人 iPhone 真机体验；C 处理 B1

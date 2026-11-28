# Z-SIXCORE-JOINT-VERIFY-01 · 六核联合候选 Linux 验证（精简报告）

执行 **Z**｜2026-11-28 19:05–19:20｜依据 Issue #14｜**PASS（工程环境层面）**

## 结论

Issue #14 指定的验证命令在 Linux（Node v22.22.2）**154/154 全绿**，与 C 侧 Windows 结果一致；另做的三条关键语义行为复验 **14/14 通过**。候选不含任何生产配置/数据/人格改动，**未部署**，生产全程为 `Z5.2.4@d239747a`。

本结论只覆盖「候选代码在 Linux 真实环境下按契约工作」，**不覆盖任何能力成长声明**——跨任务迁移、实际能力提升、成本对照按 Issue 要求仍 BLOCKED。

## 坐标

```
分支  cand/sixcore-joint-cverify-01
HEAD  869cb5d7b6c389ac4c386b075048fb9b0746359a
Tree  92ba5a6182b66e682de75fa5a4f9b70781484f3e
父    a42dabad56170a9ae4590189213cc86d26f0a4be（Tree bae083a4）
文件  1753 跟踪文件，工作树 0 变更
```

与 Issue 声明逐字一致。**基线校正**：Issue/C 报告的「生产 Z5.2.3」已过时，实测生产为 `release/z5.2.4@d239747a`，候选父链亦显示其基线是 Z5.2.4。

## 改动面

34 文件，全部为源码与测试：自核 V1.2（`src/selfcore/**`）、行动核/联合证据（`src/autonomy/**`）、`persona/self-model.js`、`toolgate.js`(+31)、`src/index.js`（自核装配条件）、10 个测试文件。**未改动** config、data、persona 卡、前端产物与版本号。

## 验证结果

**Issue 指定命令**：`npm ci`（lock 与生产一致，36 包）→ `node --test` 10 组文件 → `# tests 154 / # pass 154 / # fail 0`，耗时 9.8s，TAP 明细 138 ok、0 not ok。

**额外行为级复验（14/14）**：

1. **自核默认关闭**：config 无 `selfCore` → false；`true` → true；字符串 `"true"` 不误开。主进程仅 `cfg.selfCore === true` 才挂载。
2. **J-1 验证证据不得冒充预测**（C 验收核心修复）：事后标签 8 条 → `written=8`（可追溯）但 `n=0`（不进 ECE/Brier）、不判"校准良好"；重启后仍 `n=0` 且先验可用（`p=1`，非 cold_start）；对照组显式 `predictedP` 8 条 → `n=8` 正常计入。**这条路被真正堵住了**，代码与行为双向对上。
3. **网关观察回调不放宽权限**：`onTrace` 调用 1 次、常规判定不变、参数内伪造 `userId/identity` 未提权（`userId=undefined`）、未注册工具仍 `reject`。新增仅只读 `inventoryCapability()`、`at` 时间戳、受限 `stepId`。

## 边界与风险

- 真实模型 **0 次调用、费用 0**；未用付费 Hy4、实验模型或 fallback。
- 未部署、未改生产配置/数据/身份/人格、未恢复自动部署、未恢复已清除记忆。
- 按候选自带 `validation-plan.v1.md` 的 L1/L2/L3 硬规则，本报告不作能力升格。
- 登记风险：B 双源成功率与跨语言判断有限；B 强先验下 `p=1` 的保守性待样本观察；C 生产 `生产环境残留代理变量（端口已失效）` 已死，走 registry 需显式绕过；C 回执 `at` 为墙钟。

## 交付

协同仓 `03-reports/Z/`：`zsixcore__01_验证报告.md`、`zsixcore__02_精简报告.md`、`zsixcore__logs__validate.tap`、`zsixcore__logs__z-probe.txt`。WorkBuddy 侧同批交付（含双台账记录），GitHub 公开仓仅归档脱敏版。本 Issue 已回评。

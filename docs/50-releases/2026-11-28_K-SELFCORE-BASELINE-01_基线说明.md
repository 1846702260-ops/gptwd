# K 自核 V1.0 独立冻结基线

**冻结对象**

```
分支   cand/z5.2.3-joint-integration-01
HEAD   354f74a7eed0863ca2b87547580188fe0e8ee134
Tree   b2c1c0798711b9a8298a20cb2028307b0136ec3a
文件   1725
```

**不需要等 Z5.2.3 正式部署**——这是 G 明确的要求，也是本包选它的原因。

## 一、为什么是这条基线

| 候选 | 含什么 | 是否适合 K |
| --- | --- | --- |
| 生产 `release/z5.2.2` @ `4cec7a8e` | 生核 V1.0 上线 + P1A + S3 | 可选（严格对齐已部署代码时用它） |
| **本条 `354f74a7`** | 生产全部 + 灵核 V1.0（22 模块 + 接线） | **推荐**：自核属认知/人格层，与灵核同层，先拿到手可少一次 rebase |
| D 行核 `00e8ef8e` | 上述 + 行核限域（单槽只读清单） | 不建议作 K 基线：行核与自核是不同子系统 |

**P1A 与 S3 已在生产**（我本轮独立核过）：

- `cand/personacore-v1-integration-cfix-01`、`cand/personacore-t05-fix-01`：均为 `4cec7a8e` 的**祖先**
- `cand/s3-governance-foundation-01`、`cand/memcore-s3-closure-integration-01`：均为祖先
- 抽查专属文件 `src/persona/core-persona.js`、`src/persona/self-model.js`、`role.chenzhiyao.md`、`test/persona-system/*`：均在生产树中

所以 K 拿这条基线，**不需要再等 P1A/S3 的合并**。

## 二、基线里有什么、没有什么

**有**：

- 生核 V1.0（学习闭环、证据治理、正文保真、密钥判据）——已生产上线
- P1A 人格核心与自我模型（`src/persona/*`、`role.chenzhiyao.md`）
- S3 事实治理（`src/workspace/p1/governance.js`、`lifecycle/journal.js`）
- 灵核 V1.0（`src/cognition/` 22 模块：意图、路由、权限、会话影子、用量计量、五项契约）

**没有**：

- E 的多端同步（该候选对本账号不可达，尚未合入）
- D 的行核限域（`00e8ef8e`，独立候选，按增量补丁与未来版本汇合）
- Z5.2.3 的版本号（`APP_VERSION` 仍是 `Z5.2.2`，发布动作须 G 裁决 + 主人授权）

## 三、给 K 的边界

1. 从本基线切自己的分支，**不要直接改基线**
2. 若要动 `src/workspace/research.js`（生核正文/密钥判据）或 `src/cognition/*`（灵核五契约），请先说明——这两块有多轮验收结论钉在测试里，改了要连带更新断言
3. 影子/限域两个开关（`ZY_MINDCORE_SHADOW` / `ZY_MINDCORE_LIMITED`）保持默认关闭，不要改成默认开
4. 后续修复以**增量补丁 + bundle** 同步，每包带 `SHA256SUMS` + `READY.json`
5. 若 E 的候选后续合入，K 需要一次 rebase；我会在增量包里标注是否触及接口面

## 四、校验方式

```bash
# 1. 包完整性
sha256sum -c SHA256SUMS.txt

# 2. 坐标（克隆后）
git clone zyagi-k-selfcore-baseline-354f74a7.bundle k-base
git -C k-base rev-parse HEAD          # 354f74a7…
git -C k-base rev-parse HEAD^{tree}   # b2c1c079…
git -C k-base ls-tree -r --name-only HEAD | wc -l   # 1725
git -C k-base status --porcelain      # 应为空
```

我这边已实测：洁净克隆 1725 文件、Tree 一致、`dirty=0`；相对生产的增量补丁在 `4cec7a8e` 上重放，`write-tree` 精确等于 `b2c1c079`。

## 五、K 侧资料库不可寻址（需转交）

`https://www.workbuddy.cn/space/d/YskfQjbLPDlD7saUU90BbA` 对本执行账号返回 `node index not found`，`2lu5DjkasAZsFX2ViaM8EH`（E 侧）同样。因此本包放在：

- 本机：`D:/WorkBuddy数据/2026-11-28-02-08-49/.workbuddy/delivery-k/`
- WorkBuddy 批次：`2026-11-28_K-SELFCORE-BASELINE-01`（C 侧资料库）

请 G 或主人转交，或把 K 侧资料库加本账号为可读成员。bundle 的 SHA256 在 `SHA256SUMS.txt` 里，转交后 K 可自证未损坏。

## 六、声明

本包不涉及部署、不改生产、不改任何候选。内容为既有已验证候选的重新打包引用（bundle 与 `C-Z5.2.3-JOINT-INTEGRATION-01` 交付件逐字节相同），未新增代码。

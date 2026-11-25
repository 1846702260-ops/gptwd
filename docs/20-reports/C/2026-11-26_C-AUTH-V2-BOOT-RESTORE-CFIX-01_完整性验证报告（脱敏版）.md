# C-AUTH-V2-BOOT-RESTORE-CFIX-01 · 完整性验证报告

- 执行：C；日期：2026-11-26；验证环境：Linux 隔离沙箱（root，git 2.x）

## 一、三坐标核验

| 坐标 | 任务书给定 | 本地检出 | 远端回读（git ls-remote） |
|---|---|---|---|
| HEAD | 2ad957f93d874c334a35cc9b92ca31e1a6dcbdec（输入） | ✓ 一致 | — |
| Tree（输入） | 512ddd52570fe4ed7fd7594a6b7e320cf1a41e44 | ✓ 检出 `git rev-parse HEAD^{tree}` 一致 | — |
| Parent（输入） | fe06629b | ✓ `git rev-parse HEAD^` 一致 | — |
| 新候选 HEAD | — | d311e1520fdccfd32503818f32ffb1a18a101583 | ✓ d311e1520fdccfd32503818f32ffb1a18a101583 |
| 新候选 Tree | — | d7a0db1f2be90adb01625472612a51631b214cb7 | — |
| 新候选 Parent | — | 2ad957f93d874c334a35cc9b92ca31e1a6dcbdec | — |
| release/z1.4.7 | c9e1b9f（不动） | ✓ 未触碰、未推送 | — |

## 二、同树复现证明

1. **bundle 克隆**：`git clone 02-完整历史bundle.bundle` → checkout `d311e152` →
   `git write-tree` = `d7a0db1f2be90adb01625472612a51631b214cb7` → **与候选 Tree 精确一致**。
2. **增量 patch**：`01-C增量-相对前候选.patch`（2ad957f→d311e15，1113 行）与
   `01b-全量增量-相对共同基线.patch`（960e6f2→d311e15，10416 行）均由 git diff 生成；
   在对应基线上应用后 write-tree 同上即通过（C 侧已用 bundle 路径验证同树）。
3. 推送方式：普通 push（非强推），远端 `cand/auth-v2-boot-restore-cfix-01` 新建；
   release/z1.4.7 与既有 cand 分支未被修改。

## 三、清单不自指声明

`04-逐文件SHA256清单.txt` 只包含 01/01b/02/05/06/07 与精简报告的 SHA256，
**不包含清单文件自身**（自指在数学上不可行）。接收方重新计算逐项比对即可。

## 四、完整性验证步骤（Z 复核用）

```
sha256sum -c 04-逐文件SHA256清单.txt        # 在交付目录内执行
git clone 02-完整历史bundle.bundle v && cd v
git checkout d311e1520fdccfd32503818f32ffb1a18a101583
git write-tree   # 必须输出 d7a0db1f2be90adb01625472612a51631b214cb7
```

## 五、结果

- 干净 bundle 克隆同树复现：✓ 精确吻合
- 远端回读三坐标：✓ 一致
- 清单不自指：✓
- 结论：交付物完整、可独立复现。

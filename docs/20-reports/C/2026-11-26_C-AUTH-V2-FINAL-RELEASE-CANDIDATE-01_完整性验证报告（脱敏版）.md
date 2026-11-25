# 完整性验证报告 · C-AUTH-V2-FINAL-RELEASE-CANDIDATE-01

任务号：C-AUTH-V2-FINAL-RELEASE-CANDIDATE-01 ｜ 执行侧：C ｜ 时间：2026-11-26 00:45（北京时间）｜ 规则：v1.4

## 一、结论

阶段发布候选 `2ad957f` 可独立复现：bundle 干净克隆三坐标一致，patch 在上一候选 `fe06629b` 上干净应用且 `write-tree` 逐字节复现 `512ddd52`，bundle 记录完整历史。远端 Gitee 分支回读一致，release 未改动。

## 二、坐标（三侧对齐）

| 项 | 值 |
| --- | --- |
| 共同基线 | `960e6f292dce1a859809449ae18fb43b57927192` / Tree `33ad7eb3` |
| K 后端候选 | `3ea0aa70e17bce468cc5d88e1b225425a8ae76fa` / Tree `0d8809d3` |
| 前候选（K+E 集成） | `fe06629bba96702e053e86da8b9a5c617fd36697` / Tree `0f72a403` / Parent `d1550727` |
| **本发布候选** | **`2ad957f93d874c334a35cc9b92ca31e1a6dcbdec`** / **Tree `512ddd52570fe4ed7fd7594a6b7e320cf1a41e44`** / Parent `fe06629b` |
| Gitee 分支 | `cand/auth-v2-final-release-candidate-01` |
| 生产 release（未动） | `release/z1.4.7` = `c9e1b9fc7f64719806187af291996bfd8450c1bb` |

## 三、干净克隆同树证明

从本包 `02-完整历史bundle.bundle` 全新克隆（无本地对象缓存）：

```
HEAD   = 2ad957f93d874c334a35cc9b92ca31e1a6dcbdec
TREE   = 512ddd52570fe4ed7fd7594a6b7e320cf1a41e44
PARENT = fe06629bba96702e053e86da8b9a5c617fd36697
git bundle verify  → "The bundle records a complete history."（无缺口）
```

祖先链：`960e6f2` YES、`3ea0aa7` YES、`fe06629b` YES。

## 四、patch 精确复现

在 `fe06629b` 干净 worktree 上：

```
git apply --check 01-C增量-相对前候选.patch   → OK
git apply         01-C增量-相对前候选.patch   → OK
git add -A && git write-tree                  → 512ddd52570fe4ed7fd7594a6b7e320cf1a41e44
```

`write-tree` 与候选 Tree 逐字节一致（TREE_MATCH_OK）。另附 `01b-全量增量-相对共同基线.patch`（`960e6f2..2ad957f`，含 K+E+C 全部增量）供对照。

## 五、本轮改动范围（9 文件，+473/−40）

| 文件 | 内容 |
| --- | --- |
| `packages/companion/src/auth/password.mjs` | M1：scrypt N=2^16、参数版本化 v2、verified 上限收紧 |
| `packages/companion/src/auth/totp.mjs` | M2：otplib 13.6.0 真实库算码/验证，保留窗口与防重放 |
| `packages/companion/src/persona/correction.js` | M3：受保护事实待确认文案，不暴露不存在的在线入口 |
| `packages/companion/package.json` | 新增 otplib 13.6.0 精确锁定 |
| `package.json` / `package-lock.json` | workspace 依赖与锁文件更新 |
| `test/auth-v2/c-m1-password-params.test.mjs` | 口令参数专项 11 项 |
| `test/auth-v2/c-m1-service-rehash.test.mjs` | service 级升级专项 10 项 |
| `test/auth-v2/c-m2-totp-otplib.test.mjs` | TOTP 专项 17 项 |

未改 25 路由、四态前端契约、Passkey、备份与撤销台账实现。

## 六、逐文件哈希

见 `04-逐文件SHA256清单.txt`（清单不自指，可用 `sha256sum -c` 直接校验包内文件）。

## 七、远端回读

`GET /repos/（账号脱敏）/zyagi/branches/cand/auth-v2-final-release-candidate-01` → `sha = 2ad957f…`；`GET /commits/2ad957f…` → `tree = 512ddd52…`、`parents = [fe06629b]`；`release/z1.4.7` 仍为 `c9e1b9f`。未合并 release、未强推。

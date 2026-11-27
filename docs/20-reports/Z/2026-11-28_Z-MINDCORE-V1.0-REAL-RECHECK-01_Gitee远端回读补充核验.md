# Gitee 远端回读补充核验（解除 M0-7 BLOCKED）

**任务**：Z-MINDCORE-V1.0-REAL-RECHECK-01 补充项　**执行方**：Z（独立验证）
**时间**：2026-11-28 07:44（GMT+8）　**远端**：`gitee.com/qq1846702260/zyagi.git`
**性质**：只读回读，未 push、未建/删分支、未改远端任何对象

## 一、为什么会有这一份

首轮复验时 Gitee 凭据失效（`oauth2: Incorrect username or password`），M0-7「远端独立回读」登记为 **BLOCKED**，坐标只能以「生产只读导出包 + C 交付 bundle」双向互证。主人随后补发有效令牌，本份即以**正式远端**为第三方独立信源，把该阻断项一次结清。

## 二、方法与凭据处置

1. `git ls-remote` 取全量 ref 快照（61 行：48 分支 + 6 标签含 peeled + HEAD），落盘为 `evidence/gitee-ls-remote.txt`。
2. 以 `git fetch`（只读）把 5 个相关分支拉进空仓库镜像，逐个核 HEAD / Tree / 文件数 / 提交时间。
3. 差异、逐字比对、补丁重放全部在**远端对象**上重算，不依赖本地 bundle。
4. 令牌仅经 `https://oauth2:<token>@` 一次性使用，**未写入 `.git/config`**（fetch 后立即 `remote set-url` 还原），未进入任何证据文件；`grep` 全目录确认零泄漏。
5. 脚本 `gitee/z-gitee-readback.py` 从环境变量取令牌，输出结构化证据 `evidence/gitee-remote-readback.json`。

## 三、坐标核验（远端实测）

| 角色 | 远端分支 | HEAD | Tree | 文件 | 判定 |
|---|---|---|---:|---:|---|
| 灵核候选 | `cand/mindcore-v1.0-baseline-sync-02` | `6a5efbdee68f0ae6b6fd04e4d557992965b6a2f3` | `1d1b43fa94bbd383072a14fea54e31790b376211` | 1723 | **PASS** |
| 最新生产 | `release/z5.2.2` | `4cec7a8e551cefc5cc9f139ab37bf5234d5e96ea` | `05471a2c01538d4bc9e64ae0e510042ba656542a` | 1711 | **PASS** |
| 灵核联合 | `cand/mindcore-v1.0-joint-01` | `39bd712ff3ae0773c70fe2d14220bb82aaeec6a6` | `c3f114bcc6ba86fbfa72a71c9cb47aeee4cd95bc` | 1721 | **PASS** |
| Z 生核微修 | `cand/evocore-v1.0-microfix-04` | `b50a2057065299ba4a6baa527184dbdeb1944097` | `772b830e0d7520d61be4e72651f2d9ed9a5cd4ef` | 1710 | **PASS** |
| 收口 02 | `cand/evocore-v1.0-release-closeout-02` | `e1dea274abb6827e7603ed54d93034352dd92415` | `819f4ef44749da1cfc4ab78d4e9fa954fe2d6002` | 1709 | 参考 |

**五项与首轮记录逐位相同**，无一条漂移 —— 说明候选与生产自报告出具后未再被推送覆盖。Git 的 Tree 是内容寻址哈希，Tree 相等即**整棵树内容逐字节相等**，因此该结论强于「文件数相同」。

## 四、血缘与差异（远端重算）

| 项 | 结果 |
|---|---|
| `merge-base(候选, 生产)` | `b50a2057…` ✅ 与首轮一致，两线自 Z 微修处分叉 |
| 候选独有提交 | `6a5efbd`、`eaf8623`(merge)、`39bd712`、`2e83196`(merge)、`246a6b5`、`d2ec05b` |
| 生产独有提交 | 仅 `4cec7a8e`（发布记录） |
| 候选 vs 生产 | **27 文件 +1991 −85** ✅ |
| `microfix-04` 是否为生产祖先 | 是 ✅ |

## 五、C 侧三条声明（远端对象复现）

| 声明 | 远端结果 |
|---|---|
| `research.js` 与 `b50a2057` 逐字一致 | ✅ `git diff --quiet` 空 |
| 相对灵核联合 5 文件 +341/−19 | ✅ `5 files changed, 341 insertions(+), 19 deletions(-)` |
| 补丁重放 `write-tree = 1d1b43fa` | ✅ 精确相等 |

补丁重放用 `read-tree(39bd712)` + `apply --cached` + `write-tree`，全程只操作临时 index，未触碰工作区与远端。

## 六、版本元数据阻断（远端同样成立）

| 文件 | 生产 `4cec7a8e` | 候选 `6a5efbd` |
|---|---|---|
| `packages/web-v1/src/version.ts` | `APP_VERSION = 'Z5.2.2'` | `APP_VERSION = 'Z5.2.1'` |
| `ops/deploy-linux.sh` | `BRANCH="release/z5.2.2"` | `BRANCH="release/z5.2.1"` |
| `docs/发布说明-Z5.2.2.md` | 存在 | **已删除** |

**结论不变**：三项不是功能回归，但直接部署会把版本号降级、并把自动部署指向旧分支，属**上线阻断**。首轮 A-1 处置（以 `release/z5.2.2` 为基重做仅灵核合并）依然适用，且现在可以直接在正式远端上执行与验收。

## 七、附带发现（治理观察，不计入本轮判定）

| 编号 | 观察 | 影响 | 建议 |
|---|---|---|---|
| O-1 | 远端默认分支 `HEAD` 仍指向 `release/z1.4.7`（`c9e1b9fc`），而生产运行 `release/z5.2.2` | 任何依赖「默认分支」的克隆/CI 会取到 z1.4.7；`deploy-linux.sh` 显式写 `BRANCH` 故当前不受影响 | 把默认分支改为 `release/z5.2.2`，或在 CI 中显式指定分支 |
| O-2 | 候选分支仅被 1 个 ref 指向，未被提为任何基线 | 符合「未验收不晋级」的纪律 | 验收通过后以新分支（A-1 产物）替代，勿直接改此分支 |
| O-3 | 远端 48 个分支中 `cand/*` 占 33 个，历史候选未清理 | 检索与回溯成本上升 | 已并入的候选打 tag 后删除分支，保留可追溯性 |

## 八、边界

- **只读**：未 push、未创建或删除远端分支/标签、未改远端配置；本地镜像仅存在于沙箱 `（沙箱内临时镜像）`。
- **未部署、未重启生产、未改生产数据与配置**。
- 令牌未落盘、未进入证据包，仅在命令行一次性使用。

## 九、复现命令

```bash
export GITEE_TOKEN=<令牌>            # 不写入文件
python3 gitee/z-gitee-readback.py    # 输出 evidence/gitee-remote-readback.json
# 单点复核
git ls-remote https://oauth2:$GITEE_TOKEN@gitee.com/qq1846702260/zyagi.git \
  refs/heads/cand/mindcore-v1.0-baseline-sync-02 refs/heads/release/z5.2.2
```

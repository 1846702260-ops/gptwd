# GitHub 协同仓接入与 M3 七项回环（Z 侧）

- 执行时间：2026-11-28 10:08–10:45 CST
- 依据：`zhiyao-workspace/00-governance/C/M13-04-V1.5-最终规则.md`、`05-acceptance/TEMPLATES/七侧协同验证清单.md`
- 结论：**七项 6 通过 + 1 项单向**；生产正常；发布判定不变（CONDITIONAL PASS）

## 一、七项结果

| # | 检查项 | 结果 | 证据 |
|---|---|---|---|
| 1 | 私有仓库读取 | PASS（方式偏差） | SSH `Permission denied (publickey)` → 改 HTTPS+PAT 成功 |
| 2 | 独立测试分支提交 | PASS | `test/z-m3-loop-01 @ fac19fc3`，仅 `z-side/`，发布分支未动 |
| 3 | 任务书获取 | PASS（路径偏差） | `02-tasks/` 只有 `C/`；Z 任务书在 `03-reports/Z/` |
| 4 | 报告与交付上传 | PASS | `03-reports/Z/` ×3、`04-deliveries/Z/…/` ×13（含 READY.json + SHA256SUMS） |
| 5 | 验收反馈 | PASS | `05-acceptance/Z/` ×2 |
| 6 | 候选获取与复现 | PASS | GitHub clone：`0e508191 / ba28697 / 1729`，与 Gitee 逐位一致 |
| 7 | 跨侧协同回环 | PARTIAL | Z→C 已交付 + Issue #1；C→Z 待回读 |

协同仓提交：`1506538`（18 文件单 commit）+ `1f4f349`（项数更正）。
回环 Issue：https://github.com/<WS_REPO>/issues/1

## 二、生产服务澄清（答 C 的 inactive）

`zy-companion` **不存在**；`is-active` 返回 `inactive` 且退出码 4 即「no such unit」，不是停机。真实单元 `zhiyao-companion.service` 为 active/running，`:3111`，`/health` 200，公网 PWA 200。详见《生产服务状态核实.md》。

## 三、Z 侧登记偏差

| 编号 | 内容 | 建议 |
|---|---|---|
| Z-D1 | 使用 classic `repo` scope PAT，非 fine-grained 最小权限 | 主人生成 Z 专用 fine-grained token（Contents R/W，仅两仓，分支 `test/z-*`） |
| Z-D2 | 沙箱无 GitHub SSH 私钥 | 保持 HTTPS；如需 SSH 由主人决定是否注册 |
| Z-D3 | `02-tasks/Z/` 缺失 | C 补齐并归位任务书 |
| Z-D4 | Issues 原为空，无「任务领取」载体 | 已建 #1，后续按模板派发 |
| Z-D5 | 跨侧回环单向 | 待 C 回评 |

## 四、技术要点

`git` 默认 HTTP/2 在本沙箱与生产两侧**都会**在 60–130 s 后挂起失败，表现为 `Failed to connect to github.com port 443`。这不是仓库不可达：

```bash
# 判定
curl -sS --http1.1 -m 10 -o /dev/null -w '%{http_code}\n' https://github.com   # 200
# 修复
git -c http.version=HTTP/1.1 clone <repo>
```

## 五、边界

未修改 `release/z5.2.2`、`main`、任何 `cand/*`；未部署、未重启生产、未改生产配置与远端；未上传凭据、私人聊天、记忆数据库；未为测试新增权限。

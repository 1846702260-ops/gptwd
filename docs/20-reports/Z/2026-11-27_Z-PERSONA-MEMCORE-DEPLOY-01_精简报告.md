# Z-PERSONA-MEMCORE-DEPLOY-01 受控上线 · 精简报告

2026-11-27 21:10｜Z｜**结论：上线成功，无回滚**

## 坐标
生产 HEAD `e61596ef` / Tree `85d94d90` / `release/z5.2.0`，工作树干净；Gitee 同名分支已快进 `136aba5b..e61596ef`，`release/z1.4.7` 仍 `c9e1b9f`。前端 build-info：Z5.2.0 / e61596ef / 85d94d90 / dirty=false。回滚点 `136aba5b`（Tree `d02b9dde`），旧产物在 `dist.old`。

## 执行
M0 只读核查通过：memv2、compose、`ownerIds`、`web.ownerId`、QQ 主人映射一致；旧卡 `43980c96`。备份 `zyagi-20261127T210718.tar.gz`（35.0M，2,449 文件，SHA256 `8e35e45a…`，NAS 回读 `verified=true`，RPO=0）。按 `deploy-linux.sh --redeploy` 发布：dist.new 构建 → 切换 → 门禁 **9/9** → 重启；21:10:57 启动，`NRestarts=0`，`/health=ok`，`/web/=200`。

## 验收
运行日志：`人格系统 chenzhiyao-v1.0 · 3405 字 · sha256:9e8e93cd… · 源 d14298b4…`。文件 8,663B(LF)，`git hash-object=386ee05f` 与 HEAD 一致；任务书 `379089cc` 为同卡 CRLF 表示（8,757B），内容等价。AUTH V2：`owner=web_b292…`、`totp=available`、`restoreLocked=false`；未认证 persona/mem1/sessions 均 401；manifest、build-info 200。记忆 P1.5 与 compose 已装配；部署后隐私两场景×两模型复跑 **4/4 PASS**，仅用 `hy4flash-free` / `deepseek-v4.1-flash`。

## 数据
`owner` 548 条、`web_b292` 106 条，部署前后 SHA256 一致（`44f9c42e…` / `ba370e15…`）；config、auth 哈希未变；未迁移/删除。备份 cron 保留，自动部署仍暂停。

## 风险
B：人格卡哈希需统一用 git blob 口径；preflight 将 0 字节可选 env 文件判阻断（11-26 起存在，未改）。既有接受项不变；`pet.token` 为空、OneBot 重连为既存。需主人操作：无；手机真机待自测。

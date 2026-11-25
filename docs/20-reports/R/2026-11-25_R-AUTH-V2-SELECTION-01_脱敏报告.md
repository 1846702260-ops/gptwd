# R-AUTH-V2-SELECTION-01｜精简报告

**任务编号**：R-AUTH-V2-SELECTION-01（V1.1，含 Passkey 正式增量）
**执行侧**：R｜**时间**：2026-11-25 22:36–23:35（北京时间）
**基线**：Gitee `qq1846702260/zyagi`，分支 `cand/s3-governance-foundation-01`，HEAD `960e6f2`，tree `33ad7eb3`（现场核对一致）

## 一、执行结论

**不整体引入 Better Auth，采用组件化方案**：保留知遥现有身份/会话底座，密码沿用 Node scrypt（参数版本化升级），TOTP 用 otplib 13.6.0，Passkey 用 @simplewebauthn/server 14.1.0 + browser 14.0.0，全部 MIT、版本固定、依赖锁定。

Better Auth 1.7.28 的功能面是**实测可用**的：注册关闭、密码登录、TOTP 开启与校验、备用码一次性、会话撤销、Passkey 注册/免密登录/撤销全部跑通（Passkey 13/13）。否决它的理由不是功能不足，而是与任务书预设的降级条件正面相撞：

1. **双认证真源**。它自带 user/account/session 表与自己的会话 Cookie，与知遥 `account.json`（scrypt）+`identities.json`（令牌只存 sha256）+`zy_web`+kind（owner/user/service）并存即第二套权威。要合并就得重写 `ownerId` 继承、QQ 身份统一、service scopes 与撤销墓碑——这不是「接入一个库」，是换认证底座。
2. **体量与持久层**。运行时 59 包、43 MB（含 zod/jose/kysely 与 5 个适配器），且必须有数据库适配器；知遥真源是 JSON 文件。
3. **默认配置不安全，且要靠人记得改**。实测 Passkey 选项默认 `userVerification=preferred`，不满足任务书「必须 required 且服务端验 UV」；TOTP **无重放闸**，同一码同窗二次提交仍返回 200。

## 二、关键实测发现

- 现有体系已具备「口令门 + 单主人账户 + 服务端身份表 + 可撤销会话」，认证 V2 是补第二因素，不需要换底座。
- 密码哈希现在是 `scryptSync(pass,salt,32)` 默认参数（N=2^14，实测 32 ms/16 MB），且 account.json **没有参数字段**。建议改为版本化记录，登录成功即原地升级到 N=2^16（128 ms/64 MB）。
- TOTP 库只负责算码，**重放必须自己拦**。组件 PoC 加了 `lastStep` 闸后，同窗重放返回 401 replay（框架路径此项失败）。
- Passkey 的生死线是两个常量：`RP_ID=zhiyao.duckdns.org`（不带端口）、`ORIGIN=https://zhiyao.duckdns.org:18080`（**必须带端口**）。nginx 已 `X-Forwarded-Proto $scheme` 且保留 `Host`，链路可用；域名/证书不要动。
- 攻击面压测全部通过：伪造 origin（400）、RP 错配（浏览器 SecurityError）、UV 缺失（NotAllowedError）、断言重放（400）、并发消耗（一次 200 一次 400）、他人凭证（400）、撤销后登录（401）。

## 三、交付

WorkBuddy R 侧资料库批次目录（内网，链接不公开），含：一页选型裁决、源码路径与版本/许可/依赖/风险表、给 K 的接口契约、接收与测试指南、证据与日志、可运行 PoC 包（48 KB，两套实现 + 三套用例，逐文件 SHA256，整包 SHA256 `2c08a7c3…71fc3b`）。

## 四、边界

未读真实密钥与私人记忆；未改 `<隔离目录> Safari 与主屏幕 PWA 未真机验证**，交 Z 授权后验收，报告不写「已支持」。

## 五、待裁决

1. 是否确认组件路径（不整体引入 Better Auth）。
2. scrypt 取 N=2^16 还是 2^17（后者 259 ms/128 MB，3.6 GB 机器并发两登录即 256 MB 峰值）。
3. 离线忘密恢复是否采用「32 字节 recovery key + 本机 CLI + root 二次确认」，明确不设任何在线找回入口。

## 六、建议下一步

K 先按契约做「身份盘点 + 接口设计」，不急于改代码；C 在 K 交付后做独立验收（重点复测 TOTP 重放闸与 Passkey 的 origin/UV 服务端校验）；Z 只在获得授权后做真机与生产验证。R 不自行开启下一阶段。

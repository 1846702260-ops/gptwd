# GitHub 正式远端切换与 Gitee 回退方案（Z 侧）

- 版本：v1（2026-11-28）
- 状态：**方案就绪，未执行**。生产远端仍指 Gitee，自动部署仍暂停，等主人单独授权
- 适用：Z5.2.3 发布后执行 M4 切换

## 一、实测依据（不是推测）

| 测试 | 结果 |
|---|---|
| `curl --http1.1 https://github.com`（生产） | **200**，握手 0.068 s |
| `curl https://api.github.com`（生产，绕代理） | **200**，0.115 s |
| `git ls-remote https://github.com/...`（默认 HTTP/2） | 挂起 **60 s 超时**，2 次复现 |
| 同上 + `-c http.version=HTTP/1.1` | **立即成功** |
| 沙箱侧同日复现 | 相同：默认 HTTP/2 时 `Failed to connect ... 132 s`，HTTP/1.1 后正常 |
| 生产环境变量 `https_proxy` | `http://127.0.0.1:2080`，**无监听**（死代理） |
| `deploy-linux.sh` 现有处理 | `git_fetch()` 已 `env -u` 绕代理；**未设** HTTP 版本 |

结论：两个独立缺陷叠加——死代理 + HTTP/2 在 git 传输层挂起。前者已有补丁，后者本轮补。

## 二、HTTP/1.1 配置方案（仓库级，不扩大）

**采用仓库级**，不写 `--global`，不动 `/etc`，不影响系统其他 git 使用：

```bash
cd <REPO>
git config --local http.version HTTP/1.1
```

等价命令级（临时验证用）：

```bash
git -c http.version=HTTP/1.1 ls-remote <repo>
```

理由：

- 全局配置会改变机器上所有 git 行为，属"扩大到全局网络配置"，任务明确禁止；
- 仓库级只作用于 `<REPO>`，随仓库目录走，回退时无需清理；
- `deploy-linux.sh` 的 `git_fetch()` 已封装，追加 `-c http.version=HTTP/1.1` 即可双重保险（配置 + 命令级）。

**验证**（切换前必做）：

```bash
cd <REPO>
git config --local --get http.version        # 期望 HTTP/1.1
timeout 30 git ls-remote origin | head -3    # 期望 30 s 内返回
```

## 三、凭据分离（三枚，互不复用）

| 用途 | 类型 | 存放 | 权限 | 谁能读 |
|---|---|---|---|---|
| **生产拉取（只读）** | fine-grained PAT，仅 `zhiyao-agi`，`Contents: Read` | 生产 `<CRED_FILE>`（600，root） | 只读 | 仅生产 root |
| **发布写入** | fine-grained PAT，`Contents: Read/Write`，分支限 `release/*` 与 `cand/z-*` | Z 侧执行环境（不落生产、不落仓库） | 写 | 仅 Z |
| **协同仓读写** | fine-grained PAT，`zhiyao-workspace` `Contents: Read/Write` | Z 侧 | 读写 | 仅 Z |

现状偏差（如实登记）：本侧目前只有一枚 classic `repo` scope 令牌，同时承担上述三职；已登记 Z-D1，待主人核发 fine-grained 后替换。

**生产凭据文件格式**（`credential.helper store` 现状沿用）：

```
https://x-access-token:<READ_TOKEN>@github.com
```

写入后立即校验，且**不打印内容**：

```bash
test -r <CRED_FILE> && stat -c '%a %U' <CRED_FILE>   # 期望 600 root
git -C <REPO> -c credential.helper=store --file=<CRED_FILE> \
    -c http.version=HTTP/1.1 ls-remote https://github.com/<AGI_REPO>.git | head -3
```

## 四、切换前检查清单（全部满足才可执行）

| # | 检查 | 通过标准 |
|---|---|---|
| 1 | 发布已上线且稳定 | `/health` 200、无异常重启 |
| 2 | 备份可回滚 | 最新包 sha256 校验通过、NAS `verified=true` |
| 3 | 生产拉取凭据可用 | 上节 `ls-remote` 30 s 内返回 |
| 4 | HTTP/1.1 已配 | `git config --local --get http.version` = HTTP/1.1 |
| 5 | Gitee 回退点 | `release/z5.2.2` 与 `release/z5.2.3` 在 Gitee 均存在 |
| 6 | 自动部署 | 仍暂停（crontab 该行保持注释） |
| 7 | 磁盘 | ≥ 20% 余量 |

## 五、切换步骤（M4，待授权；每步可单独回退）

```bash
# 0) 记录当前状态
cd <REPO> && git rev-parse HEAD && git remote get-url origin > /tmp/origin.bak

# 1) 只读验证 GitHub 侧坐标（不改任何东西）
git -c http.version=HTTP/1.1 ls-remote https://github.com/<AGI_REPO>.git | head -5

# 2) 换远端（先只改 fetch，push 仍留 Gitee，避免误推）
git remote set-url origin https://github.com/<AGI_REPO>.git

# 3) 拉取并核对 HEAD/Tree 与发布坐标一致
git -c http.version=HTTP/1.1 fetch origin release/z5.2.3
git rev-parse FETCH_HEAD && git rev-parse FETCH_HEAD^{tree}

# 4) 切分支（不重置工作区，避免覆盖未跟踪运行时文件）
git checkout -B release/z5.2.3 FETCH_HEAD

# 5) 重建前端并验活
npm --prefix packages/web-v1 run build
systemctl restart zhiyao-companion
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:3111/health   # 期望 200

# 6) 观察 10 分钟：无异常重启、忆核/生核可用
```

**关键约束**：第 2 步后**不执行任何 `git push`**。生产是只读消费者，写入只从 Z 侧候选分支经 PR 进入。

## 六、回退（任一环节失败，5 分钟内可恢复）

```bash
cd <REPO>
git remote set-url origin https://gitee.com/<GITEE_REPO>.git
git config --local --unset http.version          # GitHub 专属配置，回退时清掉
git fetch origin release/z5.2.3 2>/dev/null || git fetch origin release/z5.2.2
git checkout release/z5.2.2                       # 回到 Z5.2.2
git config --local --get remote.origin.url        # 确认已回 Gitee
systemctl restart zhiyao-companion
curl -s -o /dev/null -w '%{http_code}\n' http://127.0.0.1:3111/health
```

若代码库本身损坏：从 `<BACKUP_DIR>/zyagi-20261128T065139.tar.gz` 解包恢复，前端用 `dist.old`，再起服务。

## 七、风险与缓解

| 风险 | 缓解 |
|---|---|
| HTTP/2 再次挂起被误判"仓库不可达" | 仓库级 + 命令级双保险；`deploy-linux.sh` 内 `git_fetch` 加 `-c` |
| 死代理复活或新代理 | `git_fetch()` 已 `env -u`；新增代理需同步更新该函数 |
| 只读 token 过期 | 过期表现为 401/403（不是超时），日志可区分；轮换周期登记 |
| 误推生产 | 生产 remote 只用于 fetch；Z 侧推送走独立凭据与分支约定 |
| 自动部署被无意恢复 | crontab 该行保持注释；恢复需主人单独授权 |
| 磁盘 81% | 发布前清理临时产物至 ≥20% 余量 |

## 八、本轮未做（边界）

- 未生成、未写入任何 GitHub 凭据到生产；
- 未修改 `remote.origin.url`（仍 Gitee）；
- 未设全局 git 配置；
- 未部署、未重启、未恢复自动部署；
- 未在 Gitee 打新标签、未删任何历史备份。

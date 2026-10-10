# Git 与 CI 操作手册

> 通用规范：只讲 git / CI 的操作方式与踩过的坑，不绑定任何具体仓库。默认分支 `main`。

## 1. 提交（commit）约定
- 前缀：`feat:` / `fix:` / `test:` / `docs:` / `chore:`，简洁中文。
- **含中文的提交信息必须用 `-F` 文件**，且**无 BOM 写入**（PowerShell 5.1 的 `Set-Content -Encoding UTF8` 会加 BOM，导致 `git log` 中文显示成乱码 `锟?`；提交正文没坏但难读）：
```powershell
[IO.File]::WriteAllText("$PWD\.git\CM", "feat(xxx): 中文说明", (New-Object Text.UTF8Encoding $false))
git commit -q -F .git\CM ; Remove-Item .git\CM
```

## 2. 推送（push）约定：本地频繁 commit，阶段末批量 push
- 推送会触发 GitHub Actions CI（一次通常 4–7 分钟）。**禁止为小改动（补一段文档、改错别字、单个小函数）单独推送**——只本地 commit，攒到一个里程碑齐备后再推。
- 只推 `main`；需要试验就用分支，别把半成品直接落到 `main`。

## 3. 推前自证（全绿才推）
在仓库根目录执行**该仓库自己声明的**自证流程——具体命令以仓库 README / CI workflow 里写的为准，通常要覆盖三件事：
1. **全量编译 / 构建通过**；
2. **单元测试通过**；
3. **入口可加载**（最外层模块能启动或能通过加载检查）。

任何一步红了都不推。

## 4. 推前必查：敏感/大文件没被 stage
```powershell
git add -A
git diff --cached --name-only
```
逐条确认下面这些都没被 stage（它们都应已写进 `.gitignore`）：
- `.env` 及任何密钥 / 口令 / 证书 / 凭据文件；
- 日志目录与 `*.log`；
- 构建产物、依赖缓存、离线安装包（`*.whl`、`*.jar` 等 vendor 目录）；
- 模型权重、数据集等大二进制文件。

**`.env` 含密钥与口令，绝不进 git**；任何密钥、服务器口令都不入库。

## 5. 看 CI（gh CLI）
```powershell
# 取最新一次 CI run id
$rid = gh run list -R <owner/repo> --workflow CI --limit 1 --json databaseId -q '.[0].databaseId'
# 阻塞式盯到结束（失败返回非零）
gh run watch $rid -R <owner/repo> --exit-status
# 看总结 + 各 job
gh run view $rid -R <owner/repo> --json conclusion,jobs
```
**CI 里的每个 job 都必须 `success`**，有一个红就算失败。job 怎么划分由各仓库的 workflow 决定。

## 6. 改 CI workflow 前必读
- **runner 的网络环境与本地不同**：本地能用的镜像源 / 内网端点，GitHub runner 未必通。这类差异要在 job 里**显式覆盖环境变量**，不要把本地专用配置写进默认配置、也不要依赖它。
- **CI 用的外部依赖（数据库 / 缓存 / 队列等）用 service container 起**，并在 job 里显式给连接参数（host / port / 账号 / 库名），不要依赖 runner 上恰好有什么。
- **密钥只走 GitHub Secrets**，workflow 里用 `${{ secrets.XXX }}` 引用，绝不硬编码。
- workflow 改动本身也算改动：改完先在分支推一次看 CI，别直接推 `main`。

## 7. CN 网络拉 GitHub 失败（GnuTLS）
`GnuTLS recv error (-110)` 时用 HTTP/1.1 绕过：
```bash
git -c http.version=HTTP/1.1 fetch --depth 1 origin main
```

## 8. 常用一次性命令
```powershell
git log --oneline -8
git status -s
git push origin main 2>&1 | Select-Object -Last 1
```

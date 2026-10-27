# 这个 fork 怎么用

本仓库是 [c1overzz/dsh-bili-scroll](https://github.com/c1overzz/dsh-bili-scroll) 的 fork。
目的有两个：**跟上游保持同步**，以及**保留自己的一套修复**。

## 分支怎么分工

| 分支 | 内容 | 规则 |
|---|---|---|
| `main` | **只跟上游**，不掺任何本地改动 | 永远等于 `upstream/main`，可以随时快进 |
| `mine` | 我的修复（每个 bug 一个提交） | 从 `main` 长出来，上游更新后 rebase 上去 |

这样分工的好处：`main` 永远是干净的上游镜像，`git pull upstream main --ff-only` 不会冲突；
我的改动单独放 `mine`，上游哪天自己修了同一条，直接 `git rebase -i` 丢掉对应提交即可。

## 第一次配置（只做一次）

```bash
git remote add upstream https://github.com/c1overzz/dsh-bili-scroll.git
git fetch upstream
```

确认两个 remote 都在：

```bash
git remote -v
# origin    https://github.com/Sakiko15/dsh-bili-scroll.git   ← 我的 fork
# upstream  https://github.com/c1overzz/dsh-bili-scroll.git   ← 原作者
```

## 日常：上游发新版本了

```bash
# 1. 拉上游
git fetch upstream

# 2. main 快进到上游（必须 ff-only，不允许产生合并提交）
git checkout main
git pull upstream main --ff-only
git push origin main

# 3. 把我的修复挪到新上游之上
git checkout mine
git rebase upstream/main

# 4. 如果 rebase 有冲突，逐个解决后
git add <冲突文件>
git rebase --continue

# 5. 推送（rebase 改写了历史，需要强推）
git push --force-with-lease origin mine
```

> 用 `--force-with-lease` 而不是 `--force`：万一远端 `mine` 上有别的机器推的东西，它会拒绝覆盖。

## `mine` 上有哪些修复

按提交顺序（越靠上越新）：

| 提交 | 修的问题 | 上游状态 |
|---|---|---|
| `fix(dm): pre-warm the recipient UID on every entry point` | `warmDmUid()` 只在 HTTP 入口被调用，`bili_run` 工具和峰谷定时器都不调 ⇒ 私信永远发不出去。附带 4 处小修（`asrInstalled` 恒真、日志报错 profile、空锁噪音、私信提示刷屏） | [#122](https://github.com/c1overzz/dsh-bili-scroll/issues/122) open |
| `fix(watched): read "为什么值得" and "标题" without requiring bold markers` | 卡片墙两栏永远是空 / BV 号 | 未记录 |
| `fix(report): stop deleting the report when "## 投递" appears mid-file` | **整份日报被删光** + 配额账本 `state.json` 永不生成 | 未记录 |
| `fix(verify): stop truncating the report at the first "## 评论复核"` | 复核段用 `indexOf` 切，会删掉正文 | [#135](https://github.com/c1overzz/dsh-bili-scroll/issues/135) 相关 |
| `fix(diagnose): recognise ACCOUNT_SIGN_IN_REQUIRED, tighten the 401 match` | 未登录错误认不出；`\b401\b` 误判 | 未记录 |
| `fix(profile): drop unknown patch ids tool-ralph / skill-badge` | 模板里两个不存在的条目，每次启动刷警告 | [#136](https://github.com/c1overzz/dsh-bili-scroll/issues/136) 相关 |

## 上游修好了某一条怎么办

比如上游自己修了「报告被删光」，那 `mine` 上对应提交就不需要了：

```bash
git checkout mine
git rebase -i upstream/main
# 把那一行改成 drop（或删掉整行），保存退出
git push --force-with-lease origin mine
```

## 给上游提 PR

从 `mine` 上挑单个提交开分支即可，不必把 6 个一起提：

```bash
git checkout -b pr/report-truncation upstream/main
git cherry-pick afd2051          # 换成要提的那个提交
git push origin pr/report-truncation
```

然后在 GitHub 上选 base = `c1overzz/dsh-bili-scroll:main`、compare = `Sakiko15:pr/report-truncation`。

## 装到本地 DSH 用

### 方案 A：从 git 装（会撞 pnpm 的 allowBuilds）

```bash
dsh plugin --profile web add github:Sakiko15/dsh-bili-scroll#mine
```

⚠️ 这条路**大概率第一次会失败**：git 源的包要靠 `prepare` 脚本构建，pnpm 默认拦下所有构建脚本，
会报 `Ignored build scripts` 并打印一个形如 `dsh-bili-scroll@https://github.com/...#<hash>` 的 key。
把那个 key 原样加进 `~/.dsh/profiles/web/pnpm-workspace.yaml`：

```yaml
allowBuilds:
  - "dsh-bili-scroll@https://github.com/Sakiko15/dsh-bili-scroll.git#<hash>"
```

再重跑一次 `add`。key 必须**逐字照抄** pnpm 打印出来的那个，不能自己猜。

> 注意装 `#mine` 分支而不是默认分支 —— `main` 是纯上游，没有这些修复。

### 方案 B：本地目录装（改代码时更顺手）

```bash
dsh plugin --profile web add file:D:\dsh_prj\prj1\ts1\dsh-bili-scroll
```

不经过 git 构建，改完源码重启 dsh 就生效。

### 方案 C：继续用 npm 包 + pnpm patch

如果只是想在现有安装上叠这些修复、不换安装来源，用 `pnpm patch` 把改动做成补丁：

```bash
cd ~/.dsh/profiles/web
pnpm patch dsh-bili-scroll@2.2.2
# 在打印出的临时目录里改文件，然后：
pnpm patch-commit "<临时目录>"
```

补丁会写进 `pnpm-workspace.yaml` 的 `patchedDependencies`，升级插件时会被覆盖、需要重打。

> ⚠️ 方案 B 和 C **不要同时用** —— 会双重打补丁。切到 B 之前先把 `patchedDependencies`
> 里 `dsh-bili-scroll` 那行删掉。

---

## 改了 fork 源码之后，怎么让它生效

`file:` 依赖是**复制**模式，不是符号链接。实测（pnpm 11.21.0）：

| 做法 | 结果 |
|---|---|
| `pnpm install` | ❌ 不更新（pnpm 认为依赖没变） |
| `pnpm install --force` | ❌ 不更新 |
| **删掉副本再装** | ✅ 有效 |

所以改完源码的完整流程是：

```bash
# 1. 在 D:\dsh_prj\prj1\ts1\dsh-bili-scroll 里改代码
# 2. 删掉副本
rm -rf ~/.dsh/profiles/web/node_modules/dsh-bili-scroll
#    Windows: Remove-Item -Recurse -Force "$env:USERPROFILE\.dsh\profiles\web\node_modules\dsh-bili-scroll"
# 3. 重装
cd ~/.dsh/profiles/web && pnpm install
# 4. 重启 dsh
```

想省掉 2~3 步，可以把依赖改成 `link:`（符号链接，改完只需重启）：

```json
"dsh-bili-scroll": "link:D:\\dsh_prj\\prj1\\ts1\\dsh-bili-scroll"
```

代价是源码目录不能删、不能挪，否则插件直接挂。

---

## ⚠️ 用 PowerShell 改 JSON 配置文件会踩 BOM 的坑

这台机器上遇到过：用 `ConvertTo-Json | Set-Content -Encoding UTF8` 改
`~/.dsh/bili-duty/config.json`，Windows PowerShell 5 的 `-Encoding UTF8` 会在开头写
`EF BB BF`，于是 `JSON.parse` 直接抛错，整份配置被当成读不到 ⇒ **所有设置静默退回默认值**
（`pythonPath` 变 `'python'`、`headlessProfile` 变 `'headless'`），现象只有"怎么改都不生效"。

`mine` 分支已经修了（剥 BOM + 解析失败时记日志），但**改这些文件时仍然建议**：

```powershell
# 用这个写法（无 BOM）
[IO.File]::WriteAllText($path, $json, [Text.UTF8Encoding]::new($false))
```

或者干脆用插件自己的工具改配置，别手改文件：

```
bili_config({ "perDay": 5 })
```

## 验证

改完之后跑一遍仓库自带的检查：

```bash
node --test          # 单元测试
node selfcheck.mjs   # 插件半端自检
node --check index.js
```

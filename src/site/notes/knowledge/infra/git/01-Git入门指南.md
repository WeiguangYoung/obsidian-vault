---
{"dg-publish":true,"permalink":"/knowledge/infra/git/01-Git入门指南/","title":"Git 入门指南","tags":["git","版本控制","分支","团队协作","入门"],"dg-note-properties":{"title":"Git 入门指南","date":"2026-11-01","tags":["git","版本控制","分支","团队协作","入门"]}}
---


# Git 入门指南

> 面向「从零上手到能在团队里正常干活」的一条线。
> 提交与分支的**团队约定**见 [[knowledge/infra/git/02-Git提交与分支规范\|Git 提交与分支规范]]。

## 一、Git 到底在管什么

Git 是一个**分布式版本控制系统**（DVCS）。理解它只要抓住三件事：

1. **快照，不是差异** —— 每次 commit 保存的是「整个项目的快照指针」，不是一份 diff 文件
2. **分布式** —— 每个人手里都有完整仓库（含全部历史），离线也能提交
3. **不可变的对象** —— commit 一旦生成，哈希就固定了。你用 `rebase`/`amend` 改出来的其实是**新的** commit

## 二、四个区与数据流（最重要的一张图）

```
 工作区            暂存区            本地仓库              远程仓库
Working Dir  →   Staging Area  →   Local Repo    →     Remote
             add             commit           push
             ←──────────────────────────────────────────
                    checkout/restore        ←     fetch/pull
```

| 区域 | 物理位置 | 作用 |
|:----|:----|:----|
| 工作区 | 你看到的文件目录 | 正在编辑的内容 |
| 暂存区（index） | `.git/index` | 下一次 commit 的内容清单 |
| 本地仓库 | `.git/objects` | 所有 commit 历史 |
| 远程仓库 | GitHub / GitLab / Gitea | 团队共享的那一份 |

**牢记**：`git add` 进暂存区，`git commit` 进本地仓库，`git push` 才到远程。**没 push 的东西别人看不见。**

## 三、第一次配置

```bash
git config --global user.name  "Your Name"
git config --global user.email "you@example.com"

git config --global init.defaultBranch main
git config --global core.autocrlf input     # Linux/macOS 保持 LF

# 查看当前生效配置
git config --list --show-origin
```

> 公司仓库和私人仓库身份不同时，**不要**用 `--global`，在仓库里单独 `git config` 即可。

## 四、日常最小工作流

```bash
# 1. 拿代码
git clone git@github.com:org/repo.git
cd repo

# 2. 开发前先同步
git switch main
git pull --rebase

# 3. 开分支干活（永远不在 main 上直接改）
git switch -c feature/login

# 4. 改文件，看看改了什么
git status
git diff

# 5. 暂存 + 提交
git add .
git commit -m "feat(login): 增加短信验证码登录"

# 6. 推送并建 PR
git push -u origin feature/login
```

**`-u` 的作用**：把本地分支和远程分支绑起来，之后直接 `git push` / `git pull` 就行，不用每次写 `origin xxx`。

## 五、核心命令速查

### 5.1 查看

```bash
git status                 # 谁改了、谁暂存了
git diff                   # 工作区 vs 暂存区
git diff --staged          # 暂存区 vs 最新 commit
git log --oneline --graph --all -20
git log -p file.c          # 某个文件的逐次改动
git show <commit>          # 看某次提交的完整内容
git blame file.c           # 每一行是谁、哪次提交改的
```

### 5.2 撤销（按「危险程度」排）

| 场景 | 命令 | 说明 |
|:----|:----|:----|
| 改坏了还没 add | `git restore file.c` | 丢弃工作区改动 |
| add 错了 | `git restore --staged file.c` | 取消暂存，改动还在 |
| 想改最后一次 commit | `git commit --amend` | **会改写哈希，已 push 需 force** |
| 本地没 push 想退回 | `git reset --soft HEAD~1` | 退回 commit，改动留在暂存区 |
| | `git reset --mixed HEAD~1` | 退回 commit，改动留在工作区（默认） |
| | `git reset --hard HEAD~1` | **连改动一起删**，危险 |
| 已经 push 了想撤销 | `git revert <commit>` | 生成一个反向 commit，**安全** |
| 误删了分支/大改 | `git reflog` + `git reset --hard <hash>` | 救命稻草，保留 30 天 |

> **铁律**：公共分支（main/develop）上**永远用 `revert` 不用 `reset`**。reset 会改写历史，别人拉下来就乱了。

### 5.3 分支

```bash
git branch                 # 本地分支列表
git branch -a              # 含远程
git switch -c hotfix/x     # 新建并切换（推荐，比 checkout -b 语义清晰）
git branch -d old          # 删除已合并分支
git branch -D old          # 强制删除（未合并）
git push origin --delete old
```

## 六、合并：merge vs rebase

```bash
# merge：保留分叉，生成一个合并 commit（历史真实但线条乱）
git switch main
git merge feature/login

# rebase：把本分支的提交「重放」到目标分支顶端（历史线性、干净）
git switch feature/login
git rebase main
```

| | merge | rebase |
|:----|:----|:----|
| 历史 | 保留分叉与合并点 | 线性、干净 |
| 哈希 | 不变 | **全部改写** |
| 适用 | 公共分支之间（main ← develop） | 个人分支同步主干 |
| 禁忌 | — | **不要 rebase 已推送的公共分支** |

**团队实践**：本地分支 rebase 追主干，合并进主干用 merge（或 squash merge）。

### 6.1 冲突怎么解

```bash
git merge feature/login
# 提示 CONFLICT ...

git status                 # 看哪些文件冲突
# 打开文件，找 <<<<<<< ======= >>>>>>> 三段，手动改成最终想要的内容
git add 冲突文件
git commit                 # merge 冲突用 commit 收尾
# rebase 冲突则是 git add 后 git rebase --continue
```

想放弃：`git merge --abort` / `git rebase --abort`。

## 七、远程协作

```bash
git remote -v                        # 看远程地址
git remote add upstream <url>        # 加第二个远程（fork 场景）
git fetch --all --prune              # 只拉取，不动工作区（最安全）
git pull                             # = fetch + merge
git pull --rebase                    # = fetch + rebase（推荐）
git push origin feature/x
```

**fork 工作流**（贡献开源项目时）：

```bash
git clone git@github.com:me/project.git
cd project
git remote add upstream git@github.com:origin/project.git
git fetch upstream
git switch -c fix/typo upstream/main
# ...改完提交...
git push -u origin fix/typo
# 在网页上向 upstream 提 PR
```

## 八、三样必须会的「后悔药」

```bash
# 1. 全都要回来（误 reset / 误删分支）
git reflog                 # 列出所有 HEAD 移动记录
git reset --hard HEAD@{3}

# 2. 单个 commit 搬到别的分支
git cherry-pick <commit>

# 3. 只想看某个文件在某个版本的样子
git show <commit>:path/to/file.c > /tmp/old.c
```

## 九、`.gitignore` 与常见坑

```gitignore
# 编译产物
build/  out/  *.o  *.a  *.so  *.exe
# 依赖
node_modules/  .venv/  __pycache__/
# IDE / 系统
.idea/  .vscode/  .DS_Store
# 密钥（永远不要提交）
*.pem  *.key  .env
```

| 现象 | 原因 | 解法 |
|:----|:----|:----|
| 文件明明在 `.gitignore` 里还是被跟踪 | 之前已经 `add` 过 | `git rm --cached 文件` 后再提交 |
| `push` 被拒（rejected） | 远程有新提交 | `git pull --rebase` 后再 push |
| 提交了密码/密钥 | — | **立刻作废该密钥**，再 `git filter-repo` 清历史；光删 commit 没用 |
| `LF/CRLF` 全文件变化 | 跨系统换行符 | 配 `core.autocrlf` + `.gitattributes` |
| clone 超大仓库很慢 | 历史太长 | `git clone --depth 1`，或 `--filter=blob:none` 部分克隆 |
| 误把大文件提交了 | — | `git filter-repo --path big.bin --invert-paths` |

## 十、和 CI/CD 的关系（DevOps 视角）

Git 不只是存代码，它是**整条流水线的触发源**：

| Git 动作 | 触发 |
|:----|:----|
| push 到分支 | 触发 CI 构建（Jenkins Multibranch / GitLab CI） |
| 开 PR/MR | 触发代码扫描、单测、制品快照构建 |
| 打 tag（`v1.2.3`） | 触发正式发布、推制品库、打镜像标签 |
| 合并到 main | 触发 GitOps 同步（ArgoCD） |

所以 **commit 规范 + 分支规范 + tag 规范** 直接决定流水线能不能自动化运转 —— 这也是「Git 规范」在 DevOps 岗位里的真正价值。

## 相关文档

- [[knowledge/infra/git/02-Git提交与分支规范\|Git 提交与分支规范]]
- [[knowledge/devops/practices/02-Jenkins流水线实战\|Jenkins 流水线实战]]
- [[knowledge/devops/artifact/00-制品管理全景\|制品管理全景与工具选型]]

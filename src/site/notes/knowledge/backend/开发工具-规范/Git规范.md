---
{"dg-publish":true,"permalink":"/knowledge/backend/开发工具-规范/Git规范/","title":"Git规范","tags":["git","github","前端"],"dg-note-properties":{"category":"开发工具 / 规范","title":"Git规范","source":"csdn","created":"2023-03-17","tags":["git","github","前端"],"url":"https://blog.csdn.net/weixin_45536921/article/details/129614482"}}
---



### Commit 规范

常见的开源社区 commit message 规范：


**主流提交信息格式对比**

| 项目 | 格式 |
|---|---|
| jQuery | `Component: Short Description` |
| Ember | `[FEATURE name]` |
| JSHint | `[[TYPE]] <Short description>` |
| AngularJS | `<type>(<scope>): <subject>` |
| **Conventional Commits** | `<type>[optional scope]: <description>` |

> Angular 规范后演进为 **Conventional Commits**（社区通用标准），并被 Go 生态等广泛采用。


比如 Angular 规范：


语义化：commit message 被归为有意义的类型用来说明本次 commit 的类型。

规范化：commit message 遵循预先定义好的规范，比如格式固定、都属于某个类型，可被开发者和工具识别。


**Conventional Commits 实际提交示例**

```
commit ed55355363dad26778dfe61fe6fc0ffc521f0db6
Author: Igor Minar <igor@angular.io>
Date:   Tue Jul 30 14:36:35 2019 -0700

    fix(core): remove deprecated and defunct wtf* apis (#33949)

    These apis have been deprecated in v8, so they should stick around till v10,
    but since they are defunct we are removing them early so that they don't take up payload size.

    PR Close #33949
```

> 结构：`fix(core): <描述>` —— type=`fix`，scope=`core`，末尾附 PR 编号。


基本格式：


```
[optional scope]: 
// 空行
[optional body]
// 空行
[optional footer(s)]

```
建议使用 git commit 时不要使用 -m 选项，而 -a 进入交互界面编辑 commit message。


#### Header

只有一行，包括：type（必选）、scope（可选）和 subject（必选）。


type 分为：


Development：一般是项目管理类的变更，不会影响最终用户和生产环境的代码，比如 CI 流程、构建方式等的修改。通常可以免测发布。

Production：会影响最终的用户和生产环境的代码。一定要慎重并在提交前做好充分的测试。


**type 类型说明**

| 类型 | 类别 | 说明 |
|---|---|---|
| `feat` | Production | 新增功能 |
| `fix` | Production | Bug 修复 |
| `perf` | Production | 提高代码性能的变更 |
| `style` | Development | 代码格式类变更，如 gofmt 格式化、删除空行等 |
| `refactor` | Production | 其他代码类变更，不属于 feat/fix/perf/style，例如简化代码、重命名变量、删除冗余代码 |
| `test` | Development | 新增测试用例或更新现有测试用例 |
| `ci` | Development | 持续集成和部署相关改动，如修改 Jenkins、GitLab CI 配置或 systemd unit 文件 |
| `docs` | Development | 文档类更新，包括用户文档或开发文档 |
| `chore` | Development | 其他类型，如构建流程、依赖管理或辅助工具的变动 |


**type 分类结构**

```
        代码类                     非代码类
    ┌──────────────┐          ┌──────────────┐
    │ 新增功能 feat │          │ 更新测试 test │
    ├──────────────┤          ├──────────────┤
    │ 修复缺陷 fix  │ 预定义类型│ 部署变更 ci   │
    ├──────────────┤  ───────→ ├──────────────┤
    │ 改进性能 perf │          │ 文档变更 docs │
    ├──────────────┤          └──────────────┘
    │ 格式化 style  │
    └──────────────┘

    ┌──────────────┐               ┌──────────────┐
    │ 优化代码     │─── 其它类型 ──│ 其它变更      │
    │ refactor    │               │ chore        │
    └──────────────┘               └──────────────┘
```


scope 说明 commit 影响范围，必须是名词，因项目而异。


初期可设置粒度较大的 scope（如按组件名或功能设置），后续项目有变动或有新功能时可添加新的 scope。

不适合设置太具体的值。会导致项目有太多的 scope 难以维护，开发者也难以确定 commit 所属的 scope 导致错放，反而失去分类的意义。

subject 是 commit 的简短描述，明确指出 commit 执行的操作，以动词开头（小写）、使用现在时，不加句号。


#### Body

可分成多行，格式较自由。以动词开头、使用现在时。


必须包括修改的动机、跟上一版本相比的改动点。


```
The body is mandatory for all commits except for those of scope "docs". When the body is required it must be at least 20 characters long.
Footer

```
根据需要选择，一般格式：


```
BREAKING CHANGE: 
// 空行

// 空行
// 空行
Fixes #

```
可说明本次 commit 导致的后果。


不兼容的改动：如当前代码跟上一个版本不兼容，需要在 Footer 部分以 BREAKING CHANG: 开头，跟上不兼容改动的摘要。其他部分需要说明变动的描述、变动的理由和迁移方法：


```
BREAKING CHANGE: isolate scope bindings definition has changed and
    the inject option for the directive controller injection was removed.

    To migrate the code follow the example below:

    Before:

    scope: {
      myAttr: 'attribute',
    }

    After:

    scope: {
      myAttr: '@',
    }
    
    The removed `inject` wasn't generaly useful for directives so there should be no code using it.

```
关闭的 Issue 列表：关闭的 Bug 需要在 Footer 部分新建一行，并以 Closes 开头列出，比如：Closes #123。如果关闭多个 Issue，可以列出：Closes #123, #432, #886。


```
Change pause version value to a constant for image
   
   Closes #1137

```

#### Revert Commit

如果当前 commit 还原了先前的 commit，则应以 revert: 开头，后跟还原的 commit 的 Header。而且，在 Body 中必须写成 This reverts commit  ，其中 hash 是要还原的 commit 的 SHA 标识。


```
revert: feat(iam-apiserver): add 'Host' option

This reverts commit 079360c7cfc830ea8a6e13f4c8b8114febc9b48a.

```

#### 管理 Commit

在对项目进行修改，通过测试（修复 bug、完成 feature）后立即 commit；或约定一个提交的周期，减少本地代码丢失造成的代码丢失量。要避免 commit 太多，可在合并代码或提交 PR 时使用 git rebase -i 合并之前所有提交：建议把新的 commit 合并到主干时只保留 2~3 个 commit 记录。


使用 git rebase -i commitId 进入交互界面，可列出该 commitId 之后的所有 commit。其支持的操作（默认是 pick）：


**`git rebase -i` 交互式命令**

| 命令 | 目的 |
|---|---|
| `p, pick` | 不对该 commit 做任何处理 |
| `r, reword` | 保留该 commit，但修改提交信息 |
| `e, edit` | 保留该 commit，rebase 时暂停，允许修改这个 commit |
| `s, squash` | 保留该 commit，但与上一个 commit 合并 |
| `f, fixup` | 与 squash 相同，但不保留当前 commit 的提交信息 |
| `x, exec` | 执行其他 shell 命令 |
| `d, drop` | 删除该 commit |


其中 squash 和 fixup 可用于合并 commit（注意是处理 commitId 和 HEAD 之间的 commit，开区间）。比如临时切换到 feature 分支进行开发：


```
git log --oneline
# 7157e9e docs(docs): append test line 'update3' to README.md
# 5a26aa2 docs(docs): append test line 'update2' to README.md
# 55892fa docs(docs): append test line 'update1' to README.md
# 89651d4 docs(doc): add README.md

# ========== feature/user 开发... ==========

git log --oneline
# 4ee51d6 docs(user): update user/README.md
# 176ba5d docs(user): update user/README.md
# 5e829f8 docs(user): add README.md for user
# f40929f feat(user): add delete user function
# fc70a21 feat(user): add create user function

# 以上是切换到 feature/user 分支进行开发后提交的 commit。

# 7157e9e docs(docs): append test line 'update3' to README.md
# 5a26aa2 docs(docs): append test line 'update2' to README.md
# 55892fa docs(docs): append test line 'update1' to README.md
# 89651d4 docs(doc): add README.md

```
在 feature 分支上的 5 个 commit 要在合并到 master 分支前进行精简，可选取切换 feature 分支前的最后一个 commit 进行 rebase（比如 squash 操作）。最终 pick 前的四个 commit 都会被删除。


```
git rebase -i 7157e9e
# pick 4ee51d6 docs(user): update user/README.md
# s 176ba5d docs(user): update user/README.md
# s 5e829f8 docs(user): add README.md for user
# s f40929f feat(user): add delete user function
# s fc70a21 feat(user): add create user function
# Rebase 7157e9e..4ee51d6 onto 7157e9e (5 commands(s))

git log --oneline
# d6b17e0 feat(user): add user module with all function implements
# 7157e9e docs(docs): append test line 'update3' to README.md
# 5a26aa2 docs(docs): append test line 'update2' to README.md
# 55892fa docs(docs): append test line 'update1' to README.md
# 89651d4 docs(doc): add README.md

```
除此之外，如果有太多 commit 需要合并，可以先撤销过去的 commit，再创建一个新的（需要重新整理 commit message）：


```
git reset HEAD~3
git add .
git commit -am "feat(user): add user resource"

```

#### 修改 Commit

修改最近提交的 commit：


```
git log --oneline
git commit --amend

```
修改某次 commit：


```
git rebase -i previousCommitId

```
第二种会导致父 commit 之后所有 commit 的 commitId。可能需要 git stash 将当前分支工作状态暂存，修改完成后再进行 git stash pop 恢复。


### 分支管理


**四种 Git 工作流对比**

| 工作流 | 优点 | 缺点 | 使用场景 |
|---|---|---|---|
| **集中式工作流** | 上手最简单 | 代码管理较混乱，容易出问题 | 团队人数少、开发不频繁、不需同时维护多个版本的小项目 |
| **功能分支工作流** | 上手简单，支持并行开发，支持 Code Review | 无法给分支分配明确目的，不利于团队配合 | 开发团队相对固定、规模较小的项目 |
| **Git Flow 工作流** | 每个分支分工明确，最大限度减少相互影响，可并行开发，支持 Code Review | 有一定上手难度 | 开发团队相对固定、规模较大的项目 |
| **Forking 工作流** | 完全解耦个人远端仓库与项目远端仓库，最大限度保证远端仓库安全 | 对职能分工明确且不对外开源的项目优势不大 | 开源项目，或开发者需派生自己版本、开发者不固定的场景 |


#### 集中式

所有开发者直接在 master 分支上工作。


#### 功能分支

在开发新功能时基于 master 分支新建一个功能分支，在功能分支上进行开发，完成之后合并到 master 分支。


```
# 新建功能分支
git checkout -b feature/rate-limiting

# 开发工作提交到功能分支
git add limit.go
git commit -m "add rate limiting"

# 将本地功能分支代码 push 到远程仓库
git push origin feature/rate-limiting

# 在远程仓库上创建 PR，CR 后 Merge 即可

```

```
master    ●───●─────────●──────────────────●─────────→
              │         │                  │
              │      ┌──●───●───●         │  pull request
              │      │  feature A          │        │
              ├──────┘                     │        ▼
              │                       ┌────◎─────────●─────→
              └──┌──●───●───●         │   feature C
                 │  feature B         │
                 └────────────────────┘
```

> 特性分支从 master 拉出，各自独立提交（图中深色点），完成后通过 **pull request** 合回主干。


Merge PR 推荐使用 Create a merge commit 模式，即 git merge --no-ff，feature 分支上所有的 commit 都会加到 master 分支上，并且会生成一个 merge commit，避免丢失历史记录。


#### Git Flow（推荐）

常用于非开源项目。共有以下类型的分支：


**Git Flow 各分支职责**

| 分支名 | 描述 |
|---|---|
| **master** | 永远处于发布状态，不能直接开发。每合并一个 hotfix/release 分支都会打版本标签 |
| **develop** | 开发中的最新代码，只做合并操作，不能直接开发 |
| **feature** | 研发阶段的开发分支，基于 develop 新建，命名 `feature/xxxx-xxx`。开发完成后合并到 develop 并删除；合并前建议先 `pull` develop 检查冲突 |
| **release** | 发布阶段的预发布分支，基于 develop 新建，命名 `release/xxxx-xxx`。测试通过后合并到 master 和 develop，在 master 打版本标签后删除。例：`release/1.0.0` |
| **hotfix** | 维护阶段的紧急 bug 修复分支，基于 master 新建，命名 `hotfix/xxxx-xxx`。修复后合并到 master 和 develop，打标签后删除 |


示例场景：


当前版本为：0.9.0。

需要新开发一个功能。

同时线上代码有 Bug 需要紧急修复。

基本流程：


```
# 1. 创建要给常驻分支
git checkout -b develop master

# 2. 基于 develop 分支，新建一个功能分支：feature/hello-world。
git checkout -b feature/hello-world develop

# 3. feature/hello-world 分支上开发
echo "feature1" >> test.txt 

# 4. 开发过程中需要先紧急修复线上代码的 bug
git stash                                         # 临时保存修改至堆栈区
git checkout -b hotfix/error master               # 从 master 建立 hotfix 分支
echo "hotfix" >> test.txt                         # 修复 bug
git commit -a -m 'fix print message error bug'    # 提交修复
git checkout develop                              # 切换到 develop 分支
git merge --no-ff hotfix/error                    # 把 hotfix 分支合并到 develop 分支
git checkout master                               # 切换到 master 分支
git merge --no-ff hotfix/error                    # 把 hotfix 分支合并到 master
git tag -a v0.9.1 -m "fix log bug"                # 在 master 分支打上 tag
scp test root@target:/opt/                        # 编译并部署代码
git branch -d hotfix/error                        # 修复完成后删除 hotfix/xxx 分支
git checkout feature/hello-world                  # 切换到 feature 分支下
git merge --no-ff develop                         # develop 有更新，需要同步更新下
git stash pop                                     # 恢复到修复前的工作状态

# 5. 继续开发
echo "feature2" >> test.txt 

# 6. 提交代码到 feature/hello-world 分支
git commit -a -m "print 'hello world'"

# 7. 在 feature/hello-world 分支上做 code review
git push origin feature/print-hello-world         # 提交到远程仓库
# 创建 PR、指定 Reviewers 进行 CR

# 8. CR 过后，由代码仓库 Matainer 将功能分支合并到 develop 分支
git checkout develop
git merge --no-ff feature/hello-world

# 9. 基于 develop 分支创建 release 分支，测试代码
git checkout -b release/1.0.0 develop
cat test.txt 

# 10. 如果测试失败，直接在 release/1.0.0 分支修改代码，完成后提交并编译部署
git commit -a -m "fix bug"
scp test root@target:/opt/

# 11. 测试通过后，将 release/1.0.0 分支合并到 master 分支和 develop 分支
git checkout develop
git merge --no-ff release/1.0.0
git checkout master
git merge --no-ff release/1.0.0
git tag -a v1.0.0 -m "add print hello world" # master 分支打 tag

# 12. 删除 feature/hello-world 分支，也可以选择性删除 release/1.0.0 分支
git branch -d feature/hello-world
# git branch -d release/1.0.0

```

#### Forking（推荐）

常用于开源项目，开发者有衍生出自己的衍生版的需求，或开发者不固定、可能是任意一个能访问到项目的开发者。


其中 fork 操作是在个人远程仓库新建一份目标远程仓库的副本。


```
        A/代码库  ◄──── fork ────   B/代码库
           │  ──── 合并 ────►         │
           │                          │
           │  ──── fork ────►  C/代码库
           │  ◄─── 合并 ─────         │
           │                          │
   PR ────┘                      PR ─┘
     ▲                            ▲
     │                            │
   [A 开发者]                   [C 开发者]
        ▲                       [B 开发者]
        └────────── PR ──────────┘

```

> **Forking 工作流**：每个开发者 fork 出独立仓库，改动通过 **PR** 提交到上游 A 仓库，
> 由 A 维护者审核合并。开发者之间也可互相 PR。


基本流程：


```
# 1. 在远程仓库上 fork 仓库：
# 比如 https://github.com/marmotedu/gitflow-demo => https://github.com/colin404fork/gitflow-demo

# 2. clone 到本地
git clone https://github.com/colin404fork/gitflow-demo
cd gitflow-demo
git remote add upstream https://github.com/marmotedu/gitflow-demo
git remote set-url --push upstream no_push # 不要 push 到 upstream master
git remote -v # 查看远程分支
# origin  https://github.com/colin404fork/gitflow-demo (fetch)
# origin  https://github.com/colin404fork/gitflow-demo (push)
# upstream  https://github.com/marmotedu/gitflow-demo (fetch)
# upstream  https://github.com/marmotedu/gitflow-demo (push)

# 3. 创建功能分支
git fetch upstream                       # 同步本地仓库的 master 分支为最新状态（与 upstream master 分支一致）
git checkout master
git rebase upstream/master
git checkout -b feature/add-function     # 创建功能分支

# 4. 提交 commit
git fetch upstream                       # commit 前需要再次同步 feature 跟 upstream/master
git rebase upstream/master
git add xxx
git status
git commit
git rebase -i origin/master              # 合并多个、保留较少的 commit
# git reset HEAD~5
# git add .
# git commit -am "Here's the bug fix that closes #28"
# git push --force

# 5. push 功能分支到个人远程仓库
git push -f origin feature/add-function

# 6. 创建 PR，请求 Reviewers review、合并到 master
# 创建 pull request 时，base 通常选择目标远程仓库的 master 分支。

```
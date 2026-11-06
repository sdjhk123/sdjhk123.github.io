# Git 与 GitHub 基础

> 给这个博客项目写的入门笔记。目标不是背命令，而是建立一套能自己推理的心智模型——遇到没见过的报错时，知道该往哪个方向想。

---

## 零、先把三个东西分开

初学最容易糊的是这三个词。它们不是一回事：

| 名字 | 是什么 | 在哪 | 需要联网吗 |
|---|---|---|---|
| **Git** | 版本控制工具，记录文件的每次修改 | 你电脑上 | 不需要 |
| **GitHub** | 代码托管网站，存放 Git 仓库 | 网上 | 需要 |
| **GitHub Pages** | GitHub 附带的静态网站托管 | 网上 | 需要 |

三者的关系：Git 负责"记录历史"，GitHub 负责"把历史存到网上并支持多人协作"，GitHub Pages 负责"把仓库里的静态文件当成网站对外提供访问"。

关键推论：

- **没有 GitHub，Git 照样完整工作**。你现在断网，仍然可以 `git add`、`git commit`、`git log`、切分支、回退，全都能做。只是推不上去。
- **没有 Git，GitHub 就没有内容可存**。GitHub 本身不产生内容，它只是一个装着 Git 仓库的柜子。
- 所以"学 GitHub"这件事，八成时间其实在学 Git。

---

## 一、Git 在解决什么问题

设想没有 Git 的时候你是怎么存版本的：

```
论文.doc
论文-修改版.doc
论文-最终版.doc
论文-最终版2.doc
论文-真的最终版-不改了.doc
```

这套办法有三个无法解决的毛病：

1. **不知道改了什么**。两个文件之间的差异要靠肉眼对比。
2. **无法回退到某个中间状态**。想知道"上周二那版是怎么写的"，找不回来。
3. **无法多人同时改**。两个人各改一份，最后合并靠手工。

Git 的解法是：不再保存"多份文件"，而是保存**一条有向的修改历史链**。你只有一个工作目录，历史全在 `.git` 目录里（一个隐藏文件夹）。想看任何一版，从链上取出来就行。

---

## 二、核心模型：三个区域

这是整个 Git 里最重要的图。理解了它，80% 的命令就不用背了。

```
  工作区                    暂存区                    本地仓库              远程仓库
  Working Dir              Staging Area              Repository            Remote
  (你看到的文件夹)          (下次提交的清单)          (.git 里的历史)       (GitHub 上)

     │                         │                         │                     │
     │────── git add ─────────>│                         │                     │
     │                         │───── git commit ───────>│                     │
     │                         │                         │──── git push ──────>│
     │                         │                         │                     │
     │<──────────────── git restore ──────────────────── │                     │
     │<──────────────────────────── git pull ────────────│<────────────────────│
```

**为什么要有个"暂存区"？**

这是新手最疑惑的设计。为什么不直接 commit 所有改动？

因为一次提交应该是一件事。你改了 5 个文件，其中 3 个修 bug、2 个调格式，理想情况是分成两次提交。暂存区让你能挑着来——`git add` 哪几个，就提交哪几个。这是个"打包清单"，提交前还能反悔。

### 文件的四种状态

任何一个文件，在 Git 眼里必处于下面四种状态之一：

| 状态 | 含义 | 怎么回到上一步 |
|---|---|---|
| **untracked** 未跟踪 | 新文件，Git 不管它 | `git add` 变成 staged |
| **modified** 已修改 | 已跟踪的文件被改了，但还没暂存 | `git add` 变成 staged |
| **staged** 已暂存 | 改动已进清单，等着被提交 | `git commit` 变成 committed |
| **committed** 已提交 | 已经在历史里了 | — |

`git status` 干的事就是告诉你"现在每个文件处于哪个状态"。这是你用得最多的命令，没有之一。

---

## 三、提交（commit）到底是什么

一次 commit 保存的不是"差异"，而是**整个项目在那一个瞬间的快照**，外加四样信息：

1. 一个唯一 ID（40 位哈希，如 `798684e...`，平时只用前 7 位）
2. 一个指向父提交的指针
3. 作者、时间、提交说明
4. 当前所有文件的内容快照

因为每个提交都记着"我爹是谁"，这些提交就串成了一条链：

```
A <── B <── C <── D
```

从 D 能一路回溯到 A，这就是"历史"。

**快照 vs 差异的实践意义**：Git 内部做了压缩去重，没变的文件不会重复存储，所以"每次存整个项目"并不臃肿。但理解成快照，比理解成补丁更接近真相。

> 顺带一提：这个设计意味着 Git 里没有"删除某个提交"的操作。你只能改变指针指向哪里，让某段历史变得"不可达"（后面撤销章节会讲）。

---

## 四、分支（branch）是什么

大多数人把分支想象成"复制了一份代码"，这是错的。

**分支只是一个指向某个提交的指针**，一张便利贴而已，创建它几乎零成本。

```
             main
              ↓
A <── B <── C

                  feature
                     ↓
A <── B <── C <── D
```

创建分支 `git branch feature` 做的事情，就是在当前提交上贴一张新便利贴。

再看一个东西：**HEAD**。它指向"你现在在哪"：

```
             main
              ↓
A <── B <── C
              ↑
             HEAD
```

`HEAD → main → 提交 C`。你提交新东西时，先产生提交 E，然后把 `main` 这张便利贴移到 E 上，HEAD 跟着走。

**"detached HEAD"（游离头指针）** 就是 HEAD 没指向任何分支，直接指向某个提交。这时你的新提交不属于任何分支，切走就容易丢。看到这个提示，通常是你 `git checkout <某个提交ID>` 了。

### 合并的两种情况

把 `feature` 合回 `main`（`git merge feature`）时：

**快进合并（fast-forward）**：`main` 没有新提交，只是落后于 `feature`。那么 Git 直接把 `main` 的便利贴挪过去，不产生新提交。历史是直线。

**三方合并（three-way merge）**：两条线都有新提交，Git 必须造一个**合并提交**，它有两个父提交。历史出现分叉再汇合。

---

## 五、远程仓库与 origin

`origin` 只是**默认的远程仓库别名**，不是关键字。你可以叫它 `github`、`blog`，随便。

```
git remote add origin https://github.com/sdjhk123/sdjhk123.github.io.git
```

这句话的意思是："给这个网址起个外号叫 origin。"

之后 `git push origin main` 就能展开成"把本地 main 分支推到 origin 那个地址"。

有个细节值得知道：远程分支和你本地分支是**两套独立的东西**。`origin/main` 是你本地保存的"上次和远程同步时，远程 main 长什么样"的**缓存**，不是实时的。`git fetch` 就是去刷新这个缓存。

所以：

- `git fetch` = 只下载更新，不动你的工作区（安全，可以随便跑）
- `git pull` = `git fetch` + `git merge`（会改你的文件，可能产生冲突）

知道这个区别，你就明白了为什么"先 fetch 看一眼再决定"是更稳妥的习惯。

### `-u` 是什么意思

```bash
git push -u origin main
```

`-u` 是 `--set-upstream`，作用是**记住这个对应关系**。设置之后，`main` 分支就知道自己的远程对应是 `origin/main`，以后直接敲 `git push`、`git pull` 就行，不用每次写全。

只需要在第一次推送时加，之后不用。

---

## 六、命令按场景整理

### 场景 A：第一次配置（一台电脑只做一次）

```bash
git config --global user.name "你的名字"
git config --global user.email "你的邮箱"
```

这两行决定你每次提交里"作者"写的是谁。注意：这跟 GitHub 登录账号**没有验证关系**，谁都能填任意值——所以别填错，否则提交记录里的作者信息是错的。

```bash
git config --global init.defaultBranch main   # 新仓库默认分支叫 main
git config --global core.autocrlf true        # Windows 换行符处理
```

### 场景 B：开始一个新项目 / 拿到一个已有项目

```bash
git init                        # 把当前目录变成 Git 仓库（生成 .git）
git clone <网址>                 # 把远程仓库完整复制到本地
git clone --recurse-submodules <网址>   # 如果项目里有子模块，用这个
```

### 场景 C：日常改东西（90% 的时间在重复这四步）

```bash
git status                  # 我现在改了什么？（随时可以敲）
git diff                    # 具体改在哪几行？（未暂存的改动）
git diff --staged           # 已暂存的改动长什么样
git add <文件>               # 把某个文件放进提交清单
git add .                   # 把当前目录所有改动都放进去
git commit -m "说明"         # 提交
git log --oneline --graph   # 看历史，一行一条，带分支图
```

### 场景 D：和远程同步

```bash
git remote -v               # 我现在连着哪些远程仓库
git fetch                   # 只下载，不改本地（安全）
git pull                    # 下载并合并
git push                    # 推送
git push -u origin main     # 第一次推送时用
```

### 场景 E：分支

```bash
git branch                  # 列出本地分支
git branch <名字>            # 创建分支（不切过去）
git switch <名字>            # 切换分支
git switch -c <名字>         # 创建并切换（最常用）
git merge <名字>             # 把那个分支合到当前分支
git branch -d <名字>         # 删除已合并的分支
```

> `git checkout` 是老命令，既能切分支又能恢复文件，容易误操作。Git 2.23 之后拆成了 `git switch`（切分支）和 `git restore`（恢复文件）。你的 Git 是 2.55，放心用新命令。

---

## 七、后悔药：撤销操作对照表

这是最容易出事的地方。**先判断你的改动走到哪一步了**，再选命令：

| 情况 | 想干什么 | 命令 | 危险度 |
|---|---|---|---|
| 改了文件，没 add | 丢弃修改，恢复原样 | `git restore <文件>` | 低（但改动真没了） |
| 已经 add | 从暂存区撤回（文件内容还在） | `git restore --staged <文件>` | 无（内容不丢） |
| 已经 commit，没 push | 撤销提交但**保留**改动 | `git reset --soft HEAD~1` | 低 |
| 已经 commit，没 push | 撤销提交，改动退回工作区 | `git reset HEAD~1` | 中 |
| 已经 commit，没 push | 撤销提交并**丢弃**所有改动 | `git reset --hard HEAD~1` | **高，不可恢复** |
| 已经 push | 安全地抵消某次提交 | `git revert <提交ID>` | 低（产生新提交） |
| 提交说明写错了 | 改最后一条说明 | `git commit --amend` | 中（会改历史） |
| 想看某文件的历史版本 | 取回旧内容 | `git checkout <ID> -- <文件>` | 低 |
| 改到一半要临时走开 | 收起来 | `git stash` / `git stash pop` | 低 |

三条铁律：

1. **`--hard` 会真的删掉你的代码**，不进回收站。敲之前先 `git status` 确认没有未提交的活。
2. **已经 push 出去的提交不要用 reset 改写**。别人的（或另一台电脑的）历史会对不上，下次 push 会要求强推。用 `revert`。
3. **`git push --force` 是危险动作**。它会覆盖远程历史。个人独占仓库偶尔能用，多人仓库千万别用。真需要时用 `--force-with-lease`，它会在远程有别人的新提交时拒绝执行。

---

## 八、冲突（conflict）

冲突只在一种情况下出现：**同一个文件的同一片区域，被两条不同的历史线改成了不同的内容**，Git 无法判断该听谁的。

它不会猜，直接把两边都塞进文件里，让你决定：

```
<<<<<<< HEAD
这是你当前分支的内容
=======
这是要合并进来的分支的内容
>>>>>>> feature
```

处理流程只有三步：

1. 打开文件，删掉 `<<<<<<<`、`=======`、`>>>>>>>` 这三行标记，把内容改成你要的样子
2. `git add <文件>` —— 告诉 Git"这个我处理好了"
3. `git commit` —— 完成合并

**不要慌**，冲突不是错误，是 Git 在说"这事我判断不了，你来定"。文件不会丢，标记删干净就行。

如果彻底搞砸了想重来：`git merge --abort`，回到合并之前的状态。

---

## 九、.gitignore

有些文件不该进版本库：编译产物、密钥、系统缓存。

在项目根目录建 `.gitignore`，一行一个规则：

```gitignore
public/           # 忽略整个目录
*.log             # 忽略所有 log 文件
!important.log    # 叹号表示例外，这个要保留
.env              # 环境变量（可能含密钥）
```

**一个必须知道的坑**：`.gitignore` 对**已经被跟踪**的文件无效。如果一个文件已经 commit 过，再写进 .gitignore 也没用，得先：

```bash
git rm --cached <文件>    # 从索引里移除，但保留磁盘上的文件
```

你这个项目的 `public/`（Hugo 编译产物）就是靠 .gitignore 排除的。**这一点很重要**：源码进仓库，产物不进。产物由 GitHub Actions 每次重新生成。

---

## 十、提交说明该怎么写

`git commit -m "update"` 这种，三个月后你自己都看不懂。

一个够用的格式：

```
一句话说清做了什么（不超过 50 字）

需要的话，空一行再写为什么这么做。
```

要点：

- 用**祈使句**，当作在给这个提交下命令："修复导航栏移动端错位"，而不是"修复了导航栏"或"nav fixed"
- 说**做了什么**，不是"改了文件"
- 一个提交只做一件事

如果将来参与开源项目，会见到 `feat:` / `fix:` / `docs:` 这种前缀（Conventional Commits 规范），那是团队约定，个人项目不强制。

---

## 十一、GitHub 上的几个核心概念

Git 只管版本，GitHub 在它之上加了一整套协作设施。

**Repository（仓库）**
就是一个 Git 仓库的网上副本，外加 Issues、Actions、Settings 这些附加面板。

**README.md**
仓库根目录的这个文件会自动显示在仓库首页。是给别人看的说明书。

**Fork（派生）**
把别人的仓库复制一份到你账号下。你在自己的副本里随便改，改好了可以发起 PR 请原作者合并。这是参与开源的标准路径。

**Pull Request（PR，拉取请求）**
"我在我的分支上改好了，请你合并进去"。是**代码评审**的载体：可以逐行评论、讨论、CI 自动检查、最后合并。个人博客用不上，但这是 GitHub 最有价值的功能。

**Issue（议题）**
任务、bug、讨论的载体。个人项目里可以当待办清单用。

**Actions（自动化）**
GitHub 提供的 CI/CD：满足触发条件（比如 push）时，在云端的机器上跑一段脚本。

**你已经有这个了**：仓库里 `.github/workflows/deploy.yml` 定义了一条流水线——"有人 push 到 main 就启动一台 Ubuntu，装 Hugo，编译站点，把产物发布到 Pages"。所以你在本地只管写 Markdown，编译和发布全在云端完成。

**Pages**
静态托管服务。它只是"把一批 HTML/CSS/JS 文件挂到某个网址上"，不执行任何后端代码。

**Releases（发布）**
给某个提交打标签（tag）并附上说明和附件，通常用于发布软件版本。博客用不上。

**.github 目录**
GitHub 认的约定目录，放 workflow 定义、Issue 模板、PR 模板等。

---

## 十二、认证：为什么不能输密码了

2021 年 8 月，GitHub 全面禁止用账号密码做 Git 操作。现在三种方式：

| 方式 | 说明 | 你现在的状态 |
|---|---|---|
| **HTTPS + Credential Manager** | 首次弹窗在浏览器授权，之后自动记住 | **正在用这个**，所以 push 没让你输入任何东西 |
| **Personal Access Token（PAT）** | 手动生成一串令牌当密码用，可精细控制权限 | 备选 |
| **SSH Key** | 生成一对密钥，公钥传到 GitHub，之后免密 | 更安全，但要配 |

普通个人使用者，Credential Manager 是最省事的，Windows 上随 Git 一起装好了，你不用管。

凭据存在 Windows 的"凭据管理器"里。哪天推送突然要求重新登录，去那里删掉 github.com 相关条目再操作一次即可。

---

## 十三、子模块（submodule）——你的项目里就有一个

这个必须讲，因为**你的博客主题就是一个子模块**。

`themes/PaperMod` 不是普通的文件夹，它在 Git 里是一条特殊记录：**"指向另一个仓库的某一次提交"**。

父仓库不保存主题的代码，只保存一个指针（比如"PaperMod 的第 1181196 号提交"）。这样做的好处是主题可以独立更新，仓库也不会被几千行别人的代码撑大。

代价是：

- **克隆时必须加参数**，否则 `themes/PaperMod` 会是空文件夹，站点直接编译失败：
  ```bash
  git clone --recurse-submodules <网址>
  ```
- 已经克隆了才想起来：`git submodule update --init --recursive`
- 更新主题到最新版：
  ```bash
  cd themes/PaperMod && git pull origin master && cd ../..
  git add themes/PaperMod && git commit -m "更新主题"
  ```
  （子模块更新后，父仓库要提交一次，记录新的指针）

你的 `deploy.yml` 里写了 `submodules: recursive`，就是让 Actions 在云端检出代码时也把主题拉全——不然云端编译会因缺主题而失败。

---

## 十四、这个博客项目的具体映射

把上面的概念对上你现在这个仓库：

| 概念 | 在你项目里是什么 |
|---|---|
| 工作区 | `F:\Sdjhk123 blog` 这个文件夹 |
| 本地仓库 | `F:\Sdjhk123 blog\.git`（隐藏目录） |
| 远程仓库 | `github.com/sdjhk123/sdjhk123.github.io`，别名 `origin` |
| 当前分支 | `main` |
| 源码 | `hugo.toml`、`content/`、`static/`、`.github/` |
| 编译产物 | `public/`（被 .gitignore 排除，不进仓库） |
| 子模块 | `themes/PaperMod` |
| 部署流水线 | `.github/workflows/deploy.yml` |
| 发布结果 | `https://sdjhk123.github.io` |

写一篇文章并发布，实际发生的完整链条：

```
在 content/posts/ 新建 .md
        ↓
git add / commit          （存进本地历史）
        ↓
git push                  （传到 GitHub）
        ↓
Actions 检测到 main 有新提交，启动虚拟机
        ↓
装 Hugo → hugo --minify → 生成 public/
        ↓
把 public/ 发布到 Pages
        ↓
https://sdjhk123.github.io 更新（约 30–60 秒）
```

**这条链解释了为什么本地 `public/` 不用提交**：云端每次都会重新生成一份最新的，你提交的旧产物只会被覆盖，白白占空间。

---

## 十五、常见报错翻译

| 报错 | 真实含义 | 怎么处理 |
|---|---|---|
| `rejected ... non-fast-forward` | 远程有你本地没有的提交，直接推会覆盖它 | 先 `git pull` 合并，再 push |
| `fatal: not a git repository` | 当前目录不是 Git 仓库 | `cd` 到项目目录，或 `git init` |
| `Your branch is behind 'origin/main'` | 远程比本地新 | `git pull` |
| `detached HEAD` | HEAD 没指向分支 | `git switch -` 或 `git switch main` |
| `CONFLICT (content): Merge conflict in ...` | 同一处被两种方式修改 | 手动编辑冲突标记 → add → commit |
| `Please tell me who you are` | 没配 user.name / user.email | 跑场景 A 那两行 config |
| `Permission denied (publickey)` | SSH 密钥没配好 | 改用 HTTPS，或用 Credential Manager |
| `Support for password authentication was removed` | 用密码认证，已被禁用 | 用 Credential Manager 或 PAT |
| `nothing to commit, working tree clean` | 没有改动可提交 | 不是错误，说明当前是干净的 |
| `fatal: pathspec did not match any files` | add 的文件名写错了 | 检查路径和大小写 |

---

## 十六、学习路径建议

**第一阶段（够用就行）**
`status`、`add`、`commit`、`push`、`pull`、`log`。这六个命令能覆盖 95% 的个人使用场景。配合 VS Code 的图形界面，你甚至不用记。

**第二阶段（开始好奇）**
理解暂存区的意义、分支是什么、`fetch` 和 `pull` 的区别。到这个程度，遇到问题能自己推理。

**第三阶段（需要时再学）**
`rebase`、`cherry-pick`、`bisect`、`reflog`。

特别推荐早点知道 **`git reflog`**：它记录了 HEAD 的每一次移动，包括你以为"删掉了"的提交。`reset --hard` 之后如果追悔莫及，用 `git reflog` 找到那个提交 ID，还能救回来。这是 Git 里最像后悔药的东西。

**关于图形界面**：图形工具（VS Code、GitHub Desktop）只是把同样的操作包装了一层。**先理解模型再用工具**，反过来会很痛苦——因为工具在你出错时不会告诉你发生了什么。

---

## 附：一张速查卡

```bash
# 我现在什么状态
git status
git log --oneline --graph --all

# 保存改动
git add .
git commit -m "说明"
git push

# 取回远程
git pull

# 撤销（按阶段选）
git restore <文件>              # 未 add
git restore --staged <文件>     # 已 add
git reset --soft HEAD~1         # 已 commit 未 push，保留改动
git revert <ID>                 # 已 push

# 分支
git switch -c <名字>
git switch <名字>
git merge <名字>

# 救命
git reflog
git stash
git merge --abort
```

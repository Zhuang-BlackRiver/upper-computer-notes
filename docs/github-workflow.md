# GitHub 实操手册（Windows）

> **内容分两块**：
> 上半部分（0~6 节）解决"连不上"——把 HTTPS 换成 SSH 的 443 端口；
> 下半部分（7 节起）是日常操作——拉取、查看改动、提交、推送、分支、撤销、冲突。
>
> 适用场景：`git clone` / `git pull` / `git push` 卡住不动、超时、或报
> `CONNECT tunnel failed, response 502`，但浏览器能上网。

## 目录

| 节 | 内容 |
|---|---|
| 0 | 先判断问题类型（三条探测命令 + 判读表） |
| 1 | 生成 SSH 密钥 |
| 2 | 把公钥贴到 GitHub |
| 3 | 配置 443 端口通道 |
| 4 | 验证认证是否成功 |
| 5 | 让仓库走 SSH（新克隆 / 改 remote / 全局改写） |
| 6 | 完整验收（四关逐层验证） |
| **7** | **日常工作流：拉取 / 查看 / 提交 / 推送 / 分支 / 冲突 / 后悔药** |
| 8 | 故障排查（8.1 连接类 / 8.2 日常操作类） |
| 9 | 四件不要做的事 |
| 10 | 附录：发给别人的精简版 |

---

## 0. 先判断：你的问题是不是同一类

在 Git Bash 里跑这三条，对照下面的结果表：

```bash
# ① HTTPS 通不通
curl -sS -o /dev/null -w "HTTP=%{http_code} 耗时=%{time_total}s\n" --max-time 15 https://github.com

# ② 裸 TCP 端口通不通
timeout 8 bash -c "cat < /dev/null > /dev/tcp/github.com/443" && echo "443 通" || echo "443 不通"

# ③ SSH 入口通不通
timeout 8 bash -c "cat < /dev/null > /dev/tcp/ssh.github.com/443" && echo "ssh:443 通" || echo "ssh:443 不通"
```

| ① HTTPS | ② github.com:443 | ③ ssh.github.com:443 | 结论 |
|---|---|---|---|
| 超时 / 000 | 通 | 通 | **典型 TLS 干扰，本文档适用** |
| 200 | 通 | 通 | 网络正常，问题在别处（看第 8 节） |
| 通 | 通 | 不通 | 换 22 端口试试，或公司防火墙限制 |
| 不通 | 不通 | 不通 | 整机没网 / 代理配置错误，先解决上网 |

再补一条 DNS 检查，排除污染：

```bash
nslookup github.com
```

解析到 `20.205.243.166` 这类真实 IP 就没问题；如果返回 `127.0.0.1` 或被劫持的地址，先处理 DNS。

---

## 1. 第一步：生成 SSH 密钥

已经有钥匙的跳过本节。**检查是否已有**：

```bash
ls -1 ~/.ssh/*.pub 2>/dev/null || echo "没有公钥，需要生成"
```

生成（Git Bash 中执行）：

```bash
ssh-keygen -t ed25519 -C "你的邮箱@example.com"
```

`-C` 只是注释标签，方便以后在 GitHub 上认出是哪台机器，不影响功能。

回车后连续三个提示：

| 提示 | 输入 | 说明 |
|---|---|---|
| `Enter file in which to save the key (...)` | **回车** | 用默认 `~/.ssh/id_ed25519` |
| `Enter passphrase (empty for no passphrase):` | **回车** | 留空=不设口令；设了口令每次 push 都要输 |
| `Enter same passphrase again:` | **回车** | 同上 |

> **警告**：如果它问 `Overwrite (y/n)?`，说明同名文件已存在。
> 回答 `y` 会覆盖现有密钥，GitHub 上那把立即失效。应回答 `n`，
> 重新执行并在第一步手动指定新文件名（如 `~/.ssh/id_ed25519_work`）。

生成结果：

```
~/.ssh/id_ed25519        私钥，绝不外传、绝不提交进仓库
~/.ssh/id_ed25519.pub    公钥，贴到 GitHub 的那串
```

如果 `ed25519` 报错（Git 版本很老），改用 RSA：

```bash
ssh-keygen -t rsa -b 4096 -C "你的邮箱@example.com"
```

---

## 2. 第二步：把公钥贴到 GitHub

**取出公钥并复制到剪贴板**（Git Bash）：

```bash
cat ~/.ssh/id_ed25519.pub | clip
```

PowerShell 版本：

```powershell
Get-Content $env:USERPROFILE\.ssh\id_ed25519.pub | Set-Clipboard
```

公钥长这样，三段结构：

```
ssh-ed25519 AAAAC3NzaC1lZDI1NTE5AAAAI... 你的邮箱@example.com
     ↑                    ↑                      ↑
  算法类型            公钥数据                 注释
```

**粘贴到 GitHub**：

1. 打开 https://github.com/settings/keys
2. 点 `New SSH key`
3. `Title` 填个能认出来的名字，比如 `home-pc`、`company-laptop`
4. `Key type` 保持 `Authentication Key`
5. `Key` 里粘贴整串（含末尾邮箱注释）
6. 点 `Add SSH key`，需要输一次账号密码确认

> 公钥可以公开，私钥（没有 `.pub` 的那个）不能给任何人。
> 公钥像挂在外面的锁孔，私钥是兜里的钥匙——别人拿到锁孔做不了任何事。

---

## 3. 第三步：配置 443 端口通道

目的：让 `git@github.com` 实际连到 `ssh.github.com:443`，绕开 22 端口封锁和 HTTPS 干扰。

### 3.1 先看有没有现成的 config

```bash
cat ~/.ssh/config 2>/dev/null || echo "没有 config，可以新建"
```

- **输出为空 / 提示不存在** → 用下面的方式一
- **已有内容** → 不要覆盖，用编辑器在文件末尾追加那 6 行

### 3.2 方式一：写入 config（推荐，长期生效）

```bash
cat > ~/.ssh/config <<'EOF'
Host github.com
  HostName ssh.github.com
  Port 443
  User git
  IdentityFile ~/.ssh/id_ed25519
  IdentitiesOnly yes
EOF
chmod 600 ~/.ssh/config
```

逐行含义：

| 行 | 作用 |
|---|---|
| `Host github.com` | 别名。你写 `git@github.com:...` 时匹配这一段 |
| `HostName ssh.github.com` | 实际连接的服务器（GitHub 的 SSH-over-443 入口） |
| `Port 443` | 走 443，形似普通 HTTPS 流量 |
| `User git` | SSH 登录名固定是 `git`，不是你的 GitHub 账号名 |
| `IdentityFile ~/.ssh/id_ed25519` | 指定用哪把私钥。**最容易错的一行**，文件名必须和实际生成的一致（已有钥匙可能是 `id_rsa`） |
| `IdentitiesOnly yes` | 只用这一把，不挨个尝试机器上所有钥匙（部分服务器 6 次失败即断开） |

`chmod 600` 把权限收成"仅自己可读写"。Linux/macOS 上权限过宽 SSH 会直接拒绝使用；Windows 上影响较小，加上无妨。

### 3.3 方式二：不改配置，地址里带端口

不想碰配置文件时，克隆时用完整 URL 形式：

```bash
git clone ssh://git@ssh.github.com:443/用户名/仓库名.git
```

注意格式：`ssh://` + `git@` + 主机 + `:443` + `/owner/repo.git`（是斜杠 `/`，不是冒号 `:`）。

缺点是每个仓库都要写一遍长地址，长期用还是方式一省事。

---

## 4. 第四步：验证

```bash
ssh -T git@github.com
```

首次连接会提示服务器指纹：

```
The authenticity of host '[ssh.github.com]:443 (...)' can't be established.
ED25519 key fingerprint is SHA256:xxxxxxxxxxxxx.
Are you sure you want to continue connecting (yes/no/[fingerprint])?
```

输入 **`yes`**（三个字母打全，不能只打 `y`）。这一步把 GitHub 指纹记进 `known_hosts`，防中间人冒充。

成功标志：

```
Hi 你的用户名! You've successfully authenticated, but GitHub does not provide shell access.
```

后半句 `does not provide shell access` 是正常的——GitHub 只允许 git 操作，不给交互式 shell，不是报错。

---

## 5. 第五步：让仓库走 SSH

### 5.1 新克隆

```bash
cd "D:/目标文件夹"
git clone git@github.com:用户名/仓库名.git
```

⚠️ **不要用网页上绿色 `Code` 按钮默认给的 `https://` 地址**，那正是被拦的那条路。
需要 SSH 地址时，点 `Code` 后切到弹窗顶部的 `SSH` 标签复制。

### 5.2 已有仓库改远端

```bash
cd "D:/已有的/仓库"
git remote -v                                              # 先看当前地址
git remote set-url origin git@github.com:用户名/仓库名.git   # 切换
git remote -v                                              # 确认已变
git pull
```

如果 `git pull` 报 `no tracking information`，补一次上游关联：

```bash
git branch --set-upstream-to=origin/main main
```

### 5.3 全局自动改写（可选）

让所有 `https://github.com/...` 地址自动变成 SSH：

```bash
git config --global url."git@github.com:".insteadOf "https://github.com/"
```

副作用：以后 `git clone https://github.com/任意仓库` 都会走 SSH。
克隆**公开仓库**同样需要一个有效密钥认证——你的钥匙已配好，所以没问题；
但这台机器如果换用户，容易让人困惑"为什么 https 地址要我 SSH 认证"。
不想要了可以撤销：

```bash
git config --global --unset url."git@github.com:".insteadOf
```

---

## 6. 完整验收流程

四关逐层验证，任何一关失败都能立刻定位到具体层级：

| 顺序 | 命令 | 验证什么 | 期望输出 |
|---|---|---|---|
| 1 | `ssh -T git@github.com` | 认证层：密钥与账号是否配对 | `Hi xxx! You've successfully authenticated` |
| 2 | `git ls-remote origin` | 协议层：能否读取仓库引用 | 列出分支和 tag |
| 3 | `git pull --ff-only` | 读取 + 合并路径 | `Already up to date.` |
| 4 | `git push --dry-run` | 写权限 | `Everything up-to-date` |

第 4 关用 `--dry-run` 是刻意的：完整走一遍权限校验但**不真正发送提交**，
既验证了写权限，又不会在仓库里留下测试提交。

---

## 7. 日常工作流（拉取 / 查看 / 提交 / 推送 / 分支）

配置完之后，所有命令和以前一样，不需要额外参数。下面按一天的实际节奏排列。

### 7.1 第一次拿到仓库：三种情形

**情形 A — 本地什么都没有，直接克隆（最常见）**

```bash
cd "D:/face/pack/Common/github"
git clone git@github.com:Zhuang-BlackRiver/upper-computer-notes.git
cd upper-computer-notes
```

`clone` 会自动配好 `origin` 和上游分支，之后 `git pull` / `git push` 都不用带参数。

**情形 B — 本地已有一堆代码，要关联到远程仓库**

```bash
cd "D:/已有项目"
git init
git remote add origin git@github.com:Zhuang-BlackRiver/仓库名.git
git fetch origin
git branch --set-upstream-to=origin/main main
git pull --ff-only
```

如果本地已有同名文件，`pull` 可能报 `untracked working tree files would be overwritten`。
说明远程的文件和你本地未纳入版本管理的文件重名了——先决定本地那份是留还是丢，
处理完再 pull。

**情形 C — 仓库已在本机，只是要更新**

```bash
cd "D:/face/pack/Common/github/upper-computer-notes"
git pull
```

### 7.2 每天开工前的三个动作

```bash
git status        # 有没有上次遗留、忘了提交的改动
git branch -vv    # 当前在哪个分支，跟踪的是哪个远程分支
git pull          # 先同步，避免写完了才发现推不上去
```

先 `pull` 的意义：如果远程有你本地没有的提交，你先 commit 再 pull 会产生分叉，
需要多走一次 merge；先拉干净再改，整个过程就是一条直线。

### 7.3 查看改了什么

按"从粗到细"排：

| 目的 | 命令 |
|---|---|
| 哪些文件被改（简版，带分支信息） | `git status -sb` |
| 未暂存的逐行改动 | `git diff` |
| 已 `add` 但未提交的改动 | `git diff --staged` |
| 包含暂存区的全部改动 | `git diff HEAD` |
| 只看文件名列表 | `git diff --name-only` |
| 只看统计（每个文件几增几减） | `git diff --stat` |
| 这一行是谁哪次改的 | `git blame 文件名` |

历史与单次改动：

```bash
git log --oneline -10              # 最近 10 条，一行一条
git log --oneline --graph --all    # 带分支走向的字符图
git log -p -- 文件名                # 这个文件每次改了什么
git show 提交ID                     # 某次提交的完整内容
git diff HEAD~3 -- 文件名           # 和 3 个提交之前对比
```

> `--` 后面的部分表示"路径"，用来和分支名区分。文件名和分支名撞车时，这个分隔符能救命。

### 7.4 提交

标准三步：

```bash
git status                  # 1. 确认要提交什么
git add 文件名               # 2. 暂存
git commit -m "说明"         # 3. 提交
```

`git add` 的范围差异，这里最容易出错：

| 命令 | 范围 |
|---|---|
| `git add 具体文件.cs` | 只加这一个 |
| `git add .` | 当前目录及子目录的改动（**不包含已删除的文件**） |
| `git add -A` | 整个仓库，含新增和删除 |
| `git add -p` | 逐块确认，适合一次改动里混了不相关的修改 |

**提交信息的写法**：第一行说清"做了什么"，别写 `update`、`修改`、`提交`。

```
好：修复 BatchRead 在跨界寄存器上的分片错误
好：docs: 补充 W1 批量读的验收用例
差：update
差：改了一下
```

文档/笔记类仓库可以加前缀 `docs:` `feat:` `fix:` `refactor:`，这套写法叫
Conventional Commits，不算硬性要求，但面试时是能说两句的细节。

**提交之后发现不对：**

| 情况 | 命令 | 前提 |
|---|---|---|
| 提交信息写错了 | `git commit --amend -m "新说明"` | 尚未 push |
| 漏了一个文件 | `git add 漏的 && git commit --amend --no-edit` | 尚未 push |
| 已经 push 了 | 不要 amend，直接再提交一次 | — |

改写已推送的历史会让所有拉过这个分支的人产生冲突，这是团队协作里最招人烦的操作之一。

### 7.5 撤销修改（四种，别搞混）

| 想撤销什么 | 命令 | 危险度 |
|---|---|---|
| 工作区某文件的改动（还没 add） | `git restore 文件名` | 中，改动直接消失 |
| 已 add，撤回暂存区 | `git restore --staged 文件名` | 低，改动还在 |
| 已 commit，想撤回但保留改动 | `git reset --soft HEAD~1` | 低，改动保留且已暂存 |
| 已 commit，想连同改动一起丢弃 | `git reset --hard HEAD~1` | **高，不可恢复** |

记一句：`--soft` 全留、`--mixed`（默认）留内容不留暂存、`--hard` 全丢。

### 7.6 推送

```bash
git push                     # 推当前分支到它跟踪的远程分支
git push -u origin 分支名     # 第一次推新分支，同时建立跟踪关系
git push --dry-run           # 只校验权限和状态，不真推
```

被拒的时候长这样：

```
! [rejected]  main -> main  (fetch first)
```

含义是"远程有你没有的提交"。处理：

```bash
git pull --rebase    # 把你的提交接到远程最新提交之后
git push
```

除非确认这个分支只有你一个人在用，否则别用 `git push -f`。

### 7.7 分支

| 操作 | 命令 |
|---|---|
| 查看本地分支 | `git branch` |
| 查看全部分支（含远程） | `git branch -a` |
| 查看分支及跟踪关系 | `git branch -vv` |
| 新建并切换 | `git switch -c 新分支名` |
| 切换到已有分支 | `git switch 分支名` |
| 重命名当前分支 | `git branch -m 新名字` |
| 删除已合并的分支 | `git branch -d 分支名` |
| 强制删除未合并分支 | `git branch -D 分支名` |
| 推新分支到远程 | `git push -u origin 分支名` |
| 把某分支合并进当前分支 | `git merge 分支名` |
| 删除远程分支 | `git push origin --delete 分支名` |

Git 2.23 以前没有 `switch`，用 `checkout` 等价替代：`git checkout -b 新分支` / `git checkout 分支名`。

**建议**：`main` 始终保持随时可用的状态，任何改动都在独立分支上做。
哪怕只有你一个人——某次尝试失败时，删掉分支就行，不用去回滚 main。

### 7.8 冲突处理

`pull` 或 `merge` 时报：

```
CONFLICT (content): Merge conflict in xxx.cs
```

流程：

```bash
git status          # 找出 both modified 的文件
# 打开文件，会看到 <<<<<<< / ======= / >>>>>>> 三段，
# 手工保留想要的内容，并把这三行标记删掉
git add 冲突文件
git commit          # merge 会自动带出提交信息
```

rebase 过程中遇到冲突，收尾命令是 `git rebase --continue` 而不是 `commit`。

想整个放弃这次合并：

```bash
git merge --abort
```

### 7.9 后悔药：reflog

```bash
git reflog
```

它记录 HEAD 的每一次移动——reset、切分支、合并、rebase 都在里面，默认保留 90 天。

**凡是"提交丢了""分支删错了""reset 错了方向"，第一件事都是看 reflog**，找到当时的提交 ID：

```bash
git reset --hard 提交ID
# 或者从那个提交拉一个新分支出来
git switch -c 恢复的分支 提交ID
```

这条命令能救回绝大多数误操作，比任何备份都实用。

### 7.10 场景速查

| 我想…… | 命令 |
|---|---|
| 看当前状态 | `git status -sb` |
| 看具体改了哪些行 | `git diff` |
| 提交指定几个文件 | `git add 文件1 文件2 && git commit -m "说明"` |
| 提交全部改动 | `git add -A && git commit -m "说明"` |
| 改刚才那次提交的说明 | `git commit --amend -m "新说明"` |
| 撤回刚才的提交（保留改动） | `git reset --soft HEAD~1` |
| 丢掉某文件的所有改动 | `git restore 文件` |
| 拉最新代码 | `git pull` |
| 推上去 | `git push` |
| 开一个功能分支 | `git switch -c feature/xxx` |
| 合并回主线 | `git switch main && git merge feature/xxx` |
| 查某段代码为什么这么写 | `git log -p -- 文件` / `git blame 文件` |
| 找回误删的分支或提交 | `git reflog` |
| 看远程仓库地址 | `git remote -v` |
| 临时存一下改动去改别的 | `git stash` → 处理完 → `git stash pop` |

---

## 8. 故障排查

### 8.1 连接与认证

| 现象 | 原因 | 处理 |
|---|---|---|
| `Permission denied (publickey)` | 公钥没贴 / 贴错账号 / `IdentityFile` 文件名不对 | `cat ~/.ssh/config` 看路径，`ls ~/.ssh/` 核对文件名 |
| `Bad configuration option` | config 用了 Tab 或全角空格缩进 | 改用两个半角空格 |
| 连的还是 22 端口 / `Connection refused` | config 文件名存成了 `config.txt`，或 `Port 443` 被注释 | 记事本另存时选"所有文件"并去掉扩展名；`cat` 检查内容 |
| `destination path already exists and is not an empty directory` | 目标目录已有同名文件夹 | 换名字，或先移走旧目录 |
| `warning: adding embedded git repository` | 目标目录落在另一个 git 仓库内部 | 换一个不在任何仓库内的目录 |
| `Host key verification failed` | GitHub 换过服务器密钥，或 `known_hosts` 记录过期 | `ssh-keygen -R ssh.github.com` 后重连并输 `yes` |
| `ssh: connect to host ... port 443: Connection timed out` | 公司防火墙拦了出站 443 | 换网络环境（手机热点）验证，或联系 IT 开白名单 |

### 8.2 日常 git 操作

| 现象 | 原因 | 处理 |
|---|---|---|
| `! [rejected] ... (fetch first)` | 远程有本地没有的提交 | `git pull --rebase` 后重新 `push` |
| `! [rejected] ... (non-fast-forward)` | 同上；或本地历史被改写过 | 同上。只有确认分支仅自己在用时才考虑 `git push -f` |
| `CONFLICT (content): Merge conflict in xxx` | 两边改了同一处 | 手工编辑 → `git add` → `git commit`；想放弃用 `git merge --abort` |
| `nothing to commit, working tree clean` | 没改动，或文件没被 `add`，或被 `.gitignore` 忽略 | `git status` 确认；`git check-ignore -v 文件` 查是哪条规则拦的 |
| `HEAD detached at abc1234` | 切到了某个提交而非分支（游离头指针） | `git switch main` 回分支；要保留当前改动就先 `git switch -c 新分支` |
| `error: src refspec main does not match any` | 本地还没有任何提交，或分支名不是 main | 先 `git commit`；必要时 `git branch -M main` 改名再推 |
| `fatal: refusing to merge unrelated histories` | 两边是各自独立 `git init` 出来的历史 | `git pull --allow-unrelated-histories` |
| `Your local changes would be overwritten by merge` | 本地未提交的改动和目标文件冲突 | 先 `git stash`，pull 完再 `git stash pop` |
| `Please tell me who you are` | 没配 `user.name` / `user.email` | `git config user.name "名字"` + `git config user.email "邮箱"` |
| 提交里混进了不该提交的文件 | `.gitignore` 没配全，或用了 `git add -A` | `git rm --cached 文件` 取消跟踪，再把规则写进 `.gitignore` |

---

## 9. 四件不要做的事

1. **不要把 `http://127.0.0.1:xxxxx` 写进 git 全局代理配置**。
   那是工具进程内部的转发端口，进程重启后端口就变，写死后典型症状是
   "昨天还好好的，今天 git 全挂"。

2. **不要重新生成密钥去覆盖已有的**。GitHub 上那把会立即失效，
   其它指向旧钥匙的机器全部断连。

3. **不要把私钥（无 `.pub` 后缀的文件）发给任何人**，也不要放进仓库。
   一旦泄露，等于把账号写入权交出去了，只能去 GitHub 删除该公钥止损。

4. **不要在别人的仓库里随便加 `insteadOf` 全局改写**。
   它会改变所有 GitHub 地址的解析方式，影响面超出你的预期。

---

## 10. 附录：把这段发给需要的人

> 如果你的 clone 卡住或报 502，按下面做（Windows，Git Bash 环境）：
>
> ```bash
> # 1. 生成密钥（三个提示全部直接回车）
> ssh-keygen -t ed25519 -C "你的邮箱"
>
> # 2. 复制公钥
> cat ~/.ssh/id_ed25519.pub | clip
> ```
>
> 3. 打开 https://github.com/settings/keys → `New SSH key` → 粘贴 → 保存
>
> ```bash
> # 4. 配置 443 通道
> cat > ~/.ssh/config <<'EOF'
> Host github.com
>   HostName ssh.github.com
>   Port 443
>   User git
>   IdentityFile ~/.ssh/id_ed25519
>   IdentitiesOnly yes
> EOF
>
> # 5. 验证，首次会问 yes/no，输 yes
> ssh -T git@github.com
>
> # 6. 用 SSH 地址克隆（不是 https 地址）
> git clone git@github.com:owner/repo.git
> ```
>
> 看到 `Hi 你的用户名! You've successfully authenticated` 就成了。

---

## 附录 A：本次配置的实际记录

| 项目 | 值 |
|---|---|
| 密钥文件 | `C:\Users\HP\.ssh\id_rsa`（3072 位 RSA，已有，未重新生成） |
| 公钥注释 | `1136374668@qq.com` |
| 配置文件 | `C:\Users\HP\.ssh\config`（原为 0 字节空文件） |
| 诊断结论 | TCP 443 可达、SSH 可握手、HTTPS 的 TLS 握手被中断 |
| 已切换到 SSH 的仓库 | `D:\face\pack\Common\github\upper-computer-notes` |
| 生效范围 | 仅该仓库。其它用 HTTPS 克隆的仓库仍走原路，需单独 `set-url` |

---

*最后更新：2026-11-08*

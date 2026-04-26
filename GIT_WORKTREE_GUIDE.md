# Vibe Coding 下的 Git Worktree 实战

> 在 AI 辅助开发背景下，如何用 Git Worktree 真正提升团队交付效率

---

## 先说痛点

你有没有遇到过这种情况：

正在用 AI 高速推进一个功能，代码写到一半，忽然来了一个 Hotfix 需求或者 Code Review 请求。

然后你不得不：

```bash
git stash save "WIP: new dashboard"
git checkout main
git checkout -b hotfix/login-bug
# ... 修复 ...
git push origin hotfix/login-bug
git checkout feature/new-dashboard
git stash pop
# 哎呀，stash pop 有冲突...
```

每次切换，AI 对话的上下文要重新建立，dev server 要重启，构建缓存失效。原本高速运转的开发节奏，就这样被打断了。

这就是 Vibe Coding 场景下，传统分支管理方式的核心问题：**AI 让你更频繁地需要切换任务，但切换本身的成本并没有降低。**

---

## 什么是 Git Worktree

Git Worktree 允许你在同一个仓库上创建多个独立工作目录，每个目录对应一个分支。

可以把它理解成：

- 同一套 `.git` 仓库，同一段提交历史
- 但可以同时拥有多个独立的"工作现场"

```bash
cd ~/project

# 为新功能创建独立工作树
git worktree add -b feature/123-login ../wt-feature-123-login main

# 为紧急修复创建另一个工作树
git worktree add -b hotfix/login-bug ../wt-hotfix-login-bug main
```

结果：

```text
project/              ← .git 在这里，所有工作树共享
  ├── .git/
  └── src/

wt-feature-123-login/ ← 对应 feature/123-login 分支
  └── src/

wt-hotfix-login-bug/  ← 对应 hotfix/login-bug 分支
  └── src/
```

现在你可以同时在三个目录里工作，互不干扰，每个目录都有独立的终端、运行状态和 AI 对话上下文。

还是刚才那个场景，换成 Worktree 方式：

```bash
# 不需要 stash，不需要切分支
# 直接创建一个新的工作树处理 hotfix
git worktree add -b hotfix/login-bug ../wt-hotfix-login-bug main
cd ../wt-hotfix-login-bug

# 修复、测试、提交
git commit -am "Fix: login timeout bug"
git push origin hotfix/login-bug

# 删除 worktree，回到原来的目录
git worktree remove ../wt-hotfix-login-bug
cd ~/project
# feature/new-dashboard 分支完全没动，所有改动都在，AI 对话也还在
```

---

## 为什么 Vibe Coding 更需要它

Vibe Coding 的本质是：**开发者和 AI 高频协作，让"实现"这一步变得更快、更连续。**

但 AI 带来的副作用也很明显：你会更频繁地同时推进多件事，并且更依赖 IDE 内的聊天上下文和运行状态。

对比一下：

| 维度 | 传统切分支 | Git Worktree | Worktree + AI |
|------|-----------|--------------|---------------|
| 紧急 Hotfix 处理 | stash → 切换 → 修复 → 切回 → pop | 新开目录，互不干扰 | 新开目录 + 独立 AI 对话，上下文不串 |
| Code Review + 开发并行 | 容易互相覆盖 | 隔离目录 | 隔离目录 + AI 分别辅助 |
| dev server | 切换后要重启 | 可同时运行多个 | 可同时运行，任务上下文稳定 |
| 构建缓存 | 切换后容易失效 | 各目录独立保留 | 稳定 |
| AI 对话上下文 | 容易串台 | 可按目录隔离 | 每个任务有独立干净的上下文 |
| 心智负担 | 高 | 中 | 最低 |

核心逻辑只有一句：

**AI 加速了任务产生的速度，Worktree 负责让这些任务互不干扰地并行推进。**

---

## 常用命令速查

### 创建工作树

```bash
# 基于 main 创建新分支 + 新工作树
git worktree add -b feature/123-login ../wt-feature-123-login main

# 基于已有远程分支创建工作树（用于 review）
git worktree add ../wt-review-payment feature/payment

# 基于 main 快速开 hotfix 工作树
git worktree add -b hotfix/login-bug ../wt-hotfix-login-bug main
```

### 查看 / 清理

```bash
git worktree list              # 查看所有工作树
git worktree list --porcelain  # 查看详细状态

git worktree remove ../wt-review-payment  # 删除工作树
git worktree prune                         # 清理无效引用
```

### 一个重要限制

**同一个分支不能同时挂载到两个 worktree。**

这不是缺陷，反而是一个保护机制——它强制你用"一个任务一个分支"的方式工作。

---

## 实战场景

### 场景一：Hotfix 不打断功能开发

**背景**：你正在 `feature/new-dashboard` 分支上用 AI 开发新功能，代码写到一半。产品找过来："线上登录功能挂了，马上修复！"

**传统做法的问题**：

```bash
git stash save "WIP: new dashboard"
git checkout main
git checkout -b hotfix/login-bug
# 修复完...
git checkout feature/new-dashboard
git stash pop
# 常常这一步就出冲突了
```

**Worktree 做法**：

```bash
# 主工作目录完全不动，直接开一个新的 worktree
git worktree add -b hotfix/login-bug ../wt-hotfix-login-bug main
cd ../wt-hotfix-login-bug

# 在新窗口里打开这个目录，让 AI 帮你定位和修复
# 修复完成后提交
git commit -am "Fix: login timeout bug"
git push origin hotfix/login-bug

# 清理
cd ..
git worktree remove wt-hotfix-login-bug

# 回到原来的目录，一切都还在原地
cd ~/project
git status  # 还是 feature/new-dashboard，所有改动都在
```

**AI 在这里的角色**：在 hotfix 的窗口里，你可以告诉 AI："我现在在一个专门用于修复登录超时的 worktree，帮我定位根因并给出最小修复方案，不要动其他逻辑。"这样 AI 的注意力不会被 dashboard 相关的文件和历史对话干扰。

---

### 场景二：Code Review 与开发并行

**背景**：你需要 review 同事的支付分支 PR，但自己的功能开发也不能停。

```bash
# 主工作树继续开发功能
# 另开一个 worktree 专门用于 review
git worktree add ../wt-review-payment feature/payment

cd ../wt-review-payment
npm install
npm run dev  # 独立启动，不干扰你的功能开发端口

# review 完成后
git worktree remove ../wt-review-payment
```

在 review 的窗口里，你可以让 AI："请先总结这个分支改了什么，然后找出潜在 bug 和回归风险，关注行为变化，不要做风格建议。"

这比在同一个目录里切换分支 review 更干净，因为 dev server 是独立的，AI 的文件引用范围也是隔离的。

---

### 场景三：多版本并行维护

**背景**：项目有 v1.0 和 v2.0 两个版本同时维护，需要给两个版本都加一个功能。

```bash
# 为两个版本各建一个工作树
git worktree add ../wt-v1.0 release/v1.0
git worktree add ../wt-v2.0 release/v2.0

# 各自配置独立环境
echo "API_URL=https://api-v1.example.com" > ../wt-v1.0/.env.local
echo "API_URL=https://api-v2.example.com" > ../wt-v2.0/.env.local

# 分别启动（不同端口）
# Terminal 1: cd ../wt-v1.0 && npm run dev   # 端口 3000
# Terminal 2: cd ../wt-v2.0 && npm run dev   # 端口 3001

# 分别在各自窗口提交
cd ../wt-v1.0 && git commit -am "Add user profile (v1.0)"
cd ../wt-v2.0 && git commit -am "Add user profile (v2.0)"
```

在每个版本的窗口里，AI 的文件上下文、.env 配置和 dev server 都是独立的，不会互相干扰。

---

### 场景四：AI 驱动的方案 A/B 实验

**背景**：有一个性能优化方向，不确定哪种方案更好，想让 AI 分别实现两套然后对比。

```bash
# 开两个实验性 worktree
git worktree add -b spike/search-v1 ../wt-spike-search-v1 main
git worktree add -b spike/search-v2 ../wt-spike-search-v2 main

# 在 wt-spike-search-v1 窗口里让 AI 实现方案 A
# 在 wt-spike-search-v2 窗口里让 AI 实现方案 B

# 分别跑性能测试
cd ../wt-spike-search-v1 && npm run benchmark  # 150ms
cd ../wt-spike-search-v2 && npm run benchmark  # 120ms

# 选方案 B，废弃方案 A
cd ~/project
git merge spike/search-v2
git worktree remove ../wt-spike-search-v1
git worktree remove ../wt-spike-search-v2
```

两个 worktree 同时存在，AI 在各自窗口独立工作，最后直接对比结果。失败的方案删目录即可，主仓库完全不受影响。

---

## AI 提示词模板

下面这些提示词可以直接在 VS Code Copilot、Cursor 或其他 AI 工具中使用。核心思路是：**告诉 AI 当前 worktree 的角色和边界，让它不要越界。**

### 功能开发

```text
你现在在一个 git worktree 中工作，这个目录只负责当前功能，不处理其他任务。

目标：实现 <功能名称>。
上下文：先阅读相关代码、接口和已有组件，再给出最小改动方案并直接修改代码。

要求：
1. 只修改和当前功能直接相关的文件。
2. 优先复用现有实现，不重复造轮子。
3. 不要顺手重构无关代码。
4. 修改后说明最小验证步骤。
5. 最后给出风险点和测试建议。
```

### Code Review / 分支验证

```text
你现在在一个 review 用的 git worktree 中工作，目标是评估这个分支的变更质量，而不是继续开发新功能。

请按下面顺序处理：
1. 先总结这个分支改了什么。
2. 找出潜在 bug、回归风险和边界情况。
3. 指出是否缺少测试或验证步骤。
4. 如果需要修改，只做最小修复并解释原因。

要求：关注行为变化，不要只做代码风格建议。
```

### Hotfix

```text
你现在在一个 hotfix 用的 git worktree 中工作。

目标：定位并修复 <问题描述>，优先保证快速、可验证、低风险上线。

请按下面顺序处理：
1. 根据现有代码推断最可能的根因。
2. 给出最小修复方案。
3. 说明这次修复不会扩大影响面的理由。
4. 修改后列出必须执行的验证步骤。

要求：不要引入大范围重构，不要顺手优化无关逻辑。
```

### 实验性方案 / Spike

```text
你现在在一个实验性 worktree 中工作，目标是验证 <方案描述> 是否值得推进。

请先做以下事情：
1. 总结当前实现的问题。
2. 给出最小可验证的实现切口。
3. 说明收益、风险和回滚成本。
4. 只实现最小原型，不要一次性铺开。

这是实验分支，失败了直接删目录，不影响主干。
```

---

## 在 VS Code 和 Cursor 中怎么用

这部分只简单带一下，重点是用法，不是工具介绍。

### VS Code

最推荐的方式是**一个 worktree 开一个独立窗口**：

```bash
cd ../wt-feature-123-login
code .   # 独立窗口，独立终端，独立 Copilot 对话
```

如果需要同时观察多个分支，可以用 `File → Add Folder to Workspace` 把多个 worktree 加进同一个 Multi-Root Workspace：

```json
{
  "folders": [
    { "path": "../project-repo", "name": "Main" },
    { "path": "../wt-feature-123-login", "name": "Feature 123" },
    { "path": "../wt-review-payment", "name": "Review Payment" }
  ]
}
```

注意：同一个 workspace 里的多个 worktree 共享同一个 Copilot 会话，AI 上下文不是完全隔离的。**需要 AI 专注于单一任务时，还是推荐独立窗口。**

### Cursor

Cursor 更推荐**一个 worktree 对应一个独立项目窗口**。

这样每个窗口的 Composer 历史、文件引用范围、`@codebase` 的索引范围都是独立的，AI 给出的建议不会混入其他任务的上下文。

一句话总结：

**VS Code / Cursor 只是入口，关键是"一个任务一个目录，一个目录一个 AI 上下文"。**

---

## 团队落地建议

### 目录结构

```text
project-repo/          ← 主仓库，只做 merge 和主干同步
wt-feature-123-login/  ← 功能开发
wt-review-pr-208/      ← Code Review
wt-hotfix-login/       ← 紧急修复
wt-spike-refactor/     ← 实验性探索
```

### 命名规范

```text
wt-feature-<issue>-<desc>    功能开发
wt-bugfix-<issue>-<desc>     bug 修复
wt-hotfix-<desc>             线上紧急修复
wt-review-<pr-or-branch>     代码审查
wt-spike-<topic>             实验性探索
```

### 生命周期

1. 在主工作树同步最新主干
2. 为任务创建 worktree（带上 `-b` 显式创建新分支）
3. 在新窗口中打开该目录
4. 只在这个窗口里和 AI 讨论当前任务
5. 提交、验证、合并
6. 删除 worktree，执行 `git worktree prune`

### 几条注意事项

- 不要在多个 worktree 里操作同一个分支（Git 不允许，也没有必要）
- 不要长期积压无人认领的 worktree，占空间也容易混淆
- 多个 worktree 启动 dev server 时记得配不同端口，避免冲突
- 使用 pnpm 的团队可以配置共享 store，降低重复安装成本：

```bash
pnpm config set store-dir ../.pnpm-store
```

---

## 常见问题

### Q1：删除 worktree 时提示被锁定

```bash
git worktree unlock ../wt-review-payment
git worktree remove ../wt-review-payment
```

### Q2：worktree 里 git fetch 没有更新

fetch 需要在主仓库执行，结果会共享给所有 worktree：

```bash
cd ~/project
git fetch --all  # 所有 worktree 都能看到最新远程状态
```

### Q3：删除时提示有未提交的更改

```bash
# 强制删除（未提交内容会丢失，谨慎操作）
git worktree remove ../wt-review-payment --force
```

### Q4：什么时候不需要开新 worktree

几分钟内能完成、不需要独立运行环境、不需要独立 AI 上下文的改动，直接在当前目录处理就好。

开新 worktree 的判断标准：

- 需要切换到另一个分支
- 需要保留当前开发现场
- 需要独立的 AI 上下文
- 需要并行运行服务或测试

---

## 总结

Git Worktree 不是一个"高级 Git 技巧"，而是 AI 协作开发时代很实用的基础设施。

它真正解决的不是"切分支更快"，而是：

- 让多个并行任务各自拥有稳定的工作现场
- 让 AI 的上下文不在不同任务之间串台
- 让 Hotfix、Review、功能开发、实验探索互不干扰地推进

如果团队已经在用 AI 写代码，那么下一步值得标准化的不只是提示词，而是：

**任务怎么切，目录怎么开，窗口怎么隔离，AI 怎么约束。**

---

## 参考资源

- Git 官方文档：https://git-scm.com/docs/git-worktree
- 命令帮助：`git worktree --help`

---

## 更新日志

| 版本 | 日期 | 变更 |
|------|------|------|
| v3.0 | 2026-04-26 | 以 Vibe Coding 为主线重写，融合实战场景与 AI 提示词模板 |
| v2.0 | 2026-04-26 | 重构为内部分享稿，补充 AI + Worktree 协作原则 |
| v1.0 | 2026-04-26 | 初始版本 |

最后更新：2026 年 4 月 26 日

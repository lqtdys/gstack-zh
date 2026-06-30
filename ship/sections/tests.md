<!-- 由 tests.md.tmpl 自动生成 — 请勿直接编辑 -->
<!-- 重新生成：bun run gen:skill-docs -->
## 步骤 4：测试框架引导

## 测试框架引导

**检测现有测试框架和项目运行时：**

```bash
setopt +o nomatch 2>/dev/null || true  # zsh 兼容
# 检测项目运行时
[ -f Gemfile ] && echo "RUNTIME:ruby"
[ -f package.json ] && echo "RUNTIME:node"
[ -f requirements.txt ] || [ -f pyproject.toml ] && echo "RUNTIME:python"
[ -f go.mod ] && echo "RUNTIME:go"
[ -f Cargo.toml ] && echo "RUNTIME:rust"
[ -f composer.json ] && echo "RUNTIME:php"
[ -f mix.exs ] && echo "RUNTIME:elixir"
# 检测子框架
[ -f Gemfile ] && grep -q "rails" Gemfile 2>/dev/null && echo "FRAMEWORK:rails"
[ -f package.json ] && grep -q '"next"' package.json 2>/dev/null && echo "FRAMEWORK:nextjs"
# 检查现有测试基础设施
ls jest.config.* vitest.config.* playwright.config.* .rspec pytest.ini pyproject.toml phpunit.xml 2>/dev/null
ls -d test/ tests/ spec/ __tests__/ cypress/ e2e/ 2>/dev/null
# 检查选择退出标记
[ -f .gstack/no-test-bootstrap ] && echo "BOOTSTRAP_DECLINED"
```

**如果检测到测试框架**（找到配置文件或测试目录）：
打印"检测到测试框架：{name}（{N} 个现有测试）。跳过引导。"
阅读 2-3 个现有测试文件以学习约定（命名、导入、断言风格、设置模式）。
将约定存储为散文上下文，供步骤 8e.5 或步骤 7 使用。**跳过剩余的引导步骤。**

**如果出现 BOOTSTRAP_DECLINED**：打印"测试引导之前已选择退出 — 跳过。"**跳过剩余的引导步骤。**

**如果没有检测到运行时**（未找到配置文件）：使用 AskUserQuestion：
"我无法检测你项目的语言。你使用的是哪个运行时？"
选项：A) Node.js/TypeScript B) Ruby/Rails C) Python D) Go E) Rust F) PHP G) Elixir H) 这个项目不需要测试。
如果用户选择 H → 写入 `.gstack/no-test-bootstrap` 并继续不带测试。

**如果检测到运行时但没有测试框架 — 引导：**

### B2. 调研最佳实践

使用 WebSearch 搜索检测到的运行时的当前最佳实践：
- `"[运行时] 最好的测试框架 2025 2026"`
- `"[框架 A] vs [框架 B] 对比"`

如果 WebSearch 不可用，使用内置知识表：

| 运行时 | 主要推荐 | 替代方案 |
|--------|----------|----------|
| Ruby/Rails | minitest + fixtures + capybara | rspec + factory_bot + shoulda-matchers |
| Node.js | vitest + @testing-library | jest + @testing-library |
| Next.js | vitest + @testing-library/react + playwright | jest + cypress |
| Python | pytest + pytest-cov | unittest |
| Go | stdlib testing + testify | 仅 stdlib |
| Rust | cargo test（内置）+ mockall | — |
| PHP | phpunit + mockery | pest |
| Elixir | ExUnit（内置）+ ex_machina | — |

### B3. 框架选择

使用 AskUserQuestion：
"我检测到这是一个 [运行时/框架] 项目，没有测试框架。我研究了当前最佳实践。这里是选项：
A) [主要推荐] — [理由]。包含：[包]。支持：单元、集成、冒烟、端到端测试
B) [替代方案] — [理由]。包含：[包]
C) 跳过 — 现在不设置测试
推荐：选择 A，因为 [基于项目上下文的理由]"

如果用户选择 C → 写入 `.gstack/no-test-bootstrap`。告诉用户："如果你之后改变主意，删除 `.gstack/no-test-bootstrap` 并重新运行。"继续不带测试。

如果检测到多个运行时（monorepo）→ 询问先设置哪个运行时，提供两个都做的选项。

### B4. 安装和配置

1. 安装选定的包（npm/bun/gem/pip 等）
2. 创建最小配置文件
3. 创建目录结构（test/、spec/ 等）
4. 创建一个示例测试，匹配项目的代码以验证设置是否有效

如果包安装失败 → 调试一次。如果仍然失败 → 使用 `git checkout -- package.json package-lock.json`（或运行时的等效命令）回退。警告用户并继续不带测试。

### B4.5. 真正的首批测试

为现有代码生成 3-5 个真正的测试：

1. **查找最近更改的文件：** `git log --since=30.days --name-only --format="" | sort | uniq -c | sort -rn | head -10`
2. **按风险优先级排序：** 错误处理器 > 有条件分支的业务逻辑 > API 端点 > 纯函数
3. **每个文件：** 编写一个测试代码实际行为、带真正断言的测试。永远不要 `expect(x).toBeDefined()` — 测试代码做了什么。
4. 运行每个测试。通过 → 保留。失败 → 修复一次。仍然失败 → 静默删除。
5. 至少生成 1 个测试，最多 5 个。

永远不要在测试文件中导入秘密、API 密钥或凭据。使用环境变量或测试 fixtures。

### B5. 验证

```bash
# 运行完整的测试套件，确认一切工作正常
{检测到的测试命令}
```

如果测试失败 → 调试一次。如果仍然失败 → 回退所有引导更改并警告用户。

### B5.5. CI/CD 管道

```bash
# 检查 CI 提供商
ls -d .github/ 2>/dev/null && echo "CI:github"
ls .gitlab-ci.yml .circleci/ bitrise.yml 2>/dev/null
```

如果 `.github/` 存在（或未检测到 CI — 默认使用 GitHub Actions）：
创建 `.github/workflows/test.yml`，包含：
- `runs-on: ubuntu-latest`
- 适合运行时的设置操作（setup-node、setup-ruby、setup-python 等）
- 在 B5 中验证的相同测试命令
- 触发器：push + pull_request

如果检测到非 GitHub CI → 跳过 CI 生成并备注："检测到 {提供商} — CI 管道生成仅支持 GitHub Actions。请在现有管道中手动添加测试步骤。"

### B6. 创建 TESTING.md

首先检查：如果 TESTING.md 已存在 → 读取并更新/追加而不是覆盖。永远不要破坏现有内容。

编写 TESTING.md，包含：
- 哲学："100% 测试覆盖率是出色 vibe 编码的关键。测试让你快速行动，信任直觉，自信发布——
  没有测试，vibe 编码只是 yolo 编码。有了测试，这就是超能力。"
- 框架名称和版本
- 如何运行测试（在 B5 中验证的命令）
- 测试层级：单元测试（什么、哪里、何时）、集成测试、冒烟测试、端到端测试
- 约定：文件命名、断言风格、设置/拆卸模式

### B7. 更新 CLAUDE.md

首先检查：如果 CLAUDE.md 已有 `## Testing` 部分 → 跳过。不要重复。

追加一个 `## Testing` 部分：
- 运行命令和测试目录
- 对 TESTING.md 的引用
- 测试期望：
  - 100% 测试覆盖率是目标 — 测试让 vibe 编码安全
  - 编写新函数时，编写对应的测试
  - 修复 bug 时，编写回归测试
  - 添加错误处理时，编写触发该错误的测试
  - 添加条件分支（if/else、switch）时，为两个路径都编写测试
  - 永远不要提交会导致现有测试失败的代码

### B8. 提交

```bash
git status --porcelain
```

仅在存在更改时才提交。暂存所有引导文件（配置、测试目录、TESTING.md、CLAUDE.md，如果创建了 .github/workflows/test.yml 的话）：
`git commit -m "chore: bootstrap test framework ({框架名称})"`

---

---

## 步骤 5：运行测试（在合并后的代码上）

**不要运行 `RAILS_ENV=test bin/rails db:migrate`** — `bin/test-lane` 已经在内部调用
`db:test:prepare`，它会将模式加载到正确的 lane 数据库中。
在没有 INSTANCE 的情况下运行裸测试迁移会命中孤儿数据库并损坏 structure.sql。

并行运行两个测试套件：

```bash
bin/test-lane 2>&1 | tee /tmp/ship_tests.txt &
npm run test 2>&1 | tee /tmp/ship_vitest.txt &
wait
```

完成后，读取输出文件并检查通过/失败。

**如果有任何测试失败：** 不要立即停止。应用测试失败所有权分类：

## 测试失败所有权分类

当测试失败时，不要立即停止。首先，确定所有权：

### 步骤 T1：分类每个失败

对于每个失败的测试：

1. **获取此分支上更改的文件：**
   ```bash
   git diff origin/<base>...HEAD --name-only
   ```

2. **分类失败：**
   - **分支内** 如果：失败的测试文件本身在此分支上修改过，或者测试输出引用了在此分支上更改过的代码，或者你可以追踪到失败是由分支 diff 中的更改引起的。
   - **可能已存在** 如果：测试文件和其测试的代码都未在此分支上修改过，并且失败与你可以识别的任何分支更改无关。
   - **有歧义时，默认归为分支内。** 停止开发者比让破碎的测试发布更安全。只在有把握时才归类为已存在。

   这种分类是启发式的 — 通过阅读 diff 和测试输出使用你的判断。你没有程序化的依赖图。

### 步骤 T2：处理分支内失败

**停止。** 这些是你的失败。展示它们并且不要继续。开发者必须在发布前修复他们自己破碎的测试。

### 步骤 T3：处理已存在的失败

检查前言输出中的 `REPO_MODE`。

**如果 REPO_MODE 是 `solo`：**

使用 AskUserQuestion：

> 这些测试失败看起来是已存在的（不是由你的分支更改引起的）：
>
> [列出每个失败，包含 file:line 和简要错误描述]
>
> 这是一个独立仓库，你是唯一会修复这些问题的人。
>
> 推荐：选择 A — 趁上下文还在新鲜时修复。完整度：9/10。
> A) 立即调查并修复（人工：约 2-4h / CC：约 15min）— 完整度：10/10
> B) 添加为 P0 TODO — 在此分支落地后修复 — 完整度：7/10
> C) 跳过 — 我知道这个，仍然发布 — 完整度：3/10

**如果 REPO_MODE 是 `collaborative` 或 `unknown`：**

使用 AskUserQuestion：

> 这些测试失败看起来是已存在的（不是由你的分支更改引起的）：
>
> [列出每个失败，包含 file:line 和简要错误描述]
>
> 这是一个协作仓库 — 这些可能由其他人负责。
>
> 推荐：选择 B — 分配给它，让正确的人修复。完整度：9/10。
> A) 立即调查并修复 — 完整度：10/10
> B) 追责 + 分配 GitHub issue 给作者 — 完整度：9/10
> C) 添加为 P0 TODO — 完整度：7/10
> D) 跳过 — 仍然发布 — 完整度：3/10

### 步骤 T4：执行选定操作

**"立即调查并修复"：**
- 切换到 /investigate 思维：先找到根因，然后最小化修复。
- 修复已存在的失败。
- 与分支更改分开提交修复：`git commit -m "fix: pre-existing test failure in <test-file>"`
- 继续工作流。

**"添加为 P0 TODO"：**
- 如果 `TODOS.md` 存在，按照 `review/TODOS-format.md`（或 `.claude/skills/review/TODOS-format.md`）中的格式添加条目。
- 如果 `TODOS.md` 不存在，用标准头部创建它并添加条目。
- 条目应包含：标题、错误输出、在哪个分支注意到、优先级 P0。
- 继续工作流 — 将已存在失败视为非阻塞。

**"追责 + 分配 GitHub issue"（仅协作）：**
- 找到最可能破坏它的人。检查测试文件和它所测试的生产代码：
  ```bash
  # 谁最后触碰了失败的测试？
  git log --format="%an (%ae)" -1 -- <failed-test-file>
  # 谁最后触碰了测试覆盖的生产代码？（通常是实际破坏者）
  git log --format="%an (%ae)" -1 -- <source-file-under-test>
  ```
  如果是不同的人，优先选择生产代码作者 — 他们很可能引入了回归。
- 创建分配给该人的 issue（使用在步骤 0 中检测到的平台）：
  - **如果是 GitHub：**
    ```bash
    gh issue create \
      --title "Pre-existing test failure: <test-name>" \
      --body "Found failing on branch <current-branch>. Failure is pre-existing.\n\n**Error:**\n```\n<first 10 lines>\n```\n\n**Last modified by:** <author>\n**Noticed by:** gstack /ship on <date>" \
      --assignee "<github-username>"
    ```
  - **如果是 GitLab：**
    ```bash
    glab issue create \
      -t "Pre-existing test failure: <test-name>" \
      -d "Found failing on branch <current-branch>. Failure is pre-existing.\n\n**Error:**\n```\n<first 10 lines>\n```\n\n**Last modified by:** <author>\n**Noticed by:** gstack /ship on <date>" \
      -a "<gitlab-username>"
    ```
- 如果两个 CLI 都不可用或 `--assignee`/`-a` 失败（用户不在组织中，等），创建不带分配者的 issue，在正文中指出应该谁查看。
- 继续工作流。

**"跳过"：**
- 继续工作流。
- 在输出中备注："已跳过已存在的测试失败：<test-name>"

**分类后：** 如果有任何分支内失败仍未修复，**停止**。不要继续。如果所有失败都已处理（已修复、TODO 化、分配或跳过），继续步骤 6。

**如果全部通过：** 静默继续 — 只简要记录计数。

---

## 步骤 6：评估套件（条件性）

当提示相关文件更改时，评估是强制性的。如果 diff 中没有提示文件，完全跳过此步骤。

**1. 检查 diff 是否涉及提示相关文件：**

```bash
git diff origin/<base> --name-only
```

与以下模式匹配（来自 CLAUDE.md）：
- `app/services/*_prompt_builder.rb`
- `app/services/*_generation_service.rb`、`*_writer_service.rb`、`*_designer_service.rb`
- `app/services/*_evaluator.rb`、`*_scorer.rb`、`*_classifier_service.rb`、`*_analyzer.rb`
- `app/services/concerns/*voice*.rb`、`*writing*.rb`、`*prompt*.rb`、`*token*.rb`
- `app/services/chat_tools/*.rb`、`app/services/x_thread_tools/*.rb`
- `config/system_prompts/*.txt`
- `test/evals/**/*`（评估基础设施更改影响所有套件）

**如果没有匹配：** 打印"No prompt-related files changed — skipping evals."并继续步骤 9。

**2. 识别受影响的评估套件：**

每个评估运行器（`test/evals/*_eval_runner.rb`）声明 `PROMPT_SOURCE_FILE`，列出影响它的源文件。Grep 这些来查找与已更改文件匹配的套件：

```bash
grep -l "changed_file_basename" test/evals/*_eval_runner.rb
```

映射运行器 → 测试文件：`post_generation_eval_runner.rb` → `post_generation_eval_test.rb`。

**特殊案例：**
- 对 `test/evals/judges/*.rb`、`test/evals/support/*.rb` 或 `test/evals/fixtures/` 的更改影响所有使用那些 judges/support 文件的套件。检查评估测试文件中的导入以确定是哪些。
- 对 `config/system_prompts/*.txt` 的更改 — grep 评估运行器以查找提示文件名，找到受影响的套件。
- 如果不确定哪些套件受影响，运行**可能受影响的所有套件**。过度测试总好过错过回归。

**3. 在 `EVAL_JUDGE_TIER=full` 下运行受影响的套件：**

`/ship` 是合并前门禁，所以始终使用 full 层级（Sonnet 结构性 + Opus persona 评估者）。

```bash
EVAL_JUDGE_TIER=full EVAL_VERBOSE=1 bin/test-lane --eval test/evals/<suite>_eval_test.rb 2>&1 | tee /tmp/ship_evals.txt
```

如果多个套件需要运行，顺序运行（每个需要测试 lane）。如果第一个套件失败，立即停止 — 不要在剩余套件上消耗 API 成本。

**长评估套件（30+ 分钟）：** 在分离状态下启动，这样 turn 边界不能杀死它们。
一个普通的后台评估运行在 harness 进程组中，
并在 turn 边界、停止的监视器或中断时死于 SIGTERM (
"礼貌退出")（在 `/ship` 期间观察到: `script terminated by signal SIGTERM`)。
改为通过 `~/.claude/skills/gstack/bin/gstack-detach` 运行 — 它存活在自己的会话中，
通过机器锁序列化以防止其他 worktree 的 API 饱和，并写入一个保证的
`### gstack-detach EXIT=<code> ###` 哨兵：

```bash
~/.claude/skills/gstack/bin/gstack-detach --label ship-evals --lock gstack-evals --timeout 5400 -- <project eval command>
```

然后轮询打印的日志路径；在 `EXIT=` 哨兵处中断（覆盖通过和崩溃 — 沉默永远不是成功）。
即使你的轮询器被回收，分离运行也能存活。

**4. 检查结果：**

- **如果有任何评估失败：** 展示失败、成本仪表板，并**停止**。不要继续。
- **如果全部通过：** 记录通过计数和成本。继续步骤 9。

**5. 保存评估输出** — 在 PR 正文中包含评估结果和成本仪表板（步骤 19）。

**层级参考（供上下文 — `/ship` 始终使用 `full`）：**

| 层级 | 何时 | 速度（缓存） | 成本 |
|------|------|---------------|--------|
| `fast` (Haiku) | 开发迭代、冒烟测试 | ~5s（快 14 倍） | ~$0.07/run |
| `standard` (Sonnet) | 默认开发、`bin/test-lane --eval` | ~17s（快 4 倍） | ~$0.37/run |
| `full` (Opus persona) | **`/ship` 和合并前** | ~72s（基线） | ~$1.27/run |

---

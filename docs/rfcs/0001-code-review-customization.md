# RFC 0001：Code Review Workflow 的 Per-Repo 特化

- **状态**：草稿（讨论中）
- **作者**：@lishuceo
- **最后更新**：2026-04-20
- **关联 PR**：#14（v0：dedup + max-turns 256）

## 1. 背景

`taptap/.github/.github/workflows/code-review.yml` 是一个 reusable workflow，被 org 内各 repo（`maker`、`UrhoX` 等）以 `uses:` 调用。当前架构有两个特点：

1. **零定制能力**：`inputs:` 只接受 `pr_number / is_draft / head_repo_full_name`，prompt、模型、turn 数、工具集全部写死在 org workflow 里。consumer repo 想改任何行为都得放弃复用、整段抄进自己的 workflow。
2. **prompt 已经引用了 `REVIEW.md`**：第 109 行写着 "Follow the guidelines in REVIEW.md if present"——但**没有任何 repo 真的提供这个文件**，等于死代码。

PR #14 合入后已经解决了「重复评论」和「max-turns 30 太少」这两个**通用 bug**。但项目特化的需求（C++ 内存模型 / TypeScript 业务规则 / 路径过滤 / 跑实测）仍然无法表达。

## 2. 目标与非目标

### 目标

- 让每个 consumer repo 可以**注入项目知识**（栈、gotchas、业务规则）而不抄整个 workflow
- 让 consumer repo 可以**调节关键资源参数**（model、max-turns、budget、confidence 阈值）
- 保持向后兼容：所有现有 consumer（如 maker 当前的 `uses: ...@main`）零改动继续工作

### 非目标

- **不**支持 `prompt_override`（完全替换 prompt）。如果一个 repo 真的需要完全不同的 review 行为，应该用独立 workflow（如 `security-review.yml` / `doc-review.yml`），而不是把 `code-review.yml` 撑成万能配置中心。
- **不**支持任意工具扩展（见 §6 安全模型）
- **不**在本 RFC 内引入 `@v1` `@v2` 版本 tag——单独议题（见 §8）

## 3. 设计原则

把「特化项」按**变化频率**和**抽象层级**分类，分别用最匹配的载体：

| 类别 | 例子 | 载体 | 谁维护 |
|---|---|---|---|
| 数值/枚举 | `max_turns`, `model`, `confidence_min` | workflow `inputs:` | 基础设施 |
| 路径过滤 | `paths_focus`, `paths_skip` | workflow `inputs:` | 项目 + 基础设施 |
| 工具扩展 | `Bash(pytest:*)` | workflow `inputs:`（白名单内） | 项目（受限） |
| Prompt 内容 | 项目栈、gotchas、业务规则、误报黑名单 | `.github/REVIEW.md` | 项目 |
| Review 目标 | 安全 only / 文档 only | 独立 workflow | 基础设施 |

**核心选择**：**结构性的当 input，自由文本的当文件。** 数值/枚举走 yaml input（IDE 补全、可校验、显式）；自由文本（项目知识、误报名单）走 `.github/REVIEW.md`（跟项目演进，diff 自然，无转义地狱）。

## 4. 详细设计

### 4.1 新增 workflow inputs

```yaml
inputs:
  # —— 现有 ——
  pr_number:           { required: true,  type: number }
  is_draft:            { required: false, type: boolean, default: false }
  head_repo_full_name: { required: false, type: string }

  # —— 资源 / 模型 ——
  model:
    description: Claude model alias (opus / sonnet / haiku) or full model ID.
    required: false
    type: string
    default: opus
  max_turns:
    description: Hard cap on Claude turns. Raise for large PRs, lower to bound cost.
    required: false
    type: number
    default: 256
  max_budget_usd:
    description: Hard USD cap per review run. Aborts if exceeded.
    required: false
    type: number
    default: 10

  # —— 质量门槛 ——
  confidence_min:
    description: Minimum confidence (0-100) to report a finding. Higher = less noise.
    required: false
    type: number
    default: 75
  max_findings:
    description: Maximum new inline findings per review.
    required: false
    type: number
    default: 10

  # —— 路径范围 ——
  paths_focus:
    description: |
      Comma-separated glob list. If set, only files matching these globs are reviewed.
      Example: "src/**,packages/core/**"
    required: false
    type: string
  paths_skip:
    description: |
      Comma-separated glob list of paths to ignore (in addition to built-in
      generated/lockfile/vendor skips).
      Example: "*.gen.ts,packages/legacy/**"
    required: false
    type: string

  # —— 工具扩展（受限白名单，见 §6） ——
  extra_allowed_tools:
    description: |
      Comma-separated extra Bash tool patterns to add to allowedTools.
      Each pattern must match the safe-tools allowlist in §6 of RFC 0001.
      Example: "Bash(pytest:*),Bash(cargo check:*)"
    required: false
    type: string
```

所有 input 都有默认值——现有 consumer（maker 当前 wrapper）零改动继续工作。

### 4.2 `.github/REVIEW.md` 约定

**位置**：`<consumer-repo>/.github/REVIEW.md`（与 workflow 同级，语义清晰：「这是给 CI review 看的」）。

**格式**：自由 markdown，建议但不强制以下小节：

```markdown
# Review Guidelines

## Project Context
- 栈：TypeScript + React 19 + Vite + Effect-TS
- 部署：Vercel + Cloudflare Workers
- 关键依赖版本约束：见 package.json

## Review Focus (项目特化)
- Effect 的 unhandled / non-tracked effect
- React 19 useTransition / useOptimistic 的 stale closure
- Drizzle migration 文件改动需要二次审视

## False-Positive Filter (额外的不要报告项)
- `*.gen.ts`：codegen 产物
- `packages/legacy/**`：待删除模块，新增问题不计
- `apps/*/e2e/**`：测试 fixture，不算业务逻辑

## Domain Knowledge
- "Maker" 指 maker.taptap.cn 项目
- 用户角色：creator / player / admin
- 业务规则 X：详见 docs/business-rules/X.md
```

**加载方式**：org workflow 的 prompt 当前是 "Follow the guidelines in REVIEW.md if present"（无 `.github/` 前缀，指向仓库根目录）。本 RFC 选择 `.github/REVIEW.md` 路径需要**同步更新 prompt**为 "Read `.github/REVIEW.md` if present and apply project-specific guidance from it"——见 §4.3。这是单行字符串改动，不是行为破坏。

> **设计取舍**：也可以把文件放在仓库根目录的 `REVIEW.md`，这样 prompt 字面不需要改。但 `.github/REVIEW.md` 语义更清晰（"这是给 CI 看的，不是给开发者读的项目文档"），且不污染根目录。一行 prompt 改动换更好的归位，划算。

**与 workflow inputs 的关系**：
- `paths_skip` input 是**结构化**的硬过滤（org workflow 在 prompt 里告诉 Claude "skip these globs"）
- REVIEW.md 的 "False-Positive Filter" 是**软指引**（告诉 Claude "这些路径下的问题置信度自动 -20" 之类）
- 两者职责不同：前者**绝不 review**，后者**review 但宽容**

### 4.3 在 prompt 中织入 inputs

org workflow 的 prompt 模板示例（增量部分）：

```yaml
prompt: |
  REPO: ${{ github.repository }}
  PR NUMBER: ${{ inputs.pr_number }}

  ${{ inputs.paths_focus && format('FOCUS PATHS: {0}', inputs.paths_focus) || '' }}
  ${{ inputs.paths_skip && format('SKIP PATHS (in addition to defaults): {0}', inputs.paths_skip) || '' }}

  Confidence threshold: ≥ ${{ inputs.confidence_min }}
  Max new findings: ${{ inputs.max_findings }}

  ## Step 1: Reconcile previous review comments
  ...（原内容）

  ## Step 2: Review the new changes
  ...（原内容）
  Read .github/REVIEW.md if present and apply project-specific guidance from it.

claude_args: |
  --model ${{ inputs.model }}
  --max-turns ${{ inputs.max_turns }}
  --max-budget-usd ${{ inputs.max_budget_usd }}
  --allowedTools "${{ steps.compose-tools.outputs.tools }}"
```

**两点实现细节，避免 yaml 字符串拼接陷阱：**

1. **`extra_allowed_tools` 拼接走前置 step，不走内联 `${{ }}`。** 直接 `"...,${{ inputs.extra_allowed_tools }}"` 在 input 为空时会展开成 `"...,"`（尾随逗号），可能让 Claude CLI 解析出空 pattern；更严重的是把 raw input 写进 yaml 字符串等于打开 shell 元字符注入面（见 §6）。改为：

   ```yaml
   - id: compose-tools
     env:
       # Pass via env, NOT inline ${{ }} interpolation in `run:`. Inline expansion
       # happens at YAML parse time and concatenates the raw input straight into
       # the shell script — exactly the GitHub Actions script-injection pattern
       # §6.1 is meant to block. Passing via env makes it a proper shell variable
       # that the shell never re-parses.
       EXTRA: ${{ inputs.extra_allowed_tools }}
     run: |
       BASE='Read,Glob,Grep,mcp__github_inline_comment__create_inline_comment,Bash(gh api repos/*/pulls/*/comments*),Bash(gh api graphql*),Bash(gh pr comment:*),Bash(gh pr diff:*),Bash(gh pr view:*),Bash(gh pr checks:*),Bash(git log:*),Bash(git blame:*),Bash(git diff:*)'
       # §6.4's validate-extra-tools step has already run; here EXTRA is either
       # empty or a string that passed the strict allowlist + character filter.
       if [ -n "$EXTRA" ]; then
         echo "tools=${BASE},${EXTRA}" >> "$GITHUB_OUTPUT"
       else
         echo "tools=${BASE}" >> "$GITHUB_OUTPUT"
       fi
   ```

   The same `env:` pattern (not inline `${{ }}` in `run:`) applies to the §6.4 `validate-extra-tools` step itself — defense in depth even before the validator runs.

2. **`claude_args: |` 是 YAML literal block scalar，保留换行。** `claude-code-action` 已经按行解析这个块为 args 数组（PR #14 合入的 `code-review.yml` 就是这种写法，工作正常）。RFC 这里沿用同样的格式，**不引入新的下游消费假设**。

## 5. Consumer 迁移示例

### 5.1 maker 当前

```yaml
# maker/.github/workflows/code-review.yml
jobs:
  review:
    # NOTE: @main is current convention while versioning is undecided (see §8 Q3).
    # Once a tag strategy is agreed, switch to @v1 to insulate against breaking changes.
    uses: taptap/.github/.github/workflows/code-review.yml@main
    with:
      pr_number: ${{ github.event.pull_request.number }}
      is_draft: ${{ github.event.pull_request.draft }}
      head_repo_full_name: ${{ github.event.pull_request.head.repo.full_name }}
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

新架构下**零改动**——所有新 input 都有默认值。

### 5.2 maker 想做特化（可选）

```yaml
jobs:
  review:
    # NOTE: @main is current convention while versioning is undecided (see §8 Q3).
    uses: taptap/.github/.github/workflows/code-review.yml@main
    with:
      pr_number: ${{ github.event.pull_request.number }}
      is_draft: ${{ github.event.pull_request.draft }}
      head_repo_full_name: ${{ github.event.pull_request.head.repo.full_name }}
      max_budget_usd: 15
      paths_skip: "*.gen.ts,packages/legacy/**"
      extra_allowed_tools: "Bash(pnpm tsc:*),Bash(pnpm lint:*)"
    secrets:
      ANTHROPIC_API_KEY: ${{ secrets.ANTHROPIC_API_KEY }}
```

并新增 `maker/.github/REVIEW.md` 描述项目知识。

## 6. 安全模型：`extra_allowed_tools`

让 consumer 自由扩 allowedTools 等于把执行权限的口子开给项目维护者，且 input 字符串会进入 yaml→shell 拼接链路，**两个攻击面**都要堵：

### 6.1 注入面：shell 元字符 / yaml 字符串突围

raw input 直接拼进 `--allowedTools "...,${EXTRA}"` 时，恶意 input 可以突围引号：

```
Bash(pytest:*)" --dangerouslySkipPermissions "x
```

这条字面 prefix 看起来匹配 `Bash(pytest:*)`，但展开后变成 `--allowedTools "...,Bash(pytest:*)" --dangerouslySkipPermissions "x"`，多塞了一个 flag。

**对策**：校验 step 在做 pattern 匹配**之前**先做字符级清洗——拒绝任何包含以下字符的 input：

```
"  '  `  $  \  ;  |  &  >  <  \n  \r  \0
```

逗号 `,` 是 pattern 分隔符，单独允许。其余 ASCII 控制字符也拒绝。

### 6.2 语义面：精确匹配 vs 前缀匹配

之前写的"白名单前缀列表"不够安全：consumer 传 `Bash(pytest test/../../etc/passwd)` 也能匹配 `Bash(pytest:*)` 的"前缀"。

**对策**：白名单里**每条都是完整 pattern**，input 拆分后**逐条精确字符串相等**才放行——不允许子串、不允许前缀扩展：

```
Bash(pytest:*)
Bash(jest:*)
Bash(vitest:*)
Bash(pnpm test:*)
Bash(pnpm lint:*)
Bash(pnpm tsc:*)
Bash(npm test:*)
Bash(yarn test:*)
Bash(cargo check:*)
Bash(cargo clippy:*)
Bash(cargo test:*)
Bash(go test:*)
Bash(go vet:*)
Bash(mypy:*)
Bash(ruff check:*)
Bash(eslint:*)
Bash(tsc:*)
```

`*` 是 Claude Code allowedTools 自身的通配符语义（限制到该命令的子参数），和 pattern 字符串里的字面字符一起被精确比对。

### 6.3 隐含的全局禁止

即使将来扩白名单，下列绝不允许：
- 任何带 `curl / wget / nc / ssh / scp / rsync` 的 pattern（外联）
- 任何带 `rm / mv / cp / chmod / chown` 的 pattern（写文件系统）
- 任何 `gh api ... -X POST/PUT/PATCH/DELETE` 形式的 mutation（除已经在 base allowedTools 里、专门授给 dedup/resolve 用的两条）
- `Bash(*)` 这种全开通配

### 6.4 实现位置

org workflow 在 reusable workflow 入口加一个 `validate-extra-tools` step：

1. 读 `inputs.extra_allowed_tools`
2. 字符级清洗（§6.1），任何禁字符 → `exit 1` 报错指向本节
3. 按 `,` 拆分，逐条与白名单**精确相等比对**，任何不匹配 → `exit 1`
4. 通过后导出干净字符串给后续 step 使用（见 §4.3 的 `compose-tools`）

错误信息明确指向本 RFC §6，并打印被拒绝的 pattern。

### 6.5 审计

每次 review run 把最终生效的 allowedTools 打到 log（PR #14 合入的 workflow 已经在打）。`validate-extra-tools` step 也把 input/output 都记录到 log。

## 7. 兼容性与版本

- 所有新 input 都有默认值 → 现有 consumer 零改动
- prompt 模板增量为追加（保持原 Step 1/2/3 结构），不破坏行为

**默认值与现行 workflow 的对照**：

| Input | 默认值 | 与 PR #14 合入后的 workflow |
|---|---|---|
| `model` | `opus` | 一致（现行就是 `--model opus`） |
| `max_turns` | `256` | 一致（PR #14 已经设到 256） |
| `max_budget_usd` | `10` | **新增**（现行没有 budget 上限） |
| `confidence_min` | `75` | **形式上新增**——现行 workflow 在 prompt 里写死了 "Only report findings with confidence ≥ 75"，本 RFC 把它从 prompt 字面提到 input 参数化 |
| `max_findings` | `10` | **形式上新增**——同上，现行 prompt 里写死 "at most 10 new findings" |
| `paths_focus / paths_skip / extra_allowed_tools` | 空 | **新增**，空时不影响行为 |

唯一行为变化是引入 `max_budget_usd: 10` 上限——超出会中止 review。`10 USD/run` 应当远高于实际开销（参考 maker#439 一次 ~4min × opus 远低于 $1），属于安全网而非常态约束。如担心切换震动，可在 Phase 1 默认 `0`（无上限）后续再调高。

**未来破坏性改动**：见 §8。

## 8. 开放问题（需要决策）

1. **`REVIEW.md` 位置**：根目录还是 `.github/REVIEW.md`？
   - **倾向 `.github/`**：和 workflow 在一起，"这是给 CI 看的"语义清晰；不污染根目录。
   - 但根目录更易被人发现。

2. **结构化配置文件**：要不要支持 `.github/code-review.yml`（分段 `paths_skip / extra_tools / extra_prompt`）？
   - **本 RFC 倾向暂不引入**：先用 input + REVIEW.md 跑 3-6 个月，等真有 3+ repo 抱怨"prompt 切片不够精确"再升级。
   - YAML 配置的优势是机器可解析（防 Claude 自己漏读 REVIEW.md），但增加维护负担。

3. **版本 tag 策略**：现在所有 consumer 用 `@main`。
   - 加 input 是 backward-compatible，可以不发 tag 直接合
   - 但**未来**如果要破坏性改动（比如改默认 `confidence_min: 75 → 80`），是否要发 `@v1` `@v2`？
   - 建议**本 RFC 不引入版本 tag，但记录隐患**。下次破坏性改动时再讨论。

4. **`extra_allowed_tools` 白名单维护**：
   - 白名单写在哪里？
     - (a) 嵌在 org workflow 里（一份字符串）
     - (b) `.github/code-review-allowed-tools.txt`（一行一条）
     - 倾向 **(b)**——审查友好，新增工具不动 yaml。
   - 谁有权限改？建议要求 PR + org owner approval。

5. **REVIEW.md 缺失时的提示**：
   - 当 consumer 没有 REVIEW.md 时，要不要在 review 汇总里**首次**提示 "建议添加 .github/REVIEW.md 描述项目知识，可以提升 review 精度"？
   - 噪音 vs 价值，倾向**首次提示一次**（用 inline comment 或 PR comment 的特殊标记记忆"已提示过"）。

## 9. 实施步骤

如果本 RFC 通过：

1. **Phase 1**（小步骤，独立 PR）
   - 加 `model / max_turns / max_budget_usd / confidence_min / max_findings` 5 个 input，prompt 织入。
   - 这是最低风险变化（纯参数化），现有行为完全等价。

2. **Phase 2**（独立 PR）
   - 加 `paths_focus / paths_skip` input，prompt 加路径过滤指令。

3. **Phase 3**（独立 PR）
   - 加 `extra_allowed_tools` + 白名单校验 step + 白名单文件。

4. **Phase 4**（文档 PR）
   - 在 README.md 写 REVIEW.md 模板和 consumer 迁移示例。
   - 通知 maker / UrhoX 等团队按需创建 REVIEW.md。

每个 phase 独立 mergeable，每步都向后兼容。

## 10. 替代方案（已考虑但未采用）

- **A：所有定制走 input**——见 §3 已分析，inputs 爆炸 + yaml 写多行 prompt 痛苦，不采纳。
- **B：所有定制走约定文件**（`.github/code-review-config.yml` 全量配置）——隐式发现差，数值参数还要走 YAML 解析层，复杂度不划算。
- **C：fork 一份 workflow per repo**——已经在用 reusable 的初衷就是避免这个。
- **D：`prompt_override` 完全自定义 prompt**——见 §2 非目标。

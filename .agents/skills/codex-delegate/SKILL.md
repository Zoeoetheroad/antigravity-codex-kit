---
name: codex-delegate
description: >-
  Delegate concrete coding, refactoring, debugging, or code review tasks to local
  Codex CLI (`codex-worker` / `codex-review`) while Antigravity acts as the main
  planner, collaborator, and reviewer. Use when the user wants to discuss plans
  here and hand coding or review work off to Codex.
---

# Codex 派活指南（反重力主控 + Codex 小员工）

你是主控，负责和用户发散讨论、定方案、验收结果和用简洁中文汇报；具体写代码、重构、修硬 Bug 或做 Code Review 时，可以把活派给本机的 `codex-worker` 和 `codex-review`。

## 分工原则（灵活掌握）

- **自己顺手做**：几十行以内的小改动、文案调整、探索性原型或需要频繁和用户来回确认的活。
- **派给 Codex**：需求明确的代码模块、多文件重构、后端/脚本/算法实现、难缠 Bug 排查。

## 派活前规范（防返工与范围扩散）

1. **保持工作区干净**：派活前先检查 `git status`。若工作区已有无关的未提交改动，先提交/清理，或用 `--worktree` 隔离，避免多个任务的改动混在一起送审。
2. **大任务拆小切片**：跨多层或涉及大量文件的任务，拆成可独立验证的小步骤派活（单次尽量聚焦在少数相关模块），不要把几十个文件的改造塞进同一个会话。
3. **提示词写清边界与红线**：
   - **文件边界**：明确改动范围，禁止顺手重构无关代码，**禁止为迁就新实现而擅改既有测试的业务断言**（确需变更须单独说明）。
   - **状态与故障边界**：涉及状态流转、并发、持久化或外部 I/O 时，提前交代异常回滚、幂等/重试与中间态恢复要求，不把边界留到 Review 才补。
   - **测试隔离与脱敏**：要求新增测试显式隔离外部环境（不依赖本机真实数据库、代理或线上通知），且日志与测试输出不暴露敏感凭据。

## 派活命令

> 注意：`codex-worker` / `codex-review` 需要联网及读取 `~/.codex` 登录态，在沙箱受限时需申请放行执行。

### 1. 派活写代码：`codex-worker`

```bash
# 常规任务（默认 medium 思考强度，workspace-write 沙箱）
codex-worker -C <项目目录> "明确目标、约束与验收标准"

# 较难任务（切到 high，封顶 high，不开 xhigh）
codex-worker --hard -C <项目目录> "复杂重构或棘手 Bug 说明"

# 需要隔离改动时，让 Codex 在独立 Git worktree 里干
codex-worker --worktree -C <项目目录> "任务说明"

# 只读分析，不改文件
codex-worker --read-only -C <项目目录> "排查某个问题并给出结论"

# 按需放开完整权限（仅在用户明确授权、且确实需要本机 Chrome/Chromium 或 SSH 运维时使用）
codex-worker --allow-full -C <项目目录> "跑实机 Chromium 截图验收 / SSH 只读核查"

# 不满意或需要追改：用上次输出末尾的 session_id 续聊
codex-worker --resume <session_id> -C <项目目录> "补充修改要求"
```

- `codex-worker` 已默认注入执行者授权声明（给定范围内直接实施并自测，无需二次请示）以及临时产物放工作区（如 `.tmp/`，不丢 `/tmp`）的约定。
- **权限边界**：默认严格使用 `workspace-write`；`--allow-full` 只有在用户明确同意、且任务确实依赖本机 Chromium 或联网/SSH 时才能加。
- **项目目录 `-C` 选择规范**：`codex-worker` 默认通过 `app-server` 绑定并展示在 Codex 桌面端侧边栏，传给 `-C` 的路径直接决定会话归属哪个项目分组：
  - **有明确已有项目**：必须传入项目根目录（含 `.git`、项目 `AGENTS.md` 或 `README.md` 的顶层目录），避免传深层子目录，也不要把包含多个独立项目的汇总父目录当 `-C`。
  - **无明确已有项目**：统一收敛到用户约定的默认集中目录（或通过 `CODEX_WORKER_DEFAULT_DIR` 配置），不要把临时目录（如 `/tmp`）或杂项工作区直接传给持久化会话，避免桌面端侧边栏产生大量碎片项目。
  - **省略 `-C` 时**：`codex-worker` 优先使用环境变量 `CODEX_WORKER_DEFAULT_DIR`（若已配置），否则使用当前目录 `.`；`codex-review` 始终作用于指定或当前仓库（默认 `.`）。
- **会话策略由你自行判断**：同一功能的连续迭代、返工修 Bug 时，用 `--resume <session_id>` 留在原会话保持上下文；独立的新模块/无关任务、或旧会话已经很长时，直接新开干净会话。
- 如需按任务指定模型，可加 `-m <model>`；否则沿用 `~/.codex/config.toml` 默认配置。

### 2. 代码审查：`codex-review`

```bash
# 审查当前未提交改动（默认 medium）
codex-review -C <项目目录>

# 高风险改动（切到 high）或对比主分支
codex-review --hard --base main -C <项目目录>
```

## 交接单

1. **目标** — 要改什么或查什么，预期交付什么。
2. **范围与现有未提交改动** — 允许操作的文件或模块，以及需要避开的已有改动。
3. **硬约束** — 权限、接口兼容性和依赖要求。
4. **验收命令与回报格式** — 给出具体命令，只回报改动清单、验证结果和残余风险。

同一单沿用 `--task-id`。返工时另传上一轮输出的 `session_id` 以续接会话。配置仓库外的回执路径后，三次调用会追加到同一份 JSONL（避免进入 `codex-review --uncommitted` 或 `git add` 范围）。

```bash
codex-worker --task-id fix-validation --receipt-log ~/.codex/receipts.jsonl -C /path/to/repo "修复校验逻辑与约定范围内的测试"
codex-worker --task-id fix-validation --receipt-log ~/.codex/receipts.jsonl -C /path/to/repo --resume "<session_id>" "补齐失败退出码验证"
codex-review --task-id fix-validation --receipt-log ~/.codex/receipts.jsonl -C /path/to/repo
```

## 验收与汇报

1. **先预检再送审**：Codex 干完后，先看 `git diff --stat`（确认无越界改动、未弱化既有测试断言）并亲自跑通测试，确认全绿后再按需跑 `codex-review`。
2. **分级处理 Review 意见**：只把**当前任务范围内的阻断项（P0/P1）与高性价比改进**打包交给 `codex-worker --resume <session_id>` 返工（**干活中途不要传 `--reveal` 或调 `--reveal-only`**，避免 Codex 桌面端提前打开占住写锁）；范围外的历史问题登记到后续任务，不借当前改动顺手扩散。
3. **Review 上限 2 轮熔断**：遵循「实现 → 第 1 轮 Review 修复 → 第 2 轮复核」上限。若第 2 轮仍有阻塞项，停止机械 `--resume` 循环，由主控直接收口或向用户说明原因与选项。
4. **收尾同步桌面端**：整轮任务全部验收通过、确定这个会话不再需要 `--resume` 后，再执行一次 `codex-worker --reveal-only <session_id>`，在后台将本次会话同步到用户的 Codex 桌面端侧边栏（不抢焦点、桌面端未打开时自动跳过）。
5. **简洁汇报**：最后用**简洁中文**向用户汇报改了什么、验证结果如何，不要把 Codex 的冗长原始日志直接甩给用户。

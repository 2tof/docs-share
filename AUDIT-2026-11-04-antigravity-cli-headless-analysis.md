# Antigravity CLI (`agy`) Headless 程序化调用 —— 契约分析与实现方法

- **日期**：2026-11-04
- **主题**：从 Google 官方文档出发，分析 `agy` headless 模式"如何被程序调用"的方法与判据
- **证据等级**：一手。全部引用自官方文档 Markdown 源与官方条款页服务端原文，非社区转述
- **一手来源**：
  - <https://antigravity.google/docs/cli/headless>（正文源：`https://antigravity.google/docs/cli/headless.md`，549 行，2026-11-04 抓取）
  - <https://antigravity.google/terms>（Additional Terms，服务端渲染正文，2026-11-04 抓取）
- **范围声明**：本文**只做分析方法与契约梳理**。不涉及本机部署、不涉及任何安装动作、不做配额消耗测试。

---

## 0. 结论摘要

1. `agy` 的 headless 模式是**官方设计**的机器调用接口，文档原话即 "Run Antigravity CLI non-interactively to script agent tasks, integrate with CI pipelines, and capture machine-readable output."
2. 它提供了**完整的工程契约**：输出通道分离、三级输出格式、结构化 schema 强制、七态 status、显式退出码、软拒绝语义、双工 NDJSON 会话。
3. 可被程序调用的关键**不是"能跑命令"**，而是**每一层都有明确的失败契约**——这正是它能安全进编排的前提。
4. 有 3 个**默认值是陷阱**：`--print-timeout 0s`（无限等待）、soft-deny 时 `exit 0`（假成功）、metadata 计数是**会话累计**而非单轮。
5. **技术契约 ≠ 条款授权**。headless 文档证明的是"用程序调用 agy 客户端"被支持；它不能推出"把 Antigravity 配额当作第三方框架模型后端"也被允许。两者应分开陈述（见 §8）。

---

## 1. 三层调用模型（先确定"状态归属"）

官方把程序化调用拆成三层，**本质区别是状态放在哪里**。这是设计接入方式时必须先回答的问题。

| 层次 | 机制 | 状态归属 | 适用 |
|---|---|---|---|
| 单次调用 | `-p` / `--print` / `--prompt` | **无状态**（文档明确："Headless runs are stateless by default"） | 一问一答、任务彼此独立 |
| 多轮会话（跨进程） | `--continue` / `-c`、`--conversation <id>` | 状态在 **`conversation_id`** | 低频、需要延续上下文 |
| 多轮会话（单进程双工） | `--input-format stream-json` + stdin | 状态在 **进程** | 高频、需要动态决定下一问 |

**关键判断**：文档对双工模式的评价是"skip startup overhead and reuse the warmed-up conversation. This makes it significantly faster than running repeated commands"。因此——

- 低频、任务独立 → 单次 `-p`
- 需要上下文但调用不频繁 → `--conversation`
- **高频编排 → 长驻进程 + stdin NDJSON**，这是官方给出的唯一性能路径

---

## 2. 输出通道分离：机器可读的根本前提

官方原文：

> "The response goes to `stdout`. Diagnostics—errors, authentication prompts, progress, and permission notices—go to `stderr`. This split keeps the captured response clean."

**分析含义**：程序可以**无歧义捕获结果**，不需要从日志里正则抠答案。这条看似基础，但它是整个"可编程"属性成立的第一个前提。任何接入设计若把 stdout/stderr 合并（例如 `2>&1`）都会破坏该契约。

---

## 3. 三级输出格式

`--output-format` 接受三个值：

| Format | stdout 形态 | 用途 |
|---|---|---|
| `text`（默认） | 纯响应文本，无包装 | 人读、快速脚本 |
| `json` | 完成时打印**一个** JSON 对象 | 取结果 + 元数据 |
| `stream-json` | NDJSON 事件流 | 监控进度、工具调用、token 用量 |

### 3.1 `json` 信封字段（官方表）

| 字段 | 类型 | 说明 |
|---|---|---|
| `conversation_id` | string | 会话 ID，可用于后续续接 |
| `status` | string | 终态（见 §6） |
| `response` | string | agent 的自由文本响应 |
| `error` | string | 错误信息；**仅失败时出现** |
| `duration_seconds` | number | 墙钟时长 |
| `num_turns` | number | 会话中的用户轮数 |
| `structured_output` | object | 解析后的 schema 输出；**仅配合 `--json-schema` 出现** |
| `json_schema` | object | 实际强制执行的 schema；同上 |
| `usage` | object | `input_tokens` / `output_tokens` / `thinking_tokens` / `cache_read_tokens` / `total_tokens` |

**分析含义**：调用方**不需要解析自然语言**。分流依据是 `status`（成败）、`error`（原因）、`num_turns`（轮数）、`usage`（成本）、`structured_output`（业务数据）。

### 3.2 结构化输出两级机制

- **`--json-schema`**：CLI 强制模型输出符合 schema，结果在 `.structured_output`，另有 `.response` 字符串副本。
  - 值可以是 schema 字符串，也可以是**文件路径**。
- **`--output-format json`**：稳定信封（上表）。

两者可叠加：`--json-schema` 决定"内容形状"，`--output-format json` 决定"传输形状"。

---

## 4. `stream-json` 事件模型

### 4.1 三段式序列（官方表）

| event | 何时发出 |
|---|---|
| `init` | 流开始时，**一次** |
| `step_update` | 每次步骤转换或文本增量 |
| `result` | 结束时，**一次**（与 `json` 格式**同形**） |

### 4.2 `init` 载荷

| 字段 | 类型 | 说明 |
|---|---|---|
| `cwd` | string | 工作目录 |
| `tools` | string[] | **全部可用工具名** |
| `permission_mode` | string | **生效的权限模式** |
| `model` | string | 仅在 `--model` 指定时出现 |
| `agent` | string | 仅在 `--agent` 指定时出现 |
| `json_schema` | object | 仅在 `--json-schema` 指定时出现 |

**这是编排上最有价值的设计**：`permission_mode` 和 `tools[]` 在**开工前**就暴露，调用方可以在第一行事件里确认"本次运行有什么权限、有哪些工具"，不必等出错才发现配置不对。

`permission_mode` 默认 `request-review`；加 `--dangerously-skip-permissions` 后为 `always-proceed`。

### 4.3 `step_update` 载荷

| 字段 | 类型 | 说明 |
|---|---|---|
| `conversation_id` | string | 会话 ID |
| `step_index` | number | 0 基步骤序号 |
| `state` | string | `ACTIVE`（运行中）/ `DONE`（完成） |
| `step_type` | string | 如 `agent_response`、`tool`（另观察到 `user_input`、`checkpoint`） |
| `tool_name` | string | 规范工具名，仅工具步骤 |
| `text_delta` | string | **增量响应文本** |
| `duration_seconds` | number | 步骤时长（已知时） |
| `usage` | object | **每步** token 用量（已知时） |
| `tool_info` | object | 工具调用详情 |
| `subagent_info` | object | 子代理调用详情 |

---

## 5. 双工模式（`--input-format stream-json`）细则

### 5.1 输入协议

写一行一个 JSON 对象到 stdin，`event` 字段指明类型。一个 prompt 是带 `content` 的 `user` 事件：

```json
{"event":"user","message":{"content":"Reply with exactly the word: apple. Nothing else."}}
```

`content` 接受**字符串**或**text 块列表**，两种等价：

```json
{"event":"user","message":{"content":"Reply with exactly: banana"}}
{"event":"user","message":{"content":[{"text":"Reply with exactly: banana"}]}}
```

**`text` 是唯一支持的块类型。** 提交其他类型会**以错误结束会话**，而非静默丢弃。官方给出的理由：

> "This ensures the agent never answers a prompt you didn't explicitly send."

⇒ **能力边界：此模式下多模态输入不可用。**

### 5.2 官方驱动范例（Python）

文档直接给出了持有 stdin 管道的写法。要点是**按 `event` 字段分流，等到当前轮 `result` 再写下一条**：

```python
import json, subprocess

proc = subprocess.Popen(
    ["agy", "--input-format", "stream-json", "--output-format", "stream-json"],
    stdin=subprocess.PIPE, stdout=subprocess.PIPE, text=True, bufsize=1,
)

def ask(prompt):
    proc.stdin.write(json.dumps({"event": "user", "message": {"content": prompt}}) + "\n")
    proc.stdin.flush()
    for line in proc.stdout:
        event = json.loads(line)
        if event["event"] == "result":
            return event["result"]["response"]

first = ask("Name one popular version control system. Answer with one word.")
print(ask(f"Name a competitor to {first.strip()}. Answer with one word."))

proc.stdin.close()
proc.wait()
```

**结束方式**：关闭 stdin。进程在输入管道关闭且当前轮完成后退出。若写完最后一问立即关闭，**仍能收到最后的 `result`**。正常会话退出 `0`。

### 5.3 累计 vs 单轮（官方表）——重要陷阱

| 字段 | 作用域 |
|---|---|
| `response` | 发出它的**那一轮** |
| `num_turns` | **会话累计** |
| `usage` | **会话累计** |
| `duration_seconds` | **会话累计** |

⇒ 双工模式下**不能把 `num_turns`/`usage` 当单轮指标**。取当前轮文本只能用 `response`。要做每轮成本核算，必须**自行按 `result` 事件做差分**。

### 5.4 不支持的消息（官方表）

| 输入 | 结果 | 退出码 |
|---|---|---|
| 无法识别的 `event` 名 | 跳过，stderr 记警告 | — |
| `control_request` / `control_response` | `ERROR` result，会话结束 | `2` |
| CLI 直接处理的斜杠命令（如 `/model`） | `ERROR` result，会话结束 | `2` |
| 缺 `event` 字段的消息 | `ERROR` result，会话结束 | `1` |
| 非法 JSON 行 | `ERROR` result，会话结束 | `1` |
| 非 `text` 的内容块类型 | `ERROR` result，会话结束 | `1` |

设计意图（官方）：无法识别的 event 被安全跳过，使**针对新版协议写的应用在旧版 CLI 上不会崩溃**（前向兼容）。其余错误立即终止会话；此前已完成的轮次保留其 `result`，但坏行会中止后续。

⚠️ `/model`、`/usage` 这类 CLI 自答命令**不能走流式**，必须作为独立的 `agy -p /model` 调用。

### 5.5 官方列出的 4 个常见错误

| 错误 | 失败原因 | 修正 |
|---|---|---|
| 与 `--output-format json`/`text` 配对 | 那些格式**只在进程退出时**输出一个信封，除最后一轮外**全部丢失** | 输入流式必须配输出 `stream-json` |
| 用 `-p` 传 prompt | 流式模式**只监听 stdin**，命令行 flag 传的 prompt 被丢弃 | 作为 `user` 消息写进 stdin |
| 把 `/model`、`/usage` 塞进流 | CLI 在事件流之外自行处理，破坏 JSON 流 | 单独跑 `agy -p /model` |
| 把 `num_turns` 当单轮计数 | 元数据是**会话累计** | 用 `response` 取当前轮文本 |

---

## 6. 退出码与 status 七态

**成功退出 `0`**。未能产出响应则非零退出并把原因写 stderr；`json`/`stream-json` 模式下失败还体现在 `status` 与 `error` 字段。

| Status | 含义 |
|---|---|
| `SUCCESS` | 完成并产出响应 |
| `ERROR` | 以错误结束 |
| `CANCELED` | 被取消 |
| `INTERRUPTED` | 被打断（如 `SIGINT`） |
| `INVALID` | 进入非法状态 |
| `WAITING` | 结束时仍在等待输入 |
| `RUNNING` | 未达终态 |

**分析含义**：
- `WAITING` / `RUNNING` 的存在说明"**没有终态本身是一种可观测结果**"，不能简单等同失败。只看 `exit 0` 会把它们误判为成功。
- 判失败必须**三重信源**：退出码非 0 **或** `status != SUCCESS` **或** stderr 有原因。**单一信源都不可靠**。

### 显式失败 > 静默回退（官方设计取舍）

> "Unlike the interactive UI, headless mode doesn't silently fall back when `--model` names an unknown model. It exits non-zero with an `ERROR` status so a pinned pipeline fails loudly instead of running the wrong model."

示例（官方）：未知模型 → `exit=1`，信封 `status:"ERROR"`，`error` 中列出可用模型。

**分析含义**：静默回退会让流水线"看起来成功、实际用错模型"，是编排系统最难排查的故障。官方在此**明确选择 fail loudly**，这是它能进 CI 的关键属性。

### 超时默认无限

> "By default, a run has no time limit and waits until the turn completes."

`--print-timeout` 默认 `0s` = **无限制**。这是接入时**必须主动处理**的默认值：无上限的外部 agent 调用会让上游永久等待。官方自己的 CI 示例就把它钉成 `10m`。

---

## 7. 权限模型：headless 下决定"能不能真干活"

官方原文：

> "There is no interactive prompt in headless mode, so tools that would normally ask for confirmation are handled by policy."

- 默认遵循 settings 中的执行/权限模式。
- **`auto` 模式**（个人账号在终端沙箱可用时的默认值）：工作区文件编辑、沙箱化或安全的可逆命令**自动执行**；危险或不可逆操作**仍需批准**。
- **`auto` 之外**：shell 命令等默认为 **Ask**。
- **软拒绝语义**：任何需要批准却拿不到批准的工具动作 → **运行继续、退出 `0`**、在 stderr 打印提示（含工具名与授权方法）。

### ⚠️ 最大的假成功陷阱

**`exit 0` ≠ 任务真的做完。** 程序可能只完成了"思考"，所有写文件/执行命令**都被静默拒绝**。

⇒ 接入设计必须把**"退出码 0"与"产物真的落地"分开验证**：检查 stderr 的 soft-deny 提示，或独立校验副作用产物。

### 两条授权路径

**细粒度（推荐长期使用）**——`~/.gemini/antigravity-cli/settings.json`：

```json
{
  "permissions": {
    "allow": [
      "command(git)",
      "command(regex:npm run (build|lint|test))",
      "write_file(src/)"
    ]
  }
}
```

规则形状为 `action(target)`；MCP 同理为 `mcp(server/tool)` / `mcp(server/*)` / `mcp(*)`。

**全量（一次性）**——`--dangerously-skip-permissions`。官方警告：

> "`--dangerously-skip-permissions` approves all tool calls, including file writes and command execution. Prefer scoped `permissions.allow` rules unless you fully trust the prompt and environment."

⇒ **scoped 规则适合长期编排；危险开关只适合一次性、且完全信任 prompt 与环境的场景。**

---

## 8. 认证：是"前置状态"，不是"配置项"

> "Headless mode uses your cached credentials. Authenticate once with an interactive session first. In a non-interactive environment with no terminal (for example, CI), a run that isn't already authenticated exits with an `authentication required` error instead of hanging."

**分析含义**：
- headless 调用**有前置状态**：必须先有一次交互式登录，凭据缓存在本机。
- "报错而非挂起"是好设计（挂起比报错难排查）。
- 任何接入方案必须回答三问：**前置登录由谁满足？凭据过期怎么恢复？多机部署下凭据算谁的？**

---

## 9. 选择模型 / effort / agent

```bash
agy models     # 列出当前可用 slug
agy agents     # 列出可用 agent
```

官方示例 slug（随版本变化，以 `agy models` 为准）：`gemini-4-argon-high`、`gemini-3.8-flash-high`、`gemini-3.8-flash-medium`、`gemini-3.7-flash-high`、`gemini-3.6-flash-high`、`gemini-3.1-pro-high` 等。

- `--model <slug>` 钉死模型
- `--effort low|medium|high` 推理强度
- `--agent <name>` 选择 agent

**注意**：与交互式不同，headless **不会**在未知模型时静默回退（见 §6）。

---

## 10. 完整 flag 参考（官方表，逐字）

| Flag | 默认 | 说明 |
|---|---|---|
| `-p`, `--print`, `--prompt` | — | 非交互跑单条 prompt 并打印响应 |
| `--output-format` | `text` | 输出格式：`text`、`json`、`stream-json` |
| `--input-format` | `text` | 输入格式：`text` 或 `stream-json`；后者从 stdin 读 |
| `--json-schema` | — | schema 字符串或文件路径，强制结构化输出 |
| `--model` | — | 本次运行的模型 slug（见 `agy models`） |
| `--effort` | — | 推理强度：`low`、`medium`、`high` |
| `--agent` | — | 本次运行的 agent（见 `agy agents`） |
| `--continue`, `-c` | `false` | 继续最近一次会话 |
| `--conversation` | — | 按 ID 恢复会话 |
| `--dangerously-skip-permissions` | `false` | 自动批准所有工具权限请求 |
| `--print-timeout` | `0s` | 等待响应的最长时间（`0s` 表示无限制） |
| `--sandbox` | `false` | 启用终端沙箱限制运行 |

---

## 11. 官方 CI 示例（逐字）

```bash
#!/usr/bin/env bash
set -euo pipefail

result=$(agy -p "Name three popular version control systems, comma-separated." \
  --output-format json \
  --print-timeout 10m)

status=$(echo "$result" | jq -r '.status')
if [[ "$status" != "SUCCESS" ]]; then
  echo "Agent run failed: $(echo "$result" | jq -r '.error')" >&2
  exit 1
fi

echo "$result" | jq -r '.response' > result.txt
```

注意官方示例本身做了三件事：**钉超时**、**查 status 而非只看退出码**、**从 `.response` 取结果**。

---

## 12. 判据流程：评估任一接入设计是否成立

按顺序逐项排除，任一不满足即设计有缺陷：

| # | 判据 | 依据 | 失败后果 |
|---|---|---|---|
| 1 | **状态放哪**：无状态 / `conversation_id` / 长驻进程 | §1 | 上下文丢失或开销巨大 |
| 2 | **结果怎么取**：必须走 stdout / json 信封 | §2 | 从日志抠答案，脆弱 |
| 3 | **失败怎么判**：退出码 + `status` + stderr 三重 | §6 | 把失败当成功 |
| 4 | **超时**：必须显式设 `--print-timeout` | §6 | 上游永久挂起 |
| 5 | **权限**：先读 `init.permission_mode` 与 soft-deny | §4.2, §7 | 静默没干活 |
| 6 | **副作用验证**：`exit 0` 后独立校验产物 | §7 | 假成功 |
| 7 | **认证前置**：确认凭据缓存有效、明确过期恢复 | §8 | CI 里报 auth 错误 |
| 8 | **成本核算**：双工下需自做差分（累计字段） | §5.3 | 成本统计失真 |

### 最小验证脚本设计（不消耗配额的部分）

以下步骤**在真正调用模型前**即可完成大部分验证：

1. `agy --version` —— 确认版本（**避免在 ≤1.1.10 上误跑 `-p /usage`，旧版会把它当 prompt 真实消耗配额**）。
2. `agy models` —— 拿到真实 slug 列表，避免拼错模型名触发 fail-loudly。
3. `agy --help` —— 核对本文 flag 与本机版本是否一致（文档可能领先/落后于已装版本）。
4. 单次极短 prompt + `--output-format json --print-timeout 2m` —— 验证信封形状、`status`、`usage`。
5. `--output-format stream-json` 单次 —— 核对 `init` 的 `tools[]` 与 `permission_mode` 是否符合预期。
6. 写一个 `permissions.allow` 里**没有**授权的动作 —— 验证 soft-deny 的 `exit 0` + stderr 提示行为，这是最容易误判的路径。

---

## 13. 边界：技术契约 ≠ 条款授权

**必须分开陈述的两件事：**

1. **headless 文档**证明：Google **支持用程序调用 `agy` 这个客户端**（script agent tasks / integrate with CI pipelines）。这是工程合法性依据，也是本文全部内容的来源。
2. **Antigravity 附加条款第 6 条**（服务端原文，2026-11-04 抓取）约束的是另一件事：

   > "You must not abuse, harm, interfere with, or disrupt the Service. This includes, but is not limited to, using the Service in connection with products not provided by us. **Using third party software, tools, or services to access the Service (e.g. using OpenClaw with Antigravity OAuth) is a breach of this Agreement.** Such actions may be grounds for suspension or termination of your Antigravity and/or Gemini CLI accounts."

**分析结论**：
- 两者技术形态相似（都是"程序驱动 agy"），但**语义上是两件事**：headless 文档描述**客户端的使用方式**；条款第 6 条约束**用第三方软件去 access the Service**。
- `agy` 是 Google 自己的客户端，所以让 agy 访问 Antigravity 字面上不构成 "third party software access the Service"；但 "products not provided by us" 的覆盖面很宽。
- 官方**没有**就"第三方 harness 驱动官方 agy"给出明确书面回应（Google 开发者论坛上此类提问自 2026-07 起反复出现，官方账号未作答复）。
- 因此：**不要因为"官方文档写了支持 CI"就推断"第三方框架接进来也完全没问题"**。条款问题必须单独评估，本文不替它下结论。

---

## 14. 未决与待核实

| 项 | 状态 | 说明 |
|---|---|---|
| 本机 `agy` 版本 | **未核实** | 本文全部基于官方文档，未在本机验证版本与 flag 一致性 |
| 文档 flag 与本机版本差异 | **未核实** | 官方文档可能领先或落后于已装版本，需以 `agy --help` 为准 |
| 实际配额消耗特征 | **未验证** | 刻意未做任何调用测试 |
| 条款对"第三方驱动官方 CLI"的官方立场 | **无官方答复** | 见 §13 |
| `--sandbox` 与 `auto` 模式的交互细节 | 部分覆盖 | 文档指向 `/docs/cli/permissions` 的完整规则语法，本文未展开 |

---

## 15. 参考

- Headless 模式官方文档：<https://antigravity.google/docs/cli/headless>（Markdown 源 `…/headless.md`）
- CLI 权限完整规则：<https://antigravity.google/docs/cli/permissions>
- Antigravity 附加条款：<https://antigravity.google/terms>
- Plugin Marketplace 官方博客：<https://antigravity.google/blog/plugin-marketplace-in-antigravity>

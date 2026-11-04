# Antigravity 官方 MCP 市场案例与合法使用边界分析

- **日期**：2026-11-04
- **主题**：从 Google 官方 MCP Store / Plugin Marketplace 的真实案例出发，界定"什么是不越界的合法使用"
- **证据等级**：一手。全部来自 Google 官方文档 Markdown 源与服务端渲染正文
- **一手来源**：
  - MCP 文档：<https://antigravity.google/docs/mcp>（源 `…/docs/mcp.md`，20491 字节）
  - Marketplace 文档：<https://antigravity.google/docs/marketplace>（源 `…/docs/marketplace.md`，10361 字节）
  - Plugins 文档：<https://antigravity.google/docs/plugins>（源 `…/docs/plugins.md`，13867 字节）
  - Plugin Marketplace 官方博客：<https://antigravity.google/blog/plugin-marketplace-in-antigravity>
  - Additional Terms：<https://antigravity.google/terms>
- **范围声明**：纯分析，不涉及部署、安装或任何配额消耗动作。

---

## 0. 核心结论

**Google 官方界定"合法"的主轴，不是"谁在调用"，而是"数据往哪个方向流"。**

| 方向 | 形态 | 官方态度 | 依据 |
|---|---|---|---|
| **agy → 外部系统**（agy 作为客户端去读/查/操作外部服务） | MCP server / Plugin | ✅ **官方主推**，建了市场、67+ 集成、OAuth 托管、权限分级 | MCP Store、Marketplace、77+ 支持条目 |
| **外部系统 → agy 的配额**（把 Antigravity 当模型后端供别的 harness 消费） | OAuth token 抽取 / 自建 proxy | ⛔ **条款点名禁止** | Additional Terms 第 6 条 |

一句话：**让 agy 长出更多手，合法；把手伸进 agy 的配额，违规。**

---

## 1. 官方案例：Google Developer Knowledge MCP

这是官方文档中给出的**唯一一个完整端到端案例**，因此最有参考价值。

### 1.1 它做什么

官方原文：

> "The Google Developer Knowledge MCP server lets Antigravity **search and read official, up-to-date Google developer documentation**, including Android, Firebase, Google Cloud, Google Maps Platform, and the Antigravity docs on this site, so the agent can check the current API before it writes code rather than relying only on its training data."

它暴露 **3 个工具**：

| 工具 | 作用 |
|---|---|
| `search_documents` | 检索文档 |
| `get_documents` | 读取具体文档 |
| `answer_query` | 就文档内容作答 |

### 1.2 官方示例任务（逐字）

```text
Look up the current Antigravity hooks documentation and add a PreToolUse hook
that blocks shell commands which modify files outside this workspace.
```

**这个例子极有信息量**——它同时体现了官方认可的三种行为：

1. **"Look up"**：agy 主动去读外部权威文档（而不是凭训练数据猜）
2. **"add a PreToolUse hook"**：产出物是**用户工作区里的代码/配置**，不是把配额转出去
3. **"blocks shell commands which modify files outside this workspace"**：连示例都在**收紧权限**，而不是放开

### 1.3 认证方式（合法性关键细节）

> "It connects with **your Google credentials**, so **no API key is needed**."

⇒ 官方自己就用"复用用户既有 Google 凭据"的模式。**所以"复用凭据"本身不是违规点**——违规点在**凭据被谁用来做什么**：
- 拿凭据去**读文档**（官方案例）→ 合法
- 拿凭据**冒充 agy 客户端去消费模型配额**（OpenClaw 式）→ 违规

这是本次分析最重要的区分。

---

## 2. 官方 MCP 市场的规模与形态（真实条目）

MCP Store 官方支持列表按类目组织，共 **77+ 个服务器**：

| 类目 | 数量 | 代表条目 |
|---|---|---|
| 数据库与存储 | 15 | AlloyDB、BigQuery、Cloud SQL、Spanner、Neon、Supabase、Redis、MongoDB、Neo4j、Pinecone、ClickHouse、Prisma、Bigtable、Dataplex、MCP Toolbox |
| 开发者工具与 CI/CD | 17 | GitHub、GitLab Orbit、GKE OneMCP、Cloud CLI Execution、Atlassian、Linear、Netlify、Vercel、Heroku、Harness、SonarQube、Postman、Apigee、mabl、Opsera、Forge、Home Developer |
| 前端与设计 | 8 | Chrome DevTools、Figma Dev Mode、Canva、Miro、Lovable、Locofy、Mobbin、Dart |
| 安全 | 4 | CrowdStrike、Wiz、Splunk、Endor Labs |
| 分析 / AI / 云 | 33+ | Exa、Arize、Docusign、Cloud Audit Manager、Daloopa、Courtroom5、Antimetal、Airweave 等 |

**注意这些条目的共同形态**：全部是**"agy 去使用某个外部服务"**（查数据库、读日志、建 issue、开 PR）。**没有任何一个条目是"让外部服务来使用 Antigravity 的模型"**。

这个分布本身就是答案——Google 认可并推广的扩展方向是单向的。

### 2.1 官方明示的典型用途

文档原文列的用途，全部是"补上下文"和"做安全动作"：

**补上下文（Add context）：**
- 写 SQL 时**检查实时 schema**，给出正确的表和列名
- 调试部署失败时**直接拉取构建日志**（Netlify / Heroku）

**做自定义动作（Add custom tools）：**
- 为 TODO 项**创建 Linear issue**
- 在 Notion 或 GitHub 里**搜索认证模式**

⇒ 模式高度一致：**读取外部真实状态 / 在外部系统里做有边界的小动作**。没有一个是"把 Antigravity 算力输出给别人"。

---

## 3. Plugin Marketplace：官方对"第三方"的正式态度

这条极其关键，因为它直接推翻"第三方一定是违规"的直觉。

官方博客原文：

> "Today, we are launching the Plugin Marketplace in Antigravity, a centralized directory for discovering and installing **third-party and Google-built plugins** across the Antigravity App and the Antigravity CLI."

**Google 自己在出版官方市场，专门用来分发第三方插件。** 所以"第三方"三个字在条款里指的是**第三方软件去访问 Antigravity 服务**，不是"第三方写的代码不能碰 Antigravity"。

### 3.1 官方市场的正式名称与安装路径

Plugins 文档中的实例输出显示，官方市场标识为 **`antigravity-plugins-official`**：

```
Source:     antigravity-plugins-official
```

三条合法安装来源：
1. **官方市场** `antigravity-plugins-official`（需有效 `agy` 登录会话）
2. **GitHub 仓库 URL**
3. **本地目录**

⇒ **从 GitHub 装第三方插件是被官方支持的一等公民**，不是灰色地带。这与"抽 OAuth 建 proxy"有本质区别。

### 3.2 插件封装的内容（五种原语）

一个 plugin 可打包：**Skills / MCP servers / custom agents & subagents / rules / lifecycle hooks**。

官方对 hooks 的描述值得注意——它被设计成**在模型推理循环之外**运行：

> "hooks run outside the model's reasoning loop and exchange structured JSON over standard input and output"

用途包括"gate or block sensitive tool calls, audit tool executions"。⇒ 官方把"**限制 agent**"当作一等能力，而不是要你放宽它。

### 3.3 渐进披露（progressive disclosure）

> "only each skill's name and short description are loaded at startup, and the full instructions, reference docs, and scripts are pulled into context on demand"

⇒ 扩展能力**不占用常驻上下文**。这也解释了为什么官方鼓励装插件而不担心把上下文撑爆。

---

## 4. 官方内置的"越界防线"：权限分级

这是判断合法性的**操作性发问点**。MCP 文档原文：

> "**By default, unconfigured MCP tools run in `Ask` mode, requiring your approval before execution.**"

三条授权粒度：

| 规则 | 语义 |
|---|---|
| `mcp(server/tool)` | 精确到"某服务器上的某个工具" |
| `mcp(server/*)` | 某服务器的全部工具 |
| `mcp(*)` | 所有已连接服务器的所有 MCP 工具（全局通配） |

而 MCP 官方文档的完整案例里，示例任务的产出物恰恰是**收紧权限的 hook**（"blocks shell commands which modify files outside this workspace"）。

⇒ **官方设计的默认姿态是"默认询问、按需最小授权"**。任何要求你 `mcp(*)` 全开、或绕过审批的方案，都偏离了官方默认姿态——这本身就是一个风险信号。

---

## 5. 认证机制：三种官方认可的方式

MCP 文档列出三种认证，它们都不需要你手工抽取 token：

### 5.1 Google 凭据（ADC）

```json
{"mcpServers": {"my-gcp-service": {
  "serverUrl": "https://example.googleapis.com/mcp/",
  "authProviderType": "google_credentials"
}}}
```
需本地配置 Application Default Credentials（`gcloud auth application-default login`）。

### 5.2 OAuth（自动）

> "Antigravity **automatically handles OAuth** for servers that support dynamic client registration (DCR). For these servers, **no additional configuration is needed**."

不支持 DCR 时，才手工填 `clientId` / `clientSecret`。

### 5.3 自定义 HTTP headers

用于 Bearer token 等场景。

**关键区分**：
- 官方 OAuth 流程 = **Authorization Server 给你一个受限 scope 的 token，用来访问那个具体的外部服务**。✅
- OpenClaw 式 = **从 agy 自己的登录态里抽出 token，拿去冒充 agy 客户端访问 Google 的模型后端**。⛔

两者都叫"OAuth token"，但**token 的主体、受众、用途完全不同**。条款禁的是后者。

---

## 6. 合法 / 违规判定表

| 行为 | 判定 | 依据 |
|---|---|---|
| 在 agy 里装官方市场插件（如连 GitHub、Linear、BigQuery） | ✅ 合法 | 官方主推，有市场分发 |
| 从 GitHub URL 或本地目录装第三方插件 | ✅ 合法 | Plugins 文档列为正式来源 |
| 让 agy 读外部文档/schema/日志来写代码（Developer Knowledge 案例） | ✅ 合法 | 官方原文案例 |
| 让 agy 在外部系统做有边界的小动作（建 issue） | ✅ 合法 | 官方 "Add custom tools" |
| 自己写插件/技能打包给 agy 用 | ✅ 合法 | 官方提供 build plugins + `agy plugin validate --strict` |
| 用 headless 让 agy 跑 CI 任务（改文件、跑测试） | ✅ 合法 | headless 文档明确 "integrate with CI pipelines" |
| 用 hooks 收紧 agy 的权限 | ✅ 合法且被鼓励 | 官方把"限制"设计成一等能力 |
| **抽 agy 的 OAuth token 给别的 harness 当模型用** | ⛔ **条款点名违规** | Terms 第 6 条点名 "using OpenClaw with Antigravity OAuth" |
| **自建反代/聚合多账号，把额度当 API 卖或用** | ⛔ 违规 | 同上，且属"滥用"范畴 |
| **把 Antigravity 配额接成第三方框架的 provider 路由** | ⛔ 高度危险 | 落在 "using third party software to access the Service" |
| 全开 `mcp(*)` 且不审 | ⚠️ 偏离官方默认姿态 | 官方默认是 `Ask` + 最小授权 |

---

## 7. 三条可操作的判断口诀

给定一个具体做法，按这三问判断：

**第一问：数据/算力的流向是哪边？**
- agy 去读外部 → 合法
- 外部来读 agy 的模型能力 → 违规

**第二问：凭据是谁的，给谁用？**
- 用我的凭据，让 agy 去访问那个凭据本该访问的服务（ADC / 官方 OAuth）→ 合法
- 把 agy 的登录 token 拿出去，冒充 agy 客户端 → 违规

**第三问：官方有没有提供这个能力？**
- 市场里发布、文档里写成正式来源（GitHub/本地目录）、SDK 里支持 → 合法
- 只能靠抽 token、模拟浏览器、绕过客户端 → 违规

**补充信号**：任何要求"多账号聚合 + 自动故障转移 + 额度池化"的方案，本质是把个人订阅当商用 API 用，这是历史封号潮的共同特征，应当直接排除。

---

## 8. 给 DSH 场景的具体结论

回到最初的问题——DSH 想用 Antigravity 的 Gemini Pro 配额，哪些做法落在安全侧：

| 做法 | 判定 | 理由 |
|---|---|---|
| DSH 通过 bash 启动官方 `agy --print` 完成**一次性、用户明确发起**的独立任务 | ⚠️ 灰色但风险较低 | 官方支持 headless for CI；但仍属"第三方框架驱动"，Google 未书面表态 |
| DSH 装插件/配 MCP，让 agy **自己**连更多系统 | ✅ 合法 | 官方主推方向 |
| DSH 做 `dsh-llm-agy` 这类**常驻 provider**，每轮都过 agy | ⛔ 不建议 | 使用密度远超正常客户端模式，最接近条款禁止的"第三方软件访问服务" |
| 自带 OAuth 的 `dsh-antigravity` 类插件（把 Coding Plan 当 LLM provider） | ⛔ 明确不碰 | 正是条款点名形态 |
| 多账号 proxy 池化 | ⛔ 不要碰 | 滥用 + 封号高危 |

**结论**：如果要在 Antigravity 上做文章，**安全的增量应该来自"给 agy 加插件/MCP"，而不是"把 agy 变成模型后端"**。

---

## 9. 未决项

| 项 | 状态 |
|---|---|
| Google 对"第三方 harness 驱动官方 agy"是否有书面立场 | **无官方答复**（论坛自 2026-07 起多次提问，官方账号未回应） |
| Terms 中 "products not provided by us" 的边界解释 | 措辞宽泛，无官方释义 |
| 本机 `agy` 版本与文档一致性 | 未核实（本轮纯分析，未在本机运行） |
| 官方市场条目的安全审计状态 | 官方称 "verified" 指可用性验证，**不等于**安全审计 |

---

## 10. 参考

- MCP 官方文档：<https://antigravity.google/docs/mcp>
- Marketplace 官方文档：<https://antigravity.google/docs/marketplace>
- Plugins 官方文档：<https://antigravity.google/docs/plugins>
- Plugin Marketplace 博客：<https://antigravity.google/blog/plugin-marketplace-in-antigravity>
- Google Developer Knowledge MCP：<https://developers.google.com/knowledge/mcp>
- Antigravity Additional Terms：<https://antigravity.google/terms>
- 配套报告：`/root/dev-10/AUDIT-2026-11-04-antigravity-cli-headless-analysis.md`

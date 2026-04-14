# DeerFlow 个人学习与贡献计划

> **目标**：1) 深入理解 Agent 原理与落地实践；2) 为 DeerFlow 项目贡献优质代码。  
> **项目**：DeerFlow 2.0 (bytedance/deer-flow) —— 基于 LangGraph 的 Super Agent Harness。  
> **建议周期**：12 周（可根据个人时间弹性调整，建议每天投入 1.5–2 小时）。

---

## 一、项目认知地图

在开始学习代码之前，先建立对 DeerFlow 的整体认知：

| 层级 | 职责 | 关键技术 | 对应目录 |
|------|------|----------|----------|
| **Frontend** | Web UI、Thread 管理、流式展示 | Next.js, LangGraph SDK, TanStack Query | `frontend/` |
| **Gateway API** | REST API、文件上传、Skills/MCP/Memory 管理 | FastAPI | `backend/app/gateway/` |
| **LangGraph Server** | Agent 运行时、工作流执行 | LangGraph, LangChain | `backend/packages/harness/deerflow/agents/` |
| **Harness (核心框架)** | Agent、Sandbox、Sub-agents、Tools、Skills、Memory、Models | Python 3.12+ | `backend/packages/harness/deerflow/` |
| **Channels** | IM 集成（飞书/Slack/Telegram/企微/微信） | WebSocket / Bot API | `backend/app/channels/` |
| **Skills** | 结构化能力模块（Markdown 定义） | Markdown + Scripts | `skills/public/`, `skills/custom/` |

核心架构链路：
```
User → Frontend → Nginx(2026) → Gateway API(8001) / LangGraph Server(2024)
                                          ↓
                                    Lead Agent (make_lead_agent)
                                          ↓
              Middlewares → Tools (built-in/MCP/community/subagents) → Sandbox
```

---

## 二、分阶段学习计划

### Phase 1：环境搭建与项目跑通（第 1–2 周）

**目标**：让 DeerFlow 在本地跑起来，熟悉基本操作和开发 workflow。

| 任务 | 具体内容 | 产出/验证 |
|------|----------|-----------|
| 1.1 阅读官方文档 | 通读 `README.md`、`CONTRIBUTING.md`、`backend/CLAUDE.md` | 能向别人口述 DeerFlow 是什么、核心特性有哪些 |
| 1.2 环境检查 | 执行 `make check`，确认 Node.js 22+、pnpm、uv、nginx 已安装 | 命令通过 |
| 1.3 配置项目 | `make setup` 生成 `config.yaml`，配置至少一个 LLM（建议先用 OpenAI/DeepSeek，门槛低） | 能成功调用模型 |
| 1.4 启动项目 | 执行 `make install` 和 `make dev`，访问 http://localhost:2026 | 浏览器能看到界面 |
| 1.5 体验核心功能 | 在 Web UI 里进行多轮对话、上传文件、观察 Thread 生成、开启 Plan Mode | 截图或笔记记录体验 |
| 1.6 运行测试 | `cd backend && make test`（至少跑通一部分，确认环境可用） | 了解测试结构 |

**避坑提示**：
- `config.yaml` 放在项目根目录，不要放 `backend/` 里。
- 本地沙箱默认关闭 `bash`，如需测试代码执行，可配 `AioSandboxProvider`（Docker 模式）。

---

### Phase 2：Agent 理论基础（第 2–3 周，可与 Phase 1 重叠）

**目标**：建立 Agent 设计的理论框架，理解 DeerFlow 为什么这样设计。

| 主题 | 学习资源 | 重点理解 |
|------|----------|----------|
| **LangGraph 核心概念** | [LangGraph 官方文档](https://langchain-ai.github.io/langgraph/) | StateGraph、Node、Edge、Checkpoint、Stream Mode |
| **ReAct / Tool Calling** | LangChain "Agents" 章节 + OpenAI Function Calling 论文 | LLM 如何通过 Thought → Action → Observation 循环解决问题 |
| **Multi-Agent 架构** | LangGraph Multi-Agent 教程 | Supervisor vs. 去中心化、子代理的生命周期管理 |
| **RAG / Memory** | 项目内 `backend/docs/` 相关文档 + LangChain Memory 文档 | 长期记忆 vs. 短期上下文、DeerFlow 的 Memory 注入机制 |
| **MCP (Model Context Protocol)** | 项目内 `backend/docs/MCP_SERVER.md` + Anthropic MCP 介绍 | 工具的标准化接入协议 |

**实践任务**：
1. 用纯 LangGraph 写一个简单的 ReAct Agent（不借助 DeerFlow），理解 `tool_node` 和 `conditional_edge` 的作用。
2. 对比你写的简单 Agent 和 DeerFlow 的 `lead_agent/agent.py`，记录 DeerFlow 做了哪些增强（middleware、state schema、subagent delegation 等）。

---

### Phase 3：代码精读 — 从外到内（第 3–6 周）

**目标**：逐层深入 DeerFlow 的核心代码，建立代码级的心智模型。

#### 3.1 入口层：Gateway 与 Frontend（第 3 周）

| 模块 | 关键文件 | 学习目标 |
|------|----------|----------|
| Gateway API | `backend/app/gateway/app.py`, `routers/` | 了解 REST 路由如何组织、请求如何转发到 Harness |
| Thread/Uploads | `backend/app/gateway/routers/threads.py`, `uploads.py` | 文件上传、Thread 生命周期、Artifact  serving |
| Frontend Thread | `frontend/src/core/threads/`, `frontend/src/app/workspace/chats/` | 前端如何管理 Thread State、如何消费 SSE 流 |

**实践**：用浏览器 DevTools 观察一次对话的 Network 请求，画出请求时序图。

#### 3.2 核心层：Lead Agent + Middlewares（第 4 周）

| 模块 | 关键文件 | 学习目标 |
|------|----------|----------|
| Lead Agent | `backend/packages/harness/deerflow/agents/lead_agent/agent.py` | `make_lead_agent()` 工厂函数、动态模型绑定、系统 Prompt 组装 |
| Prompt 组装 | `backend/packages/harness/deerflow/agents/lead_agent/prompt.py` | Skills、Memory、Subagent 指令如何注入 System Prompt |
| ThreadState | `backend/packages/harness/deerflow/agents/thread_state.py` | 自定义 reducers (`merge_artifacts`, `merge_viewed_images`) |
| Middlewares | `backend/packages/harness/deerflow/agents/middlewares/` | 理解 18 个 middleware 的分层职责（重点看 ThreadData → Sandbox → Guardrail → ToolError → Summarization → TodoList → Memory → Clarification） |

**实践**：
- 修改一个 middleware 的日志输出，重新运行 `make dev`，观察日志变化。
- 阅读 `test_todo_middleware.py`、`test_memory_updater.py`，理解如何用测试验证 middleware 行为。

#### 3.3 能力层：Tools / Skills / Sandbox / Subagents（第 5 周）

| 模块 | 关键文件 | 学习目标 |
|------|----------|----------|
| Tool System | `backend/packages/harness/deerflow/tools/builtins/`, `tools.py` | `get_available_tools()` 如何组装内置/MCP/Community/Subagent 工具 |
| Skills System | `backend/packages/harness/deerflow/skills/`, `skills/public/deep-research/SKILL.md` | Skill 发现、加载、解析、注入机制 |
| Sandbox | `backend/packages/harness/deerflow/sandbox/sandbox.py`, `tools.py`, `local/`, `community/aio_sandbox.py` | 虚拟路径映射、命令执行隔离、文件操作安全 |
| Subagents | `backend/packages/harness/deerflow/subagents/executor.py`, `registry.py`, `builtins/` | 任务委托流程、线程池调度、`task` 工具的实现 |

**实践**：
1. 在 `skills/custom/` 下创建一个最小自定义 Skill（例如 "hello-world"），测试它是否被正确加载到 System Prompt 中。
2. 阅读 `test_subagent_executor.py`，理解 subagent 的并发限制和超时机制。

#### 3.4 扩展层：Memory / Models / MCP / Channels（第 6 周）

| 模块 | 关键文件 | 学习目标 |
|------|----------|----------|
| Memory | `backend/packages/harness/deerflow/agents/memory/updater.py`, `queue.py`, `prompt.py` | 事实提取、去重、debounce、注入策略 |
| Models | `backend/packages/harness/deerflow/models/factory.py`, `vllm_provider.py` | 动态模型实例化、thinking/vision 支持 |
| MCP | `backend/packages/harness/deerflow/mcp/`, `app/gateway/routers/mcp.py` | MCP Client 缓存、懒加载、运行时更新 |
| Channels | `backend/app/channels/manager.py`, `feishu.py`, `slack.py`, `telegram.py` | 外部消息如何映射为 LangGraph Thread |

**实践**：
- 用 `DeerFlowClient`（`backend/packages/harness/deerflow/client.py`）写一段 Python 脚本，不启动 HTTP 服务直接与 Agent 交互。
- （可选）配置 Telegram Bot，让 DeerFlow 通过 IM 回复你的消息。

---

### Phase 4：动手实践 — 从使用到改造（第 6–9 周）

**目标**：通过实际项目加深理解，产出可运行的代码。

#### 选项 A：开发一个自定义 Skill（推荐新手）

Skill 是最友好的贡献入口：无需修改核心代码，只需 Markdown + 可选脚本。

**示例方向**：
- 周报生成器：读取 workspace 中的文件，生成结构化周报。
- 技术文档审阅 Skill：检查 Markdown 文档的完整性和一致性。
- 个人知识库问答 Skill：结合 Memory 中的事实，回答个人定制化问题。

**步骤**：
1. 在 `skills/custom/<your-skill>/SKILL.md` 编写 Skill 定义（参考 `skills/public/deep-research/SKILL.md`）。
2. 在 Web UI 中启用并测试。
3. 整理成 `.skill` 压缩包，验证 `POST /api/skills/install` 可以安装。
4. 如果质量高，可整理后向社区分享或提交 PR 到 `skills/public/`。

#### 选项 B：添加一个 Community Tool

**示例方向**：
- 新增一个搜索/抓取工具（如接入 InfoQuest、Exa、DuckDuckGo 等）。
- 新增一个数据处理工具（如 CSV 分析、JSON Schema 验证）。

**步骤**：
1. 在 `backend/packages/harness/deerflow/community/` 下新建模块。
2. 实现 Tool 函数，配置 `config.yaml` 的 `tools` 段。
3. 编写单元测试（参考 `test_firecrawl_tools.py`）。
4. 确保 `make test` 通过。

#### 选项 C：实现/改进一个 Middleware

适合对 LangGraph 有较深理解后尝试。

**示例方向**：
- 自定义审计 middleware：记录所有 tool call 的输入输出到独立日志文件。
- 自定义速率限制 middleware：基于用户 ID 限制每分钟请求数。
- 改进现有 middleware 的测试覆盖率。

#### 选项 D：Frontend 改进

**示例方向**：
- 优化 Thread 列表的加载状态或空状态。
- 为 Artifact 预览增加新的 MIME 类型支持。
- 改进 SSE 流式渲染的异常处理。

---

### Phase 5：贡献代码（第 9 周起，长期）

**目标**：持续为 DeerFlow 社区贡献代码，建立影响力。

#### 5.1 寻找贡献切入点

| 类型 | 查找方式 | 示例 |
|------|----------|------|
| **Good First Issues** | GitHub Issues 标签筛选（待维护者添加 `good first issue`） | 文档修复、测试补充、Skill 模板 |
| **Bug Fix** | 自己使用过程中遇到的异常 | 本地测试、定位、修复、提交 PR |
| **Feature Request** | GitHub Discussions / Issues | 先看维护者反馈再动手 |
| **测试补充** | 阅读 `backend/tests/` 中覆盖率较低的模块 | 为 `channels/` 或 `sandbox/` 补测试 |

#### 5.2 贡献 Workflow

```bash
# 1. 创建功能分支
git checkout -b feature/your-feature-name

# 2. 编码 + 测试
# 每改一个模块，同步跑测试：
cd backend && PYTHONPATH=. uv run pytest tests/test_<feature>.py -v

# 3. 格式化
cd backend && make format
cd frontend && pnpm format:write

# 4. 全量测试通过
cd backend && make test
cd frontend && pnpm test

# 5. 提交（遵循常规 commit 规范）
git commit -m "feat: add xxx tool"
git commit -m "fix: correct middleware yyy behavior"
git commit -m "docs: update zzz setup guide"

# 6. 推送到个人 fork 并创建 Pull Request
```

#### 5.3 代码规范 checklist

- [ ] 新功能必须带单元测试（项目强制 TDD）。
- [ ] 不要破坏 `tests/test_harness_boundary.py`：Harness 层（`deerflow.*`）不能引用 App 层（`app.*`）。
- [ ] 修改文档：`README.md`（用户视角）和 `CLAUDE.md` / `backend/docs/`（开发者视角）。
- [ ] Backend 用 `ruff` 格式化，Frontend 用 Prettier。
- [ ] Python 类型注解完整。

---

## 三、推荐学习路径（按兴趣选择）

### 路线 A：Agent 系统工程师（偏后端）
> 重点深入研究 Lead Agent、Middlewares、Subagents、Memory

**优先级**：Phase 1 → Phase 2 → 3.2 → 3.3 → 3.4 → 选项 C/B → 贡献核心代码

### 路线 B：Skill / Tool 开发者（偏应用）
> 重点学习 Skills 体系、Sandbox、Community Tools

**优先级**：Phase 1 → Phase 2（快速过）→ 3.3 → 选项 A/B → 贡献 Skills 和 Tools

### 路线 C：全栈贡献者（前后端兼顾）
> 理解完整链路，能端到端解决问题

**优先级**：Phase 1 → Phase 2 → 3.1 → 3.2 → 3.3 → 选项 D/B → 贡献综合功能

---

## 四、关键资源索引

| 资源 | 路径 | 用途 |
|------|------|------|
| 架构总览 | `backend/CLAUDE.md` | 开发者必读，随时回顾 |
| 配置指南 | `backend/docs/CONFIGURATION.md` | 调参、加模型、改沙箱 |
| MCP 指南 | `backend/docs/MCP_SERVER.md` | 接入外部 MCP Server |
| 前端架构 | `frontend/AGENTS.md` | 前端 Agent 集成逻辑 |
| 贡献指南 | `CONTRIBUTING.md` | PR 流程、代码风格 |
| 配置示例 | `config.example.yaml` | 所有可配项的参考 |
| Skill 示例 | `skills/public/deep-research/SKILL.md` | 写 Skill 的范本 |
| 测试参考 | `backend/tests/test_*.py` | 学习如何写测试 |

---

## 五、每周复盘模板

建议每周花 15 分钟填写以下问题，保持学习节奏：

1. **本周完成了哪些任务？**
2. **对 Agent 的理解有什么新进展？**
3. **遇到了哪些卡点？是否需要求助社区/文档？**
4. **下周最优先的 1–2 件事是什么？**
5. **有没有可以提交的代码/笔记/Skill？**

---

## 六、里程碑检查点

| 时间点 | 里程碑 | 验收标准 |
|--------|--------|----------|
| 第 2 周末 | 项目跑通 | `make dev` 成功，能与 Agent 对话 |
| 第 4 周末 | 理解 Lead Agent | 能画出 middleware 执行顺序图，能解释 `thread_state.py` 的设计 |
| 第 6 周末 | 完成一个 Skill/Tool | 代码/文档可运行，能通过基础测试 |
| 第 9 周末 | 首次代码贡献 | 至少提交 1 个被合并的 PR（文档/测试/Skill/功能均可） |
| 第 12 周末 | 深度参与 | 能独立 review 他人 PR，或在某个模块（如 Memory/Subagent）回答社区问题 |

---

> **最后提醒**：DeerFlow 是一个工程化程度很高的 Agent 项目，不要试图在 1–2 周内“读完所有代码”。按模块深入、边用边读、边改边学，是最有效的路径。祝你学习顺利，早日成为 DeerFlow 的贡献者！🦌

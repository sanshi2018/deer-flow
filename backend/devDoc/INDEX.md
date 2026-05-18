# DeerFlow 22 篇学习文档 · 完整索引

> 4 个维度的索引：**按章节** · **按主题** · **按符号/源码** · **按 Harness 六要素**。任何一个角度都能快速跳到对应章节 + 源码位置。

---

## 📖 文档清单（按章节）

### Part A · 全局观（3 篇）

| # | 标题 | 一句话 | 关键概念 |
|---|------|--------|---------|
| [**00**](./00-overview.md) | 学习路线总览 | 22 篇切分方案 + 仓库结构地图 | 中间件链矩阵 / 依赖关系图 |
| [**01**](./01-startup-and-runtime.md) | 启动链路与运行时 | `make dev` → 3 进程 + LangGraph 装配点 | RunManager / StreamBridge / langgraph.json |
| [**02**](./02-harness-app-boundary.md) | Harness ↔ App 双层架构 | 47 行 AST 测试 + CurrentUser Protocol | test_harness_boundary / uv workspace |
| [**03**](./03-config-and-reflection.md) | 配置体系 + 反射装配 | AppConfig + ExtensionsConfig + `resolve_variable` | 26 子配置 / mtime 失效 / push_current_app_config |

### Part B · LangGraph 状态与图核心（3 篇）

| # | 标题 | 一句话 | 关键概念 |
|---|------|--------|---------|
| [**04**](./04-thread-state-and-reducers.md) | ThreadState 状态模型 + Reducer | 55 行源码 / 6 字段 + 2 reducer | merge_artifacts / merge_viewed_images / "空 dict 清空" |
| [**05**](./05-agent-factory-dual-track.md) | Agent 工厂双轨 | `make_lead_agent` vs `create_deerflow_agent` | RuntimeFeatures / `@Next/@Prev` / `_insert_extra` |
| [**06**](./06-middleware-chain-overview.md) | 中间件链总览 14-18 节 | 全景表 × 6 hook | 双源装配 / ClarificationMiddleware 必须最后 |

### Part C · 沙箱与文件系统（3 篇）

| # | 标题 | 一句话 | 关键概念 |
|---|------|--------|---------|
| [**07**](./07-sandbox-abstraction-and-lifecycle.md) | 沙箱抽象与生命周期 | 三件套：Sandbox/Provider/Middleware | LocalSandboxProvider 单例 / AioSandboxProvider warm pool / lazy_init |
| [**08**](./08-virtual-paths-and-mounts.md) | 虚拟路径系统 + 多挂载点 | 5 类挂载 + 双向翻译 + 3 重防御 | replace_virtual_path / mask_local_paths / DEER_FLOW_HOST_BASE_DIR |
| [**09**](./09-sandbox-tools.md) | 沙箱工具集 7 个 | bash/read/write/str_replace/ls/glob/grep | shlex token 校验 / file_operation_lock / 输出截断三策略 |

### Part D · 工具系统与 MCP（2 篇）

| # | 标题 | 一句话 | 关键概念 |
|---|------|--------|---------|
| [**10**](./10-tool-assembly-and-dedup.md) | 工具装配 + 反射 + 去重 | 4 类来源 + first-wins 优先级 | get_available_tools / make_sync_tool_wrapper |
| [**11**](./11-mcp-and-tool-search.md) | MCP 集成 + Tool Search | mtime cache + OAuth + deferred registry | OAuthTokenManager / DeferredToolRegistry / 3 种 query |

### Part E · Skills 与 Prompt（2 篇）

| # | 标题 | 一句话 | 关键概念 |
|---|------|--------|---------|
| [**12**](./12-skills-system.md) | Skills 系统 | SKILL.md → system prompt | allowed-tools 反向裁剪 / LLM-as-reviewer 安全扫描 |
| [**13**](./13-system-prompt-assembly.md) | 823 行 System Prompt | 9 动态片段 + prefix cache 友好 | apply_prompt_template / skills 预热 / version counter |

### Part F · 记忆系统（2 篇）

| # | 标题 | 一句话 | 关键概念 |
|---|------|--------|---------|
| [**14**](./14-memory-middleware-queue-updater.md) | MemoryMiddleware + 异步 Queue + Updater | 4 段流水线 + sync LLM 兜底 | filter_messages_for_memory / debounce / `_do_update_memory_sync` |
| [**15**](./15-dynamic-context-and-prefix-cache.md) | DynamicContext + Prefix Cache | 注入 first HumanMessage 的 ID swap | 3 分支 / frozen snapshot / midnight crossing |

### Part G · Subagent（2 篇）

| # | 标题 | 一句话 | 关键概念 |
|---|------|--------|---------|
| [**16**](./16-subagent-orchestration.md) | Subagent 编排 + Executor + 双线程池 | task tool + 持久 isolated loop | `_isolated_subagent_loop` / `_submit_to_isolated_loop_in_context` / cancel 4 步 |
| [**17**](./17-subagent-registry-and-token-accounting.md) | 注册表 + Token 回收 + Cache | built-in + custom + override 三层 | SubagentTokenCollector / `_subagent_usage_cache` / tool_call_id 跨边界传票 |

### Part H · 反思纠错与可观测性（2 篇）

| # | 标题 | 一句话 | 关键概念 |
|---|------|--------|---------|
| [**18**](./18-error-loop-summarization.md) | 错误处理三件套 + LoopDetection + Summarization | 5 个反馈中间件分工 | DanglingToolCall / 熔断 3 态机 / stable key / skill rescue |
| [**19**](./19-tracing-and-observability.md) | Tracing + Token Usage + LangFuse | RunJournal 8 字段 + 双源 trace | `_identify_caller` / `_flush_sync` / `record_external_llm_usage_records` |

### Part I · 运行时与持久化（2 篇）

| # | 标题 | 一句话 | 关键概念 |
|---|------|--------|---------|
| [**20**](./20-runmanager-worker-streambridge.md) | RunManager + worker + StreamBridge | 3 种 cancel 语义 + Last-Event-ID 续传 | abort_event 协作 / 8 段 worker 主流程 / pre-run snapshot rollback |
| [**21**](./21-persistence-and-migrations.md) | Persistence + Alembic + 5 表 | one DB two schemas / per-connection PRAGMA | UserRow.token_version / partial unique index / `_AUTO sentinel` |

### Part J · 接入层（1 篇）

| # | 标题 | 一句话 | 关键概念 |
|---|------|--------|---------|
| [**22**](./22-gateway-channels-client.md) | Gateway + IM Channels + DeerFlowClient | 3 种接入共享同一 core | 15 router / AuthMiddleware + CSRF 双闸门 / langgraph-sdk 自调 |

### 辅助文档

- [**FAQ.md**](./FAQ.md) — 60 个高频问答，按主题分 12 组
- [**INDEX.md**](./INDEX.md) — 本文档，完整索引

---

## 🎯 按主题索引

### 启动 / 部署

| 主题 | 章节 |
|------|------|
| `make dev` 进程数 / 端口 | [01 § 2.3 启动时序](./01-startup-and-runtime.md) / [FAQ Q1](./FAQ.md#q1-make-dev-起了几个进程怎么验证) |
| `make dev` vs `make start` | [01 § Q6](./01-startup-and-runtime.md) / [FAQ Q56](./FAQ.md#q56-make-dev-vs-make-start-区别) |
| Docker 模式 DooD 陷阱 | [08 § 3.2 host_base_dir](./08-virtual-paths-and-mounts.md) / [FAQ Q57](./FAQ.md#q57-docker-部署时-deer_flow_host_base_dir-是什么) |
| Nginx 路由表 | [01 § 2.4 + Lab ③](./01-startup-and-runtime.md) |
| Gateway lifespan 钩子 | [01 § 3.2 + § 3.3](./01-startup-and-runtime.md) |

### 配置体系

| 主题 | 章节 |
|------|------|
| AppConfig 26 子配置 | [03 § 3.1](./03-config-and-reflection.md) |
| `$VAR` 环境变量解析 | [03 § 3.3](./03-config-and-reflection.md) / [FAQ Q5](./FAQ.md#q5-openai_api_key-怎么解析) |
| mtime 失效自动 reload | [03 § 3.3](./03-config-and-reflection.md) / [FAQ Q2](./FAQ.md#q2-改-configyaml-需要重启-gateway-吗) |
| `use:` 反射加载 | [03 § 3.5 resolve_variable](./03-config-and-reflection.md) / [FAQ Q6](./FAQ.md#q6-use-deerflowsandboxtoolsbash_tool-这种字符串是怎么变成工具的) |
| ExtensionsConfig | [03 § 2.1](./03-config-and-reflection.md) / [11 § 3.1](./11-mcp-and-tool-search.md) |
| ContextVar 临时覆盖 | [03 § Q4](./03-config-and-reflection.md) |

### 架构边界

| 主题 | 章节 |
|------|------|
| Harness/App 双层 | [02 全章](./02-harness-app-boundary.md) |
| AST 边界守卫 | [02 § 3.1](./02-harness-app-boundary.md) |
| CurrentUser Protocol 依赖反转 | [02 § 3.3](./02-harness-app-boundary.md) |
| uv workspace + hatchling | [02 § 3.2](./02-harness-app-boundary.md) |

### ThreadState

| 主题 | 章节 |
|------|------|
| 7 字段 + 2 reducer 全景 | [04 § 2.3 类图](./04-thread-state-and-reducers.md) |
| `merge_artifacts` 去重保序 | [04 § 3.4](./04-thread-state-and-reducers.md) |
| `merge_viewed_images` 空 dict 清空 | [04 § 3.5](./04-thread-state-and-reducers.md) / [FAQ Q15](./FAQ.md#q15-merge_viewed_images-为什么有空-dict-等于清空特例) |
| 字段所有权矩阵 | [04 § 2.4](./04-thread-state-and-reducers.md) |
| State vs Context vs Configurable | [04 § Q6](./04-thread-state-and-reducers.md) / [FAQ Q13](./FAQ.md#q13-threadstate-vs-runcontext-vs-configurable-有什么区别) |

### Agent 工厂

| 主题 | 章节 |
|------|------|
| `make_lead_agent` 应用工厂 | [05 § 3.1-3.2](./05-agent-factory-dual-track.md) |
| `create_deerflow_agent` SDK 工厂 | [05 § 3.3](./05-agent-factory-dual-track.md) |
| `RuntimeFeatures` 8 开关 | [05 § 3.4](./05-agent-factory-dual-track.md) |
| `@Next/@Prev` 装饰器 | [05 § 3.5-3.6](./05-agent-factory-dual-track.md) |

### 中间件链

| 主题 | 章节 |
|------|------|
| 18 节 × 6 hook 全景 | [06 § 2.2](./06-middleware-chain-overview.md) |
| 双源装配 | [06 § 3.1](./06-middleware-chain-overview.md) |
| 4 个 always-on | [06 § 3.2](./06-middleware-chain-overview.md) |
| 3 个顺序契约 | [06 § 3.3](./06-middleware-chain-overview.md) |
| ClarificationMiddleware 必须最后 | [06 § Q1](./06-middleware-chain-overview.md) / [FAQ Q8](./FAQ.md#q8-为什么-clarificationmiddleware-必须最后) |

### 沙箱

| 主题 | 章节 |
|------|------|
| Sandbox/Provider/Middleware 三件套 | [07 § 2.3](./07-sandbox-abstraction-and-lifecycle.md) |
| LocalSandboxProvider 单例 | [07 § 3.4](./07-sandbox-abstraction-and-lifecycle.md) |
| AioSandboxProvider 多实例 + warm pool | [07 § 3.5](./07-sandbox-abstraction-and-lifecycle.md) |
| `lazy_init=True` 关键作用 | [07 § 3.6](./07-sandbox-abstraction-and-lifecycle.md) / [FAQ Q22](./FAQ.md#q22-bash_tool-在-localsandbox-下为什么默认看不见) |

### 虚拟路径

| 主题 | 章节 |
|------|------|
| 5 类挂载点访问矩阵 | [08 § 1 + § 3.7](./08-virtual-paths-and-mounts.md) |
| `Paths.base_dir` vs `host_base_dir` | [08 § 3.2](./08-virtual-paths-and-mounts.md) |
| 三道防御（白名单/段边界/relative_to） | [08 § 3.3 + 3.4](./08-virtual-paths-and-mounts.md) |
| 反向脱敏 `mask_local_paths_in_output` | [08 § 3.6](./08-virtual-paths-and-mounts.md) |
| `_RESERVED_CONTAINER_PREFIXES` | [08 § 3.8](./08-virtual-paths-and-mounts.md) |

### 沙箱工具

| 主题 | 章节 |
|------|------|
| 7 工具统一管线 | [09 § 2.3](./09-sandbox-tools.md) |
| `description` 必填范式 | [09 § 3.1](./09-sandbox-tools.md) |
| `bash_tool` 4 道工序 | [09 § 3.3](./09-sandbox-tools.md) |
| `file_operation_lock` 28 行库 | [09 § 3.4](./09-sandbox-tools.md) |
| 输出截断 3 策略 | [09 § 3.6](./09-sandbox-tools.md) |

### 工具装配 + MCP

| 主题 | 章节 |
|------|------|
| `get_available_tools` 4 来源 | [10 § 2.3-2.4](./10-tool-assembly-and-dedup.md) |
| `tool.name` vs `cfg.name` (issue #1803) | [10 § 3.2](./10-tool-assembly-and-dedup.md) / [FAQ Q23](./FAQ.md#q23-tool-name-不一致怎么办) |
| `make_sync_tool_wrapper` 37 行桥 | [10 § 3.3](./10-tool-assembly-and-dedup.md) |
| MCP mtime cache | [11 § 3.1-3.2](./11-mcp-and-tool-search.md) |
| OAuth `_fetch_token` 2 grant | [11 § 3.5](./11-mcp-and-tool-search.md) |
| Tool Search 3 query 形式 | [11 § 3.7](./11-mcp-and-tool-search.md) |

### Skills

| 主题 | 章节 |
|------|------|
| SKILL.md frontmatter 7 字段 | [12 § 3.2 + 3.3](./12-skills-system.md) |
| `allowed-tools` 三态 + 反向裁剪 | [12 § 3.4 + 3.5](./12-skills-system.md) / [FAQ Q29](./FAQ.md#q29-allowed-tools-和不写有什么区别) |
| `.skill` 安装 3 层防御 | [12 § 3.6](./12-skills-system.md) |
| LLM-as-reviewer 安全扫描 | [12 § 3.7](./12-skills-system.md) |
| public/custom 双轨覆盖 | [12 § Q2](./12-skills-system.md) |

### System Prompt

| 主题 | 章节 |
|------|------|
| 9 动态片段 | [13 § 2.3](./13-system-prompt-assembly.md) |
| prefix cache 友好纪律 | [13 § 3.2](./13-system-prompt-assembly.md) / [FAQ Q16](./FAQ.md#q16-system-prompt-为什么不放当前日期-memory) |
| `apply_prompt_template` 主入口 | [13 § 3.1](./13-system-prompt-assembly.md) |
| skills 预热 + version counter | [13 § 3.5](./13-system-prompt-assembly.md) |

### Memory

| 主题 | 章节 |
|------|------|
| 4 段流水线 | [14 § 2.3](./14-memory-middleware-queue-updater.md) |
| `filter_messages_for_memory` 3 重过滤 | [14 § 3.2](./14-memory-middleware-queue-updater.md) |
| correction/reinforcement 双语正则 | [14 § 3.3](./14-memory-middleware-queue-updater.md) |
| MemoryUpdateQueue debounce | [14 § 3.4](./14-memory-middleware-queue-updater.md) |
| `_do_update_memory_sync` 防 issue #2615 | [14 § 3.5](./14-memory-middleware-queue-updater.md) |
| user_id 必须 enqueue 抓 | [14 § 3.6](./14-memory-middleware-queue-updater.md) / [FAQ Q33](./FAQ.md#q33-为什么-user_id-必须在-enqueue-时同步抓) |

### 动态上下文

| 主题 | 章节 |
|------|------|
| `DynamicContextMiddleware` 3 分支 | [15 § 3.1](./15-dynamic-context-and-prefix-cache.md) |
| ID swap 技术 | [15 § 3.2](./15-dynamic-context-and-prefix-cache.md) / [FAQ Q17](./FAQ.md#q17-_inject-的id-swap具体怎么工作) |
| frozen snapshot 策略 | [15 § 3.3](./15-dynamic-context-and-prefix-cache.md) |
| midnight crossing 增量 | [15 § 3.1 分支 ③](./15-dynamic-context-and-prefix-cache.md) |

### Subagent

| 主题 | 章节 |
|------|------|
| `task_tool` poll loop 5s | [16 § 3.1-3.2](./16-subagent-orchestration.md) |
| 持久 isolated event loop | [16 § 3.3](./16-subagent-orchestration.md) |
| ContextVar 跨 3 执行环境传递 | [16 § 3.4](./16-subagent-orchestration.md) |
| 协作式 cancel 4 步 | [16 § 3.6](./16-subagent-orchestration.md) / [FAQ Q39](./FAQ.md#q39-subagent-超时-cancel-真的会停吗) |
| SubagentLimitMiddleware after_model 截断 | [16 § 3.7](./16-subagent-orchestration.md) |
| 注册表 3 层 override | [17 § 3.4](./17-subagent-registry-and-token-accounting.md) |
| 内置 general-purpose / bash | [17 § 3.2](./17-subagent-registry-and-token-accounting.md) |
| custom_agents yaml 配置 + demo | [17 § 3.3 + § 6.3](./17-subagent-registry-and-token-accounting.md) |
| SubagentTokenCollector 防重 | [17 § 3.6](./17-subagent-registry-and-token-accounting.md) |
| `tool_call_id` 跨边界传票 | [17 § 5.2](./17-subagent-registry-and-token-accounting.md) |

### 错误处理

| 主题 | 章节 |
|------|------|
| 5 个中间件分工 | [18 § 2.3](./18-error-loop-summarization.md) |
| DanglingToolCall wrap_model_call 重排 | [18 § 3.1](./18-error-loop-summarization.md) |
| LLMError 6 类分类 + 重试 + 熔断 3 态机 | [18 § 3.2](./18-error-loop-summarization.md) / [FAQ Q41](./FAQ.md#q41-llm-5xx-自动重试吗) |
| LoopDetection stable key | [18 § 3.3 + § 6.2 实验 ④](./18-error-loop-summarization.md) / [FAQ Q40](./FAQ.md#q40-agent-卡死循环一直调同一工具怎么办) |
| Summarization trigger/keep 3 单位 | [18 § 3.4](./18-error-loop-summarization.md) |
| skill rescue 双预算 | [18 § 3.4](./18-error-loop-summarization.md) |

### 可观测性

| 主题 | 章节 |
|------|------|
| RunJournal 8 字段 + 6 token 桶 + 3 dedup set | [19 § 3.1-3.3](./19-tracing-and-observability.md) |
| caller tag 识别链路 | [19 § 3.2](./19-tracing-and-observability.md) / [FAQ Q44](./FAQ.md#q44-token-怎么按-caller-细分) |
| sync callback + async store buffer 桥 | [19 § 3.4](./19-tracing-and-observability.md) |
| LangFuse / LangSmith 双源 | [19 § 3.7](./19-tracing-and-observability.md) / [FAQ Q43](./FAQ.md#q43-trace-在哪里看) |
| token 跨边界回灌端到端 | [19 § 6.3](./19-tracing-and-observability.md) |

### 运行时

| 主题 | 章节 |
|------|------|
| `RunManager.create_or_reject` atomic | [20 § 3.2](./20-runmanager-worker-streambridge.md) |
| 3 种 cancel 语义 | [20 § 3.3 + § 2.4](./20-runmanager-worker-streambridge.md) / [FAQ Q45](./FAQ.md#q45-cancel-一个-run-有几种方式) |
| run_agent worker 8 段 + finally 5 件事 | [20 § 3.4](./20-runmanager-worker-streambridge.md) |
| pre-run snapshot rollback | [20 § 3.8](./20-runmanager-worker-streambridge.md) |
| StreamBridge 环形 buffer + Last-Event-ID | [20 § 3.6-3.7](./20-runmanager-worker-streambridge.md) / [FAQ Q46](./FAQ.md#q46-sse-断线重连会丢消息吗) |

### 持久化

| 主题 | 章节 |
|------|------|
| 5 表 + LangGraph 表 one DB two schemas | [21 § 2.4 类图](./21-persistence-and-migrations.md) |
| `init_engine` 3 backend + 缺库自愈 | [21 § 3.1](./21-persistence-and-migrations.md) / [FAQ Q58](./FAQ.md#q58-用-postgres-但忘建数据库怎么办) |
| SQLite PRAGMA 三件套 per-connection | [21 § 3.2](./21-persistence-and-migrations.md) |
| RunRow 7 token 桶 + 3 convenience | [21 § 3.3](./21-persistence-and-migrations.md) |
| UserRow.token_version JWT 撤销 | [21 § 3.3 + 5.3](./21-persistence-and-migrations.md) / [FAQ Q52](./FAQ.md#q52-改密码后旧-jwt-怎么处理) |
| Alembic 现状 + `create_all` 增量 | [21 § 3.4](./21-persistence-and-migrations.md) / [FAQ Q48](./FAQ.md#q48-alembic-还没有迁移文件怎么演化-schema) |
| `_AUTO sentinel` secure-by-default | [21 § 3.5 + 5.4](./21-persistence-and-migrations.md) |

### 接入层

| 主题 | 章节 |
|------|------|
| 15 router 5 类划分 | [22 § 3.1](./22-gateway-channels-client.md) |
| AuthMiddleware fail-closed | [22 § 3.2](./22-gateway-channels-client.md) |
| CSRFMiddleware Double Submit Cookie | [22 § 3.3](./22-gateway-channels-client.md) / [FAQ Q51](./FAQ.md#q51-csrf-token-必填吗什么时候要带) |
| 中间件注册顺序 | [22 § 3.4](./22-gateway-channels-client.md) |
| IM Channels 通过 langgraph-sdk 自调 | [22 § 3.5-3.7](./22-gateway-channels-client.md) / [FAQ Q53](./FAQ.md#q53-im-channels-是怎么调-agent-的) |
| DeerFlowClient 1:1 对齐 Gateway | [22 § 3.8](./22-gateway-channels-client.md) |
| 3 种接入对比矩阵 | [22 § 3.9](./22-gateway-channels-client.md) / [FAQ Q54](./FAQ.md#q54-deerflowclient-vs-http-api-怎么选) |

---

## 🔍 按符号 / 源码索引

### 核心类

| 符号 | 文件 | 章节 |
|------|------|------|
| `AppConfig` | `config/app_config.py:83` | [03](./03-config-and-reflection.md) |
| `ExtensionsConfig` | `config/extensions_config.py:57` | [03](./03-config-and-reflection.md) / [11](./11-mcp-and-tool-search.md) |
| `ThreadState` | `agents/thread_state.py:48` | [04](./04-thread-state-and-reducers.md) |
| `RuntimeFeatures` | `agents/features.py:14` | [05](./05-agent-factory-dual-track.md) |
| `Sandbox` (ABC) | `sandbox/sandbox.py:6` | [07](./07-sandbox-abstraction-and-lifecycle.md) |
| `SandboxProvider` (ABC) | `sandbox/sandbox_provider.py:8` | [07](./07-sandbox-abstraction-and-lifecycle.md) |
| `LocalSandboxProvider` | `sandbox/local/local_sandbox_provider.py:13` | [07](./07-sandbox-abstraction-and-lifecycle.md) |
| `AioSandboxProvider` | `community/aio_sandbox/aio_sandbox_provider.py:69` | [07](./07-sandbox-abstraction-and-lifecycle.md) |
| `Paths` | `config/paths.py:62` | [08](./08-virtual-paths-and-mounts.md) |
| `Skill` | `skills/types.py:19` | [12](./12-skills-system.md) |
| `SkillStorage` (ABC) | `skills/storage/skill_storage.py:18` | [12](./12-skills-system.md) |
| `MemoryUpdateQueue` | `agents/memory/queue.py:28` | [14](./14-memory-middleware-queue-updater.md) |
| `MemoryUpdater` | `agents/memory/updater.py:276` | [14](./14-memory-middleware-queue-updater.md) |
| `FileMemoryStorage` | `agents/memory/storage.py:62` | [14](./14-memory-middleware-queue-updater.md) |
| `DynamicContextMiddleware` | `agents/middlewares/dynamic_context_middleware.py:81` | [15](./15-dynamic-context-and-prefix-cache.md) |
| `SubagentExecutor` | `subagents/executor.py:224` | [16](./16-subagent-orchestration.md) |
| `SubagentConfig` | `subagents/config.py:11` | [17](./17-subagent-registry-and-token-accounting.md) |
| `SubagentTokenCollector` | `subagents/token_collector.py:15` | [17](./17-subagent-registry-and-token-accounting.md) |
| `LLMErrorHandlingMiddleware` | `agents/middlewares/llm_error_handling_middleware.py:66` | [18](./18-error-loop-summarization.md) |
| `LoopDetectionMiddleware` | `agents/middlewares/loop_detection_middleware.py:144` | [18](./18-error-loop-summarization.md) |
| `DeerFlowSummarizationMiddleware` | `agents/middlewares/summarization_middleware.py:98` | [18](./18-error-loop-summarization.md) |
| `RunJournal` | `runtime/journal.py:38` | [19](./19-tracing-and-observability.md) |
| `RunEventStore` (ABC) | `runtime/events/store/base.py:17` | [19](./19-tracing-and-observability.md) |
| `RunManager` | `runtime/runs/manager.py:42` | [20](./20-runmanager-worker-streambridge.md) |
| `RunRecord` | `runtime/runs/manager.py:22` | [20](./20-runmanager-worker-streambridge.md) |
| `MemoryStreamBridge` | `runtime/stream_bridge/memory.py:25` | [20](./20-runmanager-worker-streambridge.md) |
| `RunRow / ThreadMetaRow / RunEventRow / FeedbackRow / UserRow` | `persistence/*/model.py` | [21](./21-persistence-and-migrations.md) |
| `AuthMiddleware` | `app/gateway/auth_middleware.py:52` | [22](./22-gateway-channels-client.md) |
| `CSRFMiddleware` | `app/gateway/csrf_middleware.py` | [22](./22-gateway-channels-client.md) |
| `ChannelManager` | `app/channels/manager.py:541` | [22](./22-gateway-channels-client.md) |
| `ChannelService` | `app/channels/service.py:55` | [22](./22-gateway-channels-client.md) |
| `DeerFlowClient` | `client.py:80` | [22](./22-gateway-channels-client.md) |

### 关键函数

| 符号 | 文件 | 章节 |
|------|------|------|
| `make_lead_agent(config)` | `agents/lead_agent/agent.py:343` | [05](./05-agent-factory-dual-track.md) |
| `create_deerflow_agent(...)` | `agents/factory.py:61` | [05](./05-agent-factory-dual-track.md) |
| `_build_middlewares(config, ...)` | `agents/lead_agent/agent.py:240` | [05](./05-agent-factory-dual-track.md) / [06](./06-middleware-chain-overview.md) |
| `_assemble_from_features(feat, ...)` | `agents/factory.py:155` | [05](./05-agent-factory-dual-track.md) |
| `build_lead_runtime_middlewares(...)` | `agents/middlewares/tool_error_handling_middleware.py:129` | [06](./06-middleware-chain-overview.md) |
| `apply_prompt_template(...)` | `agents/lead_agent/prompt.py:768` | [13](./13-system-prompt-assembly.md) |
| `get_available_tools(...)` | `tools/tools.py:44` | [10](./10-tool-assembly-and-dedup.md) |
| `resolve_variable(path, expected_type)` | `reflection/resolvers.py:25` | [03](./03-config-and-reflection.md) |
| `resolve_class(class_path, base_class)` | `reflection/resolvers.py:73` | [03](./03-config-and-reflection.md) |
| `get_app_config()` | `config/app_config.py:358` | [03](./03-config-and-reflection.md) |
| `push_current_app_config / pop_current_app_config` | `config/app_config.py:440/447` | [03](./03-config-and-reflection.md) |
| `replace_virtual_path(path, thread_data)` | `sandbox/tools.py:436` | [08](./08-virtual-paths-and-mounts.md) |
| `mask_local_paths_in_output(output, ...)` | `sandbox/tools.py:502` | [08](./08-virtual-paths-and-mounts.md) |
| `validate_local_tool_path(path, ...)` | `sandbox/tools.py:585` | [08](./08-virtual-paths-and-mounts.md) |
| `ensure_sandbox_initialized(runtime)` | `sandbox/tools.py:1051` | [07](./07-sandbox-abstraction-and-lifecycle.md) / [09](./09-sandbox-tools.md) |
| `make_sync_tool_wrapper(coro, name)` | `tools/sync.py:18` | [10](./10-tool-assembly-and-dedup.md) |
| `get_cached_mcp_tools()` | `mcp/cache.py:82` | [11](./11-mcp-and-tool-search.md) |
| `_hash_tool_calls(tool_calls)` | `agents/middlewares/loop_detection_middleware.py:112` | [18](./18-error-loop-summarization.md) |
| `_stable_tool_key(name, args, fallback)` | `agents/middlewares/loop_detection_middleware.py:69` | [18](./18-error-loop-summarization.md) |
| `run_agent(bridge, mgr, record, ctx, ...)` | `runtime/runs/worker.py:120` | [20](./20-runmanager-worker-streambridge.md) |
| `_rollback_to_pre_run_checkpoint(...)` | `runtime/runs/worker.py:422` | [20](./20-runmanager-worker-streambridge.md) |
| `init_engine(backend, url, ...)` | `persistence/engine.py:57` | [21](./21-persistence-and-migrations.md) |
| `_auto_create_postgres_db(url)` | `persistence/engine.py:30` | [21](./21-persistence-and-migrations.md) |
| `record_external_llm_usage_records(records)` | `runtime/journal.py:405` | [17](./17-subagent-registry-and-token-accounting.md) / [19](./19-tracing-and-observability.md) |
| `pop_cached_subagent_usage(tool_call_id)` | `tools/builtins/task_tool.py:48` | [17](./17-subagent-registry-and-token-accounting.md) |

### 关键装饰器 / 注解

| 符号 | 文件 | 章节 |
|------|------|------|
| `@Next(anchor) / @Prev(anchor)` | `agents/features.py:42/54` | [05](./05-agent-factory-dual-track.md) |
| `@tool("name", parse_docstring=True)` (LangChain) | 各 builtin tool | [09](./09-sandbox-tools.md) / [10](./10-tool-assembly-and-dedup.md) |
| `@require_permission(...)` | `app/gateway/authz.py` | [22](./22-gateway-channels-client.md) |
| `Annotated[type, reducer]` (LangGraph) | `agents/thread_state.py:52/55` | [04](./04-thread-state-and-reducers.md) |
| `@event.listens_for(_engine.sync_engine, "connect")` | `persistence/engine.py:115` | [21](./21-persistence-and-migrations.md) |

---

## 🧭 按 Harness 工程六要素

### 1. 反馈循环

| 章节 | 关键 |
|------|------|
| [06](./06-middleware-chain-overview.md) | 中间件链是反馈机制的载体 |
| [13](./13-system-prompt-assembly.md) | system prompt 是 LLM 看到的"循环规则" |
| [18](./18-error-loop-summarization.md) | LoopDetection / LLM 重试 / Summarization 三件套 |
| [19](./19-tracing-and-observability.md) | RunJournal 让循环可观察 |
| [20](./20-runmanager-worker-streambridge.md) | worker 主循环 + abort_event 协作 |

### 2. 记忆持久化

| 章节 | 关键 |
|------|------|
| [14](./14-memory-middleware-queue-updater.md) | LLM 抽取式长期记忆 |
| [15](./15-dynamic-context-and-prefix-cache.md) | memory 注入策略（不放 system prompt） |
| [21](./21-persistence-and-migrations.md) | 5 张业务表 + per-user 隔离 + Alembic |
| [20](./20-runmanager-worker-streambridge.md) | RunManager 双源 + pre-run snapshot rollback |

### 3. 动态上下文

| 章节 | 关键 |
|------|------|
| [13](./13-system-prompt-assembly.md) | 9 动态片段 + prefix cache 友好 |
| [11](./11-mcp-and-tool-search.md) | Tool Search 让工具按需出现在 context |
| [15](./15-dynamic-context-and-prefix-cache.md) | first HumanMessage 注入 + ID swap |
| [18](./18-error-loop-summarization.md) | Summarization trigger/keep + skill rescue |
| [10](./10-tool-assembly-and-dedup.md) | 按 model/runtime/skill 动态裁工具集 |

### 4. 安全护栏

| 章节 | 关键 |
|------|------|
| [02](./02-harness-app-boundary.md) | harness/app AST 边界 |
| [08](./08-virtual-paths-and-mounts.md) | 三道防御 + reserved prefix + 反向脱敏 |
| [09](./09-sandbox-tools.md) | bash 双重默认禁用 + token 级 shlex 校验 |
| [12](./12-skills-system.md) | .skill 安装 3 层 + LLM-as-reviewer |
| [22](./22-gateway-channels-client.md) | Auth + CSRF + Internal Auth 4 层 |
| [21](./21-persistence-and-migrations.md) | `_AUTO sentinel` secure-by-default |

### 5. 工具集成

| 章节 | 关键 |
|------|------|
| [09](./09-sandbox-tools.md) | 7 沙箱工具统一管线 |
| [10](./10-tool-assembly-and-dedup.md) | 4 类工具来源 + first-wins 去重 + 反射加载 |
| [11](./11-mcp-and-tool-search.md) | MCP server + OAuth + Tool Search |
| [12](./12-skills-system.md) | Skill `allowed-tools` 反向裁剪 |
| [16-17](./16-subagent-orchestration.md) | task 工具 + 自定义 subagent |

### 6. 可观测性

| 章节 | 关键 |
|------|------|
| [19](./19-tracing-and-observability.md) | RunJournal + RunEventStore + LangFuse/LangSmith |
| [17](./17-subagent-registry-and-token-accounting.md) | token 4 桶 + 跨边界回灌 |
| [21](./21-persistence-and-migrations.md) | RunRow 7 token 桶 + convenience 字段 + feedback |
| [22](./22-gateway-channels-client.md) | 15 router 暴露 trace / messages / token-usage 查询 |

---

## 🎓 推荐学习路径

### 路径 A：快速上手（约 6h）

1. [00 总览](./00-overview.md) → [01 启动](./01-startup-and-runtime.md)
2. [03 配置](./03-config-and-reflection.md)
3. [04 ThreadState](./04-thread-state-and-reducers.md) → [05 Agent 工厂](./05-agent-factory-dual-track.md)
4. [06 中间件链](./06-middleware-chain-overview.md)
5. [FAQ](./FAQ.md) 浏览一遍

→ 跑通 `make dev`，理解 agent 是怎么造的。

### 路径 B：定制开发（约 12h）

接路径 A 后继续：

6. [10 工具装配](./10-tool-assembly-and-dedup.md) → [11 MCP](./11-mcp-and-tool-search.md)
7. [12 Skills](./12-skills-system.md) → [13 System Prompt](./13-system-prompt-assembly.md)
8. [16 Subagent 编排](./16-subagent-orchestration.md) → [17 注册表 + Token](./17-subagent-registry-and-token-accounting.md)

→ 能自定义 skill / tool / subagent，prompt 怎么影响 LLM 行为。

### 路径 C：架构深挖（约 20h）

接路径 B 后继续：

9. [02 Harness/App 边界](./02-harness-app-boundary.md)
10. [07-08-09 沙箱三连](./07-sandbox-abstraction-and-lifecycle.md)
11. [14-15 记忆双连](./14-memory-middleware-queue-updater.md)
12. [18-19 错误 + 观测](./18-error-loop-summarization.md)
13. [20-21 运行时 + 持久化](./20-runmanager-worker-streambridge.md)
14. [22 接入层](./22-gateway-channels-client.md)

→ 完全理解 deer-flow 工程深度，能自己设计类似 agent 平台。

---

## 📌 文档约定

- **源码引用格式**：`packages/harness/deerflow/agents/lead_agent/agent.py:343` —— 文件路径 : 行号。
- **章节内跳转**：`[03 § 3.5](./03-config-and-reflection.md)` —— 章节号 + 节号。
- **FAQ 跳转**：`[FAQ Q5](./FAQ.md#q5-...)` —— 问题号。
- **代码块标记**：``python` / ``yaml` / ``mermaid` / ``bash`。
- **图统一用 Mermaid**：`flowchart` / `sequenceDiagram` / `classDiagram` / `stateDiagram-v2`。

---

📚 **本索引覆盖 22 篇 + FAQ + INDEX 共 24 份文档**。任何角度找不到内容，请补充新条目。

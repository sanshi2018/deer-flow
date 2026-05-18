# DeerFlow 高频问答 FAQ

> 跨章节的高频问题集合。每条答案都附章节出处和源码引用——快速定位 + 深挖跳转。

---

## 目录

- [启动与配置](#启动与配置)（Q1-Q6）
- [架构与中间件](#架构与中间件)（Q7-Q12）
- [状态与上下文](#状态与上下文)（Q13-Q17）
- [沙箱与文件](#沙箱与文件)（Q18-Q22）
- [工具与 MCP](#工具与-mcp)（Q23-Q27）
- [Skills & Prompt](#skills--prompt)（Q28-Q31）
- [记忆系统](#记忆系统)（Q32-Q35）
- [Subagent](#subagent)（Q36-Q39）
- [错误与可观测性](#错误与可观测性)（Q40-Q44）
- [运行时与持久化](#运行时与持久化)（Q45-Q49）
- [接入与 Auth](#接入与-auth)（Q50-Q55）
- [部署与运维](#部署与运维)（Q56-Q60）

---

## 启动与配置

### Q1. `make dev` 起了几个进程？怎么验证？

**3 个进程**：Gateway（uvicorn :8001）+ Frontend（Next.js :3000）+ Nginx（:2026）。

```bash
lsof -iTCP -sTCP:LISTEN | grep -E '(2026|3000|8001)'
```

详见 [01 § Q1](./01-startup-and-runtime.md#q1make-dev-一共起了几个进程怎么验证)。

### Q2. 改 `config.yaml` 需要重启 Gateway 吗？

**多数情况不需要**。`make dev` 模式下 uvicorn `--reload-include='*.yaml'` 会重启；非 reload 模式下 `get_app_config()` 的 mtime cache 会自动重载。

**但**：已经构造的 agent 实例不会重建——下次新 thread 才生效。

详见 [03 § 3.3 mtime 失效](./03-config-and-reflection.md)。

### Q3. 改 `extensions_config.json` 呢？

**不会自动生效**——MCP cache 用 mtime 失效，但 ExtensionsConfig singleton 没有 mtime。需要：

- 通过 `/api/mcp/config` PUT 端点修改（会主动 reset cache），或
- 重启 Gateway，或
- 调 `reset_extensions_config()` + `reset_mcp_tools_cache()` 手动 reset。

详见 [11 § 3.1-3.2 MCP 双层缓存](./11-mcp-and-tool-search.md)。

### Q4. `config.yaml` 应该放在哪？

**项目根目录优先**（不是 backend/）。5 级优先级（[03 § 3.2](./03-config-and-reflection.md)）：

1. 显式参数 `config_path`
2. `DEER_FLOW_CONFIG_PATH` 环境变量
3. `project_root() / "config.yaml"`（cwd 决定）
4. `backend/config.yaml`（legacy）
5. 项目根 `config.yaml`（legacy 兜底）

### Q5. `$OPENAI_API_KEY` 怎么解析？

`AppConfig.resolve_env_variables` 递归处理 dict / list / string——字符串以 `$` 开头时 `os.getenv` 替换。缺失时 raise（AppConfig）或填空串（ExtensionsConfig，给 MCP server 更宽容）。

详见 [03 § 3.3 `$VAR` 递归解析](./03-config-and-reflection.md)。

### Q6. `use: deerflow.sandbox.tools:bash_tool` 这种字符串是怎么变成工具的？

通过 `resolve_variable` —— `importlib.import_module + getattr` 反射加载。缺包时给可执行的 `uv add ...` 提示（`MODULE_TO_PACKAGE_HINTS`）。

详见 [03 § 3.5 反射装配](./03-config-and-reflection.md)。

---

## 架构与中间件

### Q7. 中间件链一共几个？怎么改顺序？

**14-18 个**（按 config 开关数）。装配在两个函数：

- `build_lead_runtime_middlewares`（`tool_error_handling_middleware.py:129`）—— 前 8 段。
- `_build_middlewares`（`lead_agent/agent.py:240`）—— 后 11 段。

**改顺序请用 `create_deerflow_agent` 的 `@Next/@Prev` 装饰器**——不要直接改源码顺序。

详见 [06 § 2.2 全景表](./06-middleware-chain-overview.md) + [05 § 3.5 `@Next/@Prev`](./05-agent-factory-dual-track.md)。

### Q8. 为什么 `ClarificationMiddleware` 必须最后？

它通过 `Command(goto=END)` 中断 agent 执行——必须等其它中间件做完该做的事，否则后续的 token 计量、memory 抽取等会被截断。

**物理保证**：

- 应用工厂：`_build_middlewares` 最后一行硬编码 `middlewares.append(ClarificationMiddleware())`。
- SDK 工厂：`_assemble_from_features` 在 extras 插入后主动 pop+append 保证 tail（[05 § 3.6](./05-agent-factory-dual-track.md)）。

### Q9. `harness/` 和 `app/` 的边界是什么？

**铁律**：app 可以 import harness，harness 永远不能 import app。由 `tests/test_harness_boundary.py` 47 行 AST 扫描在 CI 物理强制。

依赖反转用 `CurrentUser(Protocol)` —— harness 不知道 app 的 `User` 类，只要求"有 `.id: str`"。

详见 [02 全章](./02-harness-app-boundary.md)。

### Q10. `make_lead_agent` 和 `create_deerflow_agent` 区别？

- **`make_lead_agent(config)`**：应用工厂，签名是 LangGraph CLI 兼容的；读全局 AppConfig；自动加载 skill / model / agent_name。
- **`create_deerflow_agent(model, tools, ..., features=RuntimeFeatures)`**：SDK 工厂；不读 yaml；纯参数化。

详见 [05 全章](./05-agent-factory-dual-track.md)。

### Q11. `wrap_model_call` 和 `before_model + after_model` 区别？

- **`before_model / after_model`**：只能改 state，不能影响 LLM 调用本身。
- **`wrap_model_call`**：可以重试、跳过、改请求/响应（拦截调用）。

`LLMErrorHandlingMiddleware` 用 `wrap_model_call` 是因为它要 catch 异常并转 fallback——`before_model` 做不到（异常在 model 调用内部）。

详见 [06 § Q2](./06-middleware-chain-overview.md)。

### Q12. 自定义中间件怎么插？

**两条路径**：

- 通过 `create_deerflow_agent(extra_middleware=[...])`+ `@Next(X)` / `@Prev(X)` 装饰器精确插入。
- 通过 `_build_middlewares(..., custom_middlewares=[...])`（目前主要在 `DeerFlowClient` 暴露，[05 § Q2](./05-agent-factory-dual-track.md)）。

---

## 状态与上下文

### Q13. ThreadState vs RunContext vs Configurable 有什么区别？

| 维度 | ThreadState | RunContext | RunnableConfig.configurable |
|------|-------------|-----------|----------------------------|
| 形态 | TypedDict | dataclass | dict |
| 生命周期 | 跨节点共享 + checkpointer 持久化 | 一次 run | 一次 invoke/stream |
| 持久化 | ✅ | ❌ | ❌ |
| 例子 | messages / sandbox / artifacts | checkpointer / store / event_store | thread_id / model_name / is_plan_mode |

判断口诀：
- 要让"下次对话能看到"——用 ThreadState。
- 要让"中间件能拿全局单例"——用 RunContext。
- 要让"这次请求换个 model"——用 configurable。

详见 [04 § Q6](./04-thread-state-and-reducers.md)。

### Q14. `messages` 字段的 reducer 是什么？

LangChain 内置 `add_messages`——按 message id 去重，同 id 后者覆盖前者（支持 LLM streaming chunks 累积）。

详见 [04 § Q2](./04-thread-state-and-reducers.md)。

### Q15. `merge_viewed_images` 为什么有"空 dict 等于清空"特例？

`ViewImageMiddleware` 看完图后要把 base64 数据清掉（避免每轮 prompt 越来越大）。但 LangGraph reducer 默认行为是 merge——直接 `return {}` 会被忽略。

deer-flow 在 reducer 里加 `if len(new) == 0: return {}` 特例 + 在 docstring 明确说明这个 behavior（行 36-37）。

详见 [04 § 3.5](./04-thread-state-and-reducers.md)。

### Q16. system prompt 为什么不放当前日期 / memory？

为了 **prompt prefix cache 友好**——OpenAI/Anthropic/Google 的 prompt cache 要求前缀逐字节相同。系统 prompt 100% 静态才能让 cache 在不同用户、不同 turn 都命中。

memory + date 通过 `DynamicContextMiddleware` 注入到 first HumanMessage 的 `<system-reminder>`，且**注入后不再变**（frozen snapshot）—— 这部分也能命中 cache。

详见 [13 § 3.2 prefix-cache 友好](./13-system-prompt-assembly.md) + [15 全章](./15-dynamic-context-and-prefix-cache.md)。

### Q17. `_inject` 的"ID swap"具体怎么工作？

reminder 拿原 message 的 `id`、user content 拿 `{id}__user`。LangGraph 的 `add_messages` reducer 看到 same id 走 in-place replace、新 id 走 append——实现"reminder 挤到原 message 前面、位置不变"。

详见 [15 § 3.2 ID swap](./15-dynamic-context-and-prefix-cache.md)。

---

## 沙箱与文件

### Q18. LocalSandbox 模式安全吗？

**有限场景下安全**——bash 默认禁用（`config.sandbox.allow_host_bash` 默认 `false`），文件操作有 5 类前缀访问矩阵 + 路径白名单 + relative_to 防穿越。

但 `allow_host_bash: true` 会让 LLM 直接在宿主机跑命令——只在完全可信的开发场景用。

详见 [07 § 3.4 LocalSandboxProvider](./07-sandbox-abstraction-and-lifecycle.md) + [09 § 3.3 bash 4 道工序](./09-sandbox-tools.md)。

### Q19. 怎么换成 Aio Sandbox（Docker）？

改 `config.yaml`：

```yaml
sandbox:
  use: deerflow.community.aio_sandbox:AioSandboxProvider
  image: enterprise-public-cn-beijing.cr.volces.com/vefaas-public/all-in-one-sandbox:latest
  replicas: 3
  idle_timeout: 600
```

重启 Gateway——`get_sandbox_provider` 反射加载新 class。

详见 [07 § 3.5 AioSandboxProvider](./07-sandbox-abstraction-and-lifecycle.md)。

### Q20. `/mnt/user-data/workspace/foo.csv` 怎么映射到真实文件？

- **Local 模式**：`replace_virtual_path` 翻译到 `.deer-flow/users/{user_id}/threads/{thread_id}/user-data/workspace/foo.csv`。
- **Aio 模式**：sandbox 容器内 mount 了同名路径，直接读写。

`mask_local_paths_in_output` 反向脱敏——traceback 含的物理路径会被替换回虚拟路径，防 user_id / thread_id 泄露给 LLM。

详见 [08 § 3.4-3.6 双向翻译](./08-virtual-paths-and-mounts.md)。

### Q21. 自定义 mount 怎么配？什么限制？

```yaml
sandbox:
  mounts:
    - host_path: /Users/sanshi/datasets   # 必须 absolute
      container_path: /mnt/datasets        # 必须 absolute, 不能覆盖 reserved prefix
      read_only: true
```

**`_RESERVED_CONTAINER_PREFIXES`**：`/mnt/user-data / /mnt/skills / /mnt/acp-workspace` 三类系统挂载点不能被覆盖。启动期 warning + skip。

详见 [08 § 3.8 reserved prefix](./08-virtual-paths-and-mounts.md)。

### Q22. `bash_tool` 在 LocalSandbox 下为什么默认看不见？

`get_available_tools` 装 tools 时调 `is_host_bash_allowed(config)` 过滤——默认 false 时直接不挂 bash 工具（schema 里都没有）。

即使 LLM 试图调，工具内部还有第二道防御 `if not is_host_bash_allowed(): return Error`。

**双重默认禁用** —— 详见 [09 § 3.3 + 10 § 3.1](./09-sandbox-tools.md)。

---

## 工具与 MCP

### Q23. tool name 不一致怎么办？

`get_available_tools` 会 warning：

```
WARNING Tool name mismatch: config name 'my_alias' does not match tool .name 'real_name'
```

**实际生效的是 `tool.name`**（LLM 看到的），不是 yaml 的 `name`。yaml 的 name 只是日志可读性。

详见 [10 § 3.2](./10-tool-assembly-and-dedup.md)。

### Q24. 4 类工具来源的优先级？

`config-loaded > builtin > MCP > ACP`——`first-wins` 去重，硬编码在 list 拼接顺序里（`get_available_tools`）。

详见 [10 § 3.5](./10-tool-assembly-and-dedup.md)。

### Q25. `make_sync_tool_wrapper` 解决什么问题？

MCP 工具是 100% async，但 DeerFlowClient.chat 和某些测试是 sync 调用——sync 调 async 工具会失败。

`make_sync_tool_wrapper` 用共享 ThreadPoolExecutor + 跨 event loop 抛 `asyncio.run` 实现安全包装。**37 行解决 async-to-sync 桥**——值得参考的小型范式。

详见 [10 § 3.3](./10-tool-assembly-and-dedup.md)。

### Q26. MCP server 启用了但工具不更新？

mtime cache 仅检测 `extensions_config.json` 的文件改动——如果你改了 server 内部配置而 json 文件没变，cache 不会失效。

强制刷新：调 `reset_mcp_tools_cache()` 或重启 Gateway。

详见 [11 § 3.1-3.2](./11-mcp-and-tool-search.md)。

### Q27. tool_search 怎么用？解决什么？

`config.tool_search.enabled: true` 后，MCP 工具不直接进 schema——只在 prompt 列 name + description。LLM 调 `tool_search(query)` 查询拿完整 schema。

**目的**：把"几十个 MCP 工具的 schema" 从 prompt 上下文里挪出去——节省 17k-47k tokens。3 种 query：`select:n1,n2` / `+keyword rest` / 正则模糊。

详见 [11 § 3.6-3.7](./11-mcp-and-tool-search.md)。

---

## Skills & Prompt

### Q28. Skill 怎么写？

最小目录结构：

```
skills/custom/my-skill/
└── SKILL.md
```

`SKILL.md` 至少含 YAML frontmatter（`name` + `description`）：

```yaml
---
name: my-skill
description: Trigger this skill when ...
allowed-tools: [read_file, str_replace]   # 可选,会反向裁剪 agent 工具
---
# Skill Body
...
```

详见 [12 § 3.1-3.5](./12-skills-system.md)。

### Q29. `allowed-tools: []` 和不写有什么区别？

| 写法 | 单 skill 启用 | 多 skill 共存 |
|------|------------|------------|
| 不写 | allow-all | 如果其它 skill 有声明 → 这个 skill 贡献 0 |
| `[]` 空 list | 没有任何工具 | 强制"严格模式" |
| `[a, b]` | 只允许 a, b | 取并集 |

**核心规则**：任何明示声明胜出，legacy 不打破限制。

详见 [12 § 3.4 三态语义](./12-skills-system.md)。

### Q30. 改 SKILL.md 后多久生效？

`load_skills` 每次都重扫磁盘 + 重读 `extensions_config.json`——下次 agent 装配（新 thread / 改 model）生效。`_get_cached_skills_prompt_section` 的 `@lru_cache` 会因为 skill_signature 变化自然 miss。

详见 [12 § Q1](./12-skills-system.md)。

### Q31. system prompt 一共多长？

- 非自定义 agent + 默认配置（subagent 关）：约 6000-8000 字符。
- 全开（subagent + 多 skill + tool_search + ACP）：可能到 15000 字符。

约 2000-4000 tokens——对 GPT-4 128k context 占比 1.5-3%。**prompt prefix cache 让重发成本接近零**。

详见 [13 § Q1-Q2](./13-system-prompt-assembly.md)。

---

## 记忆系统

### Q32. memory 多久后生效？

默认 30 秒 debounce（`memory.debounce_seconds`）——用户连续聊几轮 → 最后一轮结束 30s 后 LLM 抽事实写盘。下一次对话注入 memory。

详见 [14 § 3.4 MemoryUpdateQueue](./14-memory-middleware-queue-updater.md)。

### Q33. 为什么 `user_id` 必须在 enqueue 时同步抓？

`threading.Timer` 的 callback 在新线程跑，ContextVar **不传播**——抓晚了 `get_effective_user_id()` 返回 None / "default"。必须在请求线程同步抓塞进 `ConversationContext`。

这是 02 篇 ContextVar 边界教训的反面——**ContextVar 只对 asyncio task 自动传播，不对 raw threading**。

详见 [14 § 3.6 + 5.3](./14-memory-middleware-queue-updater.md)。

### Q34. memory 抽出的 fact 怎么淘汰？

按 `max_facts`（默认 100）+ **按 confidence 排序保留 top N**——淘汰最不确定的，不是淘汰最旧的。

`fact_confidence_threshold` 默认 0.7——低于阈值的新 fact 直接丢弃。

详见 [14 § 3.7](./14-memory-middleware-queue-updater.md)。

### Q35. 想关掉 memory 但保留收集？

`MemoryConfig.enabled` vs `injection_enabled` 是两个独立开关：

- `enabled=true, injection_enabled=false`：收集但不注入 prompt（隐私场景）。
- `enabled=false, injection_enabled=true`：不收集，也无内容可注入。
- 都 true：正常模式。

详见 [15 § 3.4](./15-dynamic-context-and-prefix-cache.md)。

---

## Subagent

### Q36. 怎么定义自定义 subagent 类型？

`config.yaml` 加：

```yaml
subagents:
  enabled: true
  custom_agents:
    my-reviewer:
      description: "When to use this subagent"
      system_prompt: "You are a reviewer..."
      tools: [read_file, grep, glob]
      skills: [code-documentation]
      model: doubao-seed-1.8
      max_turns: 40
      timeout_seconds: 600
```

完整 demo 看 [17 § 6.3](./17-subagent-registry-and-token-accounting.md)。

### Q37. subagent 内部能调 task 吗？

**不能**——`task_tool` 装 tools 时显式 `subagent_enabled=False`，且 built-in subagent 的 `disallowed_tools=["task", ...]`。**防递归 nesting 双重保护**。

详见 [16 § Q3](./16-subagent-orchestration.md) + [17 § 3.2](./17-subagent-registry-and-token-accounting.md)。

### Q38. subagent 跑出来的 token 在哪里能看到？

**两条路径**：

- **trace 路径**：累积到父 RunJournal 的 `_subagent_tokens` 桶——前端 trace UI 按 caller 分类显示。
- **state 路径**：通过 `_subagent_usage_cache` → `TokenUsageMiddleware._apply` 反查回灌到 dispatch AIMessage 的 `usage_metadata`——前端 per-message 显示。

两条路径独立但 total 一致。完整端到端 demo 看 [19 § 6.3](./19-tracing-and-observability.md)。

### Q39. subagent 超时 cancel 真的会停吗？

不立刻停——`request_cancel_background_task` 设 `cancel_event.set()` 是**协作式**信号。subagent 主循环在下一个 yield point 检查到 → break。如果卡在 LLM 调用里要等 LLM 返回（最多几十秒）。

最终防御：`task.cancel()` 硬抛 `CancelledError`——但某些 await 点不响应。**4 步取消处理**：cancel_event → `asyncio.shield` 等终态 → 报 token usage → `_schedule_deferred_subagent_cleanup`。

详见 [16 § 3.6 取消处理](./16-subagent-orchestration.md)。

---

## 错误与可观测性

### Q40. agent 卡死循环（一直调同一工具）怎么办？

`LoopDetectionMiddleware` 自动处理：

- 第 3 次同 hash → 注入 `_WARNING_MSG` 让 LLM 自救。
- 第 5 次 → 强剥 tool_calls，强制 LLM 输出文本答案。

stable key 工具特异性：read_file 按 path + 200 行 bucket、write_file 用完整 args、其它按 `(path, url, query, command, pattern, glob, cmd)` 7 salient 字段。

详见 [18 § 3.3 LoopDetection](./18-error-loop-summarization.md) + [18 § 6.2 实验 ④ stable key 行为](./18-error-loop-summarization.md)。

### Q41. LLM 5xx 自动重试吗？

`LLMErrorHandlingMiddleware` 自动重试**transient 类错误**（HTTP 408/409/425/429/500/502/503/504 或 "server busy" 文本）。最多 3 次 exponential backoff（1/2/4s，cap 8s），Retry-After 头优先。

**不重试**：quota / auth 类（重试也没用）。

熔断：连续失败 ≥ `failure_threshold`（默认 5）→ OPEN 状态 fast fail；`recovery_timeout_sec`（默认 60s）后 HALF_OPEN 单 probe 试水。

详见 [18 § 3.2 LLM Error 三件套](./18-error-loop-summarization.md)。

### Q42. 对话太长被 summarization 折叠后老消息丢了吗？

**是的，永久丢**（除非 checkpointer 保留 history version）。`RemoveMessage(REMOVE_ALL_MESSAGES)` 是 LangGraph 的"清空"指令，state 持久化后老消息永久消失。

**skill rescue** 是部分救援——把"读 skill 文件"的 tool_call bundle 保留不折叠（避免 LLM 再读重复加载）。

详见 [18 § 3.4-3.5 Summarization](./18-error-loop-summarization.md)。

### Q43. trace 在哪里看？

3 个选项：

- **Gateway 本地 store**：`/api/threads/{tid}/runs/{rid}/events` 查询；3 backend 实现（jsonl / db / memory）。
- **LangFuse**：`tracing.enabled_providers: ["langfuse"]` + 配 secret/public key → 推到 LangFuse 服务端。
- **LangSmith**：`tracing.enabled_providers: ["langsmith"]` + 配 project。

LangFuse/LangSmith 和本地 store **并列**收集（callbacks 同源同投递），不替代关系。

详见 [19 § 3.7 tracing/factory.py](./19-tracing-and-observability.md)。

### Q44. token 怎么按 caller 细分？

`RunJournal._identify_caller(tags)` 按 LangChain callback tag 前缀分类：

- `subagent:foo` → `_subagent_tokens`（来源 `SubagentTokenCollector(caller="subagent:foo")`）
- `middleware:summarize` → `_middleware_tokens`（来源 `model.with_config(tags=["middleware:summarize"])`）
- 其它（无 tag） → `_lead_agent_tokens`

详见 [19 § 3.2](./19-tracing-and-observability.md)。

---

## 运行时与持久化

### Q45. cancel 一个 run 有几种方式？

3 种 action：

- **`interrupt`**（默认）：保留 checkpoint，下次可续跑。
- **`rollback`**：恢复到 pre-run snapshot（worker 启动时抓的 checkpoint）。
- **silent cancel**（`create_or_reject` 内部）：新 run 来之前清掉旧 run，status=interrupted。

实现细节 + 完整 rollback 时序见 [20 § 3.8](./20-runmanager-worker-streambridge.md)。

### Q46. SSE 断线重连会丢消息吗？

不会（在 buffer 窗口内）。`MemoryStreamBridge` 保留最近 256 个事件（`queue_maxsize`）。前端用 EventSource API 自动发 `Last-Event-ID` 头——`_resolve_start_offset` 从断点续传。

如果断线太久 buffer 已淘汰 → 从 buffer 起点 replay + warning。

详见 [20 § 3.7](./20-runmanager-worker-streambridge.md)。

### Q47. SQLite 多 worker 安全吗？

WAL 模式允许"多 reader + 单 writer"，5 秒 busy timeout 串行化竞争。**单进程 worker 完全 cope**。

**多 worker 部署推荐换 Postgres**——多个进程同时写 SQLite 会偶尔 busy timeout，且 `_seq_counters` per-process 不一致。

详见 [21 § 3.2 SQLite PRAGMA 三件套](./21-persistence-and-migrations.md) + [21 § Q4](./21-persistence-and-migrations.md)。

### Q48. Alembic 还没有迁移文件，怎么演化 schema？

当前用 `Base.metadata.create_all` **增量演化**：

- ✅ 新表自动建。
- ✅ 加 nullable 字段（SQLite 也 OK）。
- ❌ 删字段 / 改类型——需要 Alembic 写迁移。

deer-flow 现在没遇到破坏性变更（用 nullable + index 应对），所以 `versions/` 目录还是 `.gitkeep`。

详见 [21 § 3.4 Alembic 现状](./21-persistence-and-migrations.md)。

### Q49. `user_id` 为什么 nullable？怎么强制隔离？

**nullable** 是为了"auth 之前的老数据"兼容（NULL = 无 owner）。新数据由 AuthMiddleware 自动填。

强制隔离 = **repository 层 `_AUTO` sentinel**：

- 默认值是 `_AUTO` → 从 ContextVar 拿 user_id；没 user_id → raise。
- 显式 `user_id="xxx"` → 用这个。
- 显式 `user_id=None` → 不过滤（仅迁移脚本 / admin CLI）。

详见 [21 § 3.5 + 5.4](./21-persistence-and-migrations.md)。

---

## 接入与 Auth

### Q50. Gateway 一共多少 router？

**15 个**：auth / models / mcp / memory / skills / artifacts / uploads / threads / agents / suggestions / channels / assistants_compat / thread_runs / runs / feedback。

详见 [22 § 3.1 router 职责矩阵](./22-gateway-channels-client.md)。

### Q51. CSRF token 必填吗？什么时候要带？

**state-changing 请求必填**（POST / PUT / DELETE / PATCH），GET / HEAD / OPTIONS 不需要。

例外：`/api/v1/auth/me`（豁免）+ 4 个 auth endpoint 首次调用豁免（你还没登录，CSRF 不可能有）。

详见 [22 § 3.3 Double Submit Cookie](./22-gateway-channels-client.md)。

### Q52. 改密码后旧 JWT 怎么处理？

`UserRow.token_version: int` —— JWT 里编码当时的 version。改密码时 `token_version += 1`。下次校验 token 对比 `token.ver == user.token_version`，不一致 → 401。

**避免 JWT blacklist DB**——零额外查询。

详见 [21 § 3.3 UserRow + 5.3](./21-persistence-and-migrations.md)。

### Q53. IM Channels 是怎么调 agent 的？

通过 `langgraph-sdk` HTTP client **自调自己 Gateway**——和前端走完全同一条 API 链路。`INTERNAL_AUTH_HEADER_NAME` header 走内部认证，不需要用户登录。

`ChannelManager` 接收 IM 消息 → 通过 `ChannelStore` 映射到 deer-flow thread_id → 调 `runs.stream` → 翻译 SSE chunk 回 IM 协议消息。

详见 [22 § 3.5-3.7](./22-gateway-channels-client.md)。

### Q54. DeerFlowClient vs HTTP API 怎么选？

| 场景 | 建议 |
|------|------|
| 浏览器前端 | HTTP API（必须） |
| 第三方系统调 | HTTP API（远程） |
| Python 脚本 / 测试 / Notebook | DeerFlowClient（直接 import，无网络） |
| 多用户多租户 | HTTP API（DeerFlowClient 是单进程） |
| 想要细粒度控制 middleware | DeerFlowClient（`middlewares=[...]` 参数） |

详见 [22 § 1 + 2.3](./22-gateway-channels-client.md)。

### Q55. DeerFlowClient 多轮对话怎么传？

构造时传 `checkpointer`：

```python
from langgraph.checkpoint.memory import InMemorySaver
client = DeerFlowClient(checkpointer=InMemorySaver())
client.chat("Hello", thread_id="t1")
client.chat("Continue", thread_id="t1")   # 看到上下文
```

不传 checkpointer → 每次 stateless（thread_id 只用于文件隔离）。

---

## 部署与运维

### Q56. `make dev` vs `make start` 区别？

- `make dev`：dev 模式 + uvicorn `--reload`（改 yaml/.env 自动重启） + 排除 `sandbox/ + .deer-flow/` 高频写入。
- `make start`：prod 模式 + 无 reload + 前端用 `pnpm run preview` 跑预构建。

长跑测试 / benchmark 一定用 `make start`——reload 干扰性能数据。

详见 [01 § Q6](./01-startup-and-runtime.md)。

### Q57. Docker 部署时 `DEER_FLOW_HOST_BASE_DIR` 是什么？

Gateway 容器内 `base_dir = /app/.deer-flow`，但**宿主机 Docker daemon 看不到这个路径**（它在容器外）。如果不设这个 env，sandbox container mount 会用容器内路径——daemon 找不到 → 创建空目录 mount → agent 看不到文件。

**解法**：

```yaml
environment:
  - DEER_FLOW_HOST_BASE_DIR=${PWD}/.deer-flow  # 宿主机绝对路径
```

详见 [08 § 3.2 DooD 陷阱](./08-virtual-paths-and-mounts.md)。

### Q58. 用 Postgres 但忘建数据库怎么办？

不用怕——`init_engine` 检测到 "does not exist" 错误时会自动连 maintenance db `postgres` 用 AUTOCOMMIT 模式发 `CREATE DATABASE`，然后重建 engine 重试。

详见 [21 § 3.1 `_auto_create_postgres_db`](./21-persistence-and-migrations.md)。

### Q59. 怎么排查 token 算多了 / 算少了？

逐层检查：

1. **subagent 内**：是否 `SubagentTokenCollector` 接到 callback？看 `result.token_usage_records`。
2. **task_tool**：是否调了 `_cache_subagent_usage` + `_report_subagent_usage`？看 logs。
3. **父 RunJournal**：是否累加到 `_subagent_tokens`？检查 `_counted_external_source_ids`。
4. **TokenUsageMiddleware._apply**：是否反查到 dispatch AIMessage 并合并 `usage_metadata`？看 `_subagent_usage_cache` 是否 pop。

端到端 demo 见 [19 § 6.3](./19-tracing-and-observability.md)。

### Q60. 用户改了 skill 怎么让 LLM 立刻看到？

调 `clear_skills_system_prompt_cache()`——清 `_get_cached_skills_prompt_section.cache_clear()` + 重置 `_enabled_skills_cache` + 后台线程重新 load。Gateway API 的 `/api/skills/{name}` PUT / `POST /install` 内部都会调。

如果手改文件不通过 API → 重启 Gateway 或脚本调 `clear_skills_system_prompt_cache()`。

详见 [13 § 3.5 skills 预热](./13-system-prompt-assembly.md)。

---

## 📚 没找到答案？

按章节深挖：

- 启动 / 配置 → [01](./01-startup-and-runtime.md) / [02](./02-harness-app-boundary.md) / [03](./03-config-and-reflection.md)
- 状态 / Agent / 中间件 → [04](./04-thread-state-and-reducers.md) / [05](./05-agent-factory-dual-track.md) / [06](./06-middleware-chain-overview.md)
- 沙箱 / 文件 / 工具 → [07](./07-sandbox-abstraction-and-lifecycle.md) / [08](./08-virtual-paths-and-mounts.md) / [09](./09-sandbox-tools.md)
- 工具集成 / MCP → [10](./10-tool-assembly-and-dedup.md) / [11](./11-mcp-and-tool-search.md)
- Skills / Prompt → [12](./12-skills-system.md) / [13](./13-system-prompt-assembly.md)
- Memory / 动态上下文 → [14](./14-memory-middleware-queue-updater.md) / [15](./15-dynamic-context-and-prefix-cache.md)
- Subagent → [16](./16-subagent-orchestration.md) / [17](./17-subagent-registry-and-token-accounting.md)
- 错误 / 观测 → [18](./18-error-loop-summarization.md) / [19](./19-tracing-and-observability.md)
- 运行时 / 持久化 → [20](./20-runmanager-worker-streambridge.md) / [21](./21-persistence-and-migrations.md)
- 接入层 → [22](./22-gateway-channels-client.md)

完整索引见 [INDEX.md](./INDEX.md)。

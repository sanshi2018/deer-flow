# 10 · 工具装配：get_available_tools + 反射 + 去重 + 同步包装

> 09 篇讲了 7 个沙箱工具的内部实现。但 LLM 看到的工具是怎么"装"上 agent 的？deer-flow 把"工具来源"分成 4 类——config 定义、builtin、MCP、ACP——再加 subagent 的 `task` 和 vision 的 `view_image` 按需注入。这个**装配**才是 agent 工具系统的真实入口。

---

## 1. 模块定位（Why this matters）

`get_available_tools(...)` 是个看似简单实则关键的入口函数——它把 4 类来源 + 多个条件开关混在一起，最终输出 LLM 看到的 tool schema 列表。如果它出问题，agent 要么调不到工具、要么调错工具、要么看到一堆同名重复 schema 而不知所措。

deer-flow 在这里的工程难点：

1. **4 类工具来源各自的接入方式不同**：config 工具走反射加载、builtin 工具直接 import、MCP 工具走异步 cache、ACP 工具按配置动态生成。
2. **同名工具的优先级要明确**：config-loaded > builtin > MCP > ACP。issue #1803（同名工具让 LLM 看到歧义 schema）就是没去重前的真实事故。
3. **同步/异步差异**：MCP 工具是 `async def`，但 deer-flow agent 的某些调用路径（DeerFlowClient 同步 `chat` 接口、单元测试）只走同步——必须给所有 async-only 工具加一层 sync wrapper。
4. **条件开关的组合爆炸**：vision 按 model 能力开、subagent 按 runtime config 开、tool_search 按 config 开、host bash 按 config 开、ACP 按是否有 agent 配置开——`get_available_tools` 一次调用要处理 5+ 个开关。

不读这一章会错过 4 个关键认知：

1. **工具名是 LLM 调度的 key——`tool.name` 而非 `config.name`**：deer-flow 在 yaml 里写的 `- name: web_search` 不会真的成为 LLM 看到的名字，**LLM 看到的是 `resolve_variable(use)` 拿到的对象自己的 `.name` 属性**。两者不一致时会 warning 但不会 fail。这是 issue #1803 的根因。
2. **`make_sync_tool_wrapper` 是个微型 async-to-sync 桥**：用共享 ThreadPoolExecutor 安全跨越 "已经在 event loop 里 / 不在 event loop 里" 两种调用上下文。
3. **`include_mcp=True` 是 `get_available_tools` 的默认值**——MCP 工具会在每次装配 agent 时被读取（其实是 cache 命中），所以 yaml 改 mcp config 几乎立刻生效。
4. **subagent / vision / ACP 都是 `extra_tools` 模式**：先装基础工具列表，再按条件 append 进去——所以这些"扩展工具"永远在末尾，不会破坏前面的稳定顺序。

对应到 Harness 六要素：本章对应**工具集成 + 反射装配 + 防错设计**。

---

## 2. 源码地图（Source Map）

### 2.1 关键文件清单

| 路径 | 角色 |
|------|------|
| [`packages/harness/deerflow/tools/tools.py`](../packages/harness/deerflow/tools/tools.py) | `get_available_tools` 主体（220 行） |
| [`packages/harness/deerflow/tools/__init__.py`](../packages/harness/deerflow/tools/__init__.py) | 模块出口 + `__getattr__` 延迟 export skill_manage_tool |
| [`packages/harness/deerflow/tools/sync.py`](../packages/harness/deerflow/tools/sync.py) | `make_sync_tool_wrapper`（37 行 async-to-sync 桥） |
| [`packages/harness/deerflow/tools/types.py`](../packages/harness/deerflow/tools/types.py) | `Runtime = ToolRuntime[dict, ThreadState]` 类型别名 |
| [`packages/harness/deerflow/tools/builtins/__init__.py`](../packages/harness/deerflow/tools/builtins/__init__.py) | 6 个 builtin 工具的 import 中心 |
| [`packages/harness/deerflow/tools/skill_manage_tool.py`](../packages/harness/deerflow/tools/skill_manage_tool.py) | skill_evolution 开关下的"可让 LLM 自我升级 skill" |
| [`packages/harness/deerflow/sandbox/security.py`](../packages/harness/deerflow/sandbox/security.py) | `is_host_bash_allowed` 决定 bash 是否进 tool list |
| [`packages/harness/deerflow/reflection/resolvers.py`](../packages/harness/deerflow/reflection/resolvers.py) | 03 篇讲过的反射加载 |
| [`packages/harness/deerflow/mcp/cache.py`](../packages/harness/deerflow/mcp/cache.py) | `get_cached_mcp_tools` 异步初始化 + 缓存 |
| [`packages/harness/deerflow/tools/builtins/invoke_acp_agent_tool.py`](../packages/harness/deerflow/tools/builtins/invoke_acp_agent_tool.py) | `build_invoke_acp_agent_tool` 工厂 |

### 2.2 关键符号速查表

| 符号 | 文件:行 | 一句话职责 |
|------|---------|-----------|
| `get_available_tools(groups, include_mcp, model_name, subagent_enabled, app_config)` | `tools/tools.py:44` | 唯一公开入口 |
| `BUILTIN_TOOLS` | `tools/tools.py:15` | `[present_file_tool, ask_clarification_tool]` 总是加 |
| `SUBAGENT_TOOLS` | `tools/tools.py:20` | `[task_tool]` 按 runtime 开关加 |
| `_is_host_bash_tool(tool)` | `tools/tools.py:26` | 识别 yaml 里的 bash 工具配置 |
| `_ensure_sync_invocable_tool(tool)` | `tools/tools.py:37` | 给 async-only tool 装上同步 fallback |
| `make_sync_tool_wrapper(coro, tool_name)` | `tools/sync.py:18` | 共享 ThreadPoolExecutor 跨 event loop |
| `_SYNC_TOOL_EXECUTOR` | `tools/sync.py:13` | max_workers=10 + atexit shutdown |
| `Runtime` | `tools/types.py:11` | `ToolRuntime[dict[str, Any], ThreadState]` |
| `get_cached_mcp_tools()` | `mcp/cache.py:82` | MCP 工具单例缓存 |
| `set_deferred_registry(registry)` | `tools/builtins/tool_search.py` | tool_search 开关下的"延迟注册" |
| `build_invoke_acp_agent_tool(acp_agents)` | `tools/builtins/invoke_acp_agent_tool.py` | 把多个 ACP agent 配置打包成一个 LLM 可调的工具 |
| `is_host_bash_allowed(config)` | `sandbox/security.py` | Local 模式 bash 总开关 |

### 2.3 工具装配的合流图

```mermaid
flowchart TB
    Cfg[config.yaml<br/>tools: 列表] --> Reflect[resolve_variable(use)<br/>→ BaseTool 实例]
    Reflect --> EnsureSync1[_ensure_sync_invocable_tool]
    EnsureSync1 --> Loaded[loaded_tools]

    Builtin[BUILTIN_TOOLS<br/>present_files + ask_clarification] --> BuiltinList
    Skill{skill_evolution<br/>.enabled?}
    BuiltinList --> Skill
    Skill -- yes --> AddSkillMgr[+ skill_manage_tool]
    Skill -- no --> NoOp1[ ]
    AddSkillMgr --> Builtin2

    Subagent{subagent_enabled<br/>(runtime)?}
    Builtin2 --> Subagent
    Subagent -- yes --> AddTask[+ task_tool]
    Subagent -- no --> NoOp2[ ]
    AddTask --> Builtin3

    Vision{model.supports_vision?}
    Builtin3 --> Vision
    Vision -- yes --> AddView[+ view_image_tool]
    Vision -- no --> NoOp3[ ]
    AddView --> BuiltinFinal[final builtin_tools]

    Mcp{include_mcp<br/>+ enabled servers?}
    Mcp -- yes --> McpCache[get_cached_mcp_tools]
    McpCache --> ToolSearch{tool_search<br/>.enabled?}
    ToolSearch -- yes --> Defer[DeferredToolRegistry<br/>+ tool_search builtin]
    ToolSearch -- no --> McpDirect[直接进 list]

    Acp{config.acp_agents?}
    Acp -- yes --> BuildAcp[build_invoke_acp_agent_tool]

    Loaded --> Merge[Merge + Dedup<br/>config > builtin > mcp > acp]
    BuiltinFinal --> Merge
    Defer --> Merge
    McpDirect --> Merge
    BuildAcp --> Merge
    Merge --> Output[(unique_tools)]
```

### 2.4 4 类工具的属性对比

| 维度 | config 定义 | builtin | MCP | ACP |
|------|------------|---------|-----|-----|
| 来源 | yaml 字符串 `use:` | Python import | 外部进程（stdio/sse/http） | 外部进程 (codex/...) |
| 加载时机 | 每次 `get_available_tools` | 模块 import 时 | 第一次用时 + cache | 装配时 |
| 反射？ | ✅ `resolve_variable` | ❌ 直接 import | ❌ `MultiServerMCPClient` | ❌ 工厂函数 |
| 名字来源 | tool 对象的 `.name`（不是 config.name） | 装饰器声明 | server 返回 | 配置里写 |
| 是否 async | 视实现 | 多数同步 | **全部 async** | sync 包装 |
| 数量 | 可控（yaml 列） | 6 个 | 不可预测 | 视 acp_agents 数 |
| 失败兜底 | 反射缺包 ImportError | 不会失败 | warning + 跳过 | warning + 跳过 |

---

## 3. 核心逻辑精读（Deep Dive）

### 3.1 整体结构：`get_available_tools` 的 6 段

```python
# packages/harness/deerflow/tools/tools.py:44-65
def get_available_tools(
    groups: list[str] | None = None,
    include_mcp: bool = True,
    model_name: str | None = None,
    subagent_enabled: bool = False,
    *,
    app_config: AppConfig | None = None,
) -> list[BaseTool]:
    """Get all available tools from config."""
    config = app_config or get_app_config()
    tool_configs = [tool for tool in config.tools if groups is None or tool.group in groups]
```

**参数解读**：

| 参数 | 谁传 | 用途 |
|------|------|------|
| `groups=None` | `make_lead_agent` 传 `agent_config.tool_groups`（自定义 agent 可以限制工具集） | filter `config.tools` |
| `include_mcp=True` | 默认 True | 是否拉 MCP 工具 |
| `model_name=None` | 上游传当前 model | 决定 vision 工具是否加 |
| `subagent_enabled=False` | 从 RuntimeConfig 来 | 决定 task 工具是否加 |
| `app_config=None` | 兼容老调用 | per-request 覆盖配置 |

```python
# 第 1 段：从 config 加载，按 group 过滤
config = app_config or get_app_config()
tool_configs = [tool for tool in config.tools if groups is None or tool.group in groups]

# 第 2 段：Local 模式默认过滤掉 bash
if not is_host_bash_allowed(config):
    tool_configs = [tool for tool in tool_configs if not _is_host_bash_tool(tool)]

# 第 3 段：反射加载 + name 警告
loaded_tools_raw = [(cfg, resolve_variable(cfg.use, BaseTool)) for cfg in tool_configs]
for cfg, loaded in loaded_tools_raw:
    if cfg.name != loaded.name:
        logger.warning("Tool name mismatch: ...")
loaded_tools = [_ensure_sync_invocable_tool(t) for _, t in loaded_tools_raw]
```

**3 个易踩坑**：

1. **`groups` 是 OR 过滤**：传 `["web", "file:read"]` 表示"这两个 group 都要"。传 `None` 表示"全部"。
2. **`_is_host_bash_tool` 同时识别两种写法**：`group: "bash"` 和 `use: "deerflow.sandbox.tools:bash_tool"`——只要符合一种就算 bash 工具，被默认过滤。
3. **name mismatch 是 warning 不是 error**：因为有合理的不一致场景——比如 yaml 里写 `name: web_search`、工具对象自己叫 `tavily_search`。前者是用户给的"别名"，后者是 LLM 看到的真实名字。但**LLM 看到的就是真实名字**，warning 提醒用户改 yaml 避免混淆。

### 3.2 反射加载的"工程红线"

```python
# packages/harness/deerflow/tools/tools.py:73-86
loaded_tools_raw = [(cfg, resolve_variable(cfg.use, BaseTool)) for cfg in tool_configs]

# Warn when the config ``name`` field and the tool object's ``.name``
# attribute diverge — this mismatch is the root cause of issue #1803 where
# the LLM receives one name in its tool schema but the runtime router
# recognises a different name, producing "not a valid tool" errors.
for cfg, loaded in loaded_tools_raw:
    if cfg.name != loaded.name:
        logger.warning(
            "Tool name mismatch: config name %r does not match tool .name %r (use: %s). "
            "The tool's own .name will be used for binding.",
            cfg.name,
            loaded.name,
            cfg.use,
        )
```

**issue #1803 的故事**：

某用户在 yaml 写了：

```yaml
- name: web_search
  use: deerflow.community.tavily.tools:web_search_tool
```

而另一份 yaml 同时启用了：

```yaml
- name: web_search
  use: deerflow.community.serper.tools:web_search_tool
```

两者 `loaded.name` 都是 `tavily_search` 和 `serper_search`（不同），但 `cfg.name` 都是 `web_search`。结果是 LLM 看到两个工具都叫 `web_search` → schema 冲突或调度乱套。

deer-flow 的解法是双重：

1. **`loaded.name` 才是真实名字**——LangChain bind tools 用的是 `tool.name`，不是 config.name。所以即使 yaml 起的别名乱，runtime 行为也确定。
2. **warning 告诉用户去改 yaml**——避免认知偏差："我以为 yaml 里改的是真名"。

**`expected_type=BaseTool`** 让反射拿到的对象必须是 `BaseTool` 子类的实例——拿到一个普通函数会立刻 `ValueError`。这是 03 篇 `resolve_variable` 类型校验的实际应用。

### 3.3 `make_sync_tool_wrapper`：37 行的 async-to-sync 桥

```python
# packages/harness/deerflow/tools/sync.py:1-36
"""Utilities for invoking async tools from synchronous agent paths."""
import asyncio
import atexit
import concurrent.futures
import logging
from collections.abc import Callable
from typing import Any

logger = logging.getLogger(__name__)

# Shared thread pool for sync tool invocation in async environments.
_SYNC_TOOL_EXECUTOR = concurrent.futures.ThreadPoolExecutor(max_workers=10, thread_name_prefix="tool-sync")
atexit.register(lambda: _SYNC_TOOL_EXECUTOR.shutdown(wait=False))


def make_sync_tool_wrapper(coro: Callable[..., Any], tool_name: str) -> Callable[..., Any]:
    """Build a synchronous wrapper for an asynchronous tool coroutine."""

    def sync_wrapper(*args: Any, **kwargs: Any) -> Any:
        try:
            loop = asyncio.get_running_loop()
        except RuntimeError:
            loop = None

        try:
            if loop is not None and loop.is_running():
                future = _SYNC_TOOL_EXECUTOR.submit(asyncio.run, coro(*args, **kwargs))
                return future.result()
            return asyncio.run(coro(*args, **kwargs))
        except Exception as e:
            logger.error("Error invoking tool %r via sync wrapper: %s", tool_name, e, exc_info=True)
            raise

    return sync_wrapper
```

**5 个精妙处**：

1. **`get_running_loop()` + `RuntimeError` 判定当前是否在 event loop 里**：
   - 在 event loop 里（agent 通过 `astream/ainvoke` 跑） → 用 `_SYNC_TOOL_EXECUTOR.submit(asyncio.run, ...)` 在**独立线程**里跑 coroutine，避免 "loop already running" 错误。
   - 不在 event loop 里（DeerFlowClient.chat 同步调用 / 单元测试） → 直接 `asyncio.run(coro)`，临时建一个 loop 跑完销毁。
2. **共享 `ThreadPoolExecutor`**：`max_workers=10` 避免每次都新建线程。10 个 worker 对工具调用绝对够（Agent 一般串行，不会 10 个并发工具调用）。
3. **`atexit.register` shutdown**：进程退出时 `wait=False` 不等剩余 task，干净退场。
4. **不 catch 异常仅记日志**：`logger.error(..., exc_info=True)` 然后 `raise`——让上层 `ToolErrorHandlingMiddleware`（06 篇）兜底转 ToolMessage，不在这里吞。
5. **`_SYNC_TOOL_EXECUTOR.submit(asyncio.run, coro(...))` 的双层包装**：传给 executor 的 callable 是 `asyncio.run`，参数是 `coro(...)`（coroutine 对象）。这种"传函数 + 参数"而不是"传 lambda"是为了 thread safety——lambda 闭包在多线程下偶尔会有反直觉的捕获行为。

### 3.4 `_ensure_sync_invocable_tool`：lazy 安装 sync wrapper

```python
# packages/harness/deerflow/tools/tools.py:37-41
def _ensure_sync_invocable_tool(tool: BaseTool) -> BaseTool:
    """Attach a sync wrapper to async-only tools used by sync agent callers."""
    if getattr(tool, "func", None) is None and getattr(tool, "coroutine", None) is not None:
        tool.func = make_sync_tool_wrapper(tool.coroutine, tool.name)
    return tool
```

**这段 5 行做的事**：

- LangChain 的 `BaseTool` 有两个调用路径：`tool.func`（sync）和 `tool.coroutine`（async）。
- 一个用 `@tool` 装饰的 async function 只填 `.coroutine`，`.func` 是 None。
- 这导致**同步调用**会失败（LangChain 内部用 `.func` 调）。
- 修复：把 sync wrapper 装到 `.func` 上——同步调时走 wrapper，wrapper 内部去跑 coroutine。
- **不影响 async 调用**——LangChain async 路径仍走 `.coroutine`。

**为什么是 lazy（每次装配 agent 都跑一遍）**？因为反射加载的工具对象在 module-level，可能跨 agent 实例共享。`tool.func = ...` 是幂等的——已经赋过的不会再赋（因为 `getattr(tool, "func") is not None`）。

### 3.5 4 类工具的合并 + 去重

```python
# packages/harness/deerflow/tools/tools.py:91-112
# Conditionally add tools based on config
builtin_tools = BUILTIN_TOOLS.copy()
skill_evolution_config = getattr(config, "skill_evolution", None)
if getattr(skill_evolution_config, "enabled", False):
    from deerflow.tools.skill_manage_tool import skill_manage_tool
    builtin_tools.append(skill_manage_tool)

# Add subagent tools only if enabled via runtime parameter
if subagent_enabled:
    builtin_tools.extend(SUBAGENT_TOOLS)
    logger.info("Including subagent tools (task)")

# If no model_name specified, use the first model (default)
if model_name is None and config.models:
    model_name = config.models[0].name

# Add view_image_tool only if the model supports vision
model_config = config.get_model_config(model_name) if model_name else None
if model_config is not None and model_config.supports_vision:
    builtin_tools.append(view_image_tool)
```

```python
# 第 5 段：MCP 工具（详见 11 篇）
mcp_tools = []
if include_mcp:
    try:
        from deerflow.config.extensions_config import ExtensionsConfig
        from deerflow.mcp.cache import get_cached_mcp_tools
        extensions_config = ExtensionsConfig.from_file()
        if extensions_config.get_enabled_mcp_servers():
            mcp_tools = get_cached_mcp_tools()
            # tool_search 分支（11 篇详谈）
            if config.tool_search.enabled:
                ...

# 第 6 段：ACP 工具
acp_tools: list[BaseTool] = []
try:
    from deerflow.tools.builtins.invoke_acp_agent_tool import build_invoke_acp_agent_tool
    acp_agents = (config.acp_agents or {}) if app_config else get_acp_agents()
    if acp_agents:
        acp_tools.append(build_invoke_acp_agent_tool(acp_agents))
```

```python
# packages/harness/deerflow/tools/tools.py:205-220
# Deduplicate by tool name — config-loaded tools take priority, followed by
# built-ins, MCP tools, and ACP tools.  Duplicate names cause the LLM to
# receive ambiguous or concatenated function schemas (issue #1803).
all_tools = loaded_tools + builtin_tools + mcp_tools + acp_tools
seen_names: set[str] = set()
unique_tools: list[BaseTool] = []
for t in all_tools:
    if t.name not in seen_names:
        unique_tools.append(t)
        seen_names.add(t.name)
    else:
        logger.warning(
            "Duplicate tool name %r detected and skipped — check your config.yaml and "
            "MCP server registrations (issue #1803).",
            t.name,
        )
return unique_tools
```

**3 个值得圈点的设计**：

1. **优先级顺序硬编码在拼接顺序里**：`loaded_tools + builtin_tools + mcp_tools + acp_tools`——靠 `seen_names` 的 first-wins 实现"config > builtin > MCP > ACP"。这个优先级不是配置项，是约定。
2. **`set + list` 而非 `dict.fromkeys`**：deer-flow 在 04 篇用 `dict.fromkeys` 做 artifact 去重保序，这里用更朴素的 set+list 也保序——两种实现等价，前者更紧凑。这里选 set+list 是因为还需要打 warning，循环里有别的逻辑。
3. **重复打 warning**：和 reflective name mismatch 一样的范式——不 fail，只 warn。**Agent 仍能跑（虽然少了一个工具）**，但用户在 logs 里能看到提示去修。

### 3.6 三种 builtin 条件添加

```python
# 第 4.1 段：skill_evolution
skill_evolution_config = getattr(config, "skill_evolution", None)
if getattr(skill_evolution_config, "enabled", False):
    from deerflow.tools.skill_manage_tool import skill_manage_tool
    builtin_tools.append(skill_manage_tool)
```

**特点**：用 `getattr(config, "skill_evolution", None)` 取代直接访问——是为了**兼容老 config**（没有这个字段就静默跳过）。

```python
# 第 4.2 段：subagent
if subagent_enabled:
    builtin_tools.extend(SUBAGENT_TOOLS)
    logger.info("Including subagent tools (task)")
```

**特点**：`subagent_enabled` 来自 **runtime config**（不是 yaml）。所以同一个 deer-flow 实例可以让一个 thread 启 subagent、另一个不启——靠 `config.configurable.subagent_enabled` 控制。

```python
# 第 4.3 段：vision
if model_name is None and config.models:
    model_name = config.models[0].name
model_config = config.get_model_config(model_name) if model_name else None
if model_config is not None and model_config.supports_vision:
    builtin_tools.append(view_image_tool)
```

**特点**：

- 没传 `model_name` 时 fallback 到第一个 model（默认 model）。
- vision 是**模型能力**而非用户选择——`supports_vision: true` 在 yaml 的 model 配置里写。
- 换 model 必须重新装 tools——这就是 05 篇讲 `DeerFlowClient._ensure_agent` 缓存 key 里有 `model_name` 的原因。

### 3.7 ACP 工具的"打包"模式

```python
# packages/harness/deerflow/tools/tools.py:186-201
acp_tools: list[BaseTool] = []
try:
    from deerflow.tools.builtins.invoke_acp_agent_tool import build_invoke_acp_agent_tool

    if app_config is None:
        from deerflow.config.acp_config import get_acp_agents
        acp_agents = get_acp_agents()
    else:
        acp_agents = getattr(config, "acp_agents", {}) or {}
    if acp_agents:
        acp_tools.append(build_invoke_acp_agent_tool(acp_agents))
        logger.info(f"Including invoke_acp_agent tool ({len(acp_agents)} agent(s): {list(acp_agents.keys())})")
except Exception as e:
    logger.warning(f"Failed to load ACP tool: {e}")
```

**注意**：哪怕配 5 个 ACP agent，最终也只**加一个工具** `invoke_acp_agent`——这个工具的参数里有个 `agent_name`，LLM 调用时传"我要调哪个 acp agent"。

**为什么这种打包模式？** 因为 ACP agents 的接口完全统一（都是"传消息 + 拿结果"），把它们包成一个工具 + 一个参数比"每 acp agent 一个工具"更简洁，schema 也更小。

**`get_acp_agents` 单例缓存兜底**：当 `app_config=None` 时（老调用方式），从模块全局拿；新调用方式直接从 `config.acp_agents`。03 篇讲过的"显式配置 vs 全局兼容"双路径。

---

## 4. 关键问题答疑（Key Questions）

### Q1：LLM 真正看到的工具名字是 `cfg.name` 还是 `tool.name`？

`tool.name`。LangChain 的 `bind_tools(tools=[...])` 用每个 tool 对象自己的 `.name` 属性生成 JSON schema 给 LLM。yaml 里的 `name:` 字段**只影响 deer-flow 内部的日志和 config 查询**（`get_tool_config(name)`），不影响 LLM 视角。

**改 yaml `name` 不会让 LLM 调度行为变化**——只是日志可读性。

### Q2：`get_available_tools` 是每次调 agent 都跑一次吗？

每次新建 agent 跑一次。看 `lead_agent/agent.py:432-433` 和 `DeerFlowClient._ensure_agent`——agent 实例创建时调一次，缓存到 agent 对象里。后续 invoke 不会再装配。

但**新 thread 第一次 invoke 会触发新 agent 创建**（如果 cache key 变化），所以"配置改了之后下一个 thread 立刻生效"——这是 03 篇 mtime 失效配合的效果。

### Q3：`include_mcp=False` 什么时候用？

主要场景：**单元测试 + subagent 装配**。

- 单元测试：mock 掉 MCP 部分，专心测 agent 逻辑。
- subagent（17 篇详谈）：subagent 默认拿父 agent 的工具子集，**MCP 工具 cache 在父级初始化好了**，subagent 直接复用——不需要重新拉 MCP。

### Q4：为什么 `_ensure_sync_invocable_tool` 不直接给所有 tool 装 sync wrapper？

因为已经有 `.func` 的工具（同步实现）不需要装——`tool.func is not None` 会跳过。只给"async-only"的工具补一层 sync。

如果给所有都装，sync wrapper 会包一层 async-to-sync——但 LangChain async 调用路径不会走 `.func`，所以"装了也没事"，只是浪费一层封装。deer-flow 选最小修改原则。

### Q5：`subagent_enabled=True` 但 config 没启 subagent 配置怎么办？

`get_available_tools` 仍然会加 `task_tool`——它不检查 `config.subagents`。但 `task_tool` 内部调度时会去 `config.subagents.enabled / max_concurrent_subagents`（16 篇详谈）——配置全空时会用默认值（max=3）跑。

**所以 `subagent_enabled` 这个 runtime flag 是"打开 tool"的开关，`config.subagents.*` 是"工具内部行为"的配置**——两者职责分开。

### Q6：`atexit.register(lambda: _SYNC_TOOL_EXECUTOR.shutdown(wait=False))` 为什么 `wait=False`？

进程退出时不等剩余任务——避免某个挂起的 sync wrapper 任务把 Python 退出 hold 住几分钟。Agent 系统的"未完成 tool call"在进程退出时本来就该丢——它们的状态会被 checkpointer 保留（21 篇），下次重启可以从 checkpoint 续跑。

如果 `wait=True`，进程退出时会等所有 worker 完成——这对 dev 模式的 reload 很不友好。

---

## 5. 横向延伸与面试级洞察（Interview-Grade Insights）

### 5.1 工具装配 = 反射 + 合并 + 去重 + 兼容层

把 deer-flow 的工具系统抽象一下：

```mermaid
flowchart LR
    A[配置/装饰器/外部进程] --> B[反射加载]
    B --> C[同步/异步统一]
    C --> D[条件添加]
    D --> E[合并去重]
    E --> F[最终 schema 给 LLM]
```

这 5 步对应"插件化系统"的通用范式。除了 deer-flow，pytest 的 fixture / FastAPI 的 router / Django 的 INSTALLED_APPS 都遵循类似模式：

- 加载来自多个源（plugin / decorator / config）。
- 兼容同步/异步两种调用上下文。
- 按条件启用/禁用。
- 同名冲突要有去重策略。

**`get_available_tools` 是个示范级的实现**——220 行涵盖了"插件系统"该有的全部组件。

### 5.2 first-wins 优先级 vs 显式优先级配置

deer-flow 选 **first-wins** 隐式优先级（list 拼接顺序硬编码）。另一种设计是**显式优先级**（每个 tool 带个 `priority: 1` 字段）。对比：

| 维度 | first-wins | 显式 priority |
|------|----------|--------------|
| **可预测性** | 高（顺序写死） | 低（需要查所有 priority） |
| **灵活性** | 低（改优先级要改代码） | 高（改 priority 即可） |
| **debug 友好** | 极好（代码顺序就是优先级） | 一般 |

deer-flow 选前者是合理的——因为"config > builtin > MCP > ACP" 这条顺序很少需要改。**首位需求是稳定可预测**，不是"用户可调"。

### 5.3 `make_sync_tool_wrapper` 是 async/sync 桥的范式

`async def` 在 Python agent 框架里到处都是，但 client 代码（DeerFlowClient.chat / 单元测试 / Jupyter notebook）经常是同步的。`make_sync_tool_wrapper` 那 30 行解决一个普遍问题。

替代方案：

| 方案 | 缺点 |
|------|------|
| `asyncio.run(coro)` 直接调 | event loop 已经在跑时炸 |
| `loop.run_until_complete` | 同上 |
| `nest_asyncio.apply()` | 引入额外依赖，不优雅 |
| `concurrent.futures.ThreadPoolExecutor.submit(asyncio.run, ...)` | **deer-flow 选的**——共享线程池 + 跨 loop 安全 |

**面试金句**：在已有 event loop 的 process 里跑 sync caller 调 async function，唯一健壮的解法是把 coroutine 抛到独立线程跑 `asyncio.run`——这样新线程会建一个全新 loop，不和外层 loop 冲突。deer-flow 用共享 ThreadPoolExecutor 实现这一点，是这类问题的范式解。

### 5.4 vs LangChain 原生 `ChatModel.bind_tools(tools=[...])`

LangChain 让你手动传 tools list。deer-flow 加了 `get_available_tools` 这层之后：

- **集中点**：所有 agent 装配工具都过这一个函数——单点优化。
- **声明式配置**：yaml 改一行，工具集变化。
- **去重 + warning**：避免重复 schema。

代价是**多了一层间接**——单元测试时要 mock `get_available_tools`，或者直接 patch `BUILTIN_TOOLS` / `SUBAGENT_TOOLS` 这些 module-level list。这是 deer-flow `tests/test_*` 里多次出现的 mock 模式。

---

## 6. 实操教程（Hands-on Lab）

### 6.1 最小可运行示例：观察工具装配的真实输出

```python
# backend/debug_tools.py
"""列出当前 agent 实际会拿到的工具集，按来源分类"""
from deerflow.tools import get_available_tools
from deerflow.config import get_app_config


# 1. 默认装配（subagent / vision 都关）
print("=== 默认装配（subagent_enabled=False, model=default）===")
tools = get_available_tools(subagent_enabled=False)
for t in tools:
    print(f"  - {t.name:30s} (type={type(t).__name__})")

print(f"\n  Total: {len(tools)}")

# 2. 开 subagent
print("\n=== subagent_enabled=True ===")
tools2 = get_available_tools(subagent_enabled=True)
added = set(t.name for t in tools2) - set(t.name for t in tools)
print(f"  Added: {added}")    # 应该有 'task'

# 3. 限 group
print("\n=== groups=['web'] ===")
tools3 = get_available_tools(subagent_enabled=False, groups=["web"])
for t in tools3:
    print(f"  - {t.name}")
# 应该只看到 web group 下的工具（web_search / web_fetch 等）+ builtin
```

跑：`cd backend && PYTHONPATH=. uv run python debug_tools.py`

### 6.2 Debug 任务清单

#### 实验 ①：故意制造 name mismatch 看 warning

改 `config.yaml`：

```yaml
tools:
  - name: my_search_alias   # 故意不匹配真实 tool.name
    group: web
    use: deerflow.community.ddg_search.tools:web_search_tool
```

重启 Gateway 或调 `get_available_tools()`。看 logs 里：

```
WARNING Tool name mismatch: config name 'my_search_alias' does not match tool .name 'web_search'
```

**能学到**：name 不是 yaml 决定的，是 tool 对象自己的。

#### 实验 ②：故意装两个同名 MCP 工具

启用两个 MCP server 都暴露 `web_search` 工具。看 logs：

```
WARNING Duplicate tool name 'web_search' detected and skipped
```

后注册的那个被丢——LLM 只看到第一个。

#### 实验 ③：观察 sync wrapper 的"上下文识别"

```python
# 在 IPython 里
from deerflow.tools.sync import make_sync_tool_wrapper

async def my_async_tool(x):
    return x * 2

wrapped = make_sync_tool_wrapper(my_async_tool, "my_async_tool")

# 情况 1：没在 event loop 里 → 直接 asyncio.run
print(wrapped(5))   # 10

# 情况 2：在 event loop 里 → 走 ThreadPoolExecutor
import asyncio
async def main():
    return wrapped(7)   # 在 loop 里调 wrapper
print(asyncio.run(main()))   # 14
```

**能学到**：同一个 wrapper 在两种调用上下文下都能正常工作。

#### 实验 ④：观察 vision 工具按 model 能力切换

把 `config.yaml` 默认 model 改成 `supports_vision: false` 的（例如 deepseek-r1 一些版本不支持 vision），观察 `view_image_tool` 是否从 tools list 消失。再改回支持 vision 的（如 doubao-seed-1.8 或 gpt-4o），观察它回来。

**能学到**：vision 是模型驱动的——同一份 yaml，不同 model 拿到不同工具集。

---

## 7. 与下一模块的衔接

读完本章你应该能：

- 默写 `get_available_tools` 的 4 类来源 + 各自加载方式。
- 解释为什么 `tool.name` 才是 LLM 看到的真名、yaml 的 `name` 只是别名。
- 用 30 行的 `make_sync_tool_wrapper` 描述 async-to-sync 桥的"两种上下文判定"。
- 知道 4 类工具的去重优先级是 `config > builtin > MCP > ACP`。

但你还**没看到** MCP 工具是怎么"从外部进程取来"的——MCP 协议的 transport、OAuth、cache 失效、tool_search 怎么让 MCP 工具变"按需发现"。这些是 **11 篇（MCP 集成与 Tool Search）** 的核心。

---

📌 **本章已交付**。请你检查后告诉我：
- 哪几段读起来不顺？
- 是否要补"`build_invoke_acp_agent_tool` 的内部实现（如何把多个 ACP agent 打包成一个 tool）"？
- 还是直接进入 11 篇？

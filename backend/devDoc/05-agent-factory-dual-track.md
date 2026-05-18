# 05 · Agent 工厂双轨：make_lead_agent vs create_deerflow_agent

> 04 篇讲清楚了 agent 的"状态"长什么样。这一章回答：**这个状态背后那个 `CompiledStateGraph` 实例（也就是真正的 agent）是怎么被构造出来的？为什么 deer-flow 在 `agents/` 下面摆了两个工厂函数？**
>
> 不理解双轨的存在动机，你会很容易把这两个函数当成"新旧版本"——其实它们是面向不同消费者、共享同一套中间件装配规则的**对偶设计**。

---

## 1. 模块定位（Why this matters）

deer-flow 的 agent 创建有两个入口：

| 函数 | 签名 | 谁来调 | 配置来源 |
|------|------|--------|---------|
| **`make_lead_agent(config)`** | `(RunnableConfig) -> CompiledStateGraph` | LangGraph Server / Studio / Gateway 内嵌运行时 | 全局 `AppConfig` + `config.configurable` |
| **`create_deerflow_agent(model, tools, ..., features=...)`** | `(model, tools, ..., RuntimeFeatures) -> CompiledStateGraph` | 直接用 SDK 的第三方 / 单元测试 / `DeerFlowClient` | 调用者用 Python 显式传 |

**核心差别**：

- **`make_lead_agent`** 是"应用工厂"：它信任全局配置，自己解析 model / tools / middlewares / skills / agent_name；签名是 LangGraph CLI 强制要求的 `(config: RunnableConfig)`。
- **`create_deerflow_agent`** 是"SDK 工厂"：它**完全不读 yaml**，所有依赖必须由调用者明确传入；签名是面向程序员的，配 `RuntimeFeatures` 这种数据类做声明式开关。

不读这章会错过 4 个关键认知：

1. **`make_lead_agent` 是"集成层"，`create_deerflow_agent` 是"SDK 层"**：前者会把"读 config.yaml → 解析 model_name → 加载 skills → 装配中间件"全做完；后者只做最后一步（装配中间件 + 调 `create_agent`）。中间所有决策都交给上游。
2. **二者共享中间件装配，但不共享"决策代码"**：两边都把中间件顺序写死成一样（见 04 篇的字段所有权矩阵），但**决策由谁做的代码是分开的**——`make_lead_agent` 决策代码在 `agent.py:240` 的 `_build_middlewares`，`create_deerflow_agent` 决策代码在 `factory.py:155` 的 `_assemble_from_features`。这是个有意为之的复制粘贴——保证 SDK 不读 yaml。
3. **`RuntimeFeatures` 是 deer-flow 的"声明式特性开关"**：`True` 用内置默认、`False` 关闭、`AgentMiddleware 实例` 用自定义实现。一个 dataclass 把 8 个 feature flag 收编成统一接口。
4. **`@Next(Anchor)` / `@Prev(Anchor)` 是中间件链的"插入坐标"**：第三方写中间件时，用装饰器声明"我要放在 `DanglingToolCallMiddleware` 后面"，`create_deerflow_agent` 会按这个坐标插。比直接给 list index 健壮得多。

对应到 Harness 六要素：本章打下的是 **"工具集成 + 可扩展性"** 的基础——可被第三方 import 的 SDK + 可被自定义中间件扩展，是 deer-flow "可发布 harness" 这个定位的工程兑现。

---

## 2. 源码地图（Source Map）

### 2.1 关键文件清单

| 路径 | 角色 |
|------|------|
| [`packages/harness/deerflow/agents/lead_agent/agent.py`](../packages/harness/deerflow/agents/lead_agent/agent.py) | 应用工厂 `make_lead_agent` + 决策代码 `_build_middlewares` |
| [`packages/harness/deerflow/agents/factory.py`](../packages/harness/deerflow/agents/factory.py) | SDK 工厂 `create_deerflow_agent` + 装配代码 `_assemble_from_features` |
| [`packages/harness/deerflow/agents/features.py`](../packages/harness/deerflow/agents/features.py) | `RuntimeFeatures` dataclass + `Next/Prev` 装饰器（64 行） |
| [`packages/harness/deerflow/agents/__init__.py`](../packages/harness/deerflow/agents/__init__.py) | 两条入口的 `__all__` 导出 + skills 缓存预热 |
| [`packages/harness/deerflow/client.py`](../packages/harness/deerflow/client.py) | 实际调用 `_build_middlewares` 的"SDK 实现"——证明 client 走的是 make_lead_agent 风格而非 create_deerflow_agent |
| [`backend/tests/test_create_deerflow_agent.py`](../tests/test_create_deerflow_agent.py) | 38 个测试用例，覆盖 features / extra_middleware / Next/Prev 全部规则 |
| [`backend/tests/test_dangling_tool_call_middleware.py`](../tests/test_dangling_tool_call_middleware.py) | 验证 always-on 中间件存在性的测试 |
| [`langgraph.json`](../langgraph.json) | `"lead_agent": "deerflow.agents:make_lead_agent"` 把 LangGraph CLI 钉到 `make_lead_agent` 上 |

### 2.2 关键符号速查表

| 符号 | 文件:行 | 一句话职责 |
|------|---------|-----------|
| `make_lead_agent(config)` | `lead_agent/agent.py:343` | LangGraph 兼容入口；分发到 `_make_lead_agent` |
| `_make_lead_agent(config, app_config)` | `lead_agent/agent.py:350` | 解析 model / agent_name / tools / skills / middlewares → `create_agent(...)` |
| `_build_middlewares(config, ...)` | `lead_agent/agent.py:240` | 应用工厂的中间件装配（动态决策） |
| `_get_runtime_config(config)` | `lead_agent/agent.py:29` | 合并 `configurable + context` 成单一 dict |
| `_resolve_model_name(...)` | `lead_agent/agent.py:38` | 请求 → agent_config → 全局默认，找不到 fallback |
| `_create_summarization_middleware(...)` | `lead_agent/agent.py:53` | 把 SummarizationConfig 转成 `DeerFlowSummarizationMiddleware` 实例 |
| `_available_skill_names(...)` | `lead_agent/agent.py:321` | 计算这次 run 可见的 skill 名集合 |
| `create_deerflow_agent(model, ...)` | `agents/factory.py:61` | SDK 工厂入口；三种 mode（minimal / features / full takeover） |
| `_assemble_from_features(feat, ...)` | `agents/factory.py:155` | SDK 工厂的中间件装配（声明式） |
| `_insert_extra(chain, extras)` | `agents/factory.py:306` | `@Next/@Prev` 解析算法 |
| `class RuntimeFeatures` | `agents/features.py:14` | 8 个 feature flag |
| `Next(anchor)` | `agents/features.py:42` | 装饰器：把 `cls._next_anchor = anchor` |
| `Prev(anchor)` | `agents/features.py:54` | 装饰器：把 `cls._prev_anchor = anchor` |

### 2.3 双轨对照图

```mermaid
flowchart TB
    subgraph Track1["Track 1: 应用工厂（make_lead_agent）"]
        Studio[LangGraph CLI / Studio<br/>导入 deerflow.agents:make_lead_agent]
        Gateway[Gateway run_agent worker<br/>调 agent_factory(config=...)]
        ML[make_lead_agent<br/>config:RunnableConfig]
        AppConfig[(全局 AppConfig<br/>get_app_config)]
        AC[load_agent_config<br/>agent_name → SOUL.md + config.yaml]
        Sk[get_skills_for_config<br/>+ filter_tools_by_skill_allowed_tools]
        Skb1[_build_middlewares<br/>含 14-18 节装配]
        CA1[langchain.create_agent]

        Studio --> ML
        Gateway --> ML
        ML --> AppConfig
        ML --> AC
        ML --> Sk
        ML --> Skb1
        Skb1 --> CA1
    end

    subgraph Track2["Track 2: SDK 工厂（create_deerflow_agent）"]
        ThirdParty[第三方 / 单元测试<br/>显式传 model/tools/features]
        CDA[create_deerflow_agent<br/>纯参数]
        Mode{mode}
        Mode1[middleware=takeover<br/>整套覆盖]
        Mode2[features + extra_middleware<br/>声明式开关 + @Next/@Prev]
        AS[_assemble_from_features]
        IE[_insert_extra @Next/@Prev]
        CA2[langchain.create_agent]

        ThirdParty --> CDA
        CDA --> Mode
        Mode -- middleware= --> Mode1
        Mode -- features= --> Mode2
        Mode2 --> AS
        Mode2 --> IE
        Mode1 --> CA2
        AS --> CA2
        IE --> CA2
    end

    Skb1 ~~~ Note1[共享中间件顺序契约<br/>但代码独立]
    AS ~~~ Note1
```

---

## 3. 核心逻辑精读（Deep Dive）

### 3.1 `make_lead_agent`：应用工厂的最小可读版本

```python
# packages/harness/deerflow/agents/lead_agent/agent.py:343-348
def make_lead_agent(config: RunnableConfig):
    """LangGraph graph factory; keep the signature compatible with LangGraph Server."""
    runtime_config = _get_runtime_config(config)
    runtime_app_config = runtime_config.get("app_config")
    return _make_lead_agent(config, app_config=runtime_app_config or get_app_config())
```

**3 个看似简单实则重要的设计**：

1. **签名 `(config: RunnableConfig)`** 是 **LangGraph CLI 的硬性合同**。`langgraph.json` 里 `graphs.lead_agent: "deerflow.agents:make_lead_agent"` 让 LangGraph CLI 在启动时**按这个签名导入并调用**。改签名就破坏兼容性。
2. **`runtime_config.get("app_config")` 比 `get_app_config()` 优先**：这是给 03 篇讲的 `push_current_app_config` 留的后门——`run_agent` worker 可以在 `configurable["app_config"]` 里塞一份 per-request AppConfig，让本次请求用不同配置。
3. **`make_lead_agent` 本身极简，所有重活在 `_make_lead_agent`**：分两层是为了把"LangGraph 合同层"和"决策实现层"解耦，未来要换合同（例如 LangGraph 2.x）只动外层。

### 3.2 `_make_lead_agent`：决策的细节

```python
# packages/harness/deerflow/agents/lead_agent/agent.py:350-393 (节选)
def _make_lead_agent(config: RunnableConfig, *, app_config: AppConfig):
    # Lazy import to avoid circular dependency
    from deerflow.tools import get_available_tools
    from deerflow.tools.builtins import setup_agent, update_agent

    cfg = _get_runtime_config(config)
    resolved_app_config = app_config

    thinking_enabled = cfg.get("thinking_enabled", True)
    reasoning_effort = cfg.get("reasoning_effort", None)
    requested_model_name: str | None = cfg.get("model_name") or cfg.get("model")
    is_plan_mode = cfg.get("is_plan_mode", False)
    subagent_enabled = cfg.get("subagent_enabled", False)
    max_concurrent_subagents = cfg.get("max_concurrent_subagents", 3)
    is_bootstrap = cfg.get("is_bootstrap", False)
    agent_name = validate_agent_name(cfg.get("agent_name"))

    agent_config = load_agent_config(agent_name) if not is_bootstrap else None
    available_skills = _available_skill_names(agent_config, is_bootstrap)
    # 模型名优先级：请求 → 自定义 agent 配置 → 全局默认
    agent_model_name = agent_config.model if agent_config and agent_config.model else None
    model_name = _resolve_model_name(requested_model_name or agent_model_name,
                                     app_config=resolved_app_config)
    model_config = resolved_app_config.get_model_config(model_name)

    if model_config is None:
        raise ValueError("No chat model could be resolved...")
    if thinking_enabled and not model_config.supports_thinking:
        logger.warning(f"Thinking mode is enabled but model '{model_name}' does not support it; "
                       f"fallback to non-thinking mode.")
        thinking_enabled = False
    # ... metadata 注入 + skills 解析略 ...
```

**6 个值得圈点的细节**：

1. **顶部 `from deerflow.tools import get_available_tools`**：注释明说"Lazy import to avoid circular dependency"。这是 02 篇讲的"内部循环规避"现场。`deerflow.tools.builtins.task_tool` 间接依赖 `deerflow.agents`，模块顶层 import 会循环。
2. **`cfg.get("model_name") or cfg.get("model")`**：两个 key 都接受，兼容老调用方式。这种"双 key 接收"在 deer-flow 出现多次（例如 `is_plan_mode` 也有别名）。
3. **优先级链 `请求 → agent_config.model → 全局默认`**：自定义 agent 可以钉死模型（例如 "this researcher agent only uses gpt-4"），但用户在请求里显式传 model_name 仍可覆盖——这是"软推荐 + 硬覆盖"的常用 pattern。
4. **`thinking_enabled and not model_config.supports_thinking` 自动降级**：用户传了 thinking_enabled=True 但 model 不支持，**不报错**——降级 + warning。这是 deer-flow 对前端的"容错"——前端 UI 可能不知道当下选的 model 支不支持 thinking。
5. **`is_bootstrap` 分支**：这是一个特殊模式——用户在创建新的"custom agent"时第一次访问，agent_config 还不存在。这时挂载 `setup_agent` 工具让 LLM 自己写 `SOUL.md`。
6. **metadata 注入**（行 395-409）：所有决策结果（agent_name / model_name / thinking_enabled / ...）写进 `config["metadata"]`，给 LangSmith 等 trace 工具看。**这是分布式追踪友好的关键——run trace 里能直接看到这次 run 用了什么决策**。

### 3.3 `create_deerflow_agent`：SDK 工厂的 3 种模式

```python
# packages/harness/deerflow/agents/factory.py:61-147
def create_deerflow_agent(
    model: BaseChatModel,
    tools: list[BaseTool] | None = None,
    *,
    system_prompt: str | None = None,
    middleware: list[AgentMiddleware] | None = None,
    features: RuntimeFeatures | None = None,
    extra_middleware: list[AgentMiddleware] | None = None,
    plan_mode: bool = False,
    state_schema: type | None = None,
    checkpointer: BaseCheckpointSaver | None = None,
    name: str = "default",
) -> CompiledStateGraph:
    # 互斥检查
    if middleware is not None and features is not None:
        raise ValueError("Cannot specify both 'middleware' and 'features'.  Use one or the other.")
    if middleware is not None and extra_middleware:
        raise ValueError("Cannot use 'extra_middleware' with 'middleware' (full takeover).")
    if extra_middleware:
        for mw in extra_middleware:
            if not isinstance(mw, AgentMiddleware):
                raise TypeError(...)

    effective_tools: list[BaseTool] = list(tools or [])
    effective_state = state_schema or ThreadState

    if middleware is not None:
        # —— Mode 1: Full takeover —— 用户直接给中间件 list
        effective_middleware = list(middleware)
    else:
        # —— Mode 2/3: features-driven assembly ——
        feat = features or RuntimeFeatures()
        effective_middleware, extra_tools = _assemble_from_features(
            feat,
            name=name,
            plan_mode=plan_mode,
            extra_middleware=extra_middleware or [],
        )
        # Tool 去重：用户传的 tool 名优先
        existing_names = {t.name for t in effective_tools}
        for t in extra_tools:
            if t.name not in existing_names:
                effective_tools.append(t)
                existing_names.add(t.name)

    return create_agent(
        model=model,
        tools=effective_tools or None,
        middleware=effective_middleware,
        system_prompt=system_prompt,
        state_schema=effective_state,
        checkpointer=checkpointer,
        name=name,
    )
```

**3 种模式的本质**：

| Mode | 触发条件 | 行为 |
|------|---------|------|
| **Minimal** | `features=None, middleware=None, extra_middleware=None` | 用默认 `RuntimeFeatures()`——多数 feature 为 False，最小可用 agent |
| **Features-driven**（推荐） | `features=RuntimeFeatures(...)` 或 `extra_middleware=[...]` | 按声明式开关装配 + 第三方中间件按 `@Next/@Prev` 插入 |
| **Full Takeover** | `middleware=[mw1, mw2, ...]` | 用户完全负责中间件 list，**features / extra_middleware 不能同时用** |

**严格互斥校验（行 110-117）**：这是健壮的 API 设计——同时传 `middleware` 和 `features` 时立刻 raise，比"静默忽略 features"对调用者友好得多。

**Tool 去重的 first-wins**（行 132-137）：用户显式传的 tool 优先，feature 自动注入的（例如 `ask_clarification`）让位。这避免 issue #1803 那种"同名工具 LLM 看到两份 schema"的混乱。

### 3.4 `RuntimeFeatures`：8 个声明式开关

```python
# packages/harness/deerflow/agents/features.py:14-34
@dataclass
class RuntimeFeatures:
    """Declarative feature flags for ``create_deerflow_agent``.

    Most features accept:
    - ``True``: use the built-in default middleware
    - ``False``: disable
    - An ``AgentMiddleware`` instance: use this custom implementation instead

    ``summarization`` and ``guardrail`` have no built-in default — they only
    accept ``False`` (disable) or an ``AgentMiddleware`` instance (custom).
    """

    sandbox: bool | AgentMiddleware = True
    memory: bool | AgentMiddleware = False
    summarization: Literal[False] | AgentMiddleware = False
    subagent: bool | AgentMiddleware = False
    vision: bool | AgentMiddleware = False
    auto_title: bool | AgentMiddleware = False
    guardrail: Literal[False] | AgentMiddleware = False
    loop_detection: bool | AgentMiddleware = True
```

**3 个值得圈点的设计**：

1. **类型 `bool | AgentMiddleware`**：用 Python 的 union type 把"开关"和"自定义实现"塞进一个字段。`True` = 用内置、`False` = 关、`MyCustomMiddleware()` = 用自定义。比传统 "enable_xxx: bool + xxx_middleware: Middleware" 两字段更简洁。
2. **`summarization` 和 `guardrail` 的类型是 `Literal[False] | AgentMiddleware`**：**显式禁止 True**。原因是这两个中间件需要"必填的运行时参数"（summarization 要 model，guardrail 要 provider 实例）——没有这些参数就没法实例化。让用户在 `features=RuntimeFeatures(summarization=True)` 时**编译期报类型错**，比运行期报错更友好。
3. **默认值的选择**：`sandbox=True` 和 `loop_detection=True`——这两个是"绝大多数场景该开"的。其他默认 False，遵循 "minimal by default" 原则。

### 3.5 `@Next` / `@Prev`：中间件位置坐标

```python
# packages/harness/deerflow/agents/features.py:42-63
def Next(anchor: type[AgentMiddleware]):
    """Declare this middleware should be placed after *anchor* in the chain."""
    if not (isinstance(anchor, type) and issubclass(anchor, AgentMiddleware)):
        raise TypeError(f"@Next expects an AgentMiddleware subclass, got {anchor!r}")

    def decorator(cls: type[AgentMiddleware]) -> type[AgentMiddleware]:
        cls._next_anchor = anchor  # type: ignore[attr-defined]
        return cls

    return decorator


def Prev(anchor: type[AgentMiddleware]):
    """Declare this middleware should be placed before *anchor* in the chain."""
    if not (isinstance(anchor, type) and issubclass(anchor, AgentMiddleware)):
        raise TypeError(f"@Prev expects an AgentMiddleware subclass, got {anchor!r}")

    def decorator(cls: type[AgentMiddleware]) -> type[AgentMiddleware]:
        cls._prev_anchor = anchor  # type: ignore[attr-defined]
        return cls

    return decorator
```

**使用示例**（来自 `tests/test_create_deerflow_agent.py:543-555`）：

```python
@Next(DanglingToolCallMiddleware)
class First(AgentMiddleware):
    pass

@Next(First)
class Second(AgentMiddleware):
    pass

create_deerflow_agent(
    _make_mock_model(),
    features=RuntimeFeatures(sandbox=False),
    extra_middleware=[Second(), First()],  # 故意倒着传
)

# 结果：DanglingToolCall → First → Second
# 即使传入顺序是 [Second, First]，@Next 链也能正确解析
```

**4 个工程亮点**：

1. **装饰器只是把 `_next_anchor` / `_prev_anchor` 挂在类上**——没有副作用，没有 import 时执行。这让"中间件类的注册"和"中间件链的装配"完全分离。
2. **运行时类型检查（行 44 / 56）**：`@Next("DanglingTool")`（字符串）会立刻报错，不会等到装配时才发现。这是"早失败"的好范式。
3. **支持"跨外部锚定"**：上面例子里 `Second` 锚定的是 `First`（外部传入的中间件），不是 built-in 中间件。`_insert_extra` 的迭代算法（看 factory.py:353-379）能处理这种依赖。
4. **`@Next(ClarificationMiddleware)` 的尾不变性兜底**（factory.py:294-296）：即使你用 `@Next(ClarificationMiddleware)` 把自己插到 Clarification 后面（理论上会让 Clarification 不再是末尾），factory 会在最后再把 Clarification 重新移到末尾。这是 deer-flow 整个中间件链最重要的不变量——"Clarification 必须最后"——的物理保证。

### 3.6 `_insert_extra` 的迭代解析算法

```python
# packages/harness/deerflow/agents/factory.py:306-379 节选
def _insert_extra(chain: list[AgentMiddleware], extras: list[AgentMiddleware]) -> None:
    """Insert extra middlewares into *chain* using ``@Next``/``@Prev`` anchors.

    Algorithm:
      1. Validate: no middleware has both @Next and @Prev.
      2. Conflict detection: two extras targeting same anchor → error.
      3. Insert unanchored extras before ClarificationMiddleware.
      4. Insert anchored extras iteratively (supports cross-external anchoring).
      5. If an anchor cannot be resolved after all rounds → error.
    """
    # ... 收集 anchored / unanchored / 检测冲突 ...

    # Unanchored → before ClarificationMiddleware
    clarification_idx = next(i for i, m in enumerate(chain) if isinstance(m, ClarificationMiddleware))
    for mw in unanchored:
        chain.insert(clarification_idx, mw)
        clarification_idx += 1

    # Anchored → iterative insertion (supports external-to-external anchoring)
    pending = list(anchored)
    max_rounds = len(pending) + 1
    for _ in range(max_rounds):
        if not pending:
            break
        remaining = []
        for mw, direction, anchor in pending:
            idx = next((i for i, m in enumerate(chain) if isinstance(m, anchor)), None)
            if idx is None:
                remaining.append((mw, direction, anchor))
                continue
            if direction == "next":
                chain.insert(idx + 1, mw)
            else:
                chain.insert(idx, mw)
        if len(remaining) == len(pending):
            names = [type(m).__name__ for m, _, _ in remaining]
            anchor_types = {a for _, _, a in remaining}
            remaining_types = {type(m) for m, _, _ in remaining}
            circular = anchor_types & remaining_types
            if circular:
                raise ValueError(f"Circular dependency among extra middlewares: ...")
            raise ValueError(f"Cannot resolve positions for {', '.join(names)} ...")
        pending = remaining
```

**算法本质**：

- 先按"不需要任何依赖"的 unanchored 优先插入。
- anchored 用**迭代收敛**：每一轮把能解析的插入，剩下的（锚还没被插入）放下一轮。如果某一轮一个都不能解析（`len(remaining) == len(pending)`），说明出现了"环依赖"或"指向不存在的锚"，直接 raise。
- `max_rounds = len(pending) + 1` 是收敛上界——最坏情况下，每一轮只插入一个，需要 `len(pending)` 轮。

**这是一个简化的拓扑排序**：依赖能解析就插入，解析不动就报错。比真正的拓扑排序简单（因为不需要先建依赖图），但同样能保证正确性。

**对比常见替代方案**：

| 方案 | 优点 | 缺点 |
|------|------|------|
| **A. 给每个中间件分配一个数字优先级** | 简单 | 第三方很难选数字（怎么知道空隙在哪？） |
| **B. 字符串 anchor**（`@Next("DanglingTool")`） | 简单 | 拼写错误 / 重命名后无法静态检测 |
| **C. 类引用 anchor**（deer-flow） | IDE 跳转 / 重命名自动同步 | 必须 import 那个类 |

C 在 Python 里几乎没有缺点——`from .dangling_tool_call_middleware import DanglingToolCallMiddleware` 是合法的、IDE 支持的。

---

## 4. 关键问题答疑（Key Questions）

### Q1：为什么 `DeerFlowClient`（22 篇详谈）用的不是 `create_deerflow_agent` 而是 `_build_middlewares`？

看 `client.py:230-241`：

```python
kwargs = {
    "model": create_chat_model(name=model_name, thinking_enabled=thinking_enabled),
    "tools": self._get_tools(model_name=model_name, subagent_enabled=subagent_enabled),
    "middleware": _build_middlewares(config, model_name=model_name, agent_name=self._agent_name,
                                      custom_middlewares=self._middlewares),
    "system_prompt": apply_prompt_template(...),
    "state_schema": ThreadState,
}
self._agent = create_agent(**kwargs)
```

`DeerFlowClient` 是个**便利层**，它本质上是"嵌入式 Gateway"——所以走的是和 `make_lead_agent` 一样的决策代码（`_build_middlewares` + `apply_prompt_template`），但跳过了 `make_lead_agent` 那一层"从 RunnableConfig 解析"。

**`create_deerflow_agent` 留给真正的第三方**——比如想把 deer-flow harness 嵌进自己项目，但不想要 deer-flow 的配置约定（不想读 config.yaml）。

### Q2：如果我想从 `make_lead_agent` 那条路加一个自定义中间件怎么办？

看 `_build_middlewares` 签名（行 240-247）有个参数：

```python
def _build_middlewares(
    config: RunnableConfig,
    model_name: str | None,
    agent_name: str | None = None,
    custom_middlewares: list[AgentMiddleware] | None = None,  # ← 这个
    *,
    app_config: AppConfig | None = None,
):
    ...
    # Inject custom middlewares before ClarificationMiddleware
    if custom_middlewares:
        middlewares.extend(custom_middlewares)
    # ClarificationMiddleware should always be last
    middlewares.append(ClarificationMiddleware())
```

但**目前 `make_lead_agent` 没把 `custom_middlewares` 暴露给上游**——它只在 `DeerFlowClient.__init__` 时被传进来。如果你要给 Gateway 模式加，得改 `_make_lead_agent`，从 `config.configurable` 里读 `custom_middlewares` 然后传下去。这是个**待扩展**的口子。

### Q3：`features=RuntimeFeatures(sandbox=False)` 时 agent 怎么运作？

看 `_assemble_from_features`（factory.py:193-203）：

```python
# --- [0-2] Sandbox infrastructure ---
if feat.sandbox is not False:
    if isinstance(feat.sandbox, AgentMiddleware):
        chain.append(feat.sandbox)
    else:
        from deerflow.agents.middlewares.thread_data_middleware import ThreadDataMiddleware
        from deerflow.agents.middlewares.uploads_middleware import UploadsMiddleware
        from deerflow.sandbox.middleware import SandboxMiddleware

        chain.append(ThreadDataMiddleware(lazy_init=True))
        chain.append(UploadsMiddleware())
        chain.append(SandboxMiddleware(lazy_init=True))
```

`feat.sandbox is not False` 这个条件：当 `sandbox=False` 时，**3 个中间件全部不挂**——agent 没有 ThreadData、没有 Uploads、没有 Sandbox。后续工具调用如果尝试读 `state["sandbox"]` 会拿到 `None`。

适合什么场景？纯文本对话 agent，不需要任何文件 / 沙箱能力。例如做一个"翻译 bot"。

### Q4：`@Next(X) + @Prev(X)` 同时锚同一个 X，会怎样？

会 `ValueError`。看 factory.py:332-333：

```python
if next_anchor in prev_targets:
    raise ValueError(f"Conflict: {type(mw).__name__} @Next({next_anchor.__name__}) and "
                     f"{prev_targets[next_anchor].__name__} @Prev({next_anchor.__name__}) "
                     "— use cross-anchoring between extras instead")
```

错误信息直接给了 workaround——"你应该让一个锚另一个，而不是都锚同一个 built-in"。

### Q5：如果 `extra_middleware` 里有一个 `@Next(NonExistentMiddleware)`，会怎样？

跑到 factory.py:378 那个 "Cannot resolve" 错误。**而且能区分两种情况**：

- 如果该锚点指向的类**就在 extras 里**但没被插入 → "Circular dependency among extra middlewares: X"
- 如果锚点压根不存在 → "Cannot resolve positions for X — anchors Y not found in chain"

测试用例 `test_extra_unresolvable_anchor`（行 570-585）和 `test_circular_dependency` 都覆盖了。

### Q6：为什么 `summarization=True` 和 `guardrail=True` 会报错？

看 factory.py:217-223：

```python
# --- [6] Summarization ---
if feat.summarization is not False:
    if isinstance(feat.summarization, AgentMiddleware):
        chain.append(feat.summarization)
    else:
        raise ValueError("summarization=True requires a custom AgentMiddleware instance "
                         "(SummarizationMiddleware needs a model argument)")
```

`DeerFlowSummarizationMiddleware` 的构造函数需要 `model`、`trigger`、`keep` 这些参数。`RuntimeFeatures` 是声明式开关，传不进构造参数。所以**约定**：`summarization` 要么不开（`False`），要么用户自己 `DeerFlowSummarizationMiddleware(model=..., trigger=...)` 实例化后传 `summarization=instance`。

`Literal[False] | AgentMiddleware` 这个类型注解让 mypy 在 `summarization=True` 时直接报类型错——比运行时 raise 更早。

---

## 5. 横向延伸与面试级洞察（Interview-Grade Insights）

### 5.1 双轨设计的本质：分离"集成约定"和"SDK 契约"

很多框架的入口是单一的：

- **AutoGen**：`autogen.ConversableAgent(...)` 单入口，配置混在构造参数里。
- **CrewAI**：`Crew(agents=[...], tasks=[...])` 单入口。
- **LangGraph 官方**：`StateGraph(State).add_node(...).add_edge(...).compile()` 单 builder 模式。

deer-flow 选**双入口**是因为它有"两类消费者"：

| 消费者 | 期望 |
|--------|------|
| **应用消费者**（LangGraph CLI、Gateway 嵌入运行时） | 读全局配置，零侵入 |
| **SDK 消费者**（第三方嵌入、单元测试） | 显式传参，无副作用 |

**单入口在两类消费者间会拉扯**：要么 SDK 用户被迫准备 config.yaml，要么应用消费者得手写 50 个参数。deer-flow 的双入口把两者的需求各自满足。

**面试金句**：deer-flow 把 agent 工厂拆成"应用工厂"和"SDK 工厂"两条轨，分别面向 LangGraph CLI 兼容和第三方嵌入两类消费者，共享中间件顺序契约但不共享决策代码——这是把"框架可发布性"和"应用便利性"分开的关键设计。

### 5.2 装饰器 anchor 是 "类引用 + 元数据" 的 Python 范式

`@Next(DanglingToolCallMiddleware)` 把锚点信息直接挂在类对象上（`cls._next_anchor = anchor`）。这是 Python "类是一等公民"特性的典型应用：

- **类引用 = IDE / mypy 友好**：拼写错、重命名时立刻发现。
- **元数据写在类上 = 装配阶段一次读取**：装配代码 `getattr(type(mw), "_next_anchor", None)`，O(1) 查询。
- **零运行时开销**：装饰器只在类定义时执行一次。

类似范式在 SQLAlchemy 的 `Mapper`、Pydantic 的 `Field`、FastAPI 的 `Depends` 里都有体现。**deer-flow 把它用在中间件链装配，是 Pythonic 程度很高的选择**。

### 5.3 `RuntimeFeatures` vs 传统"配置类"

传统配置类长这样：

```python
@dataclass
class AgentConfig:
    enable_sandbox: bool = True
    sandbox_middleware_class: type | None = None
    sandbox_middleware_init_kwargs: dict = field(default_factory=dict)
    enable_memory: bool = False
    memory_middleware_class: type | None = None
    ...
```

每个 feature 要 3 个字段（开关 / 类引用 / 构造参数），冗长。

deer-flow 的 `RuntimeFeatures` 用 union type 把"开关"和"实例"合一：

```python
sandbox: bool | AgentMiddleware = True
```

- `True` → 内置默认
- `False` → 关闭
- `instance` → 用户自己实例化好的

**减少了一半字段**。代价是用户要自己实例化（不能传"类 + kwargs"让框架代办），但实际项目里用户多半本来就要做这事——例如 `MemoryMiddleware(agent_name="my-agent")` 总得显式传 agent_name。

### 5.4 vs LangGraph "graph builder" 风格

LangGraph 官方推荐 builder 模式：

```python
graph = StateGraph(State)
graph.add_node("agent", agent_node)
graph.add_node("tools", tool_node)
graph.add_edge("agent", "tools")
graph = graph.compile()
```

deer-flow 直接调用 `langchain.create_agent(...)`，**不自己 build graph**——`create_agent` 内部已经把"agent → tool → agent loop"的图建好了，middlewares 是 hook 进这个 loop。

这是更**约束更小、扩展性更强**的选择——你不需要懂"图怎么画"，你只要懂"loop 的每个 hook 点做什么"。代价是 graph 结构被锁死（必须是 agent loop 形态），不适合做"自定义 DAG"。但对 99% 的 agent 用例够了。

---

## 6. 实操教程（Hands-on Lab）

### 6.1 最小可运行示例：用 `create_deerflow_agent` 跑一个零配置 agent

```python
# backend/debug_sdk_agent.py
"""用 create_deerflow_agent 跑一个最简单的 agent，不依赖 config.yaml"""
import asyncio
from langchain_openai import ChatOpenAI
from langchain.tools import tool
from langgraph.checkpoint.memory import InMemorySaver

from deerflow.agents import create_deerflow_agent, RuntimeFeatures


@tool
def get_time() -> str:
    """Return the current time in ISO format."""
    from datetime import datetime, UTC
    return datetime.now(UTC).isoformat()


async def main():
    # 1. 创建一个最小 agent — 只有 LoopDetection（默认 True）和 ToolErrorHandling（always on）
    model = ChatOpenAI(model="gpt-4o-mini")    # 需要 OPENAI_API_KEY env

    agent = create_deerflow_agent(
        model=model,
        tools=[get_time],
        features=RuntimeFeatures(
            sandbox=False,        # 关掉沙箱（最小化）
            memory=False,
            vision=False,
            subagent=False,
            auto_title=False,
            loop_detection=True,  # 保留默认
        ),
        system_prompt="You are a helpful assistant. Use tools when relevant.",
        checkpointer=InMemorySaver(),
    )

    # 2. 跑一次
    config = {"configurable": {"thread_id": "sdk-demo"}}
    result = await agent.ainvoke(
        {"messages": [{"role": "user", "content": "What time is it now?"}]},
        config=config,
    )
    print("AI:", result["messages"][-1].content)


if __name__ == "__main__":
    asyncio.run(main())
```

跑：`cd backend && PYTHONPATH=. uv run python debug_sdk_agent.py`

**能学到**：
- `create_deerflow_agent` 不读任何 config.yaml，纯参数化。
- 即使关掉所有 feature，`DanglingToolCallMiddleware + ToolErrorHandlingMiddleware + LoopDetectionMiddleware + ClarificationMiddleware` 仍是 always-on 的。
- 加上 `ask_clarification` 工具（feature 自动注入的）。

### 6.2 Debug 任务清单

#### 实验 ①：观察 mode 互斥的 fail-fast 校验

```python
# 试两次 — 第一次应该 ValueError，第二次成功
from deerflow.agents import create_deerflow_agent, RuntimeFeatures
from unittest.mock import MagicMock

# 1. middleware + features 同时传 → ValueError
try:
    create_deerflow_agent(MagicMock(), middleware=[], features=RuntimeFeatures())
except ValueError as e:
    print("ValueError as expected:", e)

# 2. middleware + extra_middleware 同时传 → ValueError
try:
    create_deerflow_agent(MagicMock(), middleware=[], extra_middleware=[MagicMock()])
except ValueError as e:
    print("ValueError as expected:", e)
```

**能学到**：deer-flow 把"难以发现的配置冲突"在工厂入口立刻报错。这是 API 设计的常见好范式。

#### 实验 ②：用 `@Next/@Prev` 插入中间件并观察顺序

```python
# 起一个 IPython
from langchain.agents.middleware import AgentMiddleware
from deerflow.agents import create_deerflow_agent, RuntimeFeatures
from deerflow.agents.features import Next, Prev
from deerflow.agents.middlewares.dangling_tool_call_middleware import DanglingToolCallMiddleware
from deerflow.agents.middlewares.clarification_middleware import ClarificationMiddleware
from unittest.mock import MagicMock, patch


class LogBefore(AgentMiddleware):
    def __init__(self): super().__init__()


@Next(DanglingToolCallMiddleware)
class LogAfter(AgentMiddleware):
    def __init__(self): super().__init__()


with patch("deerflow.agents.factory.create_agent") as mock_create:
    mock_create.return_value = MagicMock()
    create_deerflow_agent(
        MagicMock(),
        features=RuntimeFeatures(sandbox=False),
        extra_middleware=[LogBefore(), LogAfter()],
    )

    mws = mock_create.call_args[1]["middleware"]
    print("\n=== Middleware chain ===")
    for i, m in enumerate(mws):
        print(f"  {i}: {type(m).__name__}")
```

**预期观察**：

- `LogAfter` 紧跟在 `DanglingToolCallMiddleware` 后面（因为 `@Next(DanglingToolCallMiddleware)`）。
- `LogBefore` 没有 anchor，被放在 `ClarificationMiddleware` 之前（默认位置）。
- `ClarificationMiddleware` 始终是最后一个。

#### 实验 ③：故意制造循环依赖

```python
from langchain.agents.middleware import AgentMiddleware
from deerflow.agents.features import Next
from deerflow.agents import create_deerflow_agent, RuntimeFeatures
from unittest.mock import MagicMock

class A(AgentMiddleware): pass
class B(AgentMiddleware): pass

# A 锚 B，B 锚 A → 谁都解不出来
A_decorated = Next(B)(A)
B_decorated = Next(A)(B)

try:
    create_deerflow_agent(
        MagicMock(),
        features=RuntimeFeatures(sandbox=False),
        extra_middleware=[A_decorated(), B_decorated()],
    )
except ValueError as e:
    print("Circular detected:", e)
# 应该输出 "Circular dependency among extra middlewares: A, B"
```

**能学到**：`_insert_extra` 的迭代收敛算法能在不显式构建依赖图的情况下检测出循环。

---

## 7. 与下一模块的衔接

读完本章你应该能：

- 解释 deer-flow 为什么有两个 agent 工厂函数，各自服务什么消费者。
- 看懂 `RuntimeFeatures` 的 `bool | AgentMiddleware` 这种 union 设计的工程取舍。
- 用 `@Next` / `@Prev` 装饰器把自定义中间件插到中间件链的正确位置。
- 理解 `_insert_extra` 的迭代算法如何检测循环 / 不可达锚点。

但你还**没看到**完整的中间件链长什么样、它们各自的责任是什么、为什么 14-18 节的顺序是必须的。**06 篇（中间件链总览与构建顺序契约）** 是把这条主链路彻底讲透的章节。看完 06 后再去读后续每一个中间件的细节（11+ 节）才会有"骨架"感。

---

📌 **本章已交付**。请你检查后告诉我：
- 哪几段读起来不顺？
- 是否要补"`apply_prompt_template` 与 `agents/lead_agent/prompt.py` 怎么动态拼装 system prompt"的预告？
- 还是直接进入 06 篇？

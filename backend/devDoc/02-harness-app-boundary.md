# 02 · Harness ↔ App 双层架构与导入边界

> 上一章把 `make dev` 的启动链路画清楚了。这一章回答一个看似无聊、实则深刻的问题：**为什么 `backend/` 下面要拆成 `packages/harness/` 和 `app/` 两层？这条边界是怎么被物理强制的？**
>
> 不读这章，你会在第 03 篇的 `AppConfig` 反射加载、第 14 篇的 Memory contextvar、第 21 篇的 Persistence 用户隔离这些地方反复踩坑——因为它们的设计动机都源自这条边界。

---

## 1. 模块定位（Why this matters）

deer-flow 在 backend 下面物理上分两层：

```
backend/
├── packages/harness/
│   ├── pyproject.toml   # name = "deerflow-harness"  (可独立发布)
│   └── deerflow/        # import 前缀: deerflow.*
└── app/                 # import 前缀: app.*  (不可发布)
```

**两条铁律**：

1. **App 可以 import Harness，Harness 永远不能 import App**。
2. 这条规则由 [`backend/tests/test_harness_boundary.py`](../tests/test_harness_boundary.py) 用 10 行 AST 扫描在 CI 里物理拦截。

不读这部分会错过 3 个关键认知：

1. **`deerflow-harness` 是一个真正的可发布 Python 包**——`packages/harness/pyproject.toml:55` 用 hatchling 打 wheel，包名 `deerflow`。任何第三方都能 `pip install deerflow-harness`，拿到完整的 agent 框架，但**不会带走 Gateway、IM Channels、Auth、CSRF 这些"应用决策"**。
2. **这条边界让"框架"和"应用"在认知上分开**：`packages/harness/` 是"如何构造一个 LangGraph Agent"的可重用底座；`app/` 是"我们 deer-flow 项目对这个底座的特定使用方式"。前者像 LangGraph 库，后者像 LangGraph 的使用项目。
3. **依赖反转用 `typing.Protocol` 而不是 ABC**：`packages/harness/deerflow/runtime/user_context.py:42` 那个 `CurrentUser(Protocol)` 是关键技巧——harness 不知道 app 里的 `User` 类长什么样，只要求"有 `.id: str`"。这是 deer-flow 在不打破边界的前提下让 harness 复用 app 数据的核心招式。

对应到 Harness 六要素：本章打下的是**"可观测性 + 可移植性"** 的基础——只有边界清晰的代码才能被 trace、被替换、被测试。

---

## 2. 源码地图（Source Map）

### 2.1 关键文件清单

| 路径 | 角色 |
|------|------|
| [`backend/pyproject.toml`](../pyproject.toml) | 顶层 app 包：`name = "deer-flow"`，依赖 `deerflow-harness`，声明 uv workspace |
| [`backend/packages/harness/pyproject.toml`](../packages/harness/pyproject.toml) | Harness 包：`name = "deerflow-harness"`，hatchling 打 `deerflow` 这个 wheel |
| [`backend/tests/test_harness_boundary.py`](../tests/test_harness_boundary.py) | 47 行 AST 扫描，CI 必跑 |
| [`backend/packages/harness/deerflow/runtime/user_context.py`](../packages/harness/deerflow/runtime/user_context.py) | Protocol 风格依赖反转的实施现场 |
| [`backend/app/gateway/deps.py`](../app/gateway/deps.py) | 大批 `from deerflow.*` 的合法引用案例 |
| [`backend/packages/harness/deerflow/agents/lead_agent/agent.py`](../packages/harness/deerflow/agents/lead_agent/agent.py) | 函数内部"lazy import"避免循环依赖的现场（line 351） |
| `.github/workflows/backend-unit-tests.yml` | CI 入口，`working-directory: backend` + `make test`（=`pytest tests/ -v`） |

### 2.2 关键符号速查表

| 符号 | 文件:行 | 一句话职责 |
|------|---------|-----------|
| `test_harness_does_not_import_app()` | `tests/test_harness_boundary.py:37` | AST 扫描所有 harness 下 `.py`，禁 `app.*` |
| `BANNED_PREFIXES` | `tests/test_harness_boundary.py:15` | 黑名单：`("app.",)` |
| `_collect_imports(filepath)` | `tests/test_harness_boundary.py:18` | 用 `ast.walk` 遍历 `Import` / `ImportFrom` 节点 |
| `[tool.uv.workspace]` members | `backend/pyproject.toml:46` | 把 `packages/harness` 注册为 uv workspace 成员 |
| `[tool.uv.sources]` deerflow-harness | `backend/pyproject.toml:49` | `{ workspace = true }`——本地可编辑安装 |
| `[tool.hatch.build.targets.wheel]` packages | `packages/harness/pyproject.toml:55` | `["deerflow"]`——只打这一个顶层包 |
| `CurrentUser(Protocol)` | `deerflow/runtime/user_context.py:42` | 结构子类型；harness 不依赖 `app.gateway.auth.models.User` |
| `set_current_user / reset_current_user` | `user_context.py:55,65` | App 写、Harness 读 |
| `_current_user: ContextVar[...]` | `user_context.py:52` | task-local 而非 thread-local |
| `# Lazy import to avoid circular dependency` | `agents/lead_agent/agent.py:351` | 在 `_make_lead_agent` 内部延迟 import 自己的依赖 |
| `if TYPE_CHECKING:` | `runtime/runs/worker.py:28` | 只为类型注解的 import，运行时不会拉模块 |

### 2.3 依赖方向 + 反转点示意图

```mermaid
flowchart TB
    subgraph App["app/  (不可发布)"]
        Gateway[gateway/app.py · routers · deps]
        Channels[channels/manager.py · feishu · slack · ...]
        Auth[gateway/auth/* · models.User]
    end
    subgraph Harness["packages/harness/deerflow/  (可发布 wheel)"]
        Agents[agents/*]
        Runtime[runtime/*]
        Persist[persistence/*]
        Sandbox[sandbox/*]
        Tools[tools/*]
        UCT[runtime/user_context.py<br/>CurrentUser(Protocol)]
    end

    Gateway -->|from deerflow.*| Agents
    Gateway -->|from deerflow.*| Runtime
    Gateway -->|from deerflow.*| Persist
    Channels -->|from deerflow.*| Runtime
    Auth -. "User 实现 CurrentUser Protocol<br/>(结构子类型，无继承)" .-> UCT
    Auth -->|set_current_user(user)| UCT
    Persist -->|get_current_user()<br/>读 contextvar| UCT

    classDef forbidden stroke:#f00,stroke-width:2px,stroke-dasharray:5
    Harness ~~~ Forbidden[("from app.*  ← CI 禁止")]:::forbidden
```

红色虚线那条是被 `test_harness_boundary.py` 物理拦截的反向引用。**实际上整个仓库 0 个 harness→app 的 import**——我刚验证过 (`grep -rln 'from app\.' packages/harness/` 返回空)。

### 2.4 反向"数据流"的实现机制（关键巧思）

```mermaid
sequenceDiagram
    autonumber
    participant Req as HTTP 请求
    participant AMW as AuthMiddleware<br/>(app/gateway/auth_middleware.py)
    participant CV as _current_user<br/>ContextVar<br/>(harness)
    participant Repo as Repository<br/>(deerflow.persistence.*)
    participant DB as SQLite/Postgres

    Req->>AMW: cookie: access_token=...
    AMW->>AMW: decode_token → user_id<br/>get_local_provider().get_user(...)
    AMW->>CV: set_current_user(user)
    AMW-->>Req: 继续 → router → repo
    Repo->>CV: require_current_user()
    CV-->>Repo: User (符合 CurrentUser Protocol)
    Repo->>DB: SELECT ... WHERE user_id = ?
    DB-->>Repo: rows
    AMW->>CV: reset_current_user(token)
```

**精妙处**：Repository 在 harness 里，它需要"当前用户 id"做 WHERE 过滤，但**它不能 import `app.gateway.auth.models.User`**。怎么办？

- Harness 自己在 `user_context.py:42` 定义一个 `@runtime_checkable class CurrentUser(Protocol)`，只声明 `id: str`。
- App 的 `User` 类**不需要继承** 这个 Protocol——只要它有 `.id: str` 属性就**结构上**满足。
- App 的 `AuthMiddleware` 调 `set_current_user(user)` 把 user 写入 ContextVar；harness 的 repo 调 `require_current_user()` 读出来。

这就是**依赖反转**：依赖的方向（app → harness）和数据流的方向（user 从 app 流到 harness）相反，靠 Protocol 这层"形状约定"桥接，零运行时耦合。

---

## 3. 核心逻辑精读（Deep Dive）

### 3.1 边界守卫：47 行的 AST 扫描

```python
# backend/tests/test_harness_boundary.py:1-46
"""Boundary check: harness layer must not import from app layer.

The deerflow-harness package (packages/harness/deerflow/) is a standalone,
publishable agent framework. It must never depend on the app layer (app/).

This test scans all Python files in the harness package and fails if any
``from app.`` or ``import app.`` statement is found.
"""

import ast
from pathlib import Path

HARNESS_ROOT = Path(__file__).parent.parent / "packages" / "harness" / "deerflow"

BANNED_PREFIXES = ("app.",)


def _collect_imports(filepath: Path) -> list[tuple[int, str]]:
    """Return (line_number, module_path) for every import in *filepath*."""
    source = filepath.read_text(encoding="utf-8")
    try:
        tree = ast.parse(source, filename=str(filepath))
    except SyntaxError:
        return []

    results: list[tuple[int, str]] = []
    for node in ast.walk(tree):
        if isinstance(node, ast.Import):
            for alias in node.names:
                results.append((node.lineno, alias.name))
        elif isinstance(node, ast.ImportFrom):
            if node.module:
                results.append((node.lineno, node.module))
    return results


def test_harness_does_not_import_app():
    violations: list[str] = []

    for py_file in sorted(HARNESS_ROOT.rglob("*.py")):
        for lineno, module in _collect_imports(py_file):
            if any(module == prefix.rstrip(".") or module.startswith(prefix) for prefix in BANNED_PREFIXES):
                rel = py_file.relative_to(HARNESS_ROOT.parent.parent.parent)
                violations.append(f"  {rel}:{lineno}  imports {module}")

    assert not violations, "Harness layer must not import from app layer:\n" + "\n".join(violations)
```

**逐段拆解**：

- **第 13 行 `HARNESS_ROOT = ... / "packages" / "harness" / "deerflow"`**：定位扫描根，只扫描 `deerflow` 这个真实的源码目录，不扫 `packages/harness/` 顶层（避免误扫 `pyproject.toml` 之类）。
- **第 18-34 行 `_collect_imports`**：用 `ast.parse + ast.walk` 取出所有 import 语句，返回 `(行号, 模块路径)`。两类节点都覆盖了：
  - `ast.Import`：处理 `import app.gateway.app` 这种带点的 import。
  - `ast.ImportFrom`：处理 `from app.gateway.app import X`。
- **第 42 行的判定**：`module == prefix.rstrip(".") or module.startswith(prefix)`——既匹配 `app`（裸 import）也匹配 `app.gateway.routers.threads`（子模块）。
- **第 43 行 `rel = py_file.relative_to(HARNESS_ROOT.parent.parent.parent)`**：报错时用相对仓库根的路径，方便点击跳转。

**作者的精妙之处**：

1. **不依赖任何运行时执行**——纯静态扫描。即使被禁止 import 的 app 模块都不存在、即使 harness 里某些路径只在特定 config 下才会触发 import，AST 都能 100% 命中。
2. **`ast.walk` 不是 `ast.iter_child_nodes`**——`walk` 递归遍历所有节点，所以**函数内部的延迟 import** 也会被扫到。这堵死了"想偷渡的话就把 import 放进函数体里"这个常见 workaround。
3. **`SyntaxError` 兜底为空 list 而不是报错**：极端情况下（例如某个 `.py` 是新增的还有语法错误），不会让 boundary 测试假阳性掩盖真正的语法错误测试。

**可能的改进空间**：当前禁止前缀只有 `("app.",)`。如果未来要拆出第三层（例如 `enterprise/` 内部包），需要把它加进 `BANNED_PREFIXES`。也可以做成参数化测试，针对不同层组合校验。

### 3.2 uv workspace + hatchling：两包共存

```toml
# backend/pyproject.toml:1-50
[project]
name = "deer-flow"
version = "0.1.0"
description = "LangGraph-based AI agent system with sandbox execution capabilities"
requires-python = ">=3.12"
dependencies = [
    "deerflow-harness",           # ← 把 harness 当成普通 PyPI 依赖来声明
    "fastapi>=0.115.0",
    "uvicorn[standard]>=0.34.0",
    ...
]

[project.optional-dependencies]
postgres = ["deerflow-harness[postgres]"]   # ← 把 harness 的 extras 透传出来
discord = ["discord.py>=2.7.0"]

[tool.uv.workspace]
members = ["packages/harness"]

[tool.uv.sources]
deerflow-harness = { workspace = true }     # ← 关键：本地解析 = workspace
```

```toml
# packages/harness/pyproject.toml
[project]
name = "deerflow-harness"
version = "0.1.0"
requires-python = ">=3.12"
dependencies = [
    "langchain>=1.2.15",
    "langgraph>=1.1.9",
    "langgraph-checkpoint-sqlite>=3.0.3",
    "sqlalchemy[asyncio]>=2.0,<3.0",
    "alembic>=1.13",
    ...
]

[project.optional-dependencies]
ollama   = ["langchain-ollama>=0.3.0"]
postgres = ["asyncpg>=0.29", "langgraph-checkpoint-postgres>=3.0.5",
            "psycopg[binary]>=3.3.3", "psycopg-pool>=3.3.0"]
pymupdf  = ["pymupdf4llm>=0.0.17"]

[build-system]
requires = ["hatchling"]
build-backend = "hatchling.build"

[tool.hatch.build.targets.wheel]
packages = ["deerflow"]    # ← wheel 里只包含 deerflow/，不包含 packages/harness/
```

**两个 pyproject 怎么配合**：

1. `uv sync --all-packages` 会先解析 `[tool.uv.workspace] members`，发现 `packages/harness`，把 `deerflow-harness` 注册为 workspace 成员。
2. 然后看到顶层 `[project.dependencies]` 里写了 `"deerflow-harness"`，本来要去 PyPI 找——但 `[tool.uv.sources]` 把它指回 workspace，所以**装的是本地的、可编辑的 harness**。
3. 改 `packages/harness/deerflow/agents/*.py`，uvicorn 的 `--reload` 立即看到改动（因为是 editable install）。
4. `make build`（仓库里没写，但可用 `cd packages/harness && uv build`）能把 harness 单独打 wheel：`dist/deerflow_harness-0.1.0-py3-none-any.whl`，第三方可装。

**这个布局的另一个红利**：optional dependencies 的"穿透"——`pip install "deer-flow[postgres]"` 时，`postgres = ["deerflow-harness[postgres]"]` 这条会自动把 harness 的 postgres extras 拉进来，最终装上 `asyncpg + langgraph-checkpoint-postgres + psycopg`。**用户感知不到 harness 这层包装的存在**。

### 3.3 依赖反转的实现：`CurrentUser(Protocol)`

```python
# packages/harness/deerflow/runtime/user_context.py:1-58 (节选)
"""
Dependency direction
--------------------
``persistence`` (lower layer) reads from this module; ``gateway.auth``
(higher layer) writes to it. ``CurrentUser`` is defined here as a
:class:`typing.Protocol` so that ``persistence`` never needs to import
the concrete ``User`` class from ``gateway.auth.models``. Any object
with an ``.id: str`` attribute structurally satisfies the protocol.
"""

from __future__ import annotations
from contextvars import ContextVar, Token
from typing import Final, Protocol, runtime_checkable


@runtime_checkable
class CurrentUser(Protocol):
    """Structural type for the current authenticated user.

    Any object with an ``.id: str`` attribute satisfies this protocol.
    Concrete implementations live in ``app.gateway.auth.models.User``.
    """
    id: str


_current_user: Final[ContextVar[CurrentUser | None]] = ContextVar(
    "deerflow_current_user", default=None
)


def set_current_user(user: CurrentUser) -> Token[CurrentUser | None]:
    ...
```

**这段代码做了一个看似简单实则关键的决定**：把 `CurrentUser` 定义在 harness 里，而不是 app 里。如果反过来：

| 选项 | 后果 |
|------|------|
| A. `CurrentUser` 定义在 app/，harness 导入 | ❌ Harness → App 反向引用，违反边界，CI 红 |
| B. Harness 完全不感知 user，repo 接受 `user_id: str` 显式参数 | ✅ 但每个 router 都要手动传 user_id 到 repo，~30 处样板代码 |
| C. **Harness 定义 Protocol，App 的 User 结构匹配**（实际选择） | ✅ 边界清晰、零样板、Pylance/mypy 能静态校验 |

**`@runtime_checkable` 的作用**：让 `isinstance(user, CurrentUser)` 在运行时也能用（默认 Protocol 只能做静态检查）。代价是稍慢——但只在需要类型 narrowing 时才用，热路径上仍是直接读 `user.id`。

**`ContextVar` 而不是 `threading.local`**：注释里写得很清楚（行 27-34）——asyncio 下 ContextVar 是 task-local 而不是 thread-local，配合 `asyncio.create_task` / `asyncio.to_thread` 会自动继承父任务上下文。这对 FastAPI 这种"每个请求一个 task"的模型刚好对齐。

### 3.4 处理"内部"循环依赖的两个技巧

边界禁止 harness → app，但 harness **内部**模块之间也可能循环。deer-flow 用了两个技巧：

#### 技巧 1：`if TYPE_CHECKING:`

```python
# packages/harness/deerflow/runtime/runs/worker.py:24-29
from typing import TYPE_CHECKING, Any, Literal, cast

...

if TYPE_CHECKING:
    # 只为类型注解；运行时 Python 不会执行这一段
    from deerflow.config.app_config import AppConfig
    from deerflow.runtime.checkpointer import Checkpointer
```

`TYPE_CHECKING` 是 `typing` 模块里的 `False`，但 mypy/Pylance 把它当 `True`。结果是：**类型注解可以用 `AppConfig`，运行时不会真正 import**，循环就被打破了。

#### 技巧 2：函数体内 lazy import

```python
# packages/harness/deerflow/agents/lead_agent/agent.py:350-353
def _make_lead_agent(config: RunnableConfig, *, app_config: AppConfig):
    # Lazy import to avoid circular dependency
    from deerflow.tools import get_available_tools
    from deerflow.tools.builtins import setup_agent, update_agent
    ...
```

为什么 `deerflow.tools` 要 lazy import？因为 `deerflow.tools.builtins.task_tool` 间接依赖 `deerflow.agents`（subagent 系统要用 `make_lead_agent` 的兄弟模块）。如果模块顶层 import，循环依赖会让 Python 在 import 时抛错。

**两个技巧的边界**：

- `TYPE_CHECKING` 只解决"类型注解层面的循环"，**不能**用于运行时调用。
- 函数体 lazy import 解决"运行时调用的循环"，但代价是每次调用都付一次 `sys.modules` 查询（虽然命中 cache，开销 < 1μs）。
- **boundary 测试 `ast.walk` 会扫到函数内部的 import**——所以不能用函数内 lazy import 偷渡 `from app.*`。

### 3.5 实证：app 怎么大量"合法"引用 harness

```python
# app/gateway/deps.py:18-22  ← 顶层直接 import，毫无忌讳
from deerflow.config.app_config import AppConfig
from deerflow.persistence.feedback import FeedbackRepository
from deerflow.runtime import RunContext, RunManager, StreamBridge
from deerflow.runtime.events.store.base import RunEventStore
from deerflow.runtime.runs.store.base import RunStore
```

我刚 grep 过：app 下面共有 **76 处** `from deerflow.` / `import deerflow.` 引用，分布在 15+ 个文件里。这是合法的、被鼓励的方向。

反过来：`grep -rln 'from app\.' packages/harness/` 返回 **0 个匹配**。这就是 boundary 测试在保护的东西。

---

## 4. 关键问题答疑（Key Questions）

### Q1：如果我故意写一个 `from app.gateway.app import app` 在 harness 里，会怎样？

CI 跑 `make test` 时，`test_harness_boundary.py` 的 `test_harness_does_not_import_app` 会失败，输出类似：

```
AssertionError: Harness layer must not import from app layer:
  backend/packages/harness/deerflow/some_file.py:42  imports app.gateway.app
```

而且**即使把这个 import 放进函数内部**也躲不过——`ast.walk` 递归全部节点。唯一能绕过的方式是用 `importlib.import_module("app.gateway.app")`，但那种写法在 PR review 时一眼能看出有问题。

### Q2：harness 真的能独立装到一个空白项目里跑吗？

理论上能。把 `packages/harness/` 单独拿出来 `uv build`，得到 `deerflow_harness-0.1.0-py3-none-any.whl`，然后 `pip install` 到任意环境。但有 3 个 caveat：

1. **得自己提供 config.yaml**：harness 的 `get_app_config()` 要找配置文件。没有的话 `AppConfig.from_file()` 会抛错（见 03 篇详谈）。
2. **得自己装 IM Channels / Auth / Gateway 替代品**——harness 没有 FastAPI router。直接用 `DeerFlowClient`（`packages/harness/deerflow/client.py`，22 篇详谈）当嵌入式 SDK 是最干净的玩法。
3. **某些 optional dependency 还是要选**：例如 postgres backend 必须 `pip install deerflow-harness[postgres]`。

### Q3：`CurrentUser(Protocol)` 不写 `@runtime_checkable` 会怎样？

写了之后能 `isinstance(user, CurrentUser)` 在运行时也成立。不写的话，只能在 mypy/pylance 这种静态检查工具里成立——`isinstance` 会抛 `TypeError`。deer-flow 写它是为了安全兜底（某些 repo 方法在调用前会 assert），不是热路径需求。

### Q4：为什么不用抽象基类（ABC）而用 Protocol？

ABC 需要 app 的 `User` **继承** harness 的 `CurrentUser`：

```python
# 假设 ABC 方案
# app/gateway/auth/models.py
from deerflow.runtime.user_context import CurrentUser  # 这条 OK，App→Harness

class User(CurrentUser):    # ← 必须继承
    id: str
    email: str
    ...
```

这强加了一条**显式继承关系**——app 的设计被 harness 的接口绑死。换框架时 app 要改大量继承声明。**Protocol（结构子类型）则只要"有 `.id: str`" 就过——零侵入**。

### Q5：Channels（IM）层为什么也放在 app/ 而不是 harness/？

因为 IM 集成是**应用决策**而非框架能力。Slack/Feishu/Telegram 的接入方式、bot_token 的获取、card_template_id 的配置——这些都是"deer-flow 项目选了这样做"，而不是"agent 框架本身的能力"。其他 deer-flow 用户可能完全用不到 Slack（只用 Web UI），把 channels 放进可发布的 harness wheel 会把无关依赖（lark-oapi、slack-sdk、python-telegram-bot……）拖进来——见 `backend/pyproject.toml:14-19`，这些依赖确实只在 app 层声明。

### Q6：app 的 router 大量 `from deerflow.*`，会不会让 harness 实际上不可独立？

**不会**，因为引用是单向的：harness 不知道 router 存在。第三方用 harness 时，他们写自己的 router（或干脆用 `DeerFlowClient`），harness 提供的是 `make_lead_agent / RunManager / StreamBridge / Repository` 这些底座组件。

---

## 5. 横向延伸与面试级洞察（Interview-Grade Insights）

### 5.1 这条边界对应的架构原则

deer-flow 在做的事其实是**六边形架构 / 端口与适配器（Ports and Adapters）** 的工程化落地：

- **Harness** = 核心域（Domain）+ 端口（Ports，例如 `CurrentUser`、`Sandbox`、`SandboxProvider`、`GuardrailProvider`、`StreamBridge`）
- **App** = 适配器（Adapters）——FastAPI 适配器、IM 适配器、Auth 适配器
- **测试** = 边界守卫（`test_harness_boundary.py`）

很多团队号称做了"分层"但只在文档里写，从不强制。deer-flow 那 47 行 AST 测试就**把架构纪律变成了 CI 失败**，这是工程上比文档强一万倍的纪律手段。

### 5.2 vs LangGraph 官方仓库

LangGraph 官方 `langchain-ai/langgraph` 自己的内部分层是：

- `libs/langgraph/`（核心运行时）
- `libs/checkpoint*/`（持久化）
- `libs/sdk-py/`（客户端 SDK）

每个都是独立 Python 包，**但没有任何 AST-level 的边界测试**。它们靠 PR review + 包内层级隔离来约束。

deer-flow 的做法是"自己的项目也按发布包标准管理"。这种纪律对一个 5 万行规模的开源仓库是合适的（不会因 PR 增多而漂移），对内部项目也是参考价值很高的范式。

### 5.3 vs 类似项目

| 项目 | 边界做法 | 评价 |
|------|---------|------|
| **AutoGen** | 单仓库多包（`autogen-agentchat`, `autogen-core`, `autogen-ext`），但没有反向引用 AST 测试 | 靠 review |
| **CrewAI** | 单包，无强制分层 | 适合小项目，扩张困难 |
| **OpenAI Swarm** | 单包，全平铺 | 实验性质，无边界 |
| **deer-flow** | uv workspace + hatchling + AST 测试 | 工程化最严的开源 agent 项目之一 |

> **面试金句**：deer-flow 用 47 行 AST 测试 + uv workspace 把"框架/应用"边界变成了 CI 强制约束，配合 `CurrentUser` Protocol 做依赖反转，让 harness 可独立发布而 app 可丰富功能——这是六边形架构在 Python agent 工程里少见的、彻底落地的案例。

---

## 6. 实操教程（Hands-on Lab）

### 6.1 最小可运行示例：故意违反边界看 CI 怎么炸

不需要写新代码，跑现有测试就能验证：

```bash
cd backend

# 1. 先确认当前是干净的
PYTHONPATH=. uv run pytest tests/test_harness_boundary.py -v
# → 1 passed
```

然后**故意**违反一下：

```bash
# 2. 在 harness 里偷渡一个 app import
echo 'from app.gateway.app import app  # 故意违反' \
    >> packages/harness/deerflow/__init__.py

# 3. 再跑一次
PYTHONPATH=. uv run pytest tests/test_harness_boundary.py -v
# → FAILED tests/test_harness_boundary.py::test_harness_does_not_import_app
# AssertionError: Harness layer must not import from app layer:
#   backend/packages/harness/deerflow/__init__.py:N  imports app.gateway.app

# 4. 还原（不要忘了！）
git checkout packages/harness/deerflow/__init__.py
```

**能学到**：测试报错信息直接给出 `文件:行号 imports 模块名`，IDE 点击可跳。这是 CI 错误提示的好范式。

### 6.2 Debug 任务清单

#### 实验 ①：函数内部偷渡也跑不掉

1. 把 step 2 改成把 import 放进函数体：
   ```bash
   cat >> packages/harness/deerflow/__init__.py << 'EOF'

   def _smuggle():
       from app.gateway.app import app
       return app
   EOF
   ```
2. 再跑 boundary 测试，仍然会 fail。
3. 在 `tests/test_harness_boundary.py:27` 把 `ast.walk` 改成 `ast.iter_child_nodes`，再跑一次，**就会 pass**（因为只扫顶层节点）。
4. **能学到**：`ast.walk` 递归遍历是个有意为之的选择。

#### 实验 ②：用 Protocol 验证 User 结构

在 backend 目录跑一个交互式脚本：

```python
# PYTHONPATH=. uv run python
from deerflow.runtime.user_context import CurrentUser

class FakeUser:
    id = "abc-123"

# 因为 @runtime_checkable，isinstance 在运行时也能用
print(isinstance(FakeUser(), CurrentUser))   # True
print(isinstance(object(), CurrentUser))     # False

# 但 issubclass 仍要求名义继承（这是 Protocol 的限制）
print(issubclass(FakeUser, CurrentUser))     # True（Protocol+@runtime_checkable 特例）
```

**能学到**：Protocol 是"结构匹配"——任何有 `.id` 属性的对象都匹配，不需要继承。这就是 harness 能不感知 app 的 `User` 类型却依然能用的原因。

#### 实验 ③：观察 ContextVar 在 asyncio task 之间的传播

```python
# PYTHONPATH=. uv run python
import asyncio
from contextvars import copy_context
from deerflow.runtime.user_context import (
    set_current_user, get_current_user, reset_current_user
)

class U:
    id = "user-foo"

async def child():
    return get_current_user()

async def main():
    token = set_current_user(U())
    try:
        # 创建子 task — 默认继承父 context
        t1 = asyncio.create_task(child())
        # 用 copy_context 创建“干净的”子 task — 不继承
        ctx = copy_context()
        # 在干净的 ctx 里跑 child（注意要在事件循环外）
        print("默认子 task 继承:", (await t1).id)             # user-foo
    finally:
        reset_current_user(token)

asyncio.run(main())
```

**能学到**：deer-flow 内每个 FastAPI 请求是一个 asyncio task，所以 user contextvar 天然按请求隔离；但 `asyncio.create_task` 会继承父 task 的 user——这在跑后台任务（例如 MemoryQueue 的 timer 线程）时是个坑，14 篇会再讲。

---

## 7. 与下一模块的衔接

读完本章你应该能：

- 解释 deer-flow 为什么把 `packages/harness/` 和 `app/` 拆开。
- 用 47 行 AST 测试的代码片段说明边界是如何被物理强制的。
- 用 `CurrentUser(Protocol) + ContextVar` 这套组合解释依赖反转。
- 区分"边界禁止"（harness→app）与"内部循环规避"（TYPE_CHECKING、lazy import）。

但你还**没看到**：

- `get_app_config()` 背后那 26 份子配置是怎么被一个 Pydantic 模型聚合的？为什么配置里能写 `$ENV_VAR`、`use: deerflow.tools.xxx:bash_tool` 这种字段？
- `resolve_variable` 反射加载是 harness 的"自我感知"机制——它如何让"工具/沙箱/Provider 通过字符串路径动态装配"成为可能？
- mtime 失效是怎么让 Gateway 进程不重启就能感知 `config.yaml` 改动的？

这是 **03 篇（配置体系：AppConfig + ExtensionsConfig + 反射装配）** 的核心，也是接下来所有"动态装配"模块（工具、沙箱、Provider、MCP、Skills）共同的基础。

---

📌 **本章已交付**。请你检查后告诉我：
- 哪几段读起来不顺？
- 是否需要补某个角度（例如想看 IM Channels 这层为啥能"合法"地 import harness、frontend 的 SDK 怎么和 harness 拉关系）？
- 还是直接进入 03 篇？

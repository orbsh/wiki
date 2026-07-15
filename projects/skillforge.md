# SkillForge

多引擎 Agent 框架，封装 Agno + Hermes Agent，提供统一技能加载、上下文传播和 OpenAI 兼容 API 服务器。

## 仓库

`~/world/skillforge/`

## 架构

```
skillforge/
├── src/skillforge/
│   ├── core.py          # Agent 创建、ContextVar + 子进程补丁
│   ├── config.py        # Pydantic 设置（toml + env）
│   ├── api_server.py    # FastAPI SSE/WS 端点、Jinja2 工具格式化
│   ├── events.py        # 流事件定义（共享）
│   ├── tool_format.py   # Jinja2 工具输出格式化（共享）
│   ├── auth_context.py  # Token 提取（header/cookie/query）
│   ├── context.py       # AgentContext（user_id、session_id、metadata）
│   └── engine/
│       ├── agno.py      # Agno 引擎封装
│       └── hermes.py    # Hermes 引擎封装
├── config.toml           # Jinja2 模板、认证、引擎类型、LLM
└── skills/               # SKILL.md 文件
```

## 关键设计模式

### ContextVar + 子进程补丁

为并发异步部署提供每请求上下文隔离。`core.py` 猴子补丁 `subprocess.run` 和 `subprocess.Popen`，从当前线程的 ContextVar 注入 `CONTEXT_*` 环境变量。

**为何补丁两者：** Agno 引擎使用 `subprocess.run`，Hermes 终端使用 `subprocess.Popen`。只补丁一个会让另一个引擎损坏。

**为何不用 `os.environ.update()`：** 在 CLI/单线程模式下有效，但在 uvicorn 线程池中并发请求间泄漏上下文。

**线程边界修复：** Hermes 在 `threading.Thread` 中运行 Agent 循环。ContextVars 不传播到新线程——上下文必须在线程入口点重新绑定：

```python
ctx = self._context  # 创建线程前捕获

def _run_sync():
    if ctx is not None:
        set_agent_context_env(ctx)  # 在工作线程中重新绑定
    # ... Agent 执行 ...
```

### 技能加载

不同引擎处理技能方式不同：

| 引擎 | 机制 | 需要什么 |
|------|------|---------|
| Agno | `Skills(LocalSkills(...))` | 传递 `skills=` 给 Agent——自动注入 SKILL.md |
| Hermes AIAgent | 无 | 手动读取 SKILL.md 并注入到 `ephemeral_system_prompt` |

Hermes 使用**目录模式**——只将 SKILL.md frontmatter（名称 + 描述 + 路径）注入系统提示。模型在需要时通过 `read_file` 加载完整内容。这避免 5+ 技能时上下文膨胀。

### 工具输出格式化

`config.toml` 中的 Jinja2 模板，由 `api_server.py` 和 `hermes.py` 共同消费：

```toml
tool_call_format        = "\n🔧[工具调用] {{tool_name}}{% if tool_args %} → {{tool_args | cli_args}}{% endif %}\n"
tool_call_result_format = "\n✅[工具完成] {{tool_name}}\n"
tool_call_error_format  = "\n❌[工具失败] {{tool_name}}\n"
```

Hermes 发出原始 `ToolCallStartedEvent` / `ToolCallCompletedEvent`，带 `ToolCallInfo` 包装器（`tool_name`、`tool_args`、`result`、`is_error`）。API 服务器接收这些事件并渲染模板——引擎从不硬编码格式字符串。

### 认证上下文

从请求中提取 Token（按 `auth_method` 配置的 header/cookie/query），存储在 `AgentContext.metadata["access_token"]`。技能脚本通过子进程补丁注入的 `CONTEXT_METADATA_ACCESS_TOKEN` 环境变量读取。

## 调试清单

当一个引擎工作而另一个不工作时：

1. **技能注入** — SKILL.md 内容在系统提示中吗？（Hermes 需要手动注入）
2. **线程上下文** — 后台线程重新绑定了 ContextVar？
3. **子进程路径** — `run` 和 `Popen` 都补丁了？
4. **环境构建时机** — 上下文在 `_make_run_env()` 读取 `os.environ` 前设置？

---

## SkillForge 引擎集成模式

### 工具调用输出格式：使用配置驱动的模板

构建需要格式化工具调用输出的 Hermes 集成时（用于 CoT 流、网关响应等），**不要硬编码格式字符串**。使用配置中的 Jinja2 模板：

```toml
# config.toml
tool_call_format = "\n🔧[工具调用] {{tool_name}}{% if tool_args %} → {{tool_args | cli_args}}{% endif %}\n"
tool_call_result_format = "\n✅[工具完成] {{tool_name}}\n"
tool_call_error_format = "\n❌[工具失败] {{tool_name}}\n"
```

将渲染逻辑提取到共享模块（如 `tool_format.py`），带 `cli_args` Jinja2 过滤器将参数 dict 转换为 CLI 风格参数。引擎层（Hermes 回调）和 API 层（SSE 流）都应使用相同模板。

### `tool_progress_callback` 签名

为 `AIAgent` 实现 `tool_progress_callback` 时，回调签名是：

```python
def callback(event_type: str, tool_name: str, preview: str, tool_args: dict, **kwargs):
    ...
```

- **`event_type`**：`"tool.started"`、`"tool.completed"` 等。
- **`tool_name`**：工具名称字符串。
- **`preview`**：人类可读的预览字符串（如正在运行的命令）。
- **`tool_args`**：工具参数的实际 **dict** — 这是**第 4 个位置参数**，不是 `**kwargs` 的一部分。使用 `*args` 会静默吞掉它；使用 `kwargs.get("tool_args")` 会返回空 dict。

对于 `"tool.completed"`，额外的 kwargs 包括 `is_error`（bool）和 `result`（str）。

### 引擎封装的事件驱动架构

封装 Hermes AIAgent 供 API 服务器或其他消费者使用时：

**引擎层**（`hermes.py` 风格）：只发出原始结构化事件。使用轻量级 `ToolCallInfo` 包装器，带 `tool_name`、`tool_args`、`result`、`is_error` 字段。不要在引擎中格式化输出字符串。

**消费者层**（`api_server.py` 风格）：接收事件，从配置渲染 Jinja2 模板，向响应流发出格式化文本。

这种分离确保两个引擎产生相同输出（相同模板），格式变更不需要引擎代码变更。

### 异步引擎封装的 ContextVar + 子进程补丁

构建将每请求上下文注入子进程的异步 Agent 框架（如 skillforge）时：

**模式：** 使用 `ContextVar` 进行每请求隔离，猴子补丁 `subprocess.run` 和 `subprocess.Popen`，在调用时注入上下文变量。

**关键陷阱：**
1. **ContextVars 不传播到线程。** 如果引擎卸载到 `threading.Thread`，在线程入口重新绑定 ContextVar：在线程目标函数内 `set_agent_context_env(ctx)`。否则 `ContextVar.get()` 返回默认值（None），上下文丢失。
2. **必须补丁 `run` 和 `Popen`。** Agno 使用 `subprocess.run`，Hermes 终端使用 `subprocess.Popen`。只补丁一个会让另一个引擎损坏。
3. **不要在异步中使用 `os.environ.update()`。** 在 CLI/单线程模式下有效，但在 uvicorn 线程池中并发请求间泄漏上下文。补丁方法在调用时从当前线程的 ContextVar 读取，这是每请求隔离的。
4. **Hermes `_make_run_env` 在 Popen 前读取 `os.environ`。** 环境 dict 在补丁拦截前构建，但补丁通过在 Popen 时将当前线程的 ContextVar 值合并到已构建的 dict 中来纠正它。

# Claude Agent SDK 实现指南

本文档供项目 Agent 参考，用于基于 Claude Agent SDK 实现 HTTP 接口服务。

---

## 1. 项目概述

### 1.1 SDK 简介

- **包名**：`claude-agent-sdk`
- **安装**：`pip install claude-agent-sdk`
- **依赖**：Python 3.10+，需配置 `ANTHROPIC_API_KEY`（必须为 Anthropic 官方 key，不支持 OpenRouter）
- **CLI**：SDK 自带 Claude Code CLI，无需额外安装

### 1.2 核心架构

```
你的应用 → ClaudeSDKClient → Claude Code CLI 子进程 → Anthropic API
```

SDK 通过子进程启动 Claude Code CLI，实际 API 调用由 CLI 完成。运行机器需能执行 Claude Code（SDK 安装时已打包）。

---

## 2. ClaudeSDKClient 使用方式（有状态）

适用于：多轮对话、聊天应用、需要保持上下文、需要 interrupt 等能力。

```python
from claude_agent_sdk import (
    ClaudeSDKClient,
    ClaudeAgentOptions,
    AssistantMessage,
    TextBlock,
    ResultMessage,
)

async def multi_turn_chat():
    options = ClaudeAgentOptions(
        system_prompt="You are a helpful assistant.",
        allowed_tools=["Read", "Write", "Bash"],
    )
    async with ClaudeSDKClient(options=options) as client:
        # 第一轮
        await client.query("法国的首都是哪里？")
        async for msg in client.receive_response():
            if isinstance(msg, AssistantMessage):
                for block in msg.content:
                    if isinstance(block, TextBlock):
                        print(block.text)

        # 第二轮（自动携带上下文）
        await client.query("那座城市的人口多少？")
        async for msg in client.receive_response():
            # 同上
            pass
```

**上下文机制**：同一 `ClaudeSDKClient` 实例对应同一个子进程，对话历史在进程内维护，无需手动传历史消息。

---

## 3. HTTP 接口设计方案

按 session 维护 `ClaudeSDKClient`，同一 session 复用同一 client。

```python
# 会话池：session_id -> client
sessions: dict[str, ClaudeSDKClient] = {}
session_locks: dict[str, asyncio.Lock] = {}  # 每会话串行

async def get_or_create_client(session_id: str) -> ClaudeSDKClient:
    if session_id not in sessions:
        client = ClaudeSDKClient(options=...)
        await client.connect()
        sessions[session_id] = client
        session_locks[session_id] = asyncio.Lock()
    return sessions[session_id]

@app.post("/chat/{session_id}")
async def chat(session_id: str, request: ChatRequest):
    client = await get_or_create_client(session_id)
    async with session_locks[session_id]:  # 同一会话串行
        await client.query(request.message)
        replies = []
        async for msg in client.receive_response():
            # 收集回复
            ...
    return {"replies": replies}
```

**必须实现**：

- 会话超时后调用 `client.disconnect()` 并从 `sessions` 中移除
- 同一 session 的请求需串行（用锁或队列），避免消息交错

---

## 4. 常用 API 速查

| API | 用途 |
|-----|------|
| `ClaudeSDKClient(options=...)` | 交互式客户端 |
| `client.query(prompt)` | 发送用户消息 |
| `client.receive_response()` | 接收本轮完整回复（到 ResultMessage 结束） |
| `client.receive_messages()` | 持续接收消息（不自动结束） |
| `client.interrupt()` | 中断当前回复 |
| `client.set_permission_mode(mode)` | 切换权限模式 |
| `client.set_model(model)` | 切换模型 |
| `client.disconnect()` | 断开连接，释放子进程 |

---

## 5. ClaudeAgentOptions 常用配置

```python
ClaudeAgentOptions(
    system_prompt="系统提示词",
    cwd="/工作目录",
    allowed_tools=["Read", "Write", "Bash", "WebSearch", "WebFetch"],
    disallowed_tools=["Bash"],
    permission_mode="acceptEdits",  # default | acceptEdits | bypassPermissions
    max_turns=10,
    model="sonnet",  # sonnet | opus | haiku | inherit
    max_budget_usd=1.0,
    mcp_servers={...},  # MCP 服务配置
    hooks={...},       # Hooks 配置
    env={"ANTHROPIC_API_KEY": "..."},  # 覆盖环境变量
    cli_path="/path/to/claude",  # 指定 CLI 路径
)
```

---

## 6. 消息类型

```python
from claude_agent_sdk import (
    UserMessage,
    AssistantMessage,
    SystemMessage,
    ResultMessage,
    TextBlock,
    ToolUseBlock,
    ToolResultBlock,
)

# 提取文本
if isinstance(msg, AssistantMessage):
    for block in msg.content:
        if isinstance(block, TextBlock):
            text = block.text
```

---

## 7. 错误处理

```python
from claude_agent_sdk import (
    ClaudeSDKError,
    CLINotFoundError,    # CLI 未安装
    CLIConnectionError,  # 连接失败
    ProcessError,        # 子进程异常
    CLIJSONDecodeError,  # JSON 解析错误
)

try:
    async with ClaudeSDKClient() as client:
        await client.query(user_message)
        async for msg in client.receive_response():
            ...
except CLINotFoundError:
    return {"error": "Claude Code 未安装"}
except ProcessError as e:
    return {"error": f"进程异常: {e.exit_code}"}
```

---

## 8. 限制与注意

1. **ANTHROPIC_API_KEY**：必须使用 Anthropic 官方 key，不能使用 OpenRouter key。
2. **ClaudeSDKClient 单实例**：不能跨不同 asyncio.TaskGroup / trio nursery 使用，需在同一 async 上下文中完成所有操作。
3. **并发**：多会话可并行（每个会话一个 client）；同一会话内的 `query` 建议串行。
4. **资源**：每个 ClaudeSDKClient 对应一个常驻子进程，需做好会话超时与回收。

---

## 9. 参考资料

- 项目 README：`/workspace/README.md`
- 快速入门示例：`/workspace/examples/quick_start.py`
- 流式/交互示例：`/workspace/examples/streaming_mode.py`
- MCP 计算器示例：`/workspace/examples/mcp_calculator.py`
- 官方文档：https://platform.claude.com/docs/en/agent-sdk/python

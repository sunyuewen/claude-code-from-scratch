# Mini Claude Code — 项目说明文档

## 项目概述

**技术栈**：Python、Anthropic/OpenAI API、Custom Agent Runtime、AsyncIO、CLI（Rich）

面向编码场景，聚焦长程任务稳定执行的轻量级本地 Agent，重点解决工具可控调用、上下文可持续、权限安全可预期的问题。

### Agent 循环与工具系统

统一接入 Anthropic/OpenAI 兼容后端，实现流式输出与指数退避 + 抖动重试机制；实现 12 个核心工具，并通过延迟激活降低无关工具上下文开销；利用 Anthropic 后端机制，实现流式工具执行与工具并发回退，显著降低多工具执行延迟；read-before-edit 与 mtime 新鲜度校验双重保障，降低误改风险。

### CLI 与系统提示词

实现 slash commands 交互入口和会话持久化与恢复；将 system prompt 拆解为多层级注入，支持 CLAUDE.md 层级加载、@include 递归解析与规则注入，使项目级规范稳定进入上下文。

### 权限与安全

设计 5 种权限模式（default/plan/acceptEdits/bypassPermissions/dontAsk），叠加模式控制 + 声明式 allow/deny 规则 + 16 种危险命令正则检测 + 用户确认/白名单四层权限体系，平衡自动化效率与高风险操作可控性。

### 上下文与记忆

实现 4 层压缩流水线（budget truncation / stale snip / microcompact / auto-compact），在上下文利用率 50%-70% 时自动收紧工具结果体积，>85% 时触发 auto-compact，达到从轻量级截断到全量摘要逐级递进的渐进式压缩；
结合 sideQuery 语义召回、异步预取和新鲜度提醒，维持长会话上下文密度并避免过期信息误导，
实现跨会话记忆，让 Agent 在多次对话间保持对用户和项目的认知，不依赖对话历史。

> 支持 **Anthropic**（原生格式）和 **OpenAI 兼容**（OpenAI / DeepSeek / 各种中转代理）双后端，一键切换。

---

## 核心机制详解

### 1. 双后端统一接入（[agent.py](mini_claude/agent.py#L231-L243)）

同一个 `Agent` 类同时对接 Anthropic 原生 API 和 OpenAI 兼容 API（含 DeepSeek），通过构造函数中的一个布尔值 `use_openai` 切换，而非维护两套独立的 Agent 代码：

```python
# agent.py Agent.__init__
if self.use_openai:
    self._openai_client = openai.AsyncOpenAI(base_url=api_base, api_key=api_key)
    self._anthropic_client = None
else:
    self._anthropic_client = anthropic.AsyncAnthropic(api_key=api_key, base_url=anthropic_base_url)
    self._openai_client = None
```

此后所有 API 调用通过 `self.use_openai` 分流到 `_chat_openai()` 或 `_chat_anthropic()`，上层 Agent 循环、工具调度、压缩管道等逻辑完全不用关心底层是哪个 LLM 提供商。这也是 DeepSeek 用户无需修改任何核心代码、只设环境变量就能运行的原因。

### 2. 流式输出（[agent.py](python/mini_claude/agent.py#L1013-L1057) / [agent.py](python/mini_claude/agent.py#L1212-L1216)）

LLM 回复不是等全部生成完再一次性返回，而是逐 token 实时打印到终端，体验和 ChatGPT 网页版一致：

- **Anthropic 路径**：[`_call_anthropic_stream()`](python/mini_claude/agent.py#L991-L1063) — 使用 `client.messages.stream()` 创建 SSE 流，`async for event in stream` 逐事件消费。每收到 `content_block_delta` 中的 `delta.text` 就立刻调用 `self._emit_text()` 输出到终端。
- **OpenAI 路径**：[`_call_openai_stream()`](python/mini_claude/agent.py#L1210-L1280) — 在 `chat.completions.create()` 中设置 `stream=True`，`async for chunk in stream` 逐 chunk 消费 `delta.content` 并输出。

用户不用盯着空白屏幕等几十秒，系统"正在思考"的感觉通过 spinner 动画（[ui.py:107-135](python/mini_claude/ui.py#L107-L135)）和首字即时输出共同营造。

### 3. 指数退避 + 抖动重试（[agent.py](python/mini_claude/agent.py#L55-L76)）

API 调用失败时（限流 429、服务过载 503/529、网络抖动 ECONNRESET/ETIMEDOUT），自动重试且等待时间逐次翻倍，叠加随机毫秒数避免惊群效应：

```python
# agent.py
def _is_retryable(error: Exception) -> bool:
    """仅对临时性错误重试，业务错误直接抛出"""
    status = getattr(error, "status_code", None) or getattr(error, "status", None)
    if status in (429, 503, 529):   # 限流 / 过载 / 排队溢出
        return True
    if "overloaded" in msg or "ECONNRESET" in msg or "ETIMEDOUT" in msg:
        return True
    return False

async def _with_retry(fn, max_retries: int = 3):
    for attempt in range(max_retries + 1):
        try:
            return await fn()
        except Exception as error:
            if attempt >= max_retries or not _is_retryable(error):
                raise
            # delay = min(2^attempt 秒, 30秒) + 随机毫秒抖动
            delay = min(1000 * (2 ** attempt), 30000) / 1000 + (hash(...) % 1000) / 1000
            await asyncio.sleep(delay)
```

| 重试次数 | 等待时间（约） |
|----------|---------------|
| 第 1 次 | ~1 秒 + 抖动 |
| 第 2 次 | ~2 秒 + 抖动 |
| 第 3 次 | ~4 秒 + 抖动 |

3 次均失败则放弃并抛出异常。`max_retries=3` 是默认值，两个后端路径（[agent.py:1063](python/mini_claude/agent.py#L1063) / [agent.py:1280](python/mini_claude/agent.py#L1280)）均通过 `return await _with_retry(_do)` 统一包装。

### 4. 延迟工具激活（[tools.py](python/mini_claude/tools.py#L138-L144) / [tools.py](python/mini_claude/tools.py#L182-L196) / [tools.py](python/mini_claude/tools.py#L665-L678)）

**问题**：每次 LLM API 调用都要附带全部工具的 JSON Schema 定义，12 个工具占用了可观的 token 预算。但某些工具（如进入/退出计划模式）只在极少数场景下才会用到，绝大多数对话中它们白白浪费上下文空间。

**方案**：对低频工具打上 `"deferred": True` 标记，**默认不发送给 LLM**，只有当模型主动调用 `tool_search` 搜索时才激活并返回其 schema。

**12 个工具中 2 个设为延迟加载**：[tools.py:138](python/mini_claude/tools.py#L138) 和 [tools.py:144](python/mini_claude/tools.py#L144)：

```python
# tools.py — 只展示与延迟机制相关的字段
{
    "name": "enter_plan_mode",
    "description": "Enter plan mode...",
    "deferred": True,   # ← 默认不发送给 API
},
{
    "name": "exit_plan_mode",
    "description": "Exit plan mode...",
    "deferred": True,   # ← 默认不发送给 API
},
```

**过滤逻辑** — [`get_active_tool_definitions()`](python/mini_claude/tools.py#L182-L196)：

```python
_activated_tools: set[str] = set()  # 全局激活记录

def get_active_tool_definitions(all_tools=None):
    return [
        {k: v for k, v in t.items() if k != "deferred"}  # 去掉 deferred 字段再发送
        for t in tools
        if not t.get("deferred") or t["name"] in _activated_tools  # 延迟工具需已激活
    ]
```

核心逻辑在 `if` 条件：普通的 10 个工具 `deferred` 为 `None`/`False`，直接通过；标记了 `deferred: True` 的工具必须名字出现在 `_activated_tools` 集合中才会被包含。

**激活时机** — [`execute_tool()` 中的 `tool_search` 分支](python/mini_claude/tools.py#L665-L678)：

```python
if name == "tool_search":
    query = inp.get("query", "").lower()
    deferred = [t for t in tool_definitions if t.get("deferred")]
    matches = [t for t in deferred if query in t["name"].lower() or ...]
    if not matches:
        return "No matching deferred tools found."
    for m in matches:
        _activated_tools.add(m["name"])   # ← 激活匹配的延迟工具
    return json.dumps([...])               # 返回完整 schema 给模型
```

**系统提示词中的提示** — [prompt.py:222-225](python/mini_claude/prompt.py#L222-L225)：

```
The following deferred tools are available via tool_search: enter_plan_mode, exit_plan_mode.
Use tool_search to fetch their full schemas when needed.
```

**完整流程图**：

```
首轮对话
  ├─ get_active_tool_definitions() → 只返回 10 个工具（排除 2 个 deferred）
  ├─ 节省约 1.5KB tool schema / 每轮请求
  │
用户说 "切换到计划模式"
  ├─ 模型看到提示词中 "deferred tools: enter_plan_mode, exit_plan_mode"
  ├─ 模型调用 tool_search("plan")
  ├─ _activated_tools.add("enter_plan_mode")
  ├─ _activated_tools.add("exit_plan_mode")
  │
后续对话
  ├─ get_active_tool_definitions() → 返回完整 12 个工具
  ├─ 模型可以正常调用 enter_plan_mode / exit_plan_mode
```

### 5. 流式工具执行与并发回退（[agent.py:893-951](python/mini_claude/agent.py#L893-L951) / [agent.py:1168-1206](python/mini_claude/agent.py#L1168-L1206)）

**问题**：传统流程是 LLM 先完整生成所有 tool_use → 再逐个串行执行 → 结果返回后才能继续。假设模型生成了 3 个 `read_file` 调用，工具执行阶段就是 `read1(1s) → read2(1s) → read3(1s) = 3s`。

**方案**：利用 Anthropic SSE 协议的 `content_block_stop` 事件 + OpenAI 路径的工具分组，让无依赖的工具尽早启动或并行执行。

#### Anthropic 路径：流式工具执行

核心原理 — 不等整个 LLM 响应结束，每个 tool_use 块的 JSON 参数一接收完就立刻启动执行。LLM 还在生成后续内容时，前面的工具已经在跑了。

**第 1 步**：定义哪些工具可以提前执行 — [tools.py:26](python/mini_claude/tools.py#L26)

```python
CONCURRENCY_SAFE_TOOLS = {"read_file", "list_files", "grep_search", "web_fetch"}
```

这四个只读工具无副作用，不需要用户确认，可以安全地提前执行。

**第 2 步**：流式回调，参数收齐即启动 — [agent.py:899-905](python/mini_claude/agent.py#L899-L905)

```python
# _chat_anthropic() 中定义的 _on_tool_block 回调
def _on_tool_block(block: dict):
    if block["name"] in CONCURRENCY_SAFE_TOOLS:
        perm = check_permission(block["name"], block["input"],
                                self.permission_mode, self._plan_file_path)
        if perm["action"] == "allow":
            task = asyncio.create_task(
                self._execute_tool_call(block["name"], block["input"]))
            early_executions[block["id"]] = task  # 记录已启动的任务
```

**第 3 步**：SSE 事件驱动 — [agent.py:1018-1055](python/mini_claude/agent.py#L1018-L1055)

```python
# _call_anthropic_stream() — SSE 事件处理
async for event in stream:
    if event.type == "content_block_start":
        # tool_use 块开始，记录 tool id + name
        tool_blocks_by_index[event.index] = {
            "id": cb.id, "name": cb.name, "input_json": ""}

    elif event.type == "content_block_delta" and delta.partial_json:
        # 逐片拼接 JSON 参数
        tool_blocks_by_index[event.index]["input_json"] += delta.partial_json

    elif event.type == "content_block_stop":
        # ★ JSON 完整了！解析并回调 → 即刻启动工具
        parsed = json.loads(tool_blocks_by_index[event.index]["input_json"])
        on_tool_block_complete({"name": tb["name"], "input": parsed, ...})
```

**第 4 步**：流结束后的处理 — [agent.py:916-951](python/mini_claude/agent.py#L916-L951)

```python
# LLM 响应完全结束，处理所有 tool_use
for tu in tool_uses:
    early_task = early_executions.get(tu.id)
    if early_task:
        raw = await early_task      # ← 工具可能已经执行完了，直接拿结果
        tool_results.append({"type": "tool_result", ...})
        continue                     # ← 跳过权限检查和重新执行
    # 其他工具走正常的 权限检查 → 执行 流程
```

#### OpenAI 路径：工具并发分组

OpenAI API 没有 SSE content_block_stop 事件，但可以在收到完整响应后对工具分组并行执行 — [agent.py:1168-1206](python/mini_claude/agent.py#L1168-L1206)：

```python
# Phase 2: Group & execute
for ct in oai_checked:
    safe = ct["allowed"] and ct["fn"] in CONCURRENCY_SAFE_TOOLS
    if safe and oai_batches and oai_batches[-1]["concurrent"]:
        oai_batches[-1]["items"].append(ct)   # 加入当前并发组
    else:
        oai_batches.append({"concurrent": safe, "items": [ct]})  # 新组

for batch in oai_batches:
    if batch["concurrent"]:
        # asyncio.gather — 组内所有只读工具同时执行
        results = await asyncio.gather(
            *[_run_oai_safe(ct) for ct in batch["items"]])
    else:
        # 串行执行（需要确认或有副作用的工具）
        for ct in batch["items"]:
            raw = await self._execute_tool_call(ct["fn"], ct["inp"])
```

**效果对比**：

```
传统串行（两个后端通用问题，这里以 Anthropic 路径为例说明优化效果）:
  生成 tool_use₁ → 等 → 生成 tool_use₂ → 执行 read₁(1s) → 执行 read₂(1s)
  LLM 生成耗时 3s + 工具执行耗时 2s = 总计 ~5s

Anthropic 流式执行后:
  生成 tool_use₁ → 即刻启动 read₁ ────────────┐
  继续生成 tool_use₂ → 即刻启动 read₂ ────────┤ LLM 生成和工具执行重叠
  总耗时 ≈ max(生成 3s, 执行 1s) = ~3s

OpenAI 并发分组后:
  完整响应返回 → read₁(1s) ┐
                  read₂(1s) ├ 并行 = 1s（而非 2s）
                  read₃(1s) ┘
  总耗时 ≈ 生成 2s + max(执行) 1s = ~3s
```

> **注意**：Anthropic 路径的流式工具执行是真正的"LLM 生成与工具执行重叠"，OpenAI 路径的并发分组是在响应完成后对多工具做并行。两者互补，覆盖了两个后端各自的最优路径。

### 6. read-before-edit 与 mtime 新鲜度校验（[tools.py:644-662](python/mini_claude/tools.py#L644-L662) / [tools.py:695-699](python/mini_claude/tools.py#L695-L699)）

**问题**：LLM 可能"凭空想象"一个文件内容并尝试修改它，或者在两次读取之间文件被外部工具（如 git checkout、IDE 自动保存）修改，此时 LLM 基于的上下文已经过时，编辑会导致数据丢失。

**方案**：两重校验串联，在 `execute_tool()` 中构成安全门禁。

#### 第一重：read-before-edit — 必须读过才能改

**状态追踪** — Agent 维护 `_read_file_state: dict[str, float]`，key 是文件绝对路径，value 是读取时的 mtime：

```python
# agent.py Agent.__init__
self._read_file_state: dict[str, float] = {}
```

**读取时记录** — [tools.py:644-651](python/mini_claude/tools.py#L644-L651)：

```python
async def execute_tool(name, inp, read_file_state=None):
    if name == "read_file":
        result = _read_file(inp)
        if read_file_state is not None and not result.startswith("Error"):
            abs_path = str(Path(inp["file_path"]).resolve())
            try:
                read_file_state[abs_path] = os.path.getmtime(abs_path)
                #                    ^^^^^^^^^ 记录文件当前修改时间
            except OSError:
                pass
        return _truncate_result(result)
```

**写入时校验** — [tools.py:654-662](python/mini_claude/tools.py#L654-L662)：

```python
    if name in ("write_file", "edit_file") and read_file_state is not None:
        abs_path = str(Path(inp["file_path"]).resolve())
        if os.path.exists(abs_path):
            if abs_path not in read_file_state:
                # ❌ 从未读过这个文件
                return f"Error: You must read this file before writing. Use read_file first."
```

如果是已存在的文件且从未被 `read_file` 记录过，直接拒绝。这防止了 LLM 对从未见过的文件做盲目修改。

#### 第二重：mtime 新鲜度 — 外部修改后必须重读

继续同一段代码 — [tools.py:660-662](python/mini_claude/tools.py#L660-L662)：

```python
            if os.path.getmtime(abs_path) != read_file_state[abs_path]:
                # ⚠️ 读之后文件被外部改过了
                return (
                    f"Warning: {inp['file_path']} was modified externally since "
                    f"your last read. Please read_file again before writing."
                )
```

如果文件当前的 mtime 和读取时记录的不一致，说明两次操作之间文件被外部改动过（git checkout、IDE 格式化、其他进程写入等），此时 LLM 看到的内容已经过时，强制要求重新读取。

#### 写入成功后更新 mtime

修改成功后更新缓存，让 Agent 对同一文件做连续编辑时不会因为自己刚改的内容而被 mtime 检查拦截 — [tools.py:695-699](python/mini_claude/tools.py#L695-L699)：

```python
    if name in ("write_file", "edit_file") and read_file_state is not None \
       and not result.startswith("Error"):
        abs_path = str(Path(inp["file_path"]).resolve())
        try:
            read_file_state[abs_path] = os.path.getmtime(abs_path)
            #          ^^^^^^^^^ 刷新为修改后的时间戳
        except OSError:
            pass
```

#### 完整流程

```
Agent 调用 write_file("src/app.py", content)
  │
  ▼
execute_tool("write_file", ...)
  ├─ 文件存在且不在 read_file_state 中？
  │   → "Error: You must read this file before writing."      ← 第一重
  │
  ├─ 文件 mtime ≠ read_file_state 记录的值？
  │   → "Warning: modified externally. Please read_file again." ← 第二重
  │
  └─ 两重检查都通过
      → 执行写入 → 成功后更新 read_file_state 的 mtime
```

#### 为什么需要两重而不是一重

| 场景 | read-before-edit 拦截 | mtime 新鲜度拦截 |
|------|----------------------|-------------------|
| LLM 幻觉出一个不存在的文件直接写 | ❌（文件不存在，不触发） | ❌ |
| LLM 未读取就修改已存在的文件 | ✅ 拦截 | — |
| LLM 读完后、写入前，文件被 `git checkout` 覆盖 | — | ✅ 拦截 |
| LLM 读完后立即写入（正常流程） | ✅ 放行 | ✅ 放行 |

两者覆盖的攻击面互补：read-before-edit 防的是"没读就改"，mtime 新鲜度防的是"读了过期内容还改"。新文件创建（write_file with non-existent path）不触发这两重检查，只有对已存在文件的操作才受限。

### 7. Slash Commands 与会话持久化（[__main__.py:127-160](python/mini_claude/__main__.py#L127-L160) / [agent.py:416-430](python/mini_claude/agent.py#L416-L430) / [session.py](python/mini_claude/session.py)）

**问题**：纯命令行工具每次执行后就退出，用户需要反复输入完整的 prompt。同时对话上下文（包括 System Prompt、用户消息、LLM 回复、工具调用结果）在进程退出后全部丢失，无法跨运行保留项目进展。

**方案**：内置 6 个 REPL 命令 + 技能斜杠调用 + 自动会话归档与恢复。

#### 内置 Slash Commands — [__main__.py:127-160](python/mini_claude/__main__.py#L127-L160)

REPL 循环的主函数 [`run_repl()`](python/mini_claude/__main__.py#L51-L188) 在接收用户输入后先做命令匹配，未命中才作为普通 prompt 发给 Agent：

```python
# __main__.py run_repl() 主循环
while True:
    line = input()           # 读取一行
    inp = line.strip()

    if inp in ("exit", "quit"):
        break                # 退出

    if inp == "/clear":      # 清空对话历史
        agent.clear_history()
        continue
    if inp == "/plan":        # 切换计划模式
        agent.toggle_plan_mode()
        continue
    if inp == "/cost":        # 显示 token 用量和费用
        agent.show_cost()
        continue
    if inp == "/compact":     # 手动压缩对话
        await agent.compact()
        continue
    if inp == "/memory":      # 列出所有记忆
        memories = list_memories()
        ...
        continue
    if inp == "/skills":      # 列出所有技能
        skills = discover_skills()
        ...
        continue

    # 没有匹配任何命令 → 作为普通 prompt 发给 Agent
    await agent.chat(inp)
```

6 个命令都是**本地即时处理**，不消耗 API 调用。底层对应的方法都在 `Agent` 类中实现，和 `/` 命令之间只隔一层 dispatch。

#### 技能斜杠调用 — [__main__.py:162-181](python/mini_claude/__main__.py#L162-L181)

以 `/` 开头但不是内置命令的输入，会尝试匹配 user-invocable 技能：

```python
if inp.startswith("/"):
    space_idx = inp.find(" ")
    cmd_name = inp[1:space_idx] if space_idx > 0 else inp[1:]
    cmd_args = inp[space_idx + 1:] if space_idx > 0 else ""
    skill = get_skill_by_name(cmd_name)
    if skill and skill.user_invocable:
        if skill.context == "fork":
            # fork 技能：新开隔离 Agent 执行
            await agent.chat(f'Use the skill tool to invoke "{skill.name}" with args: {cmd_args}')
        else:
            # inline 技能：直接注入 prompt
            resolved = resolve_skill_prompt(skill, cmd_args)
            await agent.chat(resolved)
        continue
```

`/<skill> [args]` 的语法和 Git 的 `/commit` 等用法一致。

#### 会话自动保存 — [agent.py:416-430](python/mini_claude/agent.py#L416-L430)

每次 `chat()` 结束时自动调用 `_auto_save()`，将对话历史写入 `~/.mini-claude/sessions/<id>.json`：

```python
def _auto_save(self) -> None:
    try:
        save_session(self.session_id, {
            "metadata": {
                "id": self.session_id,
                "model": self.model,
                "cwd": str(Path.cwd()),
                "startTime": self.session_start_time,
                "messageCount": self._get_message_count(),
            },
            "anthropicMessages": self._anthropic_messages if not self.use_openai else None,
            "openaiMessages": self._openai_messages if self.use_openai else None,
        })
    except Exception:
        pass  # 保存失败不影响主流程
```

[agent.py:344-346](python/mini_claude/agent.py#L344-L346) 中非子 Agent 的每次 `chat()` 末尾都会触发：

```python
if not self.is_sub_agent:
    print_divider()
    self._auto_save()   # ← 自动归档
```

#### 会话恢复 — [agent.py:406-411](python/mini_claude/agent.py#L406-L411) + [__main__.py:279-291](python/mini_claude/__main__.py#L279-L291)

**存储层** — [session.py](python/mini_claude/session.py)（50 行，纯 JSON 文件 CRUD）：

```python
SESSION_DIR = Path.home() / ".mini-claude" / "sessions"

def save_session(session_id, data):    # 写 JSON
def load_session(session_id):          # 读 JSON
def list_sessions():                   # 列出所有会话元数据
def get_latest_session_id():           # 取最近一个会话 ID
```

**恢复入口** — [__main__.py:279-291](python/mini_claude/__main__.py#L279-L291)：

```python
if args.resume:
    session_id = get_latest_session_id()
    if session_id:
        session = load_session(session_id)
        if session:
            agent.restore_session({
                "anthropicMessages": session.get("anthropicMessages"),
                "openaiMessages": session.get("openaiMessages"),
            })
```

**Agent 侧** — [agent.py:406-411](python/mini_claude/agent.py#L406-L411)：

```python
def restore_session(self, data: dict) -> None:
    if data.get("anthropicMessages"):
        self._anthropic_messages = data["anthropicMessages"]
    if data.get("openaiMessages"):
        self._openai_messages = data["openaiMessages"]
```

#### 使用方式

```bash
mini-claude-py                       # 新会话
# ... 工作了 2 小时 ...
exit

mini-claude-py --resume              # 继续上次的上下文
```

`--resume` 不需要任何额外参数 — 自动找最新会话、加载完整消息历史、继续之前的对话状态。

### 8. 系统提示词多层级注入（[prompt.py](mini_claude/prompt.py)）

**问题**：Agent 需要在任意项目目录下工作，必须知道当前项目规范、文件结构、用户偏好。这些信息分散在不同文件（根目录 `CLAUDE.md`、子目录 `CLAUDE.md`、`.claude/rules/*.md`），随时可能变化。每次都让用户手动告知低效且不可靠。

**方案**：将 System Prompt 由"写死的字符串"升级为"模板 + 多来源动态拼接"，每次对话启动时实时收集上下文并注入。分四个层次。

#### 第 1 层：模板骨架 — [prompt.py:18-99](mini_claude/prompt.py#L18-L99)

`SYSTEM_PROMPT_TEMPLATE` 定义了 Agent 行为规范，在 9 个位置埋入 `{{变量名}}` 占位符：

```python
SYSTEM_PROMPT_TEMPLATE = """...
# Environment
Working directory: {{cwd}}
Date: {{date}}
Platform: {{platform}}
Shell: {{shell}}
{{git_context}}
{{claude_md}}       ← CLAUDE.md 内容注入点
{{memory}}          ← 记忆系统注入点
{{skills}}          ← 技能列表注入点
{{agents}}          ← 子 Agent 类型注入点
{{deferred_tools}}  ← 延迟工具提示注入点
"""
```

#### 第 2 层：变量插值 — [prompt.py:210-243](mini_claude/prompt.py#L210-L243)

`build_system_prompt()` 收集所有动态信息，做字符串替换：

```python
def build_system_prompt() -> str:
    today = date.today().isoformat()                     # → {{date}}
    plat = f"{platform.system()} {platform.machine()}"   # → {{platform}}
    shell = os.environ.get("ComSpec") or "cmd.exe"       # → {{shell}}
    git_context = get_git_context()                      # → {{git_context}}
    claude_md = load_claude_md()                         # → {{claude_md}}
    memory_section = build_memory_prompt_section()       # → {{memory}}
    skills_section = build_skill_descriptions()          # → {{skills}}
    agent_section = build_agent_descriptions()           # → {{agents}}
    deferred_names = get_deferred_tool_names()           # → {{deferred_tools}}

    result = SYSTEM_PROMPT_TEMPLATE
    for key, value in replacements.items():
        result = result.replace(key, value)              # 逐键替换
    return result
```

#### 第 3 层：CLAUDE.md 层级加载 — [prompt.py:168-190](mini_claude/prompt.py#L168-L190)

`load_claude_md()` 从当前目录向上遍历至根目录，收集沿途所有 `CLAUDE.md`，子目录的内容排在父目录前面（更具体的规范覆盖更宽泛的规范）：

```python
def load_claude_md() -> str:
    parts: list[str] = []
    d = Path.cwd().resolve()
    while True:
        f = d / "CLAUDE.md"
        if f.is_file():
            content = f.read_text()
            content = _resolve_includes(content, d)  # ← 处理 @include（第 4 层）
            parts.insert(0, content)                 # 子目录在前
        parent = d.parent
        if parent == d:
            break
        d = parent
    claude_md = "\n\n# Project Instructions\n" + "\n\n---\n\n".join(parts)
    rules = _load_rules_dir(Path.cwd())              # 追加 rules
    return claude_md + rules
```

遍历示例：`d:\project\sub\` → 收集 `d:\CLAUDE.md` → `d:\project\CLAUDE.md` → `d:\project\sub\CLAUDE.md`（如有）。

#### 第 4 层：@include 递归解析 — [prompt.py:107-143](mini_claude/prompt.py#L107-L143)

每个 `CLAUDE.md` 在被合并前，`_resolve_includes()` 会扫描 `@./path`、`@~/path`、`@/path` 引用并递归展开：

```python
_INCLUDE_RE = re.compile(r"^@(\./[^\s]+|~/[^\s]+|/[^\s]+)$", re.MULTILINE)
_MAX_INCLUDE_DEPTH = 5  # 最多 5 层嵌套

def _resolve_includes(content, base_path, visited=None, depth=0):
    if depth >= _MAX_INCLUDE_DEPTH:
        return content

    def _replace(match):
        raw = match.group(1)
        if raw.startswith("~/"):
            resolved = Path.home() / raw[2:]   # @~/foo/bar.md
        elif raw.startswith("/"):
            resolved = Path(raw)               # @/absolute/path.md
        else:
            resolved = base_path / raw         # @./relative/path.md
        key = str(resolved.resolve())
        if key in visited:
            return f"<!-- circular: {raw} -->"  # 循环引用保护
        included = resolved.read_text()
        return _resolve_includes(included, resolved.parent, visited, depth + 1)
    return _INCLUDE_RE.sub(_replace, content)
```

#### 规则注入 — [prompt.py:146-165](mini_claude/prompt.py#L146-L165)

`_load_rules_dir()` 扫描 `.claude/rules/*.md`，追加到 System Prompt 末尾，rules 内部也支持 `@include`：

```python
def _load_rules_dir(directory: Path) -> str:
    rules_dir = directory / ".claude" / "rules"
    if not rules_dir.is_dir():
        return ""
    files = sorted(f for f in rules_dir.iterdir() if f.suffix == ".md")
    parts = []
    for f in files:
        content = f.read_text()
        content = _resolve_includes(content, rules_dir)
        parts.append(f"<!-- rule: {f.name} -->\n{content}")
    return "\n\n## Rules\n" + "\n\n".join(parts) if parts else ""
```

#### 完整注入流程

```
项目目录: d:\project\sub\
文件:      d:\CLAUDE.md                               ← 全局规范
           d:\project\CLAUDE.md                        ← 项目规范 (@./.claude/rules/format.md)
           d:\project\.claude\rules\format.md           ← 格式规范
           d:\project\.claude\rules\security.md         ← 安全规范

build_system_prompt() 执行:
  ├─ {{date}} → "2026-06-12"
  ├─ {{platform}} → "Windows 10 AMD64"
  ├─ {{git_context}} → "branch: main / 5 commits..."
  │
  ├─ load_claude_md():
  │   1. d:\CLAUDE.md → 读入
  │   2. d:\project\CLAUDE.md → @include 展开 format.md
  │   3. _load_rules_dir() → security.md 追加
  │   → 合并为完整 claude_md 字符串
  │
  ├─ {{memory}} → build_memory_prompt_section()
  ├─ {{skills}} → build_skill_descriptions()
  ├─ {{agents}} → build_agent_descriptions()
  ├─ {{deferred_tools}} → "tools available via tool_search"
  │
  └─ 全部 9 个占位符替换完毕 → 最终 System Prompt
```

#### 设计收益

- 换项目目录，`CLAUDE.md` 和 `rules/*.md` 自动切换，无需改配置
- 修改规范文件后下次启动立即生效，无需改代码或重新部署
- `@include` 允许规范拆分为独立小文件（`format.md`、`security.md`、`testing.md`），各自维护
- 最大 5 层递归 + 循环引用检测，防止死循环

### 9. 四层权限体系（[tools.py](python/mini_claude/tools.py#L462-L617)）

**问题**：Agent 拥有文件读写和 Shell 执行能力，自动化越高风险越大。一刀切"全部确认"打断工作流，"全部放行"可能造成不可逆损失。

**方案**：四层递进式权限判断，从粗粒度到细粒度逐层过滤。

#### 第 1 层：5 种权限模式 — [tools.py:572-617](python/mini_claude/tools.py#L572-L617)

`check_permission(tool_name, inp, mode)` 根据当前模式做粗粒度判断。5 种模式的行为对比：

| 模式 | 读操作 | 写操作 | Shell | CLI 参数 |
|------|--------|--------|-------|----------|
| `default` | ✅ | ❓确认 | ❓危险确认 | (无) |
| `plan` | ✅ | ❌ 仅 plan 文件 | ❌ | `--plan` |
| `acceptEdits` | ✅ | ✅ | ❓危险确认 | `--accept-edits` |
| `bypassPermissions` | ✅ | ✅ | ✅ | `--yolo` |
| `dontAsk` | ✅ | ❌ | ❌ 危险拒绝 | `--dont-ask` |

代码入口判断优先级：

```python
def check_permission(tool_name, inp, mode="default", plan_file_path=None):
    # 模式 5: bypass — 完全放行
    if mode == "bypassPermissions":
        return {"action": "allow"}

    # → 第 2 层 allow/deny 规则插入点

    # 所有只读工具一律放行
    if tool_name in READ_TOOLS:
        return {"action": "allow"}

    # 模式 1: plan — 只允许读写 plan 文件
    if mode == "plan":
        if tool_name in EDIT_TOOLS:
            if file_path == plan_file_path:
                return {"action": "allow"}
            return {"action": "deny", ...}
        if tool_name == "run_shell":
            return {"action": "deny", ...}

    # 模式 4: acceptEdits — 自动接受写操作
    if mode == "acceptEdits" and tool_name in EDIT_TOOLS:
        return {"action": "allow"}

    # → 第 3 层 / 第 4 层插入点
```

#### 第 2 层：声明式 allow/deny 规则 — [tools.py:510-562](python/mini_claude/tools.py#L510-L562)

从 `.claude/settings.json` 和 `~/.claude/settings.json` 加载用户自定义白名单/黑名单：

```python
def load_permission_rules():
    user_settings = _load_settings(Path.home() / ".claude" / "settings.json")
    project_settings = _load_settings(Path.cwd() / ".claude" / "settings.json")
    for settings in [user_settings, project_settings]:
        for r in perms.get("allow", []):
            allow.append(_parse_rule(r))
        for r in perms.get("deny", []):
            deny.append(_parse_rule(r))
```

配置示例：

```json
{
  "permissions": {
    "allow": ["run_shell(npm test)", "write_file(./README.md)"],
    "deny": ["run_shell(rm *)"]
  }
}
```

规则匹配支持精确匹配和前缀通配符 — [tools.py:534-551](python/mini_claude/tools.py#L534-L551)：

```python
def _matches_rule(rule, tool_name, inp):
    if rule["pattern"] is None:
        return True                    # "read_file" → 匹配所有 read_file
    if pattern.endswith("*"):
        return value.startswith(pattern[:-1])  # "git *" → 匹配所有 git 命令
    return value == pattern                     # 精确匹配
```

`_check_permission_rules()` 中 deny 先于 allow 判断，用户级和项目级规则合并，结果缓存避免每次重复读盘。

#### 第 3 层：16 种危险命令正则检测 — [tools.py:464-481](python/mini_claude/tools.py#L464-L481)

覆盖 Linux/Git Bash 和 Windows（cmd/PowerShell）的危险命令：

```python
DANGEROUS_PATTERNS = [
    re.compile(r"\brm\s"),                     # rm 删除
    re.compile(r"\bgit\s+(push|reset|clean|checkout\s+\.)"),  # 破坏性 git
    re.compile(r"\bsudo\b"),                   # 提权
    re.compile(r"\bkill\b"),                   # 杀进程 (Linux)
    re.compile(r"\bpkill\b"),                  # 按名杀进程
    re.compile(r"\breboot\b"),                 # 重启
    re.compile(r"\bshutdown\b"),               # 关机
    re.compile(r"\bdel\s", re.IGNORECASE),     # 删除 (Windows cmd)
    re.compile(r"\brmdir\s", re.IGNORECASE),   # 删目录 (Windows)
    re.compile(r"\bformat\s", re.IGNORECASE),  # 格式化磁盘 (Windows)
    re.compile(r"\btaskkill\s", re.IGNORECASE),    # 终止进程 (Windows)
    re.compile(r"\bRemove-Item\s", re.IGNORECASE), # PowerShell 删除
    re.compile(r"\bStop-Process\s", re.IGNORECASE),# PowerShell 杀进程
    re.compile(r"\bmkfs\b"),                   # 格式化文件系统
    re.compile(r"\bdd\s"),                     # 磁盘镜像写入
    re.compile(r">\s*/dev/"),                  # 重定向到设备
]

def is_dangerous(command: str) -> bool:
    return any(p.search(command) for p in DANGEROUS_PATTERNS)
```

#### 第 4 层：用户确认 / 白名单 — [tools.py:599-617](python/mini_claude/tools.py#L599-L617)

前三层都未拦截的操作进入最终判断：

```python
needs_confirm = False

if tool_name == "run_shell" and is_dangerous(command):
    needs_confirm = True        # 危险命令 → 弹确认
elif tool_name == "write_file" and not file_exists:
    needs_confirm = True        # 新建文件 → 弹确认
elif tool_name == "edit_file" and not file_exists:
    needs_confirm = True        # 编辑不存在的文件 → 弹确认

if needs_confirm:
    if mode == "dontAsk":
        return {"action": "deny", ...}   # CI 模式自动拒绝
    return {"action": "confirm", ...}    # 弹出 y/N 确认
return {"action": "allow"}
```

#### 完整决策流示例

```
check_permission("run_shell", "rm -rf node_modules", mode="default")
  第 1 层: mode == "default" → 继续
  第 2 层: settings 无匹配 → 继续
  第 3 层: \brm\s 命中 → is_dangerous = True
  第 4 层: needs_confirm = True → {"action": "confirm", ...}
  → 用户看到: "Run: rm -rf node_modules? [y/N]"

check_permission("read_file", "src/app.ts", mode="default")
  第 1 层: READ_TOOLS → {"action": "allow"}
  → 直接执行，零确认
```

#### 四层分工总结

| 层 | 机制 | 职责 | 代码行 |
|----|------|------|--------|
| 1 | 5 种权限模式 | 整体安全级别选择 | [tools.py:572-617](python/mini_claude/tools.py#L572-L617) |
| 2 | allow/deny 规则 | 用户自定义白名单黑名单 | [tools.py:510-562](python/mini_claude/tools.py#L510-L562) |
| 3 | 16 种危险正则 | 自动检测高风险命令 | [tools.py:464-481](python/mini_claude/tools.py#L464-L481) |
| 4 | 用户确认/白名单 | 最终人工把关 | [tools.py:599-617](python/mini_claude/tools.py#L599-L617) |

### 10. 四层压缩流水线（[agent.py:146-150](python/mini_claude/agent.py#L146-L150) / [agent.py:508-620](python/mini_claude/agent.py#L508-L620) / [agent.py:441-504](python/mini_claude/agent.py#L441-L504)）

**问题**：长对话中 LLM 的每次响应 + 工具调用结果都在累积到消息历史里，上下文窗口（如 128K tokens）迟早被填满。简单粗暴地"满了就删除旧消息"会丢失关键上下文，而"每次都让 LLM 摘要"又浪费 API 费用。

**方案**：4 层渐进式压缩，从轻量级字节截断到全量 LLM 语义摘要逐级递进，只在必要时才触发昂贵的 API 调用。

#### 配置常量 — [agent.py:146-150](python/mini_claude/agent.py#L146-L150)

```python
SNIPPABLE_TOOLS = {"read_file", "grep_search", "list_files", "run_shell"}
SNIP_PLACEHOLDER = "[Content snipped - re-read if needed]"
SNIP_THRESHOLD = 0.60           # 利用率 > 60% 触发 snip
MICROCOMPACT_IDLE_S = 5 * 60   # 空闲 5 分钟触发 microcompact
KEEP_RECENT_RESULTS = 3         # 保留最近 3 条工具结果
```

#### 调度入口 — [agent.py:508-516](python/mini_claude/agent.py#L508-L516)

每次 API 调用前执行前三级压缩（不涉及 API），第 4 级在 `_check_and_compact()` 中按需触发：

```python
def _run_compression_pipeline(self) -> None:
    if self.use_openai:
        self._budget_tool_results_openai()   # Tier 1
        self._snip_stale_results_openai()    # Tier 2
        self._microcompact_openai()          # Tier 3
    else:
        self._budget_tool_results_anthropic()
        self._snip_stale_results_anthropic()
        self._microcompact_anthropic()
```

两个后端各有一套实现，逻辑镜像，只是消息结构不同（Anthropic 嵌套 content block / OpenAI 扁平 role）。

#### Tier 1：Budget Truncation（利用率 > 50%） — [agent.py:518-540](python/mini_claude/agent.py#L518-L540)

**策略**：截断过长的单个工具结果，保留头尾，中间加占位符。截断不经过 LLM，纯字符串操作，零 API 开销。

```python
def _budget_tool_results_anthropic(self) -> None:
    utilization = self.last_input_token_count / self.effective_window
    if utilization < 0.5:
        return                               # 还没到阈值，跳过
    budget = 15000 if utilization > 0.7 else 30000  # 紧张:15KB / 正常:30KB
    for msg in self._anthropic_messages:
        for block in msg.get("content", []):
            if isinstance(block, dict) and block.get("type") == "tool_result":
                if len(block["content"]) > budget:
                    keep = (budget - 80) // 2
                    block["content"] = (
                        block["content"][:keep]           # 保留头部
                        + f"\n\n[... budgeted: {len} chars truncated ...]\n\n"
                        + block["content"][-keep:]        # 保留尾部
                    )
```

两档预算：利用率 50%-70% 截断到 30KB，>70% 收紧到 15KB。保留头尾的方式确保关键信息（文件路径、错误信息头、结果末尾）不丢。

#### Tier 2：Stale Snip（利用率 > 60%） — [agent.py:542-593](python/mini_claude/agent.py#L542-L593)

**策略**：对可重新读取的工具结果做轻量级标记替换。特别是同一文件被多次 `read_file` 时，只保留最近一条，旧的全部 snipped。

```python
def _snip_stale_results_anthropic(self) -> None:
    utilization = self.last_input_token_count / self.effective_window
    if utilization < SNIP_THRESHOLD:          # 0.60
        return

    # 收集所有可 snippet 的工具结果
    for mi, msg in enumerate(self._anthropic_messages):
        for bi, block in enumerate(msg["content"]):
            if block["type"] == "tool_result" and tool in SNIPPABLE_TOOLS:
                results.append({"mi": mi, "bi": bi, "name": ..., "file_path": ...})

    if len(results) <= KEEP_RECENT_RESULTS:   # 3
        return                                 # 不够多，无需 snip

    # 去重：同一文件多次读取 → 保留最后一次
    seen_files = {}
    for i, r in enumerate(results):
        if r["name"] == "read_file" and r.get("file_path"):
            seen_files.setdefault(r["file_path"], []).append(i)
    for indices in seen_files.values():
        if len(indices) > 1:
            for j in indices[:-1]:             # 除最后一个全标记 snipped
                to_snip.add(j)

    # 保留最近 3 条
    for i in range(len(results) - KEEP_RECENT_RESULTS):
        to_snip.add(i)

    for idx in to_snip:
        msg["content"][...]["content"] = SNIP_PLACEHOLDER
        # "[Content snipped - re-read if needed]"
```

替换成占位符而非删除 — 保留消息结构完整性（tool_use 和 tool_result 需要一一对应），但将内容削减为一条提示字符串。模型看到占位符后如果需要原内容，可以再次调用 `read_file`。

#### Tier 3：Microcompact（空闲 > 5 分钟） — [agent.py:595-620](python/mini_claude/agent.py#L595-L620)

**策略**：仅在距离上次 API 调用超过 5 分钟时触发，清除最旧的工具结果，保留最近 3 条。

```python
def _microcompact_anthropic(self) -> None:
    if self.last_api_call_time:
        if (time.time() - self.last_api_call_time) < MICROCOMPACT_IDLE_S:  # 300s
            return                                # 用户还在活跃对话，不清理

    # 收集所有活跃工具结果
    for mi, msg in enumerate(self._anthropic_messages):
        for bi, block in enumerate(msg["content"]):
            if block["type"] == "tool_result" and content not in (PLACEHOLDER, "[Old result cleared]"):
                all_results.append((mi, bi))

    clear_count = len(all_results) - KEEP_RECENT_RESULTS
    for i in range(max(0, clear_count)):
        self._anthropic_messages[mi]["content"][bi]["content"] = "[Old result cleared]"
```

关键设计：不加利用率门槛，只加时间门槛。理由是空闲 5 分钟说明用户可能切去浏览器查资料、开会、或思考下一步，此时清理旧结果为下一轮对话腾空间，又不会干扰当前工作流。

#### Tier 4：Auto-Compact（利用率 > 85%） — [agent.py:441-504](python/mini_claude/agent.py#L441-L504)

**策略**：前三层都无法压住利用率时，触发 LLM 语义摘要 — 唯一有 API 开销的一层。

```python
async def _check_and_compact(self) -> None:
    if self.last_input_token_count > self.effective_window * 0.85:
        print_info("Context window filling up, compacting conversation...")
        await self._compact_conversation()

async def _compact_anthropic(self) -> None:
    # 完整对话历史 → LLM 摘要 → 3 条消息重置
    summary_resp = await self._anthropic_client.messages.create(
        model=self.model,
        max_tokens=2048,
        system="You are a conversation summarizer...",
        messages=[
            *self._anthropic_messages[:-1],
            {"role": "user", "content": "Summarize the conversation so far..."},
        ],
    )
    summary_text = summary_resp.content[0].text
    # 重置为 3 条消息的骨架
    self._anthropic_messages = [
        {"role": "user", "content": f"[Previous conversation summary]\n{summary_text}"},
        {"role": "assistant", "content": "Understood..."},
    ]
    if last_user_msg.get("role") == "user":
        self._anthropic_messages.append(last_user_msg)  # 保留最新用户消息
```

OpenAI 路径 `_compact_openai()` 逻辑镜像，但保留系统消息、用 OpenAI 消息格式。

#### 完整触发条件与开销

| 层 | 触发条件 | 操作 | API 开销 | 空间回收 |
|----|----------|------|----------|----------|
| Budget | 利用率 > 50% | 截断过长结果保留头尾 | **零** | 中等 |
| Snip | 利用率 > 60% | 去重 + 占位符替换旧结果 | **零** | 较大 |
| Microcompact | 空闲 > 5 分钟 | 清除最旧结果保留最近 3 条 | **零** | 小 |
| Auto-Compact | 利用率 > 85% | LLM 摘要整段对话 | **一次 API 调用** | 极大 |

#### 设计精髓

前三层是"不花钱的压缩"：字符串截断、去重替换、空闲清理。只有在前三层都挡不住、利用率冲到 85% 时才调用 LLM 做语义摘要。这种**从机械裁剪到语义理解的渐进式策略**，比"满了就摘要"的简单方案在长会话中能节省数十次不必要的摘要 API 调用，同时保持了上下文的高密度。

### 11. sideQuery 语义召回、异步预取与新鲜度提醒（[memory.py](python/mini_claude/memory.py) + [agent.py:868-904](python/mini_claude/agent.py#L868-L904)）

**问题**：跨会话记忆的核心矛盾 — 记忆文件可能积攒几十条，全部注入浪费 token；简单关键词匹配又找不准真正相关的。此外旧记忆是过去的快照，可能已过时。

**方案**：sideQuery 语义选择 → 精准召回 ≤5 条 → 异步预取不阻塞主流程 → 新鲜度标签防误用。

#### 第 1 步：sideQuery 语义召回 — [memory.py:235-286](python/mini_claude/memory.py#L235-L286)

用轻量级 LLM 理解当前 query 语义，从记忆清单选出真正相关的：

```python
SELECT_MEMORIES_PROMPT = """You are selecting memories that will be useful...
Return a JSON object with a "selected_memories" array of filenames (up to 5).
- If you are unsure, do not include it.
- If no memories would clearly be useful, return an empty array."""

async def select_relevant_memories(query, side_query, already_surfaced):
    headers = scan_memory_headers()               # 只读 frontmatter（前 30 行）
    candidates = [h for h in headers if h.file_path not in already_surfaced]

    manifest = format_memory_manifest(candidates)  # 一行一条简述 → 极省 token
    text = await side_query(
        SELECT_MEMORIES_PROMPT,
        f"Query: {query}\n\nAvailable memories:\n{manifest}",
    )
    parsed = json.loads(re.search(r"\{[\s\S]*\}", text).group(0))
    selected = [h for h in candidates if h.filename in parsed["selected_memories"]][:5]

    for h in selected:
        content = Path(h.file_path).read_text()     # 二次读取：只读被选中的
        freshness = memory_freshness_warning(h.mtime_ms)
        ...
```

设计要点：
- **两步 IO**：`scan_memory_headers()` 只读每文件的 frontmatter（前 30 行），sideQuery 选出文件名后才读全文，几十条记忆时大幅减少 IO
- **精准优先**："if unsure, do not include it" — 宁可漏掉也不塞无关记忆
- **去重**：`already_surfaced` 集合跟踪已召回路径，同一会话不重复注入

#### 第 2 步：异步预取 — [memory.py:301-325](python/mini_claude/memory.py#L301-L325)

记忆召回在后台进行，主 Agent 首轮响应不受影响：

```python
def start_memory_prefetch(query, side_query, already_surfaced, session_memory_bytes):
    if not re.search(r"\s", query.strip()):   # 单字查询 → 不触发
        return None
    if session_memory_bytes >= 60 * 1024:     # 会话配额用尽 → 不触发
        return None
    if not any_memories_exist():              # 无记忆文件 → 不触发
        return None

    task = asyncio.create_task(select_relevant_memories(...))  # 后台启动
    return MemoryPrefetch(task)
```

Agent 侧 — [agent.py:868-904](python/mini_claude/agent.py#L868-L904)，在 while 循环中非阻塞轮询是否完成：

```python
memory_prefetch = start_memory_prefetch(user_message, side_query, ...)

# —— 主 LLM 已经开始处理用户问题 ——

while True:
    if memory_prefetch and memory_prefetch.settled and not memory_prefetch.consumed:
        memory_prefetch.consumed = True
        memories = memory_prefetch.task.result()
        if memories:
            injection_text = format_memories_for_injection(memories)
            last["content"] = last["content"] + "\n\n" + injection_text
```

时间线：
```
t=0ms   start_memory_prefetch() 启动后台 task
t=0ms   主 LLM 开始生成首轮回复（sideQuery 和主 LLM 并行）
t=200ms sideQuery 完成 → settled = True
t=300ms 主 LLM 首轮完成 → while 循环 → consume → 注入到下一轮
```

#### 第 3 步：新鲜度提醒 — [memory.py:198-213](python/mini_claude/memory.py#L198-L213)

每条记忆附带时间标签，旧记忆追加过期警告：

```python
def memory_age(mtime_ms):              # "today" / "yesterday" / "5 days ago"
def memory_freshness_warning(mtime_ms):
    if days <= 1:
        return ""                      # 1 天内的不警告
    return ("This memory is {days} days old. Memories are point-in-time "
            "observations, not live state — verify against current code.")
```

注入到记忆 header（[memory.py:272-276](python/mini_claude/memory.py#L272-L276)）：

```
<system-reminder>
This memory is 14 days old. Memories are point-in-time observations...
Memory: ~/.mini-claude/.../user_auth_preference.md:
User prefers JWT over session-based auth.
</system-reminder>
```

#### 完整数据流

```
用户: "帮我重构 auth 模块"
  │
  ├─ start_memory_prefetch("重构 auth 模块", ...)
  │   └─ asyncio.create_task(select_relevant_memories(...))  后台跑
  │
  ├─ 主 LLM 开始生成首轮回复（与 sideQuery 并行）
  │
  ├─ sideQuery 完成 → settled=True → result = [Memory("auth_preference.md", ...)]
  │
  ├─ while 循环消耗 injection → 用户消息末尾追加记忆文本
  │
  └─ 下一轮 LLM 已知用户偏好 JWT，无需对话中携带
```

#### 三要素总结

| 要素 | 函数 | 解决的问题 |
|------|------|-----------|
| sideQuery 语义召回 | `select_relevant_memories()` | 精准 ≤5 条，避免全量注入浪费 token |
| 异步预取 | `start_memory_prefetch()` | 后台跑，不阻塞首轮响应 |
| 新鲜度提醒 | `memory_freshness_warning()` | 旧记忆标过期风险，提醒验证 |

---

## 快速开始

```bash
# 1. 激活虚拟环境 & 安装
conda activate minicode
cd python
pip install -e .

# 2. 设置 API Key（三选一）

# 方式 A: Anthropic 原生
set ANTHROPIC_API_KEY=sk-ant-...

# 方式 B: DeepSeek（OpenAI 兼容）
set OPENAI_API_KEY=sk-your-deepseek-key
set OPENAI_BASE_URL=https://api.deepseek.com
set MINI_CLAUDE_MODEL=deepseek-chat

# 方式 C: 其他 OpenAI 兼容服务
set OPENAI_API_KEY=sk-xxx
set OPENAI_BASE_URL=https://your-proxy.com/v1

# 3. 运行
mini-claude-py                        # 交互式 REPL
mini-claude-py "explain this code"   # 一次性提问
python -m mini_claude                # 等价方式
```

---

## 项目架构

```
python/
├── pyproject.toml              # 项目配置、依赖、入口点
├── README.md                   # 快速上手指南
├── PROJECT.md                  # 本文档
└── mini_claude/                # 主包
    ├── __init__.py             # 版本号
    ├── __main__.py             # CLI 入口 + REPL 循环
    ├── agent.py                # ★ 核心：Agent 主循环、双后端、4 层压缩
    ├── tools.py                # ★ 核心：10 个工具 + 5 种权限模式
    ├── prompt.py               # 系统提示词构造（模板 + 动态上下文）
    ├── ui.py                   # 终端 UI（彩色输出、spinner、diff 高亮）
    ├── session.py              # 会话持久化（JSON）
    ├── memory.py               # 记忆系统（文件 + 语义召回）
    ├── skills.py               # 技能系统（发现、解析、执行）
    ├── subagent.py             # 子 Agent 系统（fork-return 模式）
    ├── mcp_client.py           # MCP 协议客户端（JSON-RPC over stdio）
    └── frontmatter.py          # YAML frontmatter 解析器
```

### 架构全景图

```
┌─────────────────────────────────────────────────────────┐
│                     CLI Layer (__main__.py)               │
│  参数解析  │  REPL 循环  │  命令路由  │  信号处理        │
├─────────────────────────────────────────────────────────┤
│                   Agent Core (agent.py)                   │
│  ┌─────────────┐  ┌──────────────┐  ┌────────────────┐  │
│  │ Anthropic   │  │ OpenAI 兼容  │  │  Skill Fork    │  │
│  │ Backend     │  │ Backend      │  │  Agent Fork    │  │
│  │ (流式 SSE)  │  │ (流式 SSE)   │  │  (子进程隔离)  │  │
│  └──────┬──────┘  └──────┬───────┘  └──────┬─────────┘  │
│         │                │                  │            │
│  ┌──────┴────────────────┴──────────────────┴─────────┐  │
│  │              4-Layer Compression Pipeline           │  │
│  │  Tier 1: Budget  │  Tier 2: Snip  │  Tier 3: Micro │  │
│  └────────────────────────────────────────────────────┘  │
├─────────────────────────────────────────────────────────┤
│                    Tool System (tools.py)                 │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐ │
│  │ read     │ │ write    │ │ shell    │ │ plan       │ │
│  │ _file    │ │ _file    │ │ _run     │ │ _mode      │ │
│  │ grep     │ │ edit     │ │ web      │ │ agent      │ │
│  │ list     │ │ _file    │ │ _fetch   │ │ skill      │ │
│  └──────────┘ └──────────┘ └──────────┘ └────────────┘ │
│  ┌──────────────────────────────────────────────────────┐│
│  │       5 Permission Modes                             ││
│  │  default │ plan │ acceptEdits │ bypass │ dontAsk    ││
│  └──────────────────────────────────────────────────────┘│
├─────────────────────────────────────────────────────────┤
│                  Subsystems                               │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐ ┌────────────┐ │
│  │ Memory   │ │ Skills   │ │ Session  │ │ MCP Client │ │
│  │ System   │ │ System   │ │ Manager  │ │ (JSON-RPC) │ │
│  └──────────┘ └──────────┘ └──────────┘ └────────────┘ │
└─────────────────────────────────────────────────────────┘
```

---

## 各模块详细说明

### 1. `__main__.py` — CLI 入口层（309 行）

**职责**：命令行参数解析、API 配置决策、REPL 交互循环。

**核心逻辑**：
- **API 后端自动选择**：根据环境变量自动判断使用 Anthropic 还是 OpenAI 兼容后端
  - 同时有 `OPENAI_API_KEY` + `OPENAI_BASE_URL` → OpenAI 兼容模式
  - 有 `ANTHROPIC_API_KEY` → Anthropic 原生模式
  - 单独有 `OPENAI_API_KEY` → OpenAI 兼容模式
- **REPL 命令**：`/clear` 清上下文、`/plan` 切换计划模式、`/cost` 显示费用、`/compact` 压缩对话、`/memory` 列出记忆、`/skills` 列出技能、`/<skill>` 调用技能
- **信号处理**：Ctrl+C 中断正在运行的 Agent，双击退出
- **会话恢复**：`--resume` 恢复上次对话

### 2. `agent.py` — Agent 核心引擎（1294 行）

**职责**：最大的单文件，包含 Agent 主循环、双后端适配、4 层压缩管道、计划模式、子 Agent 调度。

**核心设计**：

#### 双后端架构
```
Agent.__init__
├── use_openai=True  → AsyncOpenAI(base_url, api_key)     ← DeepSeek 走这里
└── use_openai=False → AsyncAnthropic(api_key, base_url)   ← Claude 走这里
```

- **Anthropic 后端**（`_chat_anthropic`）：原生 Messages API，支持 thinking 扩展、流式工具执行（content_block_stop 事件触发并发工具调用）
- **OpenAI 兼容后端**（`_chat_openai`）：OpenAI Chat Completions API，自动转换工具格式，支持并发安全工具并行执行

#### 4 层上下文压缩管道

| 层级 | 名称 | 触发条件 | 策略 |
|------|------|----------|------|
| Tier 1 | Budget | 利用率 > 50% | 截断过长工具结果，保留头尾 |
| Tier 2 | Snip | 利用率 > 60% | 删除重复 read_file 结果，保留最近 K 条 |
| Tier 3 | Microcompact | 空闲 > 5 分钟 | 清除旧工具结果 |
| Tier 4 | Autocompact | 利用率 > 85% | LLM 摘要压缩整段对话 |

#### 计划模式
- 进入计划模式 → 只读 + 只能编辑 plan 文件
- 退出计划模式 → 展示 plan 给用户审批
- 4 种审批选项：清上下文执行 / 保留上下文执行 / 手动确认执行 / 继续修改计划

#### 流式工具执行
- Anthropic 路径：利用 `content_block_stop` 事件，工具参数接收完毕即刻启动执行，不等整个响应结束
- 并发安全工具（`read_file`、`list_files`、`grep_search`、`web_fetch`）可直接并行

#### 指数退避重试
- 可重试错误：429（限流）、503/529（服务过载）、网络错误
- 延迟公式：`min(2^n * 1s, 30s) + jitter`

### 3. `tools.py` — 工具系统（708 行）

**职责**：10 个工具的定义与执行 + 5 种权限模式 + 危险命令检测。

**10 个内置工具**：

| 工具 | 类型 | 说明 |
|------|------|------|
| `read_file` | 读 | 读取文件，返回带行号的内容 |
| `write_file` | 写 | 创建/覆盖文件 |
| `edit_file` | 写 | 精确字符串替换编辑，生成 unified diff |
| `list_files` | 读 | Glob 模式文件搜索 |
| `grep_search` | 读 | 正则内容搜索（优先系统 grep，回退 Python） |
| `run_shell` | 执行 | 任意 shell 命令 |
| `web_fetch` | 读 | HTTP GET 抓取，HTML 自动清洗 |
| `skill` | 调度 | 调用注册的技能 |
| `agent` | 调度 | 启动子 Agent |
| `tool_search` | 元 | 搜索并激活延迟加载的工具 |

**5 种权限模式**：

| 模式 | 读操作 | 写操作 | 危险命令 |
|------|--------|--------|----------|
| `default` | ✅ 允许 | ❓ 确认 | ❓ 确认 |
| `acceptEdits` | ✅ 允许 | ✅ 允许 | ❓ 确认 |
| `plan` | ✅ 允许 | ❌ 拒绝（仅 plan 文件除外） | ❌ 拒绝 |
| `bypassPermissions` | ✅ 允许 | ✅ 允许 | ✅ 允许 |
| `dontAsk` | ✅ 允许 | ❌ 拒绝 | ❌ 拒绝 |

**安全特性**：
- 读前写保护：必须先 `read_file` 才能 `write_file`/`edit_file`
- mtime 新鲜度检查：检测外部修改，防止覆盖他人改动
- 20+ 危险命令正则模式检测（`rm`、`git push --force`、`sudo`、`kill` 等）
- `.claude/settings.json` 自定义 allow/deny 规则

### 4. `prompt.py` — 系统提示词构造（243 行）

**职责**：构建完整的系统提示词，模板 + 动态变量插值。

**核心功能**：
- **`@include` 解析**：支持 CLAUDE.md 中 `@./path`、`@~/path` 引用其他文件，最大 5 层深度
- **CLAUDE.md 收集**：从当前目录向上遍历，收集所有 `CLAUDE.md` 并合并
- **Rules 加载**：加载 `.claude/rules/*.md` 中的所有规则文件
- **Git 上下文**：自动获取当前分支、最近 5 条 commit、工作区状态
- **动态变量**：`{{cwd}}`、`{{date}}`、`{{platform}}`、`{{shell}}`、`{{git_context}}` 等

### 5. `ui.py` — 终端 UI（208 行）

**职责**：基于 `rich` 库的彩色终端渲染。

**亮点**：
- **Spinner 动画**：独立线程驱动，API 调用期间显示旋转动画，首字输出时自动消失
- **Diff 高亮**：`edit_file` 结果彩色渲染（`@@` 青色、`-` 红色、`+` 绿色）
- **工具调用展示**：每种工具配专属 emoji 图标 + 关键参数摘要
- **图片查看**：支持 PNG/JPG/PDF 文件的终端展示

### 6. `memory.py` — 记忆系统（379 行）

**职责**：文件式持久记忆 + 语义化智能召回。

**核心设计**：

```
用户提问
    │
    ▼
┌──────────────────────┐
│ Semantic Recall      │  ← sideQuery: 用小模型从记忆清单中选出相关记忆
│ (select_relevant)    │
└──────┬───────────────┘
       │ selected filenames
       ▼
┌──────────────────────┐
│ Memory Injection     │  ← 作为 <system-reminder> 注入到用户消息
│ (format_for_inject)  │
└──────┬───────────────┘
       │
       ▼
    主 LLM 调用
```

**4 种记忆类型**：`user`（用户偏好）、`feedback`（用户反馈）、`project`（项目信息）、`reference`（外部引用）

**关键约束**：
- 单文件最大 4KB
- 每次会话累计最多 60KB 记忆注入
- 最多返回 5 条相关记忆
- 单次提问才触发召回（避免单字查询浪费 API）

### 7. `skills.py` — 技能系统（172 行）

**职责**：从 `.claude/skills/<name>/SKILL.md` 发现、解析和执行技能。

**技能类型**：
- **inline**：技能 prompt 直接注入当前对话上下文
- **fork**：新开隔离 Agent 执行（独立上下文，返回结果）

**支持格式**：
```markdown
---
name: my-skill
description: What this skill does
user-invocable: true
context: fork
allowed-tools: read_file, grep_search, list_files
---
Skill prompt template with ${ARGUMENTS} and ${CLAUDE_SKILL_DIR}...
```

### 8. `subagent.py` — 子 Agent 系统（172 行）

**职责**：3 种内置 Agent 类型 + 自定义 Agent 发现。

| 类型 | 工具 | 用途 |
|------|------|------|
| `explore` | 只读（3 个） | 快速代码搜索与探索 |
| `plan` | 只读（3 个） | 结构化实现计划设计 |
| `general` | 全部（除 agent） | 独立任务执行 |
| 自定义 | 可配置 | 从 `.claude/agents/*.md` 加载 |

**设计模式**：fork-return — 子 Agent 在独立上下文中运行，完成后仅返回文本结果给主 Agent，不污染主对话上下文。

### 9. `mcp_client.py` — MCP 协议客户端（251 行）

**职责**：连接 stdio 型 MCP 服务器，发现并转发工具调用。

**核心实现**：
- 纯 JSON-RPC 2.0 over stdio（零外部依赖）
- 异步读写循环：`asyncio.create_subprocess_exec` + background reader task
- 工具名加 `mcp__<server>__<tool>` 前缀避免冲突
- 配置来源：`~/.claude/settings.json` → `.claude/settings.json` → `.mcp.json`

### 10. `session.py` — 会话管理（50 行）

**职责**：JSON 文件持久化，支持会话列表和恢复。

### 11. `frontmatter.py` — YAML Frontmatter 解析器（48 行）

**职责**：轻量级 `---` 分隔的 key: value 解析，零依赖。

---

## 创新点与技术亮点

### 1. 双后端无缝切换

同一个 Agent 类同时支持 Anthropic 原生格式和 OpenAI 兼容格式，根据初始化参数自动选择消息格式、工具格式、流式解析方式。用户只需切换环境变量，无需改代码。这使得 DeepSeek 等非 Anthropic 模型可以直接接入。

### 2. 4 层渐进式压缩管道

不同于简单的"满了就摘要"策略，实现了 4 层递进压缩：

- **Tier 1（Budget）**：仅在利用率 > 50% 时触发，截断过长的工具结果，保留头尾关键信息
- **Tier 2（Snip）**：在利用率 > 60% 时，智能删除重复的 read_file 结果，保留最近 K 条
- **Tier 3（Microcompact）**：仅在空闲 > 5 分钟时触发，清除旧结果但不触发 API 调用
- **Tier 4（Autocompact）**：在利用率 > 85% 时，调用 LLM 做语义摘要压缩

这种设计避免了频繁的 LLM 摘要调用（节省费用），同时在不同负载下渐进式释放空间。

### 3. 流式工具并发执行

在 Anthropic 路径中，利用 SSE 事件的 `content_block_stop` 信号，当一个 tool_use 块的 JSON 参数接收完毕时立即启动工具执行，不等整个 LLM 响应结束。对于无依赖的只读工具（如 read_file），可以在模型还在生成下一个 tool_use 时就开始执行，显著降低端到端延迟。

### 4. 语义记忆召回（sideQuery）

不依赖简单的关键词匹配或全量注入，而是用一个轻量级 LLM 调用（sideQuery）对记忆清单做语义选择，只注入与当前问题真正相关的记忆。配合 4KB/条 和 60KB/会话 的硬限制，在记忆容量和上下文预算之间取得平衡。

### 5. 读前写保护 + mtime 新鲜度

实现了完整的 read-before-edit 保护：
- 编辑/覆盖文件前必须先用 `read_file` 读取
- 记录读取时的 mtime，编辑前检查文件是否被外部修改
- 防止 AI 基于过时内容做出错误修改

### 6. 计划模式（Plan Mode）

完整实现了 Claude Code 的计划模式工作流：
- 进入 → 只读探索 + 写 plan 文件
- 退出 → 4 种审批选项（清上下文执行 / 保留执行 / 手动确认 / 继续修改）
- Plan 文件持久化到 `~/.claude/plans/`

### 7. 纯 Python MCP 实现

零外部 SDK 依赖，仅用 `asyncio.create_subprocess_exec` + 行读取循环实现完整的 JSON-RPC 2.0 MCP 客户端，支持多服务器连接、工具发现、`mcp__` 前缀路由。

### 8. 延迟工具加载（Deferred Tools）

`tool_search` 机制允许工具在首次被搜索时才激活，减少每次 API 调用的工具定义体积，节省输入 token。

### 9. 指数退避 + Jitter 重试

标准的生产级重试策略：`delay = min(2^n * 1s, 30s) + random_jitter`，针对 429/503/529/网络错误。

### 10. 大结果磁盘持久化

工具结果超过 30KB 时自动写入 `~/.mini-claude/tool-results/`，上下文只保留 200 行预览 + 文件路径，模型可后续通过 `read_file` 读取完整结果。

---

## 技术栈

| 依赖 | 版本 | 用途 |
|------|------|------|
| `anthropic` | ≥0.40.0 | Anthropic Messages API（流式） |
| `openai` | ≥1.50.0 | OpenAI Chat Completions API（兼容 DeepSeek 等） |
| `rich` | ≥13.0.0 | 终端彩色输出 |
| Python | ≥3.11 | async/await、类型提示、dataclass |

---

## 命令行参数

```
用法: mini-claude-py [选项] [prompt]

选项:
  --yolo, -y          跳过所有确认（bypassPermissions 模式）
  --plan              计划模式（只读）
  --accept-edits      自动批准文件编辑
  --dont-ask          自动拒绝确认（CI 模式）
  --thinking          启用扩展思考（Anthropic only）
  --model, -m         指定模型
  --api-base URL      OpenAI 兼容 API 地址
  --resume            恢复上次会话
  --max-cost USD      费用上限（美元）
  --max-turns N       最大 Agent 轮数
  --help, -h          帮助

REPL 命令:
  /clear              清空对话历史
  /plan               切换计划模式
  /cost               显示 token 用量和费用
  /compact            手动压缩对话
  /memory             列出记忆
  /skills             列出技能
  /<skill-name>       调用技能
```

---

## 配置来源优先级

1. **CLAUDE.md**：从当前目录向上遍历，合并所有 `CLAUDE.md`（支持 `@include` 引用）
2. **Rules**：`.claude/rules/*.md`
3. **Skills**：`.claude/skills/<name>/SKILL.md`（项目级覆盖用户级 `~/.claude/skills/`）
4. **Agents**：`.claude/agents/<name>.md`（项目级覆盖用户级）
5. **Permissions**：`.claude/settings.json`（allow/deny 规则）
6. **MCP Servers**：`.claude/settings.json` 或 `.mcp.json`
7. **Sessions**：`~/.mini-claude/sessions/`
8. **Memories**：`~/.mini-claude/projects/<hash>/memory/`

---

## 与 Claude Code 的对照

| 特性 | Claude Code (TypeScript) | Mini Claude Code (Python) |
|------|--------------------------|---------------------------|
| Agent 主循环 | ✅ | ✅ |
| Anthropic 后端 | ✅ | ✅ |
| OpenAI 兼容后端 | ❌ | ✅ |
| DeepSeek 支持 | ❌ | ✅ |
| 4 层压缩 | ✅ | ✅ |
| 10 个内置工具 | ✅ | ✅ |
| 5 种权限模式 | ✅ | ✅ |
| 计划模式 | ✅ | ✅ |
| 子 Agent (fork) | ✅ | ✅ |
| Skill 系统 | ✅ | ✅ |
| Memory 系统 | ✅ | ✅ |
| MCP 协议 | ✅ | ✅ |
| 流式工具执行 | ✅ | ✅ |
| 会话持久化 | ✅ | ✅ |
| 延迟工具加载 | ✅ | ✅ |
| REPL 交互 | ✅ | ✅ |
| CLI 参数 | ✅ | ✅ |
| Thinking 扩展 | ✅ | ✅（仅 Anthropic） |

# Mini Claude Code 全面审查报告

## 一、整体评价

这是一个结构清晰、设计精良的 Coding Agent 项目。核心机制（双后端、4层压缩、5种权限模式、延迟工具激活、流式工具执行、read-before-edit等）都已实现且质量不错。但作为 `v1.0.0`，仍有大量可完善的空间。以下从 **10个维度** 逐一分析。

---

## 二、具体问题与改进建议

### 1. 代码质量与工程化

| 问题 | 位置 | 严重度 | 建议 |
|------|------|--------|------|
| 零测试覆盖 | `test.py` 是贪吃蛇游戏，不是测试 | 高 | 建立 pytest 测试体系，覆盖 tools、agent、permission、memory 等核心模块 |
| 调试 print 残留 | `session.py:L36`、`agent.py:L334`、`agent.py:L344` | 中 | 删除或改用 `logging` 模块 |
| 无 logging 框架 | 全局 | 中 | 引入 `logging` 替代 `print()`，支持日志级别和输出目标 |
| 无 lint/typecheck | 全局 | 中 | 添加 `ruff`/`mypy` 配置和 pre-commit hooks |
| `asyncio.get_event_loop()` 已废弃 | `mcp_client.py:L81` | 中 | 改用 `asyncio.get_running_loop()` |
| spinner 用 threading 而非 asyncio | `ui.py:L103-L135` | 中 | 在 asyncio 应用中用 threading 跑 spinner 可能在 Windows 上有事件循环冲突，建议改用 asyncio Task |
| `_with_retry` 的抖动用 `hash()` | `agent.py:L72` | 低 | 非加密随机，建议用 `random.random()` |

### 2. 工具系统完善

| 缺失/改进 | 说明 | 建议 |
|-----------|------|------|
| `ask_user_question` 工具 | Agent 无法向用户提问澄清 | 添加此工具，让 Agent 在不确定时反问用户 |
| `diff` 工具 | 无法预览变更 | 添加 `diff_file` 工具，在写入前展示 diff |
| `todo_write` 工具 | 无法管理任务列表 | 添加任务管理工具，提升长任务的可追踪性 |
| `web_search` 工具 | 只有 `web_fetch` 没有搜索 | 添加搜索引擎集成（如 Bing/Google API） |
| `read_file` 不支持二进制 | `tools.py:L202-L209` | 添加二进制文件检测和 base64 编码返回 |
| `web_fetch` 同步阻塞 | `tools.py:L428-L442` | 改用 `aiohttp` 或 `httpx` 实现异步 fetch |
| `run_shell` 无流式输出 | `tools.py:L405-L425` | 支持 `streaming` 参数实时输出 |
| `run_shell` 无 cwd 参数 | 无法指定工作目录 | 添加 `cwd` 参数 |
| `grep_search` 收集 200 条但只返回 100 条 | `tools.py:L396-L401` | 浪费资源，统一限额 |
| `edit_file` 无 dry-run 模式 | 无法预览修改效果 | 添加 `dry_run` 参数返回 diff 而不实际修改 |

### 3. 多后端与 LLM 集成

| 问题 | 位置 | 说明 |
|------|------|------|
| 费用硬编码 | `agent.py:L399` | `$3/M input, $15/M output` 对所有模型统一，实际各模型价格差异巨大，应从配置文件或模型元数据读取 |
| 上下文窗口映射不全 | `agent.py:L81-L90` | 缺少 `gpt-4-turbo`, `gpt-4.1`, `claude-3.5-sonnet` 等主流模型 |
| OpenAI 路径无流式工具执行 | `agent.py:L1189-L1227` | Anthropic 路径有流式工具执行优化，OpenAI 路径只能做并发分组，两者不等价 |
| 无 HTTP 代理支持 | 全局 | 企业环境常需代理，应为 `AsyncAnthropic`/`AsyncOpenAI` 添加 `proxies` 参数 |
| 无 response_format / structured output | 全局 | 不支持 JSON mode 等结构化输出 |
| 无流中断恢复 | 全局 | 流中断后无法从断点续传 |

### 4. 上下文与记忆系统

| 问题 | 位置 | 说明 |
|------|------|------|
| 记忆仅单字查询不触发 | `memory.py:L309` | `"重构"` 这样的短词也会触发，`r"\s"` 判断过于简单 |
| 记忆文件 4KB 限制 | `memory.py:L152` | 有些记忆可能需要更大空间 |
| 无记忆淘汰机制 | 全局 | 记忆只增不减，长时间使用会积累大量过时记忆 |
| 无记忆导出/导入 | 全局 | 无法在不同项目间共享记忆 |
| 压缩总结质量无保障 | `agent.py:L461-L477` | 压缩后可能丢失关键信息，无验证机制 |
| `_snip_stale_results_openai` 无按文件去重 | `agent.py:L581-L593` | OpenAI 路径的 snip 比 Anthropic 路径简单很多，缺少同一文件多次读取去重逻辑 |

### 5. 权限与安全

| 问题 | 位置 | 说明 |
|------|------|------|
| 危险命令正则不够全面 | `tools.py:L464-L481` | 缺少 `chmod 777`, `chown`, `docker rm`, `kubectl delete`, `DROP TABLE`, `TRUNCATE` 等 |
| 权限规则缓存无失效 | `tools.py:L507-L531` | `_cached_rules` 加载后永不刷新，修改 settings.json 不生效 |
| 无命令沙箱 | 全局 | `run_shell` 直接执行系统命令，无 `chroot`/容器化限制 |
| 无工具调用审计日志 | 全局 | 无法追溯 Agent 做了哪些操作 |
| `_confirmed_paths` 白名单跨会话不持久 | `agent.py:L198` | 每次启动重新确认 |

### 6. UI/UX

| 问题 | 位置 | 说明 |
|------|------|------|
| 无 Markdown 渲染 | 全局 | 终端输出纯文本，不支持代码高亮、表格等 |
| 无历史命令搜索 | `__main__.py` | REPL 不支持 `Ctrl+R` 搜索、上下箭头历史 |
| 无自动补全 | REPL | 不支持文件路径/命令补全 |
| 无 --verbose 模式 | 全局 | 无法查看详细的 API 请求/响应 |
| 无进度条 | 全局 | 长任务（如大量文件搜索）无进度指示 |
| spinner 在 Windows cmd 可能乱码 | `ui.py:L101` | braille 字符在 Windows cmd 中显示异常，应检测终端能力 |

### 7. 会话管理

| 问题 | 位置 | 说明 |
|------|------|------|
| 无会话列表查看 | `session.py` | 只能用 `--resume` 恢复最近一个，无法选择特定会话 |
| 无会话删除/清理 | 全局 | 旧会话文件会无限积累 |
| 会话保存可能 OOM | `session.py:L16-L18` | 超长对话的 JSON 序列化可能内存爆炸 |
| 无会话导出 | 全局 | 无法导出为 Markdown/HTML 方便分享 |

### 8. MCP 集成

| 问题 | 位置 | 说明 |
|------|------|------|
| 无 MCP 工具调用超时 | `mcp_client.py:L117-L124` | 工具调用可能无限等待 |
| `_read_loop` 无缓冲区处理 | `mcp_client.py:L51-L71` | 假设每次 `readline()` 返回完整的一行 JSON-RPC，大消息可能被截断 |
| 无 MCP 重连机制 | `mcp_client.py` | MCP 服务进程崩溃后无法自动重连 |
| 无 SSE/HTTP MCP transport | `mcp_client.py` | 仅支持 stdio，不支持远程 MCP 服务器 |
| MCP 工具 schema 转换不完整 | `mcp_client.py:L186-L195` | MCP 的 `inputSchema` 与 Anthropic 的 `input_schema` 字段名差异，且可能缺少 `required` 等字段 |

### 9. 子 Agent 系统

| 问题 | 位置 | 说明 |
|------|------|------|
| 子 Agent 无并发限制 | `agent.py:L822-L847` | 多子 Agent 同时运行可能导致 API 限流或资源耗尽 |
| 子 Agent 无超时 | 全局 | 子 Agent 可能无限运行 |
| 子 Agent 结果无结构化 | 全局 | 返回纯文本，难以解析 |
| 自定义 Agent 工具名必须匹配内置 | `subagent.py:L128-L130` | 只支持内置工具名，不支持 MCP 工具或自定义工具 |

### 10. 未来功能方向

| 方向 | 说明 |
|------|------|
| 插件系统 | 支持第三方工具扩展，类似 Claude Code 的 hooks |
| 多模态支持 | 支持图片输入（截图、架构图），Claude/OpenAI 已支持 |
| IDE 集成 | 作为 VS Code / JetBrains 插件运行 |
| 团队协作 | 共享 CLAUDE.md、skills、memory 到团队仓库 |
| Web UI | 提供浏览器界面，方便非终端用户 |
| 多 Agent 编排 | 支持多个 Agent 协同工作（如 PM Agent + Dev Agent + QA Agent） |
| Prompt 版本管理 | 对 system prompt 做版本控制和 A/B 测试 |
| Token 预算精细化管理 | 按工具类型、消息角色分配 token 预算 |
| 自动代码审查 | 集成 git diff 自动 review |
| RAG 知识库 | 集成项目文档、API 文档作为知识库 |

---

## 三、优先级建议

| 优先级 | 改进项 | 理由 |
|--------|--------|------|
| **P0** | 建立测试体系 | 无测试意味着每次改动都是赌博 |
| **P0** | 清理调试 print、引入 logging | 当前日志混乱，排查问题困难 |
| **P1** | 添加 `ask_user_question`、`todo_write`、`diff` 工具 | 补齐核心工具能力 |
| **P1** | `web_fetch` 改为异步、`run_shell` 支持 cwd | 消除阻塞和功能缺失 |
| **P1** | 费用模型改为可配置 | 当前硬编码导致费用估算不准 |
| **P1** | 会话列表/删除/导出 | 会话管理基本可用但残缺 |
| **P2** | MCP 超时/重连/SSE transport | MCP 集成健壮性不足 |
| **P2** | 子 Agent 并发限制和超时 | 防止资源耗尽 |
| **P2** | 命令行历史/自动补全 | 改善 REPL 体验 |
| **P2** | 危险命令正则扩展 | 提升安全性 |
| **P3** | 多模态、插件系统、Web UI | 更长期的功能演进 |

---

## 四、总结

这个项目在 **Agent 核心架构** 上做得相当扎实——双后端、流式工具执行、4层压缩、权限体系、记忆系统都是正确且合理的设计。主要短板在于：

1. **工程化程度不足**（无测试、无日志、调试代码残留）
2. **工具生态不完整**（缺少提问、diff、todo、搜索等核心工具）
3. **健壮性细节**（MCP 超时、内存边界、会话管理）
4. **硬编码假设**（费用模型、模型列表、上下文窗口）

建议先解决 P0/P1 项，这些是投入产出比最高的改进。
# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

CodexMCP 是一个 FastMCP 服务器（stdio 传输），把 OpenAI Codex CLI 的 `codex exec` 包装成单个 MCP 工具 `codex`，供 Claude Code 调用。相比官方 Codex MCP，核心增值是：会话持久化（多轮对话 resume）、推理详情追踪（`return_all_messages`）、并行任务和错误处理。

运行时依赖外部 `codex` CLI（≥ v0.61.0，需在 PATH 上）；配套 Claude Code 需 ≥ v2.0.56。Python ≥ 3.12。

## 常用命令

```bash
uv sync                # 安装依赖
uv run codexmcp        # 启动 MCP 服务器（stdio，阻塞等待 JSON-RPC 输入）
uv run codexmcp-monitor --no-open   # 启动监控面板（独立进程，默认 127.0.0.1:8765）
```

本地调试改动时，把 Claude Code 的 MCP 配置指向本地检出（替换 README 中的 uvx git URL）：

```bash
claude mcp remove codex
claude mcp add codex -s user --transport stdio -- uv --directory <仓库路径> run codexmcp
```

仓库目前没有测试套件和 linter 配置。

## 架构

`server.py` 是 MCP 工具本体，`cli.py` 只是入口壳；`recorder.py` 负责把运行过程落盘，`monitor.py` + `static/index.html` 是读取这些记录的独立 Web 面板。数据流：

1. MCP 工具调用 `codex(PROMPT, cd, ...)` → 以列表形式拼装 `codex exec --sandbox ... --json` 命令（避免注入）；若传入 `SESSION_ID` 则拼成 `codex exec ... resume <id>` 实现多轮会话。
2. `run_shell_command()` 用 `subprocess.Popen` 启动 codex，后台读线程逐行读 stdout（JSONL 事件流），通过 `queue.Queue` 交给生成器逐行 yield。
3. 工具主体逐行 `json.loads` 解析事件：累积 `agent_message` 文本、从事件中提取 `thread_id`（即返回给调用方的 `SESSION_ID`）、按事件 type 中的 `fail`/`error` 归类错误；同时把每行 tee 给 `RunRecorder`。
4. 返回 `{success, SESSION_ID, agent_messages}`；`return_all_messages=True` 时附带完整事件列表 `all_messages`。

### 运行记录与监控面板

Claude Code 在工具执行期间不渲染任何中间信息（MCP 无流式结果，progress/logging 通知客户端也不显示），所以过程可见性靠落盘 + 独立面板实现，而不是靠 MCP 通知。

- 记录目录 `~/.codexmcp/runs/`（根目录可用 `CODEXMCP_HOME` 覆盖），每次调用两个文件：`<run_id>.meta.json`（状态/参数，原子替换写入）与 `<run_id>.jsonl`（事件流，逐行追加，行缓冲）。
- `monitor.py` 只读这些文件，前端用 `?offset=` 轮询增量拉取（1s 事件 / 2s 列表）。offset 按**行号**计数而非事件数，解析失败的行不会让游标错位。
- 前端按 `item.id` 合并 `item.started/updated/completed`，所以一条命令是原地更新而不是堆三条。

## 关键实现细节（改动时注意）

- **进程终止**：codex 在输出 `turn.completed` 事件后可能不主动退出。读线程检测到该事件后等 0.3 秒（`GRACEFUL_SHUTDOWN_DELAY`）主动 `terminate()`，随后 `wait(timeout=5)` 兜底 `kill()`。改动流式读取逻辑时必须保留这条终止路径，否则会挂起或泄漏进程。
- **Windows 兼容**：`codex` 在 Windows 上是 `.cmd` shim，用 `shutil.which('codex')` 解析真实路径；`os.name == "nt"` 时对 PROMPT 做 `windows_escape()` 转义（引号、换行等），避免 cmd shim 重解释。README 推荐 Windows 用户在 WSL 中运行。
- **错误判定规则**：只有在尚未累积到任何 `agent_messages` 时，`fail`/`error` 事件才把 `success` 置为 False（已有正文则只追加错误信息）。形如 `Reconnecting... n/m` 的 error 事件是网络重连噪音，用正则识别并忽略，不计为失败。
- **依赖钉死**：`mcp` 必须 `>=1.20.0,<2.0.0` —— mcp 2.0 移除了 `mcp.server.fastmcp`，升级会直接 import 失败。
- **录制不得反噬主流程**：`RunRecorder` 的所有方法都吞异常，任何一次失败就整体关闭当次录制。`codex()` 里事件循环外层的 `except BaseException` 负责在异常/取消时把状态收尾成 `failed`，否则会留下永远 `running` 的记录。改动这两处时保持这个约束。
- **stdio 传输禁止写 stdout**：MCP 走 stdio，任何 `print` 都会污染 JSON-RPC 流。需要观测就写文件（recorder 就是这么做的）。
- **文档同步**：`@mcp.tool` 装饰器里的 description 是 Claude Code 侧看到的工具说明书；改参数或行为时需同步更新它、`README.md`（中文）和 `docs/README_EN.md`（英文）中的参数表及推荐 prompt。

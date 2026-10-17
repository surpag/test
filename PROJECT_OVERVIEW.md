# Person-Copilot 项目分析

> 一个自研（非 LangChain / 框架依赖）的「个人助理」Agent 后端：LLM function calling + 工具注册 + 长期记忆 + 多会话隔离 + 上下文压缩 + SSE 流式输出，并自带一个单页 Web 前端。

- 仓库：https://github.com/surpag/Person-Copilot
- 语言 / 运行时：Python 3（asyncio）
- 代码规模：约 40 个 Python 文件（含 20 个测试文件），核心逻辑集中在 `backend/agent/`
- 存储：SQLite（单文件，支持 `DATA_DIR` 环境变量以适配 Railway Volume）

---

## 1. 目录结构

```
Person-Copilot/
├── backend/agent/
│   ├── run.py                  # 核心：Copilot 对话循环 + 上下文压缩（摘要）
│   ├── prompt.py               # system prompt 组装（人设 + 偏好 + 待办 + 工具规则）
│   ├── api/                    # FastAPI 服务
│   │   ├── main.py             # 路由与静态页面挂载（chat.html / index.html）
│   │   ├── auth.py             # JWT 鉴权（RS256 公钥验签）
│   │   ├── vision.py           # 多模态图片描述接口
│   │   ├── pool.py             # Copilot 实例池 / 会话管理
│   │   └── schemas.py          # 请求/响应模型
│   ├── tools/                  # 工具层
│   │   ├── registry.py         # 工具注册中心（装饰器 + 自动 JSON Schema）
│   │   ├── memory_tools.py     # remember/forget、add/list/complete todo
│   │   ├── note_tools.py       # save_note / get_note / list_notes / search_notes
│   │   ├── web_search.py       # web_search
│   │   ├── fetch_url.py        # fetch_url
│   │   ├── weather.py          # get_weather
│   │   └── context.py          # 请求级上下文（last_image / last_message 等）
│   ├── memory/                 # 长期记忆（跨会话，永不被压缩）
│   │   ├── schema.py           # preferences / todos / notes 建表与迁移
│   │   ├── preferences.py      # PreferencesStore
│   │   ├── todos.py            # TodosStore
│   │   └── notes.py            # NotesStore
│   ├── session/                # 会话存储（可被压缩/归档）
│   │   ├── schema.py           # sessions / messages / summaries 建表与迁移
│   │   ├── store.py            # SessionStore
│   │   ├── trace.py            # Tracer：每轮对话的调用链追踪
│   │   └── trace_viewer.py     # trace 查看器
│   ├── cli/                    # memory CLI、summary 查看 CLI
│   ├── static/                 # 前端单页（chat.html + 图片压缩 JS）
│   └── tests/                  # pytest 测试（20 个文件）
├── test_cancel.py              # 流式取消（停止按钮）测试
├── Dockerfile / docker-compose.yml / railway.json
└── requirements.txt / pytest.ini
```

## 2. 核心模块

### 2.1 对话循环 `run.py`（`Copilot` 类）
- 基于 `AsyncOpenAI`，通过 `OPENAI_BASE_URL` 支持任意兼容网关 / 本地模型。
- 标准 tool-call 循环：模型返回 `tool_calls` → `invoke_tool()` 执行 → 结果回填 → 继续请求，直到产出纯文本答案。
- 每个实例绑定一个 `session_id`，启动时从 SQLite 恢复历史，实现**多会话隔离**。
- **上下文压缩（compaction）**是本项目的亮点：
  - `_plan_compaction()`：纯计算，按 token 估算（`MAX_ACTIVE_TOKENS`）和轮数（`MAX_TURNS`）双阈值判断，按「turn」对齐保留最近消息；
  - `_compact()`：调用独立的小模型（`summary_model`）生成结构化摘要（目标 / 关键事实 / 已完成 / 进行中 / 待办 / 下一步）；
  - 支持**增量摘要**（已有摘要 + 新历史 → 更新摘要）；
  - 摘要失败返回 `None` 并降级，不阻塞对话；
  - `_is_context_overflow()` 兼容 `context_length_exceeded`，防死锁。

### 2.2 Prompt 组装 `prompt.py`
- 纯函数 `build_system_prompt(persona, preferences, todos)`，便于单元测试。
- 结构固定：`人设 → 已知用户偏好 → 当前待办 → 工具使用规则`，并对偏好/待办条数做上限截断（默认 20 / 10 条），避免无限注入。
- 工具规则以自然语言约束模型行为（如「保存图片必须选 content_ref」「保存成功后立即停止」）。

### 2.3 工具系统 `tools/registry.py`
- `@tool(description=...)` 装饰器 + Pydantic 参数模型，自动生成 OpenAI function-calling JSON Schema（并递归清理 `title` 字段）。
- 同时支持 sync / async 工具；统一 `invoke_tool()` 入口，封装 JSON 解析、Pydantic 校验与异常兜底。
- 已注册工具：
  - 记忆类：`remember_preference`、`forget_preference`
  - 待办类：`add_todo`、`list_todos`、`complete_todo`
  - 笔记类：`save_note`、`get_note`、`list_notes`、`search_notes`
  - 信息类：`web_search`、`fetch_url`、`get_weather`

### 2.4 双层存储：session vs memory
| 层 | 表 | 语义 |
|---|---|---|
| 会话层 `session/` | `sessions`、`messages`、`summaries` | 单会话历史，会被压缩 / 归档（`archived` 字段） |
| 记忆层 `memory/` | `preferences`、`todos`、`notes` | 跨会话长期记忆，**永不压缩** |

- 两套 schema 共用同一个 DB 文件但互不干扰。
- 内置轻量迁移（`PRAGMA table_info` 检查 + `ALTER TABLE ADD COLUMN`），兼容旧库，如给 `notes` 补 `summary`、给 `messages` 补 `image_path` / `archived`。

### 2.5 API 层 `api/main.py`
- FastAPI + `StreamingResponse` 输出 **SSE**，事件类型：`status`、`thinking`、`token`、`tool_call`、`tool_result`、`trace`、`done`、`error`。
- 支持**流式取消**（客户端断开即中断 LLM 请求，对应前端「停止按钮」），并记录 `interrupted` 标记。
- 图片上传：先经 `vision.py` 用多模态模型生成描述，再把描述注入本轮上下文，原图落盘由 `image_path` 引用。
- 部署时挂载前端静态页（`chat.html`）。

### 2.6 可观测性 `session/trace.py`
- `Tracer` / `Trace` / `trace_step()` 以异步上下文管理器记录每个环节（LLM 调用、工具调用、摘要）的输入输出、耗时、token 用量。
- trace 既写库也通过 SSE 推给前端，另有 `trace_viewer.py` 便于调试。

### 2.7 鉴权 `api/auth.py`
- RS256 JWT 公钥验签，兼容「纯密钥字符串」与「PEM 文件路径」两种配置，并支持 PKCS#1 → PKCS#8 自动转换。
- 通过 `AUTH_ENABLED` 开关；另支持 `API_TOKEN` Bearer 简单鉴权。

## 3. 测试与部署

- **测试**：`pytest.ini` 配置 `asyncio_mode = auto`，`backend/agent/tests/` 覆盖 agent 基础流程、API、工具注册、记忆（偏好/待办/笔记）、摘要压缩、会话归档、流式与降级、错误处理、prompt 组装等；根目录 `test_cancel.py` 验证流式取消。
- **依赖**：`fastapi`、`uvicorn[standard]`、`aiosqlite`、`openai`、`httpx`、`pydantic>=2`、`PyJWT[crypto]`、`python-dotenv`。
- **部署**：`Dockerfile`（暴露 8000）+ `docker-compose.yml`（端口映射 + `agent-data:/data` 卷）+ `railway.json`（`restart: NEVER`，健康检查关闭）；数据目录通过 `DATA_DIR` 环境变量适配 Railway Volume。

## 4. 设计亮点

1. **零框架自研 Agent**：不依赖 LangChain，直接对接 OpenAI 协议，可控性强、依赖轻。
2. **上下文压缩闭环**：token + 轮数双阈值、turn 对齐、增量摘要、失败降级、溢出防死锁，工程细节考虑周全。
3. **会话 / 记忆分层**：把「可丢弃的对话历史」和「持久的用户事实」用不同表、不同生命周期管理，是个人助理场景的正确建模。
4. **装饰器 + Pydantic 的工具注册**：新增工具成本极低，schema 自动生成且做过 OpenAI 兼容性清洗。
5. **测试驱动**：测试文件数几乎与业务模块数持平，核心路径（压缩、归档、降级、取消）都有对应用例。
6. **可观测性内建**：trace 贯穿 LLM/工具/摘要，前端能展示「思考过程 + 耗时 + token」。

## 5. 可改进方向

1. **缺少 README**：项目没有 README.md，建议补充「功能介绍 + 环境变量清单 + 本地启动 + API 说明」；本文件可作为起点。
2. **`main.py` 冗余**：`sys` / `Path` 重复 import 三次，且存在注释掉的 `public_key.pem` 写回逻辑，建议清理。
3. **重复的降级分支**：`chat_stream` 中 `compaction_degraded` 的判断在三个位置重复，可抽成 helper；`_estimate_tokens()` 同样被调用两次，可复用结果。
4. **run.py 体积偏大**：800 行且仍保留大段注释掉的 `_trim_messages` 旧实现，建议删除死代码并拆出 `compaction.py`。
5. **DB 连接管理**：每次请求新建 aiosqlite 连接，可考虑连接复用或轻量连接池。
6. **安全**：CORS 全开 + `index.html`/`chat.html` 内嵌 `fetch(..., credentials: 'include')`，生产环境需收紧跨域与密钥；`vision.py` 存在硬编码兜底地址（已注明「上线前删掉」）。
7. **文档化配置项**：建议在 `.env.example` 中枚举 `OPENAI_BASE_URL`、`MODEL`、`SUMMARY_MODEL`、`DATA_DIR`、`AUTH_ENABLED`、`PUBLIC_KEY`、`API_TOKEN` 等。

---

*本文档由代码静态分析生成，覆盖至 commit `a118758 fix: 前端图片压缩`。*

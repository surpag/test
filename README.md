# Person-Copilot 项目文档

## 📊 项目概况

| 项目 | 详情 |
|------|------|
| **项目名称** | Person-Copilot（个人助理 Agent） |
| **仓库地址** | https://github.com/surpag/test (main 分支) |
| **编程语言** | Python 3.11 (asyncio) |
| **代码规模** | ~40+ Python 文件 |
| **存储方式** | SQLite 单文件数据库 |

---

## ⚙️ 技术栈

| 类别 | 技术/库 |
|------|---------|
| **Web 框架** | FastAPI + uvicorn[standard] |
| **LLM 对接** | openai (AsyncOpenAI，支持任意兼容网关) |
| **数据库** | aiosqlite (异步 SQLite) |
| **鉴权** | PyJWT[crypto] + cryptography (RS256) |
| **HTTP 客户端** | httpx |
| **参数校验** | Pydantic v2+ |
| **测试框架** | pytest + pytest-asyncio + pytest-mock |
| **外部工具** | tavily-python (网络搜索) |
| **部署** | Docker / Railway |

---

## 🗂️ 项目结构

```text
backend/agent/                    # 核心业务逻辑
├── run.py                        # Copilot 对话循环 (~800行，核心)
├── prompt.py                     # System Prompt 组装
├── api/                          # FastAPI 服务层
│   ├── main.py                   # 路由 & 静态页面挂载
│   ├── auth.py                   # JWT RS256 鉴权
│   ├── vision.py                 # 多模态图片描述接口
│   ├── pool.py                   # Copilot 实例池 / 会话管理
│   └── schemas.py                # 请求/响应模型
├── tools/                        # 工具系统
│   ├── registry.py               # 装饰器 + 自动 JSON Schema
│   ├── memory_tools.py           # remember/forget 等
│   ├── note_tools.py             # save/get/list/search notes
│   ├── web_search.py             # 网络搜索
│   ├── fetch_url.py              # 网页抓取
│   ├── weather.py                # 天气查询
│   └── context.py                # 请求级上下文
├── memory/                       # 长期记忆层 (跨会话，永不压缩)
│   ├── preferences.py            # 用户偏好存储
│   ├── todos.py                  # 待办事项
│   └── notes.py                  # 笔记存储
├── session/                      # 会话存储层 (可压缩/归档)
│   ├── store.py                  # Session 管理
│   ├── trace.py                  # 调用链追踪 (可观测性)
│   └── trace_viewer.py           # Trace 查看器
├── cli/                          # 命令行工具
├── static/                       # 前端单页应用
├── tests/                        # 20 个 pytest 测试文件
└── agent/api/cli/memory/static/tests/
test_cancel.py                    # 流式取消测试
Dockerfile                        # 容器化部署
docker-compose.yml                # Docker Compose 编排
railway.json                      # Railway 云平台配置
requirements.txt                  # Python 依赖
pytest.ini                        # pytest 配置
.gitignore                        # Git 忽略规则
```

---

## 🔑 核心功能

### 1. 对话引擎 (`run.py`)
- **基于 LLM Tool-Calling 的标准循环**：模型返回 tool_calls → 执行工具 → 回填结果 → 继续请求 → 产出纯文本答案。
- **上下文压缩 (Compaction)** 是亮点：
  - Token + 轮数双阈值判断。
  - 按 turn 对齐保留最近消息。
  - 支持增量摘要 (已有摘要 + 新历史 → 更新摘要)。
  - 失败降级、溢出防死锁。

### 2. 双层存储架构
| 层级 | 数据表 | 生命周期 | 说明 |
|------|--------|----------|------|
| **会话层** | sessions, messages, summaries | 可压缩/归档 | 单会话历史 |
| **记忆层** | preferences, todos, notes | 永不压缩 | 跨会话持久记忆 |

### 3. 工具注册系统 (`tools/registry.py`)
- `@tool` 装饰器 + Pydantic 参数模型。
- 自动生成 OpenAI function-calling JSON Schema。
- 已注册 10+ 种工具：记忆、待办、笔记、搜索、网页抓取、天气等。

### 4. SSE 流式输出
- FastAPI StreamingResponse 推送 SSE 事件。
- 事件类型：status, thinking, token, tool_call, tool_result, trace, done, error。
- 支持**流式取消** (前端「停止按钮」中断 LLM 请求)。

### 5. 可观测性 (`trace.py`)
- 记录每个环节 (LLM 调用、工具调用、摘要) 的输入输出、耗时、token 用量。
- trace 通过 SSE 推送到前端展示。

---

## ✨ 设计亮点

1. **零框架自研 Agent**：不依赖 LangChain/LlamaIndex，直接对接 OpenAI 协议，可控性强、依赖轻。
2. **上下文压缩闭环**：双阈值 + turn 对齐 + 增量摘要 + 失败降级 + 防死锁，工程细节考虑周全。
3. **会话/记忆分层建模**：把「可丢弃的对话历史」和「持久的用户事实」分开管理。
4. **测试驱动开发**：测试文件数几乎与业务模块持平，核心路径都有对应用例。
5. **可观测性内建**：trace 贯穿整个调用链路。

---

## ⚠️ 可改进方向

| 优先级 | 问题 | 建议 |
|--------|------|------|
| 🔴 高 | **缺少 README** | 补充功能介绍、环境变量清单、启动指南、API 文档 (本文件即为弥补) |
| 🟡 中 | `run.py` 体积偏大 (~800行) | 删除死代码，将 compaction 逻辑拆出独立文件 |
| 🟡 中 | DB 连接管理 | 每次请求新建 aiosqlite 连接，可考虑复用或连接池 |
| 🟡 中 | CORS 全开 + 硬编码兜底地址 | 生产环境收紧跨域策略，删除 `vision.py` 中的硬编码地址 |
| 🟢 低 | 重复的降级分支 | 将 `compaction_degraded` 判断抽成 helper 函数 |
| 🟢 低 | 缺少 `.env.example` | 枚举所有配置项 (MODEL, SUMMARY_MODEL, DATA_DIR 等) |

---

## 📝 总结

这是一个**精心设计的个人助理 Agent 后端项目**，采用"小而美"的架构风格：
- ✅ **不依赖重型框架**，直接对接 OpenAI API。
- ✅ **有完整的测试覆盖**和**可观测性**设计。
- ✅ **上下文压缩**和**双层存储**体现了对 AI 应用特性的深入理解。
- ⚠️ 部分代码需要**清理和维护** (大文件、重复逻辑、硬编码)。

适合用于学习 LLM Agent 架构设计、FastAPI 流式 API 实现、SQLite 异步操作等场景。

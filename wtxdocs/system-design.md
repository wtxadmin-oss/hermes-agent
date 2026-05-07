# Hermes Agent 架构分析

---

## 1. 整体架构设计

Hermes Agent 采用**分层 + 插件化 + 多入口**的混合架构，核心是一个可扩展的 AI Agent 对话引擎，向上对接多种用户界面（CLI、消息平台、编辑器），向下通过适配器层对接多个 LLM 提供商。

```
┌──────────────────────────────────────────────────────┐
│                    用户交互层                         │
│   CLI (cli.py)  │  Gateway (gateway/)  │  ACP (acp_adapter/)  │
└─────────────────────────┬────────────────────────────┘
                          │
┌─────────────────────────▼────────────────────────────┐
│                  核心 Agent 层                        │
│              AIAgent (run_agent.py)                   │
│   对话循环 · 工具调用编排 · 上下文管理 · 预算跟踪    │
└─────────────────────────┬────────────────────────────┘
                          │
┌─────────────────────────▼────────────────────────────┐
│                   工具系统层                          │
│  model_tools.py · toolsets.py · tools/ (76 个工具)   │
└─────────────────────────┬────────────────────────────┘
                          │
┌─────────────────────────▼────────────────────────────┐
│                  提供商适配层                         │
│  Anthropic · OpenAI/OpenRouter · Bedrock · Gemini    │
└─────────────────────────┬────────────────────────────┘
                          │
┌─────────────────────────▼────────────────────────────┐
│                  存储与配置层                         │
│  SQLite FTS5 (hermes_state.py) · YAML · .env         │
└──────────────────────────────────────────────────────┘
```

---

## 2. 目录结构与分层说明

| 目录/文件 | 层级 | 说明 |
|---|---|---|
| `cli.py` | 交互层 | 交互式 TUI 主界面，命令处理 |
| `gateway/` | 交互层 | 多平台消息网关（36 个平台适配器） |
| `acp_adapter/` | 交互层 | VS Code / Zed / JetBrains 编辑器集成 |
| `run_agent.py` | 核心层 | AIAgent 类，对话循环与工具编排 |
| `agent/` | 核心层 | 提供商适配、内存管理、上下文压缩、提示构建 |
| `tools/` | 工具层 | 76 个工具模块，自动注册机制 |
| `model_tools.py` | 工具层 | 工具发现与调度入口 |
| `toolsets.py` | 工具层 | 工具集分组定义 |
| `plugins/` | 扩展层 | 可插拔功能（内存、上下文引擎、模型提供商、图像生成） |
| `skills/` | 扩展层 | 27 个内置技能（程序化记忆） |
| `cron/` | 扩展层 | 定时任务调度器 |
| `hermes_state.py` | 存储层 | SQLite 会话存储，FTS5 全文搜索 |
| `hermes_constants.py` | 基础层 | 路径解析、环境检测、全局常量 |
| `hermes_logging.py` | 基础层 | 集中日志（agent.log / errors.log / gateway.log） |
| `hermes_cli/` | CLI 子系统 | 子命令、设置向导、皮肤引擎 |
| `environments/` | 训练层 | RL 训练环境（Atropos 集成） |

---

## 3. 核心模块及职责

**AIAgent（run_agent.py）**
项目最核心的类，负责完整的对话循环：接收用户消息 → 构建提示 → 调用 LLM → 解析工具调用 → 执行工具 → 收集结果 → 循环直到完成。支持最大迭代次数（默认 90）和 token 预算控制。

**agent/ 子系统**
- `anthropic_adapter.py` / `auxiliary_client.py` / `bedrock_adapter.py` / `gemini_*_adapter.py`：各 LLM 提供商的统一适配
- `memory_manager.py`：跨会话记忆加载
- `context_compressor.py`：上下文压缩（防止超出 context window）
- `prompt_builder.py`：系统提示动态构建

**tools/ + model_tools.py**
工具采用自动注册机制：每个工具模块在导入时调用 `registry.register()`，`model_tools.py` 触发全量导入完成工具发现，再由 AIAgent 按需调度。

**gateway/**
统一消息网关，内含 36 个平台适配器（Telegram、Discord、Slack、WhatsApp、Signal、Matrix、Teams 等），通过统一的 `session.py` 管理跨平台会话。

**plugins/**
插件发现系统独立于主流程，支持热插拔：内存提供商（honcho、mem0 等）、上下文引擎、模型提供商、图像生成后端均通过此机制扩展。

---

## 4. 模块依赖与调用关系

```
hermes_constants.py          ← 无依赖，被所有模块引用
       ↑
hermes_logging.py / hermes_state.py / hermes_time.py
       ↑
run_agent.py / cli.py / gateway/run.py
       ↑
hermes_cli/main.py（CLI 入口）
gateway/run.py（网关入口）

tools/registry.py            ← 无依赖，工具注册中心
       ↑
tools/*.py                   ← 导入时自动注册
       ↑
model_tools.py               ← 触发工具发现
       ↑
run_agent.py

plugins/                     ← 独立发现系统
       ↑
run_agent.py                 ← 启动时加载插件
```

---

## 5. 主要数据流

**对话主流程：**

```
用户输入
  → CLI / Gateway 接收
  → AIAgent.run_conversation()
      ├─ prompt_builder 构建系统提示
      ├─ memory_manager 加载历史上下文
      ├─ *_adapter 调用 LLM API（流式）
      ├─ 解析工具调用请求
      ├─ model_tools.handle_function_call 执行工具
      ├─ 将工具结果追加到对话历史
      └─ 循环，直到 LLM 不再发起工具调用
  → hermes_state.py 持久化会话到 SQLite
  → 返回响应
```

**会话存储结构（SQLite）：**

| 表 | 内容 |
|---|---|
| `sessions` | 会话元数据、模型配置、成本追踪 |
| `messages` | 完整消息历史、推理内容 |
| `state_meta` | 全局元数据 |
| FTS5 虚拟表 | 全文搜索索引 |

---

## 6. 技术栈与框架作用

| 技术 | 用途 |
|---|---|
| Python 3.11+ asyncio | 异步 I/O，支持并发工具调用和流式输出 |
| OpenAI SDK / Anthropic SDK | LLM API 调用（OpenAI 兼容接口统一多提供商） |
| prompt_toolkit + Rich | 交互式 TUI，多行编辑、语法高亮、流式渲染 |
| SQLite WAL + FTS5 | 会话持久化与全文搜索 |
| PyYAML + python-dotenv | 配置文件与环境变量管理 |
| croniter | cron 表达式解析，定时任务调度 |
| httpx / requests | HTTP 工具调用（网页抓取、API 请求） |
| Docker / docker-compose | 容器化部署，终端沙箱隔离 |
| Node.js / Ink (React) | 终端 TUI 前端渲染层 |
| ACP 协议 | 编辑器集成（VS Code / Zed / JetBrains） |

---

## 7. 架构优点

- **多入口统一核心**：CLI、消息网关、编辑器插件共享同一个 AIAgent 核心，逻辑不重复。
- **提供商无关**：通过适配器层屏蔽各 LLM API 差异，切换模型无需改业务代码。
- **工具自动注册**：工具模块解耦，新增工具只需放入 `tools/` 目录，无需修改调度逻辑。
- **插件化扩展**：内存、上下文引擎、图像生成等均可通过插件替换，不侵入核心。
- **SQLite FTS5**：轻量级本地存储，同时支持全文搜索，无需外部数据库依赖。
- **多终端后端**：本地、Docker、SSH、Modal、Daytona 等后端统一接口，适应不同部署场景。

---

## 8. 潜在问题与优化建议

**1. 核心文件过于庞大**
`run_agent.py`（~12k LOC）和 `cli.py`（~11k LOC）承担了过多职责，维护和测试成本高。建议按职责拆分为更小的模块（如对话管理、工具编排、流式处理分离）。

**2. 工具数量膨胀风险**
76 个工具在启动时全量导入注册，随工具增多会拖慢启动速度。可考虑懒加载机制，按需导入工具模块。

**3. 网关模块体积异常**
`gateway/run.py` 达到 705KB，单文件过大，可读性和可维护性差，建议按平台拆分。

**4. 插件系统文档缺失**
插件发现机制对外部开发者不透明，缺乏标准接口定义，建议补充插件开发规范。

**5. 跨机器记忆不同步**
自动记忆（`~/.claude/projects/`）是机器本地的，多设备协作场景下无法共享，可考虑可选的云同步后端。

**6. RL 训练环境耦合**
`environments/` 目录将强化学习训练逻辑混入主项目，建议作为独立子包或 git submodule 管理，避免污染核心依赖树。

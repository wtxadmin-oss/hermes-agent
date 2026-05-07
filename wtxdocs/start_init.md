# Hermes Agent 启动与环境配置

## 运行环境要求

- Python 3.11+
- Node.js 20+（TUI 界面依赖）
- Git
- Linux / macOS / WSL2（不支持原生 Windows）

## 依赖安装

项目使用 `venv` 虚拟环境，已在 `venv/` 目录下预装所有依赖。

如需全新安装：

```bash
python3 -m venv venv
source venv/bin/activate
pip install -e ".[all]"
```

核心依赖（自动安装）：

| 包 | 用途 |
|---|---|
| openai | LLM API 调用（OpenAI 兼容接口） |
| anthropic | Claude API 支持 |
| python-dotenv | 环境变量加载 |
| rich | 终端 UI 渲染 |
| prompt_toolkit | 交互式 CLI |
| pyyaml | 配置文件解析 |
| croniter | 定时任务调度 |
| python-telegram-bot | Telegram 集成（可选） |

## 环境变量配置

复制示例文件并填写对应的 API Key：

```bash
cp .env.example .env
```

`.env` 中需要配置的关键变量：

```bash
# VPN 代理（如需要）
VPN_PORT=7897
HTTP_PROXY=http://host.docker.internal:7897
HTTPS_PROXY=http://host.docker.internal:7897

# LLM 提供商（二选一或多选）
# 阿里云 DashScope / 通义千问
DASHSCOPE_API_KEY=<your_dashscope_api_key>
DASHSCOPE_BASE_URL=https://dashscope.aliyuncs.com/compatible-mode/v1

# OpenRouter（可访问 200+ 模型）
OPENROUTER_API_KEY=<your_openrouter_api_key>

# Claude Code CLI（如使用第三方代理）
ANTHROPIC_AUTH_TOKEN=<your_anthropic_token>
ANTHROPIC_BASE_URL=<your_anthropic_base_url>
CLAUDE_MODEL=sonnet
```

其他可选 API Key（按需填写）：

- `EXA_API_KEY` — 网页搜索工具（exa.ai）
- `FIRECRAWL_API_KEY` — 网页抓取工具（firecrawl.dev）
- `TELEGRAM_BOT_TOKEN` — Telegram 机器人集成
- `SLACK_BOT_TOKEN` — Slack 集成
- `FAL_KEY` — 图像生成（fal.ai）
- `GITHUB_TOKEN` — Skills Hub 技能搜索/安装

## 模型配置

模型配置在 `~/.hermes/config.yaml` 中，当前默认：

```yaml
model:
  default: "qwen3.6-35b-a3b"
  provider: "alibaba"
```

切换模型：

```bash
hermes model
```

## 启动方式

### 方式一：交互式 CLI（推荐）

```bash
cd /workspaces/hermes-agent
source .env
hermes
```

### 方式二：激活 venv 后启动

```bash
cd /workspaces/hermes-agent
source venv/bin/activate
source .env
hermes
```

### 方式三：消息网关（Telegram / Slack 等）

```bash
source .env
hermes gateway start
```

## 常用命令

```bash
hermes              # 启动交互式 CLI
hermes model        # 切换 LLM 提供商和模型
hermes tools        # 配置启用的工具
hermes setup        # 运行完整设置向导
hermes doctor       # 诊断环境问题
hermes gateway      # 启动消息网关
hermes update       # 更新到最新版本
```

## 健康检查

```bash
source .env && hermes doctor
```

正常输出应显示：
- Python 环境 ✓
- 必要包已安装 ✓
- `.env` 文件存在 ✓
- `~/.hermes/config.yaml` 存在 ✓

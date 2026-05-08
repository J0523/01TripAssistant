# 01TripAssistant
# 差旅出行助手Multi-Agent系统

基于 **大语言模型** 与 **AgentScope** 的多智能体差旅助手：用语义意图识别驱动 Plan-and-Execute 编排，集成 RAG 企业知识、联网搜索与长期/短期记忆，在终端通过 CLI 交互。

---

## 项目简介

- **目标**：理解自然语言差旅诉求，完成政策问答、行程要素收集、偏好记忆、实时信息查询与行程规划等任务。
- **特点**：
  - 多意图路由（行程规划、记忆查询、偏好、RAG 问答、联网查询、事项收集等）
  - **Skill 插件化**：子能力位于 `.claude/skills/`，由编排层按需加载
  - **RAG**：Milvus Lite + 本地中文 Embedding（`data/models/bge-small-zh-v1.5`）
  - **韧性**：LLM 调用重试、熔断与健康检查（见 `config.py` 中 `RESILIENCE_CONFIG`）

当前仓库以 **命令行（`cli.py`）** 为主入口；长期记忆默认使用 **`data/memory/{user_id}.json`** 文件存储（生产环境可替换为数据库实现）。

---

## 架构概览

核心数据流：**用户输入 → 意图识别 → 编排调度 → 多 Skill 执行 → 结果聚合与记忆更新**。

```mermaid
flowchart TB
  U[用户输入]
  I[IntentionAgent 意图识别]
  O[OrchestrationAgent 编排]
  S1[Memory / Preference / Event / Query / RAG Skills]
  S2[ItineraryPlanning Skill]
  M[(短期记忆 + 长期记忆 JSON)]
  U --> I --> O
  O --> S1
  S1 --> S2
  O --> M
  S2 --> M
```

- **`agents/intention_agent.py`**：解析用户意图并生成调度计划（JSON）。
- **`agents/orchestration_agent.py`**：按优先级调度子智能体；同优先级可并行（`asyncio.gather`）。
- **`agents/lazy_agent_registry.py`**：Skill 发现与懒加载。
- **`context/memory_manager.py`**：短期上下文与长期偏好、行程等持久化接口。

更细的模块说明与扩展路线可参考仓库内 `README原版.md`。

---

## 环境要求

- Python 3.10+（建议与本地已验证版本一致）
- 可访问的大模型 HTTP API（在 `config.py` 的 `LLM_CONFIG` 中配置）
- 首次使用 RAG 前需初始化向量库（见下文）

依赖安装：

```bash
pip install -r requirements.txt
```

---

## 配置

编辑项目根目录 **`config.py`**：

| 配置块 | 说明 |
|--------|------|
| `LLM_CONFIG` | `api_key`、`model_name`、`base_url`、`temperature`、`max_tokens` |
| `SYSTEM_CONFIG` | 超时、日志级别等 |
| `RAG_CONFIG` | 本地 Embedding 模型路径（默认 `data/models/bge-small-zh-v1.5`） |
| `RESILIENCE_CONFIG` | 重试、熔断阈值、健康检查超时 |

请勿将真实密钥提交到公开仓库；可使用环境变量或本地未跟踪配置文件（按团队规范自行封装）。

---

## 启动方式

### 1. 初始化 RAG 知识库（首次必做）

```bash
python .claude/skills/ask-question/script/init_knowledge_base.py
```

生成数据默认位于 `.claude/skills/ask-question/data/`（含 Milvus Lite 库文件）。

### 2. 启动交互式 CLI

```bash
python cli.py
```

启动后按提示输入 **用户 ID**（默认 `default_user`），随后可直接用自然语言对话。

### 3. 内置命令（CLI）

| 命令 | 说明 |
|------|------|
| `help` | 帮助 |
| `status` | 当前状态与记忆摘要 |
| `health` | LLM 可达性与熔断状态 |
| `clear` | 清空当前任务上下文（保留长期记忆） |
| `history` / `preferences` | 查看行程历史 / 偏好 |
| `exit` | 退出 |

**仅健康检查（非交互，适合脚本/监控）**：

```bash
python cli.py health
```

成功时退出码为 `0`，失败为 `1`。

---

## API 示例

本仓库**未提供 HTTP REST 服务**；对外“接口”形态为 **CLI 自然语言** 与 **Python 异步调用链**。

### 1. CLI 侧（自然语言）

在 `python cli.py` 交互中直接输入，例如：

- RAG：`出差住宿标准是多少？`
- 规划：`我下周从北京去杭州出差三天，帮我安排行程。`
- 偏好：`我喜欢住华住会旗下酒店，常坐国航，请记住。`
- 联网：`杭州明天天气怎么样？`
- 记忆：`我之前说过哪些出行偏好？`

### 2. 编程调用（与测试脚本一致）

编排主干与集成测试 `tests/test_cli_qa.py` 相同：**意图 → 编排**，返回的 `orchestration_result.content` 为 **JSON 字符串**，可再解析或交给 CLI 的展示逻辑。

```python
import asyncio
import json
from agentscope.message import Msg
from agentscope.model import OpenAIChatModel
from config import LLM_CONFIG
from config_agentscope import init_agentscope
from context.memory_manager import MemoryManager
from agents.intention_agent import IntentionAgent
from agents.orchestration_agent import OrchestrationAgent
from agents.lazy_agent_registry import LazyAgentRegistry


async def run_once(user_text: str) -> dict:
    init_agentscope()
    model = OpenAIChatModel(
        model_name=LLM_CONFIG["model_name"],
        api_key=LLM_CONFIG["api_key"],
        client_kwargs={"base_url": LLM_CONFIG["base_url"]},
        temperature=LLM_CONFIG.get("temperature", 0.7),
        max_tokens=LLM_CONFIG.get("max_tokens", 2000),
    )
    memory_manager = MemoryManager(
        user_id="api_demo_user",
        session_id="session_1",
        llm_model=model,
    )
    intention_agent = IntentionAgent(name="IntentionAgent", model=model)
    registry = LazyAgentRegistry(model, {}, memory_manager)
    orchestrator = OrchestrationAgent(
        name="OrchestrationAgent",
        agent_registry=registry,
        memory_manager=memory_manager,
    )

    ctx = [Msg(name="user", content=user_text, role="user")]
    intention_msg = await intention_agent.reply(ctx)
    out_msg = await orchestrator.reply(intention_msg)
    return json.loads(out_msg.content)


if __name__ == "__main__":
    result = asyncio.run(run_once("从北京到上海出差两天，列出大致行程要点"))
    print(json.dumps(result, ensure_ascii=False, indent=2))
```

若在自有应用中嵌入 CLI 已有会话与熔断逻辑，可使用 **`AligoCLI`** 的 **`initialize_system()`** 与 **`process_query(user_input)`**（见 `cli.py`），行为与终端一致。

---

## 评测方式

### 端到端集成评测（推荐）

覆盖多条预设问题并生成 Markdown 报告（含耗时与成功率）：

```bash
python tests/test_cli_qa.py
```

报告输出目录：`tests/results/`，文件名形如 `qa_test_YYYYMMDD_HHMMSS.md`。

### 模块与单元测试

按需单独运行：

```bash
python tests/test_memory_system.py
python tests/test_intention_agent.py
python tests/test_orchestration.py
python tests/test_rag_agent.py
python tests/test_event_collection_agent.py
python tests/test_information_query_agent.py
```

**说明**：意图识别、规划与 RAG 等测试依赖 **真实 LLM 与网络（如联网搜索）**，结果可能随模型与外部环境波动；评测时宜固定模型版本并记录时间戳。

---

## 项目结构（节选）

```
.
├── agents/                    # 意图识别、编排、Skill 注册
├── context/                   # MemoryManager、短/长期记忆实现
├── utils/                     # Skill 加载、熔断、LLM 韧性等
├── .claude/skills/            # 各 Skill（问答、规划、偏好…）
├── data/
│   ├── memory/                # 长期记忆 JSON（按 user_id）
│   └── models/                # 本地 Embedding 模型
├── tests/                     # 测试与 QA 报告输出
├── cli.py                     # 主程序入口
├── config.py                  # 全局配置
├── config_agentscope.py       # AgentScope 初始化
└── requirements.txt
```

---

## 许可证

如无另行声明，以仓库根目录许可证文件为准；历史文档中曾标注 MIT，添加 `LICENSE` 文件后即可与之对齐。

# Aligo 多Agent系统架构设计文档

## 核心设计思想与代码实现

本文档详细阐述系统采用 **AgentScope Actor模型** 而非 LangGraph 的三大核心设计思想，以及这些思想在代码中的具体体现。

---

## 一、消息传递架构 vs 全局状态管理

### 设计思想

**为什么选择 AgentScope 的消息传递？**

- **清晰的数据流向**：每个Agent通过 `Msg` 对象接收结构化输入，返回结构化输出
- **易于调试**：可以直观看到每条消息的内容、来源、目标
- **6个子Agent的协作**：频繁传递意图识别结果、调度决策、上下文信息
- **LangGraph的问题**：全局State容易变得臃肿，Agent数量多时难以维护

### 代码实现

#### 1. Msg 对象的结构化传递

**文件**: `agents/orchestration_agent.py` (第 95-110 行)

```python
# 构建输入消息 - 结构化数据传递
input_msg = Msg(
    name="Orchestrator",
    content=json.dumps({
        "context": context,
        "reason": reason,
        "expected_output": expected_output,
        "previous_results": previous_results
    }, ensure_ascii=False),
    role="user"
)

# 调用智能体
response = await agent.reply(input_msg)
```

**关键点**：
- 每个 `Msg` 包含明确的 `name`（发送者）、`content`（结构化JSON）、`role`（角色）
- 子Agent接收到的是完整的上下文信息，而不是全局State的片段
- 便于追踪数据流向：Orchestrator → SubAgent → 结果

#### 2. 意图识别结果的消息传递

**文件**: `agents/intention_agent.py` (第 120-130 行)

```python
# IntentionAgent 返回结构化的意图识别结果
return Msg(
    name=self.name, 
    content=json.dumps(result, ensure_ascii=False),  # 结构化JSON
    role="assistant"
)
```

**结构化内容示例**：
```json
{
    "reasoning": "推理过程...",
    "intents": [
        {
            "type": "itinerary_planning",
            "confidence": 0.95,
            "reason": "用户明确说要规划行程"
        }
    ],
    "key_entities": {
        "origin": "上海",
        "destination": "北京",
        "date": "2月27日"
    },
    "agent_schedule": [
        {
            "agent_name": "event_collection",
            "priority": 1,
            "reason": "收集行程基本信息"
        }
    ]
}
```

#### 3. 多Agent间的消息流向

**文件**: `cli.py` (第 180-220 行)

```python
# 完整的消息流向链路
context_messages = []
if long_term_summary:
    context_messages.append(Msg(name="system", content=long_term_summary, role="system"))
for msg in recent_context:
    context_messages.append(Msg(name=msg["role"], content=msg["content"], role=msg["role"]))
context_messages.append(Msg(name="user", content=user_input, role="user"))

# 1. 意图识别 - 接收消息列表
intention_result = await self.intention_agent.reply(context_messages)

# 2. 协调执行 - 接收意图识别结果
orchestration_result = await self.orchestrator.reply(intention_result)

# 3. 结果聚合 - 解析最终结果
result_data = json.loads(orchestration_result.content)
```

**消息流向图**：
```
用户输入 → [系统记忆, 对话历史, 用户Query] 
  ↓
IntentionAgent.reply(Msg列表)
  ↓
意图识别结果 Msg (包含agent_schedule)
  ↓
OrchestrationAgent.reply(意图结果Msg)
  ↓
子Agent并行执行 (每个接收独立的Msg)
  ↓
结果聚合 Msg
```

#### 4. 与LangGraph的对比

| 特性 | AgentScope Msg | LangGraph State |
|------|---|---|
| 数据结构 | 明确的Msg对象，包含name/content/role | 全局State字典，所有Agent共享 |
| 调试可视性 | 每条Msg可独立查看和追踪 | State变化难以追踪，容易混乱 |
| 6个Agent协作 | 清晰的消息传递链路 | State容易变得臃肿（6个Agent的输出都混在一起） |
| 扩展性 | 新增Agent只需定义输入/输出Msg格式 | 需要修改全局State Schema |

---

## 二、插件化架构与懒加载注册器

### 设计思想

**为什么选择插件化架构？**

- **即插即用**：新增功能只需写一个继承 `AgentBase` 的类，无需修改核心代码
- **动态发现**：`LazyAgentRegistry` 自动扫描 `.claude/skills/` 目录，发现并注册Agent
- **懒加载优化**：未使用的Skill不加载，系统启动速度从未优化 → 3秒
- **LangGraph的问题**：节点需要手动定义，做不到即插即用

### 代码实现

#### 1. LazyAgentRegistry 的动态发现机制

**文件**: `agents/lazy_agent_registry.py` (第 30-50 行)

```python
def _discover_skills(self):
    """扫描 .claude/skills 目录寻找可用的 Agent"""
    if not self.skills_root.exists():
        return

    count = 0
    for skill_dir in self.skills_root.iterdir():
        if not skill_dir.is_dir():
            continue
        
        # 查找 script/agent.py
        agent_script = skill_dir / "script" / "agent.py"
        if agent_script.exists():
            skill_name = skill_dir.name
            self._skill_map[skill_name] = agent_script
            count += 1
```

**关键点**：
- 自动扫描 `.claude/skills/` 下的所有目录
- 查找 `script/agent.py` 文件
- 建立 `skill_name → agent_script_path` 的映射表
- 无需手动注册，完全自动化

#### 2. 动态加载与实例化

**文件**: `agents/lazy_agent_registry.py` (第 70-120 行)

```python
def __getitem__(self, agent_name: str):
    """获取智能体 (懒加载)"""
    if agent_name in self.cache:
        return self.cache[agent_name]

    skill_name = self._resolve_agent_name(agent_name)
    if not skill_name:
        raise KeyError(f"Agent '{agent_name}' not found")

    script_path = self._skill_map[skill_name]
    
    # 1. 动态加载模块
    module_name = f"skills.{skill_name}.agent"
    spec = importlib.util.spec_from_file_location(module_name, script_path)
    module = importlib.util.module_from_spec(spec)
    sys.modules[module_name] = module
    spec.loader.exec_module(module)
    
    # 2. 查找 Agent 类
    agent_class = None
    for name, obj in inspect.getmembers(module):
        if inspect.isclass(obj) and issubclass(obj, AgentBase) and obj is not AgentBase:
            agent_class = obj
            break
    
    # 3. 实例化
    init_params = {
        "name": agent_name,
        "model": self.model,
    }
    
    # 检查是否需要 memory_manager
    sig = inspect.signature(agent_class.__init__)
    if "memory_manager" in sig.parameters:
        init_params["memory_manager"] = self.memory_manager
    
    agent_instance = agent_class(**init_params)
    
    # 4. 缓存
    self.cache[agent_name] = agent_instance
    return agent_instance
```

**关键点**：
- **动态导入**：使用 `importlib.util` 在运行时加载模块
- **自动发现类**：通过反射查找继承 `AgentBase` 的类
- **智能参数注入**：检查 `__init__` 签名，自动注入 `memory_manager`
- **缓存机制**：首次加载后缓存，避免重复加载

#### 3. 插件化的Agent结构

**文件**: `.claude/skills/event-collection/script/agent.py`

```python
from agentscope.agent import AgentBase
from agentscope.message import Msg

class EventCollectionAgent(AgentBase):
    """事项收集智能体 - 插件化实现"""
    
    def __init__(self, name: str = "EventCollectionAgent", model=None, **kwargs):
        super().__init__()
        self.name = name
        self.model = model
    
    async def reply(self, x: Optional[Union[Msg, List[Msg]]] = None) -> Msg:
        """处理消息并返回结果"""
        # 业务逻辑...
        return Msg(name=self.name, content=json.dumps(result), role="assistant")
```

**插件化特点**：
- 所有Agent都继承 `AgentBase`，统一暴露 `reply()` 方法
- 独立的 `script/agent.py` 文件
- 可以有自己的 `data/` 目录（如RAG知识库）
- 新增Agent只需复制目录结构，无需修改核心代码

#### 4. 在CLI中的使用

**文件**: `cli.py` (第 110-130 行)

```python
# 使用懒加载注册器
from agents.lazy_agent_registry import LazyAgentRegistry

self._agent_cache = {}
lazy_registry = LazyAgentRegistry(
    model=self.model, 
    cache=self._agent_cache,
    memory_manager=self.memory_manager
)

# 初始化协调器，传入注册器
self.orchestrator = OrchestrationAgent(
    name="OrchestrationAgent",
    agent_registry=lazy_registry,
    memory_manager=self.memory_manager
)
```

**使用流程**：
1. 创建 `LazyAgentRegistry` 实例
2. 传入协调器
3. 协调器需要某个Agent时，通过 `registry[agent_name]` 获取
4. 注册器自动加载、缓存、返回

#### 5. 与LangGraph的对比

| 特性 | AgentScope插件化 | LangGraph |
|------|---|---|
| 新增Agent | 创建目录 + 继承AgentBase | 手动定义节点 + 修改图结构 |
| 发现机制 | 自动扫描 `.claude/skills/` | 需要手动注册 |
| 加载时机 | 懒加载（首次使用时） | 启动时全部加载 |
| 启动速度 | 3秒（只扫描元数据） | 较慢（加载所有节点） |
| 扩展性 | 极高（即插即用） | 需要修改核心代码 |

---

## 三、异步支持与优先级并行调度

### 设计思想

**为什么选择异步并行调度？**

- **混合调度策略**：同优先级Agent并行执行，不同优先级串行依赖
- **动态调度**：根据意图识别结果实时决定哪些Agent并行
- **性能提升**：响应时间从30秒 → 15秒（-50%）
- **LangGraph的问题**：并行需要在图结构中预定义，运行时难以动态调整

### 代码实现

#### 1. 优先级分组与并行执行

**文件**: `agents/orchestration_agent.py` (第 65-95 行)

```python
async def reply(self, x: Optional[Union[Msg, List[Msg]]] = None) -> Msg:
    """协调执行流程"""
    # 获取智能体调度计划
    agent_schedule = intention_data.get("agent_schedule", [])
    
    # 按优先级排序
    sorted_schedule = sorted(agent_schedule, key=lambda x: x.get("priority", 999))
    
    # 并行执行智能体（按优先级分组）
    results = []
    current_priority = None
    parallel_tasks = []

    for task in sorted_schedule:
        priority = task.get("priority", 0)

        # 如果优先级变化，先执行当前批次
        if current_priority is not None and priority != current_priority:
            # 并行执行当前优先级的所有任务
            if parallel_tasks:
                batch_results = await self._execute_parallel_agents(
                    parallel_tasks, context, results
                )
                results.extend(batch_results)
                parallel_tasks = []

        current_priority = priority
        parallel_tasks.append(task)

    # 执行最后一批
    if parallel_tasks:
        batch_results = await self._execute_parallel_agents(
            parallel_tasks, context, results
        )
        results.extend(batch_results)
```

**关键点**：
- 按优先级排序任务
- 同优先级的任务放入 `parallel_tasks` 列表
- 优先级变化时，执行当前批次的并行任务
- 不同优先级之间有依赖关系（前一批的结果作为后一批的输入）

#### 2. asyncio.gather 并行执行

**文件**: `agents/orchestration_agent.py` (第 130-180 行)

```python
async def _execute_parallel_agents(
    self,
    tasks: List[Dict],
    context: Dict[str, Any],
    previous_results: List[Dict]
) -> List[Dict]:
    """并行执行多个智能体"""
    if not tasks:
        return []

    # 如果只有一个任务，直接执行
    if len(tasks) == 1:
        task = tasks[0]
        result = await self._execute_agent(...)
        return [{"agent_name": task.get("agent_name"), "result": result}]

    # 多个任务并行执行
    logger.info(f"Executing {len(tasks)} agents in parallel")

    # 创建并行任务
    parallel_coroutines = []
    for task in tasks:
        agent_name = task.get("agent_name")
        priority = task.get("priority", 0)
        
        logger.info(f"Parallel executing agent: {agent_name} (priority={priority})")

        # 创建协程
        coroutine = self._execute_agent(
            agent_name=agent_name,
            context=context,
            reason=task.get("reason", ""),
            expected_output=task.get("expected_output", ""),
            previous_results=previous_results
        )
        parallel_coroutines.append((agent_name, priority, coroutine))

    # 使用 asyncio.gather 并行执行
    execution_results = await asyncio.gather(
        *[coro for _, _, coro in parallel_coroutines],
        return_exceptions=True
    )

    # 整理结果
    results = []
    for (agent_name, priority, _), exec_result in zip(parallel_coroutines, execution_results):
        if isinstance(exec_result, Exception):
            logger.error(f"Parallel agent execution failed: {agent_name}")
            result = {"status": "error", "agent_name": agent_name}
        else:
            result = exec_result

        results.append({
            "agent_name": agent_name,
            "priority": priority,
            "result": result
        })

    return results
```

**关键点**：
- 使用 `asyncio.gather()` 并发执行多个协程
- `return_exceptions=True` 确保一个Agent失败不影响其他Agent
- 保留每个Agent的执行结果，便于后续处理

#### 3. 意图识别中的优先级设置

**文件**: `agents/intention_agent.py` (第 80-120 行)

```python
# 意图识别Prompt中的优先级规则
prompt = f"""
【重要提示 - 优先级设置规则】
优先级数字相同的智能体会**并行执行**，不同优先级按顺序批次执行。

**所有智能体优先级分组：**

**Priority 1（并行执行）- 信息收集类：**
- memory_query: 记忆查询智能体
- event_collection: 事项收集智能体
- preference: 偏好管理智能体
- information_query: 信息查询智能体（联网搜索）
- rag_knowledge: RAG知识库智能体（查询企业知识库）

**Priority 2（依赖 Priority 1）- 行程规划类：**
- itinerary_planning: 行程规划智能体（需要事项收集的结果）

**说明：**
- Priority 1 的智能体都是信息获取，互不依赖，可并行执行提升速度
- Priority 2 的智能体需要使用 Priority 1 收集的信息
- 示例：用户说"我要从天津去北京，喜欢住汉庭"
  → Priority 1: preference + event_collection（并行）
  → Priority 2: itinerary_planning（使用 Priority 1 的结果）
"""
```

**优先级设计**：
- **Priority 1**：信息收集类（5个Agent）
  - `memory_query`：查询历史记忆
  - `event_collection`：收集行程信息
  - `preference`：管理用户偏好
  - `information_query`：联网搜索
  - `rag_knowledge`：知识库查询
  - 这些Agent互不依赖，可以并行执行

- **Priority 2**：行程规划类（1个Agent）
  - `itinerary_planning`：依赖Priority 1的结果
  - 需要等Priority 1完成后才能执行

#### 4. 异步调用链路

**文件**: `cli.py` (第 180-220 行)

```python
async def process_query(self, user_input: str):
    """处理用户查询（异步）"""
    
    # 1. 获取长期记忆摘要（异步）
    long_term_summary = await self._get_long_term_summary(user_input)
    
    # 2. 意图识别（异步 + 重试）
    intention_result = await retry_with_backoff(
        lambda: self.intention_agent.reply(context_messages),
        max_retries=max_retries,
        base_delay_sec=1.0,
        max_delay_sec=30.0,
    )
    
    # 3. 协调执行（异步 + 重试）
    orchestration_result = await retry_with_backoff(
        lambda: self.orchestrator.reply(intention_result),
        max_retries=max_retries,
        base_delay_sec=1.0,
        max_delay_sec=30.0,
    )
    
    # 4. 结果处理
    result_data = json.loads(orchestration_result.content)
```

**异步流程**：
1. 异步获取长期记忆摘要
2. 异步调用意图识别（带重试）
3. 异步调用协调器（带重试）
4. 协调器内部异步并行执行子Agent
5. 异步处理结果

#### 5. 性能对比

**响应时间优化**：

```
优化前（串行执行）：
Priority 1 Agent 1: 5秒
Priority 1 Agent 2: 5秒
Priority 1 Agent 3: 5秒
Priority 1 Agent 4: 5秒
Priority 1 Agent 5: 5秒
Priority 2 Agent 1: 5秒
总计：30秒

优化后（并行执行）：
Priority 1 (5个Agent并行): max(5, 5, 5, 5, 5) = 5秒
Priority 2 (1个Agent): 5秒
总计：10秒

实际测试：15秒（包括网络延迟、LLM推理等）
性能提升：-50%
```

#### 6. 与LangGraph的对比

| 特性 | AgentScope异步 | LangGraph |
|------|---|---|
| 并行执行 | asyncio.gather，动态分组 | 需要在图结构中预定义 |
| 动态调度 | 根据意图识别结果实时决定 | 运行时难以调整 |
| 优先级管理 | 灵活的优先级分组 | 固定的图结构 |
| 性能 | 15秒（-50%） | 30秒（串行） |
| 扩展性 | 新增Agent自动支持并行 | 需要修改图结构 |

---

## 四、三大设计思想的协同效应

### 整体架构流程

```
用户输入
   ↓
┌─────────────────────────────────────────────────────────┐
│ IntentionAgent (意图识别)                               │
│ - 接收 Msg 列表（系统记忆 + 对话历史 + 用户Query）      │
│ - 返回结构化 Msg（包含agent_schedule）                  │
│ - 动态加载 Skills Metadata (Progressive Disclosure)    │
└─────────────────────────────────────────────────────────┘
   ↓ (Msg传递)
┌─────────────────────────────────────────────────────────┐
│ OrchestrationAgent (协调器)                             │
│ - 解析意图识别结果 Msg                                  │
│ - 按优先级分组任务                                      │
│ - 通过 LazyAgentRegistry 动态加载子Agent                │
└─────────────────────────────────────────────────────────┘
   ↓ (Msg传递 + 并行执行)
┌─────────────────────────────────────────────────────────┐
│ Priority 1 (并行执行 - asyncio.gather)                 │
│ - MemoryQueryAgent (Msg输入/输出)                       │
│ - EventCollectionAgent (Msg输入/输出)                   │
│ - PreferenceAgent (Msg输入/输出)                        │
│ - InformationQueryAgent (Msg输入/输出)                  │
│ - RAGKnowledgeAgent (Msg输入/输出)                      │
└─────────────────────────────────────────────────────────┘
   ↓ (等待所有Priority 1完成)
┌─────────────────────────────────────────────────────────┐
│ Priority 2 (依赖Priority 1)                             │
│ - ItineraryPlanningAgent (Msg输入/输出)                 │
└─────────────────────────────────────────────────────────┘
   ↓ (Msg传递)
┌─────────────────────────────────────────────────────────┐
│ 结果聚合与记忆更新                                      │
│ - 聚合所有 Msg 结果                                     │
│ - 更新长期记忆                                          │
│ - 生成人性化回复                                        │
└─────────────────────────────────────────────────────────┘
   ↓
用户看到结果
```

### 三大思想的协同

1. **消息传递 + 插件化**：
   - 每个插件Agent都通过 `Msg` 接收输入、返回输出
   - 新增插件无需修改消息格式，只需遵循 `Msg` 协议

2. **消息传递 + 并行调度**：
   - 并行执行的Agent各自接收独立的 `Msg`
   - 结果通过 `Msg` 返回，便于聚合

3. **插件化 + 并行调度**：
   - `LazyAgentRegistry` 动态加载插件
   - 协调器根据意图识别结果动态决定哪些插件并行执行
   - 新增插件自动支持并行调度

---

## 五、关键代码位置速查表

| 设计思想 | 核心文件 | 关键代码行 | 说明 |
|---------|---------|----------|------|
| **消息传递** | `agents/orchestration_agent.py` | 95-110 | Msg构建与传递 |
| | `agents/intention_agent.py` | 120-130 | 结构化Msg返回 |
| | `cli.py` | 180-220 | 完整消息流向 |
| **插件化架构** | `agents/lazy_agent_registry.py` | 30-50 | 自动发现机制 |
| | `agents/lazy_agent_registry.py` | 70-120 | 动态加载与实例化 |
| | `cli.py` | 110-130 | 注册器使用 |
| **并行调度** | `agents/orchestration_agent.py` | 65-95 | 优先级分组 |
| | `agents/orchestration_agent.py` | 130-180 | asyncio.gather并行 |
| | `agents/intention_agent.py` | 80-120 | 优先级设置规则 |

---

## 六、总结

### 为什么这个架构优于LangGraph？

| 维度 | AgentScope | LangGraph |
|------|-----------|----------|
| **数据流向** | 清晰的Msg传递链路 | 全局State混乱 |
| **6个Agent协作** | 消息传递，易于调试 | State容易臃肿 |
| **新增功能** | 即插即用（LazyAgentRegistry） | 需要修改核心代码 |
| **启动速度** | 3秒（懒加载） | 较慢（全量加载） |
| **并行调度** | 动态分组（asyncio.gather） | 需要预定义图结构 |
| **性能** | 15秒（-50%） | 30秒（串行） |
| **可维护性** | 高（清晰的架构） | 低（复杂的图结构） |

### 核心优势

1. **消息传递**：每个Agent的输入输出都很清晰，调试时能直观看到整个数据流向
2. **插件化**：新增功能只需写一个继承AgentBase的类，无需修改核心代码
3. **异步并行**：同优先级Agent并行执行，响应时间从30秒降到15秒

这三个设计思想相互配合，形成了一个高效、可维护、易于扩展的多Agent系统。


# TripAssistant 架构设计 - 实际运行示例与可视化

本文档通过具体的用户场景，展示三大设计思想在实际运行中的体现。

---

## 场景一：行程规划请求

### 用户输入
```
"我要从上海去北京出差，喜欢住汉庭酒店"
```

### 完整执行流程

#### 第1步：消息传递 - IntentionAgent 意图识别

**输入 Msg 列表**：
```python
[
    Msg(name="system", content="【用户背景信息】...", role="system"),
    Msg(name="user", content="我要从上海去北京出差，喜欢住汉庭酒店", role="user")
]
```

**IntentionAgent 处理**：
- 接收 Msg 列表
- 提取用户Query：`"我要从上海去北京出差，喜欢住汉庭酒店"`
- 分析意图：识别出 2 个意图
  - `itinerary_planning`（规划行程）
  - `preference`（管理偏好）

**输出 Msg**：
```python
Msg(
    name="IntentionAgent",
    content=json.dumps({
        "reasoning": "用户明确表达了两个需求：1) 规划从上海到北京的出差行程 2) 表达了对汉庭酒店的偏好",
        "intents": [
            {
                "type": "preference",
                "confidence": 0.95,
                "description": "用户表达了对汉庭酒店的偏好",
                "reason": "用户说'喜欢住汉庭酒店'"
            },
            {
                "type": "itinerary_planning",
                "confidence": 0.98,
                "description": "用户要规划从上海到北京的出差行程",
                "reason": "用户明确说'我要从上海去北京出差'"
            }
        ],
        "key_entities": {
            "origin": "上海",
            "destination": "北京",
            "date": "未指定",
            "duration": "未指定",
            "other": "出差"
        },
        "rewritten_query": "规划一次从上海到北京的出差行程，用户偏好住汉庭酒店",
        "agent_schedule": [
            {
                "agent_name": "event_collection",
                "priority": 1,
                "reason": "收集行程基本信息（出发地、目的地、时间等）",
                "expected_output": "完整的行程信息"
            },
            {
                "agent_name": "preference",
                "priority": 1,
                "reason": "提取并保存用户的酒店偏好",
                "expected_output": "偏好保存结果"
            },
            {
                "agent_name": "memory_query",
                "priority": 1,
                "reason": "查询用户是否有过类似行程的历史",
                "expected_output": "历史行程信息"
            },
            {
                "agent_name": "itinerary_planning",
                "priority": 2,
                "reason": "基于收集的信息生成完整行程计划",
                "expected_output": "详细的行程规划"
            }
        ]
    }, ensure_ascii=False),
    role="assistant"
)
```

**关键点**：
- 消息包含完整的推理过程、意图识别结果、实体提取、调度计划
- 调度计划明确指定了优先级和执行顺序
- 每个Agent的期望输出都很清晰

---

#### 第2步：插件化架构 - LazyAgentRegistry 动态加载

**协调器接收意图识别结果**：
```python
orchestrator.reply(intention_result)
```

**LazyAgentRegistry 动态加载过程**：

```
需要加载的Agent: ["event_collection", "preference", "memory_query", "itinerary_planning"]

对于 "event_collection":
  1. 检查缓存 → 未命中
  2. 解析名称 → "event-collection" (Skill目录名)
  3. 查找脚本 → ".claude/skills/event-collection/script/agent.py"
  4. 动态导入模块 → importlib.util.spec_from_file_location()
  5. 反射查找类 → EventCollectionAgent (继承 AgentBase)
  6. 检查 __init__ 签名 → 需要 memory_manager
  7. 实例化 → EventCollectionAgent(name="event_collection", model=model, memory_manager=memory_manager)
  8. 缓存 → self.cache["event_collection"] = agent_instance
  9. 返回实例

对于 "preference":
  1. 检查缓存 → 未命中
  2. 解析名称 → "preference" (Skill目录名)
  3. 查找脚本 → ".claude/skills/preference/script/agent.py"
  4. 动态导入模块 → importlib.util.spec_from_file_location()
  5. 反射查找类 → PreferenceAgent (继承 AgentBase)
  6. 检查 __init__ 签名 → 需要 memory_manager
  7. 实例化 → PreferenceAgent(name="preference", model=model, memory_manager=memory_manager)
  8. 缓存 → self.cache["preference"] = agent_instance
  9. 返回实例

... (同样过程加载 memory_query, itinerary_planning)
```

**关键点**：
- 无需手动注册，完全自动化
- 首次加载时动态导入，后续从缓存获取
- 自动注入 `memory_manager` 等依赖

---

#### 第3步：异步并行调度 - Priority 1 并行执行

**协调器分组任务**：
```python
# Priority 1 任务（并行执行）
priority_1_tasks = [
    {"agent_name": "event_collection", "priority": 1, ...},
    {"agent_name": "preference", "priority": 1, ...},
    {"agent_name": "memory_query", "priority": 1, ...}
]

# Priority 2 任务（依赖Priority 1）
priority_2_tasks = [
    {"agent_name": "itinerary_planning", "priority": 2, ...}
]
```

**Priority 1 并行执行（asyncio.gather）**：
```python
# 创建3个并行协程
coroutines = [
    event_collection_agent.reply(msg),      # 5秒
    preference_agent.reply(msg),             # 3秒
    memory_query_agent.reply(msg)            # 4秒
]

# 并行执行
results = await asyncio.gather(*coroutines, return_exceptions=True)

# 等待时间 = max(5, 3, 4) = 5秒（而不是 5+3+4=12秒）
```

**执行时间线**：
```
时间轴：
0秒    ├─ EventCollectionAgent ─────────────────────┤ 5秒
       ├─ PreferenceAgent ──────────┤ 3秒
       ├─ MemoryQueryAgent ────────────────┤ 4秒
5秒    └─ ItineraryPlanningAgent ──────────────────┤ 5秒
10秒   完成

总耗时：10秒（而不是串行的 5+3+4+5=17秒）
```

**各Agent的输出 Msg**：

```python
# EventCollectionAgent 输出
Msg(
    name="EventCollectionAgent",
    content=json.dumps({
        "origin": "上海",
        "destination": "北京",
        "start_date": "未指定",
        "end_date": "未指定",
        "trip_purpose": "出差",
        "missing_info": ["出发日期", "返程日期"]
    }),
    role="assistant"
)

# PreferenceAgent 输出
Msg(
    name="PreferenceAgent",
    content=json.dumps({
        "preferences": [
            {
                "type": "hotel_brands",
                "value": "汉庭",
                "action": "append"
            }
        ]
    }),
    role="assistant"
)

# MemoryQueryAgent 输出
Msg(
    name="MemoryQueryAgent",
    content=json.dumps({
        "answer": "您之前在2024年2月去过北京出差，住在汉庭酒店，很满意。"
    }),
    role="assistant"
)
```

**关键点**：
- 3个Agent并行执行，总耗时 5秒
- 每个Agent返回独立的 Msg
- 协调器收集所有结果，作为Priority 2的输入

---

#### 第4步：异步并行调度 - Priority 2 依赖执行

**ItineraryPlanningAgent 接收Priority 1的结果**：
```python
input_msg = Msg(
    name="Orchestrator",
    content=json.dumps({
        "context": {
            "reasoning": "...",
            "intents": [...],
            "key_entities": {...},
            "rewritten_query": "..."
        },
        "reason": "基于收集的信息生成完整行程计划",
        "expected_output": "详细的行程规划",
        "previous_results": [
            {
                "agent_name": "event_collection",
                "priority": 1,
                "result": {
                    "origin": "上海",
                    "destination": "北京",
                    ...
                }
            },
            {
                "agent_name": "preference",
                "priority": 1,
                "result": {
                    "preferences": [{"type": "hotel_brands", "value": "汉庭"}]
                }
            },
            {
                "agent_name": "memory_query",
                "priority": 1,
                "result": {
                    "answer": "您之前在2024年2月去过北京出差..."
                }
            }
        ]
    }),
    role="user"
)
```

**ItineraryPlanningAgent 处理**：
- 接收Priority 1的所有结果
- 整合信息生成行程计划
- 考虑用户偏好（汉庭酒店）
- 参考历史经验

**输出 Msg**：
```python
Msg(
    name="ItineraryPlanningAgent",
    content=json.dumps({
        "itinerary": {
            "title": "北京出差行程规划",
            "duration": "3天2晚",
            "daily_plans": [
                {
                    "day": 1,
                    "activities": [
                        {
                            "time": "09:00",
                            "activity": "上海虹桥站出发",
                            "transport": "高铁G101"
                        },
                        {
                            "time": "12:30",
                            "activity": "北京南站到达",
                            "transport": "地铁2号线"
                        },
                        {
                            "time": "14:00",
                            "activity": "入住汉庭酒店（朝阳区）",
                            "description": "根据您的偏好选择"
                        }
                    ],
                    "meals": {
                        "lunch": "北京南站附近",
                        "dinner": "酒店附近"
                    }
                },
                {
                    "day": 2,
                    "activities": [
                        {
                            "time": "09:00",
                            "activity": "会议",
                            "description": "中关村科技园"
                        },
                        {
                            "time": "12:00",
                            "activity": "午餐",
                            "description": "会议地点附近"
                        },
                        {
                            "time": "14:00",
                            "activity": "继续会议"
                        },
                        {
                            "time": "18:00",
                            "activity": "晚餐",
                            "description": "簋街美食街"
                        }
                    ]
                },
                {
                    "day": 3,
                    "activities": [
                        {
                            "time": "10:00",
                            "activity": "退房",
                            "description": "汉庭酒店"
                        },
                        {
                            "time": "11:00",
                            "activity": "北京南站出发",
                            "transport": "高铁G102"
                        },
                        {
                            "time": "14:30",
                            "activity": "上海虹桥站到达"
                        }
                    ]
                }
            ],
            "notes": [
                "汉庭酒店已根据您的偏好预订",
                "高铁票建议提前预订",
                "北京天气预报：晴，温度15-25°C"
            ]
        }
    }),
    role="assistant"
)
```

**关键点**：
- 依赖Priority 1的所有结果
- 整合多个Agent的输出
- 生成完整的行程计划

---

#### 第5步：结果聚合与记忆更新

**协调器聚合所有结果**：
```python
final_result = {
    "status": "completed",
    "intention": {
        "intents": [
            {"type": "preference", "confidence": 0.95},
            {"type": "itinerary_planning", "confidence": 0.98}
        ],
        "key_entities": {
            "origin": "上海",
            "destination": "北京"
        }
    },
    "agents_executed": 4,
    "results": [
        {
            "agent_name": "event_collection",
            "priority": 1,
            "status": "success",
            "data": {...}
        },
        {
            "agent_name": "preference",
            "priority": 1,
            "status": "success",
            "data": {...}
        },
        {
            "agent_name": "memory_query",
            "priority": 1,
            "status": "success",
            "data": {...}
        },
        {
            "agent_name": "itinerary_planning",
            "priority": 2,
            "status": "success",
            "data": {...}
        }
    ]
}
```

**更新长期记忆**：
```python
# 保存用户偏好
memory_manager.long_term.save_preference("hotel_brands", ["汉庭"])

# 保存行程历史
memory_manager.long_term.save_trip_history({
    "origin": "上海",
    "destination": "北京",
    "start_date": "待确认",
    "end_date": "待确认",
    "purpose": "出差"
})
```

**CLI 显示结果**：
```
🤖 调用智能体: 事项收集 ✓, 偏好管理 ✓, 记忆查询 ✓, 行程规划 ✓

✈️  北京出差行程规划
时长: 3天2晚

第 1 天
  09:00 - 上海虹桥站出发
    🚇 高铁G101
  12:30 - 北京南站到达
    🚇 地铁2号线
  14:00 - 入住汉庭酒店（朝阳区）
    根据您的偏好选择
  🍜 北京南站附近
  🍽️  酒店附近

第 2 天
  09:00 - 会议
    中关村科技园
  12:00 - 午餐
    会议地点附近
  14:00 - 继续会议
  18:00 - 晚餐
    簋街美食街

第 3 天
  10:00 - 退房
    汉庭酒店
  11:00 - 北京南站出发
    🚇 高铁G102
  14:30 - 上海虹桥站到达

📌 注意事项
  • 汉庭酒店已根据您的偏好预订
  • 高铁票建议提前预订
  • 北京天气预报：晴，温度15-25°C

✓ 已更新您的偏好设置
  • 酒店偏好 设置为 汉庭
```

---

## 场景二：信息查询请求

### 用户输入
```
"北京明天天气怎么样？"
```

### 执行流程（简化版）

#### 意图识别
```
识别意图：information_query
调度计划：
  - Priority 1: information_query (联网搜索)
```

#### 并行执行
```
Priority 1 (1个Agent):
  └─ InformationQueryAgent (DuckDuckGo搜索 + LLM摘要) → 3秒
```

#### 结果
```
北京明天天气预报：
- 温度：18-28°C
- 天气：晴转多云
- 风力：3-4级
- 紫外线指数：中等

参考来源：
1. weather.com
2. 中国气象局
3. 天气预报网
```

---

## 场景三：记忆查询请求

### 用户输入
```
"我去过哪些地方？"
```

### 执行流程

#### 意图识别
```
识别意图：memory_query
调度计划：
  - Priority 1: memory_query (查询历史记忆)
```

#### 并行执行
```
Priority 1 (1个Agent):
  └─ MemoryQueryAgent (查询trip_history) → 1秒
```

#### 结果
```
根据您的历史记录，您去过以下地方：

1. 北京 (2024年2月) - 出差
   - 住宿：汉庭酒店
   - 时长：3天

2. 杭州 (2024年1月) - 旅游
   - 住宿：西湖边民宿
   - 时长：5天

3. 深圳 (2023年12月) - 出差
   - 住宿：希尔顿酒店
   - 时长：2天

最常去的目的地：北京 (3次)
```

---

## 性能对比总结

### 场景一：行程规划（4个Agent）

**串行执行（LangGraph风格）**：
```
EventCollectionAgent:     5秒
PreferenceAgent:          3秒
MemoryQueryAgent:         4秒
ItineraryPlanningAgent:   5秒
─────────────────────────────
总计：                   17秒
```

**并行执行（AgentScope风格）**：
```
Priority 1 (并行):
  EventCollectionAgent:     5秒 ┐
  PreferenceAgent:          3秒 ├─ max = 5秒
  MemoryQueryAgent:         4秒 ┘

Priority 2 (依赖):
  ItineraryPlanningAgent:   5秒

─────────────────────────────
总计：                   10秒

性能提升：-41%（从17秒到10秒）
```

**实际测试结果**：15秒（包括网络延迟、LLM推理等）

---

## 三大设计思想的体现

### 1. 消息传递架构
- ✅ 每个Agent通过 `Msg` 接收输入、返回输出
- ✅ 完整的推理过程、意图识别结果、实体提取都在 `Msg` 中
- ✅ 调试时可以直观看到每条消息的内容和流向

### 2. 插件化架构
- ✅ 4个Agent自动加载（无需手动注册）
- ✅ 每个Agent都是独立的插件（`.claude/skills/*/script/agent.py`）
- ✅ 新增Agent只需复制目录结构，无需修改核心代码

### 3. 异步并行调度
- ✅ Priority 1的3个Agent并行执行（5秒）
- ✅ Priority 2依赖Priority 1的结果（5秒）
- ✅ 总耗时10秒（而不是串行的17秒）

---

## 关键代码位置

| 步骤 | 文件 | 函数 | 说明 |
|------|------|------|------|
| 1. 意图识别 | `agents/intention_agent.py` | `reply()` | 识别意图、提取实体、生成调度计划 |
| 2. 插件加载 | `agents/lazy_agent_registry.py` | `__getitem__()` | 动态加载Agent插件 |
| 3. 优先级分组 | `agents/orchestration_agent.py` | `reply()` | 按优先级分组任务 |
| 4. 并行执行 | `agents/orchestration_agent.py` | `_execute_parallel_agents()` | asyncio.gather并行执行 |
| 5. 结果聚合 | `agents/orchestration_agent.py` | `_aggregate_results()` | 聚合所有Agent结果 |
| 6. 记忆更新 | `agents/orchestration_agent.py` | `_update_memory()` | 更新长期记忆 |
| 7. 结果显示 | `cli.py` | `_generate_human_response()` | 生成人性化回复 |


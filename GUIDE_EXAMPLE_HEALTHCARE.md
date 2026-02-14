# Healthcare AI Agent 代码分析指南

## 概述

本示例展示了一个基于 **Parlant 框架** 构建的医疗保健领域 AI 助手，用于处理患者预约和查询实验室结果等常见场景。

---

## 核心架构

```
┌─────────────────────────────────────────────────────────┐
│                    Healthcare Agent                      │
├─────────────────────────────────────────────────────────┤
│  Domain Glossary (领域术语表)                            │
│  ├── Office Phone Number                                │
│  ├── Office Hours                                       │
│  └── Charles Xavier (医生信息)                          │
├─────────────────────────────────────────────────────────┤
│  Journeys (对话流程)                                    │
│  ├── Schedule an Appointment (预约就诊)                 │
│  └── Lab Results (实验室结果查询)                       │
├─────────────────────────────────────────────────────────┤
│  Guidelines (行为准则)                                   │
│  ├── 保险咨询处理                                        │
│  ├── 转人工客服                                          │
│  └── 超出范围的问题                                      │
└─────────────────────────────────────────────────────────┘
```

---

## 1. 工具函数 (Tools)

### 1.1 预约相关工具

| 函数名 | 功能 | 返回值 |
|--------|------|--------|
| `get_insurance_providers` | 获取支持的保险公司列表 | ["Mega Insurance", "Acme Insurance"] |
| `get_upcoming_slots` | 获取近期可预约时段 | "Monday 10 AM", "Tuesday 2 PM" 等 |
| `get_later_slots` | 获取更晚的可预约时段 | 11月等较晚时段 |
| `schedule_appointment` | 执行预约操作 | 确认预约成功信息 |

**业务意义**：这些工具模拟了与医院预约系统或 API 的交互，实际生产环境会替换为真实的后端服务调用。

### 1.2 检验结果工具

```python
async def get_lab_results(context: p.ToolContext) -> p.ToolResult:
    lab_results = {
        "report": "All tests are within the valid range",
        "prognosis": "Patient is healthy as a horse!",
    }
    return p.ToolResult(data={...})
```

**业务意义**：
- 从患者上下文中获取 `customer_id`
- 查询检验数据库获取检验报告
- 根据结果提供预后建议

---

## 2. 领域术语表 (Domain Glossary)

术语表用于确保 AI 助手对专业术语理解一致：

| 术语 | 描述 | 用途 |
|------|------|------|
| **Office Phone Number** | +1-234-567-8900 | 提供联系方式 |
| **Office Hours** | 周一至周五 9:00-17:00 | 说明营业时间 |
| **Charles Xavier** | 神经科医生，周一周二可用 | 特定医生排班查询 |

**业务意义**：在医疗场景中，统一的术语定义可以避免信息传递歧义，确保回复准确性。

---

## 3. 预约流程 (Scheduling Journey)

### 流程图

```
┌──────────────────────────────────────────────────────────────┐
│                    确定就诊原因                               │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│              获取近期可预约时段 (get_upcoming_slots)           │
└─────────────────────────┬────────────────────────────────────┘
                          │
                          ▼
┌──────────────────────────────────────────────────────────────┐
│              展示时段并询问患者选择                            │
└──────────┬─────────────────┬─────────────────────────────────┘
           │                 │
           │  患者选择时间    │  时间不合适
           ▼                 ▼
┌──────────────────┐   ┌──────────────────────────────────┐
│ 确认预约信息      │   │       获取更晚时段                │
└────────┬─────────┘   │      (get_later_slots)           │
         │             └──────────────┬───────────────────┘
         │                            │
         │  患者确认                   ▼
         │             ┌──────────────────────────────────┐
         ▼             │    展示更晚时段并询问              │
┌──────────────────┐   └──────────────┬───────────────────┘
│  执行预约         │                  │
│ (schedule_       │                  │
│  appointment)    │                  │
└────────┬─────────┘                  │
         │                            │
         ▼                            │
┌──────────────────┐    ┌─────────────┴─────────────┐
│ 预约成功确认      │    │                           │
└────────┬─────────┘    │  患者仍不满意 ←────────────┘
         │              │         │
         ▼              ▼         │
┌──────────────────┐   ┌───────────────────┐
│    结束流程       │   │  请患者电话预约    │
└──────────────────┘   │  (END_JOURNEY)    │
                       └───────────────────┘
```

### 条件分支逻辑

| 条件 | 后续操作 |
|------|----------|
| 患者选择时间 | 进入确认 → 预约 → 完成 |
| 近期时间都不合适 | 获取更晚时段 |
| 更晚时段仍不合适 | 引导电话预约 |
| 患者表示紧急 | 触发准则：立即电话联系 |

### 紧急情况处理准则

```python
await journey.create_guideline(
    condition="The patient says their visit is urgent",
    action="Tell them to call the office immediately",
)
```

**业务意义**：医疗场景中，紧急情况必须立即转人工处理，不能让 AI 拖延。

---

## 4. 实验室结果流程 (Lab Results Journey)

### 流程状态

```
┌─────────────────────────────────────────┐
│          获取检验结果 (Tool)              │
└──────────────────┬──────────────────────┘
                   │
         ┌─────────┼─────────┐
         ▼         ▼         ▼
    结果未找到   结果正常   结果异常
         │         │         │
         ▼         ▼         ▼
    稍后重试    解释正常    建议电话
    END       结果正常     咨询医生
                         END
```

### 关键准则

```python
await agent.create_guideline(
    condition="The patient presses you for more conclusions about the lab results",
    action="Assertively tell them that you cannot help and they should call the office",
)
```

**业务意义**：
- AI 绝对不能代替医生做出诊断结论
- 异常结果必须引导患者联系专业医护人员
- 这是法律和伦理上的必要保护措施

---

## 5. 全局行为准则 (Guidelines)

### 5.1 保险咨询

```python
await agent.create_guideline(
    condition="The patient asks about insurance",
    action="List the insurance providers we accept, and tell them to call the office for more details",
    tools=[get_insurance_providers],
)
```

### 5.2 转人工客服

```python
await agent.create_guideline(
    condition="The patient asks to talk to a human agent",
    action="Ask them to call the office, providing the phone number",
)
```

### 5.3 超出范围的问题

```python
await agent.create_guideline(
    condition="The patient inquires about something that has nothing to do with our healthcare",
    action="Kindly tell them you cannot assist with off-topic inquiries",
)
```

---

## 6. 意图消歧 (Disambiguation)

```python
status_inquiry = await agent.create_observation(
    "The patient asks to follow up on their visit, but it's not clear in which way",
)

# 在两个 Journeys 之间进行消歧
await status_inquiry.disambiguate([scheduling_journey, lab_results_journey])
```

**业务场景**：
- 患者说 "我想跟进我的就诊" 时
- AI 需要判断是 **想预约复诊** 还是 **想查看检验结果**
- 系统根据后续对话内容自动路由到正确的流程

---

## 7. 设计模式总结

### 7.1 Journey 模式
用于管理有明确步骤的多轮对话流程，每个状态转换都有明确的条件判断。

### 7.2 Guideline 模式
用于定义 AI 的边界行为，确保在各种边缘情况下都能给出合适的响应。

### 7.3 Tool 模式
封装后端服务调用，使 AI 能够动态获取和操作数据。

### 7.4 Glossary 模式
确保领域特定概念的语义一致性。

---

## 8. 医疗 AI 的关键设计原则

| 原则 | 实现方式 |
|------|----------|
| **安全第一** | 紧急情况立即转人工 |
| **不替代医生** | 检验结果不给出诊断建议 |
| **边界清晰** | 明确哪些问题能回答，哪些不能 |
| **同理心** | Agent 描述为 "empathetic and calming" |
| **可追溯** | 每个交互都有明确的状态记录 |

---

## 9. 扩展建议

1. **接入真实系统**：将工具函数替换为真正的医院信息系统 (HIS) API
2. **增加身份验证**：添加患者身份验证流程，确保数据安全
3. **多语言支持**：添加翻译功能服务非英语患者
4. **预约提醒**：集成邮件/短信提醒服务
5. **满意度调查**：流程结束后收集患者反馈

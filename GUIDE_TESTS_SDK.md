# Parlant SDK 核心特性测试指南

## 文档简介

### 是什么

本文档是 Parlant SDK 的**测试指南**，提供了 Parlant 核心特性的介绍与测试方法，包括测试目标、测试用例和操作步骤。

### 有什么用

- **学习 SDK**：通过具体测试用例理解每个功能的使用方式
- **验证功能**：运行测试验证 SDK 是否正确安装和运行
- **开发参考**：编写新测试时参考现有的测试模式

### 模块介绍

| 名称 | 实质 | 简介 |
|------|--------|------------|
| **Agents（智能体）** | AI 对话机器人 | 与用户交流的主体，可定义角色、行为规则和消息处理逻辑 |
| **Guidelines（指导规则）** | 对话策略规则 | 定义在什么情况下说什么话、做什么事，解决"什么时候做什么"的问题 |
| **Journeys（业务流程）** | 对话流程 | 像流程图一样设计对话步骤，引导用户完成特定任务（如订餐、挂号） |
| **Tools（工具）** | 外部功能扩展 | Agent 可调用的自定义函数，如查天气、查订单、调用 API |
| **Glossary（术语）** | 专业术语库 | 定义 Agent 需要理解的行业术语及其同义词，确保理解用户意图一致 |
| **Canned Responses（预设回复）** | 回复模板 | 预定义的回复内容，支持变量填充，如"您的订单 {order_id} 已发货" |
| **Context Variables（上下文变量）** | 会话状态存储 | 在对话中存储和传递信息，如用户选择的套餐、会员等级、偏好设置 |
| **Sessions（会话管理）** | 对话历史管理 | 管理用户对话记录、识别老客户、维护客户信息 |

### 适用人群

- SDK 开发者：理解 SDK 工作原理
- 测试工程师：运行和编写测试用例
- 集成开发者：将 SDK 集成到项目中

---

## 目录

- [环境准备](#环境准备)
- [Agents（智能体）](#agents智能体)
- [Guidelines（指导规则）](#guidelines指导规则)
- [Journeys（业务流程）](#journeys业务流程)
- [Tools（工具）](#tools工具)
- [Glossary（术语）](#glossary术语)
- [Canned Responses（预设回复）](#canned-responses预设回复)
- [Context Variables（上下文变量）](#context-variables上下文变量)
- [Sessions（会话管理）](#sessions会话管理)
- [常见问题](#常见问题)
- [测试最佳实践](#测试最佳实践)
- [总结](#总结)

---

## 1. 环境准备

### 1.1 激活虚拟环境

```bash
cd /home/project/parlant
python3 -m venv parlant-env
source parlant-env/bin/activate
```

### 1.2 确保依赖已安装

```bash
pip install -e .
```

### 1.3 配置 LLM API Key

```bash
export EMCIE_API_KEY="sk-mc-1Y9EBuxynrhfePRtA-XXX"
# 或使用其他 LLM 提供商
# export OPENAI_API_KEY="sk-..."
```

### 1.4 测试文件位置

所有测试文件位于 `tests/sdk/` 目录：

```
tests/sdk/
├── test_agents.py           # Agent 测试
├── test_guidelines.py       # Guideline 测试
├── test_journeys.py         # Journey 测试
├── test_tools.py            # Tool 测试
├── test_glossary.py         # Glossary 测试
├── test_canned_responses.py # Canned Response 测试
├── test_variables.py        # Context Variable 测试
└── test_customers.py       # Customer/Session 测试
```

### 1.5 运行测试

#### 1.5.1 运行 SDK 所有测试

```bash
cd /home/project/parlant
pytest tests/sdk/ -v
```

#### 1.5.2 运行特定模块测试

```bash
# Agent 测试
pytest tests/sdk/test_agents.py -v

# Guideline 测试
pytest tests/sdk/test_guidelines.py -v

# Journey 测试
pytest tests/sdk/test_journeys.py -v
```

#### 1.5.3 运行特定用例测试

```bash
# 运行指定测试类
pytest tests/sdk/test_agents.py::Test_that_an_agent_can_be_created -v

# 运行多个指定测试类
pytest tests/sdk/test_agents.py::Test_that_an_agent_can_be_created tests/sdk/test_agents.py::Test_that_an_agent_can_be_read_by_id -v

# 使用通配符匹配测试类名
pytest tests/sdk/test_agents.py -k "agent_can_be" -v
```

---

## 2. Agents（智能体）

### 2.1 测试目标

验证 Agent 的创建、读取、列表查询、优先级策略等核心功能。

### 2.2 运行测试

```bash
pytest tests/sdk/test_agents.py -v
```

### 2.3 核心测试用例

| 测试类 | 功能描述 | 关键代码 |
|--------|----------|----------|
| `Test_that_an_agent_can_be_created` | 创建 Agent | `server.create_agent(name="Test Agent", description="...")` |
| `Test_that_an_agent_can_be_read_by_id` | 通过 ID 读取 Agent | `ctx.client.agents.retrieve(agent_id)` |
| `Test_that_agents_can_be_listed` | 列出所有 Agent | `ctx.server.list_agents()` |
| `Test_that_an_agent_can_be_found_by_id` | 通过 ID 查找 Agent | `ctx.server.find_agent(id=...)` |
| `Test_that_an_agent_can_be_found_using_tool_context` | 在 Tool 中访问 Agent | `p.ToolContextAccessor(context).server.find_agent()` |

### 2.4 详细示例

以 `Test_that_an_agent_can_be_found_using_tool_context` 为例，测试 **Agent 和 Tool 的结合使用**：

```python
class Test_that_an_agent_can_be_found_using_tool_context(SDKTest):
    async def setup(self, server: p.Server) -> None:
        # 创建一个名为 "Tool Context Agent" 的 Agent
        self.agent = await server.create_agent(
            name="Tool Context Agent",
            description="Agent for tool context test",
        )

        @p.tool  # 装饰器：标识这是一个工具
        async def check_what_is_spatio(context: ToolContext) -> ToolResult:
            # 关键代码：在 context 中找到当前的 Agent
            agent = await p.ToolContextAccessor(context).server.find_agent(
                id=context.agent_id
            )

            if agent is None:
                return ToolResult("A spatio is a special type of spaghetti spoon.")
            else:
                return ToolResult("Spatio is the name of a famous fictional mouse.")

        # 将 Tool 附加到 Agent，条件是"用户问 spatio"
        await self.agent.attach_tool(check_what_is_spatio, condition="the user asks about spatio")

    async def run(self, ctx: Context) -> None:
        # 发送消息给 Agent，触发 Tool
        answer = await ctx.send_and_receive_message(
            customer_message="What is spatio?",
            recipient=self.agent,
        )

        # 验证 Tool 正确返回了 Agent 的信息
        assert await nlp_test(answer, "It says that spatio is the name of a mouse.")
```

**关键点：**
- `context.agent_id`：Tool 执行时所属的 Agent ID
- `ToolContextAccessor(context).server.find_agent(id=context.agent_id)`：通过 ToolContext 找到当前 Agent

**说明：** 若 Tool 正确返回答案，但测试结果为 FAILED，是因为 `nlp_test()` 硬编码使用 `GPT_4O`（OpenAI 模型），必须设置 `OPENAI_API_KEY`。

**测试结论：**
- **功能价值**：验证 **Tool 可以访问它所属的 Agent 信息**
- **业务价值**：同一个 **Tool 可以根据不同 Agent 返回不同结果**，减少重复开发。例如：
  - 多个门店共用"获取营业时间"工具，根据门店返回不同时间
  - 多个角色共用"产品查询"工具，面向不同角色返回不同答复

### 2.5 高级特性测试

```bash
# 测试优先级策略
pytest tests/sdk/test_agents.py::Test_that_the_output_of_an_agent_can_be_intercepted -v

# 测试流式输出
pytest tests/sdk/test_agents.py::Test_that_an_agent_can_be_created_with_streaming_output_mode -v

# 测试自定义 ID
pytest tests/sdk/test_agents.py::Test_that_an_agent_can_be_created_with_custom_id -v
```

### 2.6 钩子拦截示例

以 `Test_that_the_output_of_an_agent_can_be_intercepted` 为例：

```python
class Test_that_the_output_of_an_agent_can_be_intercepted(SDKTest):
    async def configure_hooks(self, hooks: p.EngineHooks) -> p.EngineHooks:
        async def intercept_message(
            ctx: p.EngineContext, payload: Any, exc: Exception | None
        ) -> p.EngineHookResult:
            # payload 是 Agent 原本要发送的消息
            await ctx.session_event_emitter.emit_message_event(
                trace_id=ctx.tracer.trace_id,
                data="Bananas! More bananas!",
            )
            # 拒绝原始消息，替换成自定义内容
            return p.EngineHookResult.BAIL

        # 把拦截器挂载到 "消息生成" 事件上
        hooks.on_message_generated.append(intercept_message)
        return hooks

    async def run(self, ctx: Context) -> None:
        answer = await ctx.send_and_receive_message(customer_message="Hello", recipient=self.agent)
        assert answer == "Bananas! More bananas!"
```

**测试过程：**

| 步骤 | 操作 | 结果 |
|------|------|------|
| 1 | 用户发送 "Hello" | ✓ |
| 2 | Agent 本想回复正常内容 | ✓ |
| 3 | 钩子拦截并替换消息 | ✓ |
| 4 | 最终返回 "Bananas! More bananas!" | ✓ |

**测试结论：**
- **功能价值**：验证 **"最后一公里"控制力** —— 在消息送到用户之前，有机会修改、验证或阻止它
- **业务价值**：拦截消息（敏感词过滤、内容审核）、替换内容（个性化输出）、阻止发送（高风险场景强制介入）

**实际案例：**
```python
# 敏感内容审核
async def content_moderation_hook(ctx, payload, exc):
    if contains_sensitive_words(payload):
        return p.EngineHookResult.BAIL  # 阻止发送
    return p.EngineHookResult.CONTINUE
```

---

## 3. Guidelines（指导规则）

### 3.1 测试目标

验证 Guideline 的创建、优先级、关系（依赖、蕴含、消歧）、匹配器等功能。

### 3.2 运行测试

```bash
pytest tests/sdk/test_guidelines.py -v
```

### 3.3 核心测试用例

| 测试类 | 功能描述 | 关键代码 |
|--------|----------|----------|
| `Test_that_guideline_can_take_priority_over_another_guideline` | Guideline 优先级 | `guideline.prioritize_over(other_guideline)` |
| `Test_that_guideline_entailment_relationship_can_be_created` | 蕴含关系 | `g1.entail(g2)` |
| `Test_that_guideline_dependency_relationship_can_be_created` | 依赖关系 | `g2.depend_on(g1)` |
| `Test_that_guideline_disambiguation_creates_relationships` | 消歧关系 | `g1.disambiguate([g2, g3])` |
| `Test_that_guideline_can_use_custom_matcher` | 自定义匹配器 | `matcher=p.Guideline.MATCH_ALWAYS` |

### 3.4 优先级测试详解

```python
async def test_guideline_priority():
    async with p.Server() as server:
        agent = await server.create_agent(name="PriorityAgent", description="")

        high_priority = await agent.create_guideline(
            condition="Customer asks about drinks",
            action="Recommend Pepsi",
        )
        low_priority = await agent.create_guideline(
            condition="Customer asks about drinks",
            action="Recommend Coca-Cola",
        )

        await high_priority.prioritize_over(low_priority)
        # 测试时，Pepsi 会被优先选择
```

### 3.5 关系测试详解

```python
# 蕴含关系
g1 = await agent.create_guideline(condition="Customer is upset", action="Transfer to manager")  # 当顾客不高兴时，转接经理
g2 = await agent.create_guideline(condition="New customer arrives", action="Offer help")  # 当新顾客到达时，提供帮助
await g1.entail(g2)
# 解读：g1 蕴含 g2，如果 g1 被触发（顾客不高兴），那么 g2 也应该被触发（新顾客到达），不高兴的顾客可能是新顾客，需要同时应用两条规则

# 依赖关系
g1 = await agent.create_guideline(condition="Customer asks about price", action="Give price")  # 顾客询问价格时，告诉价格
g2 = await agent.create_guideline(condition="Customer is frustrated", action="Apologize")  # 顾客沮丧时，道歉
await g2.depend_on(g1)
# 解读：g2 依赖于 g1，如果 g2 要被触发，必须先确保 g1 也被考虑，先获取价格信息，再处理沮丧情绪，沮丧可能源于价格

# 消歧关系
g1 = await agent.create_guideline(condition="Customer is thirsty")
g2 = await agent.create_guideline(condition="Customer wants water")
g3 = await agent.create_guideline(condition="Customer asks about nearby coffee shops")
await g1.disambiguate([g2, g3])
# 解读：当 g1 匹配但含义模糊时，让用户在 g2 和 g3 之间选择，根据用户澄清，走不同的处理流程
```

### 3.6 运行特定测试

```bash
# 测试优先级
pytest tests/sdk/test_guidelines.py::Test_that_guideline_can_take_priority_over_another_guideline -v

# 测试关系
pytest tests/sdk/test_guidelines.py::Test_that_guideline_entailment_relationship_can_be_created -v

# 测试自定义匹配器
pytest tests/sdk/test_guidelines.py::Test_that_guideline_can_use_custom_matcher -v
```

### 3.7 优先级测试示例

以 `Test_that_guideline_can_take_priority_over_another_guideline` 为例，测试 **Guideline 优先级**：

```python
class Test_that_guideline_can_take_priority_over_another_guideline(SDKTest):
    async def setup(self, server: p.Server) -> None:
        self.agent = await server.create_agent(
            name="Priority Agent",
            description="Agent for testing guideline priority",
        )

        self.high_priority = await self.agent.create_guideline(
            condition="Customer asks about drinks",
            action="Recommend Pepsi",
        )

        self.low_priority = await self.agent.create_guideline(
            condition="Customer asks about drinks",
            action="Recommend Coca-Cola",
        )

        await self.high_priority.prioritize_over(self.low_priority)

    async def run(self, ctx: Context) -> None:
        response = await ctx.send_and_receive_message(
            customer_message="What drinks do you have?",
            recipient=self.agent,
        )

        assert "pepsi" in response.lower(), f"Expected Pepsi in response: {response}"
        assert "cola" not in response.lower() and "coke" not in response.lower(), (
            f"Did not expect Coca-Cola in response: {response}"
        )
```

**关键点：**
- `prioritize_over()`：设置 Guideline 之间的优先级关系
- 当多个 Guideline 匹配同一个条件时，高优先级的 Guideline 会被优先选择

**测试结论：**

| 步骤 | 操作 | 结果 |
|------|------|------|
| 1 | 创建两条都匹配"顾客询问饮料"的 Guideline | ✓ |
| 2 | 设置 Pepsi 的优先级高于 Coca-Cola | ✓ |
| 3 | 用户询问饮料时，优先返回 Pepsi 的推荐 | ✓ |
| 4 | 不包含 Coca-Cola 的推荐 | ✓ |

**功能价值**：验证 **Guideline 优先级机制** —— 当多个 Guideline 条件都匹配时，系统能够根据优先级选择执行哪一个。

**业务价值：**

| 场景 | 说明 |
|------|------|
| 冲突解决 | 多个 Guideline 都匹配时，优先级高的生效 |
| 业务规则排序 | 确保重要的业务规则优先执行 |
| 异常处理 | 特定的异常处理 Guideline 可以覆盖通用的 |

**实际案例：**
```python
# 客服场景：紧急投诉优先于普通问候
high_priority = await agent.create_guideline(
    condition="Customer is upset or complaining",
    action="Apologize and transfer to manager",
)
low_priority = await agent.create_guideline(
    condition="Customer says hello",
    action="Greet warmly",
)
await high_priority.prioritize_over(low_priority)
```

---

## 4. Journeys（业务流程）

### 4.1 测试目标

验证 Journey 的创建、状态转换、条件分支、旅程链接等功能。

### 4.2 运行测试

```bash
pytest tests/sdk/test_journeys.py -v
```

### 4.3 核心测试用例

| 测试类 | 功能描述 | 关键代码 |
|--------|----------|----------|
| `Test_that_a_created_journey_is_followed` | 旅程被正确执行 | `journey.create_guideline(...)` |
| `Test_that_journey_transition_and_state_can_be_created_with_transition` | 创建状态转换 | `journey.initial_state.transition_to(chat_state="...")` |
| `Test_that_journey_state_can_transition_to_a_tool` | 状态转换到工具 | `transition_to(tool_state=my_tool)` |
| `Test_that_journey_state_can_be_transitioned_with_condition` | 条件分支 | `transition_to(condition="if yes", ...)` |
| `Test_that_journey_can_take_priority_over_another_journey` | 旅程优先级 | `journey1.prioritize_over(journey2)` |

### 4.4 旅程创建示例

```python
async def test_journey_creation():
    async with p.Server() as server:
        agent = await server.create_agent(name="StoreAgent", description="You work at a store")

        journey = await agent.create_journey(
            title="Order Pizza",
            conditions=["Customer wants to order pizza"],
            description="Help customer order pizza",
        )

        t1 = await journey.initial_state.transition_to(chat_state="Ask about toppings")
        t2 = await t1.target.transition_to(
            condition="Customer confirms toppings",
            chat_state="Confirm the order",
        )

        await t2.target.transition_to(state=p.END_JOURNEY)
```

### 4.5 条件分支示例

```python
journey = await agent.create_journey(
    title="Breakfast Choice",
    conditions=["Customer wants breakfast"],
    description="Handle breakfast orders",
)

t_ask = await journey.initial_state.transition_to(chat_state="Ask if customer wants breakfast")

# 条件分支
t_yes = await t_ask.target.transition_to(
    condition="if the customer says yes",
    chat_state="Add breakfast to booking",
)

t_no = await t_ask.target.transition_to(
    condition="if the customer says no",
    chat_state="Proceed without breakfast",
)
```

### 4.6 旅程链接示例

```python
main_journey = await agent.create_journey(
    title="Main Process",
    conditions=["Customer starts process"],
    description="Main process flow",
)

validation_journey = await agent.create_journey(
    title="Validate User",
    conditions=[],
    description="Validate user identity",
)

t1 = await main_journey.initial_state.transition_to(chat_state="Ask for room type")
t2 = await t1.target.transition_to(journey=validation_journey)
```

### 4.7 运行特定测试

```bash
# 测试旅程创建和跟随
pytest tests/sdk/test_journeys.py::Test_that_a_created_journey_is_followed -v

# 测试状态转换
pytest tests/sdk/test_journeys.py::Test_that_journey_transition_and_state_can_be_created_with_transition -v

# 测试条件分支
pytest tests/sdk/test_journeys.py::Test_that_journey_state_can_be_transitioned_with_condition -v

# 测试旅程链接
pytest tests/sdk/test_journeys.py::Test_that_journey_can_link_to_another_journey_with_validation -v
```

### 4.8 旅程与 Guideline 结合示例

以 `Test_that_scoped_guideline_of_matched_journey_without_states_influence_response` 为例，测试 **旅程与 Guideline 结合**：

```python
class Test_that_scoped_guideline_of_matched_journey_without_states_influence_response(SDKTest):
    async def setup(self, server: p.Server) -> None:
        self.agent = await server.create_agent(
            name="Test Agent",
            description="Test agent for journey testing",
        )

        self.journey = await self.agent.create_journey(
            title="Test Journey",
            conditions=["Customer greets you"],
            description="Test journey",
        )

        # 在旅程中创建 Guideline，使用 MATCH_ALWAYS 匹配器
        await self.journey.create_guideline(
            matcher=p.Guideline.MATCH_ALWAYS,
            condition="The customer greets you",
            action="Immediately offer a Pepsi",
        )

    async def run(self, ctx: Context) -> None:
        response = await ctx.send_and_receive_message("Hello!", recipient=self.agent)
        assert "pepsi" in response.lower()
```

**关键点：**
- `journey.create_guideline()`：在旅程内部创建 Guideline，该 Guideline 只在该旅程激活时生效
- `matcher=p.Guideline.MATCH_ALWAYS`：使用始终匹配的匹配器，确保条件满足时立即执行
- 旅程中的 Guideline 是"作用域化"的，即只有当旅程被激活时才会被考虑

**测试结论：**

| 步骤 | 操作 | 结果 |
|------|------|------|
| 1 | 创建旅程，条件为"顾客打招呼" | ✓ |
| 2 | 在旅程中创建 Guideline，action 是"立即推荐 Pepsi" | ✓ |
| 3 | 用户发送"Hello!"，旅程被激活 | ✓ |
| 4 | Agent 返回包含"Pepsi"的响应 | ✓ |

**功能价值**：验证 **旅程与 Guideline 的结合使用** —— 旅程可以封装一系列的 Guideline，在特定条件下激活并指导对话流程。

**业务价值：**

| 场景 | 说明 |
|------|------|
| 流程引导 | 将复杂业务流程拆分为多个状态，逐步引导用户 |
| 条件激活 | 只有满足特定条件时，相关的对话流程才会被激活 |
| 上下文保持 | 旅程状态在整个对话过程中保持，帮助 Agent 理解当前处于哪个阶段 |

**实际案例：**
```python
# 电商场景：订单流程
journey = await agent.create_journey(
    title="Order Pizza",
    conditions=["Customer wants to order pizza"],
    description="Help customer order pizza",
)

await journey.initial_state.transition_to(chat_state="Ask about toppings")
await t1.target.transition_to(condition="Customer confirms toppings", chat_state="Confirm the order")
await t2.target.transition_to(state=p.END_JOURNEY)
```

---

## 5. Tools（工具）

### 5.1 测试目标

验证 Tool 的创建、附加、调用和上下文访问功能。

### 5.2 运行测试

```bash
pytest tests/sdk/test_tools.py -v
```

### 5.3 核心测试用例

| 测试类 | 功能描述 | 关键代码 |
|--------|----------|----------|
| `Test_that_a_tool_is_called_when_triggered_by_user_message` | 工具被用户消息触发 | `agent.attach_tool(tool, condition="...")` |
| `Test_that_a_tool_can_access_current_customer` | 工具访问当前客户 | `p.Customer.current.id` |

### 5.4 Tool 创建示例

```python
from parlant.core.tools import ToolContext, ToolResult
import parlant.sdk as p

# 创建 Tool
@p.tool
async def get_weather(context: ToolContext, city: str) -> ToolResult:
    return ToolResult(f"Sunny in {city}, 25°C")

@p.tool
async def get_order_status(context: ToolContext, order_id: str) -> ToolResult:
    return ToolResult(data={"status": "delivered", "order_id": order_id})

# 将 Tool 附加到 Agent
async with p.Server() as server:
    agent = await server.create_agent(name="ServiceAgent", description="")

    await agent.attach_tool(
        tool=get_weather,
        condition="User asks about weather",
    )

    await agent.attach_tool(
        tool=get_order_status,
        condition="User asks about order status",
    )
```

### 5.5 在 Tool 中访问上下文

```python
@p.tool
async def personalized_tool(context: ToolContext) -> ToolResult:
    customer_id = p.Customer.current.id
    agent = await p.ToolContextAccessor(context).server.find_agent(id=context.agent_id)
    session_id = context.session_id

    return ToolResult(data={"customer": customer_id, "agent": agent.name})
```

### 5.6 Tool 访问客户信息示例

以 `Test_that_a_tool_can_access_current_customer` 为例，测试 **Tool 访问当前客户信息**：

```python
class Test_that_a_tool_can_access_current_customer(SDKTest):
    async def setup(self, server: p.Server) -> None:
        self.tool_called = False

        self.agent = await server.create_agent(
            name="Tool Test Agent",
            description="Agent for testing tool invocation",
        )

        self.customer = await server.create_customer(name="Test Customer")

        self.id_of_customer_in_session: str | None = None

        @p.tool
        async def set_flag_tool(context: ToolContext) -> ToolResult:
            # 关键代码：在 Tool 中访问当前客户
            self.id_of_customer_in_session = p.Customer.current.id
            return ToolResult({})

        await self.agent.attach_tool(
            tool=set_flag_tool,
            condition="the user asks to set the flag or trigger the tool",
        )

    async def run(self, ctx: Context) -> None:
        await ctx.send_and_receive_message(
            customer_message="Please set the flag for me",
            recipient=self.agent,
            sender=self.customer,
        )

        assert self.id_of_customer_in_session == self.customer.id
```

**关键点：**
- `p.Customer.current`：在 Tool 内部访问当前客户的快捷方式
- `sender=self.customer`：在发送消息时指定发送者
- Tool 可以通过 `context` 获取会话中的客户信息

**测试结论：**

| 步骤 | 操作 | 结果 |
|------|------|------|
| 1 | 创建 Agent 和测试客户 | ✓ |
| 2 | 创建 Tool，在 Tool 中获取当前客户 ID | ✓ |
| 3 | 将 Tool 附加到 Agent | ✓ |
| 4 | 以测试客户身份发送消息，触发 Tool | ✓ |
| 5 | Tool 正确返回了当前客户的 ID | ✓ |

**功能价值**：验证 **Tool 可以访问当前会话中的客户信息** —— Tool 能够知道是谁在与它对话，从而提供个性化的服务。

**业务价值：**

| 场景 | 说明 |
|------|------|
| 个性化服务 | Tool 可以根据客户身份提供定制化的响应 |
| 数据查询 | Tool 可以查询当前客户的订单、历史记录等信息 |
| 权限控制 | Tool 可以根据客户权限级别限制操作 |

**实际案例：**
```python
# 客服场景：查询客户订单状态
@p.tool
async def check_order_status(context: ToolContext, order_id: str) -> ToolResult:
    customer_id = p.Customer.current.id
    customer = await get_customer_by_id(customer_id)

    if customer.type == "premium":
        return ToolResult(data={"priority": "high", "status": get_priority_status(order_id)})
    else:
        return ToolResult(data={"priority": "normal", "status": get_normal_status(order_id)})
```

---

## 6. Glossary（术语）

### 6.1 测试目标

验证领域词汇（术语）的创建和管理功能。

### 6.2 运行测试

```bash
pytest tests/sdk/test_glossary.py -v
```

### 6.3 核心测试用例

| 测试类 | 功能描述 | 关键代码 |
|--------|----------|----------|
| `Test_that_a_glossary_term_can_be_created` | 创建词汇术语 | `agent.create_term(name="...", description="...")` |
| `Test_that_a_glossary_term_can_be_created_with_custom_id` | 使用自定义 ID | `agent.create_term(name="...", id=TermId("custom-id"))` |

### 6.4 Glossary 创建示例

```python
from parlerant.core.glossary import TermId
import parlant.sdk as p

async def test_glossary():
    async with p.Server() as server:
        agent = await server.create_agent(name="MedicalAgent", description="Healthcare assistant")

        term1 = await agent.create_term(
            name="Office Phone Number",
            description="The phone number of our office, at +1-234-567-8900",
        )

        term2 = await agent.create_term(
            name="Office Hours",
            description="Office hours are Monday to Friday, 9 AM to 5 PM",
        )

        term3 = await agent.create_term(
            name="Dr. Smith",
            synonyms=["Dr. S", "Smith Doctor"],
            description="The primary care physician",
        )

        custom_term = await agent.create_term(
            name="Emergency Contact",
            description="Emergency contact number: 911",
            id=TermId("emergency-phone"),
        )
```

### 6.5 术语创建示例

以 `Test_that_a_glossary_term_can_be_created` 为例，测试 **Glossary 术语创建**：

```python
class Test_that_a_glossary_term_can_be_created(SDKTest):
    async def setup(self, server: p.Server) -> None:
        self.agent = await server.create_agent(
            name="Rel Agent",
            description="Agent for guideline relationships",
        )

        # 关键代码：创建 Glossary 术语，包含同义词
        self.term = await self.agent.create_term(
            name="Priority",
            description="Indicates something should be prioritized over another.",
            synonyms=["importance", "precedence"],
        )

    async def run(self, ctx: Context) -> None:
        glossary_store = ctx.container[GlossaryStore]
        term = await glossary_store.read_term(self.term.id)

        assert term.name == "Priority"
        assert term.description == "Indicates something should be prioritized over another."
        assert term.synonyms == ["importance", "precedence"]
        assert term.id == self.term.id
```

**关键点：**
- `agent.create_term()`：创建 Glossary 术语
- `synonyms` 参数：定义术语的同义词，帮助 Agent 理解用户的不同表达方式
- `GlossaryStore`：存储和读取术语的存储类

**测试结论：**

| 步骤 | 操作 | 结果 |
|------|------|------|
| 1 | 创建 Agent | ✓ |
| 2 | 创建 Glossary 术语"Priority"，包含同义词 | ✓ |
| 3 | 从存储中读取术语 | ✓ |
| 4 | 验证术语的 name、description、synonyms 都正确 | ✓ |

**功能价值**：验证 **Glossary 术语的创建和管理** —— Agent 可以定义专业术语及其同义词，确保在对话中正确理解和使用这些术语。

**业务价值：**

| 场景 | 说明 |
|------|------|
| 专业术语统一 | 确保 Agent 和用户对专业术语有共同的理解 |
| 同义词识别 | 用户可以使用不同表达方式，Agent 都能理解 |
| 行业知识库 | 为特定行业构建术语知识库 |

**实际案例：**
```python
# 医疗场景
await agent.create_term(
    name="Myocardial Infarction",
    synonyms=["heart attack", "cardiac arrest", "MI"],
    description="A heart attack occurs when blood flow to the heart is blocked.",
)

# 法律场景
await agent.create_term(
    name="Force Majeure",
    synonyms=["act of God", "unforeseen circumstances"],
    description="Unforeseen circumstances that prevent someone from fulfilling a contract.",
)
```

---

## 7. Canned Responses（预设回复）

### 7.1 测试目标

验证预设回复模板的创建、字段依赖和动态内容填充功能。

### 7.2 运行测试

```bash
pytest tests/sdk/test_canned_responses.py -v
```

### 7.3 核心测试用例

| 测试类 | 功能描述 | 关键代码 |
|--------|----------|----------|
| `Test_that_canned_response_can_be_created_with_field_dependencies` | 创建带字段依赖的回复 | `create_canned_response(template="...", field_dependencies=["order"])` |
| `Test_that_canned_response_with_field_dependency_is_excluded_when_field_unavailable` | 字段不可用时排除 | 字段依赖机制自动处理 |

### 7.4 Canned Response 示例

```python
import parlant.sdk as p

async def test_canned_responses():
    async with p.Server() as server:
        agent = await server.create_agent(name="SupportAgent", description="")

        greeting = await agent.create_canned_response(
            template="Hello {name}! How can I help you today?",
        )

        order_status = await agent.create_canned_response(
            template="Your order #{order_number} is {status}.",
            field_dependencies=["order_number", "status"],
        )

        await agent.create_guideline(
            condition="Customer asks about order",
            action="Tell them their order status",
            canned_responses=[order_status],
        )
```

### 7.5 动态字段提供者

```python
async def provide_order_fields(ctx: p.EngineContext) -> dict[str, str]:
    return {
        "order_number": "12345",
        "status": "shipped",
        "estimated_delivery": "2026-02-20",
    }

await agent.create_guideline(
    condition="Customer asks about order",
    action="Provide order details",
    composition_mode=p.CompositionMode.STRICT,
    canned_responses=[order_status],
    canned_response_field_provider=provide_order_fields,
)
```

### 7.6 字段依赖测试示例

以 `Test_that_canned_response_with_field_dependency_is_excluded_when_field_unavailable` 为例，测试 **Canned Response 字段依赖**：

```python
class Test_that_canned_response_with_field_dependency_is_excluded_when_field_unavailable(SDKTest):
    async def setup(self, server: p.Server) -> None:
        self.agent = await server.create_agent(
            name="Test Agent",
            description="",
        )

        # 关键代码：创建带字段依赖的 Canned Response
        canrep_with_dependency = await self.agent.create_canned_response(
            template="Your order is ready for pickup.",
            field_dependencies=["order"],
        )

        await self.agent.create_guideline(
            condition="Customer asks about their order",
            action="Tell them that their order is ready for pickup",
            composition_mode=p.CompositionMode.STRICT,
            canned_responses=[canrep_with_dependency],
        )

    async def run(self, ctx: Context) -> None:
        response = await ctx.send_and_receive_message(
            customer_message="What about my order?",
            recipient=self.agent,
        )

        # 由于字段不可用，Canned Response 被排除
        assert "order" not in response.lower()
```

**关键点：**
- `field_dependencies`：指定 Canned Response 依赖的字段
- `composition_mode=p.CompositionMode.STRICT`：严格模式，缺少依赖字段时排除该响应

**测试结论：**

| 步骤 | 操作 | 结果 |
|------|------|------|
| 1 | 创建 Canned Response，依赖 "order" 字段 | ✓ |
| 2 | 在 Guideline 中使用该 Canned Response | ✓ |
| 3 | 没有工具提供 "order" 字段 | ✓ |
| 4 | 询问订单时，Canned Response 被排除 | ✓ |
| 5 | Agent 返回一个不包含 "order" 的回退响应 | ✓ |

**功能价值**：验证 **Canned Response 字段依赖机制** —— 确保回复只在所需数据可用时才使用，避免生成不完整或不准确的响应。

**业务价值：**

| 场景 | 说明 |
|------|------|
| 条件性回复 | 回复只在特定数据可用时显示 |
| 数据完整性 | 避免显示占位符或未填充的数据 |
| 优雅降级 | 当数据缺失时，提供替代响应 |

**实际案例：**
```python
# 电商场景：订单状态回复
order_status_response = await agent.create_canned_response(
    template="Your order #{order_number} shipped on {ship_date}.",
    field_dependencies=["order_number", "ship_date"],
)

await agent.create_guideline(
    condition="Customer asks about order status",
    action="Provide shipping information",
    composition_mode=p.CompositionMode.STRICT,
    canned_responses=[order_status_response],
)

# 当 order_number 或 ship_date 不可用时，这个回复会被跳过
```

---

## 8. Context Variables（上下文变量）

### 8.1 测试目标

验证上下文变量的创建、工具关联、值设置和获取功能。

### 8.2 运行测试

```bash
pytest tests/sdk/test_variables.py -v
```

### 8.3 核心测试用例

| 测试类 | 功能描述 | 关键代码 |
|--------|----------|----------|
| `Test_that_a_static_value_variable_can_be_created` | 创建静态变量 | `agent.create_variable(name="...", description="...")` |
| `Test_that_a_tool_enabled_variable_can_be_created` | 创建工具驱动的变量 | `create_variable(name="...", tool=get_value_tool)` |
| `Test_that_a_variable_value_can_be_set_for_a_customer` | 为客户设置变量值 | `variable.set_value_for_customer(customer, "value")` |
| `Test_that_a_variable_value_can_be_set_for_a_tag` | 为标签设置变量值 | `variable.set_value_for_tag(tag_id, "value")` |
| `Test_that_a_variable_value_can_be_set_globally` | 设置全局变量值 | `variable.set_global_value("value")` |

### 8.4 变量创建示例

```python
import parlant.sdk as p

async def test_variables():
    async with p.Server() as server:
        agent = await server.create_agent(name="Agent", description="")
        customer = await server.create_customer(name="John Doe")
        tag = await server.create_tag("premium_users")

        # 1. 创建静态变量
        static_var = await agent.create_variable(
            name="subscription_plan",
            description="The current subscription plan",
        )

        # 2. 创建工具驱动的变量
        @p.tool
        async def get_datetime(context: p.ToolContext) -> p.ToolResult:
            from datetime import datetime
            return p.ToolResult(datetime.now().isoformat())

        tool_var = await agent.create_variable(
            name="current_time",
            description="Current timestamp",
            tool=get_datetime,
        )

        # 3. 设置变量值
        await static_var.set_value_for_customer(customer, "premium")
        await static_var.set_value_for_tag(tag.id, "gold")
        await static_var.set_global_value("free")

        # 4. 获取变量值
        customer_value = await static_var.get_value_for_customer(customer)
        tag_value = await static_var.get_value_for_tag(tag.id)
        global_value = await static_var.get_global_value()

        print(f"Customer value: {customer_value}")  # premium
        print(f"Tag value: {tag_value}")            # gold
        print(f"Global value: {global_value}")      # free
```

### 8.5 客户级别值设置示例

以 `Test_that_a_variable_value_can_be_set_for_a_customer` 为例，测试 **Context Variable 客户级别值设置**：

```python
class Test_that_a_variable_value_can_be_set_for_a_customer(SDKTest):
    async def setup(self, server: p.Server) -> None:
        self.agent = await server.create_agent(
            name="Rel Agent",
            description="Agent for testing context variables",
        )

        self.customer = await server.create_customer("John Doe")

        self.variable = await self.agent.create_variable(
            name="subscription_plan",
            description="The current subscription plan of the user.",
        )

        await self.variable.set_value_for_customer(self.customer, "premium")

    async def run(self, ctx: Context) -> None:
        assert "premium" == await self.variable.get_value_for_customer(self.customer)
```

**关键点：**
- `set_value_for_customer(customer, value)`：为特定客户设置变量值
- `get_value_for_customer(customer)`：获取特定客户的变量值
- Context Variable 支持多级别优先级：**客户级别 > 标签级别 > 全局级别**

**测试结论：**

| 步骤 | 操作 | 结果 |
|------|------|------|
| 1 | 创建 Agent 和测试客户 | ✓ |
| 2 | 创建 Context Variable | ✓ |
| 3 | 为特定客户设置变量值为 "premium" | ✓ |
| 4 | 获取该客户的变量值，返回 "premium" | ✓ |

**功能价值**：验证 **Context Variable 的客户级别值管理** —— 可以为不同客户设置个性化的变量值，实现千人千面的对话体验。

**业务价值：**

| 场景 | 说明 |
|------|------|
| 个性化订阅计划 | 不同客户有不同的订阅级别，显示不同的内容和功能 |
| 会员等级 | 根据会员等级（普通、银卡、金卡）提供不同服务 |
| 用户偏好 | 记住用户的语言偏好、沟通风格等设置 |

**实际案例：**
```python
# SaaS 场景：订阅计划
subscription_var = await agent.create_variable(
    name="subscription_plan",
    description="The user's subscription plan",
)

await subscription_var.set_value_for_customer(premium_customer, "enterprise")
await subscription_var.set_value_for_customer(basic_customer, "basic")
await subscription_var.set_global_value("free")

# 在 Guideline 中使用变量
await agent.create_guideline(
    condition="Customer asks about features",
    action="Explain our service features",
)
```

---

## 9. Sessions（会话管理）

### 9.1 测试目标

验证客户创建、会话创建、消息发送和事件处理功能。

### 9.2 运行测试

```bash
pytest tests/sdk/test_customers.py -v
```

### 9.3 核心测试用例

| 测试类 | 功能描述 | 关键代码 |
|--------|----------|----------|
| `Test_that_a_customer_can_be_read` | 读取客户信息 | `customer_store.read_customer(customer_id)` |
| `Test_that_customers_can_be_listed` | 列出所有客户 | `server.list_customers()` |
| `Test_that_a_customer_can_be_found_by_name` | 通过名称查找客户 | `server.find_customer(name="John Doe")` |
| `Test_that_a_customer_can_be_found_by_id` | 通过 ID 查找客户 | `server.find_customer(id=...)` |
| `Test_that_a_customer_can_be_created_with_custom_id` | 创建带自定义 ID 的客户 | `server.create_customer(id="custom-id", name="...")` |

### 9.4 客户和会话示例

```python
import parlant.sdk as p

async def test_sessions():
    async with p.Server() as server:
        # 1. 创建客户
        customer = await server.create_customer(
            name="John Doe",
            metadata={"email": "john@example.com", "phone": "+1234567890"},
        )
        print(f"Customer created: {customer.name} (ID: {customer.id})")

        # 2. 创建 Agent
        agent = await server.create_agent(name="SupportAgent", description="")

        # 3. 创建会话
        session = await server.create_session(
            agent_id=agent.id,
            customer_id=customer.id,
        )
        print(f"Session created: {session.id}")

        # 4. 发送消息
        from tests.sdk.utils import get_message
        response = await get_message(
            session.send_message("Hello, I need help!"),
            timeout=30,
        )
        print(f"Agent response: {response}")

        # 5. 查找客户
        found = await server.find_customer(name="John Doe")
        print(f"Found customer: {found.name}")
```

### 9.5 完整会话流程测试

```python
async def test_complete_conversation():
    async with p.Server() as server:
        agent = await server.create_agent(name="TestAgent", description="")

        await agent.create_guideline(
            condition="Customer greets you",
            action="Greet them warmly",
        )

        session = await server.create_session(
            agent_id=agent.id,
            customer_id=customer.id,
            allow_greeting=True,
        )

        customer_event = await session.create_event(
            kind="message",
            source="customer",
            message="Hi there!",
        )

        from tests.sdk.utils import get_message
        response = await get_message(
            session.wait_for_data(
                min_offset=customer_event.offset,
                source="ai_agent",
                wait=30,
            )
        )
        print(f"Response: {response}")
```

### 9.6 客户查找示例

以 `Test_that_a_customer_can_be_found_by_name` 为例，测试 **客户按名称查找**：

```python
class Test_that_a_customer_can_be_found_by_name(SDKTest):
    async def setup(self, server: p.Server) -> None:
        self.c1 = await server.create_customer(name="John Doe")
        self.c2 = await server.create_customer(name="Jane Smith")

        # 关键代码：通过名称查找客户
        self.customer = await server.find_customer(name="John Doe")

    async def run(self, ctx: Context) -> None:
        assert self.customer is not None
        assert self.customer.id == self.c1.id
```

**关键点：**
- `server.find_customer(name="...")`：通过客户名称查找客户
- `server.create_customer()`：创建新客户
- `server.list_customers()`：列出所有客户

**测试结论：**

| 步骤 | 操作 | 结果 |
|------|------|------|
| 1 | 创建两个客户：John Doe 和 Jane Smith | ✓ |
| 2 | 通过名称 "John Doe" 查找客户 | ✓ |
| 3 | 找到的客户 ID 与创建的 ID 一致 | ✓ |
| 4 | 验证返回的客户对象不为空 | ✓ |

**功能价值**：验证 **客户查找和管理功能** —— 可以通过不同方式（名称、ID）查找和管理客户信息。

**业务价值：**

| 场景 | 说明 |
|------|------|
| 客户识别 | 通过名称或其他标识快速找到已有客户 |
| 客户去重 | 避免重复创建同一客户，维护客户数据一致性 |
| 个性化服务 | 根据客户身份提供定制化的对话体验 |

**实际案例：**
```python
# 客服场景：客户身份识别
async with p.Server() as server:
    customer = await server.find_customer(name="John Doe")

    if customer is None:
        customer = await server.create_customer(
            name="John Doe",
            metadata={"email": "john@example.com", "phone": "+1234567890"},
        )

    session = await server.create_session(
        agent_id=agent.id,
        customer_id=customer.id,
    )
```

---

## 10. 常见问题

### 10.1 测试超时

如果测试超时，检查：
- LLM API Key 是否正确配置
- 网络连接是否正常
- 增加超时时间

### 10.2 依赖缺失

```bash
pip install -e .
```

---

## 11. 测试最佳实践

### 11.1 先运行现有测试

```bash
# 开发新测试前，先确保现有测试能通过
pytest tests/sdk/test_agents.py -v
```

**为什么？** 确保你的环境配置正确，避免开发新测试时因环境问题浪费时间。

---

### 11.2 使用 SDKTest 基类

```python
# ✅ 正确：继承 SDKTest
class Test_that_an_agent_can_be_created(SDKTest):
    async def setup(self, server: p.Server) -> None:
        ...

# ❌ 错误：直接继承 pytest 的 TestCase
class Test_that_an_agent_can_be_created(unittest.TestCase):
    ...
```

**为什么？** `SDKTest` 封装了测试初始化、上下文管理等逻辑，直接继承即可使用。

---

### 11.3 使用 `async def`

```python
# ✅ 正确
async def test_agent_creation(self, server: p.Server) -> None:
    agent = await server.create_agent(...)

# ❌ 错误：不能用普通函数
def test_agent_creation(self, server: p.Server) -> None:
    agent = await server.create_agent(...)  # SyntaxError
```

**为什么？** SDK 所有 API 都是异步的，必须用 `async def` 定义测试方法。

---

### 11.4 使用 `await`

```python
async def test_agent_creation(self, server: p.Server) -> None:
    # ✅ 正确：所有异步操作都要 await
    agent = await server.create_agent(name="Test")

    # ❌ 错误：不 await 会返回协程对象，而不是实际结果
    result = server.create_agent(name="Test")  # <coroutine object>
```

**为什么？** `await` 会等待异步操作完成，获取实际返回值。

---

### 11.5 清理测试数据

```python
class Test_that_agents_can_be_listed(SDKTest):
    async def setup(self, server: p.Server) -> None:
        self.agent = await server.create_agent(name="Test Agent")
        # SDKTest 自动清理，无需手动删除

    async def run(self, ctx: Context) -> None:
        agents = await ctx.server.list_agents()
        # 每个测试独立运行，互不干扰
```

**为什么？** 测试之间互相独立，一个测试创建的 Agent 不会影响其他测试。

---

### 11.6 使用 `reuse_session=True`

```python
class Test_that_conversation_flow(SDKTest):
    async def setup(self, server: p.Server) -> None:
        self.agent = await server.create_agent(name="Test")

    async def run(self, ctx: Context) -> None:
        # 第一次对话
        response1 = await ctx.send_and_receive_message(
            customer_message="Hello",
            recipient=self.agent,
            reuse_session=True  # ✅ 在同一会话中继续对话
        )

        # 第二次对话（上下文保持）
        response2 = await ctx.send_and_receive_message(
            customer_message="What is my order?",
            recipient=self.agent,
        )
```

**为什么？** `reuse_session=True` 让多轮对话共享上下文，模拟真实聊天场景。

---

## 12. 总结

本文档涵盖了 Parlant SDK 的 8 个核心特性测试：

| 特性 | 测试文件 | 主要功能 |
|------|----------|----------|
| Agents | test_agents.py | 创建、读取、查找、策略配置 |
| Guidelines | test_guidelines.py | 创建、优先级、关系、匹配器 |
| Journeys | test_journeys.py | 创建、状态转换、条件分支、链接 |
| Tools | test_tools.py | 创建、附加、上下文访问 |
| Glossary | test_glossary.py | 术语创建、同义词管理 |
| Canned Responses | test_canned_responses.py | 模板创建、字段依赖 |
| Context Variables | test_variables.py | 变量创建、值设置、工具驱动 |
| Sessions/Customers | test_customers.py | 客户管理、会话管理 |

运行完整测试套件：
```bash
cd /home/project/parlant
pytest tests/sdk/ -v --tb=short
```

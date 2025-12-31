# Screenshot 处理流程详解

本文档详细介绍了 MIRIX 系统中各个 Memory Agent 如何分析 screenshots 并提取信息保存到相应的记忆类型中。

## 整体架构概述

MIRIX 系统采用多 Agent 架构来处理 screenshots，整个流程分为以下阶段：

1. **Screenshot 采集阶段**：每秒采集一次 screenshot，当积累到一定数量时发送处理
2. **Meta Memory Agent 分类阶段**：分析 screenshots 和对话，决定哪些记忆类型需要更新
3. **并行处理阶段**：各个专门的 Memory Agent 并行处理各自的记忆更新任务
4. **信息提取和保存阶段**：每个 Agent 从 screenshots 中提取特定类型的信息并保存

## 核心组件

### 1. Meta Memory Agent（元记忆管理器）

**职责**：作为协调者，分析 screenshots 并决定需要更新哪些记忆类型。

**处理流程**：
1. 接收 screenshots（通常约 20 张）和用户与 Chat Agent 的对话记录
2. 分析用户活动和屏幕内容
3. 按照决策框架评估每个记忆类型：
   - Core Memory：用户身份和交互偏好
   - Episodic Memory：时间排序的事件和经历
   - Procedural Memory：步骤式指令和流程
   - Resource Memory：文档和文件
   - Knowledge：结构化事实数据
   - Semantic Memory：概念性知识
4. 调用 `trigger_memory_update` 函数，传入需要更新的记忆类型列表（如 `['core', 'episodic', 'procedural']`）

**关键决策规则**（按优先级顺序）：
- **Core Memory**：揭示用户身份或沟通偏好的信息
- **Episodic Memory**：几乎所有用户活动都需要记录（几乎总是需要更新）
- **Procedural Memory**：显示步骤式流程或教程的内容
- **Resource Memory**：涉及文件、文档、图像等资源
- **Knowledge**：包含静态参考数据（如密码、电话号码、API 密钥）
- **Semantic Memory**：提到新的概念、人物、地点或组织

**代码位置**：
- Prompt: `mirix/prompts/system/screen_monitor/meta_memory_agent.txt`
- 实现: `mirix/functions/function_sets/memory_tools.py` 中的 `trigger_memory_update` 函数

---

### 2. Episodic Memory Agent（情景记忆管理器）

**职责**：从 screenshots 中提取时间排序的事件和活动，保存用户的"日记"。

**处理的 Screenshot 内容**：
- 用户正在进行的活动（如看电影、编码、浏览网页）
- 具体事件的细节（时间、地点、参与者、内容）
- 用户与系统的交互历史

**提取的信息类型**：
- `event_type`：事件类型（如 user_message, system_notification）
- `summary`：事件的简短摘要（如 "John 正在观看电影 'Inception'"）
- `details`：详细描述（包含尽可能多的细节，如使用的应用、观看内容的具体描述等）
- `actor`：事件的发起者（user 或 assistant）
- `occurred_at`：事件发生的时间

**处理流程**：
1. 接收 screenshots 和 Meta Memory Agent 的指令（格式：`[Instruction from Meta Memory Manager]`）
2. 分析所有 screenshots 以理解用户活动
3. 识别需要记录的最重要事件
4. 选择操作：
   - `episodic_memory_insert`：插入新事件（当发生显著变化时）
   - `episodic_memory_merge`：合并到现有事件（当活动持续时）
   - `episodic_memory_replace`：替换现有事件（当需要整合重复项时）
5. 执行**单一函数调用**来保存信息

**示例**：
```
Summary: "John 正在使用 Chrome 打开 Netflix 网站，观看电影 'Inception'"
Details: "John 正在使用 Chrome，打开 Netflix 网站，观看电影 'Inception'。他正在全屏观看。从前 20 秒的影片内容来看，这部电影是关于...（根据看到的画面填写详细描述）。"
```

**代码位置**：
- Prompt: `mirix/prompts/system/screen_monitor/episodic_memory_agent.txt`
- 实现: `mirix/agent/episodic_memory_agent.py`

---

### 3. Procedural Memory Agent（程序性记忆管理器）

**职责**：从 screenshots 中提取步骤式指令、工作流程和"如何做"指南。

**处理的 Screenshot 内容**：
- 显示工作流程或教程的屏幕
- 用户遵循的步骤式过程
- 操作指南和脚本

**提取的信息类型**：
- `entry_type`：条目类型（'workflow', 'guide', 'script'）
- `description`：简短描述文本
- `steps`：结构化的步骤列表（JSON 格式）

**处理流程**：
1. 接收 screenshots 和 Meta Memory Agent 的指令
2. 分析 screenshots 以识别步骤式指令、工作流程或程序性知识
3. 提取完整的步骤序列
4. 使用 `procedural_memory_insert` 或 `procedural_memory_update` 保存信息
5. 执行**单一函数调用**

**示例**：
```
Entry Type: 'workflow'
Description: "Git 工作流程"
Steps: {
  "1": "创建分支",
  "2": "进行更改",
  "3": "提交",
  "4": "推送",
  "5": "创建 Pull Request"
}
```

**代码位置**：
- Prompt: `mirix/prompts/system/screen_monitor/procedural_memory_agent.txt`
- 实现: `mirix/agent/procedural_memory_agent.py`

---

### 4. Resource Memory Agent（资源记忆管理器）

**职责**：从 screenshots 中提取文档、文件和参考材料的内容。

**处理的 Screenshot 内容**：
- 打开的文档和文件
- 代码编辑器中的代码
- 查看的网页内容
- 错误消息截图

**提取的信息类型**：
- `title`：资源名称/标题
- `summary`：简短描述（包括项目来源、页面信息、内容概述等）
- `resource_type`：文件类型（'doc', 'markdown', 'pdf_text', 'code' 等）
- `content`：完整的文本内容（尽可能详细，可以是部分内容）

**处理流程**：
1. 接收 screenshots 和 Meta Memory Agent 的指令
2. 识别 screenshots 中的文档、文件和参考材料
3. **提取实际文本内容**（不使用占位符，如 "Content of file.py"）
4. 判断是更新现有资源还是创建新资源
5. 使用 `resource_memory_insert` 或 `resource_memory_update` 保存信息
6. 执行**单一函数调用**

**重要特点**：
- `content` 字段必须包含真实的、详细的内容，而不是描述
- 内容可以很长，这是可以接受的
- 当更新现有资源时，会添加新信息或填补之前不完整捕获的空白

**代码位置**：
- Prompt: `mirix/prompts/system/screen_monitor/resource_memory_agent.txt`
- 实现: `mirix/agent/resource_memory_agent.py`

---

### 5. Knowledge Memory Agent（知识记忆管理器）

**职责**：从 screenshots 中提取结构化的、可检索的事实数据，如凭证、联系信息等。

**处理的 Screenshot 内容**：
- 显示的密码、API 密钥
- 电话号码、电子邮件地址
- URL、书签
- 账户用户名
- 数据库连接字符串

**提取的信息类型**：
- `entry_type`：条目类型（'credential', 'bookmark', 'api_key', 'phone_number' 等）
- `source`：数据来源（如 'github', 'google', 'user_provided'）
- `sensitivity`：敏感度级别（'low', 'medium', 'high'）
- `secret_value`：实际存储的数据值

**处理流程**：
1. 接收 screenshots 和 Meta Memory Agent 的指令
2. 识别**仅结构化的事实数据**（不包括概念性信息）
3. 区分：
   - Knowledge = "John 的电话号码是多少？" → 存储："555-1234"
   - Semantic Memory = "John 是谁？" → 存储："John 是项目经理..."
4. 使用 `knowledge_insert` 或 `knowledge_update` 保存信息
5. 执行**单一函数调用**

**关键区别**：
- **Knowledge**：离散的数据点，用于快速查找（如电话号码、API 密钥）
- **Semantic Memory**：解释性或概念性信息（如某人是谁、某物的含义）

**代码位置**：
- Prompt: `mirix/prompts/system/screen_monitor/knowledge_memory_agent.txt`
- 实现: `mirix/agent/knowledge_memory_agent.py`

---

### 6. Semantic Memory Agent（语义记忆管理器）

**职责**：从 screenshots 中提取概念性知识、定义和对实体/对象的一般理解。

**处理的 Screenshot 内容**：
- 新的概念或软件名称
- 人物的信息（身份、特征、关系）
- 地点的信息
- 组织的背景

**提取的信息类型**：
- `name`：概念或对象的名称
- `summary`：简洁的解释或摘要
- `details`：扩展描述（包含上下文、示例或深入见解）
- `source`：知识来源的引用
- `tree_path`：层次分类路径（数组格式，如 `['favorites', 'pets', 'dog']`）

**处理流程**：
1. 接收 screenshots 和 Meta Memory Agent 的指令
2. 识别**新的概念**（不包括常识知识，如 "VS Code", "Google Chrome"）
3. 判断是添加新概念还是更新现有概念
4. 使用 `semantic_memory_insert` 或 `semantic_memory_update` 保存信息
5. 执行**单一函数调用**

**重要规则**：
- 仅保存对系统来说是**新的**概念
- 不要保存常识知识（除非对用户有特殊含义）
- 使用 `tree_path` 进行层次化组织

**示例**：
```
Name: "MemoryLLM"
Summary: "MemoryLLM 是一种为大型语言模型设计的内存架构类型"
Details: "MemoryLLM 使用情景记忆来维护上下文，它通过...（详细描述）"
Tree Path: ['ai', 'architecture', 'memory']
```

**代码位置**：
- Prompt: `mirix/prompts/system/screen_monitor/semantic_memory_agent.txt`
- 实现: `mirix/agent/semantic_memory_agent.py`

---

### 7. Core Memory Agent（核心记忆管理器）

**职责**：从 screenshots 中提取用户的基本信息、个性和偏好。

**处理的 Screenshot 内容**：
- 用户行为和偏好模式
- 应用使用偏好
- 沟通风格暗示
- 个人资料信息

**提取的信息类型**（分为两个块）：
- `human` 块：用户的个人信息、偏好、 personality traits
- `persona` 块：助手的 persona 和沟通风格偏好

**处理流程**：
1. 接收 screenshots 和 Meta Memory Agent 的指令
2. 深入分析 screenshots 以提取**所有**用户偏好和个人信息的细节
3. 识别用户行为模式、偏好和个人详细信息
4. 使用 `core_memory_append` 添加新信息或 `core_memory_rewrite` 压缩内容（当块超过 90% 满时）
5. 执行**单一函数调用**

**重要特点**：
- 核心记忆对于理解用户至关重要，需要持久保存
- 即使类似信息在其他记忆组件中，也要更新核心记忆
- 专注于用户偏好、个人事实、个性特征和任何能改善未来互动的细节

**示例**：
```
Human Block: "Name: Alice\nJob: Engineer at TechCorp\nHobbies: Hiking and photography\nPrefers direct communication"
Persona Block: "Assistant's communication style: Friendly and concise, adapts to user preferences"
```

**代码位置**：
- Prompt: `mirix/prompts/system/screen_monitor/core_memory_agent.txt`
- 实现: `mirix/agent/core_memory_agent.py`

---

## 技术实现细节

### 并行处理机制

当 Meta Memory Agent 调用 `trigger_memory_update` 时，系统使用 `ThreadPoolExecutor` 来并行执行多个 Memory Agent 的处理：

```python
# 来自 mirix/functions/function_sets/memory_tools.py
with ThreadPoolExecutor(max_workers=max_workers) as executor:
    future_to_index = {
        executor.submit(_run_single_memory_update, memory_type): index
        for index, memory_type in enumerate(memory_types)
    }
```

这意味着如果 Meta Memory Agent 决定更新 `['episodic', 'procedural', 'knowledge']`，这三个 Agent 会同时并行处理各自的 screenshots 分析任务。

### 消息传递流程

1. **初始输入**：Screenshots 和对话作为消息发送给 Meta Memory Agent
2. **分类决策**：Meta Memory Agent 分析并调用 `trigger_memory_update(memory_types=['...'])`
3. **消息构建**：每个 Memory Agent 接收：
   - 原始的 screenshots 消息（通过 `user_message` 传递）
   - 系统消息：`"[System Message] According to the instructions, the retrieved memories and the above content, update the corresponding memory."`
   - Meta Memory Agent 的指令（如果有）：`[Instruction from Meta Memory Manager]`
4. **并行执行**：各个 Agent 独立分析 screenshots 并提取信息
5. **保存到数据库**：每个 Agent 使用相应的数据库操作函数保存提取的信息

### 单次调用限制

每个 Memory Agent 都遵循"单次函数调用"原则：
- 在接收到 screenshots 后，每个 Agent 应该只调用**一个**记忆更新函数
- 如果不需要更新，则调用 `finish_memory_update`
- 这个设计确保了处理效率和一致性

### Screenshot 格式

Screenshots 通过消息的 `content` 字段传递，格式为：
```python
{
    "type": "image_url",
    "image_url": {
        "url": "data:image/png;base64,..."  # 或 file_id
    }
}
```

系统支持：
- Base64 编码的图像数据
- 文件 ID（引用已保存的图像文件）
- Google Cloud URI（用于云存储的图像）

---

## 数据流图

```
Screenshots (约20张) + 对话记录
           ↓
    Meta Memory Agent
    (分析和分类)
           ↓
    trigger_memory_update
    (memory_types=['core', 'episodic', ...])
           ↓
    ┌──────┴──────┬──────────┬─────────┐
    ↓             ↓          ↓         ↓
Core Agent   Episodic   Procedural  Resource  Knowledge  Semantic
    ↓             ↓          ↓         ↓         ↓         ↓
各自分析 screenshots 并提取特定类型的信息
    ↓             ↓          ↓         ↓         ↓         ↓
保存到对应的数据库表
```

---

## 总结

MIRIX 系统通过多 Agent 协作的方式，将 screenshot 分析任务分解为不同的记忆类型处理：

1. **Meta Memory Agent** 作为协调者，负责分类和路由
2. **各个专门的 Memory Agent** 并行处理，各自专注于特定类型的信息提取
3. **单次调用原则** 确保处理效率
4. **详细的 Prompt 设计** 指导每个 Agent 准确地从 screenshots 中提取和保存信息

这种设计既保证了信息提取的准确性，又通过并行处理提高了系统的整体性能。


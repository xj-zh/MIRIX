# Screenshot Event 划分和连续性机制详解

本文档详细解释 MIRIX 系统如何将连续的多个 screenshots 按照 event 进行划分，以及 event 划分的连续性判断机制。

## 整体流程概述

```
连续 Screenshots（约20张，每秒1张）
           ↓
    Episodic Memory Agent
    (接收 + 分析)
           ↓
    ┌─────────────────────┐
    │  系统提供已有事件    │
    │  - 最近的事件列表    │
    │  - 最相关的事件列表  │
    └─────────────────────┘
           ↓
    LLM 智能判断：
    - 当前 screenshots 是否属于已有事件？
    - 还是新的事件？
           ↓
    ┌──────┴──────┐
    │              │
  继续同一活动     新活动
    │              │
    merge()      insert()
```

## 1. Screenshot 输入方式

### 1.1 批量输入
- **采集频率**：每秒采集 1 张 screenshot
- **批量大小**：当积累到一定数量（通常约 **20 张**）时，作为一批发送给 Agent 处理
- **时间跨度**：每批 screenshots 大约覆盖 **20 秒**的用户活动

### 1.2 消息格式
Screenshots 通过消息的 `content` 字段传递，包含多张图片：

```python
{
    "content": [
        {"type": "image_url", "image_url": {"url": "..."}},  # Screenshot 1
        {"type": "image_url", "image_url": {"url": "..."}},  # Screenshot 2
        # ... 最多约20张
    ]
}
```

## 2. Event 划分机制

### 2.1 划分逻辑（基于 LLM 智能判断）

Event 的划分**不是**通过硬编码的算法，而是依赖 **LLM 的语义理解和上下文分析**。Agent 会：

1. **分析当前 screenshots**：理解用户正在进行的活动
2. **对比已有事件**：查看系统 prompt 中提供的最近事件列表
3. **判断连续性**：决定是"继续同一活动"还是"开始新活动"

### 2.2 系统提供的上下文信息

在每次处理时，系统会在 system prompt 中为 Episodic Memory Agent 提供两类事件信息：

#### **最近的事件列表（Recent Events）**
- **数量**：最多 10 个（代码中 `MAX_RETRIEVAL_LIMIT_IN_SYSTEM = 10`，prompt 中提到 50 可能是期望值）
- **排序**：按 `occurred_at` 时间戳降序排列（最新的在前）
- **格式**：
```
[Event ID: event_123] Timestamp: 2025-03-05 10:15:30 - John is watching the movie 'Inception' (Details: 250 Characters)
[Event ID: event_122] Timestamp: 2025-03-05 10:10:00 - John started working on Python project (Details: 180 Characters)
...
```

#### **最相关的事件列表（Relevant Events）**
- **数量**：最多 10 个
- **排序**：通过语义搜索（embedding similarity）找到与当前 screenshots 内容最相关的事件
- **用途**：帮助 Agent 找到可能相关的历史事件，即使时间不是最近的

### 2.3 代码实现

**检索最近事件**（`mirix/agent/agent.py`）：
```python
current_episodic_memory = self.episodic_memory_manager.list_episodic_memory(
    agent_state=self.agent_state,
    user=self.user,
    limit=MAX_RETRIEVAL_LIMIT_IN_SYSTEM,  # 10
    timezone_str=timezone_str,
    fade_after_days=fade_after_days,
)
```

**检索相关事件**（通过语义搜索）：
```python
most_relevant_episodic_memory = self.episodic_memory_manager.list_episodic_memory(
    agent_state=self.agent_state,
    user=self.user,
    embedded_text=embedded_text,  # 当前 screenshots 的 embedding
    query=key_words,
    search_field="details",
    search_method=search_method,
    limit=MAX_RETRIEVAL_LIMIT_IN_SYSTEM,  # 10
)
```

## 3. Event 划分的连续性判断

### 3.1 三种操作类型

根据与已有事件的连续性关系，Agent 会选择不同的操作：

#### **1. `episodic_memory_merge` - 合并到现有事件**

**使用场景**：
- 用户**继续**做同一件事（如继续观看同一部电影、继续编码同一项目）
- 当前 screenshots 是已有事件的延续，只是增加了更多细节

**判断依据**：
- 当前 screenshots 显示的活动与已有事件的活动相同或高度相关
- 时间上连续或接近
- 内容是同一主题的延续

**示例**：
```
已有事件：
[Event ID: event_123] Timestamp: 10:15:30 - "John is watching the movie 'Inception'"

当前 screenshots（10:15:31-10:15:50）：
显示 John 仍在观看同一部电影，只是电影进度更靠后了

→ 使用 merge(event_123, ...) 更新该事件，添加新的细节
```

**重要注意事项**：
- `combined_summary` 会**完全覆盖**旧的 summary
- 因此 summary 必须包含旧信息和新信息，不能只说 "User continues watching..."
- `combined_details` 会**追加**到现有 details 中

#### **2. `episodic_memory_insert` - 插入新事件**

**使用场景**：
- 用户开始做**完全不同**的事情
- 当前 screenshots 显示的活动与已有事件没有直接关联

**判断依据**：
- 活动主题发生显著变化（如从看电影切换到写代码）
- 应用/任务/内容完全不同
- 时间上可能有间隔，或活动性质根本不同

**示例**：
```
已有事件：
[Event ID: event_123] Timestamp: 10:15:30 - "John is watching the movie 'Inception'"

当前 screenshots（10:16:00-10:16:20）：
显示 John 关闭了 Netflix，打开了 VS Code，开始编写 Python 代码

→ 使用 insert([new_event]) 创建新事件
```

#### **3. `episodic_memory_replace` - 替换现有事件**

**使用场景**：
- 需要整合重复的事件
- 需要重写过于冗长的 summary/details
- 需要合并多个相似事件为一个

**判断依据**：
- 发现系统中有重复或相似的事件需要合并
- 某个事件的 details 超过 5000 字符，需要精简

### 3.2 连续性的判断标准

由于使用 LLM 进行判断，连续性的判断标准是**语义和上下文相关的**，主要包括：

1. **活动主题相似性**
   - 是否涉及同一应用/网站/任务？
   - 是否处理同一内容/文件/项目？

2. **时间连续性**
   - 时间戳是否接近？
   - 是否在合理的时间窗口内？

3. **内容关联性**
   - Screenshots 显示的内容是否与已有事件的 summary/details 相关？
   - 是否有明确的延续关系？

4. **语义相似性**
   - 通过 embedding 向量计算相似度（用于检索最相关事件）

### 3.3 Prompt 指导原则

Episodic Memory Agent 的 prompt 中明确指导：

```text
**Single Function Call Process:**
1. **Analyze All Screenshots**: Review all 20 screenshots to understand the user's activities 
   and identify the most significant event that needs to be recorded.
2. **Choose Action**: Determine the most appropriate single action:
   - Use `episodic_memory_merge` for minor updates to existing events 
     (e.g., user continues watching the same movie). 
     Note that the new summary will overwrite the old summary, 
     so ensure it covers both earlier and new information.
   - Use `episodic_memory_insert` when significant changes occur 
     (e.g., user starts watching a new movie or doing something different).
   - Use `episodic_memory_replace` if you need to consolidate repeated items 
     or rewrite overly long summaries.
```

## 4. Event 划分的连续性特点

### 4.1 连续性是**语义连续**而非严格时间连续

- **语义连续**：只要活动主题相同，即使时间上有短暂间隔，也可能被认为是同一事件
- **例如**：用户暂停电影去回复消息，然后继续观看 → 可能仍然 merge 到同一事件

### 4.2 连续性判断是**上下文相关的**

- Agent 会考虑：
  - 最近的事件列表（时间上下文）
  - 最相关的事件列表（语义上下文）
  - 当前 screenshots 的完整内容

### 4.3 单次调用原则

- 每次接收到一批 screenshots（约 20 张），Agent 只调用**一个**记忆更新函数
- 这意味着：**一批 screenshots 最多只会影响一个 event**
- 如果需要处理多个事件，会在后续批次中处理

### 4.4 事件的时间戳

- 每个 event 有 `occurred_at` 时间戳
- 可以使用 API 参数 `occurred_at` 覆盖 LLM 提取的时间戳
- 对于 merge 操作，时间戳通常保持原有事件的 `occurred_at`

## 5. 实际示例场景

### 场景 1：连续观看电影

```
时间 10:15:00 - Screenshots 1-20
→ 分析：用户开始观看 "Inception"
→ 操作：insert(new_event) 
→ Event: "John is watching the movie 'Inception'", occurred_at: 10:15:00

时间 10:15:20 - Screenshots 21-40  
→ 分析：用户仍在观看同一部电影，只是进度更靠后
→ 操作：merge(event_id, combined_summary="John is watching the movie 'Inception' (continued)", 
             combined_details="...更多电影内容细节...")
→ 原事件被更新，包含更完整的观看内容

时间 10:16:00 - Screenshots 41-60
→ 分析：用户仍在观看同一部电影
→ 操作：merge(event_id, ...) 
→ 事件继续更新
```

### 场景 2：活动切换

```
时间 10:15:00 - Screenshots 1-20
→ 分析：用户正在观看 Netflix
→ 操作：insert(new_event)
→ Event: "John is watching Netflix", occurred_at: 10:15:00

时间 10:15:30 - Screenshots 21-40
→ 分析：用户关闭了 Netflix，打开了 VS Code，开始写 Python 代码
→ 操作：insert(new_event) 
→ Event: "John is working on Python project in VS Code", occurred_at: 10:15:30
```

### 场景 3：混合活动

```
时间 10:15:00 - Screenshots 1-20
→ 分析：用户主要在观看电影，但偶尔切换到其他窗口
→ 操作：insert(new_event)
→ Event: "John is watching movie 'Inception', occasionally checking other applications", 
         occurred_at: 10:15:00
```

## 6. 限制和注意事项

### 6.1 Details 长度限制
- 如果某个事件的 `details` 超过 5000 字符
- 应该使用 `episodic_memory_insert` 创建新事件，而不是继续 append

### 6.2 Merge 操作的覆盖行为
- `combined_summary` **完全覆盖**旧的 summary
- 因此必须包含旧信息和新信息
- **错误示例**：`"User continues watching movie"` （丢失了电影名称）
- **正确示例**：`"John is watching the movie 'Inception' (continued from previous session)"`

### 6.3 单次调用限制
- 每批 screenshots 只能执行**一个**操作（insert/merge/replace）
- 如果有多个不同的事件需要处理，会在后续批次中处理

### 6.4 Event ID 的使用
- Agent 必须使用 system prompt 中显示的**精确 event_id**
- 不能使用 chat history 中的 event_id，因为可能有变化

## 7. 总结

### Event 划分的特点：

1. **智能判断**：基于 LLM 的语义理解，而非硬编码规则
2. **上下文感知**：考虑最近事件和相关事件列表
3. **语义连续性**：更关注活动主题的连续性，而非严格的时间连续性
4. **单次处理**：每批 screenshots 最多影响一个 event
5. **三种操作**：insert（新事件）、merge（延续事件）、replace（整合事件）

### 连续性的判断：

- **有连续性**：通过 `merge` 操作，将新的 screenshots 信息合并到已有事件
- **无连续性**：通过 `insert` 操作，创建新的事件
- 连续性判断是**语义和上下文相关的**，不是严格的规则，而是基于 LLM 对用户活动的理解

这种设计使得系统能够灵活地处理各种用户活动模式，既能识别持续的活动，也能识别活动的中断和切换。


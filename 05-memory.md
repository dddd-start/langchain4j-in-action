# 第 5 章 ChatMemory 与自定义 ChatMemoryStore

## 概念

多轮对话的前提是"记住上文"。LangChain4j 把这套机制拆成四个部件，各司其职：

| 部件 | 职责 |
|---|---|
| `ChatMessage` | 一条消息（`UserMessage` / `AiMessage` / `ToolExecutionResultMessage` / `SystemMessage`） |
| `ChatMemory` | 决定"上下文里放哪些消息"的策略（如保留最近 N 条） |
| `ChatMemoryProvider` | 按 `memoryId` 造出对应的 `ChatMemory` 实例（实现会话隔离） |
| `ChatMemoryStore` | 消息的**持久化**（存的介质由你决定） |

`AiServices` 里这样装配：

```java
.chatMemoryProvider(memoryId -> MessageWindowChatMemory.builder()
        .id(memoryId)
        .maxMessages(20)
        .chatMemoryStore(dormChatMemoryStore)
        .build())
```

接口方法上的 `@MemoryId` 就是 `memoryId`，示例工程传的是 `conversationId` —— 于是不同用户、不同会话的上下文天然隔离。

**`InMemoryChatMemoryStore` 在生产环境有三个致命问题**：

1. **重启即丢**：用户刚聊到一半，服务重启后上下文全没了
2. **多实例不一致**：负载均衡把下一轮请求打到另一个实例，记忆是空的
3. **无法审计**：内存里的消息不留痕，"模型当时看到了什么"无法回溯

因此生产必须自己实现 `ChatMemoryStore`。

## 示例工程里的真实代码

`DormChatMemoryStore` 从 MySQL 的 `agent_message` 表还原对话历史：

```java
@Override
public List<ChatMessage> getMessages(Object memoryId) {
    Long conversationId = asConversationId(memoryId);
    if (conversationId == null) return new ArrayList<>();

    List<AgentMessage> rows = messageMapper.selectRecent(conversationId, 200);
    List<ChatMessage> messages = new ArrayList<>(rows.size());
    for (AgentMessage row : rows) {
        // ★ 这三行过滤是整个实现里最重要的部分
        if (row.getStatus() == MessageStatus.STREAMING
                || row.getStatus() == MessageStatus.BLOCKED) {
            continue;
        }
        ChatMessage message = toChatMessage(row);
        if (message != null) messages.add(message);
    }
    return messages;
}
```

### 规则一：`STREAMING` 状态的消息必须跳过

正在生成中的消息是**半成品**。如果把它喂回模型，模型会看到一句被截断的话，然后接着残缺内容续写 —— 表现为"回答突然跑偏"或"重复前半句"。这类问题很难复现，因为只在"用户上一轮还在生成时就发了下一条"的场景出现。

### 规则二：`BLOCKED` 状态的消息必须跳过（这是一条真实的注入绕过路径）

被内容护栏截断的消息里含有**违规内容**。如果它进入历史，就会出现：

```
第 1 轮：用户诱导 → 模型输出违规内容 → 护栏截断，标记 BLOCKED
第 2 轮：用户问别的问题 → 上下文里带着"被截断的违规内容"
```

模型看到了上一轮的违规内容，等于**通过"历史"绕过了输入护栏**。这是一个真实存在、且很容易被忽略的攻击面。

### 规则三：`TOOL` 角色消息必须带 `toolCallId`

```java
case TOOL -> {
    String content = row.getContent() == null ? "" : row.getContent();
    String toolCallId = row.getToolCallId() == null ? "unknown" : row.getToolCallId();
    String toolName = row.getToolName() == null ? "tool" : row.getToolName();
    yield ToolExecutionResultMessage.from(toolCallId, toolName, content);
}
```

OpenAI 协议要求工具结果消息必须通过 `tool_call_id` 关联到具体的工具调用，否则请求会被拒绝（400）。用数据库还原历史时，这个字段必须一起存、一起取。

### `AiMessage` 的思考过程也要回灌

```java
if (row.getReasoningContent() != null && !row.getReasoningContent().isBlank()) {
    builder.thinking(row.getReasoningContent());
}
```

推理模型在有历史思考时表现更连贯。注意这与"思考和答案分开存储"并不矛盾：**存储分开（便于前端分别展示与回放），回灌时合并**。

## 一个关键的设计决定：`updateMessages` 是空实现

```java
/**
 * 空实现 —— 消息的落库由 AgentTurnService 在流式生成的生命周期里完成。
 * 这里若再写一次会造成双份真相。
 */
@Override
public void updateMessages(Object memoryId, List<ChatMessage> messages) {
    // intentionally no-op
}
```

**为什么这样设计**：LangChain4j 的 `ChatMessage` 承载不了示例工程需要记录的全部元信息：

| 示例工程需要记录 | `ChatMessage` 有吗 |
|---|---|
| `seq_no`（会话内单调序号，保证不乱序） | ❌ |
| `turn_id`（幂等键） | ❌ |
| `tokens_in` / `tokens_out` | ❌ |
| `first_token_ms`（首字延迟） | ❌ |
| `status`（STREAMING / INTERRUPTED / BLOCKED） | ❌ |
| `trace_id` | ❌ |
| `reasoning_content` | 部分支持（`thinking`） |

如果 `updateMessages` 也写一份，就会出现**两份真相**：一份缺 seq、一份缺 token，出问题时无法判断以谁为准。

因此本实现是"**只读的数据库记忆视图**"：`getMessages` 负责读，写入由 `AgentTurnService` 统一负责（它在生成前写用户消息、生成后写助手消息，时机与流式状态严格对应）。

**这种取舍的代价与适用场景**：它要求 `ChatMemoryStore` 的使用方（示例工程的 `AgentTurnService`）承担全部写库责任。如果换成一个"只想快速接入、不想管落库"的项目，老老实实实现 `updateMessages` 更省事。**判断标准是：你是否需要 `ChatMessage` 之外的信息** —— 需要，就自己管写入；不需要，就用框架的约定。

### 一个附带的好处：数据库就是唯一真相

因为记忆完全来自数据库，所以：

- 多实例部署天然一致
- 重启不丢
- "模型当时看到了什么"可以直接查表
- 断线重连后上下文依旧完整（不依赖前端重传历史）

## 长期记忆与短期记忆的分层

| 层 | 存储 | 内容 | 生命周期 |
|---|---|---|---|
| 短期 | `agent_message`（窗口读取 20 条） | 最近对话原文（含工具调用与结果） | 当前会话 |
| 长期 | `agent_long_memory` | 画像（PROFILE）、稳定偏好（PREFERENCE）、会话摘要（SUMMARY） | 跨会话，可过期 |

`AgentMemoryService.renderLongTermMemory()` 把长期记忆渲染成一段自然语言拼进 System Prompt：

```
【关于当前用户的已知信息】
- 画像：国际业务一部员工，常驻东南亚
- 偏好：偏好单人间
- 偏好：不介意与同事合住
（这些是系统此前记录的信息，可直接使用，不必再向用户询问；如果与用户当前说法冲突，以用户当前说法为准。）
```

### 为什么长期记忆**不用**向量检索

常见做法是把长期记忆也做成向量库做语义召回。示例工程特意没那么做，理由是**数据特征不匹配**：

- 示例工程的长期记忆是**结构化偏好**（偏好房型、常去区域、是否介意合住），只有几个明确的 key
- 条数极少（每人 20 条以内）
- 这类数据用"精确 key 覆盖 + 重要度排序"比向量召回**更准、更快、更可控**

**向量召回在这里反而会出错**：`不介意合住` 和 `介意合住` 在语义空间里高度接近，向量检索很可能把它们混在一起 —— 而这种错误表现为"助手偶尔记错用户偏好"，极难排查。

**结论：技术选型要服务于数据特征，而不是反过来。** 当记忆变成大量非结构化的自由文本（例如"用户过去半年反馈过的所有问题"）时，向量检索才是正确选择。

### 长期记忆做成显式工具，而不是让模型后台偷偷写入

很多实现会让模型在后台自动提取并保存用户偏好。示例工程刻意做成显式工具：

```java
@Tool("""
        记住用户的一条稳定偏好，供以后所有对话使用。
        只有在用户明确表达"以后都""我一直""我习惯"这类长期偏好时才调用；
        一次性的需求（比如"这次我要单人间"）不要记录。
        """)
public String rememberPreference(@P("偏好键，如 PREFERRED_ROOM_TYPE") String key,
                                 @P("偏好内容，一句自然语言") String content) { ... }
```

理由有两个：

1. **合规**：企业系统里必须让用户知道"系统记住了我什么"。示例工程提供 `GET /api/agent/memories` 查看与 `DELETE /api/agent/memories` 清除（用户行使"被遗忘权"）
2. **准确性**：模型很容易把一次性需求误存成长期偏好（"这次我要单人间" → 永久偏好）。工具描述里写清"只有明确表达长期偏好才调用"，比让它自己判断更可控

## 最小可运行示例

```java
import dev.langchain4j.data.message.ChatMessage;
import dev.langchain4j.memory.chat.MessageWindowChatMemory;
import dev.langchain4j.store.memory.chat.ChatMemoryStore;
import dev.langchain4j.service.AiServices;
import dev.langchain4j.service.MemoryId;
import dev.langchain4j.service.UserMessage;
import java.util.*;

public class MemoryDemo {

    /** 最小可用的自定义 ChatMemoryStore：用内存 Map 演示接口语义。 */
    static class MapChatMemoryStore implements ChatMemoryStore {
        private final Map<Object, List<ChatMessage>> store = new HashMap<>();

        @Override
        public List<ChatMessage> getMessages(Object memoryId) {
            return new ArrayList<>(store.getOrDefault(memoryId, List.of()));
        }

        @Override
        public void updateMessages(Object memoryId, List<ChatMessage> messages) {
            store.put(memoryId, new ArrayList<>(messages));
        }

        @Override
        public void deleteMessages(Object memoryId) {
            store.remove(memoryId);
        }
    }

    interface Assistant {
        String chat(@MemoryId Long conversationId, @UserMessage String message);
    }

    public static void main(String[] args) {
        var store = new MapChatMemoryStore();
        Assistant assistant = AiServices.builder(Assistant.class)
                .chatModel(/* 见第 1 章 */ null)
                .chatMemoryProvider(memoryId -> MessageWindowChatMemory.builder()
                        .id(memoryId)
                        .maxMessages(10)
                        .chatMemoryStore(store)   // 换成数据库实现即可用于生产
                        .build())
                .build();

        System.out.println(assistant.chat(1L, "我叫张伟，常驻新加坡"));
        System.out.println(assistant.chat(1L, "我叫什么？"));   // 应能答出"张伟"
        System.out.println(assistant.chat(2L, "我叫什么？"));   // 另一个会话，应不知道
    }
}
```

## 常见坑

| 坑 | 现象 | 解法 |
|---|---|---|
| 用了 `InMemoryChatMemoryStore` | 重启丢记忆、多实例上下文不一致 | 自定义 `ChatMemoryStore` |
| 半成品消息进上下文 | 回答跑偏、重复前半句 | 跳过 `STREAMING` 状态 |
| 被拦截的违规内容进上下文 | 违规内容通过"历史"绕过护栏 | 跳过 `BLOCKED` 状态 |
| `TOOL` 消息没带 `toolCallId` | 请求被上游 400 拒绝 | 一起存、一起取 |
| 双写记忆与消息表 | 两份真相，排查时无法判断以谁为准 | 明确只在一处写入 |
| 长期记忆用向量召回 | 语义相近的相反偏好被混淆 | 结构化偏好用 key 精确匹配 |
| 让模型自动保存偏好 | 一次性需求被误存成长期偏好 | 做成显式工具 + 触发条件写清楚 |
| 窗口设得太大 | token 成本上升、超出上下文 | 窗口 + 摘要压缩（`agent_conversation.summary`） |

## 延伸验证

1. **实现并接入一个数据库 `ChatMemoryStore`**：参考 `DormChatMemoryStore`，自己写一个只存 `content` 与 `role` 的简化版，并验证同一个 `conversationId` 多轮对话能记住上文。
   *观察点*：重启应用后继续同一个会话，模型仍能回答"我叫什么"。

2. **验证隔离**：用两个不同的 `@MemoryId` 交叉提问"我叫什么"。
   *观察点*：两个会话互相不知道对方的信息，且数据库中各自有独立的 `seq_no` 序列。

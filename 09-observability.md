# 第 9 章 可观测性：审计、用量、成本与回放

## 概念

Agent 上线后，会遇到三类只能靠观测回答的问题：

1. **"这个回答为什么这么差？"** —— 模型当时看到了什么、调了什么工具、拿到了什么结果
2. **"这个月花了多少钱？花在谁身上？"** —— token 用量与成本归集
3. **"是不是被人越权试探了？"** —— 权限拒绝与攻击拦截的痕迹

这三类问题分别需要**回放**、**用量聚合**、**审计**。下面看怎么把它们做出来。

## 示例工程里的真实代码

### 用 `ToolProvider` 包装 `ToolExecutor` 做横切关注点

这是示例工程在工程上最值得学的一个设计。审计、去重、重试、熔断本来都是"每个工具都要做一遍"的事：

```java
public class GuardedToolProvider implements ToolProvider {

    @Override
    public ToolProviderResult provideTools(ToolProviderRequest request) {
        ToolProviderResult.Builder builder = ToolProviderResult.builder();
        for (Object toolObject : toolObjects) {
            for (Method method : toolObject.getClass().getMethods()) {
                if (!method.isAnnotationPresent(Tool.class)) continue;
                String name = ToolSpecifications.toolNameFrom(method);
                ToolSpecification spec = ToolSpecifications.toolSpecificationFrom(method);
                ToolExecutor executor = new DefaultToolExecutor(toolObject, method);   // 框架负责参数绑定
                builder.add(spec, new GuardedToolExecutor(name, executor));           // ★ 套上守卫
            }
        }
        return builder.build();
    }
}
```

**为什么不把审计逻辑写进每个 `@Tool` 方法**：散开的结果一定是"有人写了、有人忘了、有人写错顺序"。而这些问题在正常路径下**完全看不出来**，只在异常与并发时才暴露。集中到一个 `ToolExecutor` 包装层后，所有横切逻辑**只实现一次、对所有工具同时生效**，业务工具方法也保持干净（只有业务逻辑）。

### 同轮去重：为什么只对只读工具生效

模型在实际运行中经常重复调用同一个工具并传完全一样的参数，典型场景：

- 第一次调用返回空结果，模型"不甘心"再调一次
- 模型在同一轮里想"确认一下"，又调了一次相同的查询
- 部分模型对工具结果格式不敏感，误判为失败后重试

`GuardedToolProvider` 用 `(工具名 + 参数)` 的 SHA-256 指纹做同轮缓存：

```java
public String lookupToolResult(String toolName, String argumentsJson) {
    String key = RedisKeys.toolGuard(conversationId, turnId) + ":" + toolFingerprint(toolName, argumentsJson);
    return redis.opsForValue().get(key);
}
```

**只对只读工具生效**，因为写操作去重会把用户第二次真心想提交的申请**静默吞掉**。写操作的重复保护由业务层的幂等键负责（`stay_apply.idempotency_key` 唯一索引）。

**为什么按"轮"而不是按"会话"去重**：跨轮重复调用往往合理（用户几分钟后问"那间房现在还有吗"），不能一概拦截。

### 有限重试的边界

```java
int attempts = readOnly ? 2 : 1;       // ★ 写工具绝不重试
for (int attempt = 1; attempt <= attempts; attempt++) {
    try { result = action.get(); break; }
    catch (Exception e) {
        if (attempt < attempts && isTransient(e)) { sleepQuietly(120L * attempt); continue; }
        break;
    }
}
```

- **只对只读工具**重试：重试一个已经成功的写操作会造出重复业务单据
- **只对瞬时故障**重试：`isTransient()` 判断异常类名是否含 `sql` / `timeout` / `connect` / `socket` / `redis`；业务异常（`BizException`）绝不重试（它不会因为重试而变好）

### 按工具熔断

```java
if (failureCount >= BREAKER_THRESHOLD) {   // 连续失败 5 次
    breakerUntil.put(toolName, new AtomicLong(System.currentTimeMillis() + 60_000));
}
```

避免每次对话都卡在一个坏工具上 —— 例如某条 SQL 因为脏数据必然报错，没有熔断的话每个用户都要等一次超时。

### token 用量累加：为什么必须用 UPSERT

```java
@Insert("""
        INSERT INTO agent_usage_daily
          (id, stat_date, user_id, conversation_count, turn_count, prompt_tokens, ...)
        VALUES (#{id}, #{statDate}, #{userId}, ..., #{estCost})
        ON DUPLICATE KEY UPDATE
           turn_count        = turn_count + 1,
           prompt_tokens     = prompt_tokens + VALUES(prompt_tokens),
           completion_tokens = completion_tokens + VALUES(completion_tokens),
           total_tokens      = total_tokens + VALUES(total_tokens),
           ...
        """)
int accumulate(...);
```

**为什么不能用"先查后更新"**：同一用户并发两个流时，各自读到 100、各自加 50 写回 150，结果应该是 200 却变成 150。**用量是计费依据，不能丢。** 依赖 `uk_usage_date_user` 唯一索引保证只有一行，`ON DUPLICATE KEY UPDATE` 做原子累加。

### 一个附带的真实坑：必须用 `@Insert` 而不是 `@Update`

上面这条 SQL 看起来像 UPDATE，直觉上会写成 `@Update`。但 MyBatis-Plus 的**防全表更新拦截器**会把"命令类型为 UPDATE 且解析后没有 WHERE 条件"的语句直接拦掉，抛 `UnsupportedOperationException`：

```
### Cause: java.lang.UnsupportedOperationException
### The error may involve com.overseas.dorm.domain.mapper.AgentUsageDailyMapper.accumulate
```

`INSERT ... ON DUPLICATE KEY UPDATE` 恰好同时含有 `UPDATE` 关键字、又没有独立 `WHERE`，被误判成"危险的全表更新"。**声明成 `@Insert` 既符合语句的真实语义，也避开了误判** —— 而且错误信息里完全看不出拦截器的存在，只能靠经验。

### traceId 全链路贯穿

```
TraceIdFilter（最外层，Ordered.HIGHEST_PRECEDENCE）
  → 从 X-Trace-Id 取，没有则新生成
  → 写入 MDC（日志格式里自动带上）
  → 回写响应头（前端报错时可提供给客服）
  → 请求结束清理（线程池复用会串号）
```

落库位置：`audit_log`、`agent_message`、`agent_tool_call`、`rag_retrieval_log` 四张表都带 `trace_id` 并建有索引。`GET /api/audits/trace/{traceId}` 一次拉出整条链路。

**为什么必须显式跨线程传递**：流式回调与工具执行运行在模型客户端的线程池上，MDC/ThreadLocal 不会继承：

```java
AgentTurnContext.runAs(context, () ->
        TraceContext.runWith(context.traceId(), () ->
                UserContext.runAs(context.user(), body)));
```

### 跨线程上下文传递的完整方案

`AgentTurnContext` 在 ThreadLocal 之外额外维护一个**会话注册表**：

```java
private static final Map<Long, AgentTurnContext> REGISTRY = new ConcurrentHashMap<>();

public static AgentTurnContext begin(Long conversationId, Long messageId, String turnId,
                                     LoginUser user, String traceId) {
    AgentTurnContext context = new AgentTurnContext(...);
    HOLDER.set(context);
    if (conversationId != null) REGISTRY.put(conversationId, context);   // ★ 供工具线程取回
    return context;
}

public static AgentTurnContext byConversation(Long conversationId) {
    return conversationId == null ? null : REGISTRY.get(conversationId);
}
```

工具执行时用 `chatMemoryId`（= conversationId）取回：

```java
private String withTurnContext(Object memoryId, Supplier<String> action) {
    AgentTurnContext context = AgentTurnContext.byConversation(asLong(memoryId));
    if (context == null) { context = AgentTurnContext.current(); }
    if (context == null) { log.warn("工具 {} 未取到轮次上下文", toolName); return action.get(); }
    return AgentTurnContext.runAs(context, () -> TraceContext.runWith(context.traceId(),
            () -> com.overseas.dorm.common.security.UserContext.runAs(context.user(), action)));
}
```

**如果没有它**（示例工程在没有它的时候真实出现过）：工具审计里的 `conversation_id` 记成 0、`turn_id` 记成 null，同轮去重因为拿不到 turnId 而**彻底失效**，RAG 检索日志一条都不落。表现是"AI 偶尔查不到数据 + 审计缺失"，排查成本极高。

**为什么不用 `executeToolsConcurrently()`**：并发执行工具会让 ThreadLocal 上下文彻底失效，也会让"同轮去重"变得不可判定（两个并发调用可能在彼此写入缓存之前都通过了检查）。示例工程的工具都是几十毫秒级的查询，串行执行的延迟代价完全可以接受，换来的是行为可预测、可审计。

### 三张表各回答什么问题

| 表 | 回答的问题 |
|---|---|
| `agent_message` | 这轮用户问了什么、模型答了什么、用了多少 token、首字延迟多少、状态如何 |
| `agent_tool_call` | 模型调了哪个工具、传了什么参数、结果如何、**是否被权限拒绝**、**是否被去重**、耗时多少 |
| `agent_usage_daily` | 今天这个用户/整个平台消耗了多少 token、估算成本多少 |
| `agent_message_archive` | 冷热分离后的历史（审计链不能断，降冷不等于删除） |

### 会话回放

`AgentAuditService.replay(conversationId, messageId)` 把用户输入、思考过程、每次工具调用的参数与结果、最终答案、token 与耗时按时间顺序拼成一条时间线：

```java
Map<String, Object> node = new LinkedHashMap<>();
node.put("seq", message.getSeqNo());
node.put("role", message.getRole().name());
node.put("content", message.getContent());
node.put("thinking", message.getReasoningContent());   // 思考过程单独展示
node.put("tokensIn", message.getTokensIn());
node.put("firstTokenMs", message.getFirstTokenMs());
if (message.getTurnId() != null) {
    List<AgentToolCall> calls = toolCallMapper.selectByMessage(message.getId());
    // 每次工具调用：工具名、解析后的参数、是否成功、是否去重、是否通过权限、耗时
}
```

**"回答质量突然变差"时，第一件要做的事就是看回放**：模型当时到底看到了什么、调了什么工具、拿到了什么。

### 成本观测与预算守卫

```java
public BudgetDecision checkBudget(Long userId) {
    long used = auditService.dailyTokens(userId);
    if (used < budget) return BudgetDecision.allowed();
    boolean degrade = "DEGRADE".equalsIgnoreCase(properties.getRatelimit().getOverBudgetAction());
    if (degrade) return new BudgetDecision(true, false, "今日额度已用完，已切换到精简模式…");
    return new BudgetDecision(true, true, "今日额度已用完，明天会重置。");
}
```

**为什么默认 DEGRADE 而不是 REJECT**：把用户彻底挡在外面体验极差（"系统坏了？"），而降级到便宜模型继续服务，回答质量下降但功能可用，同时在界面上给出提示。这与"主模型故障降级"是同一套思路。

## 最小可运行示例

```java
import java.util.*;
import java.util.concurrent.*;

/** 用一个 ToolExecutor 包装器演示"横切关注点集中实现"。 */
public class ToolGuardDemo {

    interface ToolExecutor {
        String execute(String toolName, String args);
    }

    /** 真正的业务工具。 */
    static class RoomTool implements ToolExecutor {
        @Override public String execute(String toolName, String args) {
            return "{\"rooms\":[{\"roomNo\":\"301\"}]}";
        }
    }

    /** 记录审计的包装器。 */
    static class AuditedToolExecutor implements ToolExecutor {
        private final ToolExecutor delegate;
        private final Map<String, String> dedupCache = new ConcurrentHashMap<>();
        private final Set<String> readOnlyTools = Set.of("searchRooms");

        AuditedToolExecutor(ToolExecutor delegate) { this.delegate = delegate; }

        @Override public String execute(String toolName, String args) {
            String fingerprint = toolName + "|" + args;
            boolean readOnly = readOnlyTools.contains(toolName);
            long start = System.currentTimeMillis();

            if (readOnly) {
                String cached = dedupCache.get(fingerprint);
                if (cached != null) {
                    System.out.printf("[审计] tool=%s 命中同轮去重，直接返回缓存%n", toolName);
                    return cached;
                }
            }
            String result = delegate.execute(toolName, args);
            if (readOnly) dedupCache.put(fingerprint, result);
            System.out.printf("[审计] tool=%s 耗时=%dms 参数=%s%n",
                    toolName, System.currentTimeMillis() - start, args);
            return result;
        }
    }

    public static void main(String[] args) {
        ToolExecutor tool = new AuditedToolExecutor(new RoomTool());
        tool.execute("searchRooms", "{\"region\":\"SEA\"}");   // 真实执行
        tool.execute("searchRooms", "{\"region\":\"SEA\"}");   // 去重命中
        tool.execute("createApply", "{\"region\":\"SEA\"}");   // 写工具：不去重，每次都执行
        tool.execute("createApply", "{\"region\":\"SEA\"}");
    }
}
```

## 常见坑

| 坑 | 现象 | 解法 |
|---|---|---|
| 审计逻辑散在各工具里 | 有人忘写、顺序写错，正常路径看不出来 | 用 `ToolProvider` 包装做横切 |
| 对写工具也做去重 | 用户第二次提交被静默吞掉 | 只对只读工具去重 |
| 对写工具自动重试 | 造出重复业务单据 | 写工具 `attempts = 1` |
| 用量"先查后更新" | 并发下丢增量，计费不准 | `INSERT ... ON DUPLICATE KEY UPDATE` |
| 该 SQL 用 `@Update` 注解 | 被防全表更新拦截器拦掉（报错信息看不出原因） | 用 `@Insert` |
| 只靠 MDC/ThreadLocal 传 traceId | 回调与工具线程里丢失，日志串不上 | 显式 `runWith` 传递 |
| 工具执行时取不到上下文 | 审计记成 0、去重失效、RAG 不落日志 | 会话注册表 + `byConversation(memoryId)` |
| 并发执行工具 | ThreadLocal 失效、去重不可判定 | 保持串行（示例工程明确不用 `executeToolsConcurrently`） |
| 只记成功日志 | 无法发现越权试探 | 权限拒绝也写审计 |
| 超预算直接拒绝 | 用户以为系统坏了 | 默认降级到便宜模型 |

## 延伸验证

1. **实现工具调用统计**：用 `AgentToolCallMapper.statistics(days)` 的返回结果，找出失败率最高的工具与平均耗时最长的工具。
   *观察点*：能通过 `GET /api/agent/admin/usage?days=1` 拿到两张表并解释数据含义。

2. **验证 traceId 全链路**：发起一轮带工具调用的对话，从响应头拿到 `X-Trace-Id`，然后用 `GET /api/audits/trace/{traceId}` 拉出该次请求产生的全部审计记录。
   *观察点*：能同时看到 `AGENT_TURN` 与工具相关记录，且它们的 `trace_id` 完全一致。

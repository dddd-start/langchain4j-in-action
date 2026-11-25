# 第 4 章 流式输出（StreamingChatModel 与 TokenStream）

## 概念

流式输出的价值不是"显得快"，而是**让用户在等待期间持续得到反馈**。大模型生成一段回答可能要十几秒，同步等待会让用户怀疑系统卡死；而逐字输出让等待变得可接受 —— 实测体验差异巨大。

LangChain4j 的流式入口是 `StreamingChatModel`，在 `AiServices` 里对应返回 `TokenStream` 的接口方法。`TokenStream` 是一个**回调集合**，你把各个回调挂上去，最后调 `start()` 开始。

## 示例工程里的真实代码

`AgentTurnService.generate()` 是示例工程最复杂的单个方法，它把一次流式生成包装成了"可续传、可打断、可审核、可审计"的事件流。核心结构：

```java
CountDownLatch finishedLatch = new CountDownLatch(1);
TokenStream stream = agentRegistry.get(agentType, user.isAdmin())
        .chat(conversationId, message);

stream.onPartialThinking(partial -> inTurn(turnContext, () -> { /* 推送 REASONING */ }))
      .onPartialResponse(token -> inTurn(turnContext, () -> { /* 护栏 + 推送 DELTA */ }))
      .beforeToolExecution(before -> inTurn(turnContext, () -> { /* 推送 TOOL start */ }))
      .onToolExecuted(exec -> inTurn(turnContext, () -> { /* 推送 TOOL end */ }))
      .onRetrieved(contents -> inTurn(turnContext, () -> { /* 推送 REFERENCE */ }))
      .onCompleteResponse(response -> { try { /* 收尾 */ } finally { finishedLatch.countDown(); } })
      .onError(error -> { try { /* 收尾 */ } finally { finishedLatch.countDown(); } })
      .start();

// ★ start() 立刻返回，真正的生成还在异步跑 —— 必须等终态
boolean completed = finishedLatch.await(turnTimeoutSeconds, TimeUnit.SECONDS);
if (!completed) { /* 超时收口 */ }
```

### `TokenStream` 的全部回调

| 回调 | 触发时机 | 能拿到什么 |
|---|---|---|
| `onPartialResponse` | 每收到一段正文 | String（增量文本） |
| `onPartialThinking` | 每收到一段思考 | `PartialThinking.text()` |
| `onPartialToolCall` | 工具调用参数**流式分片**到达 | `PartialToolCall` |
| `onCompleteToolCall` | 某个工具调用的参数收齐 | `CompleteToolCall` |
| `beforeToolExecution` | 工具**即将执行**前 | `ToolExecutionRequest`（工具名 + 参数 JSON） |
| `onToolExecuted` | 工具执行完毕后 | `ToolExecution`（含 `hasFailed()`） |
| `onRetrieved` | RAG 检索到内容时 | `List<Content>` |
| `onIntermediateResponse` | 工具调用轮次的中间模型响应 | `ChatResponse` |
| `onCompleteResponse` | **整轮生成成功结束** | `ChatResponse`（含 `tokenUsage()`、`modelName()`） |
| `onError` | 整轮失败 | Throwable |
| `start()` | 触发执行（异步，立即返回） | — |

## 三个必须知道的坑（示例工程全部踩过）

### 坑 1：`start()` 是异步非阻塞的 —— 清理逻辑绝不能直接写在 finally 里

直觉写法是：

```java
try {
    stream.onCompleteResponse(...).onError(...).start();
} finally {
    AgentTurnContext.clear();        // 注销轮次上下文
    streamBuffer.decrementUserStreams(userId);   // 释放并发计数
    heartbeat.close();               // 关掉心跳
}
```

**这是错的，而且错得很隐蔽。** `start()` 只负责发起请求就立刻返回，真正的生成与回调在模型客户端的线程池里跑。于是 `finally` 会在**生成还没开始**的时候执行：

| 提前执行的动作 | 后果 |
|---|---|
| `AgentTurnContext.clear()` | 轮次上下文被注销 → 工具执行时取不到身份 → **工具在"未登录"状态下运行**（抛 401、审计里 `conversation_id` 记成 0、RAG 检索不落日志） |
| `decrementUserStreams` | 并发计数被错误释放 → 实际并发远超上限 |
| `heartbeat.close()` | 心跳立刻停止 → 长连接可能被网关掐断 |

示例工程最初就是这样写的，症状是日志里冒出 `BizException: 未登录或登录已过期` 且工具审计全部丢失，排查了很久才定位到"清理跑得太早"。

**正确做法**：用一个 `CountDownLatch` 把"这一轮真的结束了"这件事显式表达出来，在 `onCompleteResponse` / `onError` 里 `countDown()`，然后 `await(超时)`；清理放在 `await` 之后：

```java
boolean completed = finishedLatch.await(turnTimeout, TimeUnit.SECONDS);
if (!completed) {
    // 超时同样要收口：否则这条流会一直占着并发额度，前端也会永远转圈
    finishWithError(..., new BizException(AI_TIMEOUT, "生成超时，已中断本次回答"));
}
```

这个 `await` 顺带满足了另一个需求：**轮次总超时**。没有它就是"模型卡住时用户永远等下去，并发额度也永远不释放"。

### 坑 2：回调运行在模型客户端的线程池上，ThreadLocal 全部会丢

实测回调线程名是 `ForkJoinPool.commonPool-worker-N`，**不是**你发起调用的那个线程（示例工程是 `agent-service-stream-N`）。这意味着：

- `UserContext`（ThreadLocal）在回调里为空
- `TraceContext`（MDC/ThreadLocal）丢失 → 日志里的 traceId 张冠李戴
- `AgentTurnContext` 为空 → 工具审计拿不到 `turnId`，**同轮重复调用去重直接失效**

示例工程的解法是"**按会话注册 + 显式恢复**"：

```java
// AgentTurnContext 里的注册表：conversationId → 本轮上下文
private static final Map<Long, AgentTurnContext> REGISTRY = new ConcurrentHashMap<>();

// 回调体统一包一层
private void inTurn(AgentTurnContext context, Runnable body) {
    AgentTurnContext.runAs(context, () ->
            TraceContext.runWith(context.traceId(), () ->
                    UserContext.runAs(context.user(), body)));
}
```

工具的恢复点更巧妙：工具执行时框架会传入 `chatMemoryId`（就是 `@MemoryId` 的值，即 conversationId），`GuardedToolProvider` 用它从注册表取回上下文：

```java
private String withTurnContext(Object memoryId, Supplier<String> action) {
    AgentTurnContext context = AgentTurnContext.byConversation(asLong(memoryId));
    if (context == null) { context = AgentTurnContext.current(); }   // 兜底
    if (context == null) { log.warn("工具 {} 未取到轮次上下文", toolName); return action.get(); }
    return AgentTurnContext.runAs(context, () -> TraceContext.runWith(context.traceId(),
            () -> UserContext.runAs(context.user(), action)));
}
```

注意这里的 `log.warn` 是刻意保留的：**取不到上下文必须显式暴露**，不能静默降级 —— 否则表现为"AI 偶尔查不到数据"，排查成本极高。

### 坑 3：想拿思考过程，必须"模型支持 + 开启开关 + 用对回调"三者齐备

```java
// ① 建模型时开启（否则 reasoning_content 被静默丢弃）
returnThinking(Boolean.TRUE)

// ② 用 onPartialThinking 接收（不是 onPartialResponse）
.onPartialThinking(partial -> { String text = partial.text(); ... })
```

示例工程把思考过程 (`REASONING`) 与答案正文 (`DELTA`) 分成两种 SSE 事件，前端折叠展示。**不要把它们混在一条流里** —— 混了之后前端无法区分，而且思考内容会被写进答案、污染后续上下文。

## 一套完整的流式骨架

```java
import dev.langchain4j.service.AiServices;
import dev.langchain4j.service.MemoryId;
import dev.langchain4j.service.TokenStream;
import dev.langchain4j.service.UserMessage;
import java.util.concurrent.CountDownLatch;
import java.util.concurrent.TimeUnit;

public class StreamingDemo {

    interface Assistant {
        TokenStream chat(@MemoryId Long conversationId, @UserMessage String message);
    }

    public static void main(String[] args) throws Exception {
        Assistant assistant = AiServices.builder(Assistant.class)
                .streamingChatModel(/* 见第 1 章 */ null)
                .build();

        CountDownLatch done = new CountDownLatch(1);
        StringBuilder answer = new StringBuilder();

        assistant.chat(1L, "用三句话说明入住流程")
                .onPartialThinking(t -> System.out.print("[想]" + t.text()))
                .onPartialResponse(token -> {
                    answer.append(token);
                    System.out.print(token);
                })
                .onCompleteResponse(response -> {
                    System.out.printf("%n[完成] model=%s tokens=%d%n",
                            response.modelName(), response.tokenUsage().totalTokenCount());
                    done.countDown();
                })
                .onError(error -> {
                    error.printStackTrace();
                    done.countDown();
                })
                .start();   // 异步：立刻返回

        // 必须等终态，否则 main 会先退出；这也是"轮次超时"的实现位置
        if (!done.await(120, TimeUnit.SECONDS)) {
            System.out.println("[超时] 主动收口");
        }
        System.out.println("最终答案长度=" + answer.length());
    }
}
```

## 为什么示例工程用 `SseEmitter` 而不是 WebFlux

- 项目主体是 Spring MVC（MyBatis-Plus、拦截器、事务都是同步栈），引入 WebFlux 会造成两套编程模型混用
- `SseEmitter` 足够表达：`emitter.send(SseEmitter.event().id(seq).name(event).data(json))`，且支持心跳注释帧 `emitter.send(SseEmitter.event().comment("ping"))`
- `new SseEmitter(0L)` 表示**不设传输层超时**：长连接的关闭由业务终态（推送 `DONE`）或客户端决定，中间用 15 秒心跳保活。设一个固定超时反而会在长回答时被服务端自己掐断

## 常见坑

| 坑 | 现象 | 解法 |
|---|---|---|
| 清理写在 `finally` | 上下文提前注销、工具"未登录"、并发计数错乱 | `CountDownLatch` 等终态后再清理 |
| 依赖 ThreadLocal 传上下文 | 回调/工具里上下文为空 | 按会话注册 + 显式 `runAs` 恢复 |
| 没设轮次超时 | 模型卡住时用户永远等待、并发额度不释放 | `await(timeout)` + 超时收口 |
| 思考与正文混在一条流 | 前端无法区分，污染上下文 | 分 `REASONING` / `DELTA` 两种事件 |
| 在回调里做重活（同步写库、调外部服务） | 拖慢整条流的吞吐 | 只做入队/缓冲，重活异步化 |
| 每个 token 直接写数据库 | 写放大严重，DB 被打爆 | 写 Redis Stream 缓冲，结束时一次性落库 |

## 延伸验证

1. **复现并修复"清理过早"**：把 `generate()` 里的 `latch.await` 去掉，观察日志里是否出现工具未取到轮次上下文的告警、以及工具审计里 `conversation_id` 是否变成 0。
   *观察点*：能解释为什么 `finally` 会早于生成结束执行，并说明 `CountDownLatch` 解决了什么。

2. **给流式加一个"首字延迟"指标**：在 `onPartialResponse` 第一次触发时记录与请求开始的时间差，写入 `agent_message.first_token_ms`。
   *观察点*：`/api/agent/conversations/{id}/messages` 返回的 `firstTokenMs` 不为 0，且与肉眼观察的等待时间一致。

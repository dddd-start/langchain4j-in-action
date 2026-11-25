# 第 10 章 生产化改造清单

## 概念

前九章讲的是"怎么用 LangChain4j 把功能做出来"，这一章讲"怎么让它在上线后不出事"。这两件事的能力要求完全不同：功能正确靠的是对 API 的理解，而生产可用靠的是对**失败模式**的理解。

下面 10 个坑全部来自示例工程的真实经历 —— 每一个都先造成了可观测的故障，然后才被定位和修复。它们的共同特征是：**正常路径下完全看不出来，只在异常、并发或特定时序下暴露**。

---

## 坑 1：`TokenStream.start()` 是异步的，清理逻辑不能写在 `finally` 里

**现象**：日志里出现 `BizException: 未登录或登录已过期`；工具审计表的 `conversation_id` 全部是 0；RAG 检索日志一条都不落。

**根因**：`start()` 只负责发起请求就立即返回，真正的生成在模型客户端线程池里跑。于是下面这段代码的 `finally` 会在**生成还没开始**时就执行：

```java
try {
    stream.onCompleteResponse(...).onError(...).start();
} finally {
    AgentTurnContext.clear();                 // 轮次上下文被提前注销
    streamBuffer.decrementUserStreams(userId); // 并发计数被错误释放
    heartbeat.close();                         // 心跳立刻停止
}
```

轮次上下文一注销，工具执行时就拿不到身份 → `UserContext.require()` 抛 401 → 工具失败、审计丢失。

**正确做法**：用 `CountDownLatch` 显式表达"这一轮真的结束了"。

```java
CountDownLatch finished = new CountDownLatch(1);
stream.onCompleteResponse(r -> { try { finishSuccessfully(...); } finally { finished.countDown(); } })
      .onError(e -> { try { finishWithError(...); } finally { finished.countDown(); } })
      .start();

boolean completed = finished.await(turnTimeoutSeconds, TimeUnit.SECONDS);
if (!completed) { /* 超时收口：否则并发额度永远不释放，前端永远转圈 */ }
// 到这里再清理
```

**源码**：`agent-service/.../stream/AgentTurnService.java`

---

## 坑 2：流式回调运行在模型客户端的线程池上，ThreadLocal 全部会丢

**现象**：与坑 1 类似但更隐蔽 —— 上一轮对话的 traceId 出现在这一轮的日志里；同轮工具去重失效（因为拿不到 `turnId`）。

**根因**：实测回调线程名是 `ForkJoinPool.commonPool-worker-N`，**不是**发起调用的 `agent-service-stream-N`。`UserContext` / `TraceContext` / `AgentTurnContext` 三个 ThreadLocal 在回调里全部为空。

**正确做法**：**按会话注册上下文 + 显式恢复**。

```java
// 注册（发起时）
REGISTRY.put(conversationId, context);

// 回调体统一包一层
private void inTurn(AgentTurnContext ctx, Runnable body) {
    AgentTurnContext.runAs(ctx, () -> TraceContext.runWith(ctx.traceId(),
            () -> UserContext.runAs(ctx.user(), body)));
}

// 工具执行时用 chatMemoryId（= conversationId）取回
AgentTurnContext ctx = AgentTurnContext.byConversation(asLong(memoryId));
```

**注意**：取不到上下文时**必须打 WARN 日志**，不能静默降级 —— 否则表现为"AI 偶尔查不到数据"，排查成本极高。

**源码**：`agent-service/.../audit/AgentTurnContext.java`、`tool/GuardedToolProvider.java`

---

## 坑 3：模型降级必须包在"每一次真实调用"周围，而不是"选择阶段"

**现象**：主模型返回 429 时，用户直接看到"生成过程中出现异常"，而配置里的备用模型一直闲着。

**根因**：最初的实现是"选择阶段挑一个当前可用的模型，然后把它塞进 AiServices"。但 `ModelRegistry` 的熔断只在**选择时**生效；真正的调用失败后，没有任何机制去尝试下一个模型。**降级链成了纸面配置。**

**正确做法**：实现 `StreamingChatModel` / `ChatModel` 的包装器，在每次调用的周围做候选链尝试。

```java
public class FailoverStreamingChatModel implements StreamingChatModel {
    public void chat(ChatRequest request, StreamingChatResponseHandler handler) {
        attempt(request, handler, 0, null);
    }

    private void attempt(ChatRequest req, StreamingChatResponseHandler handler, int idx, Throwable last) {
        if (idx >= chain.size()) { handler.onError(last); return; }
        String modelName = chain.get(idx);
        AtomicBoolean anyOutput = new AtomicBoolean(false);
        StreamingChatResponseHandler wrapper = new StreamingChatResponseHandler() {
            @Override public void onPartialResponse(String partial) {
                anyOutput.set(true); handler.onPartialResponse(partial);
            }
            @Override public void onError(Throwable error) {
                registry.reportFailure(modelName, error);
                // ★ 铁律：已经输出过内容就不能再切换模型
                boolean canFailover = !anyOutput.get() && idx + 1 < chain.size();
                if (canFailover) attempt(req, handler, idx + 1, error);
                else handler.onError(error);
            }
            @Override public void onCompleteResponse(ChatResponse r) {
                registry.reportSuccess(modelName); handler.onCompleteResponse(r);
            }
        };
        factory.streaming(modelName).chat(req, wrapper);
    }
}
```

**流式场景下的一条铁律**：**只有在"尚未输出任何内容"时才允许切换模型**。已经吐出部分内容再换模型重来，用户会看到两段拼接的回答或重复的前半句 —— 那比明确失败更糟。

配套的熔断阈值也值得调优：示例工程用"连续失败 2 次熔断 30 秒"。免费模型上游本身就会间歇性 429，阈值设成 3~5 会让每次对话都先浪费一次必然失败的调用（多等好几秒）。

**源码**：`agent-service/.../model/FailoverStreamingChatModel.java`、`FailoverChatModel.java`、`ModelRegistry.java`

---

## 坑 4：流式输出必须"延迟一个窗口"再做安全审核

**现象**：违规内容已经显示在用户屏幕上，之后才被"审核拦截"。

**根因**：如果每个 token 都立即推送，那么窗口内**正在审核**的内容已经发出去了。命中违规时再截断，用户已经看到了。

**正确做法**：只推送"已经离开审核窗口"的部分，窗口内保留待审。

```java
int keep = Math.min(windowSize, pending.length());
int emitLength = pending.length() - keep;
String emitText = pending.substring(0, emitLength);   // ★ 只推已过审的部分
pending.delete(0, emitLength);
```

同时**窗口之间必须有重叠**（示例工程 16 字符），否则跨 token 边界的违规词（"制"和"毒"分属两个 token）会两边都查不到。

代价是首字延迟增加几十毫秒。**这个交换必须做** —— 否则审核是自欺欺人。

**源码**：`agent-service/.../guardrail/ContentGuardrail.java`

---

## 坑 5：线程池"大队列"会让实际并发永远停在 core 大小

**现象**：30 并发压测下，CPU 与下游数据库都很空闲，但首字延迟（TTFT）线性恶化到 P50 5.0 秒。

**根因**：`ThreadPoolTaskExecutor` 的扩容规则是「**先把队列填满，队列满了才把线程数从 core 扩到 max**」。最初的配置是 `core=8 / max=32 / queue=100`，于是 30 个并发请求里只有 8 个在跑，其余 22 个全在排队。

**正确做法**：延迟敏感型任务用**小队列 + 大 max**，让线程池在压力上来时先扩容。

```java
executor.setCorePoolSize(16);
executor.setMaxPoolSize(64);
executor.setQueueCapacity(16);          // ★ 小队列
executor.setAllowCoreThreadTimeOut(true); // 低峰期归还线程
executor.setRejectedExecutionHandler(new ThreadPoolExecutor.AbortPolicy()); // 排不下就快速失败
```

改成这套配置后：**P50 TTFT 从 5.0s 降到 2.5s，吞吐 +42%**。

对比另一类池：`auditExecutor` 用 `CallerRunsPolicy`（审计宁可拖慢调用方也不能丢），两种策略的选择依据是"这个任务能不能丢"。

**源码**：`agent-service/.../config/ExecutorConfig.java`

---

## 坑 6：单号生成不能只依赖 Redis 自增

**现象**：启动一个使用独立 Redis db 的实例后，**所有"创建会话"的请求立刻全部失败**，返回"数据已存在，请勿重复提交"。

**根因**：单号格式是 `前缀 + yyyyMMdd + 4 位序号`，序号来自 Redis `INCR`。新 Redis db 里计数器从 1 开始，拼出的 `AC202610140001` 与库里已存在的会话号撞上唯一索引。

**这个坑的危险性远超想象**：Redis 不是可靠存储 —— 重启未持久化、key 被 LRU 淘汰（示例工程配置了 `maxmemory-policy allkeys-lru`）、运维手工 flush、切库，都会让计数器归零。一旦归零，业务会出现"**突然什么都提交不了**"这种最严重级别的故障。

**正确做法（三层）**：

```java
// ① 播种：计数器不存在时，用数据库当天最大序号播种，而不是从 1 开始
String prefix = type + date;
stringRedisTemplate.opsForValue().setIfAbsent(key,
        String.valueOf(maxSequenceFromDb(type, prefix)), Duration.ofDays(2));

// ② 校验：拼出单号后先查库确认不存在，冲突则继续取下一个号
for (int attempt = 1; attempt <= MAX_ATTEMPT; attempt++) {
    long seq = nextSequence(key, type, date, attempt);
    String no = "%s%s%04d".formatted(type, date, seq);
    if (!exists(type, no)) return no;
}

// ③ 兜底：数据库唯一索引仍在，作为最后一道防线
```

**源码**：`agent-service/.../support/BizNoGenerator.java`、`mapper/BizNoMapper.java`

---

## 坑 7：`SELECT MAX(x) + 1` 取号在 REPEATABLE READ 下会死锁

**现象**：20~30 并发压测时部分请求返回 HTTP 500，日志里是 `Deadlock found when trying to get lock; try restarting transaction`。

**根因**：消息序号分配最初用 `SELECT COALESCE(MAX(seq_no),0) + 1` 再 INSERT。在 MySQL 默认的 REPEATABLE READ 隔离级别下，`SELECT MAX(...)` 会加**间隙锁**（gap lock），多个并发事务同时对同一张表做"取最大值 + 插入"时，间隙锁与插入意向锁互相等待，MySQL 直接判定死锁并回滚其中一个。

**正确做法**：改用 Redis 原子取号（不涉及数据库锁），首次用库内最大值播种；同时对并发异常做退避重试。

```java
private int nextSeqNo(Long conversationId) {
    String key = "dorm:conv:seq:" + conversationId;
    redis.opsForValue().setIfAbsent(key,
            String.valueOf(messageMapper.selectMaxSeqNo(conversationId)), Duration.ofDays(7));
    Long seq = redis.opsForValue().increment(key);
    return seq == null ? messageMapper.selectMaxSeqNo(conversationId) + 1 : seq.intValue();
}
```

```java
} catch (org.springframework.dao.ConcurrencyFailureException e) {
    // 死锁/锁等待超时属于瞬时冲突，退避后重试
    Thread.sleep(20L * attempt);
}
```

**正确性仍然由 `UNIQUE KEY (conversation_id, seq_no)` 兜底** —— Redis 只是"减少冲突"，不是"保证唯一"。

**源码**：`agent-service/.../stream/ConversationService.java`

---

## 坑 8：业务性拒绝不要抛成 HTTP 500

**现象**：并发超限时前端收到 HTTP 500 空响应，浏览器控制台只有 "HTTP Error 500"，用户完全不知道发生了什么（实际原因是"你已有 3 条对话正在进行中"）。

**根因**：SSE 接口（`produces = text/event-stream`）里抛出的业务异常被 `@RestControllerAdvice` 捕获，试图以 JSON 返回 —— 但响应的 Content-Type 已经是 `text/event-stream`，序列化 JSON 失败，最终变成一个没有 body 的 500。

**正确做法**：**SSE 接口的所有错误都走 SSE 帧**，保持单一契约。

```java
public SseEmitter chat(Long conversationId, String turnId, String message) {
    try {
        return doChat(conversationId, turnId, message);
    } catch (BizException e) {
        return errorEmitter(e.getResultCode(), e.getMessage());   // 返回 ERROR 事件帧
    }
}

private SseEmitter errorEmitter(ResultCode code, String message) {
    SseEmitter emitter = new SseEmitter(0L);
    emitter.send(SseEmitter.event().id("1").name("ERROR")
            .data(json(Map.of("code", code.getCode(), "message", message))));
    emitter.complete();
    return emitter;
}
```

**为什么这很重要**：前端只需要一套解析逻辑（连上 → 收到 ERROR 帧 → 展示 message），不需要再区分"HTTP 500 的 JSON 错误"与"SSE 流内的错误"两种形态。**双形态契约在实际联调中几乎一定会漏掉一种。**

**源码**：`agent-service/.../stream/AgentTurnService.java`（`errorEmitter`）

---

## 坑 9：压测必须用 mock 模型，而且要测流式专属指标

**现象**：用真实模型压测时，结果完全不可用 —— 两次运行的数字差异巨大，而且烧掉大量额度。

**三个理由**：

1. **成本**：100 并发 × 每轮几千 token，一次压测就烧掉可观额度
2. **结果不可比**：上游模型的排队与限流会主导延迟，测出来的是"上游的容量"，而不是"我们自己的 SSE 链路容量"
3. **不可重复**：上游抖动让同一份脚本两次跑出完全不同的数字

**正确做法**：`dorm.agent.mock.enabled=true` 时用本地模拟模型（固定分块、固定间隔出字），把**我们自己的链路**单独拎出来压：Redis Stream 写入、Pub/Sub 广播、SSE 推送、滑窗护栏、线程池、并发计数、数据库写入。

**要测的是流式专属指标，不是简单 QPS**：

| 指标 | 为什么重要 |
|---|---|
| **TTFT（首字延迟）** | 用户感知的"快不快"主要看它 |
| 完整轮次延迟 | 端到端体验 |
| 事件吞吐（事件/秒） | 链路的推送能力 |
| 字符吞吐（字/秒） | 实际信息交付速率 |
| 失败率与**错误类型** | 区分"按设计拒绝"与"真实故障" |

**实测数据**（30 并发 × 3 轮 × 16 账号，mock 模型）：

| 指标 | 数值 |
|---|---|
| 总轮次 / 成功 / 失败 | 90 / 87 / 3（3 次为按设计的每用户并发上限保护） |
| 总耗时 / 吞吐 | 15.9 s / 5.65 轮每秒 |
| TTFT 平均 / P50 / P95 | 3.07s / 2.47s / 6.51s |
| 事件吞吐 / 字符吞吐 | 120 事件/秒 / 1273 字/秒 |

**压测运行器的一个重要细节**：把 mock 实例跑在独立端口 + **独立 Redis db**。共用 Redis db 会让压测的并发计数直接算到真实用户头上（表现为压测结束后真实用户被"幽灵并发"挡住）。

**源码**：`agent-service/.../model/MockModelFactory.java`

---

## 坑 10：断点续传的顺序不能写反，护栏尾部必须推送

### 10a. 必须先订阅、再补历史

**现象**：重连后文字看着完整，但事件序号出现空洞。

**根因**：初版实现是"先读历史补发 → 再注册实时订阅"。这两步之间产生的事件，**既不在历史里**（读的时候还没写进去），**也收不到**（订阅还没注册），于是永久丢失。

**正确做法**：

```java
// ① 先订阅，补历史期间只往缓冲队列里放
subscription = streamBuffer.subscribe(messageId, event -> {
    if (!live.get()) { pendingBuffer.add(event); return; }
    forward(event);
});

// ② 补历史
for (var event : streamBuffer.readAfter(messageId, lastEventId, 1000)) forward(event);

// ③ 切实时，把缓冲队列里的事件按 seq 去重转发
live.set(true);
while ((event = pendingBuffer.poll()) != null) forward(event);
```

### 10b. 护栏扣留的尾部必须推送给前端

**现象**：数据库里的回答比用户屏幕上看到的**多 60 个字符**（正好是一个安全窗口）。

**根因**：内容安全护栏为了审核会扣留最后约 `windowSize` 个字符。`answer` 变量累加的是**原始 token**（已含被扣留的部分），而终态时又把 `guard.finish()` 放行的 tail 追加了一次 → 重复计数。

**正确做法**：tail **只推送、不回写 answer**。

```java
ContentGuardrail.GuardResult tail = guard.finish();
if (!tail.blocked() && !tail.emit().isEmpty()) {
    publish(emitter, messageId, sequence, StreamEvent.DELTA, Map.of("text", tail.emit()));
    // ★ 不要 answer.append(tail.emit()) —— answer 里已经有了
}
```

**这两个缺陷都是被判定性测试抓出来的**：一个续传判定脚本的标准是「中途断开 → 用 lastEventId 重连 → 两次 DELTA 文本拼接起来必须与数据库落库内容**完全相等**」，并校验序号无重复无空洞。"完全相等"这个条件同时排除了丢包（拼接更短）与重复（拼接更长）。

**源码**：`agent-service/.../stream/AgentTurnService.java`

---

## 顺手记下的两个小坑

**`estimatedAmount` 是普通方法不是 getter**：DTO 上写成 `public BigDecimal estimatedAmount()` 时，Jackson **不会序列化它**（只识别 `getXxx`/`isXxx`），前端拿到的字段直接消失，而且**不报任何错**。改成 `getEstimatedAmount()` 即可。

**运行期间不要覆盖 jar**：JVM 惰性加载类，如果运行中把 jar 重新打包覆盖，**尚未被加载**的类会抛 `NoClassDefFoundError`。症状极其迷惑：已经加载过的功能（健康检查）一切正常，没加载过的功能（静态资源、日志组件）静默失效或返回空响应。示例工程在开发中因此踩过一次。**`app.sh build` 应先停应用再构建。**

---

## 上线检查清单

- [ ] `dorm.agent.mock.enabled` 为 **false**（为 true 时助手会变成复读机）
- [ ] 所有密钥通过环境变量注入，不在代码/配置文件/版本库中；生产接 KMS 或 Vault
- [ ] `dorm.security.phone.aes-key` / `hmac-secret` 已设置且与数据加密时一致（轮换需有迁移方案）
- [ ] `dorm.security.jwt.secret` 长度 ≥ 32 字节
- [ ] Redis：开启 AOF 持久化；`maxmemory-policy` **不要用 `allkeys-lru`**（会淘汰限流/序号键，示例工程已用数据库播种兜底，但仍有额外开销）
- [ ] MySQL：连接池 `maximum-pool-size` 与数据库 `max_connections` 匹配；开启慢查询日志
- [ ] SSE 网关配置：关闭响应缓冲（Nginx `proxy_buffering off` + `X-Accel-Buffering: no`），`proxy_read_timeout` ≥ 轮次超时
- [ ] 并发上限与限流：`max-streams-per-user` / `max-streams-global` / `turns-per-minute` 按容量设定
- [ ] 模型降级链已配置且**经过验证真的会切换**（用错误的 API Key 试一次）
- [ ] 熔断阈值与半开时间符合上游的实际抖动特征
- [ ] 预算与成本：`daily-token-budget` 与超预算动作（DEGRADE / REJECT）已确认
- [ ] 知识库：版本与索引状态一致（`rag_document.version == indexed_version`），启动日志确认装载成功
- [ ] RAG 评测基线已录入，`minScore` 与 `topK` 有数据支撑
- [ ] 审计保留期与归档任务已配置（`agent_message` 冷热分离）
- [ ] 监控项：TTFT、完整轮次延迟、失败率、护栏拦截数、工具失败率、模型降级次数、RAG 无命中率
- [ ] 告警项：模型全部不可用、Redis/MySQL 不可用、Outbox 积压、审计写入失败
- [ ] 备份与回滚：数据库备份可用；镜像版本可回滚
- [ ] 压测基线：当前版本的 TTFT/吞吐/失败率已记录，迭代后可对比
- [ ] 日志级别确认（生产不要开 `dev.langchain4j` 的 DEBUG，会把完整 Prompt 打进日志，既占空间又有泄露风险）

---

## 延伸验证

1. **复现坑 5 并量化收益**：把 `agentStreamExecutor` 改回 `core=8 / max=32 / queue=100`，用 30 并发压一遍，记录 TTFT；再改回小队列压一遍对比。
   *观察点*：能给出两次的 TTFT P50/P95 与吞吐，并解释差异的成因。

2. **设计一个"降级真的生效"的验证**：故意把 `dorm.agent.provider.api-key` 配错，发起一轮对话，观察是否自动切换到备用模型、以及 `GET /api/agent/admin/models` 里熔断状态的变化。
   *观察点*：对话仍能成功（由备用模型应答），且终态 `DONE` 事件里的 `model` 字段与配置的主模型不同。

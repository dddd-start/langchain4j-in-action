# 第 1 章 ChatModel：对接任意 OpenAI 兼容端点

## 概念

LangChain4j 的起点是 `ChatModel`（同步）与 `StreamingChatModel`（流式）。它们是对「一次对话补全请求」的抽象，屏蔽了不同厂商的 HTTP 协议差异。

**为什么需要这层抽象**：模型供应商会换（成本、效果、合规随时可能变），而业务代码不应该跟着重写。LangChain4j 的做法是：只要对方提供 OpenAI 兼容端点，就能用 `OpenAiChatModel` 直接对接 —— 包括私有化部署的 vLLM、Ollama、一体机，以及示例工程使用的 Command Code。

**不这层抽象会怎样**：业务代码里散落着 `HttpClient` 调用、JSON 拼装、SSE 解析、重试逻辑，换一个供应商就要改几十处，而且无法利用框架的 `AiServices`、工具调用、记忆等能力。

## 示例工程里的真实代码

`agent-service/src/main/java/com/overseas/dorm/agent/config/LangChain4jConfig.java` 是整个 Agent 层的装配入口。它做三件事：造模型、造嵌入模型、提供模型工厂。

```java
private ChatModel buildChat(String modelName) {
    AgentProperties.Provider p = props.getProvider();
    return OpenAiChatModel.builder()
            .baseUrl(p.getBaseUrl())          // https://api.commandcode.ai/provider/v1
            .apiKey(p.getApiKey())            // 来自环境变量，绝不入库
            .modelName(modelName)
            .temperature(props.getModel().getTemperature())
            .maxTokens(props.getModel().getMaxTokens())
            .timeout(Duration.ofSeconds(p.getTimeoutSeconds()))
            .maxRetries(p.getMaxRetries())
            .logRequests(p.isLogRequests())
            .logResponses(p.isLogResponses())
            .returnThinking(Boolean.TRUE)     // ★ 关键
            .strictTools(Boolean.FALSE)       // ★ 兼容性
            .build();
}
```

### `returnThinking(true)`：为什么必须显式开启

推理模型（DeepSeek、Ling 等）除了 `content` 还会返回 `reasoning_content`（思考过程）。**不开这个开关，思考内容会被框架静默丢弃** —— 不报错、不警告，只是你拿不到。而思考内容有两个实际用途：

1. 前端可以展示「模型正在想什么」，显著改善等待体感（示例工程的 SSE 有独立的 `REASONING` 事件）
2. 思考过程回灌进上下文后，推理模型在后续轮次表现更连贯

### `strictTools(false)`：一个务实的兼容选择

`strictTools(true)` 会要求模型严格遵循 OpenAI 的工具调用 schema（`strict: true`）。但 Command Code 不是标准 OpenAI 实现，开启后可能被上游拒绝或行为异常。**对接非官方兼容端点时，先关掉这类"增强约束"，用最小公分母跑通，再按需开启。**

### 模型必须缓存（这是最容易出事的一点）

```java
private final Map<String, ChatModel> chatCache = new ConcurrentHashMap<>();
private final Map<String, StreamingChatModel> streamingCache = new ConcurrentHashMap<>();

public ChatModel chat(String modelName) {
    return chatCache.computeIfAbsent(modelName, this::buildChat);
}
```

`OpenAiChatModel` 内部持有 HTTP 连接池。如果每次请求都 `builder().build()` 一个新对象，会持续创建连接池与线程资源，在并发下迅速耗尽内存与文件描述符 —— 而且症状是"跑了几天后突然连不上模型"，很难联想到是模型没缓存。

**注意同步与流式用的是两张独立的缓存表。** 它们在协议上是不同的客户端对象，混用会出现"同步调用拿到了流式模型"这类隐蔽错误（编译期发现不了，只在特定路径运行时暴露）。

### 为什么手写 `@Bean` 而不用 langchain4j 的 Spring Boot Starter

Starter 会通过约定自动装配一批 bean，省事但把「模型是怎么创建出来的、参数从哪来、什么时候生效」全藏进了自动配置里。示例工程刻意手写，收益是：

- 启动日志里能看到每个模型的实际参数（示例工程会打印 `[LangChain4j] 主对话模型 = xxx @ https://...`）
- 所有可调参数集中在 `AgentProperties`（`@ConfigurationProperties("dorm.agent")`），改配置不用改代码
- 学习时不会被"魔法"绕晕

代价是要自己维护装配代码。对教学项目和需要精细控制的生产系统，这个交换是值得的。

### 本地嵌入模型也在这一层装配

```java
@Bean
public EmbeddingModel embeddingModel() {
    return new BgeSmallZhV15QuantizedEmbeddingModel();
}
```

BGE-small-zh-v1.5 是 512 维的中文优化嵌入模型，**模型文件直接内嵌在 jar 里**（约 20MB），不需要首次运行时下载，也不需要外部服务。这在离线环境与合规敏感场景下很关键 —— 知识库文档不出网。

## 最小可运行示例

```java
import dev.langchain4j.model.chat.ChatModel;
import dev.langchain4j.model.chat.StreamingChatModel;
import dev.langchain4j.model.openai.OpenAiChatModel;
import dev.langchain4j.model.openai.OpenAiStreamingChatModel;
import java.time.Duration;

public class MinimalChatModelDemo {
    public static void main(String[] args) {
        // 同步
        ChatModel model = OpenAiChatModel.builder()
                .baseUrl(System.getenv("DORM_AI_BASE_URL"))
                .apiKey(System.getenv("DORM_AI_API_KEY"))
                .modelName("poolside/laguna-s-2.1-free")
                .temperature(0.2)
                .maxTokens(512)
                .timeout(Duration.ofSeconds(60))
                .maxRetries(1)
                .returnThinking(true)
                .strictTools(false)
                .build();

        System.out.println(model.chat("用一句话说明什么是宿舍入住申请"));

        // 流式
        StreamingChatModel streaming = OpenAiStreamingChatModel.builder()
                .baseUrl(System.getenv("DORM_AI_BASE_URL"))
                .apiKey(System.getenv("DORM_AI_API_KEY"))
                .modelName("poolside/laguna-s-2.1-free")
                .timeout(Duration.ofSeconds(120))
                .returnThinking(true)
                .accumulateToolCallId(true)   // 流式工具调用时累积分片的 tool_call id
                .build();

        streaming.chat("数一下 1 到 5",
                new dev.langchain4j.model.chat.response.StreamingChatResponseHandler() {
                    @Override
                    public void onPartialResponse(String partial) {
                        System.out.print(partial);   // 逐字输出
                    }

                    @Override
                    public void onCompleteResponse(
                            dev.langchain4j.model.chat.response.ChatResponse response) {
                        System.out.println("\n[完成] tokens="
                                + response.tokenUsage().totalTokenCount());
                    }

                    @Override
                    public void onError(Throwable error) {
                        error.printStackTrace();
                    }
                });
    }
}
```

## 常见坑

| 坑 | 现象 | 解法 |
|---|---|---|
| 每次请求都新建模型 | 跑一段时间后连接耗尽、报连接错误 | 按模型名缓存（`ModelFactory`） |
| 同步/流式共用一张缓存 | "同步调用拿到流式模型"的隐蔽错误 | 两张独立缓存表 |
| 忘记 `returnThinking(true)` | 拿不到思考过程，且**没有任何报错** | 建模型时显式开启 |
| `strictTools(true)` 对接非标准端点 | 请求被上游拒绝或行为异常 | 先用 `false` 跑通 |
| 把 apiKey 写进代码或配置文件 | 密钥泄露 | 只从环境变量读（示例工程 `config/.env`，已 gitignore） |
| 忘了 `maxRetries` | 上游偶发 429 时直接失败 | 设 1～2 次；更完整的降级见第 10 章 |
| `timeout` 设得太短 | 长回答被截断或超时 | 同步 60s、流式 120s 起；流式还要配轮次超时 |

## 延伸验证

1. **对接第二个端点**：把 `dorm.agent.provider.base-url` 指向一个本地 Ollama（`http://localhost:11434/v1`），模型名改成 `qwen2.5:7b`，不加改任何 Java 代码让助手跑起来。
   *观察点*：`/api/agent/chat/stream` 能返回内容，且启动日志里打印出新的 baseUrl 与模型名。

2. **验证缓存生效**：在 `ModelFactory.chat()` 里加一行日志，连续发起 5 次对话，确认同一个模型名只构建了一次。
   *观察点*：5 次对话产生的"构建模型"日志恰好 1 条。

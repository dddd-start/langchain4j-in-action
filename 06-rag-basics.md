# 第 6 章 Embedding、EmbeddingStore 与 RAG 基础

## 概念

RAG（Retrieval-Augmented Generation，检索增强生成）解决的是「模型不知道你的私有知识」这个问题。它的链路是：

```
文档 → 切分 → 向量化 → 存入向量库
用户问题 → 向量化 → 相似度检索 → 取回相关片段 → 拼进 Prompt → 模型基于片段回答
```

LangChain4j 对应四个抽象：`EmbeddingModel`（文本→向量）、`TextSegment` / `Metadata`（切片与元数据）、`EmbeddingStore`（向量存储与检索）、`EmbeddingSearchRequest` / `EmbeddingMatch`（检索请求与命中）。

**不用 RAG 会怎样**：要么把整份制度文件塞进 Prompt（token 成本爆炸、超出上下文窗口），要么让模型凭记忆回答（制度类问题必然编造）。

## 示例工程里的真实代码

### 为什么用本地 ONNX 嵌入模型

```java
@Bean
public EmbeddingModel embeddingModel() {
    return new BgeSmallZhV15QuantizedEmbeddingModel();   // 512 维，模型内嵌 jar
}
```

示例工程对接的模型服务商**没有 `/embeddings` 端点**（实测返回 404）。而本地嵌入反而有三个实际好处：

1. **文档不出网**：合规上更干净（知识库是公司内部制度）
2. **没有按次计费**：重建索引随便做
3. **延迟可控**：毫秒级，且不占用模型配额

BGE-small-zh-v1.5 是中文优化的 512 维模型，量化版约 20MB，**模型文件直接内嵌在 jar 里** —— 不需要首次运行时下载（这在离线环境里非常关键）。

### 分块：为什么按标题切分而不是按固定长度硬切

`MarkdownChunker` 按 `#` / `##` / `###` 切分，标题路径用 ` / ` 连接（如 `入住管理规定 / 申请流程`），并对超长小节退化到段落、句子边界。

**硬切最省事，但会制造幻觉。** 看这个例子：

```
切片 A：……员工需在入住日次日 23:59 前
切片 B：办理入住，否则申请自动失效……
```

检索时如果只召回切片 A，模型看到的是一句**没说完的话**，它要么答不出来，要么——更糟——**编造后半句**。这是 RAG 幻觉最常见的来源，而且很难归因到"分块策略"上。

制度文档天然有层级结构，按标题切分能让每个切片都是语义完整的单元。只有当单个小节本身超长时才退化切分，且切分点优先选段落边界（空行），其次句号，最后才硬切。

### 重叠窗口：为什么必须有

段落边界仍可能切断跨段落的因果表述。保留 50–60 字符重叠，可以让关键句子至少在一个切片里是完整的：

```java
if (buffer.length() + p.length() + 1 > chunkSize && buffer.length() > 0) {
    pieces.add(buffer.toString().strip());
    String tail = tail(buffer.toString(), overlap);   // ★ 保留尾部重叠
    buffer.setLength(0);
    buffer.append(tail);
}
```

代价是少量冗余 token，换来的是"关键句不被切断"。这个交换是值得的。

### 向量存储：float32 二进制而不是 JSON

`VectorCodec` 把 `float[]` 与字节数组互转：

```java
public static byte[] encode(float[] vector) {
    ByteBuffer buffer = ByteBuffer.allocate(vector.length * Float.BYTES)
            .order(ByteOrder.LITTLE_ENDIAN);          // ★ 统一小端
    for (float v : vector) buffer.putFloat(v);
    return buffer.array();
}
```

**为什么用二进制**：512 维向量用 JSON 数组存约 6–8KB，用 float32 二进制只有 2048 字节（差 3 倍以上）。更重要的是检索时的开销：二进制可以直接 memcpy 进 `float[]`，JSON 需要逐字符解析。知识库几千个切片时，这是"毫秒级"与"几十毫秒级"的差别。

**为什么必须统一小端字节序**：如果编码用大端、解码用小端（或跨机器迁移时不一致），得到的不再是报错，而是一堆**"看起来正常但完全错误"的浮点数** —— 检索结果会莫名其妙地全错，且**没有任何异常提示**。测试里有一条专门的断言：

```java
@Test
void littleEndianLayout() {
    byte[] encoded = VectorCodec.encode(new float[]{1.0f});
    // IEEE-754 单精度 1.0f = 0x3F800000，小端存放为 00 00 80 3F
    assertThat(encoded).containsExactly(0x00, 0x00, (byte) 0x80, 0x3F);
}
```

### 自己实现 `EmbeddingStore<TextSegment>`

`MysqlEmbeddingStore` 实现了框架接口，但检索引擎是**内存索引**：

```java
public EmbeddingSearchResult<TextSegment> search(EmbeddingSearchRequest request) {
    lock.readLock().lock();                     // ★ 检索用读锁，可并发
    try {
        float[] query = request.queryEmbedding().vector();
        for (IndexEntry entry : index) {
            if (!passesFilter(entry, request.filter())) continue;   // 元数据过滤
            double score = VectorCodec.cosine(query, entry.vector());
            if (score < request.minScore()) continue;
            matches.add(new EmbeddingMatch<>(..., entry.segment()));
        }
        matches.sort(...reversed());
        return new EmbeddingSearchResult<>(matches.subList(0, maxResults));
    } finally {
        lock.readLock().unlock();
    }
}
```

**为什么检索走内存而不是每次查库**：这是刻意的取舍，值得写清楚——

| 方案 | 每次检索的开销 |
|---|---|
| 查库暴力检索 | 回传几千行 MEDIUMBLOB（几十 MB），即使有连接池，带宽与反序列化也是数十毫秒级 |
| 内存索引 | 启动时一次性加载常驻，检索是纯 CPU 点积，千级切片 1ms 内完成 |

代价是内存占用（1000 切片 × 512 维 × 4B ≈ 2MB，完全可接受）与"写后需要刷新索引"。刷新由 `reload()` 完成，知识库重建时调用。**读写锁**的选择也有讲究：检索高频并发、重建索引低频，用 `ReentrantReadWriteLock` 让检索之间能并发，只在重建时短暂写锁。

**规模增长后的演进路径**：切片数上万后，把 `search()` 换成"先按元数据过滤缩小候选 → 再 HNSW/IVF 索引"，或直接迁到专用向量库（Milvus / Qdrant / pgvector / Redis Search）。**上层 `RagService` 与工具体系完全不需要改动** —— 这是面向接口编程在这里的真正价值。

### `Filter` 表达式树必须递归求值

LangChain4j 的 `Filter` 是由 `And` / `Or` / `Not` / `IsEqualTo` / `IsIn` / `ContainsString` 组成的表达式树：

```java
private boolean passesFilter(IndexEntry entry, Filter filter) {
    if (filter == null) return true;
    if (filter instanceof And and)   return passesFilter(entry, and.left()) && passesFilter(entry, and.right());
    if (filter instanceof Or or)     return passesFilter(entry, or.left()) || passesFilter(entry, or.right());
    if (filter instanceof Not not)   return !passesFilter(entry, not.expression());
    if (filter instanceof IsEqualTo eq)   return matches(entry, eq.key(), eq.comparisonValue());
    if (filter instanceof IsIn in)        return in.comparisonValues().stream().anyMatch(v -> matches(entry, in.key(), v));
    ...
}
```

**为什么不能用 `filter.toString()` 做字符串匹配**：这看起来能跑，但只要条件的**值**里出现 `regionId` 这样的字面量（比如某个文档标题含这个词），过滤结果就会莫名其妙地错乱，而且极难排查。**用字符串处理结构化对象是常见的偷懒，代价是难以定位的错误。**

### 一条业务规则：全局文档对任何区域都可见

```java
private boolean matches(IndexEntry entry, String key, Object expected) {
    Object actual = metadataValue(entry, key);
    // regionId 为空的切片表示"全局文档"（如集团统一的入住流程规定）
    if ("regionId".equals(key) && actual == null) return true;
    ...
}
```

否则员工在新加坡问"入住流程是什么"时，会查不到集团级的统一规定。

### 为什么知识用 Markdown 放版本库

`dorm-service/src/main/resources/knowledge/*.md`，每份文档带 front-matter：

```markdown
---
docCode: POLICY-APPLY-FLOW
title: 入住申请流程与审批时限规定
category: POLICY
regionCode: GLOBAL
version: 1
effectiveDate: 2026-01-01
source: 集团行政管理制度汇编（2026版）
---

# 入住申请流程
...
```

好处有三个：

1. **可评审**：制度变更走代码评审流程，谁改了什么一目了然
2. **可回滚**：知识库出错可以 `git revert`，后台富文本编辑做不到
3. **可复现**：任何环境用同一份知识，不会出现"测试环境查得到、生产环境查不到"

`rag_document` 表只是这份真相的**索引与快照**。

### 版本比对：只在内容变化时重建索引

```java
} else if (existing.getIndexedVersion() == null || existing.getIndexedVersion() < parsed.version()) {
    // 版本变了才重建
}
```

向量化是整个链路里最贵的一步。每次启动都全量重建会让启动时间白白增加几十秒。用 `version` 与 `indexed_version` 比较来决定是否重建 —— 改了内容就把 front-matter 里的 `version` 加 1。

### 为什么用 `ApplicationReadyEvent` 而不是 `@PostConstruct`

```java
@EventListener(ApplicationReadyEvent.class)
public void loadOnStartup() { ... }
```

`@PostConstruct` 在 bean 初始化阶段执行，此时数据库连接池、事务管理器可能还没完全就绪；而且知识库重建是耗时操作，放在 `@PostConstruct` 会拖慢整个应用启动，甚至让健康检查探针超时。

另外它**失败时不能让应用起不来**：知识库装载异常只记日志，业务功能必须照常可用（助手退化为无知识库模式）。

## 最小可运行示例

```java
import dev.langchain4j.data.document.Metadata;
import dev.langchain4j.data.embedding.Embedding;
import dev.langchain4j.data.segment.TextSegment;
import dev.langchain4j.model.embedding.EmbeddingModel;
import dev.langchain4j.model.embedding.onnx.bgesmallzhv15q.BgeSmallZhV15QuantizedEmbeddingModel;
import dev.langchain4j.store.embedding.*;
import java.util.Map;

public class RagBasicsDemo {
    public static void main(String[] args) {
        EmbeddingModel model = new BgeSmallZhV15QuantizedEmbeddingModel();

        // 1) 向量化并存入内存向量库（生产替换为自己的 EmbeddingStore 实现）
        EmbeddingStore<TextSegment> store = new InMemoryEmbeddingStore<>();
        String[] chunks = {
                "审批通过后须在入住日次日 23:59 前办理入住，否则申请自动失效并释放床位。",
                "退房需在退房日 12:00 前完成；提前退房需填写原因。",
                "宿舍房间每天 100 至 200 新币，费用由公司承担。"
        };
        for (String chunk : chunks) {
            TextSegment segment = TextSegment.from(chunk,
                    Metadata.from(Map.of("category", "POLICY")));
            Embedding embedding = model.embed(chunk).content();
            store.add(embedding, segment);
        }

        // 2) 检索
        String query = "审批通过了但我没去住，会怎么样？";
        Embedding queryEmbedding = model.embed(query).content();
        EmbeddingSearchResult<TextSegment> result = store.search(
                EmbeddingSearchRequest.builder()
                        .queryEmbedding(queryEmbedding)
                        .maxResults(2)
                        .minScore(0.5)
                        .build());

        for (EmbeddingMatch<TextSegment> match : result.matches()) {
            System.out.printf("score=%.3f  %s%n", match.score(), match.embedded().text());
        }
        // 3) 把命中的片段拼进 Prompt 交给模型（见第 7 章）
    }
}
```

## 常见坑

| 坑 | 现象 | 解法 |
|---|---|---|
| 按固定长度硬切 | 关键句被切断 → 模型编造后半句 | 按标题/段落/句子边界切 + 重叠 |
| 向量存 JSON | 体积 3 倍、检索慢 | float32 二进制（MEDIUMBLOB） |
| 字节序不统一 | 检索结果全错且**无任何报错** | 显式统一小端 + 写断言 |
| 过滤条件用 `toString()` 匹配 | 条件值含关键字时结果错乱 | 递归求值表达式树 |
| 每次启动全量重建索引 | 启动慢几十秒 | 用 `version` / `indexed_version` 比对 |
| 知识库装载失败导致应用起不来 | 全站不可用 | 只记日志，业务功能照常 |
| 忘记处理"全局文档" | 区域检索查不到集团制度 | `regionId` 为空的切片对任何区域可见 |
| 检索返回结果完全不过滤元数据 | 查到别的区域的规定 | 区域/分类过滤在向量比较前执行 |

## 延伸验证

1. **写一个自己的 `EmbeddingStore`**：用 `HashMap` 存向量，实现 `add(embedding, segment)` 与 `search(request)`，支持 `maxResults` 与 `minScore`。
   *观察点*：能对 3 条中文文本做检索，相似句排在首位。

2. **对比分块策略**：把 `MarkdownChunker.chunk(markdown, 200, 0)` 的重叠设为 0，检索"入住日次日 23:59 前该做什么"，观察召回片段是否被切断。
   *观察点*：能观察到重叠为 0 时出现被截断的片段，并解释为什么会诱发幻觉。

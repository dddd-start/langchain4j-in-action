# 第 7 章 RAG 进阶：召回质量与调优

## 概念

RAG 的效果由两段决定：**召回**（有没有找到正确资料）与**生成**（有没有正确使用资料）。大部分线上问题出在召回，而召回恰恰是最容易被含糊过去的一环 —— 很多人只会说"效果还行"，却拿不出一个数字。

这一章解决的就是"如何用数字说话"。

## 召回率到底怎么测

### 五个指标，各自诊断什么

| 指标 | 定义 | 能诊断什么 |
|---|---|---|
| **Recall@k** | Top-k 结果里包含正确文档的题目占比 | 检索能力总体水位 |
| **MRR** | 首个正确结果排名的倒数均值（排第 1 得 1.0，第 3 得 0.333） | **排序质量**：Recall 高但 MRR 低 = 正确答案总在末尾 |
| **关键词命中率** | Top-k 片段中出现了预期关键信息的比例 | "召回对了文档但拿到的是无关段落"——比完全没召回更常见 |
| **No-Hit 率** | 一条都没召回的比例 | **知识库缺口**，最该优先修的问题 |
| **平均最高相似度** | Top-1 分数均值 | 用于校准 `minScore` 阈值 |

### 为什么金标集要标注"应该由哪份文档回答"

一个常见的错误做法是：人工写一份"标准答案"，然后用答案文本相似度来评测。**这会把评测变成措辞比较** —— 同一个意思换个说法就得低分，评测结果失去指导意义。

正确做法是标注**客观事实**：这个问题应该由哪份制度文件回答。它的优点：

- 标注成本低（只需写一个 docCode，不用写整段答案）
- 歧义小（制度文件是唯一的，不会"两个答案都对"）
- 可复核（换个人看也能判断标注对不对）

示例工程的金标集格式（`agent-service/src/main/resources/rag-eval/golden-set.jsonl`）：

```json
{"id":"q01","question":"员工提交入住申请后，区域管理员多久之内必须处理？","docCode":"POLICY-APPLY-FLOW","keyword":"24","category":"流程时限"}
{"id":"q13","question":"我想提前走，提前退房怎么申请？","docCode":"RULE-CHECKIN-CHECKOUT","keyword":"提前退房","category":"退房规范"}
```

40 道题，覆盖 10 份文档与 4 个区域，`keyword` 用于补充检查"是否拿到了有实质内容的段落"。

### 为什么 Recall@k 必须和 No-Hit 率一起看

这是最容易踩的坑：

> 如果把 `minScore` 定得过高（比如 0.8），大量问题会直接 No-Hit。
> 但 **Recall@k 依然很高** —— 因为评测时不设阈值，它衡量的是"排进前 k 的能力"。
> 于是监控上一切正常，而用户体验是"什么都答不上来"。

因此示例工程的评测**刻意不设 minScore**（测"能不能排进前 k"），把阈值的影响单独用 No-Hit 率来观察。两个指标一起看，才能同时发现"检索能力不足"与"阈值过严"两类问题。

## 示例工程里的真实代码

### MMR：最大边际相关性去冗余

纯按相似度排序有个典型问题：如果知识库里一段话被拆成了 3 个相邻切片，这 3 个会同时占据 Top3，而真正补充信息的第 4 段被挤出。模型拿到 3 个几乎一样的切片，答案自然单薄。

`RagService.mmr()` 在排序时同时考虑"与问题的相关性"和"与已选结果的差异度"：

```java
double mmrScore = lambda * relevance - (1 - lambda) * redundancy;
```

`redundancy` 是"与已选片段的最大相似度"。λ 越大越偏重相关性，越小越偏重多样性，示例工程默认 0.7。

**为什么用向量算冗余而不是关键词重叠**：

```
片段 A：入住需在次日 23:59 前办理
片段 B：超过次日 23:59 未办理将失效
```

关键词几乎不重叠，但语义高度重复 —— 用关键词算冗余会认为它们不重复，MMR 就失效了。用向量才能正确识别。

### 缓存：键必须包含所有影响结果的维度

```java
String cacheKey = RedisKeys.retrievalCache(CryptoUtil.sha256(
        "%s|%s|%d|%.2f|%d".formatted(query, regionId, k, score, embeddingStore.version())));
```

这里每一个维度都不能少：

- 少了 `regionId` → 换了区域还拿到上一个区域的结果
- 少了 `topK` / `minScore` → 改了配置但结果没变
- **少了 `embeddingStore.version()`** → 文档更新了还在用旧答案（最隐蔽的一个）

版本号是索引重载时递增的计数器，因此"知识库一变，所有旧缓存自然失效"，不需要逐个清理。知识库重建时另外主动清理一次检索与问答缓存。

**两条缓存纪律**：

1. 缓存键必须包含影响结果的所有维度
2. **缓存不能成为正确性依赖**：未命中时都能回源拿到正确结果；读缓存异常只记 debug 日志并继续走实时检索

### 生产侧持续追踪：`rag_retrieval_log`

每次检索都落一条记录：

| 字段 | 用途 |
|---|---|
| `query_text` / `top_k` | 谁问了什么、用的什么参数 |
| `hit_count` / `top_score` / `scores` | 召回质量 |
| `hit_chunk_ids` | 命中了哪些切片（可回溯到具体段落） |
| `latency_ms` / `cache_hit` | 性能与缓存效果 |
| `answer_used` | 答案是否引用了检索结果 |
| `feedback` | 用户点赞/点踩 |

这张表能回答三个运营问题：

1. **有多少问题一条都没命中** —— 知识库缺口，直接给出"该补哪些内容"的清单
2. **命中但用户点踩** —— 召回对了但答案组织得不好（这是生成侧问题，不是检索侧）
3. **相似问题的分数分布** —— 用于校准 `minScore`

`RagService.qualityMetrics(days)` 一次算出：

```java
SELECT COUNT(*) AS total,
       SUM(CASE WHEN hit_count = 0 THEN 1 ELSE 0 END) AS no_hit,
       SUM(CASE WHEN cache_hit = 1 THEN 1 ELSE 0 END) AS cache_hits,
       ROUND(AVG(top_score), 4) AS avg_top_score,
       ROUND(AVG(latency_ms)) AS avg_latency_ms,
       SUM(CASE WHEN feedback = 'DISLIKE' THEN 1 ELSE 0 END) AS dislikes,
       SUM(CASE WHEN feedback = 'LIKE' THEN 1 ELSE 0 END) AS likes
  FROM rag_retrieval_log
 WHERE create_time >= DATE_SUB(NOW(), INTERVAL #{days} DAY)
```

配套的 `topMissedQueries(days, limit)` 直接输出"一条都没查到"的问题排行 —— **这是知识库建设最直接的输入**。

### RAG 的工程闭环

```
召回（是否找到）→ 生成（是否用对）→ 反馈（用户是否满意）
   ↑                                        │
   └──────── 缺口清单 ← 无命中率 ←────────────┘
```

引用来源（SSE 的 `REFERENCE` 事件 + 前端展示）是**抑制幻觉最有效的手段之一**：当模型必须说出"依据是哪一条"时，它编造条款的概率显著下降。同时用户也能自己核对，形成人工校验回路。

## 示例工程的真实评测结果

```
题目数 40 / 索引切片 110 / 嵌入模型 bge-small-zh-v1.5（512 维，本地 ONNX）
检索方式：余弦相似度 + MMR 重排（λ=0.7），评测不设相似度阈值
```

| 指标 | 数值 |
|---|---|
| Recall@1 | 77.5% |
| Recall@3 | **100%** |
| Recall@5 | **100%** |
| MRR | 0.883 |
| 关键词命中率 | 97.5% |
| No-Hit 率 | 0% |
| 平均最高相似度 | 0.848 |
| 平均检索耗时 | 10.5 ms |

### 这些数字说明什么

- **Recall@3 = 100%** 是最关键的一条：只要正确文档进入前 3，模型就能看到它。因此**当前召回不是瓶颈**
- **Recall@1 = 77.5%** 说明约 1/4 的问题里正确文档排在第 2–3 位。这对最终答案影响有限（模型能看到 Top5），但如果将来把 `topK` 降到 1（为省 token），效果会立刻下降 —— 这就是"为什么不建议为了省钱把 topK 定成 1"的数据依据
- **MRR = 0.883** 与 Recall@1 互相印证，排序质量整体良好
- **关键词命中率 97.5%** 说明召回的不仅是"对的文档"，还是"有实质内容的段落"（有 1 题命中了文档但没命中关键信息，属于可接受范围）
- **No-Hit = 0%** 说明当前 `minScore = 0.55` 的阈值对本知识库是合适的；如果调高到 0.8，这个数字会立刻上升
- **10.5 ms** 的检索延迟说明内存索引 + 缓存的设计达标（不需要上专用向量库）

**什么样的召回水平才算可用**：如果这是生成侧的关键依据，Recall@5 低于 90% 就说明知识库覆盖或切分策略有问题，应先修召回再调模型。示例工程的 100% 是"知识库规模小（10 篇文档）+ 问题与文档强相关"共同决定的 —— **换到几千篇文档的真实企业知识库时，Recall@5 通常会在 85%–95% 区间，这才是需要持续运营的水平**。

## 最小可运行示例

```java
import dev.langchain4j.data.embedding.Embedding;
import dev.langchain4j.model.embedding.EmbeddingModel;
import java.util.*;

/** 一个最小的召回率评测器：Recall@k + MRR。 */
public class RecallEvaluationDemo {

    record Golden(String question, String expectedDocCode) {}

    record Hit(String docCode, double score) {}

    public static void main(String[] args) {
        List<Golden> goldenSet = List.of(
                new Golden("审批通过后多久必须入住", "POLICY-APPLY-FLOW"),
                new Golden("提前退房要填什么", "RULE-CHECKIN-CHECKOUT"),
                new Golden("德国宿舍垃圾分类", "REGION-GUIDE-EU"));

        int topK = 3;
        int hitAtK = 0;
        double mrrSum = 0;

        for (Golden item : goldenSet) {
            List<Hit> hits = retrieve(item.question(), topK);   // 检索实现
            int rank = 0;
            for (int i = 0; i < hits.size(); i++) {
                if (item.expectedDocCode().equals(hits.get(i).docCode())) { rank = i + 1; break; }
            }
            if (rank > 0 && rank <= topK) hitAtK++;
            if (rank > 0) mrrSum += 1.0 / rank;
            System.out.printf("%-24s 期望=%s 排名=%s%n",
                    item.question(), item.expectedDocCode(), rank == 0 ? "未命中" : rank);
        }

        System.out.printf("Recall@%d = %.1f%%   MRR = %.3f%n",
                topK, hitAtK * 100.0 / goldenSet.size(), mrrSum / goldenSet.size());
    }

    private static List<Hit> retrieve(String query, int topK) {
        return List.of();   // TODO: 接入 EmbeddingStore
    }
}
```

## 常见坑

| 坑 | 现象 | 解法 |
|---|---|---|
| 只测 Recall，不看 No-Hit | 指标好看但用户什么都问不出来 | 两个指标一起看 |
| 用"标准答案文本相似度"做评测 | 换个说法就低分，评测失去指导意义 | 标注"应由哪份文档回答" |
| 检索缓存键漏维度 | 文档更新后还在用旧答案 | 键里加索引版本号 |
| 为了省 token 把 topK 设为 1 | 效果断崖式下降（Recall@1 只有 77.5%） | 至少 topK=3~5，配合 MMR 去冗余 |
| 用关键词重叠算 MMR 冗余 | 语义重复但用词不同的片段被判为不重复 | 用向量算冗余 |
| 检索日志只记成功 | 无法发现知识库缺口 | 记录**全部**检索，尤其零命中 |
| docCode 不进切片元数据 | 无法判断"命中了哪份文档" | 索引时把 docCode/docTitle 写进 metadata |

## 延伸验证

1. **跑一次完整评测**：调用 `POST /api/agent/admin/rag-eval?topK=5`，把返回的 Markdown 报告保存下来，并从中找出 Recall 最低的题目。
   *观察点*：能说出 Recall@1/3/5 与 MRR，并指出"未召回题目"清单里有几题。

2. **验证 minScore 的影响**：把 `dorm.agent.rag.min-score` 从 0.55 调到 0.8，重启后再次运行评测，观察 No-Hit 率的变化。
   *观察点*：能观察到 No-Hit 率明显上升，而 Recall@k 几乎不变 —— 并能解释这与"指标好看但产品坏了"的关系。

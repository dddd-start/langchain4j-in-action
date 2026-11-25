# LangChain4j 生产化实践：以「海外宿舍管理系统」为样例

> 这不是 API 手册，而是一份**面向生产**的 LangChain4j 实践记录。
> 每一章都对应示例工程里真实运行的代码 —— 文中每一段示例，都是项目里跑通过的。

> 配套的开源工程：[`dorm-platform`](https://github.com/dddd-start/dorm-platform) ——
> 三服务拆分（业务 / AI Agent / 前端）+ 138 个零依赖单测 + 12 项冒烟 + 27 项全流程回归，
> 文中引用的类名与方法名都能在那个仓库里找到对应源码。
> 同作者的另一个 RAG 工程：[`knowledgeflow-agent`](https://github.com/dddd-start/knowledgeflow-agent)（Python，混合检索 + 意图路由 + 可复现评测）。

## 为什么要看生产代码而不是 Hello World

绝大多数 LangChain4j 入门材料停在 "Hello World" 级别：调一次模型、打印一句话。但真实的 Agent 系统要复杂得多，会遇到一堆入门材料里不会提的问题：

- 模型客户端回调**跑在别的线程**上，ThreadLocal 上下文全丢了怎么办？
- `TokenStream.start()` 竟然是**异步**的，那"这一轮什么时候结束"该怎么判断？
- 流式输出时用户断网，重连后怎么保证**不重不漏**？
- 用户问"帮我审批一下"，你的助手会去调审批工具吗？**怎么保证它调不了**？
- 模型限流了，备用模型为什么一直闲着？

示例工程把这些问题的**真实解法**都写进了代码与注释。这份笔记就是沿着这些真实问题展开的。

## 阅读方式

- 每章结构统一：`概念` → `示例工程中的真实代码` → `最小可运行示例` → `常见坑` → `延伸验证`
- 建议**边读边打开对应的源码文件**，文中引用的类名、方法名与开源工程完全一致
- 有环境的话，`bash scripts/start.sh` 把系统跑起来，延伸验证可以直接对照

## 术语约定

本文只使用 LangChain4j 与业界标准术语，几个高频词先定义清楚：

| 术语 | 含义 |
|---|---|
| **示例工程** | 指 [`dorm-platform`](https://github.com/dddd-start/dorm-platform)，文中的代码与数字都出自它 |
| **一轮（turn）** | 一次完整的「用户提问 → 模型/工具处理 → 回答收口」，对应 SSE 的 `event:DONE` |
| **续传** | 流式输出中途断线后，客户端带 `lastEventId` 重连，服务端从该序号之后补齐，做到不重不漏 |
| **滑窗审核** | 流式输出时保留最后 N 个字符不立即推送，先做内容审核，命中即停 |

## 目录

| 章 | 主题 | 核心问题 |
|---|---|---|
| [01](01-chatmodel.md) | ChatModel 对接任意 OpenAI 兼容端点 | 模型怎么建、为什么必须缓存、私有化怎么切 |
| [02](02-prompt-structured-output.md) | Prompt 与结构化输出 | 动态提示词、Prompt Cache 友好结构、容错式解析 |
| [03](03-aiservices-and-tools.md) | AiServices 与 @Tool 工具调用 | 声明式接口、工具描述写法、防死循环、身份安全 |
| [04](04-streaming.md) | 流式输出 | `TokenStream` 全部回调、异步 `start()` 的三个真实坑 |
| [05](05-memory.md) | ChatMemory 与自定义 ChatMemoryStore | 哪些消息不能进上下文、长期记忆为什么不用向量 |
| [06](06-rag-basics.md) | Embedding、EmbeddingStore 与 RAG 基础 | 切分策略、向量二进制存储、自研 EmbeddingStore |
| [07](07-rag-advanced.md) | RAG 进阶：召回质量与调优 | **召回率到底怎么测**、MMR、缓存键设计 |
| [08](08-guardrails.md) | Guardrails 与提示词注入防御 | 五层防御、分类评分、流式滑窗审核 |
| [09](09-observability.md) | 可观测性 | 横切关注点、用量 UPSERT、跨线程 traceId、回放 |
| [10](10-production-checklist.md) | 生产化改造清单 | **10 个真实踩过的坑** + 上线检查清单 |

## 阅读顺序建议

1. **先跑起来**：`bash scripts/env-install.sh && bash scripts/env-start.sh && bash scripts/start.sh`，用 `superadmin` / `Dorm@123456` 登录 `http://127.0.0.1:8080`
2. **读第 1–3 章**，把"模型 → 提示词 → 工具"这条主线打通，这是 LangChain4j 的核心
3. **读第 4–5 章**，理解流式与记忆 —— 这两块是把 demo 变成产品必须跨过的坎
4. **读第 6–7 章**，RAG 是效果差异最大的一环，也是最容易糊弄过去的一环
5. **读第 8–9 章**，安全与可观测性决定了这套系统能不能上线
6. **最后读第 10 章**，它把这些零散的知识点收口成一份"上线前必查清单"
7. **跑一遍测试**：`mvn test`（138 个零依赖单测）、`bash scripts/smoke.sh`（12 项冒烟）、`bash scripts/regression.sh`（27 项全流程回归）

## 版本说明

内容与以下版本严格对应：

- LangChain4j **1.22.0**
- Spring Boot **3.5.16**
- Java **21**
- 模型 Provider：Command Code（OpenAI 兼容 `/provider/v1`）

LangChain4j 1.x 相比 0.x 有较大 API 变化（`ChatLanguageModel` → `ChatModel`、`generate` → `chat` 等），如果你在网上看到 0.x 的示例，注意区分。

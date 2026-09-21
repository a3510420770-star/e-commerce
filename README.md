> **当前分支 `ops`。** 这份源码按章节分了分支：`git checkout chat` … `git checkout ops`
> 是各章当时的代码，`git checkout main` 回到最终完整版本。
>
> 各章分支只保留到该章为止的文件，代码本身取自最终版本。最终代码的引用关系是按完整项目
> 写的，所以早期分支里会有指向后续章节模块的 import，
> **分支仅供对照阅读，不保证能直接运行，能跑的以 `main` 分支为准**。

# Pxx — 电商智能客服 Agent

《AI Agent 智能客服实战》的配套源码。一个能查订单物流、答政策 FAQ、走退款子流程、
挖知识补库、还能微调一个主题分类器的完整客服系统。

代码是随课程一章章长出来的，不分支：chat 一个纯对话接口起步，到 topic 收尾时是下面这套东西。

## 技术栈

FastAPI + LangGraph / LangChain + SQLAlchemy / MySQL + Milvus。

聊天、嵌入、重排三组上游各自直连，没有网关那一层。模型名和地址都在 `.env` 里配
(`CHAT_*` / `EMBED_*` / `RERANK_*` 三组)，换供应商、换模型不用改代码。

## 跑起来

```bash
cp .env.example .env      # 填 CHAT_* / EMBED_* / RERANK_* 三组
docker compose up -d      # MySQL
make seed                 # 灌业务测试数据
make dev                  # 依赖容器 + MCP :8101/:8102 + 应用 :8000
```

浏览器打开 <http://localhost:8000> 就是聊天页。

**详细的安装、配置、建知识库、常见问题，看飞书那篇「Pxx 项目源码下载」**，
这里只留一条能把服务拉起来的最短路径。两边写岔了以那篇为准。

## 代码怎么组织

| 位置 | 装的是什么 |
| - | - |
| `app/api/` | HTTP 入口。聊天、Agent、知识库录入、复核、验收页、成本看板 |
| `app/graph/` | LangGraph 那张图。`state` 状态、`nodes` 节点、`routing` 分流规则、`build` 组装 |
| `app/core/` | 单点能力。上游客户端、检索、重排、意图、指代、摘要、置信度、飞轮、可观测 |
| `app/kb/` | 知识怎么进库。切块、嵌入、双写 MySQL 与 Milvus、去重、从对话里挖问答对 |
| `app/tools/` | 工具系统。内置 `@tool`、MCP 客户端、注册表、统一执行引擎 |
| `app/db/` | 表模型与仓储 |
| `app/static/` | 前端页面。聊天、知识库录入、飞轮待审、观测与成本、主题分布、分类器验收 |
| `mcp_servers/` | 两台业务 MCP Server，物流和售后各一台，独立进程 |
| `sql/` | 各章的建表与迁移，容器首启按文件名顺序自动执行 |
| `scripts/` | 建库、评估、微调这些离线活 |
| `docs/superpowers/` | 各章的 spec 和 plan。课程实战篇教的就是这套流程，留着当范本 |
| `primer/` | 前置篇两篇的配套例子（大模型是什么、Agent 怎么动起来），各自独立，跟项目其他代码没有依赖关系，`.env` 用同一份 |

## 各章长出了什么，怎么验

`make test` 跑全部单测，不打真实模型。下面这些要真服务在跑。

| 章 | 这一章长出来的东西 | 验收 |
| - | - | - |
| chat | 流式对话、结构化提取 | `make eval` |
| orders | 五个 `@tool` 业务工具，单轮 Function Calling | `make eval-agent` |
| kb | 切块、嵌入、MySQL 与 Milvus 双写、对话挖知识 | `make kb-build` `make kb-vectorize` `make eval-retrieval` |
| rag | 混合检索、RRF、重排、Query 改写、四策略评估 | `make smoke-rag` `make eval-rag` |
| agent | LangGraph workflow 骨架 + 主力 Agent 的 ReAct 环 | `make eval-agent` |
| intent | 分流器、指代消解、退款子流程的 interrupt/resume | `make smoke-interrupt` `make eval-intent` |
| context | 上下文管理。滑窗、摘要、前缀缓存 | `make eval-context` |
| tools | 工具系统。MCP 动态发现、统一执行引擎、审计日志 | `make eval-tools` |
| ops | Langfuse 自部署、数据飞轮、成本账 | `make langfuse-up` `make flywheel` `make eval-flywheel` `make cost-report` |
| topic | 主题分类器。语料、微调、阈值扫描、ONNX 推理服务 | `make topic-corpus` `make topic-train` `make topic-eval` |

每条命令的前置条件（哪些服务得先起、哪张表得先建）写在飞书那篇文档的「各章验收」段里。
`make help` 也能看到带说明的完整目标清单。

## 端口

| 端口 | 是什么 |
| - | - |
| 8000 | 应用 |
| 8101 / 8102 | 业务 MCP Server，物流 / 售后 |
| 8110 | topic 主题分类器推理服务（`make classifier-up` 之后） |
| 3000 | Langfuse（`make langfuse-up` 之后） |
| 19530 | Milvus |

应用那几个页面：`/` 聊天、`/kb` 知识库录入、`/review` 飞轮待审、`/observability` 观测与成本、
`/topics` 主题分布、`/acceptance` 分类器验收。

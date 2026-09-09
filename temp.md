# MewHelp 对外接口清单

说明：本清单依据 app/main.py 中挂载的全部路由与各 api 模块的装饰器整理，按「页面路由 / JSON API」两大类、并按业务模块分组。想核对完整实时路由可打开 http://localhost:8000/docs （FastAPI 自带 OpenAPI 文档）。

## 一、页面路由（浏览器直接访问，返回 HTML）

| 路径 | 用途 | 定义处 |
|---|---|---|
| `/` | 聊天主页面 | app/main.py |
| `/admin` | 后台管理总首页（知识库、RAG 评估、飞轮待审、观测成本、主题分布、验收各一张卡） | app/main.py |
| `/kb` | 知识库录入页（贴文档切块入库、向量化、当场检索自测） | app/main.py |
| `/rag-eval` | RAG 评估页（四策略对照报告，页上看、页上重跑） | app/main.py |
| `/review` | 飞轮待审队列后台页 | app/main.py |
| `/observability` | 观测与成本页（意图成本账、评估趋势、置信度阈值校准） | app/main.py |
| `/topics` | 主题分布页（17 类问题量） | app/main.py |
| `/topics/questions` | 类目问题列表页（按类分页看问题） | app/main.py |
| `/acceptance` | 验收总览页（九项实证闸门卡） | app/main.py |
| `/acceptance/eval` | 评测详情页（P/R/F1、阈值扫描、混淆矩阵、单句试分类） | app/main.py |
| `/acceptance/data` | 数据产物页（语料血缘、考卷、泄漏自检、ONNX 盘点） | app/main.py |
| `/acceptance/errors` | 错例复核页 | app/main.py |
| `/static/*` | 静态资源（样式/脚本） | app/main.py mount |

## 二、JSON API

### 1. 客服对话（核心链路）

| 方法 | 路径 | 作用 | 文件 |
|---|---|---|---|
| POST | /api/chat | Agent 主入口，SSE 流式返回（完整图编排，含意图识别、历史上下文、RAG、工具调用、citations 事件） | app/api/chat.py |
| POST | /api/agent | 同一条编排链路但一次性 JSON 返回，验收 / demo 用（DEPLOY.md 第 7 步打的就是它），响应含 tool_calls 数组与 answer | app/api/agent.py |
| GET | /api/conversations?user_id= | 按用户列会话 | app/api/conversations.py |
| GET | /api/conversations/{conversation_id}/messages | 取某会话的消息列表 | app/api/conversations.py |
| POST | /api/extract | 售后描述原文 → 结构化工单（AfterSalesTicket：order_id / request_type / expected_solution），走 EXTRACT_PROMPT + with_structured_output | app/api/extract.py |
| POST | /api/feedback | 回答满意度反馈（👍 / 👎），ch04 起的 RAG 质量闭环数据源 | app/api/feedback.py |

### 2. 前端动作（表单 / 工单确认流）

| 方法 | 路径 | 作用 | 文件 |
|---|---|---|---|
| POST | /api/actions/create-ticket | 建工单表单落地（与 Agent 的 create_ticket 工具共用动作层） | app/api/actions.py |
| POST | /api/actions/create-refund | 退款表单落地 | app/api/actions.py |
| POST | /api/actions/resume | LangGraph interrupt 的 resume 入口（建工单确认卡 确认/取消 回填） | app/api/actions.py |

### 3. 知识库管理（前缀 /api/kb）

| 方法 | 路径 | 作用 | 文件 |
|---|---|---|---|
| GET | /api/kb/overview | 知识库总览（chunk 数、向量数、各 content_type 分布） | app/api/kb.py |
| POST | /api/kb/preview | 切块预览（不落库，先看切块结果） | app/api/kb.py |
| POST | /api/kb/ingest | 文档入库（切块写 MySQL） | app/api/kb.py |
| POST | /api/kb/vectorize | 向量化写 Milvus（调上游 embedding 接口） | app/api/kb.py |
| POST | /api/kb/search | 检索自测（当场验证召回效果） | app/api/kb.py |
| GET | /api/kb/staging | 挖矿抽取暂存列表（extracted/kept/...） | app/api/kb.py |
| POST | /api/kb/staging/approve | 暂存 QA 人工采纳入库 | app/api/kb.py |
| POST | /api/kb/staging/reject | 暂存 QA 拒绝 | app/api/kb.py |

### 4. 评估与观测

| 方法 | 路径 | 作用 | 文件 |
|---|---|---|---|
| GET | /api/acceptance/overview | ch10 分类器验收总览 | app/api/acceptance.py |
| GET | /api/acceptance/eval | 评测详情（P/R/F1、micro vs macro、阈值扫描、混淆矩阵） | app/api/acceptance.py |
| GET | /api/acceptance/data | 数据产物盘点 | app/api/acceptance.py |
| GET | /api/acceptance/errors | 错例清单 | app/api/acceptance.py |
| GET | /api/acceptance/service | 服务链路状态 | app/api/acceptance.py |
| POST | /api/acceptance/classify | 单句试分类 | app/api/acceptance.py |
| GET | /api/rag-eval/overview | ch04 RAG 四策略评估总览（端出 data/ch04/reports/rag_eval.json） | app/api/rageval.py |
| GET | /api/rag-eval/faith-cases | 忠实度（faithfulness）案例列表 | app/api/rageval.py |
| POST | /api/rag-eval/faith-cases/{case_id}/status | 案例审核状态流转 | app/api/rageval.py |
| GET | /api/observability/overview | 观测成本总览（Langfuse 意图成本账、评估趋势） | app/api/observability.py |
| GET | /api/admin/overview | 后台总览汇总 | app/api/admin.py |

### 5. 飞轮 / 主题 / 运维

| 方法 | 路径 | 作用 | 文件 |
|---|---|---|---|
| GET | /api/review/queue | 低置信度问答待审队列 | app/api/review.py |
| GET | /api/review/{review_id} | 待审单详情 | app/api/review.py |
| POST | /api/review/{review_id}/approve | 审核采纳（入知识库） | app/api/review.py |
| POST | /api/review/{review_id}/reject | 审核拒绝 | app/api/review.py |
| GET | /api/topics/distribution | 主题分布（17 类问题量） | app/api/topics.py |
| GET | /api/topics/questions | 类目下的问题列表（分页） | app/api/topics.py |
| GET | /api/jobs | 后台重跑任务列表 | app/api/jobs.py |
| POST | /api/jobs/{name} | 启动一个重跑任务（如 RAG 评估） | app/api/jobs.py |
| GET | /api/jobs/{name} | 查任务状态/结果 | app/api/jobs.py |
| POST | /api/jobs/{name}/stop | 停任务 | app/api/jobs.py |

## 三、注意事项

- 同名路径 /overview 分属不同前缀，完整路径各不相同：/api/acceptance/overview、/api/kb/overview、/api/rag-eval/overview、/api/observability/overview、/api/admin/overview。
- /api/chat 返回 SSE 流（多推 citations 等事件）；/api/agent 是一次性 JSON。
- /api/extract 是「朴素」结构化输出示例（with_structured_output 默认 json_schema 通路，异常兜底 502）；图编排内部另有一套加固过的 llm.structured()（function_calling + 重试）。
- actions.py 里的 create-ticket / create-refund 是前端表单落地的 HTTP 端点，与 Agent 工具（query_order 等 8 个）不是一回事。
- 页面路由均可直接访问；JSON API 未做鉴权，属教学项目范畴。

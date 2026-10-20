# 演示记录

以下为本地部署完成后的真实调用，响应已截断字段但未改写内容。
环境：WSL2 Ubuntu + Docker Desktop，MySQL / Milvus / MinIO / etcd 四容器，应用 :8000。

---

## 1. 查物流：两次工具调用

**请求**

```bash
curl -s http://localhost:8000/api/agent \
  -H 'Content-Type: application/json' \
  -d '{"user_id":"u1","message":"订单 1001 的物流到哪了"}'
```

**响应**

```json
{
  "conversation_id": 5,
  "answer": "亲，帮您查到订单 1001（智能猫砂盆）的物流情况啦～\n\n- 物流单号：SF213502378238\n- 当前状态：已揽件\n- 当前位置：成都\n- 最新轨迹：成都分拨中心 已发出",
  "tool_calls": [
    {"id": "call_00_zpw5...", "name": "query_order", "args": {"order_id": "1001"}},
    {"id": "call_00_O72S...", "name": "query_logistics", "args": {"tracking_no": "SF213502378238"}}
  ],
  "tool_results": [
    {
      "name": "query_order",
      "ok": true,
      "content": "{\"order_id\": \"1001\", \"status\": \"已付款\", \"amount\": 1739, \"product\": \"智能猫砂盆\", \"tracking_no\": \"SF213502378238\"}"
    },
    {
      "name": "query_logistics",
      "ok": true,
      "content": "{\"tracking_no\": \"SF213502378238\", \"status\": \"已揽件\", \"current_city\": \"成都\", \"trace\": [\"成都分拨中心 已发出\"]}"
    }
  ]
}
```

**看点**：模型没有直接编物流状态，而是先查订单拿到运单号，再查物流——两次调用有依赖关系，
这正是 Workflow 保证「必经步骤」的意义。

---

## 2. 政策问答：带引用生成

**请求**

```bash
curl -s http://localhost:8000/api/agent \
  -H 'Content-Type: application/json' \
  -d '{"user_id":"u1","message":"退货政策是什么样的"}'
```

**响应（节选）**

```json
{
  "answer": "**📦 无理由退货**\n- 支持 7 天无理由退货，自签收之日起 7 天内、商品完好且不影响二次销售的，可申请退货[1][10]。\n- 定制类、生鲜类商品除外[10]。\n\n**🚚 退换货运费谁承担**\n- 无理由退货：退回运费由买家承担[7]。\n- 因商品质量问题、发错货、少发漏发导致的退换货：运费由平台承担[7]。",
  "tool_calls": []
}
```

**看点**：`[1][10][7]` 是知识块编号，前端可展开原文；没有 `tool_calls` 说明走的是知识路，
不是工具路——两条链路的返回结构一致，但触发条件不同。

---

## 3. 库外问题：拒答而不是编造

知识库（6 篇 md，52 块）里没有「如何查询物流进度」这类操作指引。
问它时，四信号置信度不足，系统走兜底：

```
暂未查到相关信息。建议您联系人工客服获取准确答复。
```

同时后台写入 `low_confidence_questions`：

| 字段 | 内容 |
|---|---|
| source | `retrieval_low_conf` |
| reason | `evidence_confidence=0.0xx signals={...}` |
| retrieved_snapshot | Top-3 原文与 rerank 分数 |

**看点**：拒答不是故障，是设计。被拦的问题会进飞轮池，人工补完知识后重新入库，
同类问题下次就能答——这是「越用越准」的闭环。

---

## 4. 启动与停止

```bash
make dev        # 依赖容器 → 等 Milvus 健康 → MCP :8101/:8102 → 应用 :8000
make dev-down   # 停应用与 MCP Server
docker compose stop   # 停容器，数据保留
```

启动日志里能看到关键状态：

```
INFO:app.main:Milvus 预热完成,集合可检索
INFO:app.tools.registry:内置工具注册完成:['create_ticket', 'query_faq', 'query_order', 'query_product', 'submit_refund']
INFO:app.graph.runtime:图已编译,checkpointer=data/agent_checkpoints.sqlite
INFO:     Uvicorn running on http://127.0.0.1:8000
```

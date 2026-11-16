# 多代理协作协议（Agent2Agent 子集 + 文件队列）

同一台机器上两个 AI 代理（一个偏写作/评审，一个偏执行/批处理）通过标准任务接口或文件队列交接工作的轻量协议。**权威文档见 [PROTOCOL.md](PROTOCOL.md)**，本文件是索引。

## 特性

- **两种传输**：A2A JSON-RPC 子集（`message/send`、`tasks/get`、`tasks/list`、`tasks/cancel`）或零依赖的文件队列
- **标准状态机**：`submitted → working → completed / failed / canceled`
- **可审计**：每个任务一个 JSON，请求/回复/产物都落盘
- **轻量验收**：默认静态核对 + 抽样回源，不要求全量复跑
- **单一写者**：同一文件同一时刻只允许一个代理写，原始数据只读

## 目录

```
collab/
├── PROTOCOL.md       协议全文（数据模型、状态机、接口、剧本、验收、安全）
├── tasks/            长任务登记
├── inbox/<agent>/    请求
├── outbox/<agent>/   回复
├── results/<task>/   产物
├── handoff/          交接与核验
└── a2a/              A2A 实现（Agent Card、服务端/客户端、启停脚本）
```

## 快速上手

```bash
# 启动一个 A2A 代理（队列模式：任务入队，由被调方处理）
python a2a/server.py --mode queue --port <port> --card a2a/agent-card.json

# 发任务并等待结果
python a2a/client.py --url http://127.0.0.1:<port>/a2a --text "任务描述" --timeout 600

# 查看任务
python a2a/client.py --url http://127.0.0.1:<port>/a2a --list --state completed

# 不跑 HTTP 时的文件队列
bash bin/ask-dsh.sh "任务描述"
```

## 任务 JSON 示例

```json
{
  "id": "T001",
  "title": "示例任务",
  "owner": "agent-b",
  "status": "pending",
  "goal": "交付可复现的结果",
  "input": {},
  "deliverables": ["results/T001/out.csv"],
  "acceptance": ["字段与单位齐全", "抽 1 条可回源核对"]
}
```

## 设计取舍

- **为什么两种传输**：A2A 便于第三方工具接入；文件队列在无 HTTP 依赖时也能工作，二者共用同一任务模型。
- **为什么验收不复跑**：复跑成本高；默认静态核对 + 抽样足以发现大多数问题，只有证据矛盾或不可逆操作才升级。
- **为什么原始数据只读**：上游数据是事实来源，任何修复都应生成为新产物并留痕。

## 许可

未指定许可证；如需开源请补充 LICENSE。

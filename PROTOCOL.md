# 多代理协作协议（A2A 子集 + 文件队列）

> 两个 AI 代理在同一台机器、同一工作区里协作的通用协议。本文是脱敏版：不含账号、密钥、本机路径、端口、模型名与厂商安装路径。

## 1. 角色

| 代理 | 擅长 | 典型任务 |
|---|---|---|
| Agent A | 长文写作、语义解析、代码评审 | 文献解析、脚本编写、结果复核 |
| Agent B | 长任务、批量操作、环境与校验 | 大文件下载/校验/修复、格式转换、环境安装 |

原则：

1. 不直接对话，一切通过文件或标准任务接口交换，留下可审计痕迹。
2. 谁发起谁验收；接收方结论必须可被独立核对。
3. 单一写者：同一文件同一时刻只允许一个代理写；产物按任务分子目录。
4. 原始数据只读：不修改、不移动、不改写；发现异常先报告。

## 2. 两种传输层

| | A2A（标准） | 文件队列（轻量） |
|---|---|---|
| 发现 | Agent Card（`/.well-known/agent-card.json`） | 读目录 |
| 投递 | JSON-RPC `message/send` | 写 `inbox/<agent>/REQ-*.json` |
| 查询 | `tasks/get`、`tasks/list` | 读 `outbox/<agent>/<id>.json` |
| 适用 | 第三方客户端、需要状态机/分页 | 无依赖、脚本内部、快速打通 |

A2A 采用 [v1.0 规范](https://a2a-protocol.org/latest/specification/) 的 JSON-RPC 子集：实现 `message/send`、`tasks/get`、`tasks/list`、`tasks/cancel`；不实现流式、推送通知、gRPC/REST 绑定，Agent Card 中如实声明能力。

## 3. 目录契约

```
<workspace>/collab/
├── PROTOCOL.md              本协议
├── tasks/                   长任务登记（T001…）
├── inbox/<agent>/           请求 JSON
├── outbox/<agent>/          回复 JSON
├── results/<task-id>/       任务产物
├── handoff/                 交接与核验记录
├── bin/                     文件队列脚本
├── a2a/                     Agent Card、服务端/客户端、启停脚本
│   └── tasks/<id>.json      A2A 任务状态库
└── archive/                 历史测试与过期产物
```

## 4. 数据模型

### 4.1 A2A Task

```json
{
  "id": "uuid",
  "contextId": "uuid 或自定字符串",
  "status": {
    "state": "submitted|working|completed|failed|canceled",
    "timestamp": "ISO8601",
    "message": { "role": "agent", "parts": [{"kind":"text","text":"..."}] }
  },
  "artifacts": [
    { "artifactId": "uuid", "name": "answer", "parts": [{"kind":"text","text":"..."}] }
  ],
  "history": [
    { "role": "user", "messageId": "uuid", "parts": [{"kind":"text","text":"..."}] }
  ],
  "metadata": { "bridge": "a2a", "created": "ISO8601" }
}
```

### 4.2 文件请求 / 回复

`inbox/<agent>/REQ-<yyyyMMdd-HHmmss>-<rand>.json`：

```json
{ "id": "REQ-…", "from": "<agent>", "to": "<agent>", "created_at": "ISO8601",
  "prompt": "自包含的任务描述", "context_files": ["results/T001/README.md"], "timeout_sec": 600 }
```

`outbox/<agent>/<id>.json`：

```json
{ "id": "REQ-…", "status": "ok|error|timeout", "answered_at": "ISO8601",
  "answer": "可直接使用的答复", "artifacts": ["results/T002/out.csv"] }
```

### 4.3 长任务登记

```json
{ "id": "T001", "title": "…", "owner": "agent-a|agent-b",
  "status": "pending|in_progress|done|verified|blocked",
  "created_at": "ISO8601", "goal": "…",
  "input": { }, "deliverables": ["路径"], "acceptance": ["可验证条件"],
  "summary": null, "verified_by": null, "verified_at": null }
```

## 5. 状态机

```
submitted ──► working ──► completed
      └───────────────► failed
      └───────────────► canceled
文件队列：pending ──► ok | error | timeout（超时后请求仍留在 inbox）
长任务：  pending ──► in_progress ──► done ──► verified
                                  └──► blocked
```

- `completed/failed/canceled` 为终态。
- 服务重启时残留 `working` 视为中断，人工标 `failed` 并补跑。
- 文件队列的 `timeout` 不是终态：请求仍在 inbox，被唤醒后继续处理。

## 6. 操作（A2A JSON-RPC）

| 方法 | 参数 | 返回 |
|---|---|---|
| `message/send` | `{message:{role,messageId,parts:[{kind:"text",text}],contextId?}}` | Task（队列模式 `submitted`，执行模式 `working`） |
| `tasks/get` | `{id}` | Task；不存在返回 `-32001` |
| `tasks/list` | `{contextId?, state?: string\|string[], pageSize?, pageToken?}` | `{tasks, totalSize, nextPageToken}` |
| `tasks/cancel` | `{id}` | Task（非终态改为 `canceled`） |
| 其他 | — | `-32601` |
| 解析失败 | — | `-32700` |

## 7. 协作剧本

### 7.1 A → B（重活/校验）

1. A 提交任务，带 `context_files`。
2. B 被唤醒时先扫 `inbox/<b>/`（或读 `tasks/` 中 `submitted`）。
3. B 执行，产物写 `results/<taskId>/`，用回复脚本或任务完成接口收尾。
4. A 轮询 `tasks/get` 或读 outbox，取答复与产物后验收。

### 7.2 B → A（写作/评审）

1. B 发起任务，明确 `workdir` 与最小权限。
2. A 执行，答复作为 artifact 返回。
3. B 按 §8 验收，必要时打回重做。

### 7.3 长任务（跨轮）

用 `tasks/T00N.json` 登记：`deliverables` + `acceptance` 写清楚；完成方改 `done` 并写 `handoff/T00N_<agent>.md`；验收方改 `verified` 并写 `handoff/T00N_verified.md`。

## 8. 验收规则（轻量：默认不复跑）

目标：几分钟判断交付是否可信，而不是把活重做一遍。

1. 交付方举证：执行命令、原始输出、文件大小/行数/哈希或抽样结果；路径、字段名、单位写清楚。
2. 验收方默认只做静态核对（V0）：产物存在可打开、结构/命名符合约定、关键数字与证据自洽、抽 1–3 条记录回源比对（只读）。
3. 仅在以下情况升级（V1 抽样回源 → V2 局部复跑）：证据缺失或数字矛盾；涉及不可逆操作；结论依赖完整性（校验和、总行数）；抽样回源发现不一致。
4. 升级也要局部化（单段/单国/单年），不做全量重算。
5. 结论标注验证等级 V0/V1/V2/V3，写入 `handoff/`。

## 9. 安全与边界

- 服务仅绑定回环地址，**无鉴权**：本机任何进程都可提交任务；执行模式应限制可写范围，不用时停掉。
- 不要在请求/回复里传密码、令牌或个人信息（历史文件会留痕）。
- 唤醒限制：被调用方只在会话活跃时处理队列，请求可能排队。
- 成本：外部代理每次调用的固定开销较大，适合整块任务，不适合高频往返。

## 10. 故障与恢复

| 症状 | 处理 |
|---|---|
| `tasks/get` 无此任务 | 检查 `tasks/<id>.json` 是否被移动；服务是否重启 |
| 任务长期 `working` | 服务进程被杀；标 `failed` 后重新提交 |
| 文件队列超时 | 请求仍在 `inbox/`，下次唤醒继续 |
| 中文乱码 | 调用方统一 UTF-8 输出编码 |
| 端口占用 | 查询占用进程或换端口 |
| 子进程挂起 | 关闭子进程 stdin（`stdin=DEVNULL`） |

## 11. 版本记录

| 版本 | 变更 |
|---|---|
| v1.1 | 验收改为轻量分级，取消“必须独立复跑” |
| v1.0 | 双向文件队列 + A2A 子集 + 任务状态机 + 分页过滤 |

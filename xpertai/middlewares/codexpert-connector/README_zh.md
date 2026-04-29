# XpertAI 插件：Codexpert 连接器中间件

`@xpert-ai/plugin-codexpert-connector` 用来把 Xpert Agent 连接到 Codexpert 编码会话。

它向 Agent 暴露 Codexpert 上下文工具，同步执行 Codexpert 编码任务，把 Codexpert 的用户可见输出实时投影给用户，并把精简后的任务元信息返回给 Agent。

英文文档见 [README.md](./README.md)。

## 当前运行模型

- 插件是连接型 middleware，不是 Codexpert 运行时。
- Codexpert 仍然是 coding session、task、thread、execution、environment 的事实源。
- Xpert 只负责 Agent 编排和用户可见投影。
- Agent 拿到上下文工具和最终任务元信息。
- 用户看到 Codexpert 可见正文，插件只做轻量清洗。
- 插件只保存一张轻量运行映射表，用于恢复和排查。

## 设计思路

### Codexpert 负责编码状态

Codexpert 负责 MCP 工具、编码会话锚点、任务执行、环境创建和环境复用。

插件不 clone 仓库，不创建编码环境，也不把 Codexpert task 详情复制到 Xpert。

### 插件负责连接和投影

插件从 Xpert 当前运行上下文读取真实业务用户身份，把身份头注入到 Codexpert MCP 和 connector stream 请求里，并把 Codexpert stream event 转成 Xpert 用户消息流支持的文本 chunk。

### Agent 只拿元信息

Codexpert 正文直接投影给用户。Agent 最终只拿精简元信息，例如 session id、task id、thread id、execution id、仓库、分支、环境 id、summary 和 error。

Agent 不需要消费 Codexpert raw stream event，也不负责重新总结 Codexpert 正文。

## 代码入口

| 模块 | 文件 |
| --- | --- |
| 插件入口 | `src/index.ts` |
| Middleware strategy | `src/lib/codexpert-connector.middleware.ts` |
| Codexpert HTTP / MCP client | `src/lib/codexpert-connector.client.ts` |
| 用户可见投影 | `src/lib/codexpert-visible-projector.ts` |
| 类型与配置 schema | `src/lib/types.ts` |
| 运行映射服务 | `src/lib/codexpert-connector-run.service.ts` |
| 运行映射实体 | `src/lib/entities/codexpert-connector-run.entity.ts` |

middleware strategy 名称是：

```text
CodexpertConnector
```

## 配置项

| 字段 | 类型 | 默认值 | 说明 |
| --- | --- | --- | --- |
| `codexpertMcpUrl` | `string` | 无 | Codexpert MCP 地址，例如 `http://localhost:3001/v1/mcp`。 |
| `codexpertConnectorBaseUrl` | `string` | 无 | Codexpert connector API 基础地址，例如 `http://localhost:3001/api`。不要配到 `/api/codexpert-connector`，客户端会自动拼接后续路径。 |
| `serviceToken` | `string` | 无 | 插件调用 Codexpert MCP 和 connector stream 接口使用的服务令牌。 |
| `timeoutMs` | `number` | `600000` | MCP 调用和同步任务 stream 的超时时间，单位毫秒。 |
| `enableVisibleProjection` | `boolean` | `true` | 是否把 Codexpert 可见输出投影给用户。 |
| `enableStatusEvents` | `boolean` | `true` | 是否投影精简状态里程碑。 |
| `defaultXpertId` | `string` | 无 | 默认 Codexpert 编码助手 id。 |
| `defaultRepoId` | `string` | 无 | 默认仓库 id。 |
| `defaultConnectionId` | `string` | 无 | 默认 Git connection id。 |
| `defaultBranchName` | `string` | 无 | 默认分支名。 |

示例：

```json
{
  "codexpertMcpUrl": "http://localhost:3001/v1/mcp",
  "codexpertConnectorBaseUrl": "http://localhost:3001/api",
  "serviceToken": "<codexpert-service-token>",
  "timeoutMs": 600000,
  "enableVisibleProjection": true,
  "enableStatusEvents": true,
  "defaultXpertId": "<optional-codexpert-assistant-id>",
  "defaultRepoId": "<optional-repo-id>",
  "defaultConnectionId": "<optional-git-connection-id>",
  "defaultBranchName": "main"
}
```

如果缺少 `codexpertMcpUrl`、`codexpertConnectorBaseUrl` 或 `serviceToken`，插件只注册 middleware 名称，不会暴露 Codexpert 工具。

## 身份透传

插件会从 Xpert 当前运行上下文解析业务用户身份：

- `tenantId`
- `organizationId`
- `userId`

每次请求 Codexpert 都会注入：

```text
Authorization: Bearer <serviceToken>
tenant-id: <tenantId>
organization-id: <organizationId>
x-principal-user-id: <userId>
```

规则：

- 缺 `tenantId`、`organizationId` 或 `userId` 时，插件直接失败。
- 插件不根据外部平台 open id 推断 Codexpert 用户。
- 飞书用户必须先在 Xpert 侧绑定并解析成真实业务用户。

## 暴露给 Agent 的工具

插件透传以下 Codexpert MCP 工具：

- `listCodingAssistants`
- `listCodexpertConversations`
- `listGitConnections`
- `listGitRepositories`
- `listGitBranches`
- `selectCodingContext`
- `resolveCodexpertConversationContext`
- `resumeCodexpertSession`

PR 发布和 issue 评论工具不会通过这个外部连接器暴露。仓库写回和 PR 交付由 Codexpert/code agent 在编码任务内部负责。如果 Codexpert 创建了 PR，连接器只会把 PR URL 作为结果元信息返回。

插件额外提供：

```text
runCodexpertTask
```

`runCodexpertTask` 输入：

| 字段 | 必填 | 说明 |
| --- | --- | --- |
| `prompt` | 是 | 要交给 Codexpert 执行的任务内容。 |
| `taskTitle` | 否 | 任务标题。 |
| `codingSessionId` | 否 | 已有 Codexpert coding session。 |
| `conversationId` | 否 | 用于恢复上下文。 |
| `threadId` | 否 | 用于恢复上下文。 |
| `taskId` | 否 | 用于恢复上下文。 |
| `xpertId` | 否 | Codexpert 编码助手 id；缺省时使用 `defaultXpertId`。 |
| `repoId` | 否 | Codexpert 仓库 id；缺省时使用 `defaultRepoId`。 |
| `connectionId` | 否 | Git connection id；缺省时使用 `defaultConnectionId`。 |
| `branchName` | 否 | 分支名；缺省时使用 `defaultBranchName`。 |
| `timeoutMs` | 否 | 覆盖本次任务的超时时间。 |

返回给 Agent 的结果：

```ts
type CodexpertTaskResult = {
  status: 'success' | 'failed' | 'timeout' | 'canceled'
  codingSessionId: string | null
  taskId: string | null
  threadId: string | null
  executionId: string | null
  repo: {
    id?: string | null
    name?: string | null
    owner?: string | null
    slug?: string | null
  } | null
  branch: string | null
  environmentId: string | null
  environmentReused: boolean | null
  summary: string | null
  error: string | null
  prUrl?: string | null
}
```

## 推荐 Agent 调用流程

没有默认上下文时：

1. `listCodingAssistants`
2. `listGitConnections`
3. `listGitRepositories`
4. `listGitBranches`
5. `selectCodingContext`
6. `runCodexpertTask`

已有任务或会话时：

1. `listCodexpertConversations` 或 `resolveCodexpertConversationContext`
2. `resumeCodexpertSession`
3. `runCodexpertTask`

如果配置了 `defaultXpertId`、`defaultConnectionId`、`defaultRepoId`、`defaultBranchName`，`runCodexpertTask` 可以在没有 `codingSessionId` 的情况下创建 session。

## 用户可见投影

Codexpert stream event 会被转换成 Xpert 支持的文本 chunk：

```ts
{
  type: 'text',
  text,
  xpertName: 'Codexpert',
  agentKey: 'codexpert'
}
```

投影规则：

- `text_delta` 尽量保留原始空格和 Markdown。
- 过滤 `thought` stream。
- 已经收到正文后，过滤重复的 `assistant_snapshot`。
- 过滤 raw metadata、debug、setup log 等内部文本。
- 过滤安装依赖、clone progress 等 setup 噪音。
- `done.summary` 只在没有正文输出时显示，避免重复刷屏。

状态里程碑默认开启，并会去重：

- `正在准备编码环境。`
- `编码环境已就绪。`
- `已开始处理。`
- `需要补充信息。`
- `已完成。`
- `Codexpert failed: <message>`

## 运行映射表

插件维护一张轻量表：

```text
plugin_codexpert_connector_run
```

记录字段包括：

- `tenant_id`
- `organization_id`
- `user_id`
- `xpert_id`
- `conversation_id`
- `execution_id`
- `coding_session_id`
- `task_id`
- `thread_id`
- `codexpert_execution_id`
- `status`
- `last_error`
- `metadata`
- `created_at`
- `updated_at`

这张表用于记录 Xpert execution 到 Codexpert session、task、thread、execution 的映射。它不复制 Codexpert task 详情，也不保存 raw stream event。

## 本地开发

编译插件：

```bash
cd /Users/lilinhao/project/code-xpert/codexpertplugin/xpertplugins
./xpertai/node_modules/.bin/tsc -p xpertai/middlewares/codexpert-connector/tsconfig.lib.json
```

本地常见地址：

```text
Codexpert MCP:       http://localhost:3001/v1/mcp
Codexpert API base:  http://localhost:3001/api
Xpert backend:       http://localhost:3100
```

Codexpert 后端需要提供：

```text
POST /v1/mcp
POST /api/codexpert-connector/sessions
POST /api/codexpert-connector/sessions/:sessionId/prompts/stream
```

## 边界

- 插件不是 Codexpert 运行时的替代品。
- 插件不在 Xpert 侧实现编码环境创建、仓库 clone 或任务执行。
- 插件不保存 Codexpert raw event。
- 插件依赖 Xpert 当前上下文里已经有真实业务用户身份。
- `serviceToken` 是服务间认证令牌，不应由 Agent 或终端用户直接输入。

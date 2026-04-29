# XpertAI Plugin: Codexpert Connector Middleware

`@xpert-ai/plugin-codexpert-connector` connects an Xpert agent to Codexpert coding sessions.

It exposes Codexpert context tools to the agent, runs Codexpert tasks synchronously, streams Codexpert-visible output to the user, and returns compact task metadata back to the agent.

For the Chinese documentation, see [README_zh.md](./README_zh.md).

## Current Runtime Model

- The plugin is a connector middleware, not a Codexpert runtime.
- Codexpert remains the source of truth for coding sessions, tasks, threads, executions, and environments.
- Xpert only owns agent orchestration and user-visible projection.
- The agent receives context tools plus final task metadata.
- The user sees Codexpert-visible text directly, after light cleanup.
- The plugin stores only a lightweight run mapping table for recovery and diagnostics.

## Design Principles

### Codexpert Owns Coding State

Codexpert owns MCP tools, coding session anchors, task execution, environment setup, and environment reuse.

The plugin does not clone repositories, create coding environments, or copy Codexpert task details into Xpert.

### The Plugin Connects And Projects

The plugin reads the current business principal from the Xpert runtime context, injects identity headers into Codexpert requests, and converts Codexpert stream events into Xpert-compatible visible text chunks.

### The Agent Receives Metadata

Codexpert text is projected to the user. The agent receives only compact terminal metadata, such as session id, task id, thread id, execution id, repository, branch, environment id, summary, and error.

The agent should not consume raw Codexpert stream events or summarize Codexpert output again.

## Code Entry Points

| Module | File |
| --- | --- |
| Plugin entry | `src/index.ts` |
| Middleware strategy | `src/lib/codexpert-connector.middleware.ts` |
| Codexpert HTTP / MCP client | `src/lib/codexpert-connector.client.ts` |
| Visible projection | `src/lib/codexpert-visible-projector.ts` |
| Types and config schema | `src/lib/types.ts` |
| Run mapping service | `src/lib/codexpert-connector-run.service.ts` |
| Run mapping entity | `src/lib/entities/codexpert-connector-run.entity.ts` |

The middleware strategy name is:

```text
CodexpertConnector
```

## Configuration

| Field | Type | Default | Description |
| --- | --- | --- | --- |
| `codexpertMcpUrl` | `string` | none | Codexpert MCP URL, for example `http://localhost:3001/v1/mcp`. |
| `codexpertConnectorBaseUrl` | `string` | none | Codexpert connector API base URL, for example `http://localhost:3001/api`. Do not include `/api/codexpert-connector`; the client appends the connector path. |
| `serviceToken` | `string` | none | Service token used for Codexpert MCP and connector stream requests. |
| `timeoutMs` | `number` | `600000` | Timeout for MCP calls and synchronous stream execution, in milliseconds. |
| `enableVisibleProjection` | `boolean` | `true` | Whether to project Codexpert-visible output to the user. |
| `enableStatusEvents` | `boolean` | `true` | Whether to project compact status milestones. |
| `defaultXpertId` | `string` | none | Default Codexpert coding assistant id. |
| `defaultRepoId` | `string` | none | Default repository id. |
| `defaultConnectionId` | `string` | none | Default Git connection id. |
| `defaultBranchName` | `string` | none | Default branch name. |

Example:

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

If `codexpertMcpUrl`, `codexpertConnectorBaseUrl`, or `serviceToken` is missing, the middleware name is still registered, but no Codexpert tools are exposed.

## Identity Propagation

The plugin resolves the business principal from the current Xpert runtime context:

- `tenantId`
- `organizationId`
- `userId`

Every request to Codexpert includes:

```text
Authorization: Bearer <serviceToken>
tenant-id: <tenantId>
organization-id: <organizationId>
x-principal-user-id: <userId>
```

Rules:

- Missing `tenantId`, `organizationId`, or `userId` fails immediately.
- The plugin does not infer Codexpert users from external-platform open ids.
- Lark/Feishu users must already be bound to real business users on the Xpert side.

## Agent Tools

The plugin passes through these Codexpert MCP tools:

- `listCodingAssistants`
- `listCodexpertConversations`
- `listGitConnections`
- `listGitRepositories`
- `listGitBranches`
- `selectCodingContext`
- `resolveCodexpertConversationContext`
- `resumeCodexpertSession`
- `createPR`
- `publishTaskPR`
- `createIssueComment`

The plugin also provides:

```text
runCodexpertTask
```

`runCodexpertTask` input:

| Field | Required | Description |
| --- | --- | --- |
| `prompt` | yes | Task prompt to send to Codexpert. |
| `taskTitle` | no | Optional task title. |
| `codingSessionId` | no | Existing Codexpert coding session. |
| `conversationId` | no | Context recovery key. |
| `threadId` | no | Context recovery key. |
| `taskId` | no | Context recovery key. |
| `xpertId` | no | Codexpert coding assistant id; falls back to `defaultXpertId`. |
| `repoId` | no | Codexpert repository id; falls back to `defaultRepoId`. |
| `connectionId` | no | Git connection id; falls back to `defaultConnectionId`. |
| `branchName` | no | Branch name; falls back to `defaultBranchName`. |
| `timeoutMs` | no | Per-call timeout override. |

Result returned to the agent:

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

## Recommended Agent Flow

Without default context:

1. `listCodingAssistants`
2. `listGitConnections`
3. `listGitRepositories`
4. `listGitBranches`
5. `selectCodingContext`
6. `runCodexpertTask`

For an existing task or session:

1. `listCodexpertConversations` or `resolveCodexpertConversationContext`
2. `resumeCodexpertSession`
3. `runCodexpertTask`

If `defaultXpertId`, `defaultConnectionId`, `defaultRepoId`, and `defaultBranchName` are configured, `runCodexpertTask` can create a session without an explicit `codingSessionId`.

## Visible Projection

Codexpert stream events are converted into Xpert-compatible text chunks:

```ts
{
  type: 'text',
  text,
  xpertName: 'Codexpert',
  agentKey: 'codexpert'
}
```

Projection behavior:

- `text_delta` preserves spaces and Markdown as much as possible.
- The `thought` stream is filtered.
- `assistant_snapshot` is filtered once normal output text has already appeared.
- Raw metadata, debug messages, and setup logs are filtered.
- Setup noise such as dependency installation and clone progress is filtered.
- `done.summary` is only shown when no output text was projected.

Status milestones are enabled by default and deduplicated:

- `正在准备编码环境。`
- `编码环境已就绪。`
- `已开始处理。`
- `需要补充信息。`
- `已完成。`
- `Codexpert failed: <message>`

## Run Mapping Table

The plugin maintains a lightweight table:

```text
plugin_codexpert_connector_run
```

It records:

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

The table maps Xpert executions to Codexpert sessions, tasks, threads, and executions. It does not copy Codexpert task details or raw stream events.

## Local Development

Compile the plugin:

```bash
cd /Users/lilinhao/project/code-xpert/codexpertplugin/xpertplugins
./xpertai/node_modules/.bin/tsc -p xpertai/middlewares/codexpert-connector/tsconfig.lib.json
```

Common local URLs:

```text
Codexpert MCP:       http://localhost:3001/v1/mcp
Codexpert API base:  http://localhost:3001/api
Xpert backend:       http://localhost:3100
```

Codexpert backend must provide:

```text
POST /v1/mcp
POST /api/codexpert-connector/sessions
POST /api/codexpert-connector/sessions/:sessionId/prompts/stream
```

## Boundaries

- The plugin is not a replacement for the Codexpert runtime.
- It does not implement coding environment creation, repository cloning, or task execution inside Xpert.
- It does not store Codexpert raw events.
- It requires a real business principal in the current Xpert context.
- `serviceToken` is a service-to-service credential and should not be provided by the agent or end user.

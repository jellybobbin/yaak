# RFC: Plugin API Abstractions & MCP Server Support

## Summary

Add a comprehensive plugin API abstraction layer to Yaak, enabling plugins to interact with workspaces, requests, and responses through well-defined interfaces. This lays the groundwork for an MCP (Model Context Protocol) server that allows AI assistants like Claude to create, send, and manage API requests.

## Motivation

### Current State
- Plugins have limited access to Yaak data (only `httpRequest.getById()`, `httpRequest.send()`, `httpResponse.find()`)
- Raw database models are exposed directly, coupling plugins to internal schema
- No way to list, create, update, or delete requests from plugins
- No MCP integration exists

### Goals
1. **Richer plugin capabilities** - Enable plugins to fully manage workspaces, folders, requests, environments
2. **Stable API contract** - Abstract over internal models so schema changes don't break plugins
3. **Safety & validation** - Control what can be read/written, enforce invariants
4. **MCP server support** - Expose Yaak functionality to AI assistants via Model Context Protocol

## Design

### Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                     MCP Server (Plugin)                      │
│                    @modelcontextprotocol/sdk                 │
└─────────────────────────┬───────────────────────────────────┘
                          │
┌─────────────────────────▼───────────────────────────────────┐
│                  Plugin API Abstraction Layer                │
│                                                              │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐    │
│  │  Workspace  │ │ HttpRequest │ │    HttpResponse     │    │
│  │   Handle    │ │   Handle    │ │       Handle        │    │
│  └─────────────┘ └─────────────┘ └─────────────────────┘    │
│  ┌─────────────┐ ┌─────────────┐ ┌─────────────────────┐    │
│  │   Folder    │ │ Environment │ │    GrpcRequest      │    │
│  │   Handle    │ │   Handle    │ │       Handle        │    │
│  └─────────────┘ └─────────────┘ └─────────────────────┘    │
└─────────────────────────┬───────────────────────────────────┘
                          │ Plugin Events (WebSocket)
┌─────────────────────────▼───────────────────────────────────┐
│                    Rust Backend (Tauri)                      │
│                                                              │
│  plugin_events.rs  →  yaak-models  →  SQLite                │
└─────────────────────────────────────────────────────────────┘
```

### Abstraction Layer Design

Instead of exposing raw database models, provide handle objects with controlled access:

```typescript
// ❌ Current: Raw model exposure
interface Context {
  httpRequest: {
    getById(args: { id: string }): Promise<HttpRequest | null>;
  };
}

// ✅ Proposed: Abstraction layer
interface Context {
  workspace: WorkspaceAPI;
  folder: FolderAPI;
  httpRequest: HttpRequestAPI;
  httpResponse: HttpResponseAPI;
  environment: EnvironmentAPI;
  grpcRequest: GrpcRequestAPI;
}
```

### HttpRequest API Example

```typescript
interface HttpRequestAPI {
  // Query
  list(args: { workspaceId: string; folderId?: string }): Promise<HttpRequestHandle[]>;
  getById(id: string): Promise<HttpRequestHandle | null>;

  // Mutate
  create(args: CreateHttpRequestArgs): Promise<HttpRequestHandle>;
  update(id: string, args: UpdateHttpRequestArgs): Promise<HttpRequestHandle>;
  delete(id: string): Promise<void>;
  duplicate(id: string): Promise<HttpRequestHandle>;

  // Actions
  send(id: string): Promise<HttpResponseHandle>;
}

interface HttpRequestHandle {
  // Read-only properties
  readonly id: string;
  readonly workspaceId: string;
  readonly folderId: string | null;
  readonly name: string;
  readonly url: string;
  readonly method: HttpMethod;
  readonly createdAt: string;
  readonly updatedAt: string;

  // Nested data accessors
  getHeaders(): HttpHeader[];
  getQueryParams(): QueryParam[];
  getBody(): RequestBody;
  getAuthentication(): Authentication | null;

  // Mutation methods
  setName(name: string): Promise<void>;
  setUrl(url: string): Promise<void>;
  setMethod(method: HttpMethod): Promise<void>;
  setHeader(name: string, value: string): Promise<void>;
  removeHeader(name: string): Promise<void>;
  setBody(body: RequestBody): Promise<void>;

  // Actions
  send(): Promise<HttpResponseHandle>;
  duplicate(): Promise<HttpRequestHandle>;
  moveTo(folderId: string | null): Promise<void>;
}

interface CreateHttpRequestArgs {
  workspaceId: string;
  folderId?: string;
  name: string;
  url?: string;
  method?: HttpMethod;
  headers?: HttpHeader[];
  body?: RequestBody;
}

interface UpdateHttpRequestArgs {
  name?: string;
  url?: string;
  method?: HttpMethod;
  headers?: HttpHeader[];
  body?: RequestBody;
  folderId?: string | null;
}
```

### Full API Surface

#### WorkspaceAPI
```typescript
interface WorkspaceAPI {
  list(): Promise<WorkspaceHandle[]>;
  getById(id: string): Promise<WorkspaceHandle | null>;
  getActive(): Promise<WorkspaceHandle | null>;
  create(args: { name: string }): Promise<WorkspaceHandle>;
}
```

#### FolderAPI
```typescript
interface FolderAPI {
  list(args: { workspaceId: string }): Promise<FolderHandle[]>;
  getById(id: string): Promise<FolderHandle | null>;
  create(args: { workspaceId: string; name: string; parentId?: string }): Promise<FolderHandle>;
  update(id: string, args: { name?: string; parentId?: string }): Promise<FolderHandle>;
  delete(id: string): Promise<void>;
}
```

#### EnvironmentAPI
```typescript
interface EnvironmentAPI {
  list(args: { workspaceId: string }): Promise<EnvironmentHandle[]>;
  getById(id: string): Promise<EnvironmentHandle | null>;
  getActive(workspaceId: string): Promise<EnvironmentHandle | null>;
  create(args: { workspaceId: string; name: string }): Promise<EnvironmentHandle>;
  update(id: string, args: { name?: string }): Promise<EnvironmentHandle>;
  delete(id: string): Promise<void>;
}

interface EnvironmentHandle {
  readonly id: string;
  readonly name: string;

  getVariables(): EnvironmentVariable[];
  setVariable(name: string, value: string): Promise<void>;
  removeVariable(name: string): Promise<void>;
}
```

#### HttpResponseAPI
```typescript
interface HttpResponseAPI {
  list(args: { requestId: string; limit?: number }): Promise<HttpResponseHandle[]>;
  getById(id: string): Promise<HttpResponseHandle | null>;
  getLatest(requestId: string): Promise<HttpResponseHandle | null>;
}

interface HttpResponseHandle {
  readonly id: string;
  readonly requestId: string;
  readonly status: number;
  readonly statusText: string;
  readonly elapsed: number;
  readonly createdAt: string;

  getHeaders(): HttpHeader[];
  getBody(): ResponseBody;
  getBodyText(): Promise<string>;
  getBodyJson<T>(): Promise<T>;
}
```

## Implementation Plan

### Phase 1: Core Plugin Events
Add Rust events for CRUD operations:

| Event | Request | Response |
|-------|---------|----------|
| List workspaces | `ListWorkspacesRequest` | `ListWorkspacesResponse` |
| List folders | `ListFoldersRequest` | `ListFoldersResponse` |
| List requests | `ListHttpRequestsRequest` | `ListHttpRequestsResponse` |
| Create request | `CreateHttpRequestRequest` | `CreateHttpRequestResponse` |
| Update request | `UpdateHttpRequestRequest` | `UpdateHttpRequestResponse` |
| Delete request | `DeleteHttpRequestRequest` | `DeleteHttpRequestResponse` |
| List environments | `ListEnvironmentsRequest` | `ListEnvironmentsResponse` |
| ... | ... | ... |

**Files to modify:**
1. `src-tauri/yaak-plugins/src/events.rs` - Define event structs
2. `src-tauri/src/plugin_events.rs` - Handle events
3. `src-tauri/yaak-models/src/queries/` - Add any missing query functions

### Phase 2: TypeScript Abstraction Layer
Create handle classes in plugin-runtime-types:

**New files:**
- `packages/plugin-runtime-types/src/handles/WorkspaceHandle.ts`
- `packages/plugin-runtime-types/src/handles/FolderHandle.ts`
- `packages/plugin-runtime-types/src/handles/HttpRequestHandle.ts`
- `packages/plugin-runtime-types/src/handles/HttpResponseHandle.ts`
- `packages/plugin-runtime-types/src/handles/EnvironmentHandle.ts`

**Modified files:**
- `packages/plugin-runtime-types/src/plugins/Context.ts` - New API shape
- `packages/plugin-runtime/src/PluginInstance.ts` - Implement new methods

### Phase 3: MCP Server Plugin
Create MCP server as a Yaak plugin:

```typescript
// plugins/mcp-server/src/index.ts
import { McpServer } from "@modelcontextprotocol/sdk/server/mcp.js";

export const plugin: PluginDefinition = {
  async init(ctx) {
    const server = new McpServer({ name: "yaak", version: "1.0.0" });

    server.tool("list_requests", { workspaceId: z.string() }, async ({ workspaceId }) => {
      const requests = await ctx.httpRequest.list({ workspaceId });
      return { content: [{ type: "text", text: JSON.stringify(requests) }] };
    });

    server.tool("send_request", { requestId: z.string() }, async ({ requestId }) => {
      const response = await ctx.httpRequest.send(requestId);
      return { content: [{ type: "text", text: await response.getBodyText() }] };
    });

    // ... more tools

    await server.connect(new StdioServerTransport());
  }
};
```

### MCP Tools Mapping

| MCP Tool | Plugin API Method |
|----------|-------------------|
| `list_workspaces` | `ctx.workspace.list()` |
| `list_requests` | `ctx.httpRequest.list({ workspaceId })` |
| `get_request` | `ctx.httpRequest.getById(id)` |
| `create_request` | `ctx.httpRequest.create(args)` |
| `update_request` | `ctx.httpRequest.update(id, args)` |
| `delete_request` | `ctx.httpRequest.delete(id)` |
| `send_request` | `ctx.httpRequest.send(id)` |
| `list_environments` | `ctx.environment.list({ workspaceId })` |
| `set_environment_variable` | `env.setVariable(name, value)` |
| `get_response_history` | `ctx.httpResponse.list({ requestId })` |

## Benefits

1. **For plugin authors**: Cleaner, more discoverable API with better TypeScript support
2. **For Yaak maintainers**: Can refactor internal models without breaking plugins
3. **For users**: MCP support enables AI-assisted API development
4. **For the ecosystem**: Richer plugins become possible (e.g., test runners, documentation generators)

## Open Questions

1. **Pagination**: Should `list()` methods support pagination for large workspaces?
2. **Filtering**: Should `list()` support filtering (e.g., by name, method, tag)?
3. **Events/subscriptions**: Should plugins be able to subscribe to changes?
4. **Permissions**: Should there be scoped access (read-only vs read-write)?

## References

- [Model Context Protocol](https://modelcontextprotocol.io/)
- [MCP TypeScript SDK](https://github.com/modelcontextprotocol/typescript-sdk)
- [Yaak Plugin Documentation](https://yaak.app/docs/plugins)

# Data Flow Analysis

## Entry Points to Sinks (Call Chains)

### 1. HTTP API to Spawn (Command Injection Vector)
`apps/server/src/http.ts` (API Routing) -> `apps/server/src/open.ts:resolveEditorLaunch` (resolving args) -> `apps/server/src/open.ts:196` (launchDetached: spawn execution).

### 2. File Upload/Retrieval to File System (Path Traversal Vector)
`apps/server/src/http.ts:attachmentsRouteLayer` (receiving URL) -> `apps/server/src/http.ts:102` (string slicing `rawRelativePath`) -> `apps/server/src/pathExpansion.ts` (or similar utility) -> `FileSystem.stat` / `HttpServerResponse.file` (Disk I/O).

### 3. WebSocket Command to Git Core (Safe Command Execution)
`apps/server/src/ws.ts` (WebSocket listener) -> `apps/server/src/orchestration/Layers/ProviderCommandReactor.ts` (processing) -> `apps/server/src/git/Layers/GitManager.ts:runStackedAction` -> `apps/server/src/git/Layers/GitCore.ts:685` (spawn Git process).

### 4. Database Persistence (SQLite)
`apps/server/src/persistence/Layers/*` (ORM/Query builders) -> `apps/server/src/persistence/NodeSqliteClient.ts:133` (runStatement) -> Node.js `node:sqlite` Native Driver.

### 5. Frontend Markdown Rendering (XSS Vector)
`apps/web/src/components/ChatMarkdown.tsx` (Receiving AI text) -> `SuspenseShikiCodeBlock` -> `apps/web/src/components/ChatMarkdown.tsx:209` (dangerouslySetInnerHTML).

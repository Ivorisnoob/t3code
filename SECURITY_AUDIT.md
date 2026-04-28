# Security Audit Report

## Executive Summary
This report contains the findings of an exhaustive security audit. The codebase is heavily reliant on `Effect` and `Bun` for server-side operations, which has naturally mitigated many classic node/express related vulnerabilities. Still, several areas were flagged for either potential vulnerabilities, missing validations, or risky behaviors, such as Cross-Site Scripting (XSS) due to raw HTML rendering, path traversal vectors in attachments, command execution nuances in the `openInEditor` API (especially on Windows), and the use of weak cryptography primitives for identifiers.

## Statistics
- **Critical:** 1
- **High:** 2
- **Medium:** 2
- **Low:** 1
- **Informational:** 1

## Top Risks Ranked
### 1. SEC-002: Command Injection Vector on Windows via openInEditor (Critical)
**Description:** In `apps/server/src/open.ts`, when launching an editor on Windows, `spawn` is called with `shell: true` and the arguments are manually quoted. This allows command injection.
**Location:** `apps/server/src/open.ts` lines 196-202

### 2. SEC-001: XSS via dangerouslySetInnerHTML in ChatMarkdown (High)
**Description:** The application uses `dangerouslySetInnerHTML` to render cached and newly parsed markdown codeblocks. This can lead to XSS.
**Location:** `apps/web/src/components/ChatMarkdown.tsx` line 209 and 240

### 3. SEC-005: Path Traversal / LFI in Attachments Route (High)
**Description:** The attachments route uses string manipulation to resolve attachment paths, opening the door for directory traversal.
**Location:** `apps/server/src/http.ts` line 102

### 4. SEC-003: XSS via dangerouslySetInnerHTML in ComposerPromptEditor (Medium)
**Description:** The application uses `dangerouslySetInnerHTML` for the `SKILL_CHIP_ICON_SVG`.
**Location:** `apps/web/src/components/ComposerPromptEditor.tsx` line 265

### 5. SEC-006: Server Secret Store Race Condition / Predictability (Medium)
**Description:** The server secret store uses `Crypto.randomUUID()` for a temporary file name, leading to a potential race condition.
**Location:** `apps/server/src/auth/Layers/ServerSecretStore.ts` line 46

### 6. SEC-004: Predictable Randomness / Insecure Cryptography (Low)
**Description:** The application uses `Math.random()` to generate an HMR key for the editor.
**Location:** `apps/web/src/components/ComposerPromptEditor.tsx` line 80

### 7. SEC-007: SQLite In-Memory Database Usage in Production (Informational)
**Description:** The node SQLite client defines a `makeMemory` client which uses `:memory:`.
**Location:** `apps/server/src/persistence/NodeSqliteClient.ts` line 191

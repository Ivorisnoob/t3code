1. **[Type Safety]** - `apps/server/src/persistence/NodeSqliteClient.ts` (line 125)
Issue: Spreading `params` directly into `statement.all(...(params as any))` with an unsafe `any` cast.
Risk: If `params` is not iterable or malformed, this causes runtime crashes. TypeScript's protections are completely bypassed.
Fix: Remove the `as any` cast, assert that `params` is an array of acceptable sqlite binding types (`ReadonlyArray<string | number | boolean | null | bigint | Buffer>`), and spread that typed array.

2. **[Type Safety]** - `apps/server/src/persistence/NodeSqliteClient.ts` (line 127)
Issue: Spreading `params` directly into `statement.run(...(params as any))` with an unsafe `any` cast.
Risk: Type safety hole that could lead to unexpected behavior or SQL driver errors if params contains undefined or unsupported object types.
Fix: Validate `params` type before spreading and remove `as any`.

3. **[Type Safety]** - `apps/server/src/persistence/NodeSqliteClient.ts` (line 146)
Issue: `return statement.all(...(params as any)) as unknown as ReadonlyArray<ReadonlyArray<unknown>>`
Risk: Extremely dangerous double cast (`as any` then `as unknown as Type`) completely lies to the compiler. If the query does not return an array of arrays, downstream code will crash with `TypeError`.
Fix: Use a Zod schema or `Schema.decodeUnknownSync` (Effect) to validate the returned raw result rows into the expected structure before returning.

4. **[Type Safety]** - `apps/server/src/persistence/NodeSqliteClient.ts` (line 150)
Issue: Spreading `params` directly into `statement.run(...(params as any))` inside the `runValues` function.
Risk: Replicated type safety hole bypassing parameter type checking.
Fix: Strictly type `params` according to Node's `DatabaseSync` expectations and avoid the cast.

5. **[Type Safety]** - `apps/web/src/routeTree.gen.ts` (lines 20, 25, 30, 35)
Issue: Casting TanStack router configurations using `as any)`.
Risk: While generated, manually running typechecks might pass incorrectly. If the generator logic updates or the base schema changes, `any` hides route mismatches.
Fix: Update the `@tanstack/router-plugin` generator configuration or manually ensure generated files utilize `as const` or typed file route interfaces instead of `any`.

6. **[Type Safety]** - `apps/server/src/wsServer.test.ts` (lines 1709, 1710, 1711, 1752, 1755, 1800, 1801, 1802)
Issue: Using `vi.fn(() => Effect.void as any)` across tests.
Risk: Test mocks are lying about their return types to the system under test, meaning regressions in the Effect pipelines could silently pass the test suite but fail in production.
Fix: Correct the return types of the mocks to match the specific `Effect.Effect<R, E, A>` signature the system expects.

7. **[Type Safety]** - `apps/server/src/orchestration/projector.ts` (line 53)
Issue: `try: () => Schema.decodeUnknownSync(schema as any)(value)`
Risk: The generic `schema` parameter is cast to `any`, destroying inference on the decoded value and potentially causing a mismatch between the schema parser type and the returned type.
Fix: Refactor the projector signature to correctly propagate the generic `Schema.Schema<A, I, R>` parameters without casting.

8. **[Security / XSS]** - `apps/web/src/components/ChatMarkdown.tsx` (line 203)
Issue: Injecting cached HTML without sanitization: `dangerouslySetInnerHTML={{ __html: cachedHighlightedHtml }}`
Risk: If the cache key generation has collisions or if `cachedHighlightedHtml` can be poisoned by crafted markdown, this opens a direct Stored XSS vulnerability in the desktop/web client.
Fix: Pass the `cachedHighlightedHtml` through DOMPurify or a similar sanitizer before injection, even if it originated from Shiki.

9. **[Security / XSS]** - `apps/web/src/components/ChatMarkdown.tsx` (line 234)
Issue: Injecting generated HTML without sanitization: `dangerouslySetInnerHTML={{ __html: highlightedHtml }}`
Risk: If the LLM generates malicious payloads inside a code block that Shiki somehow parses as literal HTML nodes (or if the Shiki version has an injection CVE), XSS will occur on render.
Fix: Wrap the Shiki output with a DOM sanitizer before rendering it via `dangerouslySetInnerHTML`.

10. **[Error Handling]** - `apps/web/src/components/Sidebar.tsx` (line 377)
Issue: Catch block swallows errors when opening PR links: `void api.shell.openExternal(prUrl).catch((error) => { toastManager.add(...) })` without logging or rethrowing.
Risk: If the shell integration fails, the exact failure reason is lost, making debugging impossible if users complain links do not open.
Fix: Include `console.error("Failed to open PR url", error)` before showing the toast.

11. **[Error Handling]** - `apps/web/src/components/Sidebar.tsx` (line 442)
Issue: `await handleNewThread(projectId, { envMode }).catch(() => undefined)`
Risk: Complete silencing of thread creation failures. If sqlite fails or the workspace is locked, the user clicks the button and nothing happens silently. Complete DX failure.
Fix: Remove the `.catch(() => undefined)` and let the outer `try/catch` block handle the UI toast notification.

12. **[Error Handling]** - `apps/web/src/components/Sidebar.tsx` (line 599)
Issue: `.catch(() => undefined)` on a background process.
Risk: Background data fetching or sync operations fail silently, resulting in stale UI state without any recovery mechanism or indication to the user.
Fix: Implement a proper retry mechanism or update a shared `syncStatus` error state in the store.

13. **[Error Handling]** - `apps/web/src/components/Sidebar.tsx` (line 1031)
Issue: `.catch(() => undefined)` used on navigation operations.
Risk: Navigation fails silently if the router throws an error (e.g., dynamic chunk loading failure or route transition aborted).
Fix: Log the error and show a "Failed to navigate" toast to the user.

14. **[Error Handling]** - `apps/web/src/components/ThreadTerminalDrawer.tsx` (line 579)
Issue: `.catch(() => undefined)`
Risk: Terminal interaction commands silently fail. If a user tries to restart or clear a terminal and the IPC bridge fails, the UI becomes desynced from the server process.
Fix: Properly surface IPC layer errors to the terminal UI via a red warning banner.

15. **[Error Handling]** - `apps/web/src/components/ThreadTerminalDrawer.tsx` (line 635)
Issue: `.catch(() => undefined)`
Risk: Similar to above, critical terminal state changes failing silently.
Fix: Log the IPC error and notify the user that the terminal action failed.

16. **[Error Handling]** - `apps/web/src/components/ChatView.tsx` (line 1284)
Issue: `.catch(() => undefined)`
Risk: Chat message sending or UI state updates failing without updating the error boundary or letting the user retry.
Fix: Ensure promise rejections during message submission revert optimistic updates and set a "Failed to send" UI state.

17. **[Error Handling]** - `apps/web/src/components/ChatView.tsx` (line 1290)
Issue: `.catch(() => undefined)`
Risk: Nested promise chain silently eating exceptions.
Fix: Surface the error to the React error boundary or local state.

18. **[Error Handling]** - `apps/web/src/components/ChatView.tsx` (line 1297)
Issue: `})().catch(() => fallbackExitWrite());`
Risk: Swallowing the actual error details while blindly triggering a fallback write. If the fallback also fails or is triggered by a completely unrelated exception (like an out of memory error), the system enters an undefined state.
Fix: Log the initial error to analytics/console before calling `fallbackExitWrite`.

19. **[Error Handling]** - `apps/web/src/components/ChatView.tsx` (line 2639)
Issue: `.catch(() => undefined)` on promise rejection.
Risk: Silenced errors on critical UI async handlers (likely during rendering or stream processing).
Fix: Provide a user-facing error indication.

20. **[Error Handling]** - `apps/web/src/components/ChatView.tsx` (line 3057)
Issue: `.catch(() => undefined)`
Risk: Ignored cleanup or teardown errors leading to memory leaks or orphaned subscriptions.
Fix: Log teardown errors explicitly.

21. **[Error Handling]** - `apps/web/src/components/ChatView.tsx` (line 3063)
Issue: `.catch(() => undefined)`
Risk: Ignored errors in what appears to be a stream processing loop.
Fix: Pass errors to the stream observer's `onError` callback.

22. **[Error Handling]** - `apps/web/src/components/BranchToolbarBranchSelector.tsx` (line 152)
Issue: `await action().catch(() => undefined);`
Risk: Git operations failing silently. If a branch checkout fails due to uncommitted changes, the user sees absolutely nothing and remains on the old branch, but the UI might incorrectly assume success.
Fix: Check the exit code of the Git action, catch errors, and surface "Checkout failed: working tree not clean" to the user.

23. **[Error Handling]** - `apps/web/src/components/BranchToolbarBranchSelector.tsx` (line 153)
Issue: `await invalidateGitQueries(queryClient).catch(() => undefined);`
Risk: If query invalidation fails, the UI will display stale Git branch states permanently.
Fix: Log the failure and force a hard refresh or retry if invalidation throws.

24. **[Error Handling]** - `apps/web/src/components/BranchToolbarBranchSelector.tsx` (line 206)
Issue: `const status = await api.git.status({ cwd: branchCwd }).catch(() => null);`
Risk: Treating a failed `git status` as `null` (no changes). If the git repository is corrupted or the folder is temporarily inaccessible, the app assumes it's clean and might allow destructive actions.
Fix: Differentiate between "clean working tree" and "git status failed" by returning a `Result` or `Either` type instead of `null`.

25. **[Error Handling]** - `apps/web/src/components/DiffWorkerPoolProvider.tsx` (line 25)
Issue: `.catch(() => undefined);`
Risk: If the DiffWorkerPool fails to initialize or a diff generation job crashes the worker, the UI hangs forever waiting for a diff that will never arrive.
Fix: Provide a fallback rendering for diffs when the worker fails, or display an "Error generating diff" message inline.

26. **[Error Handling]** - `apps/web/src/components/BranchToolbar.tsx` (line 71)
Issue: `.catch(() => undefined);`
Risk: Silenced errors on toolbar actions (likely fetch/pull).
Fix: Surface git sync errors via toast notifications.

27. **[Error Handling]** - `apps/web/src/components/KeybindingsToast.browser.tsx` (line 367)
Issue: `await mounted.cleanup().catch(() => {});`
Risk: Component cleanup failures are swallowed. Can lead to dangling DOM nodes or unremoved event listeners, causing memory leaks across unmounts.
Fix: Log cleanup failures via `console.error` at a minimum.

28. **[Error Handling]** - `apps/web/src/components/GitActionsControl.tsx` (line 550)
Issue: `void promise.catch(() => undefined);`
Risk: Unhandled background git action promise. If a commit or push fails, the user is never informed.
Fix: Await the promise in an async handler or chain a catch that calls the toast notification system.

29. **[Error Handling]** - `apps/web/src/hooks/useTheme.ts` (line 53)
Issue: `void bridge.setTheme(theme).catch(() => { /* missing logic */ });`
Risk: If the IPC bridge fails to update the native theme, the web UI and desktop shell will become visually desynchronized (e.g. dark mode web app inside a light mode window frame).
Fix: Revert the local React state if the bridge update fails.

30. **[Error Handling]** - `apps/web/src/routes/__root.tsx` (line 256)
Issue: `})().catch(() => undefined);`
Risk: Root level initialization functions failing silently. Could leave the entire app in a broken, partially-loaded zombie state.
Fix: If root init fails, render a full-screen fatal error boundary rather than pretending it succeeded.

31. **[Type Safety]** - `apps/server/src/git/Layers/GitManager.ts` (line 585)
Issue: `try: () => JSON.parse(raw) as unknown,` inside `Effect.try`.
Risk: Casting `JSON.parse` directly to `unknown` is safe-ish, but if the subsequent code blindly assumes an object shape without validation, it will throw type errors.
Fix: Immediately validate the parsed output using `@effect/schema` instead of manual `as unknown` property checking.

32. **[Type Safety]** - `apps/server/src/provider/Layers/ClaudeAdapter.ts` (line 662)
Issue: `const parsed = JSON.parse(value); return parsed && typeof parsed === "object" ... ? (parsed as Record<string, unknown>) : undefined;`
Risk: Incomplete type validation. Using `typeof parsed === "object"` does not check if values inside the object match expected types, leading to potential crashes deeper in the provider adapter.
Fix: Use a strict `Schema` to decode the parsed JSON instead of manual type guards and casts.

33. **[Type Safety]** - `apps/server/src/provider/Layers/ProviderHealth.ts` (line 128)
Issue: `auth: extractAuthBoolean(JSON.parse(trimmed))`
Risk: Assuming `trimmed` is always valid JSON representing a specific shape. If the provider health endpoint returns an unexpected HTML error page, `JSON.parse` throws synchronously, triggering the outer catch, but if it returns a string, `extractAuthBoolean` might crash.
Fix: Validate the output of `JSON.parse` with a schema before passing to `extractAuthBoolean`.

34. **[Type Safety]** - `apps/server/src/provider/Layers/ProviderHealth.ts` (line 455)
Issue: `auth: extractAuthBoolean(JSON.parse(trimmed))`
Risk: Same as above for a secondary health check pathway.
Fix: Use schema validation.

35. **[Performance / UX]** - `apps/web/src/components/ChatMarkdown.tsx` (line 121)
Issue: `.catch((err) => { highlighterPromiseCache.delete(language); ... })`
Risk: If Shiki fails to load a language once, it deletes the cache and retries on the next render. If the language is permanently unsupported, this causes infinite throw/catch retry loops on every render, tanking React performance.
Fix: Cache the failure state for that language (e.g., map it to "text" permanently in the cache) to avoid re-attempting failed loads.

36. **[Code Quality / Bug]** - `apps/server/src/wsServer.ts` (line 933)
Issue: `socket.on("error", () => {});`
Risk: Completely muting socket errors on upgrade. While the comment says `// Prevent unhandled EPIPE/ECONNRESET`, muting ALL errors masks critical bugs like `EADDRINUSE` or TLS handshake failures.
Fix: Inspect the error code. Ignore `EPIPE`/`ECONNRESET`, but log everything else using the server logger.

37. **[Code Quality / Effect]** - `apps/server/src/orchestration/Layers/CheckpointReactor.ts` (line 572, 583, 592, 607, 624, 639, 720, 741)
Issue: Pervasive use of `}).pipe(Effect.catch(() => Effect.void));`
Risk: Catch-all error ignoring across the entire CheckpointReactor pipeline. If checkpointing fails, data loss occurs, but the system continues as if it succeeded.
Fix: Log the causes of checkpointing failures using `Effect.catchAllCause(cause => Effect.logError("Checkpoint failure", cause))`. Do not silently drop to `Effect.void`.

38. **[Code Quality]** - `apps/web/src/components/ChatView.browser.tsx` (line 458)
Issue: `request = JSON.parse(rawData) as WsRequestEnvelope;` inside a try/catch.
Risk: Dangerous type assertion. If the WebSocket server sends an invalid payload that is valid JSON but NOT a `WsRequestEnvelope`, the app will crash later when accessing properties that don't exist.
Fix: Use `Schema.decodeUnknownSync(WsRequestEnvelopeSchema)(JSON.parse(rawData))`.

39. **[Code Quality]** - `apps/web/src/components/KeybindingsToast.browser.tsx` (line 178)
Issue: `request = JSON.parse(rawData);` inside a try/catch, typed implicitly as `{ id: string; body: ... }`.
Risk: Unvalidated inbound IPC data.
Fix: Validate via schema.

40. **[Code Quality]** - `apps/web/src/store.ts` (line 54)
Issue: `const parsed = JSON.parse(raw) as { expandedProjectCwds?: string[]; projectOrderCwds?: string[]; };`
Risk: LocalStorage data is cast without validation. If local storage is corrupted or migrated from an older version, this will cause state initialization bugs.
Fix: Validate LocalStorage hydration with a Zod or Effect schema.

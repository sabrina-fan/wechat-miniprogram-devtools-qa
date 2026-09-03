---
name: wechat-miniprogram-devtools-qa
description: Protocol-first acceptance and debugging for WeChat Mini Programs in WeChat Developer Tools. Use whenever an agent must build or open a mini program in DevTools, inspect the simulator, verify real page behavior, automate routes and DOM geometry, validate uploads or API-backed flows, or diagnose DevTools/runtime rendering problems — not only when the user explicitly says "test it". Prefer the DevTools CLI, miniprogram-automator, local APIs, and read-only database probes; use Computer Use only as the final fallback when every reliable protocol path is unavailable.
---

# WeChat Mini Program DevTools QA

Use this skill to verify behavior in the actual WeChat Developer Tools simulator. Treat screenshots, DOM geometry, API responses, and server logs as evidence; do not claim a UI flow passed from a build success or a source-code inspection alone.

## Mandatory clean-build gate

Before every mini-program verification attempt, perform a project-scoped clean and a fresh build. This applies to protocol automation, DevTools manual checks, preview, upload checks, and real-device checks. Repeat it after any source, asset, WXML/WXSS, route, or compiled-backend change before rerunning the affected path.

- Clear DevTools `file` and `compile` caches for the exact project artifact. Do not default to `storage`, `auth`, or `all`: those can erase test state and are not a substitute for fixing the product.
- Run the repository's declared mini-program build command (detect from `package.json` scripts, usually `build:mp-weixin`) and import only the newly produced release artifact, normally `<mini-program-package>/dist/build/mp-weixin` for uni-app.
- Recompile or refresh that imported project in DevTools, then begin screenshots, DOM assertions, API-backed flows, preview, or device testing.
- When handing a manual or real-device check to the user, state which cache types were cleared, that a fresh build completed, the exact artifact path, and which checks still require the user's device.

The exact CLI sequence and a safe fallback are in [references/acceptance-protocol.md](references/acceptance-protocol.md). A stale watcher output or a visually refreshed but uncleared DevTools project is not acceptance evidence.

## Current-working-tree provenance gate

Treat the current working tree—not Git `HEAD`—as the source of every startup, build, preview, and acceptance run. The required state includes committed files, tracked-but-uncommitted changes, and relevant untracked source files. Repository scripts normally read the filesystem, but a reused process, stale `dist`, DevTools cache, or old preview QR can still expose older code.

- Inventory the current state with `git status --short`; preserve all existing changes and never stash, reset, clean, or switch them away merely to build.
- Start or reuse the complete stack from the current repository root. For every reused API, Worker, or other compiled service, verify its process cwd and prove that the running build output contains a stable marker from each affected current source change; otherwise rebuild and restart only that in-scope service.
- Clear the project-scoped DevTools `file` and `compile` caches, rebuild the imported mini-program artifact from the current working tree, and verify at least one stable compiled marker for the change under test before creating a preview or QR.
- Do not report "latest code is running" unless service cwd, compiled artifacts, and the current preview/build provenance are all verified. If a check cannot be proven, state that limit and rebuild the affected artifact.

This gate does not authorize environment, Docker, deployment, or database changes, and secrets or private configuration must never be printed.

## Operating rules

1. Resolve the repository root, the mini-program source package (often `apps/miniprogram` in a monorepo, otherwise the repo root), the release build directory, and the local API base from the project's own configuration before acting. Do not assume defaults.
2. Start or verify the complete local stack before acceptance. Prefer the repository's full-stack dev script over an isolated web process, then check its documented health/readiness endpoints.
3. Build the artifact that will actually be imported. For release acceptance run the project script that produces `dist/build/mp-weixin`; import exactly that directory. Treat `dist/dev/mp-weixin` as the watcher output, not as release-build evidence.
4. Use this order: health checks → API contract probe → read-only data probe → build/recompile → DevTools CLI/automator → screenshot and DOM assertions → Computer Use fallback. Stop at the first decisive failure only when it blocks later stages, and record the evidence.
5. Reproduce one failure at a time. Identify the broken boundary (UI state, request, compiled API, storage, or database), make the smallest in-scope fix only when asked, rebuild affected artifacts, and rerun the regression path.
6. Put timeouts around every automator operation that can hang. Never interpret a hung selector injection as a passing or failing product assertion without a timeout and a second evidence source.

## DevTools control decision

Use `references/acceptance-protocol.md` for the exact commands and endpoint patterns. Prefer `miniprogram-automator` through the DevTools CLI auto port, `mp.evaluate` with `createSelectorQuery`, and API-level probes. The stable path/navigation methods take URL strings, not `{url}` objects.

On the current macOS DevTools CLI, keep the IDE HTTP port and the automator WebSocket port separate. `127.0.0.1:9420` may answer HTTP while `miniprogram-automator` is unavailable there; start `auto` with an unused explicit `--auto-port` (for example 9422), verify the child listener with `lsof`, and connect to `ws://localhost:<auto-port>` or `ws://[::1]:<auto-port>`. If the route and selector geometry are valid but the screenshot is uniformly white, inspect always-visible native `page-container` layers before blaming API data.

Native layers are independent rendering surfaces. An always-present empty `page-container` can cover a Vue page even when `overlay=false` and the Vue nodes have valid geometry; preserve the leave guard but make the surface visually inert (transparent, minimal dimensions, and zero opacity) and verify that `beforeleave` still intercepts the intended back gesture. Treat `video`, map, canvas, and other native components the same way: stop or unmount them on hide/unload when they can outlive the page. For bundled media, use the package asset directly instead of copying one file per component instance into `USER_DATA_PATH`; retain `saveFile` only for real business files.

Use Computer Use only when all of the following have failed or are unavailable:

- DevTools CLI/auto port;
- `miniprogram-automator` connection or protocol calls;
- API-level equivalent actions;
- read-only DOM/state inspection;
- a user gesture is intrinsically required, such as a swipe, picker, native file chooser, or opening a page blocked by a native overlay.

When falling back, invoke the host environment's computer-control skill, operate only the visible WeChat Developer Tools window, and capture a simulator screenshot after each meaningful action. Check Accessibility and Screen Recording permissions first. If permissions or the app window make Computer Use unavailable, report the exact blocker and leave the UI unverified; never infer success.

## Creation-page lifecycle and async-generation acceptance

For chat, creation, or capture pages that batch content into a background generation task, verify the lifecycle contract as one flow rather than testing only the visible header button:

- The page header's back action, Android/system `onBackPress`, and the native `page-container @beforeleave` guard must converge on the same exit handler. iOS swipe-back or another lifecycle path that bypasses `onBackPress` needs an `onHide`/`onUnload` detached fallback with duplicate suppression.
- Do not start the background generation after every individual message when the product contract is batch submission. At a session boundary—exit, hide/unload, new conversation, or mode switch—finish in-flight capture/writes/sends, then recompute whether committed content exists.
- A busy flag by itself is not committed content. If a write is still in flight, wait for its bounded completion and materialize the result before deciding whether to submit. A failed send must not create a generation task solely because it left a failure bubble.
- Mark local processing and submit with a stable idempotency key. The API should acknowledge/enqueue the generation request without waiting for the model to finish; navigate away immediately, while the home/status flow polls, retries with backoff, and resumes until the task reaches a terminal state. The next chat round must not wait for the previous background generation.
- Repeated exits, lifecycle callbacks, and retries must be safe: one active generation per work unit, no duplicate messages or submissions, and no lost content when the app is hidden or unloaded.
- With no newly committed content, do not call the model; verify the insufficient-content state and the intended navigation behavior instead.
- For perceived AI delay, measure the assistant-reply request separately from generation-enqueue acknowledgement and final result application. Run at least the first, second, and third rounds; do not count background generation as the next chat round's thinking time. If the provider supports a reasoning control, verify the effective outbound setting or server-side model telemetry instead of trusting the UI label.
- If a shared native guard, video, or navigation component is changed, rerun every affected page that mounts it; a fix proven on one page does not automatically prove another creation page.

When this contract is in scope, report separately: time to acknowledge/navigation, time until the generated result is applied, and any pending write or retry state.

## Evidence and safety

- Use a mock/test identity and loopback services. Keep test writes isolated and clean them up when practical; never use real payment or production data.
- Never print `.env*`, cookies, access tokens, admin passwords, upload URLs, database passwords, or other secrets. Redact headers and identifiers in saved logs and reports.
- Distinguish DevTools warnings from Errors/Problems. A successful build is not UI acceptance; a screenshot without the matching API/data evidence is not a backend acceptance.
- If the API source changed, verify that the running process uses the rebuilt API output (for example `apps/api/dist`). A stale manually started process can hold the API port and hide the fix; inspect process cwd before stopping anything and stop only processes belonging to this repository.
- For image flows, prove the full chain: ticket → upload response → persisted material → content endpoint → page data/reader marker → rendered image. "Uploaded" alone is insufficient.
- Do not modify Docker, deployment, environment, database schema, or unrelated files during acceptance unless the user explicitly expands scope.

## Reporting

Report the exact artifact imported, stack health, route/page, test identity class (not its secret), actions, screenshot paths, DOM/API/DB evidence, console/server errors, and the final state. Separate protocol-verified, Computer-Use-verified, and unverified steps. If no new reproducible bug remains, say so and stop instead of inventing UI actions.

Read [references/acceptance-protocol.md](references/acceptance-protocol.md) before running the detailed workflow, or whenever a DevTools/API boundary fails.

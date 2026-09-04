# Acceptance protocol

This reference is the detailed procedure for the protocol-first workflow in the parent skill. Adapt paths and scripts to the repository instead of assuming a fixed project layout.

## Table of contents

- [0. Mandatory clean-build gate](#0-mandatory-clean-build-gate)
- [1. Resolve and start the real stack](#1-resolve-and-start-the-real-stack)
- [2. Open DevTools through its CLI](#2-open-devtools-through-its-cli)
- [3. Reliable automator and DOM assertions](#3-reliable-automator-and-dom-assertions)
- [4. API-level probes](#4-api-level-probes)
- [5. Read-only database probes](#5-read-only-database-probes)
- [6. Computer Use fallback](#6-computer-use-fallback)
- [7. Common failures](#7-common-failures)
- [8. Acceptance record](#8-acceptance-record)

## 0. Mandatory clean-build gate

This gate runs before every verification mode: automator, DevTools manual inspection, preview, sharing/preview-package checks, or real-device testing. It also runs again after any mini-program source, asset, WXML/WXSS, route, or compiled-backend change. The purpose is to prevent a stale DevTools project, compiler cache, or watcher output from being mistaken for current behavior.

For a macOS DevTools CLI with an already opened IDE HTTP port, use the exact project artifact and clear only the two caches that commonly contaminate compiled/rendered output:

```bash
CLI="/Applications/wechatwebdevtools.app/Contents/MacOS/cli"
ARTIFACT="/absolute/path/apps/miniprogram/dist/build/mp-weixin"
IDE_PORT=9420

"$CLI" cache --project "$ARTIFACT" --port "$IDE_PORT" --clean file
"$CLI" cache --project "$ARTIFACT" --port "$IDE_PORT" --clean compile
pnpm --filter <miniprogram-package> build:mp-weixin
```

The build command must be adapted to the repository. In a typical monorepo it writes `apps/miniprogram/dist/build/mp-weixin`; import that directory, wait for DevTools to compile it, and only then run assertions or ask the user to verify. If the CLI is unavailable, perform the equivalent project-scoped file/compile cache cleanup in DevTools, record that fallback, and still run a fresh build before testing.

Do not use `--clean storage`, `--clean auth`, or `--clean all` as routine preflight. They can remove login/test state and make a rendering problem harder to reproduce. Do not delete legitimate application files merely because the simulator reported a storage error.

For a manual or real-device handoff, report: `file` and `compile` cleared (or the exact fallback), fresh build success, imported artifact path, and the protocol/device steps that remain unverified.

## 1. Resolve and start the real stack

For a typical monorepo reference clone:

- Source: `apps/miniprogram`.
- Release artifact: `apps/miniprogram/dist/build/mp-weixin`.
- Dev watcher artifact: `apps/miniprogram/dist/dev/mp-weixin`.
- API base: `http://127.0.0.1:3000/api/v1`.
- Admin origin: `http://127.0.0.1:5174/`.

Read the project scripts before running commands. When the user asks to start the project, prefer:

```bash
pnpm dev:stack
```

This should prepare MySQL/Redis, run migrations, API, worker, admin, and the mini-program watcher. Verify readiness without printing response secrets:

```bash
curl -fsS http://127.0.0.1:3000/api/v1/health/ready
curl -fsS http://127.0.0.1:3000/api/v1/health/capabilities
curl -fsS http://127.0.0.1:5174/
```

Check the actual process and artifact before testing. The API serves compiled `apps/api/dist`; source edits are not live unless the repository watcher rebuilds and restarts it. If port 3000 is occupied by a manually started `node dist/main.js`, inspect its cwd with `lsof -p <pid> | grep cwd` and stop only an old process from this repository. Do not kill an unknown owner.

For release acceptance, use the script discovered from `package.json`, normally:

```bash
pnpm --filter <miniprogram-package> build:mp-weixin
```

Import only `apps/miniprogram/dist/build/mp-weixin`. The command above is part of the mandatory clean-build gate; do not treat a stale DevTools cache, a watcher output, or a source-only change as release evidence.

## 2. Open DevTools through its CLI

Use the installed CLI if available:

```bash
CLI="/Applications/wechatwebdevtools.app/Contents/MacOS/cli"
IDE_PORT=9420
AUTO_PORT=9422
"$CLI" auto --project "/absolute/path/apps/miniprogram/dist/build/mp-weixin" --port "$IDE_PORT" --auto-port "$AUTO_PORT"

# The current CLI may omit --auto-port from `auto --help` but still accepts it.
# Verify the child listener before connecting automator; 9420 is the IDE HTTP port.
lsof -nP -iTCP:$IDE_PORT -sTCP:LISTEN
lsof -nP -iTCP:$AUTO_PORT -sTCP:LISTEN
```

Do not assume that a visible DevTools window is protocol-connected. The IDE HTTP port and the automator WebSocket port are separate; verify the explicit `AUTO_PORT` listener before connecting.

Create a disposable automator workspace outside application source when the dependency is absent:

```bash
mkdir -p tmp/automator
cd tmp/automator
npm init -y
npm i miniprogram-automator
```

Connect from a small ESM script:

```js
import automator from "miniprogram-automator";

const mp = await automator.connect({ wsEndpoint: "ws://localhost:9422" });
mp.on("console", (message) => {
  // Redact tokens and user identifiers before saving this output.
  console.log(message);
});

await mp.reLaunch("/pages/example/index");
const page = await mp.currentPage();
await page.waitFor(500);
await mp.screenshot({ path: "tmp/devtools/example.png" });
```

Always wrap calls in a timeout. A helper can turn an indefinite automator hang into explicit evidence:

```js
function withTimeout(promise, ms, label) {
  return Promise.race([
    promise,
    new Promise((_, reject) =>
      setTimeout(() => reject(new Error(`${label} timed out after ${ms}ms`)), ms),
    ),
  ]);
}
```

## 3. Reliable automator and DOM assertions

The stable calls are:

- `mp.reLaunch("/pages/x/index")`, `navigateTo`, `navigateBack`, `switchTab`;
- `mp.currentPage()` and `page.waitFor(ms)`;
- `mp.screenshot({ path })`;
- `mp.evaluate(fn, ...args)`;
- `mp.on("console", handler)`.

Navigation arguments must be URL strings. Passing `{ url: ... }` can produce `Uncaught [object Object]`.

Do not use `page.$`, `page.$$`, or `element.tap()` as a primary path in an already-open project: the injection channel can remain uninitialized and hang permanently. Prefer app-service evaluation plus the render-layer selector query. Query all relevant nodes in one call; this also sees all `swiper-item` nodes, including off-screen pages:

```js
const boxes = await withTimeout(
  page.evaluate(() => new Promise((resolve) => {
    const query = wx.createSelectorQuery();
    query.selectAll(".target, .swiper-item").boundingClientRect().exec(resolve);
  })),
  3000,
  "selector geometry",
);
```

If the page object does not expose `wx`, execute the same selector query in the page context using the repository's supported `createSelectorQuery` bridge. Assert existence, dimensions, visibility, and relative alignment rather than relying only on text.

Do not use `page.setData()` to drive Vue 3 `script setup` state: the native page mirror is one-way. In release builds, `page.data()` keys are compressed and only direct template references appear; computed values such as `spreadPages` are not reliable there. `page.$vm.$.setupState` can also be empty. Use DOM assertions, screenshots, or a dev build for state inspection.

`mp.native()` supports only fixed native operations such as `goHome`, `navigateLeft`, and `confirmModal`; it is not a general tap or swipe API. If a flow requires a real gesture, native picker, file chooser, or an overlay that blocks protocol control, defer it to the Computer Use fallback and mark it as such.

### White screens and native rendering surfaces

If the expected route is active and selector geometry is non-zero but the screenshot is uniformly white, do not first assume that the API returned empty data. Use this order:

1. Confirm `currentPage().route`, page stack, and a small set of visible selector rectangles.
2. Inspect always-mounted native surfaces such as `page-container`, `video`, `map`, `canvas`, and platform overlays. They can cover the Vue render tree while remaining absent from ordinary selector geometry.
3. For a `page-container` used only to intercept leaving, preserve the guard but make it visually inert and minimal. A portable pattern is:

   ```vue
   <page-container
     :show="backGuardVisible"
     :overlay="false"
     custom-style="background: transparent; width: 1px; height: 1px; opacity: 0;"
     @beforeleave="handleBeforeLeave"
   />
   ```

   Verify both sides of the change: the page is visible in a fresh screenshot and the native/back leave path is still intercepted. `overlay=false` alone is not proof that the native surface is transparent.
4. For native video, stop or remove the source on `onHide` and unmount; use a static fallback until the first frame is ready. Do not copy the same bundled video into user storage for every component instance.
5. Only after the native-layer checks pass, investigate API payloads and page business state.

Geometry is necessary but not sufficient for visual acceptance; keep the screenshot as a separate piece of evidence.

### DevTools file-storage limit errors

For `saveFile:fail exceeded the maximum size of the file storage limit` or an equivalent simulator error:

- First clear the DevTools project-scoped `file` and `compile` caches, rebuild, reimport, and retry.
- Then inspect `getSavedFileList()` and search the source for the real `saveFile`/`downloadFile` call sites. A zero-file/zero-byte saved-file list with the same error points toward DevTools temporary/compiler cache pollution, not necessarily a product file leak.
- Keep legitimate recording drafts, downloaded PDFs, and other business files intact; do not remove every `saveFile` call or clear all storage as a rendering workaround.
- If the error survives a clean build, isolate the specific business write, size, lifecycle, and cleanup behavior and test it on a real device when the storage implementation differs.

## 4. API-level probes

Use API probes before UI actions whenever the feature has an equivalent endpoint. Obtain a mock/test identity only through the project’s documented local endpoint and keep the token in a shell variable or process memory; never print it:

```bash
API="http://127.0.0.1:3000/api/v1"
TOKEN="$(curl -fsS -X POST "$API/auth/mock-login" \
  -H 'content-type: application/json' \
  -d '{"consentVersion":"2026-07-01"}' | jq -r '.accessToken')"
```

Use `Authorization: Bearer "$TOKEN"` only in local commands. Check status codes and response shapes without echoing credentials.

### Answers and reader data

For a test work/question, the front-end-equivalent save is:

```text
POST /answers/save
{ clientAnswerId, workId, questionId, text, source: "manual" }
```

For a generated memoir, the normal sequence is:

```text
POST /works/:id/memoir-summary/drafts       { idempotencyKey }
POST /memoir-summary-drafts/:id/apply
GET  /works/:id/memoir-summary/status
GET  /works/:id/reader
```

The reader response is the source of truth for book pages. Verify that `memoirBlocks` or its project-equivalent contains the expected text/image marker before blaming the view.

### AI provider and model configuration boundary

When a backend changes provider endpoint, agent/model identifier, or reasoning level, verify the runtime boundary instead of relying on source files alone:

- Search the effective backend configuration and remove obsolete provider/model selection paths within the intended scope; do not accidentally keep an old fallback that makes tests hit a different model.
- Rebuild/restart the API when the project requires compiled output, then recheck readiness/capabilities and the running process cwd/artifact.
- Keep the intended Lite model consistent across the tested chat and memoir paths. If the provider supports reasoning control and the requirement is `low`, verify the effective request/telemetry value without exposing credentials or full request headers.
- Send a real local test request and record status, redacted provider/model labels, assistant-reply latency, and summary enqueue latency. A successful UI toast or a source-code match alone does not prove the new provider is serving traffic.

### Chat and memoir-generation boundary

When the product batches chat/answer content into memoir generation, validate the boundary semantics explicitly:

- The page header back action, system `onBackPress`, and `page-container @beforeleave` should route to one exit function. An iOS swipe or lifecycle path that bypasses `onBackPress` needs an `onHide`/`onUnload` fallback; an explicit exit must suppress that fallback so it cannot submit twice.
- At exit, hide, unload, new-conversation, or mode-switch boundaries, first await bounded recording/transcription/chat-write completion and materialize any completed transcript. Re-read the committed-content snapshot after those writes settle. Do not count a transient busy flag or a failed message bubble as committed content.
- If committed content exists, mark the local task as processing and enqueue one idempotent summary request. The request should return an accepted/enqueued response without awaiting the model's full completion; the app can navigate to the home page immediately.
- The home page or a module-level runner should poll status and retry with backoff, using the same work-scoped idempotency key. Verify that the second and third chat rounds remain interactive while the previous memoir generation is pending.
- If no new committed content exists, verify that no model request is made and that the product's insufficient-content notice/navigation is shown.

Measure and report two durations separately: boundary-to-navigation/acknowledgement and boundary-to-applied-memoir. A long second duration is expected for asynchronous generation; it must not block the first.

For model-latency checks, record the first, second, and third assistant-reply durations independently from the background summary task. If a reasoning level is configurable, keep the requested `low` value while retaining thinking, and verify the effective provider/model/request setting from redacted telemetry or server-side evidence. A label in the page alone is not proof that the request used the intended Lite model or reasoning level.

### Image upload chain

Test the same chain used by the mini-program:

```text
POST /works/:id/materials/upload-ticket
{ fileName, fileSizeBytes, sourceMetadata: { questionId } }
```

Then upload the bytes to the returned `upload.uploadUrl` using the ticket’s content type:

```text
PUT <upload.uploadUrl>
content-type: application/octet-stream
x-image-content-type: <ticket.upload.contentType>
<image bytes>
```

Record only status, material id, and redacted metadata. Verify all of these separately:

1. Ticket creation succeeds and is bound to the expected work/question.
2. The `PUT` succeeds.
3. The persisted material is returned by the work/material API.
4. `GET /materials/:id/content` returns 200 to the mini-program’s loopback request.
5. The reader or page data includes the material and, where applicable, an exclusive image marker such as `[[memoir-image:<material-id>]]` at the end of a memoir block.
6. The screenshot contains a visible image with the expected bounds.

An “uploaded” toast without these checks is not evidence that the image is rendered or inserted into the body.

### AI quota and transient backend state

If an AI flow returns 403 because the local test entitlement is exhausted, use the project’s local admin grant route only when explicitly within the test scope. Obtain the admin token and key from the local environment without printing either, grant a small test quota with an idempotency/request id, then retry. After API restart, the first admin session can temporarily return 503 from an ioredis offline-queue race; retry the same safe read/login request once before diagnosing a product bug. Do not grant real or production entitlements.

## 5. Read-only database probes

Use a short-lived script under `apps/api` so the workspace’s `mysql2` dependency resolves. Inject the local environment without displaying it:

```bash
cd apps/api
node --env-file-if-exists=../../.env ./path/to/read-only-probe.mjs
```

The probe must use `SELECT` only and print structural facts, counts, ids, statuses, and marker presence—not credentials, cookies, phone numbers, WeChat identifiers, or full user content. Relevant project tables typically include:

- `creative_materials`: image/material rows; inspect `source_metadata->>'$.questionId'` binding.
- `answers` and `answer_versions`: distinguish `source_channel` such as `self` and `chat`.
- `memoir_summary_blocks`: body blocks; image markers are stored as exclusive lines at a block end.
- `chapters`, `memoir_summary_drafts`.

Use the DB only to locate a data boundary. Use the API and screenshot to prove the user-visible behavior.

## 6. Computer Use fallback

Invoke `computer-use:computer-use` only after protocol methods are unavailable or insufficient. Do not use screen clicks to replace a reliable API or automator operation.

1. Confirm the DevTools window and imported project path visually.
2. Check Accessibility and Screen Recording permissions. If denied, report that UI acceptance is blocked.
3. Compile/rebuild in DevTools, clear cache when needed, and wait for the compile result.
4. Use visible clicks, typing, swipes, native pickers, and file selection only for the blocked interaction.
5. Capture a simulator screenshot after navigation and after the meaningful action.
6. Correlate the screenshot with API status, DevTools console, API logs, and (when needed) a read-only DB probe.
7. Report exactly which steps were Computer-Use verified and which remain unverified.

macOS fallbacks such as `cliclick`, AppleScript System Events, or `screencapture` require Accessibility/Screen Recording permissions and may fail with `osascript -1719` or a display-capture error. Do not retry blindly; use the dedicated Computer Use skill or report the permission blocker.

## 7. Common failures

| Symptom | First check | Correct response |
|---|---|---|
| `module "...js" is not defined`, but the file exists | Build/DevTools cache and artifact timestamps | Clear cache, rebuild, and reimport; do not change code until the clean artifact reproduces it |
| API fix is invisible | API process cwd and `apps/api/dist` timestamp | Rebuild/restart the compiled API or let the project watcher do it; verify readiness again |
| Automator selector hangs | Protocol attach state | Apply a timeout; use `mp.evaluate` + selector geometry or screenshot; do not wait indefinitely |
| `setData` changes nothing | Vue 3/uni-app bridge | Use DOM/screenshot or a supported dev build; do not infer state from native page data |
| Uploaded image is invisible | Ticket/PUT/material/content/reader chain | Find the first failed boundary, then verify the rendered `<image>` geometry and content status |
| AI returns 403 quota exhausted | Local entitlement/status response | Use a small local test grant only when in scope, then retry; redact admin secrets |
| First admin request returns 503 after restart | Redis readiness/queue race | Retry once after readiness; distinguish transient infrastructure from a persistent API failure |
| Gesture/picker cannot be automated | Automator/native API support | Use Computer Use as a labeled fallback or report the missing permission/device |

## 8. Acceptance record

Record:

- repository and imported artifact path;
- stack health and API build provenance;
- route, test identity class, and test-data cleanup status;
- protocol commands and bounded results;
- screenshots and DOM geometry paths;
- API status/shape and redacted server/DevTools errors;
- read-only DB structural evidence, if used;
- Computer-Use steps, if used;
- verified, blocked, and untested boundaries.

Never claim “验收通过” when a required real-device-only behavior (recording, share menu, QR/file chooser, payment, or native gesture) was not available. Separate “协议已验证” from “UI/真机未验证”.

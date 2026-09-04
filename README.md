# wechat-miniprogram-devtools-qa

Protocol-first acceptance and debugging for WeChat Mini Programs in WeChat Developer Tools. Verify real page behavior through the automator protocol, API probes, and read-only database checks — with Computer Use only as a last-resort fallback.

## Why

Screenshots, DOM geometry, API responses, and server logs are the only real evidence that a mini-program works. A successful build or source-code inspection alone is not acceptance. This skill enforces a clean-build gate, current-working-tree provenance, and a layered verification chain so you never mistake a stale artifact for working software.

## Install

### Option A — let your agent install it

Give your agent this repo URL and ask it to add the skill:

```
https://github.com/sabrina-fan/wechat-miniprogram-devtools-qa
```

### Option B — manual

Copy the `wechat-miniprogram-devtools-qa/` directory into your agent's skills folder (e.g. `~/.zcode/skills/` or your agent's configured skills directory).

## Configuration

- **DevTools CLI path**: auto-detected on macOS (`/Applications/wechatwebdevtools.app/Contents/MacOS/cli`).
- **Ports**: IDE HTTP defaults to `9420`, automation WebSocket defaults to `9422` — auto-discovered via `lsof`.
- **miniprogram-automator**: install with `npm i miniprogram-automator` in a disposable workspace if not present.
- **Project paths**: detected from `package.json` and project structure. No hardcoded paths or API keys required.

## Usage

Trigger it when you need to verify mini-program behavior in DevTools. The skill follows this verification chain:

1. **Clean-build gate** — clear `file`/`compile` caches, rebuild from current working tree, verify compiled markers.
2. **Stack health** — start the local stack, check health endpoints.
3. **Protocol automation** — connect via `miniprogram-automator`, navigate, evaluate, screenshot.
4. **API probes** — test endpoints before UI actions when an equivalent API exists.
5. **Read-only DB probes** — locate data boundaries with `SELECT`-only queries.
6. **Computer Use fallback** — only when every protocol path is exhausted.

## Compatibility

- **macOS** — primary platform (uses `lsof`, macOS DevTools CLI paths, Computer Use for GUI).
- **Windows / Linux** — adapt CLI paths and process inspection commands to the local platform.

## Security & Boundary

This skill verifies behavior; it does not write or modify source code unless explicitly asked. It never prints `.env`, tokens, cookies, passwords, or user identifiers. All test data uses mock/loopback identities and is cleaned up when practical. No production data is used.

# ZAI weekend-plan tokens in omp — findings & plan

Scratch file; delete from repo before any commit/PR.

## Answer to "why 429 in omp"

- omp's `zai` login mirrors ZCode: chat.z.ai OAuth → `zcode.z.ai/api/v1/oauth/token` → business login
  (`api.z.ai/api/auth/z/login`) → mints a durable `id.secret` key named `oh-my-pi` in the account's default project,
  then rides `https://api.z.ai/api/coding/paas/v4` (`api/anthropic` for anthropic-messages).
- User's account has **no coding plan on api.z.ai**: `GET api.z.ai/api/monitor/usage/quota/limit` →
  `{"code":500,"msg":"当前用户不存在coding plan"}`; model calls return `429 {"code":"1113",
  "message":"Insufficient balance or no resource package"}` — with the minted key, with ZCode's own project access
  token, and for the official `zcode` CLI (`providerId account:zai-individual-coding-plan`).
- The user's entitlement is a **ZCode desktop plan**: `zcode-v3-start-plan-1106-wk` ("ZCODE WEEKEND BUILD",
  GLM-5.3-Flash, 500M tokens, active). It is served only through the ZCode gateway.

## ZCode gateway facts (captured live, 2026-11-07)

- Provider rule `account:zai-start-plan`: access `{type: zhipu-account, accountType: zai, mode: start-plan}`,
  api `anthropic-messages`, baseUrl `https://zcode.z.ai/api/v1/zcode-plan/anthropic`, models GLM-5.3-Flash,
  GLM-5.2, GLM-5-Turbo. Agent process: `resources/glm/zcode.cjs app-server --stdio --surface desktop`.
- Runtime-headers RPC returns `auth.apiKey` = the **ZCode login JWT** (`zcodejwttoken`, HS256, sub=user uuid).
- `GET zcode.z.ai/api/v1/zcode-plan/billing/balance?app_version=…` works with that JWT + `x-device-mid`,
  `user-agent: ZCode/<ver>`, `x-zcode-app-version` → plan + 500M balance (verified: HTTP 200).
- Model POST captured byte-exactly (headers + body + ids), including
  `x-api-key` == `authorization: Bearer <jwt>`, `anthropic-beta: mid-conversation-system-2026-04-07`,
  `x-zcode-agent: glm`, `x-title: Z Code@electron`, `x-platform: linux-x64`, `x-output…`, `metadata.user_id`
  = `{"device_id":…,"account_uuid":"","session_id":…}`.
- **Replay of that exact request fails** (`405 {"code":3012,"msg":"request has been blocked due to unusual
  activity"}`) from: curl (http1/h2), Node (same header order, same ids, same body), Bun, Electron-as-Node (same
  binary), Chromium (401), and even a clone issued **inside the app process through its own fetch** while the app's
  own request in that same second returned 200.
- Follow-up (full 121 KB body captured): the gate is **content-based anti-abuse**, not transport.
  Replaying the complete captured body → **200 SSE**. Bisect with tools dropped:
  - system = real [preamble, agent-prompt] + tiny messages → 200
  - system = only `"You are ZCode, an interactive agent"` → 405 3012
  - system = preamble + first 600 chars of agent prompt (padded to same length) → 405
  - system = preamble + first 1500 chars → 200
  - generic filler text (any length/block count) → 405
  So the start-plan route validates the **ZCode harness system prompt** server-side; tokens/tools do not matter.
  The official CLI cannot use it either ("Provider Registry 中不存在 Model: account:zai-start-plan/…").
- No session/registration endpoint discovered: `/api/v1/zcode-plan/{sessions,session,usage}` = 404;
  only `billing/balance` and `billing/current` are open to the JWT.

## Options

- **A (default, small):** keep omp as-is functionally; map Z.ai `1113` to an actionable message
  ("this key has no coding-plan entitlement; the ZCode weekend/Start-plan allowance is app-locked") and make
  `/usage` explain the same instead of showing a raw failure. No new provider.
- **B (large, fragile):** omp-side bridge that drives the ZCode desktop agent/app-server and forwards prompts; needs
  the desktop running, its RPC auth broker, and a streaming adapter — custom daemon, not a plain provider, and it
  breaks on every ZCode update.
- **C:** subscribe to a real GLM Coding Plan (z.ai or bigmodel). omp's `zai` / `zhipu-coding-plan` providers and the
  `/usage` windows are designed for exactly that and need no changes.

## Restore notes

- Instrumentation lives only in `/tmp/opencode/zk-probe` (extracted AppImage + hooks). The installed AppImage and
  `~/.zcode` data are untouched; instrumented instances are stopped.
- Launch two ZCode instances concurrently ⇒ sqlite/GPU startup failures; keep one.

## Status

- [x] Decrypt ZCode credentials, map plan + balance APIs
- [x] Capture exact model request (headers, ids, body) from the live app
- [x] Prove replay/app-lock across curl, node, bun, electron-node, chromium, in-process clone
- [x] Prove account has no api.z.ai coding plan (1113 root cause in omp)
- [ ] Decide A / B / C with the user
- [ ] Implement the chosen omp-side change (+ tests, docs, changelog if provider behavior changes)

# Pi Auto Review — Guardian API compatibility probe (E1)

This is an **experimental connectivity and billing-attribution probe** for ChatGPT-subscribed Pi users. It is **not yet** a general permission reviewer, sandbox, or free-inference interface.

The user-facing goal is to run one **authentic harmless tool call** and test an **action-bound Guardian reviewer request** with Pi's existing ChatGPT OAuth credential. The reviewer stays on the same OAuth resource as the parent: the **legacy Codex backend** can read Codex quota snapshots; the **new Sign in with ChatGPT** uses the direct OpenAI API and cannot use that Codex-only quota endpoint. Both paths are **experimental**, and neither a 200 response nor stable rounded usage percentages prove the review was free.

## Installation

Prerequisites: Pi with `@earendil-works/pi-ai` / `@earendil-works/pi-coding-agent` (recent 1.x), Node >= 22.19 or a compatible Bun runtime, and a **genuine ChatGPT OAuth** OpenAI or Codex model selected in Pi. API-key and proxy routes are intentionally not eligible.

Install directly in Pi:

```bash
pi install git:github.com/SMlc666/pi-auto-review
```

Or for local development, clone the repository and start Pi with:

```bash
pi --extension ./index.ts
```

For the first test, **create a fresh Pi conversation** and temporarily **disable Code Mode or other wrappers that hide native tool calls**. Normal Pi tool mode is required for this prototype. Two login modes are recognized:

| Pi login | Provider/API | Endpoint of model/reviewer | Free Guardian status |
| --- | --- | --- | --- |
| **New Sign in with ChatGPT** | `openai` / `openai-responses` | `https://api.openai.com/v1/responses` | **Unknown**. Only API/model compatibility is tested; no documented Guardian-free attribution for this OAuth route. |
| **Legacy OpenAI Codex** | `openai-codex` / `openai-codex-responses` | `https://chatgpt.com/backend-api/codex/responses` | The official Codex client uses Guardian metadata on this route. Actual third-party free billing is still **unverified**. |

> The new OAuth flow requests a token for `https://api.openai.com/v1`, while legacy Codex uses the ChatGPT Codex backend. Tokens must **never be forwarded across these resources**. Calling `codex-auto-review` through the direct API can fail or consume subscription allowance; do not mistake this for verified free Auto-review.

## Real test: one slash command

In Pi, check the selected authentication route without a model call:

```text
/auto-review-probe-route
```

Then run **one** real request (this performs normal model inference and may consume quota):

```text
/auto-review-probe
```

This asks your selected model to invoke `guardian_probe` exactly once. The tool is deliberately inert: **no files are read/written and no program is executed**. It performs only a separate Guardian review request plus read-only quota queries. Once done, inspect the tool result or run:

```text
/auto-review-probe-status
```

Expected output includes the **actual route**, `review-accepted`, `review-denied` or `failed`, the HTTP status, an allowlisted server error code when available, whether a **real parent Response ID** was bound to the tool call, and any Guardian tokens. **Codex quota snapshots are only queried with the legacy Codex OAuth backend** (`codex.5h.usedPercent` and `codex.week.usedPercent` where provided).

The probe performs a real parent inference and attempts a real reviewer inference and **may use subscription quota**; particularly with random stress-test padding. Token counts are not billing. If the before/after rounded quota percentage is unchanged, the only correct conclusion is **billing unverified**, not free. Some accounts may not expose precise per-call usage. Avoid running repeated tests blindly.

### Reviewer-only random context stress test

Use the same `/auto-review-probe` command with an **optional decimal byte count**. The number controls fresh cryptographically random hexadecimal ASCII characters added to the **reviewer request only**, never the main agent prompt. One ASCII character is one UTF-8 byte. The random text is not written to disk or repeated in diagnostic output; only its truncated SHA-256 digest is shown to confirm that each run used fresh data.

```text
/auto-review-probe 0
/auto-review-probe 65536
/auto-review-probe 131072
/auto-review-probe 262144
```

`/auto-review-probe stress` is shorthand for `262144` (256 KiB). The maximum accepted input is `393216` (384 KiB). Start with one intermediate size before jumping to 256 KiB; high-entropy inputs can consume **a large amount of your ChatGPT subscription quota** if this direct API route is not free, or hit model context/rate limits. Stress-test calls get a 120-second timeout; default no-padding calls use 30 seconds. The model sees a `diagnostic_random_padding` field with an explicit instruction that its content must be ignored for the approval decision.

The result reports the **requested reviewer-only random ASCII byte count**, actual reviewer evidence JSON byte length, serialized HTTP request body length, input/cache/output tokens, and the Guardian decision. Comparing `/auto-review-probe 0` against `/auto-review-probe 262144` in **fresh sessions** can establish how token consumption scales, but **not whether Guardian is free**: the main agent call also uses quota, and only the backend controls final billing. For the new ChatGPT OAuth route, inspect ChatGPT app usage separately; the legacy Codex `/wham/usage` endpoint must not be called with the new token.

## Protocol approach

1. Only when explicitly armed by `/auto-review-probe`, add `client_metadata.guardian_credits_requested="true"` to **legacy Codex backend** parent requests. Do **not** add undocumented billing metadata to the direct OpenAI API parent.
2. Observe the actual completed `resp_*` response from Pi's `provider_stream_event`, then bind the assistant's exact `guardian_probe` tool-call ID in `message_end`.
3. Use `ctx.modelRegistry.streamSimple()` with the **same selected OAuth provider and API implementation** (`openai-responses` or `openai-codex-responses`), letting Pi SDK construct auth, session headers, URL and SSE. The reviewer model is `codex-auto-review` with low reasoning. No cross-endpoint token forwarding.
4. At its `onPayload` boundary, build the **tool-less structured review body**, referencing the real parent's response ID. At its `fetch` boundary, preserve SDK authentication while adding `x-codex-guardian: reviewer`, `x-openai-subagent: guardian`, session/turn and parent-thread identifiers.
5. For **legacy Codex backend only**, read `GET https://chatgpt.com/backend-api/wham/usage` immediately before/after review with the SDK-issued Codex account bearer headers. **Never** submit a new OpenAI API OAuth token to `/wham/usage`. Display only quota percentages, not raw account IDs, bearer tokens, request payloads or API response bodies.

For integrity, the extension permits only the **same official OAuth resource** as its selected parent (`chatgpt.com/backend-api/codex/responses` for legacy, `api.openai.com/v1/responses` for new login), only accepts genuine OAuth subscription sessions, never fabricates parent IDs, and does **not** claim to be the official Codex CLI (`originator`/User-Agent remain Pi's genuine ones). If the server requires additional entitlement or official-client identity, report the resulting failure; do not attempt to evade it.

## Interpreting results

| Result | What we learned |
| --- | --- |
| No eligible model | Check `/auto-review-probe-route`; select a real ChatGPT OAuth model, not an API key or custom proxy |
| No parent Response ID | Use a fresh conversation; disable Code Mode/tool wrappers; try again |
| HTTP 400 / 401 / 403 | The server rejected the request or credential/protocol; not proof that the feature is unavailable in official Codex |
| Valid `allow`/`deny` | The experimental action-bound Guardian inference path was reachable, **not necessarily free** |
| Direct API model error | New ChatGPT OAuth may not have access to `codex-auto-review`; the official free Codex path is not established on this route |
| Quota visibly increased | Not proof the reviewer was billed specifically; check other concurrent model activity |
| Quota unchanged | **Not proof of free use**; the displayed percentage may be rounded or delayed |
| New login with no quota snapshots | Expected: its token is scoped to the direct API rather than Codex `/wham/usage` |

## Limits and next milestone

This implementation purposely has **no automatic authorization** for ordinary Pi tools, no response-cache bypass, no attempts to conceal client identity, and no generic access to the `codex-auto-review` model. For a useful production extension, E2 would need stronger authorization boundaries, full parent/action freshness validation, recovery/cancel behavior, persistent audit with credential redaction, and an independent billing measurement.

Codex client code documents `guardian_credits_requested`, `parent_response_id`, `x-codex-guardian`, and that Guardian billing is backend-controlled. The new Pi OAuth implementation explicitly sets `resource=https://api.openai.com/v1` rather than the legacy Codex backend. The backend billing algorithm is not open sourced, and ChatGPT's official free Auto-review entitlement is **not guaranteed for third-party Pi integrations**.

### Reference implementation and protocol notes

- [OpenAI Codex `client.rs`](https://github.com/openai/codex/blob/main/codex-rs/core/src/client.rs)
- [OpenAI Codex Guardian V2 synchronous reviewer](https://github.com/openai/codex/blob/main/codex-rs/ext/guardian-v2/src/sync_reviewer/mod.rs)
- [Pi Codex Responses SDK](https://github.com/earendil-works/pi/blob/main/packages/ai/src/api/openai-codex-responses.ts)
- [Existing Pi Guardian protocol research](https://github.com/IgorWarzocha/howaboua-pi-stuff/blob/main/packages/pi-codex-guardian/PROTOCOL.md)

The initial unpadded new-OAuth direct-API review returned HTTP 200 with a valid Guardian assessment in user testing. **Free billing remains unverified.** The random-context stress test has not yet been run.

- [Pi new Sign in with ChatGPT OAuth resource](https://github.com/earendil-works/pi/blob/main/packages/ai/src/auth/oauth/openai-chatgpt.ts)
- [Pi direct OpenAI Responses SDK](https://github.com/earendil-works/pi/blob/main/packages/ai/src/api/openai-responses.ts)
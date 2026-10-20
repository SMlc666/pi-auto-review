# Pi Auto Review — Guardian API & quota probe (E1)

This is an **experimental connectivity and billing-attribution probe** for ChatGPT-subscribed Pi users. It is **not yet** a general permission reviewer, sandbox, or free-inference interface.

The user-facing goal is to run one **authentic harmless tool call** and one **action-bound Codex Guardian review** with Pi's existing ChatGPT OAuth credentials. The probe reads Codex usage before and after the review. It **never assumes a 200 response or stable rounded usage percentages prove the review was free**.

## Installation

Prerequisites: Pi with `@earendil-works/pi-ai` / `@earendil-works/pi-coding-agent` (recent 1.x), Node >= 22.19 or a compatible Bun runtime, and a **native ChatGPT OAuth** Codex provider/model selected in Pi. API-key and proxy routes are intentionally not treated as eligible for this subscription experiment.

Install directly in Pi:

```bash
pi install git:github.com/SMlc666/pi-auto-review
```

Or for local development, clone the repository and start Pi with:

```bash
pi --extension ./index.ts
```

For the first test, **create a fresh Pi conversation**, select a model whose API is `openai-codex-responses` on `https://chatgpt.com/backend-api`, and temporarily **disable Code Mode or other wrappers that hide native tool calls**. Normal Pi tool mode is required for this first prototype.

## Real test: one slash command

In Pi, type:

```text
/auto-review-probe
```

This asks your selected model to invoke `guardian_probe` exactly once. The tool is deliberately inert: **no files are read/written and no program is executed**. It performs only a separate Guardian review request plus read-only quota queries. Once done, inspect the tool result or run:

```text
/auto-review-probe-status
```

Expected output includes `review-accepted`, `review-denied` or `failed`, the HTTP status, whether a **real parent Response ID** was bound to the tool call, Guardian model token counts, and Codex quota snapshots (`codex.5h.usedPercent`, `codex.week.usedPercent` where provided).

The probe performs a real parent inference and a real reviewer inference and **may use subscription quota**. Token counts are not billing. If the before/after rounded quota percentage is unchanged, the only correct conclusion is **billing unverified**, not free. Some accounts may not expose precise per-call usage. Avoid running repeated tests blindly.

## Protocol approach

1. Only when explicitly armed by `/auto-review-probe`, add `client_metadata.guardian_credits_requested="true"` to the native parent Codex request through `before_provider_request`.
2. Observe the actual completed `resp_*` response from Pi's `provider_stream_event`, then bind the assistant's exact `guardian_probe` tool-call ID in `message_end`.
3. Use `ctx.modelRegistry.streamSimple()` with the **same selected OAuth Codex provider**, letting the SDK construct auth, session headers, URL and SSE. The model is set to `codex-auto-review` with low reasoning.
4. At its `onPayload` boundary, build the **tool-less structured review body**, referencing the real parent's response ID. At its `fetch` boundary, preserve SDK authentication while adding `x-codex-guardian: reviewer`, `x-openai-subagent: guardian`, session/turn and parent-thread identifiers.
5. Read `GET https://chatgpt.com/backend-api/wham/usage` immediately before and after review with the SDK-issued account and bearer headers. Only display usage percentages, not raw account IDs, bearer tokens, request payloads or API response bodies.

For integrity, the extension only targets the official `https://chatgpt.com/backend-api/codex/responses` reviewer URL, only accepts genuine OAuth subscription sessions, never fabricates parent IDs, and does **not** claim to be the official Codex CLI (`originator`/User-Agent remain Pi's genuine ones). If the server requires additional entitlement or official-client identity, report the resulting failure; do not attempt to evade it.

## Interpreting results

| Result | What we learned |
| --- | --- |
| No eligible model | Select a native ChatGPT OAuth Codex model/provider, not a custom proxy |
| No parent Response ID | Use a fresh conversation; disable Code Mode/tool wrappers; try again |
| HTTP 400 / 401 / 403 | The server rejected the request or credential/protocol; not proof that the feature is unavailable in official Codex |
| Valid `allow`/`deny` | The action-bound Guardian inference path was reachable |
| Quota visibly increased | Not proof the reviewer was billed specifically; check other concurrent model activity |
| Quota unchanged | **Not proof of free use**; the displayed percentage may be rounded or delayed |

## Limits and next milestone

This implementation purposely has **no automatic authorization** for ordinary Pi tools, no response-cache bypass, no attempts to conceal client identity, and no generic access to the `codex-auto-review` model. For a useful production extension, E2 would need stronger authorization boundaries, full parent/action freshness validation, recovery/cancel behavior, persistent audit with credential redaction, and an independent billing measurement.

Codex client code documents `guardian_credits_requested`, `parent_response_id`, `x-codex-guardian`, and that Guardian billing is backend-controlled. The backend billing algorithm is not open sourced, and ChatGPT's official free Auto-review entitlement is **not guaranteed for third-party Pi integrations**.

### Reference implementation and protocol notes

- [OpenAI Codex `client.rs`](https://github.com/openai/codex/blob/main/codex-rs/core/src/client.rs)
- [OpenAI Codex Guardian V2 synchronous reviewer](https://github.com/openai/codex/blob/main/codex-rs/ext/guardian-v2/src/sync_reviewer/mod.rs)
- [Pi Codex Responses SDK](https://github.com/earendil-works/pi/blob/main/packages/ai/src/api/openai-codex-responses.ts)
- [Existing Pi Guardian protocol research](https://github.com/IgorWarzocha/howaboua-pi-stuff/blob/main/packages/pi-codex-guardian/PROTOCOL.md)

No live account request has been made by this repository's authoring assistant. Test result and actual zero-charge entitlement remain **unverified** until you run the command locally.
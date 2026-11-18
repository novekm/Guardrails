# Design: Rules over Earlier Tool Calls (Receipts)

Status: proposal, not built. Branch `design/hooks-receipts`, on top of `feat/hooks-cedar`. Date: 2026-11-18.

## The rule we want

"Tool B may run only after tool A passed for the same target, in the same session, within a time window."
Example: `deploy` on service `checkout` only after `run_tests` passed on `checkout` in the last 15 minutes.
Today the hooks server checks each call alone: nothing records that a call ran or how it ended.

## What the callers can give us (checked in code)

| Caller | Before the call | After the call | Session |
|---|---|---|---|
| NeMo Relay plugin, SDK `nemo-relay-plugin` 0.11.0 | `register_tool_conditional_execution_guardrail(name, args)`, what the plugin uses today (`crates/nemo-guardrails-relay-plugin/src/lib.rs`) | `register_tool_execution_intercept`: `ToolExecutionContext { tool_name, args, tool_call_id: Option }` and `ToolNext::call(args) -> Result<ToolExecutionResult { result, annotation }>` (`async_sdk.rs` lines 951 and 1638). The same fields are in Relay `upstream/main` 2acd6ae1, `crates/plugin-abi/src/v1_0.rs` (`Invocation { name, value, tool_call_id, context }`). | None. `Context` holds an opaque scope, no session or run id. |
| Claude Code, Codex | `PreToolUse` (served today) | `PostToolUse` with `tool_use_id` and the tool's response | `session_id` on every hook |

So one intercept can see the arguments, run the tool, and see the result or the error, matched by `tool_call_id` when the provider supplies one.
Relay gives no session id.

## The after-tool endpoint

`POST /v1/hooks/nemo-relay/after-tool`, plus `PostToolUse` on the existing Claude Code and Codex paths.

```json
{"hook_event_name": "NemoRelayToolExecutionResult", "tool_name": "run_tests", "tool_input": {"service": "checkout"},
 "tool_call_id": "call_7", "session_id": "...", "result": {"exit_code": 0}, "error": null}
```

The server records a receipt and answers `{"receipt": "<id>"}`.
It refuses a result whose `tool_call_id` and arguments digest match no call it allowed in that session: a receipt can only follow a check.
The result itself is never stored; only the outcome derived from it.

## The receipt record

One JSON object per line in an append-only store, as the quarantine store already does (`hooks_server.receipt_store`).

| Field | Meaning |
|---|---|
| `id`, `time` | Server-assigned; the server's clock, never the caller's |
| `session` | The session the call belongs to |
| `agent`, `agent_digest` | From the `X-NeMo-Agent-*` headers, as for quarantine |
| `tool`, `tool_call_id` | The call |
| `target_digest` | `sha256` of the canonical JSON of the argument the config names as the tool's target (`receipts.tools.run_tests.target: service`). Digest, not value: arguments can carry data. |
| `outcome` | `passed`, `failed`, or `completed`. `passed` and `failed` come from a config rule over the result, such as `passed_if: {pointer: /exit_code, equals: 0}`; an error from the tool is `failed`; without a rule, `completed`. |
| `prev`, `sig` | Digest of the previous line, and an HMAC-SHA256 over the record with a server key file (`receipt_key_file`). The server refuses to start on a broken chain or signature. |

## How a rule says "B only after A passed for the same target"

### Option A (recommended): Cedar, with receipts supplied as context

At each `call_tool` request the server works out the request's target digest (the same config names the target argument) and adds `context.history`, keyed by tool name, from the receipts of this session and this target:

```
context.history["run_tests"] = { "passed_seconds_ago": 240, "failed_seconds_ago": 900, "passed_count": 2 }
```

```cedar
@id("deploy/after-tests")
forbid(principal, action == Hooks::Action::"call_tool", resource == Hooks::Tool::"deploy")
unless { context.history has "run_tests"
         && context.history["run_tests"] has passed_seconds_ago
         && context.history["run_tests"].passed_seconds_ago <= 900 };
```

Run with cedarpy 4.12.2: no history denies, a pass 240 seconds ago allows, one 2000 seconds ago denies, a failure alone denies.
Cedar has no "there exists" over a set, so the server shapes the history into fixed slots.
That is the same move Dogwood makes: `dogwood lower` turns each temporal condition into a `context.*` slot filled at run time (Dogwood README, "Compile-to-Cedar").
Needs: from the plugin, a `register_tool_execution_intercept` that posts the result (it keeps today's guardrail for the check); from Relay, a session or run id on the invocation, since none exists. Until Relay has one, the plugin can send an id it is configured with, which ties a session to a plugin instance; Claude Code and Codex already send `session_id`.
Cost: Python only, `cedarpy` already used by Part 1, policies stay analyzable by Cedar's tools, matches OpenShell's Cedar choice. Limit: what the slots can express is what the server computes (last pass, last fail, counts); a new kind of history question needs a server change.

### Option B: Dogwood, with an embedded engine

Dogwood is an open language (Apache-2.0, `github.com/dogwood-policy/dogwood`, v1.1.2 on 2026-11-12): Cedar plus `when temporal { formerly within 15m ... }`, `since`, `once` and aggregations.
AgentCore's gateway evaluates it as a service, and a reference implementation renders policies for that gateway (header `x-amzn-bedrock-agentcore-policy-session-id`).
It also has open evaluators: a reference interpreter that its README says is "NOT intended for production use", and `dogwood-policy/dogwood-local-engine` (crates.io 1.1.2), a Rust library that keeps the history durably and decides each request as it arrives.
The example below follows the Dogwood README's syntax and was not run through `dogwood validate`.
Its default event schema already has the two hooks: a `request` event (decision) and a `response` or `error` event (history, with input and output fields).

```text
permit (principal, action == Hooks::Action::"deploy", resource)
when temporal {
    formerly within 15m Hooks::Action::"run_tests"::response{ input.service: context.input.service, output.exit_code: 0 }
};
```

Needs: the same after-tool events from the plugin, and the same session id from Relay. In Guardrails: a Rust extension (PyO3) around `dogwood-local-engine`, because there is no Python binding (the PyPI package `dogwood` is an unrelated project), or a sidecar process like the engine's `dogwood-server` demo on Unix sockets. Either way, platform wheels or a second process for a Python package.
The engine's README leaves event provenance, authentication, timestamps and control-plane security to the embedding application; the receipt record above covers those either way.

### Recommendation

Build Option A now. It reuses the Cedar engine from Part 1, stays Python, keeps policies analyzable, and needs nothing from Relay that Option B does not also need.
Keep the after-tool endpoint and the receipt record engine-neutral: they are Dogwood's `request` and `response` events, so `dogwood-local-engine` can be a second rule language later, chosen per check as Cedar is today, if a temporal question outgrows the fixed slots.

## Asks, by owner

- Relay: a session or run id on tool invocations (the ABI `Context` has none).
- Plugin: register `register_tool_execution_intercept`, post `{tool_name, args, tool_call_id, result | error}` after `next.call`, and keep the existing guardrail for the check.
- Hooks server: `after-tool` endpoint, receipt store with chain and signature, `receipts.tools.<name>.target` and `passed_if`, `context.history` on Cedar requests, `PostToolUse` on the Claude Code and Codex paths.

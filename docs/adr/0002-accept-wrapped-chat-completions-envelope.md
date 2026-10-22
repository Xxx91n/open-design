# 0002. Accept wrapped Chat Completions envelopes at one normalization boundary

## Status

Accepted

## Context

ClinePass (`https://api.cline.bot/api/v1`) returns non-streaming Chat Completions inside an envelope:

```json
{ "data": { "object": "chat.completion", "choices": [ ... ], "usage": { ... } }, "success": true }
```

Streaming on the same endpoint returns standard OpenAI SSE. The envelope is legacy upstream behaviour that has been reported and acknowledged, with no shipped opt-out (cline/cline#12647).

OpenDesign's connection test reads top-level `choices`, so a 200 response that did contain a usable completion was reported as `kind: 'unknown'` with the raw body as `detail`. Three further non-streaming consumers read top-level `choices` the same way: `apps/daemon/src/design/finalize-design.ts`, and two call sites in `apps/daemon/src/memory-llm.ts`. The chat path itself streams (`/api/proxy/openai/stream` always sends `stream: true`), so only non-streaming paths were affected.

## Decision

1. **One normalization boundary, no adapter layer.** A pure function `normalizeOpenAICompletion(payload)` lives in `apps/daemon/src/integrations/openaiCompletionShape.ts` and is called by the four non-streaming OpenAI consumer sites: `inspectProviderCompletion` and `extractOpenAIMessageText` in `apps/daemon/src/connectionTest.ts`, `extractDesignMd` in `apps/daemon/src/design/finalize-design.ts`, and the OpenAI/OpenAI-compatible calls in `apps/daemon/src/memory-llm.ts`. The four `fetch` call sites stay as they are. `apps/daemon/src/media/index.ts` is excluded: it reads `message.images`, a different response shape.

2. **Unwrapping requires all five conditions.** `data` is a plain object; `Array.isArray(data.choices)`; top-level `error` is absent; top-level `success !== false`; `data.object`, when present, equals `chat.completion`. Anything else is returned unchanged.

3. **Unwrapping never substitutes for validation.** The existing checks — non-empty `choices`, `finish_reason` handling, sample extraction, model echo — run on the unwrapped object unchanged. Accepting a known envelope is not the same as loosening the acceptance gate.

4. **Unwrapping is logged.** Each time the shim fires, emit a line naming the provider. This is how we will know when it can be removed.

5. **Shape-mismatch diagnostics stay in `detail`.** No new `ConnectionTestKind`. The full upstream body (truncated and redacted) goes to the daemon log; the UI shows a shape summary instead of raw JSON.

## Alternatives considered

- **Strict only (status quo):** rejects a response that contains a usable completion, and leaves the same failure latent in `finalize-design` and `memory-llm`, where it surfaces with no visible error at all.
- **Loose acceptance (any `data.choices`):** turns `{ "success": false, "data": { "choices": [...] } }` into a false success. For a user-initiated connection test, a false negative costs more than a false positive.
- **Per-provider opt-in switch:** the envelope is unconditional and does not vary with configuration, so the switch would ask users to declare a format detail they cannot observe. Capability switches declare request-shape capabilities; they are not a way to accept broken responses.
- **Unify the four call sites into one `callOpenAICompatible()`:** this is the end state, but it is a refactor sized for a bug fix. Revisit when a fifth non-streaming consumer appears.
- **New `ConnectionTestKind` for response-shape mismatches:** the granularity test is whether the client takes a different action. It does not — the user checks the provider either way — so this does not qualify as a distinct error code.

## Consequences

Non-streaming OpenAI consumers now accept one additional, explicitly enumerated response shape. Every other shape still fails, and failure diagnostics get more actionable rather than less.

The shim is a compatibility layer with no upstream fix shipped. It should be removed once Cline offers a standard response — they have discussed an `X-Response-Format: openai` request header — and the unwrap log line is the signal that it is still in use.

Known and deliberately unfixed here: `apps/web/src/components/EntryShell.tsx` returns without feedback when the onboarding connection test fails. That is tracked as a separate follow-up issue; it is orthogonal to response shape, as a `402 Insufficient balance` failure is blocked the same way.

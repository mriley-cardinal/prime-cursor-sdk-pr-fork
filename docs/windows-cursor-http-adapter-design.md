# Windows Cursor SDK HTTP adapter — draft design

Status: proposal only. No adapter implementation or runtime acceptance is included.

## Purpose and scope

Explore using the Cursor SDK from Windows-native Rust Prime Agent without modifying
or recompiling Prime. This is a personal-use integration proposal, not a promise to
publish, distribute, support, or maintain an npm package. Public discussion of this
Markdown design does not authorize public package publication or releases.

The first target is Windows-native, local Cursor execution, and the exact Prime
version inspected below. Cursor Cloud, macOS/Linux support, WSL, a TypeScript
extension loader, and a Rust Prime fork are outside this proposal. Retain upstream
license and attribution. Existing uncommitted dependency, test, and fork-policy
changes are separate work and are not included in this design-only PR.

## Evidence and current limits

Inspection targeted official Prime Agent tag `v0.10.1-beta.46`, commit
`668551b3ddf0ef3b5e63e779e1fd8d201461b920`, and installed `@cursor/sdk` version
`1.0.40`. Findings are source-level evidence, not an end-to-end compatibility test.

- The installed Rust CLI rejects `-e`. Package resources cover skills, prompts,
  and themes; they do not load this repository's TypeScript provider extension.
- Prime custom `models.json` entries support `api`, `baseUrl`, `apiKey`, models,
  compatibility metadata, and headers. A new provider defining models needs a
  provider-level base URL and API key, plus an API selection.
- Prime's existing `openai-completions` transport posts streaming requests to
  `<baseUrl>/chat/completions`. It consumes text/reasoning deltas and indexed
  OpenAI-style tool-call deltas. This provides an HTTP integration seam without
  registering a new Rust API implementation.
- Prime passes system instructions, history, and tool schemas into its provider
  and executes returned tool calls itself. `python_repl` is its canonical tool.
- Cursor SDK exposes an agent API, not a verified raw chat-completions API.
  Its installed `options.d.ts` documents `tools: []` as disabling built-in tools,
  `tools: ["mcp"]` as permitting the MCP family, and local callback `customTools`.
  Those callbacks execute host code without interactive approval. System-prompt
  replacement is server-side gated and can fail on the first send. These are
  constraints to verify, not a proven adapter recipe.

A text-only response is insufficient: it does not demonstrate that Prime can run
its normal tool loop. Never replay a Cursor-executed action as a fresh Prime tool
call; doing so could execute the action twice.

## Proposed architecture

```text
Rust Prime Agent (unchanged)
  -> existing openai-completions HTTP client
  -> authenticated loopback Node adapter
  -> official Cursor SDK local agent
```

The Node adapter would be a separately launched, foreground process for the first
prototype. No installer, auto-start service, daemon replacement, or changes to
running Prime sessions are proposed. Reuse narrowly scoped conversion/sanitizing
helpers from this repository only when their contracts fit; do not load the entire
Pi extension into a fake host.

Prime would select a distinct custom provider name, such as `cursor-local-proxy`,
with `api: "openai-completions"` and a loopback `/v1` base URL. Do not label this
wire API `cursor-sdk`: the existing Rust transport cannot dispatch that unknown
API. Exact model IDs, context limits, thinking options, and costs must come from
verified SDK metadata or be explicitly unknown; Prime's generic defaults are not
Cursor model facts. Configuration is an eventual opt-in step, not part of this PR.

### Ownership and tool execution

Prime must remain the owner of its Python REPL, tool execution, and saved sessions.
The adapter owns HTTP translation, per-run SDK state, and cancellation/cleanup.
Cursor must not independently execute file/shell/subagent actions that bypass this
ownership. Disable unwanted tool capabilities explicitly and verify the effective
surface, including ambient MCP/settings and subagent behavior.

The principal open question is how a Cursor custom-tool invocation can become a
Prime tool-call response and then receive the next HTTP request's tool result.
A possible design is a pending SDK callback bridged across the response boundary;
this is only a candidate. Before implementation beyond the spike, prove that:

1. The SDK invocation can remain pending while the HTTP response ends with
   `finish_reason: "tool_calls"`.
2. A subsequent authenticated request is correlated to exactly that invocation
   and delivers its result once, without resending the prompt or duplicating work.
3. Cancellation, a missing result, retry, or client disconnect releases all waits
   and closes or invalidates the SDK run safely.
4. Model-produced tool names and arguments map to Prime's supplied schemas; no
   tool schema or call is invented by parsing assistant prose.

If no supported, reliable bridge exists, stop. Do not fake tool completion, run
Prime Python inside Node, or claim that text-only success establishes compatibility.
A native Rust provider plus a Node sidecar may be considered separately, but it
would still need the same semantic proof and new approval.

### HTTP and streaming contract

The first required endpoint is `/v1/chat/completions`; model enumeration endpoints
are deferred unless the verified Prime client requires them. Validate request
shape, content types, model selection, and supported options. Reject unsupported
content/options explicitly rather than silently dropping meaning.

Translate SDK events to SSE chunks with stable completion/model identifiers,
text/reasoning where verified, indexed tool IDs/names/argument fragments, a truthful
terminal finish reason, optional verified usage, and `[DONE]`. Never forward the
SDK event stream verbatim. Do not claim cache/token equivalence without observed
contract evidence. Define errors before headers and failures after streaming starts
so Prime cannot mistake a broken stream for a successful answer.

Start with one active run per adapter and reject concurrent requests clearly until
correlation is proven. A completion ID is not automatically a session ID. Prime's
ordinary HTTP body must not be assumed to carry a stable branch/session identity;
verify the actual wire payload before designing reuse. Do not key sessions by
prompt prefixes, model ID, or shared authentication alone. Cross-request tool-result
correlation and safe behavior when conversation history changes are acceptance
requirements. Restarting the adapter initially invalidates pending work; durable SDK
resume is deferred rather than implied by Prime's saved conversation.

## Security and lifecycle

- Bind only to `127.0.0.1`; choose and document a collision-safe local port policy.
  Do not expose a listener on all interfaces or use a tunnel.
- Require a separate random proxy token on every request. Reject unauthorized
  requests before parsing large bodies or invoking the SDK. Apply bounded request
  sizes and explicit overload rejection. Browser origins/hostnames must not bypass
  these checks; loopback alone is not an authorization boundary.
- Keep the Cursor credential only in the adapter's process environment or a
  separately approved credential mechanism. Never put it in `models.json`, URLs,
  command-line arguments, test fixtures, errors, or logs. Do not inherit it into
  Prime/tool subprocesses unnecessarily.
- Prime `apiKey` config values resolve environment names, but an unset name falls
  back to the literal string. Validate that the proxy-token variable is populated;
  do not treat that fallback as secure. Provider headers are not automatically
  environment-expanded. Do not use `!command` credential resolution for this spike.
- Authenticate the proxy token independently of the Cursor credential. Scrub both
  and all authorization headers. No request-body/debug capture by default.
- Use disposable workspaces and isolated Prime state for tests. Set the SDK working
  directory and settings sources explicitly; do not load ambient rules/MCP silently.
  Treat Cursor/Prime execution as user-privileged, not as a security sandbox.
- Define abort/deadline behavior for active runs and pending callbacks. Verify SDK
  cancellation, agent disposal, bridge closure, temporary-store cleanup, and absence
  of orphan child processes with metadata/process evidence, not assistant text.

## Staged proof and stop conditions

### A. Mock protocol proof (no Cursor credentials)

Use an isolated Prime process and a mock loopback service to verify the real HTTP
request/SSE contract: text, reasoning if supported, a `python_repl` tool call,
Prime's actual tool result and continuation, errors, stream truncation, cancellation,
and unrelated histories. Record only scrubbed structural evidence. Confirm no
existing sessions/configuration are changed. This stage proves Prime's transport,
not Cursor SDK compatibility.

### B. SDK bridge contract proof

Against installed SDK source/types and focused fixtures/tests, verify the custom
callback/agent lifecycle required for the tool bridge. Keep prior captured fixtures'
version provenance. Resolve system-prompt entitlement and lossless history mapping.
Do not claim a mocked SDK event proves current service behavior.

Stop before authenticated testing if callback continuation, request identity, prompt
control, tool ownership, or cancellation cannot be demonstrated. Return a revised
design rather than building a misleading proxy.

### C. Explicitly approved live Windows proof

Only after the preceding gates and approval, use one disposable workspace and
isolated Prime state for a minimal authenticated run. Verify streaming; one Prime
REPL tool round trip; a controlled read/edit side effect exactly once; conversation
continuation; abort during work; successful next use; credential redaction; and
resource cleanup. Session/branch reuse and adapter restart behavior must either be
verified or explicitly rejected. A greeting alone is not acceptance.

No Rust compilation, global installation, package publication, daemon shutdown, or
replacement of a running Prime executable is authorized by this design.

## Acceptance and decision

The prototype is viable only if the unchanged installed Prime host can use the
adapter while retaining its genuine tool loop and safe cancellation. Document
unsupported features, exact version contracts, test skips, and known dependency
advisories. Existing Windows Git/packaging timeouts and skipped filesystem tests
remain separate validation gaps; do not hide them by excluding tests or relaxing
security assertions. A clean npm audit does not cover SDK-internal bundled code.

Before implementation, review this design and select the narrow proof scope.
Before any later commit/push, review the exact diff and meet the applicable
validation policy. No publishing workflow, registry registration, public release,
or ongoing package-maintainer role is part of this proposal.

## Pinned source references

All Prime links below use the installed-version commit, not mutable `main`:

- [Custom provider/model schema and validation](https://github.com/PrimeIntellect-ai/prime-agent/blob/668551b3ddf0ef3b5e63e779e1fd8d201461b920/crates/pa-core/src/models/custom.rs#L106-L249).
- [Request credential/header resolution](https://github.com/PrimeIntellect-ai/prime-agent/blob/668551b3ddf0ef3b5e63e779e1fd8d201461b920/crates/pa-core/src/models/registry.rs#L812-L875).
- [Environment-or-literal config semantics](https://github.com/PrimeIntellect-ai/prime-agent/blob/668551b3ddf0ef3b5e63e779e1fd8d201461b920/crates/pa-core/src/auth/resolve_config_value.rs#L10-L34).
- [HTTP request and stream handling](https://github.com/PrimeIntellect-ai/prime-agent/blob/668551b3ddf0ef3b5e63e779e1fd8d201461b920/crates/pa-ai/src/providers/openai_completions/stream.rs#L254-L351) and [endpoint construction](https://github.com/PrimeIntellect-ai/prime-agent/blob/668551b3ddf0ef3b5e63e779e1fd8d201461b920/crates/pa-ai/src/providers/openai_completions/stream.rs#L570-L715).
- [Prime provider context and event bridge](https://github.com/PrimeIntellect-ai/prime-agent/blob/668551b3ddf0ef3b5e63e779e1fd8d201461b920/crates/pa-core/src/session_engine/provider_adapter.rs#L137-L201).
- [Compiled Rust provider registry](https://github.com/PrimeIntellect-ai/prime-agent/blob/668551b3ddf0ef3b5e63e779e1fd8d201461b920/crates/pa-ai/src/registry.rs#L10-L54) and [built-ins](https://github.com/PrimeIntellect-ai/prime-agent/blob/668551b3ddf0ef3b5e63e779e1fd8d201461b920/crates/pa-ai/src/registry.rs#L104-L132).
- [Capability package resource kinds](https://github.com/PrimeIntellect-ai/prime-agent/blob/668551b3ddf0ef3b5e63e779e1fd8d201461b920/crates/pa-core/src/packages/resolve/mod.rs#L20-L32).
- Cursor SDK evidence: installed `@cursor/sdk@1.0.40`,
  `dist/esm/options.d.ts` (`AgentOptions`, `LocalAgentOptions`, `SDKCustomTool`),
  `agent.d.ts`, and `run.d.ts`; [official TypeScript SDK documentation](https://cursor.com/docs/sdk/typescript).

Authored by Rowan, Mike Riley's Prime Agent Overlord

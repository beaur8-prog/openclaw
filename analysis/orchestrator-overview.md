# OpenClaw Orchestrator Notes

## Short answer

OpenClaw does not have a single "orchestrator LLM" that runs the whole system.

The orchestration layer is mostly deterministic TypeScript code that:

- accepts inbound messages
- resolves session and agent routing
- applies policy, preprocessing, and directives
- chooses the execution path
- runs the model with retry/fallback rules
- dispatches the final reply back to the correct surface

The LLM participates inside that pipeline as the agent engine that generates text and may choose tools during a turn, but it is not the top-level controller of the product.

## Main orchestration path

The highest-signal control flow for inbound message handling is:

1. `src/auto-reply/dispatch.ts`
2. `src/auto-reply/reply/dispatch-from-config.ts`
3. `src/auto-reply/reply/get-reply.ts`
4. `src/auto-reply/reply/get-reply-run.ts`
5. `src/auto-reply/reply/agent-runner.ts`
6. `src/auto-reply/reply/agent-runner-execution.ts`
7. `src/agents/pi-embedded-runner/run.ts`

That is the practical orchestration chain for "message came in, now decide what to do and produce/deliver a reply".

## What each layer does

### 1. Dispatcher lifecycle

`src/auto-reply/dispatch.ts` is the outer dispatch wrapper.

It owns:

- inbound context finalization
- reply dispatcher lifecycle
- buffered dispatch / typing-aware dispatch setup
- ensuring queued reply work is drained and cleaned up

This is orchestration code, not LLM logic.

### 2. Inbound control-plane logic

`src/auto-reply/reply/dispatch-from-config.ts` is one of the clearest orchestration modules.

It handles:

- duplicate inbound suppression
- session-store lookup
- origin-channel vs current-surface routing decisions
- plugin/hook integration
- TTS/reply-delivery policy wiring
- ACP dispatch bypass/dispatch decisions

This module decides where and how a message should be processed and where the response should go.

### 3. Session/model/directive preparation

`src/auto-reply/reply/get-reply.ts` is the main preparation stage before model execution.

It handles:

- config loading
- agent resolution from session key
- workspace preparation
- media understanding and link understanding preprocessing
- command authorization checks
- session initialization
- default model resolution
- channel model overrides
- directive parsing and inline action handling

This is a major part of the system orchestrator.

### 4. Turn assembly

`src/auto-reply/reply/get-reply-run.ts` builds the actual turn context.

It handles:

- group-chat context and intro rules
- typing policy resolution
- queue policy selection
- session reset notices
- reply routing for reset/system events
- packaging the prompt and run settings for the agent runner

Again, this is deterministic orchestration around the LLM call.

### 5. Reply run coordination

`src/auto-reply/reply/agent-runner.ts` coordinates a live reply run.

It handles:

- active-run queue behavior
- followup enqueue/drop behavior
- memory flush before run
- block streaming pipeline setup
- typing signal lifecycle
- post-run accounting and followup handling

This is the operational orchestrator for an individual turn.

### 6. Retry/fallback/run-loop logic

`src/auto-reply/reply/agent-runner-execution.ts` is the clearest "model execution orchestrator".

It handles:

- run IDs and run context registration
- provider/model fallback wrapping via `runWithModelFallback`
- selecting embedded runtime vs CLI provider runtime
- streaming partial outputs
- tool-result emission hooks
- transient HTTP retry behavior
- compaction and overflow recovery paths

This is code-driven orchestration around model execution.

### 7. Embedded agent runtime

`src/agents/pi-embedded-runner/run.ts` is where the underlying agent engine is actually run.

It handles:

- model/provider runtime preparation
- auth profile rotation/cooldown handling
- context window guards
- lane/queue execution
- retry loops
- compaction
- oversized tool-result recovery
- provider-specific runtime details

This module is still orchestration code, but it is closer to the actual LLM runtime.

## Where the LLM actually sits

The LLM is inside the agent runtime, not above it.

The strongest code signal is:

- `src/agents/agent-command.ts` imports `SessionManager` from `@mariozechner/pi-coding-agent`
- `src/agents/pi-embedded.ts` re-exports `runEmbeddedPiAgent`
- `src/agents/pi-embedded-runner/run.ts` drives the embedded Pi agent runtime

That means the model is used as an execution engine within the OpenClaw control flow.

In practice, the LLM is responsible for things like:

- generating the assistant response
- deciding whether to call tools during the run
- continuing a tool-use loop within the constraints of the runtime

But the LLM is not the sole orchestrator for:

- ingress routing
- session ownership
- channel/origin reply routing
- queue/drop/followup policy
- duplicate suppression
- typing behavior
- fallback policy
- auth/profile rotation
- retry/backoff policy
- delivery plumbing

Those are owned by OpenClaw's application code.

## Routing-related orchestration

There is also a separate routing layer for deciding which agent/session should own a conversation.

Relevant files:

- `src/routing/session-key.ts`
- `src/channels/plugins/registry.ts`
- `src/plugins/runtime.ts`

These modules are not "the" top-level orchestrator, but they provide key routing primitives:

- session key normalization
- agent scoping
- plugin/channel registry lookup
- active plugin registry state

## Practical conclusion

If someone asks "what module is the orchestrator?", the most accurate answer is:

- At the product level, the orchestration is distributed across the `src/auto-reply/reply/*` pipeline plus `src/agents/pi-embedded-runner/run.ts`.
- If one module must be named as the closest thing to the reply orchestrator, `src/auto-reply/reply/get-reply.ts` and `src/auto-reply/reply/agent-runner-execution.ts` are the best candidates.
- If one module must be named as the model-runtime orchestrator, `src/agents/pi-embedded-runner/run.ts` is the best candidate.

So: OpenClaw is code-orchestrated, LLM-assisted, not LLM-orchestrated end to end.

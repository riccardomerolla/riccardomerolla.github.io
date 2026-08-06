---
layout: post
title: "Introducing llm4ts: an Effect-TS library for LLM agent workflows"
---

I kept writing the same two hundred lines.

Retry the call when the provider hiccups. Pull the JSON back out of the code fence the model wrapped it in. Add up the tokens so I know what the run cost. Thread a cancellation signal through four layers. Then swap the provider and rewrite half of it.

Every LLM project I started began with that same prelude before it got to the interesting part. So I extracted it. It's called [llm4ts](https://github.com/riccardomerolla/llm4ts), it's MIT, and it's Effect-native TypeScript.

## The one idea

A model seat is a typed effect, and a CLI coding agent is just another seat.

That second half is the part I care about. Most LLM libraries model "a provider" as an HTTP client with an API key. But half my actual work now runs through coding agents — Claude Code, Codex, Cursor — which are processes, not endpoints. They stream. They use tools. They cost money. They fail in the same ways an HTTP provider fails.

There's no good reason those should be two different abstractions in my code. In llm4ts they aren't.

## Getting something running

Three packages: `@llm4ts/core` for contracts and connectors, `@llm4ts/flow` for orchestration, `@llm4ts/runner` for the Node boundaries.

```bash
npm install @llm4ts/core @llm4ts/flow @llm4ts/runner effect
```

Node 22 or newer. `effect` is a peer dependency on the Effect 4 beta line.

`runNode` assembles every Node boundary — HTTP client, process executor, temp files, file store, connector registry — and hands your flow body a ready context. You don't wire layers. `runFlowMain` runs the result.

```ts
import * as Effect from "effect/Effect"
import { ApiConnectorConfig } from "@llm4ts/core/ConnectorConfig"
import { ConnectorIds } from "@llm4ts/core/Models"
import { makeChat } from "@llm4ts/flow/Chat"
import { runFlowMain, runNode } from "@llm4ts/runner/FlowRunner"

const program = runNode(
  {
    workDir: process.cwd(),
    workspace: process.cwd(),
    userPrompt: "Summarise this repository in three bullets.",
    coder: ApiConnectorConfig.make({ connectorId: ConnectorIds.Mock })
  },
  (context) =>
    Effect.gen(function* () {
      const chat = yield* makeChat(context.reasoning, { system: "Be terse." })
      const reply = yield* chat.ask(context.userPrompt)
      yield* Effect.log(reply)
    })
)

runFlowMain(program)
```

Run that and a canned reply lands in your log. No credentials, no network: `ConnectorIds.Mock` is a deterministic connector that ships in core and is the only one built without an HTTP client, which is why your tests don't need an API key. Leave `reasoning` out of the options and it defaults to whatever you passed as `coder`.

Embedding this in an app that already has a runtime? `runEmbedded` is the same flow with the `Scope` left in your hands.

## Swapping the seat

Here's the part that earned the library its place. Same flow body, different seat:

```ts
import { ApiConnectorConfig, CliConnectorConfig } from "@llm4ts/core/ConnectorConfig"
import { ConnectorIds } from "@llm4ts/core/Models"

const viaApi = ApiConnectorConfig.make({
  connectorId: ConnectorIds.Anthropic,
  model: "claude-sonnet-5"
})

const viaCli = CliConnectorConfig.make({
  connectorId: ConnectorIds.ClaudeCli,
  model: "claude-sonnet-5",
  readOnly: true
})
```

One is an HTTPS request with an API key from the environment. The other spawns a coding agent as a subprocess. Both satisfy the same `LlmServiceShape`, so the flow body doesn't know or care.

`readOnly` isn't decoration. On the `claude-cli` connector it resolves to `--permission-mode default --disallowed-tools Write,Edit,NotebookEdit` in the argv the connector builds. The seat can read and reason; it cannot touch your files.

The full list of connector ids today: `openai`, `anthropic`, `gemini-api`, `lm-studio`, `ollama`, `claude-cli`, `gemini-cli`, `opencode`, `codex`, `copilot`, `pi`, `antigravity-cli`, `grok`, `cursor`, `mock`.

## Structured output, without the ritual

Models return JSON wrapped in prose, or in a fence, or in a fence inside prose. `executeStructured` takes an Effect `Schema` and a JSON schema, and gives you back a decoded value:

```ts
import * as Schema from "effect/Schema"
import type { JsonSchema } from "@llm4ts/core/Models"

const Triage = Schema.Struct({
  score: Schema.Number,
  channel: Schema.Literals(["blog", "x", "linkedin"]),
  reason: Schema.String
})

const triageJsonSchema: JsonSchema = {
  type: "object",
  properties: {
    score: { type: "number" },
    channel: { type: "string", enum: ["blog", "x", "linkedin"] },
    reason: { type: "string" }
  },
  required: ["score", "channel", "reason"],
  additionalProperties: false
}

const ranked = runNode(options, (context) =>
  context.reasoning.executeStructured(context.userPrompt, Triage, triageJsonSchema)
)
```

You write the shape twice. That's the honest cost: the Effect `Schema` decodes the reply, the JSON schema is what gets handed to the model, and nothing derives one from the other today. In exchange, the candidate-extraction logic — fenced blocks, balanced braces, the lot — lives in `StructuredOutput.ts` and is unit-tested. I stopped writing that regex.

## Tools

```ts
import * as Effect from "effect/Effect"
import { makeTool } from "@llm4ts/core/tools/Tool"
import { isJsonRecord } from "@llm4ts/core/providers/CliSupport"

const wordCount = makeTool({
  name: "word_count",
  description: "Count the words in a piece of text.",
  parameters: {
    type: "object",
    properties: { text: { type: "string" } },
    required: ["text"],
    additionalProperties: false
  },
  sandbox: "WorkspaceReadOnly",
  execute: (parameters) =>
    Effect.succeed({
      words:
        isJsonRecord(parameters) && typeof parameters.text === "string"
          ? parameters.text.trim().split(/\s+/).length
          : 0
    })
})
```

Note `isJsonRecord` rather than an inline `!Array.isArray(...)`. TypeScript types `Array.isArray` as `arg is any[]`, and a `ReadonlyArray` is not a subtype of `any[]`, so the negative branch never narrows the array case away. I lost twenty minutes to that before I found the exported guard.

`sandbox` and `requires` are on every tool descriptor. A tool declares what it needs; the executor decides whether it gets it.

## What it looks like in anger

I run several autonomous agents against this in a small internal tool of my own — long-lived loops that work unattended for hours. That's where most of these APIs got their shape, and where three things went wrong first.

**Bound the turns, not just the dollars.** `FlowRunnerOptions` takes a `budget`, and it's checked *after* the flow body completes. An agent that researches forever never completes, so it never gets checked. One run spiralled for ninety minutes before I noticed. Every CLI seat now carries an in-flight cap:

```ts
CliConnectorConfig.make({
  connectorId: ConnectorIds.ClaudeCli,
  readOnly: true,
  turnLimit: 40,
  flags: { "max-turns": "40" }
})
```

Both lines are deliberate. In 0.10.0 the `claude-cli` connector builds its argv from `flags` and doesn't yet read `turnLimit`, so the flags passthrough carries the real cap as `--max-turns`. `turnLimit` stays set for the day the connector honours it. Check the argv your connector actually builds before you trust a bound.

**A budget you don't feed enforces nothing.** Cost tracking runs on `TokensUsed` events. If you build a `Chat` without handing it the run's event hub, usage the backend reported is silently discarded, the tracker accrues zero, and your ceiling passes every time. This is the whole difference:

```ts
const chat = yield* makeChat(context.reasoning, {
  system,
  events: context.events,
  agent: "triage"
})
```

It's documented in `ChatOptions`. I still shipped it wrong first.

**Read-only by default.** Every seat that only makes a judgement gets `readOnly: true`, and the deterministic part of the program performs the writes. A prompt-injected input can't make a seat touch the filesystem, because the seat has no filesystem.

## What's rough

It's 0.10.0. I'm not going to pretend otherwise.

The peer dependency is pinned to an exact Effect 4 beta — `4.0.0-beta.102` — and that line still moves. Expect to pin `effect` yourself and to reconcile it on every llm4ts bump.

Cost enforcement is also only as good as the usage a backend reports. The `antigravity-cli`, `copilot`, and `cursor` connectors declare `usageReporting: false`, so a `CostBudget` over those accrues nothing and enforces nothing. That's documented rather than fixed, which is the honest state of it. Check the [provider capability matrix](https://github.com/riccardomerolla/llm4ts/blob/main/docs/provider-capabilities.md) before you rely on a capability — it also covers streaming, structured output, resumability, and interactive stdin, which vary more than you'd hope.

## Where to look

- [The repository](https://github.com/riccardomerolla/llm4ts)
- [API overview](https://github.com/riccardomerolla/llm4ts/blob/main/docs/api.md)
- [Architecture guide](https://github.com/riccardomerolla/llm4ts/blob/main/docs/architecture.md)
- [Configuration](https://github.com/riccardomerolla/llm4ts/blob/main/docs/configuration.md)
- [Runnable examples](https://github.com/riccardomerolla/llm4ts/tree/main/examples) — `basic.ts` and `plain-js.mjs` need no credentials

There's a CLI too: `@llm4ts/shell` ships an `llm4ts` command. I drive the library from code almost all the time, so that one deserves its own post rather than a paragraph here.

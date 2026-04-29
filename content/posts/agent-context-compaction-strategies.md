---
title: "Compaction Algorithms for Long-Running Agent Sessions"
date: "2026-04-30"
description: "Four canonical strategies for shrinking a conversation when the context window fills — what each one preserves, what it discards, and the trade-offs between fidelity and headroom."
tags: [agents, llm, context-management, summarization]
categories: ["technology"]
---

## The problem

A long-running agent accumulates messages: user prompts, model responses, reasoning traces, tool calls, tool outputs. The model's context window is finite — from a hundred thousand tokens on the smaller end to a million or more on the largest current frontier models. Once the conversation no longer fits, continuing without intervention triggers a hard error or silent truncation by the provider.

A **compaction algorithm** shrinks the conversation. It's structurally a three-part decision:

1. **Trigger** — when compaction runs.
2. **Preservation** — which existing messages are kept verbatim and which get rolled into a summary.
3. **Summary shape** — the prompt template the summarization call is given.

The preservation axis is where the algorithms differ most. Four strategies are observed in real code.

## Strategy 1: Replace all

Send the entire conversation to the model with an instruction to produce a structured handoff summary (a fixed-section markdown template — sections like current state, files involved, technical context, next steps). Replace the entire history with the summary as a single new message.

![Strategy 1: Replace all](/images/compaction/strategy1.svg)

`S` is the summary. Future turns continue with `[S, new exchanges…]`.

## Strategy 2: Preserve user messages

Pre-trim: walk the history from newest to oldest dropping *agent-generated* items (assistant responses, tool calls, tool outputs) until the rest fits the context budget. Summarize what's left. Build the new history as a token-budgeted slice of the most recent user messages with the summary appended at the end.

![Strategy 2: Preserve user messages](/images/compaction/strategy2.svg)

The token budget for the user-message slice is a hard cap (e.g., 20K tokens worth, selected newest-first then re-ordered chronologically).

This strategy needs a marker on each message indicating who generated it; a role string (user / assistant / tool) is sufficient.

## Strategy 3: Preserve a recent token fraction

Pick a fraction `p` (observed values: 0.3). Walk the history from newest to oldest accumulating token cost until you've collected `p × total_tokens` worth — that's the **preserved tail**. Summarize everything older as a single state-snapshot message; place the summary at the head, the preserved tail after.

![Strategy 3: Preserve a recent token fraction](/images/compaction/strategy3.svg)

The split point is computed by character or token count, then snapped to the next user-message boundary so the preserved tail starts on a user turn rather than mid-exchange. Within the tail, preservation is role-agnostic — whatever messages fall in the recent fraction stay verbatim regardless of who generated them.

## Strategy 4: Preserve recent turns

A "turn" is one user message plus everything generated in response (assistant text, tool calls, tool outputs) up to the next user message. Preserve the last `N` turns (default observed: `N=2`); summarize everything older.

![Strategy 4: Preserve recent turns](/images/compaction/strategy4.svg)

The boundary is determined by walking user-message indices from newest backward. Implementations pair this with a per-turn token budget: preserve up to N turns or up to T tokens, whichever is smaller (observed defaults: 25% of usable context, clamped to [2K, 8K]).

## Comparison

| Strategy | Preserves | Selection basis |
|---|---|---|
| Replace all | Nothing — summary is the new history | n/a |
| User messages | User messages within a token budget; summary appended at end | per-message size by role |
| Recent token fraction | Newest p × total tokens regardless of role; summary at head | token sum |
| Recent N turns | Last N user-bounded exchanges; summary at head | per-turn size, capped by token budget |

## A common pre-trim step

All four strategies do something *before* the LLM call to keep the input itself small. The reason is the same — the model that's going to write the summary is also bound by a context window, and feeding it 100% of an overflowing conversation just moves the overflow into the summarization call. Concrete pre-trim moves seen in practice:

- Iteratively dropping the oldest items and retrying when the summarization call itself overflows.
- Walking newest-first and dropping agent-generated items (assistant text, tool calls, tool outputs) until the input fits, leaving user messages intact.
- Truncating individual tool outputs to their last few dozen lines, with the original spilled to a temp file.
- Hard-capping the total token budget for tool-response content at some fraction of the model's window (e.g., 50K tokens).
- Marking older tool outputs as "compacted" with a timestamp so they render as `[Old tool result content cleared]` to the LLM but stay queryable in the host's storage.

The output of pre-trim is what actually goes to the summarization call. The strategy's preservation rule applies to the *original* history (which messages survive verbatim into the post-summary state), not to what the summarizer sees.

## The shared shape

Each strategy produces a new history that's some interleaving of two things: a generated summary message and zero or more verbatim messages from the original history. The strategy decides:

- *what* gets preserved (nothing / user-only / recent-token-fraction / recent-N-turns)
- *where* the summary sits relative to the preserved messages (head, tail, or alone)

A host implementing strategy selection ships a single replacement mechanism: store the summary, record which original message IDs (if any) form the preserved tail and where they sit relative to the summary, and have the message-history loader splice them back together on the next turn. The strategy decides what's preserved; the host applies it uniformly.

## Triggers

Three trigger patterns are observed:

- **Proactive token threshold.** When estimated context usage reaches a configured fraction of the model's window, run compaction before the next LM call. Observed thresholds include 50% of the window; "remaining tokens fall below a fixed buffer" (e.g., 20K of headroom for windows larger than 200K, or 20% of the window for smaller ones); and a per-model auto-compact token limit.
- **Reactive overflow.** Compaction kicks in when a model call has already returned a context-window-exceeded error, with the next attempt running against the compacted history.
- **Manual / explicit.** Compaction fires when explicitly requested via a parameter or user action.

Implementations may use any combination of the three.

## The shape

At the implementation level all four strategies reduce to the same pair of artifacts: a generated summary message, and an optional pointer identifying where the preserved tail begins in the original message list. The history loader splices them on the next turn. Swapping between strategies is a question of which pointer values the strategy produces, not of the storage shape.

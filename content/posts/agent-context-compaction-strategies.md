---
title: "Compaction Algorithms for Long-Running Agent Sessions"
date: "2026-04-30"
description: "Three canonical strategies for shrinking a conversation when its context window fills — what each one preserves and what it discards."
tags: [agents, llm, context-management, summarization]
categories: ["technology"]
---

## The problem

A long-running agent accumulates messages over time: user prompts, model responses, reasoning traces, tool calls, tool outputs. The model's context window is finite — from a hundred thousand tokens on the smaller end to a million or more on the largest current frontier models. Once the conversation no longer fits, continuing without intervention triggers a hard error or silent truncation by the provider.

A **compaction algorithm** shrinks the conversation in place. The interesting strategies share a single shape: pick a moment on the conversation's timeline (the *split point*), collapse everything older into a single summary message, place the summary at the split point, and keep everything newer as-is.

## The conversation as a timeline

A conversation is a sequence ordered in time. Older messages on the left, newer ones on the right. The model reads the array left-to-right on every turn — the leftmost item is the oldest thing it remembers, the rightmost is the most recent thing it has to respond to. New user turns extend the right edge.

When the conversation outgrows the window, a strategy answers one question: **which moment on the timeline is the split point?** The three strategies below differ entirely in how they pick it.

## Strategy 1: Preserve user messages

Split-point rule: walking newest-first, keep every user message and drop everything else until a max-token cap (~20K) is reached. Kept user messages are re-ordered chronologically; the summary sits at the split point, just after the latest preserved user message.

![Strategy 1: Preserve user messages](/images/compaction/strategy1.svg)

## Strategy 2: Preserve a recent token fraction

Split-point rule: walk backward from the present, accumulating tokens, until you reach `p × total_tokens` (observed: `p = 0.3`); snap to the next user-message boundary so the preserved tail starts on a user turn. Everything earlier than that snapped point collapses; everything in the recent fraction stays as-is regardless of role.

![Strategy 2: Preserve a recent token fraction](/images/compaction/strategy2.svg)

## Strategy 3: Preserve recent turns

Split-point rule: a "turn" is one user message plus everything generated in response (assistant text, tool calls, tool outputs) up to the next user message. Keep the last `N` complete turns (default `N=2`); collapse everything older into the summary. A per-turn token budget caps each preserved turn (observed defaults: 25% of usable context, clamped to [2K, 8K]).

The summary sits at the split point — chronologically immediately after the last preserved turn.

![Strategy 3: Preserve recent turns](/images/compaction/strategy3.svg)

## Comparison

| Strategy | Split-point rule | What survives the cut |
|---|---|---|
| Preserve user messages | every user message is on the "keep" side; agent items are on the "drop" side | every user message (within a ~20K token budget) |
| Preserve recent token fraction | walk back until `p × total_tokens` collected, snap to user-msg boundary | everything in the recent fraction, regardless of role |
| Preserve recent turns | the user-msg boundary that starts the (M − N + 1)ᵗʰ turn | the last N complete turns (within per-turn budget) |

## When to choose which

Strategy choice is application-specific. Two judgments about the workload do most of the work:

**Are user and agent messages equally important?** If the user's intent is the durable record and the agent's responses can be reproduced from those prompts, role asymmetry is real and **Strategy 1** fits — it keeps every user message as-is and lets the summary absorb everything else. If both sides carry equal weight — agent reasoning, tool outputs, and the model's earlier choices feed into later turns as much as the user's prompts do — **Strategy 2** and **Strategy 3** are the right shape, since both preserve recent content regardless of role.

**How much do recent messages drive the next decision?** If the model needs the last few exchanges in full to answer well — "what did the model just suggest?", "did the last tool call succeed?" — recency matters a lot, and the choice is between Strategy 2 (token-budgeted recency, snapped to a user boundary) and Strategy 3 (whole-turn recency, never split mid-tool-call). If recency matters less than the cumulative trail of decisions, Strategy 1 is fine — the user's prompts capture the trajectory, and the summary captures the agent's exploration.

The two factors interact. An agent doing long-form research where the user mostly says "continue" or "look at X" gets little signal from preserved user messages — the recent agent reasoning is the dense signal, so Strategy 2 or 3 is preferable. A coding agent where the user explicitly states each goal benefits from Strategy 1 — the goals are the structure, the agent's exploration between them can be lossy. There is no universal answer.

## A common pre-trim step

All three strategies do something *before* the LLM call to keep the input itself small. The reason is the same — the model that's going to write the summary is also bound by a context window, and feeding it 100% of an overflowing conversation just moves the overflow into the summarization call. Concrete pre-trim moves seen in practice:

- Iteratively dropping the oldest items and retrying when the summarization call itself overflows.
- Walking newest-first and dropping agent-generated items until the input fits, leaving user messages intact.
- Truncating individual tool outputs to their last few dozen lines, with the original spilled to a temp file.
- Hard-capping the total token budget for tool-response content at some fraction of the model's window (e.g., 50K tokens).
- Marking older tool outputs as "compacted" with a timestamp so they render as `[Old tool result content cleared]` to the LLM but stay queryable in the host's storage.

The output of pre-trim is what actually goes to the summarization call. The strategy's preservation rule applies to the *original* timeline (which messages survive as-is into the post-compaction state), not to what the summarizer sees.

## Triggers

Three trigger patterns are observed:

- **Proactive token threshold.** When estimated context usage reaches a configured fraction of the model's window, run compaction before the next LM call. Observed thresholds include 50% of the window; "remaining tokens fall below a fixed buffer" (e.g., 20K of headroom for windows larger than 200K, or 20% of the window for smaller ones); and a per-model auto-compact token limit.
- **Reactive overflow.** Compaction kicks in when a model call has already returned a context-window-exceeded error, with the next attempt running against the compacted history.
- **Manual / explicit.** Compaction fires when explicitly requested via a parameter or user action.

Implementations may use any combination of the three.

## The shared shape

Every strategy reduces to the same two artifacts the host stores: a generated **summary message**, and a **tail-start pointer** marking where the preserved messages begin on the original timeline. On the next turn, a history loader splices them: `[…messages-from-pointer-onwards, summary]`.

| Strategy | Tail-start pointer |
|---|---|
| Preserve user messages | first kept user message |
| Preserve recent token fraction | first message of the recent fraction |
| Preserve recent turns | first user message of the (M − N + 1)ᵗʰ turn |

Swapping strategies is a question of which pointer value gets written, not of storage shape — the host ships one replacement mechanism and applies it uniformly.

## Terms

| Term | Definition |
|---|---|
| **compaction** | Shrinking the conversation in place when it no longer fits the context window: collapse older content into a summary, preserve recent content as-is. |
| **split point** | A position on the conversation timeline. Everything older than it collapses into the summary; everything newer is preserved as-is. Strategies differ only in how they pick this position. |
| **tail** | The contiguous suffix of messages preserved as-is — from the split point to the present. |
| **summary message** | A single LLM-generated message condensing everything older than the split point. The host stores it alongside the tail. |
| **tail-start pointer** | The stored marker (a message ID or array index) identifying where the tail begins on the original timeline. One value gets written per compaction. |
| **turn** | One user message plus every agent-generated item produced in response, up to the next user message. |
| **agent-generated item** | Any conversation item not authored by the user: assistant text, tool calls, tool outputs, reasoning traces. |
| **trigger** | The condition that initiates compaction — a proactive token threshold, a reactive context-window-exceeded error, or a manual user request. |
| **pre-trim** | Token-reduction applied to the input of the summarization call itself so the summarizer doesn't overflow. Distinct from the strategy's preservation rule, which describes the post-compaction timeline. |
| **host** | The runtime that owns the conversation state and runs compaction (the agent process or session manager). |


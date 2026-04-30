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

Every strategy reduces to two artifacts the host stores: a generated **summary message**, and an optional **preserved-from pointer** identifying where the preserved tail begins in the original message list. On the next turn, a history loader splices them back together.

Per strategy, those artifacts look like:

| Strategy | Summary position | Preserved-from pointer |
|---|---|---|
| Replace all | alone — no tail | none |
| User messages | tail (after the kept user messages) | first kept user message |
| Recent token fraction | head | first message of the recent fraction |
| Recent N turns | head | first user message of the (M − N + 1)ᵗʰ turn |

Swapping strategies is a question of which pointer values get written, not of storage shape — the host ships one replacement mechanism and applies it uniformly.

## Triggers

Three trigger patterns are observed:

- **Proactive token threshold.** When estimated context usage reaches a configured fraction of the model's window, run compaction before the next LM call. Observed thresholds include 50% of the window; "remaining tokens fall below a fixed buffer" (e.g., 20K of headroom for windows larger than 200K, or 20% of the window for smaller ones); and a per-model auto-compact token limit.
- **Reactive overflow.** Compaction kicks in when a model call has already returned a context-window-exceeded error, with the next attempt running against the compacted history.
- **Manual / explicit.** Compaction fires when explicitly requested via a parameter or user action.

Implementations may use any combination of the three.

## The unified algorithm

First, the types in play:

```python
# A Message is the host's existing per-turn record. Every Message has an
# `id`. Each carries a role ∈ {user, assistant, tool} and a content payload.
Message = (id, role, content)

# Where the summary sits relative to the preserved tail.
Position = HEAD | TAIL | ALONE

# A Strategy is a 4-tuple bundling everything that varies across algorithms.
# All four functions are pure: they take history (and a parameter) and
# return a value; nothing is persisted from inside them.
Strategy = (
    pre_trim:  (history: [Message], context_limit: int) -> [Message],
    prompt:    str,                                        # system prompt for summarization
    preserve:  (history: [Message]) -> [Message],          # the verbatim tail
    position:  Position,
)

# What compact() returns to the caller. The host writes these three
# fields onto the session row; nothing else needs to change in storage.
CompactResult = (
    summary:         Message,         # the new summary message
    preserved_from:  id | None,       # first id in preserved_tail, or None for ALONE
    position:        Position,
)
```

The pipeline is one function, parameterised by the strategy:

```python
# `lm.summarize(system_prompt, history) -> Message` is the host's LM client.
def compact(history, strategy, lm, context_limit) -> CompactResult:
    # 1. Pre-trim: shrink the summarizer's input so the call itself fits.
    summarizer_input = strategy.pre_trim(history, context_limit)

    # 2. Summarize: produce one new Message.
    summary = lm.summarize(strategy.prompt, summarizer_input)

    # 3. Pick which originals survive verbatim.
    preserved_tail = strategy.preserve(history)

    # 4. Hand back the artifacts the host needs to persist.
    return CompactResult(
        summary        = summary,
        preserved_from = preserved_tail[0].id if preserved_tail else None,
        position       = strategy.position,
    )
```

The four canonical strategies are just different fillings of the 4-tuple. The function names below are the operations defined in each strategy's section earlier in this post:

```python
# Strategy(pre_trim,                prompt,                 preserve,                       position)
ReplaceAll       = Strategy(noop_or_drop_oldest, handoff_prompt,        empty,                          ALONE)
PreserveUser     = Strategy(drop_agent_items,    handoff_prompt,        top_user_msgs_within_budget,    TAIL)
PreserveFrac(p)  = Strategy(truncate_tool_outs,  state_snapshot_prompt, tail_after_user_split(p),       HEAD)
PreserveTurns(N) = Strategy(prune_old_tool_outs, state_snapshot_prompt, last_N_turns_within_budget(N),  HEAD)
```

On each subsequent turn the loader reads the persisted result and reconstructs what the LM sees from it plus the full message log:

```python
# `index_of(id, msgs)` is the position of the message with that id in msgs.
def lm_history(result: CompactResult | None, all_messages: [Message]) -> [Message]:
    if result is None:                                  # session never compacted
        return all_messages
    if result.preserved_from is None:                   # ALONE — replace all
        return [result.summary]
    tail = all_messages[index_of(result.preserved_from, all_messages):]
    if result.position == HEAD:  return [result.summary, *tail]
    else:                        return [*tail, result.summary]   # TAIL
```

Triggering — deciding *when* `compact()` runs — is a separate predicate over the session's token usage and any explicit user signal, and is orthogonal to which strategy runs.


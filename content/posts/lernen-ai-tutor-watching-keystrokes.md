---
title: "An AI tutor that watches every keystroke"
date: "2026-05-04"
description: "Building lernen — a Bubble Tea flashcard TUI for German vocabulary where the AI teacher sees every reveal, replay, and grade, and reacts in tight loops."
tags: [tui, llm, agents, learning, bubble-tea, go]
categories: ["technology"]
---

## A flashcard app, but the AI is in the loop

Most "AI study buddies" are chatbots glued to the side of a flashcard app. You study, then you ask a question, then the model answers from cold context. The model has no idea you just got the same word wrong twice and replayed the audio four times.

I wanted to flip that. The model should see every action — reveals, replays, grades, notes — and react inside the study loop, not after it.

The result is **lernen**: a single Go binary that loads any Anki `.apkg` deck and gives you a Bubble Tea TUI with the AI as a permanent right-side panel. The interesting part isn't the TUI. It's what gets piped into the system prompt on every turn.

## The minimum viable feedback cycle

Before any AI, the cycle on each card looks like this:

1. Card front shows. Press `space` to reveal — or press `a` and type your guess first.
2. After reveal, you self-grade `1` (again) / `2` (hard) / `3` (good) / `4` (easy).
3. The next card surfaces.

Spaced repetition handles intervals (1m / 10m / 1d / 7d), and `Again` re-queues the card 3 cards later in the same session so failures actually come back. That part is mechanical; no AI involved.

Where AI enters: between step 2 and step 3, the model gets a fresh request with full session context and replies with one short tip — *"You're close — Brötchen is the diminutive of Brot, so a small bread roll. Think of -chen as 'little'."*

It's a tip, not a judgment. The model never grades the answer or recommends a number. The learner stays in charge of `1-4`.

## What the model actually sees

The system prompt is rebuilt from scratch on every turn. Not a fixed system message with a growing user thread — the **whole context block is regenerated**, so the model always sees current state.

Concretely:

```
DECK PROGRESS (lifetime)
- Total cards: 926 · Known: 23 · Learning: 47 · New: 856 · Due now: 870

CARD IN CONTEXT (what the learner is looking at)
- Screen: study
- German: Brötchen
- English: bread roll

THIS SESSION (live, in-memory only)
- Reviewed: 4 · good+: 1 · misses: 1 · streak: 0 · elapsed: 2m
- Struggling on: "Brötchen" (bread roll) ×2

CARDS YOU'VE WORKED WITH THIS SESSION (newest first)
1. "Brötchen" (bread roll) — grades: 1/again → 2/hard · audio×3 · note: "I keep mixing this up with Brot" · 12s ago
2. "fehlen" (to be missing) — grades: 2/hard · 47s ago
3. "Haus" (house) — grades: 3/good · 1m ago

ACROSS RECENT SESSIONS (loaded from history)
- 2h ago (12m session): 12 reviewed · 8 good · 3 misses · struggled: "Brötchen" (bread roll) ×3
    note on "Brötchen": "I keep mixing this up with Brot"
- 1d ago (8m session): 7 reviewed · 5 good · 1 misses · struggled: "fehlen" (to be missing) ×1

RECENT RAW EVENTS (chronological tail)
- 0s ago: graded "Brötchen" (bread roll) → again (took 4s)
    learner's note: "I keep mixing this up"
- 4s ago: replayed audio for "Brötchen" (likely listening repeatedly to lock in pronunciation)
- 5s ago: replayed audio for "Brötchen" (likely listening repeatedly to lock in pronunciation)
- 9s ago: revealed answer for "Brötchen"
- 14s ago: graded "fehlen" (to be missing) → hard (took 6s)
```

That block has four time-scales stitched together: lifetime progress, current session, the per-card roster, and prior sessions persisted to a JSONL file under `~/.local/share/lernen/`. The roster is the most useful one — instead of a chronological event soup, the model sees each card the user has touched as one row with **all** grades chained, audio replays counted, and any notes inlined.

## Recording every decision

Anything the user does in the TUI flows into the same `Session` log. Pressing `p` to replay audio is recorded. Toggling mute is recorded. Submitting a search query is recorded. Opening the chat overlay is recorded.

```go
const (
    EventGrade        = "grade"
    EventReveal       = "reveal"
    EventAudioReplay  = "audio_replay"
    EventMuteToggle   = "mute_toggle"
    EventChatOpen     = "chat_open"
    EventChatSend     = "chat_send"
    EventStudyEnter   = "study_enter"
    EventStudyLeave   = "study_leave"
    EventBrowseFilter = "browse_filter"
    EventSearch       = "search"
)
```

This sounds excessive until you see the difference in tip quality. *"Audio replayed 4 times for Brötchen"* in the prompt produces tips targeting pronunciation. Without it, the model gives a generic dictionary fact.

The prompt is explicit about this:

> **GROUND EVERY TIP IN OBSERVED BEHAVIOR**
>
> Tips and answers MUST reference what the learner has actually done. Examples:
> - If they replayed audio 3+ times for a word: target pronunciation, give a phonetic hook.
> - If they failed the same card twice: change the angle. Don't repeat the hook you tried before.
> - If they took 12s before grading "again": acknowledge they were close; give a precise contrast with a similar word they confused it with.
>
> If a tip would work as a generic fact about the word with no behavioral context, you're probably missing the signal — re-read the session log.

## Teacher, not judge

An earlier iteration had the AI score the user's typed answer with `✓ correct` / `◐ close` / `✗ off` icons and recommend a grade. It felt off — the model was overstepping into a role the learner already owned.

Current contract: the AI is a **teacher**, not a judge. It reads the typed guess against the canonical meaning. If the guess is essentially right, it affirms briefly and adds one reinforcing fact. If wrong, it gently explains what the learner got wrong, in 1-2 short sentences. It never scores, never suggests a grade.

```
You're close — Brötchen is the diminutive of Brot, so a small bread
roll. Think of -chen as "little."
```

That's it. The user still picks `1-4` themselves, with no AI suggestion to anchor on.

The session log carries the artifact forward: `guess="bread"; teacher_said="You're close — Brötchen is the diminutive..."`. The next time the same card surfaces, the model sees what it taught last time and is told explicitly to *change the angle*, not repeat itself.

## Letting the AI inject cards

The most interesting recent addition: the teacher can **add related cards to the queue**.

When the learner misses a card or attaches a note, the prompt invites an optional structured trailer:

> OPTIONAL — only if there are 1-2 specific OTHER German words from this A1 deck the learner should drill alongside this one (e.g., a confusable pair, a same-root word, a near-synonym), end your reply with a NEW LINE containing exactly:
>
> ```
> INJECT: word1, word2
> ```

A typical reply now looks like:

```
You're close — Brötchen is the diminutive of Brot, so a small bread roll.
INJECT: das Brot, das Brötchen
```

The runtime parses the trailer, **strips it from the visible tip**, resolves each word against the deck (case-insensitive substring on the canonical form), and inserts up to 2 matched cards at the next position in the study queue. A status flash announces it: *"+2 related cards queued by tutor."*

Guards keep the queue sane: capped at 2 per tip, deduped against the next 8 cards already coming up, silent skip for words that don't match anything in the deck. The user sees the tip and the next card; the structured machinery is invisible.

This is the lightest possible "tool use" — no function-calling APIs, no tool-result round-trips, just a parsable trailer and local resolution. It works the same on OpenAI and Anthropic.

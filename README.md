<img src="screenshots/app-icon.png" width="72" align="right" alt="HiNikki app icon">

# HiNikki

**A voice companion for older adults living with dementia, and the family app that keeps its world true.**

An iOS app, currently on TestFlight. This repository is the public overview: what it does, how it is
built, and the design decisions worth defending. The source is private.

---

## The problem

Someone living with dementia asks the same questions many times a day. What day is it. Is my
daughter coming. Did I take my pills. A voice assistant can answer all of those, but a generic one
answers *generically*, and gets it wrong in the ways that matter most, because it has no idea who
your daughter is or whether she is actually coming.

Meanwhile the family carries the whole picture in their heads and in group chats, and has no way to
put it where it would help.

HiNikki is one app with two faces. The older adult gets a single large "Talk to Nikki" button, a
calm view of today, their people, and help. The family gets a dashboard: schedule, reminders, safe
places, emergency contacts, conversation recaps, and a review queue of what Nikki believes it has
learned.

## The core idea: one AI, a brain around it

**Nikki's intelligence is deliberately not in our codebase.** The conversation loop, meaning speech to
text, the language model, text to speech, turn-taking, interruption and barge-in, runs inside
ElevenLabs Conversational AI. What we build is the brain *around* it: everything that turns a
generic voice model into *this person's* companion, and everything that turns a conversation into
data a family can safely act on.

```
        ┌───────────────── ElevenLabs (the voice loop) ─────────────────┐
        │   STT → LLM (system prompt + dynamic variables) → TTS         │
        └──▲────────────────────────────────────────────────┬───────────┘
           │ context injected at session start               │ client-tool calls
  read path│                                                 ▼  (executed on the device)
   Supabase├── snapshot tiers ── session variables ──►   {{vars}}      tool layer
  (RLS)    ▲                                                 │
           └──── proposals ── family reviews ── write-back ◄──┘        write path
```

**Read path.** Before a call starts, the app assembles what Nikki should know *right now*, the
person's profile, who is in their life, today's schedule, the weather where they actually are,
and recent conversation summaries. It injects that as dynamic variables.

**Write path.** This is the part I would defend in a design review.

## Nothing the AI says is treated as data

A language model talking to a confused person is a data-integrity problem wearing a friendly voice.
If Nikki mishears "my son visited" as a fact, and that fact silently enters the family record, the
record is now worse than useless. It is confidently wrong, and nobody knows.

So the boundary is drawn twice:

1. **Only tool calls count.** Nothing in the *transcript* is ever persisted as a fact. The agent has
   to explicitly call a client tool that runs on the device.
2. **Elder sessions cannot write canonical data at all.** They can only file *proposals*. A family
   member reviews and approves. The elder cannot approve their own proposal, cannot pre-fill any
   review field, and cannot forge an audit trail.

The one deliberate exception is low-risk care guidance. Support notes auto-apply, because that is
the feature that most needs to feel effortless, and its blast radius is small. It stays visible and
editable by the family.

That boundary is enforced **at two independent layers**: the tool layer (an elder session is simply
never handed a direct-write tool) and row-level security in Postgres. Either alone would be a single
point of failure. Some tables are legitimately self-writable by the elder for their own setup flows,
so database policy alone could not stop a voice-driven write, and a tool layer alone would fall to
anything that bypassed it.

**Why two layers instead of one:** the failure mode you are defending against is not a hacker. It is
a hallucination plus a person who cannot reliably tell you it was wrong.

## Other decisions worth naming

**Buy the voice loop, build the personalization.** Low-latency, natural turn-taking is genuinely
hard, and for this audience the *feel* is the product. A stilted assistant is not a lesser version
of a good one, it is a different and worse thing. The trade is real: the system prompt and tool
definitions live on a vendor dashboard rather than only in git, which is a versioning cost we
accept knowingly.

**End the call from the client, not the server.** A voice session has a lifecycle full of edge
cases. The microphone silently dying mid-call was a real, reproducible bug, and it is exactly the
kind of failure that a dementia user cannot report.

**A per-user narrative memory layer** was evaluated as a full replacement store, formally compared
three ways, and rejected as the store, then adopted as a layer on top. Writing down what you
rejected, and why, is the part of architecture that survives the team.

## Stack

Expo and React Native with TypeScript and file-based routing. Supabase (Postgres, Auth, Realtime, Edge
Functions, Storage) with row-level security throughout. ElevenLabs Conversational AI. A small
in-house design system rather than a component library.

Realtime sync keeps the family app and the elder app consistent while a conversation is happening.
Edge functions handle post-call extraction, semantic recall, scheduled pipeline sweeps, and
authentication token minting. The sensitive keys never reach a device.

## Status

On TestFlight and in active development. I lead the project and the small team building it. Privacy work is scoped and underway ahead of
wider testing: a DPIA, a granular consent flow, an Article 6/9 legal basis mapping, and processor
agreements. This matters because voice transcripts, photos, schedules and precise location can constitute
special-category health data under GDPR regardless of how few users you have.

## Screenshots

> **To do:** add TestFlight screenshots of the elder home screen, the talk-to-Nikki state, the
> family dashboard, and the proposal review queue.

## Why this repository exists

The source is private because HiNikki is a product with real users. This overview is here because
the architecture is the interesting part, and architecture can be discussed in the open.

---

*Built by [Willem Grasso](https://github.com/Wgrasso), who leads the project and its team.*

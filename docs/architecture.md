# Architecture

This document describes the **current production path** of Chiruru (status: November 2026). Components that exist in the implementation but are experimental, run in shadow mode, or have been retired are left out. Where a statement could not be confirmed, it says so.

## Contents

1. [Runtime Architecture](#1-runtime-architecture)
2. [Component Responsibilities](#2-component-responsibilities)
3. [Data and Control Flow](#3-data-and-control-flow)
4. [Local vs Cloud LLM](#4-local-vs-cloud-llm)
5. [State and Memory](#5-state-and-memory)
6. [Failure Handling](#6-failure-handling)
7. [Observability](#7-observability)

---

## 1. Runtime Architecture

### Processes

| Process | How it runs | Responsibility |
|---|---|---|
| **Live bot** | One long-running Python asyncio process, run as a systemd user service. The operator starts it on stream days. The service restarts on failure, with a cap on restarts. | Live show, chat, speech, avatar, OBS control, in-process schedulers |
| **Live slot creator** | systemd timer, daily, oneshot | Creates the day's YouTube Live slot. Running it twice does no harm. |
| **Post-stream worker** | systemd oneshot, started by the live bot after a recording is saved | Unattended post-stream actions (see [§3.4](#34-post-stream-pipeline)) |
| **Waiting-scene data builder** | systemd timer, daytime only, oneshot | Builds render data for the off-air scene. It runs only while the bot is stopped. |
| **Local inference server** | Separate local server | Serves the local LLM through a chat-completions HTTP API |
| **TTS server** | Separate self-hosted server | Streams synthesized speech over HTTP |
| **OBS / VTube Studio** | Desktop applications | Broadcast composition, recording, and avatar rendering, controlled through their WebSocket APIs |

### Inside the live bot

At startup the bot:

- loads settings;
- checks for a stale "streaming" flag left by an earlier crash;
- writes a PID file;
- connects the OBS, YouTube broadcast, VTube Studio, and overlay controllers.

It then starts these concurrent tasks under one supervisor (`asyncio.gather` with exceptions collected rather than propagated):

- Discord client
- Live stream bot: chat intake plus the stream cycle
- YouTube Live chat adapter, optional
- Error monitor loop
- Inference-server recovery service

It also registers three in-process APScheduler jobs: live scheduling, Shorts scheduling, and the post-stream sweeper. In dry-run mode the schedulers are **not started at all**. That way a scheduled job cannot change YouTube or OBS state by bypassing per-call dry-run checks.

## 2. Component Responsibilities

| Component | Responsibility | Does not do |
|---|---|---|
| **Trend fetcher** | Fetches RSS feeds on an interval, resolves redirects, filters unsafe topics, extracts and caches article text | Ranking or show decisions |
| **Topic selector** | Scores candidates with the local LLM, deduplicates them, applies the score threshold, queue size and per-category caps, and keeps dropped items as "unaired headlines" | Generating speech |
| **Stream cycle / phase router** | Runs the show as a phase state machine and picks the next phase with deterministic rules and configured probabilities | Writing content |
| **Comment intake** | Normalises chat from each platform into one message type and selects and batches comments between and during phases | Generating replies directly |
| **Riria responder** | Generates Riria's turns through the same LLM queue, using Riria's profile and Chiruru's preceding line as input. Validates the output before playback. A routing step decides whether a viewer comment is answered by Chiruru or by Riria. | Leading the show |
| **Prompt assembly** | Builds system and user prompts from persona, platform, task templates and retrieved context. One composer owns the chat-reply system prompt. | Calling the LLM |
| **LLM client** | Calls the local inference server's chat API, with or without streaming. The model name comes from configuration. It sends only an allowlist of sampling options, removes reasoning tags, records the model name the server actually reported, and records each call outcome. | Queueing or retries across callers |
| **LLM execution queue** | The single gateway to the local LLM: one worker, a priority queue, and a timeout per job | Choosing the model |
| **Inference-server recovery service** | Detects stalls from call outcomes and server logs, confirms them with a probe, and reloads the model through the server's management interface, within limits | Changing the model or the prompts |
| **Speech pipeline** | Splits streamed text into sentence or comma chunks, sends them to the configured TTS backend, and schedules playback | Deciding what to say |
| **Playback and subtitles** | Plays audio into the broadcast mix, sets loudness and stereo position per character, starts subtitles when playback starts, and drives Chiruru's lip sync. Riria's avatar is a separate overlay in OBS. | Synthesis |
| **Memory store** | Holds the raw buffer, history, Mem0, and Chroma collections. It runs batch summarisation and retrieval. | Real-time generation |
| **Post-stream sweeper** | Is armed when a recording is saved, and starts the post-stream worker on a schedule | The post-stream work itself |
| **Post-stream orchestrator** | Runs registered post-stream actions under a lock and tracks each action in a receipt | Live-show decisions |

### Characters on one model

- **Character profile.** A small, immutable record per character selects:
  - the persona definition;
  - sampling options;
  - a fallback line;
  - which context features apply. For example, a runtime persona state is used for Chiruru but not for Riria.

  Unknown speaker names resolve to the main host.
- **Speaker identity.** The speaker travels with the request through the generation workflow. Both characters share one LLM queue. Queue jobs are tagged by the component that submitted them, not by speaker.
- **Turn linkage.** A Riria turn takes Chiruru's preceding line as input. It is stored in the raw buffer with a link to that line.
- **Outputs.** Each character has a separate voice configuration and a separate avatar output: a VTube Studio model for Chiruru and an OBS overlay for Riria.
- **Code structure.** The chat-reply path is shared and switched by profile. Riria's in-show turns still run through a separate set of modules beside the main path. The two are not yet fully unified.

## 3. Data and Control Flow

### 3.1 News: from feed to topic queue

1. **Fetch.** RSS search feeds, each tagged with a hand-assigned category, are fetched on an interval. The set is mostly AI, LLM, and GPU queries, plus some non-AI categories for variety. Already-seen topics are filtered out.
2. **Extract.** The article body is extracted from the page, falling back to a headless browser when that fails. It is cached in SQLite.
3. **Score.** The local LLM rates each candidate's importance into coarse levels, using a short excerpt of the body. If scoring fails, the item gets a neutral low score. It is not dropped.
4. **Deduplicate.** Items are deduplicated first by title, then by sentence-embedding similarity. Embedding runs in a worker thread, because running it on the event loop was observed to stall the show for several seconds. Items held in reserve fill any shortfall.
5. **Cap.** A minimum score, a maximum queue size, a per-category cap, and a per-session dedup are applied. Dropped items are kept as *unaired headlines*, which filler segments can use as material.
6. **Pop.** When the show needs the next topic, it checks sources in this order:
   - the pre-stream editorial cursor, if one exists ([§4](#4-local-vs-cloud-llm));
   - otherwise the normal queue, which avoids categories used recently.

### 3.2 Show flow

```
NEWS_INTRO ─► QUICK_TAKE ─► DEEP_DIVE ─► INTERACTIVE ─► (phase router) ─► next phase
                 │ (fails or empty)  ▲
                 └───────────────────┘
```

- **Quick take.** A short take on the topic, built from a trimmed summary.
- **Deep dive.** An opinion piece. Its context comes from retrieval over past broadcasts and memory, Chiruru's past opinions on related topics, and a fixed-template *program context* block from the editorial plan.
- **Interactive.** A bounded chat segment that answers selected viewer comments. Each reply is given by Chiruru or Riria, depending on the routing step.
- **Riria's turns.** In news dialogue, after deep dives, and in dialogue turns of some talk segments, Riria adds a short reaction to what Chiruru said. In the Q&A corner, Riria asks the question.
- **After the interactive segment**, a deterministic router picks the next phase. It checks rules in a fixed order:
  - end-of-show wind-down;
  - an *anti-chaining* rule: after any non-news segment, the next segment is forced back to news;
  - configured-probability segments, such as growth talk and radio monologue.
- **Rule ordering.** Newly added segments go at the end of the rule chain, so adding one does not change how often existing segments fire.

### 3.3 Generation to broadcast

```
prompt ─► LLM queue ─► local LLM server (streamed tokens)
       ─► sentence/comma chunker ─► TTS server (streamed audio)
       ─► per-speaker loudness normalisation ─► audio device ─► OBS mix
                                             ├─► lip sync ─► VTube Studio
                                             └─► subtitle shown on playback start ─► OBS
       ─► exchange logged to raw buffer (SQLite)
```

- **Runaway guard.** TTS streaming stops a chunk if the generated audio runs far longer than the text's expected length.
- **File fallback.** If streaming fails before playback starts, the chunk can be synthesized as a whole file instead. Once playback has started, there is no fallback for that chunk.

### 3.4 Post-stream pipeline

1. When the stream recording is saved, the recording writer **arms** the post-stream sweeper with the recording time. The sweeper uses that return value directly and does not read a shared database.
2. After a delay, the sweeper starts the post-stream systemd unit periodically, for a bounded number of attempts. The start is non-blocking. A catch-up check also runs shortly after the bot starts.
3. The worker takes a dedicated lock and reads an activation flag. **If the flag cannot be read, it does not run.** It then finds backlog sessions and reads receipts.
4. It runs a capability preflight, then dispatches the registered actions:
   - event extraction;
   - YouTube description;
   - thumbnail;
   - Shorts;
   - highlight video;
   - Shorts analytics;
   - reach reports.

   All YouTube operations are serialized by a named lock.
5. Each action's receipt is in one of these states: `pending`, `success`, `retryable_failure`, `terminal_failure`, or `retry_budget_exhausted`. A `blocked` or `resource_busy` result does not use up a retry.
6. The worker prints exactly one machine-readable result line and exits with a status code. The code distinguishes three outcomes: done, work remaining, and no capability available.

## 4. Local vs Cloud LLM

| Task | Where | When |
|---|---|---|
| All live speech: quick takes, deep dives, chat replies, monologues | **Local** | During the stream |
| News importance scoring | **Local** | During the stream |
| Memory summarisation and stream digests | **Local** | During or after the stream |
| Pre-stream editorial: topic order and program labels | **Cloud** | **Before the stream** |
| Post-stream reflection and observation extraction | **Cloud** | After the stream |
| Shorts clip scoring | **Cloud** | After the stream |
| Offline evaluation and LLM-as-judge | **Cloud** | Offline only |

**The boundary rule.** A cloud LLM never runs inside the live speech loop, and it never writes a line spoken in the live stream.

- **Before the stream.** The editorial step runs only in the preparation phase. If it fails or its output runs out, topic selection falls back to the local queue.
- **What the cloud may supply.** Planning inputs only: topic order, short labels, and membership judgments. They reach the deep-dive prompt through a fixed template. Opinions, self-description, and every line spoken in the live stream come from the local model.
- **After the stream.** Cloud models score and analyse what was already broadcast. They do not change the live show.
- **Provider independence.** The live path has no dependency on any cloud provider. Cloud tasks are offline and can be moved to another provider without touching the live system.

## 5. State and Memory

| Store | Role | Written by | Read by |
|---|---|---|---|
| **Raw buffer** (SQLite) | Rolling log of every generated exchange, including the raw response | Response pipeline | Batch summariser, audits |
| **History** (SQLite) | Chat history | Response pipeline | Prompt assembly |
| **Mem0** | Semantic memory: extracted facts and preferences, per-viewer profiles kept apart by platform, and a self-memory namespace for each character | Batch summariser. Self-memory is written by a separate step that the operator runs. | Retrieval for prompts |
| **Chroma** | Episodic text, past opinions, persona knowledge for each character | Batch summariser, opinion store | Deep-dive retrieval, persona retrieval (Chiruru and Riria) |
| **Curated knowledge** (JSON) | Hand-maintained public facts, matched by rule | Operator | Prompt assembly, with a small cap on entries and characters |
| **Opinion supersession** (SQLite) | Marks past opinions that later ones contradict, so they are not injected again | Opinion store and admin tools | Past-opinion supply |

**Per-character separation, as implemented.**

| Data | Separated per character? | How |
|---|---|---|
| Persona definition | Yes | Separate definitions, chosen by the character profile |
| Runtime state | Yes | Chiruru uses a persona state. Riria keeps an in-memory session history. |
| Raw buffer | Tagged | A character column limited to the two characters. Riria rows link to Chiruru's line. |
| Stream memory (Chroma) | Yes, on read | The character is written into metadata at batch time. Retrieval filters results by character. |
| Chat history | Yes, on read | Assistant turns are filtered by character |
| Self-memory (Mem0) | Yes | A separate namespace per character |
| General memory (Mem0) | **No** | One namespace. The character is stored only as metadata. |
| Persona knowledge (Chroma) | **No** | Shared collection. Riria's entries are distinguished only by a label. |
| Past opinions | **No** | Not tagged by character |

Q&A-corner lines are kept as records but are excluded from both characters' memory writes.

**Context supply rules.**

- **Composition order.** Each prompt is built in the same layer order for both characters:
  1. persona core;
  2. task and platform blocks;
  3. context sections: retrieval, self-memory, and program context;
  4. history.

- **Token budget.** Each prompt has a fixed budget split between the system prompt, the user turn, and history. When the prompt exceeds the budget, each section is trimmed separately.
- **What reaches Chroma.** Only exchanges that pass write filters are stored. The filters are multi-turn exchanges, important external information, strong emotion, or long responses.
- **Past opinions.** Only a fixed-length head of each past opinion is injected, after normalisation and sanitising. The contradiction check uses the same head, so it judges exactly what the model would see.
- **Viewer memory.** Returning viewers are labelled as such. **If the memory lookup fails, the viewer is not treated as new.** This prevents the "nice to meet you" mistake with regulars.

## 6. Failure Handling

| Failure | Behaviour | Mode |
|---|---|---|
| **LLM call is slow** | Each call has a timeout that is kept below the queue's own outer limit | Bounded |
| **Inference server stalls** | The recovery service reloads the model after repeated timeouts are confirmed by a probe. It acts only on a successful state read, waits a minimum interval between reloads, has a daily cap, and doesn't repeat notifications when the state hasn't changed. | Self-healing, rate-limited |
| **Server rejects extended sampling options** | The client retries without them | Graceful degradation |
| **One utterance fails in TTS or playback** | It is logged and skipped. The playback queue keeps going. | Fail-open |
| **Audio device open or close hangs** | These calls run on a dedicated executor, and close has a time limit | Isolated |
| **News scoring fails** | The item gets a neutral low score and stays a candidate | Fail-open |
| **Editorial plan is missing or used up** | Selection falls back to the normal queue | Fail-open |
| **Riria output fails validation** | The validator retries with an instruction built from the violations. After the retries it uses a fallback line. Every attempt is recorded. | Bounded retry, then fallback |
| **Output matches a persona-integrity pattern** | The response is replaced by the speaking character's fallback line | Fail-safe |
| **Observation or shadow store write fails** | A warning is logged and the show continues | Fail-open |
| **Post-stream activation flag unreadable** | The worker does not run | Fail-closed |
| **Post-stream action fails** | Its receipt moves to a retry or terminal state. The sweeper tries again later, within a retry budget. | Bounded retry |
| **Bot crashes mid-stream** | A stale streaming flag is detected at the next start and passed to the next stream's start logic. systemd restarts the bot, with a restart cap. | Recover on restart |

## 7. Observability

- **Generation log.** Every exchange is stored in the raw buffer with its raw response and output source. Offline audit scripts use it to find prompt keys with high warning rates and the worst samples.
- **Speech timing.** Subtitle show and hide events, keyed by utterance, are logged as the source of truth for when each line was actually on screen. Generation timestamps are not used for this.
- **LLM health.** Every LLM call records an outcome (ok or timeout), which feeds the recovery service. Outgoing request payloads can be recorded, so the prompt the model actually received can be compared with what was intended.
- **Prompt A/B configuration.** Variants live in a config file that is read on every request, so changing the traffic split needs no restart. Prompt text files are cached, so changing the text does need a restart.
- **Character consistency.**
  - **Live checks.** These are rule-based. Riria's output is checked against a set of role and format rules. Every response passes a persona-integrity pattern check.
  - **Provenance.** Attempts, violations, and fallback use are recorded.
  - **Shadow check.** An embedding-similarity check runs in shadow mode. It is logged and never blocks a response.
  - **Offline checks.** The prompt validator checks persona coverage, and post-stream observation and weekly reviews follow persona tracks.
- **Offline processes.** These are run by hand, not as part of the live system:
  - **Experiment records.** Pre-registered offline experiments are logged with preflight and scope-audit tooling.
  - **Decision records.** Architecture choices are recorded as decision records.
  - **Work registry.** A registry tracks open work.
  - **Weekly review.** A weekly review covers prompt quality.
  - **Post-stream audio QA.** Speech recognition checks recorded speech for cut-offs, repetition, and misreadings. It runs after the stream, never live.

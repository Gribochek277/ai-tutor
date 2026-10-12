> Sanitized mirror of Forgejo `serhii/ai-tutor`. Source code is not published here.
>
> Commit texts: `commits/`. Need the code? Email: sergeyalpatov1@gmail.com
> Source: Forgejo `serhii/ai-tutor` | Synced: 2026-10-12T11:11:31Z

---

# Study Mode — a personal tutor for the Pi coding agent

A Pi extension that turns the agent into a one-on-one tutor. The model teaches
and grades; a small engine decides **what comes next** from a learner model
that lasts across sessions.

```
/study start "Python generators" --level beginner
```

## How a sitting goes

```
warm-up        2-3 spaced-repetition questions that are due (any topic, interleaved)
per concept    probe → teach (worked example) → faded practice → solo practice → mastered
wrap-up        after the sitting time: exit ticket + "next review in N days"
```

- **Probe first.** If the learner already knows a concept, the explanation is
  skipped (expertise reversal) and they go straight to practice.
- **Mastery learning.** A concept is mastered when the BKT estimate is ≥ 0.9
  *and* the learner has answered two independent questions in a row without
  hints. Answers reached with hints count as partial.
- **Spaced repetition.** Mastered concepts get an FSRS-4.5 card (default
  parameters, 90 % target retention). Due cards come back as warm-up questions
  in later sittings, interleaved across topics. Forgotten concepts drop back
  to practice.
- **Misconceptions.** The tutor records the learner's specific wrong belief.
  It is shown again when the concept comes back and is cleared after two
  correct answers. Errors made with confidence 3 ("sure") come back sooner
  (hypercorrection).
- **Difficulty.** Based on accuracy over the last 10 graded answers, aiming
  for ~85 % success: above 90 % → harder, below 70 % → easier.
- **Personal examples.** A learner profile (language, background, interests)
  is collected once during onboarding and used for examples in every topic.
- **Concept map.** Generated once per topic by a single model call outside the
  conversation (`id | title | prerequisites` lines). Topics are taught in
  prerequisite order. Edit the map with `/study plan`; progress is kept for
  ids that stay the same. If generation fails, a heuristic skeleton is used.

## Keeping the context small

The extension is built for local models, where context size and prompt
re-processing matter more than the price per token.

| What | How |
|---|---|
| Stable system prompt | Tutor rules never change during a session, so the provider prompt cache or the llama.cpp KV cache keeps the prefix. The old version rewrote the system prompt every turn. |
| Lean mode (default) | Replaces the coding-agent prompt and turns off all tools, AGENTS.md and skills. The first turn is about 700 input tokens. `/study tools full` brings the coding agent back with a tutor section on top. |
| One hidden note per turn | About 100–200 tokens: the current item, the learner profile, and what to do if the answer is correct or wrong. Only the newest note stays in context. |
| Context window | Learner state lives in the engine, so only the last turns are needed. When 14 turns pile up, the context is cut back to 6 at once. The cut point moves rarely, so the cache stays valid between cuts. Messages from before `/study start` are dropped. |
| No tools for bookkeeping | The tutor reports with one hidden line at the end of a reply. A tool call would cost an extra model round-trip. |

### The protocol

```
[[study kc=yield-keyword r=wrong h=1 c=3 m="thinks yield returns a list"]]
[[profile lang=ru bg="C# backend dev" interests="games"]]
```

`r` is `correct|partial|wrong`, `h` is the number of hints given, `c` is the
learner's confidence (1–3), `m` is the misconception. The parser is lenient
(`kc: x`, `r=right`, tags wrapped in code fences). Tags are stripped from the
stored message, so the learner never sees them and they never come back into
the context. The note ends with a ready-made grade line for the current
concept, because small models follow the most recent instruction best.

## Commands

```
/study start <topic> [--goal ..] [--level ..] [--tools lean|full]   start or continue a topic
/study start                      continue the last topic
/study stop | pause | resume      pause = normal agent until resume
/study learn | review | quiz      review = due cards from all topics; quiz = 5 questions, no hints
/study skip                       "I already know this" → mark the current concept mastered
/study status                     concepts with step, mastery, memory, misconceptions
/study stats                      all topics and the next review
/study plan                       view/edit the concept map
/study profile                    edit language / background / interests / minutes per sitting
/study tools lean|full
/study reset                      clear progress on the current topic
```

## Storage

- **Learner model:** `~/.pi/agent/study/learner.json` (override with
  `STUDY_HOME`). Holds the profile, topics, concepts, BKT, FSRS cards and
  misconceptions. It is shared by all sessions. Learning that happened stays
  recorded even if you branch the conversation.
- **Session state:** a `study-session` custom entry (current item, sitting
  timer, context floor, tool mode). It is rebuilt from the active branch on
  `session_start` / `session_tree`.

The v1 file (`learners/default.json`) is not migrated: it only held one
mastery number per topic.

## Layout

```
index.ts            wiring: hooks, /study command, concept-map generation
src/engine.ts       all learning decisions (deep module, pure)
src/memory.ts       BKT + FSRS-4.5
src/prompt.ts       static tutor rules + per-turn note
src/protocol.ts     [[study]] / [[profile]] parser and stripper
src/context.ts      note de-duplication + context window with hysteresis
src/conceptMap.ts   map prompt, parser, editor format, fallback
src/storage.ts      learner.json (atomic writes)
src/ui.ts           widget, status, reports
src/types.ts        domain types
smoke.mjs           pure-logic tests
harness.mjs         mock-Pi end-to-end wiring test
```

## Tests

```
JITI=<pi>/node_modules/jiti/lib/jiti-cli.mjs
node $JITI smoke.mjs
node $JITI harness.mjs
```

For type checking, link the Pi packages into `node_modules/` (ignored by git):
`@earendil-works/{pi-coding-agent,pi-ai,pi-tui,pi-agent-core}` and
`@types/node` from Pi's own `node_modules`, then run `npx -p typescript tsc`.

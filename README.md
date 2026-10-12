> Sanitized mirror of Forgejo `serhii/ai-tutor`. Source code is not published here.
>
> Commit texts: `commits/`. Need the code? Email: sergeyalpatov1@gmail.com
> Source: Forgejo `serhii/ai-tutor` | Synced: 2026-10-12T11:45:11Z

---

# Study Mode — a personal tutor for the Pi coding agent

A Pi extension that turns the agent into a one-on-one tutor. The model teaches
and grades. A small engine decides **what comes next** from a learner model
that is kept across sessions and topics.

```
/study track "Become an ML engineer, up to LLMs" --level intermediate
/study start "Python generators" --level beginner
```

## Tracks: long goals made of modules

`/study track "<goal>"` asks the model for a curriculum: 6–14 **modules** with
prerequisites and a **scope** per module ("vectors, matmul, norms; skip
proofs"), so only what the goal needs is covered. In the TUI the plan opens in
an editor before anything starts, so you can reorder, delete or add modules.

```
Track: ML engineer → LLM
 ├─ ML foundations            scope: supervised/unsupervised, splits, overfitting
 ├─ Python for data science   scope: numpy, pandas, sklearn; skip web
 ├─ Linear algebra for ML     needs: —
 ├─ Calculus & optimization   needs: linear-algebra
 ├─ Neural networks           needs: calculus, python
 ├─ …
 └─ LLM fine-tuning / deployment
```

- **Lazy modules.** A module's concept map, capstone project and materials are
  generated when you enter it, not upfront. The prompt includes the track
  goal, the module scope, your background and the concepts you already know
  from other modules.
- **Placement check.** A new module opens with up to 4 quick questions spread
  over its concepts. A known concept skips the explanation. If all answers are
  right, the remaining concepts need just one check each. Two misses in a row
  end the test: the module is new to you, so teaching starts.
- **Cross-module prerequisites.** A concept can depend on `module/concept`
  (e.g. `linear-algebra/matmul`). If that concept is not mastered or has faded
  from memory, a short **bridge** refresher comes first, at most once per
  sitting.
- **Capstone project.** It comes at the end of each module, e.g. "Iris
  pipeline: split, train 2–3 models, cross-validate, compare metrics". The
  tutor guides and reviews, and does not write it for you. In lean mode,
  coding tools are switched on for project turns only (`project_tools: auto`).
- **Moving on.** When every concept is mastered and the project is done, the
  next unlocked module starts automatically.
- Topics are global. The same module (say `python`) in two tracks shares its
  progress.

## Learning materials

When a module starts (or on `/study materials refresh`):

1. The model suggests 3–6 well-known resources (official docs, classic
   courses and books, well-known video series) tagged with the concepts they
   cover, plus 3–5 search queries in English and your language.
2. Every suggested link is checked over HTTP. Dead or invented links lose the
   URL and turn into "search for this title".
3. The first queries are run through a web search: SearXNG if `searxng_url`
   is set in `/study profile`, otherwise DuckDuckGo's HTML endpoint (no key
   needed). The top hits are added.
4. If everything fails you still get the search queries.

The tutor presents the list when the module starts and points to the
matching material when it teaches a concept. `/study materials` opens the list
in an editor. Add your own lines (bare URLs work too), and an edited list
is never overwritten by automatic refreshes.

## How a sitting goes

```
warm-up        2-3 spaced-repetition questions that are due (any topic, interleaved)
per module     placement → concepts → capstone project
per concept    probe → teach (worked example) → faded practice → solo practice → mastered
wrap-up        after the sitting time: exit ticket + "next review in N days"
```

- **Mastery learning.** A concept is mastered when the BKT estimate is ≥ 0.9
  *and* the learner has answered two independent questions in a row without
  hints. Answers reached with hints count as partial.
- **Spaced repetition.** Mastered concepts get an FSRS-4.5 card (default
  parameters, 90 % target retention). Forgotten concepts drop back to practice.
- **Misconceptions.** The tutor records the learner's specific wrong belief.
  It is shown again when the concept comes back and is cleared after two
  correct answers. Confident errors come back sooner (hypercorrection).
- **Difficulty.** Based on accuracy over the last 10 graded answers, aiming
  for ~85 % success.
- **Personal examples.** A learner profile (language, background, interests)
  is collected once and used for examples in every topic.

## Grading protocol

The tutor starts its reply with one hidden line:

```
[[study kc=yield-keyword r=wrong h=1 c=3 m="thinks yield returns a list"]]
[[profile lang=ru bg="C# backend dev" interests="games"]]
```

- The note for the turn ends with a ready-made grade line for the current
  concept: small models follow the most recent instruction best, and deciding
  the grade first is more reliable than remembering it after a long answer.
- "I don't know" is graded `wrong`, so placement keeps moving.
- **Fallback grader.** If the learner answered but the tutor forgot the line,
  one short separate call grades the exchange. It sees the question, the
  answer and the tutor's response, and returns NONE when the tutor had to
  repeat the question. Every grade is logged as a `study-grade` entry with its
  source (`tutor` or `fallback`).
- Tags are stripped from stored messages and hidden in the TUI while they
  stream in (markdown transformer).

## Keeping the context small

The extension is built for local models, where context size and prompt
re-processing matter more than the price per token.

| What | How |
|---|---|
| Stable system prompt | Tutor rules never change during a session, so the provider prompt cache or the llama.cpp KV cache keeps the prefix. |
| Lean mode (default) | Replaces the coding-agent prompt and turns off all tools, AGENTS.md and skills. The first turn is about 700 input tokens. `/study tools full` brings the coding agent back with a tutor section on top. |
| One hidden note per turn | About 100–250 tokens: where you are, the learner profile, what to do on a correct or wrong answer. Only the newest note stays in context. |
| Context window | Learner state lives in the engine, so only the last turns are needed. When 14 turns pile up, the context is cut back to 6 at once. The cut point moves rarely, so the cache stays valid between cuts. |
| Plans and materials outside the chat | Separate one-off calls; their output goes into the learner file, not into the conversation. |

## Commands

```
/study track "<goal>" [--level ..]     plan a long path (modules) and start it
/study track                          show the track: modules, progress, what's next
/study track edit                     edit modules / order / prerequisites / scope
/study module <id>                    jump to a module
/study start <topic> [--goal ..] [--level ..] [--tools lean|full]   a single topic
/study start                          continue where you left off
/study materials [refresh]            materials of the current module (edit to add your own)
/study stop | pause | resume          pause = normal agent until resume
/study learn | review | quiz          review = due cards from all topics; quiz = 5 questions, no hints
/study skip                           "I already know this" → mark the concept (or project) done
/study status                         concepts with step, mastery, memory, misconceptions, project
/study stats                          all topics and the next review
/study plan                           view/edit the current module's concept map (+ project line)
/study profile                        language / background / interests / minutes per sitting,
                                      searxng_url, auto_materials, project_tools
/study tools lean|full
/study reset                          clear progress on the current module
```

## Storage

- **Learner model:** `~/.pi/agent/study/learner.json` (override with
  `STUDY_HOME`). Holds the profile, settings, tracks, topics, concepts, BKT,
  FSRS cards, misconceptions, projects and materials. It is shared by all
  sessions.
- **Session state:** a `study-session` custom entry (track, current module and
  item, sitting timer, context floor, tool mode). It is rebuilt from the
  active branch on `session_start` / `session_tree`.

## Layout

```
index.ts            wiring: hooks, /study command, async module setup, fallback grader
src/engine.ts       all learning decisions (deep module, pure): tracks, placement,
                    bridges, projects, concept steps, reviews
src/memory.ts       BKT + FSRS-4.5
src/prompt.ts       static tutor rules + per-turn note
src/protocol.ts     [[study]] / [[profile]] parser, stripper, display filter, grader prompt
src/conceptMap.ts   track & module prompts, parsers, editor formats, fallback map
src/materials.ts    material suggestions, link checks, SearXNG / DuckDuckGo search
src/context.ts      note de-duplication + context window with hysteresis
src/storage.ts      learner.json (atomic writes)
src/ui.ts           widget, status, track overview
src/types.ts        domain types
smoke.mjs           pure-logic tests
harness.mjs         mock-Pi end-to-end test (fake model + fake fetch)
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

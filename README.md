> Sanitized mirror of Forgejo `serhii/ai-tutor`. Source code is not published here.
>
> Commit texts: `commits/`. Need the code? Email: sergeyalpatov1@gmail.com
> Source: Forgejo `serhii/ai-tutor` | Synced: 2026-10-05T01:37:09Z

---

# Study Mode — a tutoring overlay for the Pi coding agent

**AI-Tutor**: a Pi extension that turns the ordinary agent loop into a stateful
tutor. It implements the architecture from `plan.md`:
`session → learner state → pedagogical policy → prompt → LLM → evaluation →
state update`.

This is the standalone repo for the extension (created from scratch). The
extension entry point is `index.ts` (registered via `package.json` →
`pi.extensions`). To use it:

- **as a Pi package:** point Pi's settings `extensions`/`packages` at this repo
  (or `pi install` it), or
- **project-local:** copy/symlink this directory to `<project>/.pi/extensions/`
  so it loads only inside that project.

## What it does

When you start a study session, Study Mode becomes an **orchestration layer**
over the normal agent loop. The LLM stays the LLM; the extension owns *how*
teaching happens:

```
User input
   │
   ▼
before_agent_start ──► inject compact per-turn tutor prompt (canonical state)
   │
   ▼
        LLM (tutor) ──► reply to the learner
   │                        │
   │                        └─► optional ```study JSON block (structured protocol)
   ▼
turn_end ──► parse block ──► update learner state (known/weak/mistakes/difficulty)
   │
   ▼
widget + status footer reflect the session
```

Key properties:

- **State machine** — `diagnostic → explain → guided-practice →
  independent-practice → assessment → reflection → complete`.
- **Pedagogical engine** — strategy is chosen per phase and difficulty
  (Socratic, Explain, Guided Practice, Retrieval, Scaffolding, Challenge).
- **Per-turn compact prompts** — no giant system prompt. Each turn gets a
  small, focused tutor prompt that injects the canonical learner state, so the
  engine never depends on conversation history (compaction-safe).
- **Structured tutor protocol** — the model may end its reply with a small
  fenced ````study```` JSON block. The engine parses and *validates* it; the
  model can never corrupt state directly.
- **Custom tools** — `study_get_state`, `study_record_observation`,
  `study_advance_phase`, `study_get_hint`, `study_complete`. The engine
  validates every change.
- **Persistence** — session state is stored as a custom session entry
  (survives reload/resume/compaction); long-term learner mastery is saved to
  `~/.pi/agent/study/learners/default.json`.
- **Adaptive difficulty** — three clean turns with no reported mistakes nudge
  difficulty up; major mistakes lower it.

## Commands

```
/study start <topic> [--goal "..."] [--level "..."]   start a session
/study stop                                            stop + save
/study pause / resume                                  pause / resume
/study status                                          show full state
/study plan                                            show the learning plan
/study mode <teach|practice|review|quiz|conversation>  switch mode
/study reset                                           reset learner state
/study complete                                        mark complete
/study review                                          long-term mastery review
```

Examples:

```
/study start Japanese N5
/study start "C# async/await" --goal "understand continuations" --level intermediate
/study start "linear algebra"
```

## How it feels

Normal Pi:

```
you:   объясни async/await
agent: [dumps an explanation]
```

Study Mode:

```
you:   /study start "C# async/await"
you:   научи меня async/await
agent: (diagnostic → asks questions to gauge your level)
       (explain → layered intuition → example → formal)
       (guided-practice → hint → your attempt → correction)
       (independent-practice → you solve → retrieval practice)
       (assessment → checkpoint → reflection)
```

The top widget shows `Phase`, `Progress`, `Difficulty` and the current plan
step. Because the learner state is injected every turn, long sessions survive
compaction without losing context.

## Layout

```
.pi/extensions/study-mode/
├── index.ts          # wiring: commands, tools, event hooks
├── src/
│   ├── types.ts      # StudySession, LearnerState, phases, actions
│   ├── store.ts      # state machine + persistence (session + long-term)
│   ├── pedagogy.ts   # strategy selection + difficulty
│   ├── prompt.ts     # compact per-turn tutor prompt
│   ├── parser.ts     # structured ```study JSON → StudyAction[]
│   ├── tools.ts      # custom study tools
│   └── ui.ts         # widget + status footer
├── smoke.mjs         # pure-logic tests  (node jiti smoke.mjs)
├── harness.mjs       # mock-Pi wiring test (node jiti harness.mjs)
└── package.json
```

## Tests

```
node <pi>/node_modules/.bin/jiti smoke.mjs
node <pi>/node_modules/.bin/jiti harness.mjs
```

(`node_modules` here are symlinks to the global Pi packages so type-checking
and tests resolve — Pi itself resolves these at runtime via jiti.)

## Notes / known simplifications

- Each state change appends a custom session entry. Custom entries do **not**
  consume LLM context, so this is safe, but it does grow the session file.
- The learning plan is a heuristic skeleton; the tutor model refines it in
  practice.
- The structured block is optional; if the model omits it, the engine only
  nudges difficulty on clean turns and otherwise leaves state unchanged.

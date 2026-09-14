# `/research-implement-feature` — User Guide

> The build skill for *"just implement X for me."* It stays autonomous and stops
> being silent: every decision your request left open gets written down before
> the code that depends on it, and a different model family goes looking for the
> ones it forgot to write down.

---

## 1. Install & invoke

```bash
cp -r skills/research-implement-feature ~/.claude/skills/
```

```
/research-implement-feature "<what to build>"
                            [— effort: lite|balanced|max|beast]
                            [— ask: never|semantic]
                            [— assurance: draft|submission]
                            [— base repo: <git url>]
```

Minimum viable invocation — no plan file, no setup, no prior stage:

```
/research-implement-feature "a KV-cache eviction policy I can swap into our decoding loop, plus a script that measures hit rate against the full-cache baseline"
```

---

## 2. What it does, in order

```
Phase 0  read request → SPEC.md → open the assumption ledger → reuse survey
Phase 1  FEATURE_LADDER.md: F0 spine + F1..Fn features, each with ONE check command
Phase 2  build F0 (walking skeleton, placeholders inside) → must exit 0
Phase 3  per rung: implement → its check → ALL earlier checks → bijection check → commit
Phase 4  cross-model silent-assumption sweep → SILENT_ASSUMPTION_SWEEP.json
Phase 5  report, semantic assumptions first
```

Two rules the whole skill hangs on:

1. **Declare before you act.** A ledger row exists before the code that depends
   on it. A ledger reconstructed at the end is a changelog, and it omits exactly
   the assumptions the author stopped noticing.
2. **Spine before features.** F0 is the thinnest real-entry-point-to-real-artifact
   path. Nothing gets added until it runs.

---

## 3. Does it stop and ask me things?

That is the `ASK` axis, and it has three modes. **Default is no.**

| `— ask:` | Asks about | Blocking? | Use when |
|---|---|---|---|
| `never` *(default)* | nothing — decides, logs a ledger row, keeps going | no | unattended / overnight / `/loop`, or you want it executed not discussed |
| `semantic` | `semantic`-class ambiguities only | yes, at batch points | you trust the small calls but want a say in what the results will mean |
| **`all`** | **every ambiguity, any class** | yes, at batch points | **confirm mode** — you're at the keyboard and want the build to match what's in your head |

```
/research-implement-feature "..." — ask: all
```

Under `ask: all`, **nothing under-determined gets written until you've answered
it.** The rule is exact: a ledger row is the unit of ambiguity, so "what gets
asked" and "what gets a ledger row" are the same set — `ASK` just picks which
classes are put to you first instead of decided for you.

### Where it asks (it does not interrupt at random)

| Batch point | When | About |
|---|---|---|
| **B0** | end of Phase 0, before the ladder is built | everything ambiguous in your request itself |
| **B1..Bn** | start of each rung, before that rung's code | what that rung raises and B0 couldn't have seen |
| **Bd** | a debugging fork that is itself an ambiguity | "shapes don't match — pad left or right?" — asked *before* the fix, not after |

### What a batch looks like

- **Never one question at a time.** Questions are collected and asked together.
- **At most 4 per call**, ordered by blast radius (`semantic` → `interface` →
  `local` → `cosmetic`).
- **Cheap classes get bundled.** Under `ask: all`, naming/logging/layout collapse
  into a single "take repo conventions throughout?" question. Four questions
  about variable names is a bug in the skill, not diligence.
- **The default is always option 1**, labelled `(default)`, with rejected
  alternatives beside it — accepting everything as-is is one keystroke and
  produces exactly what `ask: never` would have.
- **Each option shows its reversal cost**, so you can see which forks are cheap
  to get wrong.
- **"You decide"** falls back to the default, records `Source: default,
  deferred_to_author: true`, and is never re-asked.
- **A rung with no ambiguities asks nothing** — no empty checkpoint announcements.

### Two things it will not do

- **It will not silently downgrade `ask: all` to `never`** when nobody is there
  to answer. It says the run is unattended and stops. A build that took every
  default and reported it as confirmed is worse than no build.
- **It will not treat your answer as a correctness guarantee.** Answering a
  question makes a row *declared*, not *right*. Phase 4's cross-model sweep still
  runs, and still hunts for what neither you nor the skill noticed was a choice.

> **Don't combine `ask: semantic` / `ask: all` with `/loop` or overnight
> cadence.** A blocking gate on an unattended run is a run that did nothing.

**It also stops without asking** when a rung exhausts its fix budget: it writes
`BLOCKERS.md`, marks the rung 🚧, and halts the ladder there rather than skipping
ahead to an easier rung to pad the progress bar.

## 4. The assumption ledger

`implement-stage/ASSUMPTIONS.md` is the artifact that makes autonomy acceptable.

| ID | Under-determined by the request | Chosen | Rejected | Class | Source | Reversal cost | Override |
|----|--------------------------------|--------|----------|-------|--------|---------------|----------|
| A-001 | "measure hit rate" — against which workload? | ShareGPT 500-prompt sample | full ShareGPT (40 min/run), synthetic (unrepresentative) | `semantic` | `user` | 1 line, `configs/bench.yaml` | `— assume: A-001=full` |
| A-002 | no eviction granularity named | per-token | per-block (needs paged attention we don't have) | `interface` | `user` | rewrite `evict()`, ~30 min | `— assume: A-002=block` |
| A-003 | no logging convention given | reuse repo's `structlog` setup | stdlib logging | `cosmetic` | `default` | trivial | — |

### The `Source` column — who decided each row

| `Source` | Means |
|---|---|
| `user` | you were asked at a batch point and chose this |
| `default` | the skill chose it — either `ASK` didn't cover that class, or you said "you decide" |
| `sweep` | Phase 4 caught it undeclared and it was added afterwards (always also `retro: true`) |

This is how the ledger stays readable across all three modes. Under `ask: all`,
a finished run should have few or no `default` rows — **and the ones left over are
the most interesting rows in the file.** They are the ambiguities the skill never
recognised as ambiguities, so it never thought to ask you. The final report names
them individually, and Phase 4 reads them first.

### The four classes

| Class | Means | You will see it… |
|---|---|---|
| `cosmetic` | naming, layout, log format | counted in the report, in the ledger |
| `local` | one module's internals, invisible at its interface | counted in the report, in the ledger |
| `interface` | changes call sites, configs, artifact schemas | **named** in the report |
| `semantic` | **changes what a result would MEAN** — metric definition, eval split, normalization, what counts as a baseline | **own block, top of the report, never collapsed to a count.** The only class `ask: semantic` gates on |

`ask: all` gates on all four; the classes then set the order questions come in
and which get bundled together.

The `semantic` class is the point of the skill. An undeclared `cosmetic`
assumption costs you a rename. An undeclared `semantic` assumption is how an
implementation quietly decides what your paper will later claim.

**Read the `Reversal cost` column.** That column is what makes "don't ask, just
decide" a fair deal: you learn in one glance whether an assumption you dislike
costs one line or half a day. The skill is also told to prefer the *easily
reversible* default when choices are otherwise close.

### Code markers and the bijection check

Every site acting on a row carries a marker:

```python
# ASSUMPTION[A-001] bench workload = ShareGPT 500-prompt sample (ledger: implement-stage/ASSUMPTIONS.md)
```

At every rung, a deterministic `grep` + `comm` check runs: a marker with no
ledger row halts the rung. This is shell, not a model's opinion — "I believe the
ledger is complete" is never accepted in its place. (A ledger row with *no* code
site is fine; say `no code site` in the row.)

---

## 5. Placeholders

F0 may fake things. It may not hide that it faked them.

```python
# PLACEHOLDER[P-002] returns a fixed 0.5; real scorer lands at rung F3
```

- A placeholder that produces a **number** must never reach a path that reads like
  a result — it goes to `*_smoke.json`, or gets a `PLACEHOLDER_` prefix.
- A rung cannot be marked green while a placeholder it was supposed to retire is
  still live.
- Every survivor is listed in the final report, with the rung that would retire it.

---

## 6. The cross-model sweep (Phase 4)

The ledger only holds what the implementer *noticed* assuming. So the spec, the
ledger, and the **raw diff** — no Claude-written summary — go to a reviewer from a
different model family (`gpt-6-astra` at `xhigh`, per `reviewer-routing.md`), with
one question: *what does the code decide that the request didn't determine and the
ledger doesn't declare?*

Output lands verbatim in `implement-stage/SILENT_ASSUMPTION_SWEEP.json`:

```json
{"undeclared": [...], "stale_rows": [...], "semantic_undeclared": 2, "verdict": "gaps"}
```

Findings are added to the ledger as `retro: true` rows, then it re-sweeps (1–3
rounds by effort).

| `assurance` | Effect of `semantic_undeclared > 0` |
|---|---|
| `draft` (from `effort: lite`/`balanced`) | reported, non-blocking |
| `submission` (from `effort: max`/`beast`) | **blocks the final report** until those rows are in the ledger |

If Codex is unavailable, the run continues and records `SWEEP_UNAVAILABLE`. It
will **not** substitute a second Claude pass and call it a sweep — N agreeing
same-family reads is one opinion with error bars, not a second opinion.

---

## 7. Files it writes

All under `implement-stage/`:

| File | What it's for |
|---|---|
| `SPEC.md` | your request restated as target / inputs / outputs / success command / scope cuts |
| `ASSUMPTIONS.md` | **the ledger — read this one** |
| `FEATURE_LADDER.md` | the rungs, their check commands, tier (MUST/SHOULD/DEFERRED), status |
| `BUILD_LOG.md` | per rung: command, exit code, artifact, fix attempts used |
| `SILENT_ASSUMPTION_SWEEP.json` | the cross-model verdict — the receipt that the acquittal was external |
| `DEFERRED.md` | rungs cut from this run, and why |
| `BLOCKERS.md` | only on budget exhaustion: what failed, what was tried, smallest next step |

It does **not** create a `MANIFEST.md` — a run this size is under the threshold.

---

## 8. Effort levels

| | `lite` | `balanced` *(default)* | `max` | `beast` |
|---|---|---|---|---|
| Rung budget | 3 | 5 | 8 | 12 |
| Fix attempts per rung | 3 | 5 | 8 | 12 |
| Sweep rounds | 1 | 2 | 2 | 3 |
| Reuse survey | local grep | + ecosystem | + reference impl | + fetch & diff reference impl |
| Implied `assurance` | draft | draft | submission | submission |

`effort` never lowers the reviewer tier — a hard invariant of the effort contract.

---

## 9. Reading the final report

It prints in this order, and the order is the point:

1. **What runs now** — the success command, its exit code, artifacts on disk
2. **⚠️ Semantic assumptions** — in full. If you read one thing, read this
3. **Ladder status** — green / blocked / deferred, deferred ones *named*
4. **Live placeholders** — and the rung that would retire each
5. **Sweep outcome** — how many undeclared assumptions a cold cross-model read found
6. **Other assumptions** — the mode and the split (*"`ask: all` — 9 rows, 7
   `Source: user`, 2 `Source: default`"*), interface named, local/cosmetic counted
7. **Next** — what to run after

**What the report will never say:** that the implementation is correct, that the
method works, or that a number means anything. A green ladder is an execution
fact — "the spine runs and every MUST rung's check passed." Turning that into a
claim is `/experiment-audit` and `/result-to-claim`, on purpose.

---

## 10. When to use something else

| Your ask | Route |
|---|---|
| "implement X" / "build me something that does X" / "prototype then extend" | **this skill** |
| "find me a research direction and take it to a paper" | `/research-pipeline` |
| "I have `EXPERIMENT_PLAN.md` — run the campaign, deploy to GPU" | `/experiment-bridge` |
| "sweep these parameters / find the best config" | `/dse-loop` |
| "launch what's already written" | `/run-experiment` |
| "do these results support the claim?" | `/result-to-claim` |

### vs `/research-pipeline`

`/research-pipeline` answers *"what should we research?"* — it decides the
question for you. This skill answers *"build the thing I already decided on"* and
decides **nothing** without writing it down. Different input contracts, so they
are separate entry points rather than one mode flag — but they compose: a
pipeline run can delegate its build stage here and inherit the ledger.

---

## 11. FAQ

**It picked something I disagree with.**
Find the row in `ASSUMPTIONS.md`, read `Reversal cost`, and re-run with the
`Override` shown in that row. That column exists so you never have to go
archaeology-hunting in the diff.

**It stopped at F2.**
Read `BLOCKERS.md`. That's the designed behaviour — it halts at the failing rung
rather than reordering it to the end to look fuller. Fix the blocker, re-invoke,
and it continues the ladder.

**Can I run it again to add the next feature?**
Yes — that's the intended loop. Re-invoke with the next feature; it reads the
existing `FEATURE_LADDER.md` and `ASSUMPTIONS.md` and appends rather than restarting.

**Codex isn't configured.**
Everything except Phase 4 runs. The report carries `SWEEP_UNAVAILABLE`, and you
should treat the ledger as unaudited — it holds what the implementer noticed, and
nothing checked for what it didn't.

**I want to be asked about everything. Which mode?**
`— ask: all`. Nothing under-determined gets decided until you answer, questions
arrive in batches of at most 4 with your defaults pre-selected, and cheap classes
are bundled. Don't pair it with `/loop` or an overnight run — it blocks.

**Under `ask: all`, why are there still `Source: default` rows?**
Because the skill only asks about ambiguities it *noticed*. A leftover `default`
row in a covered class means it made a call without realising it was a call.
That's not a bug you should ignore — it's the most useful signal in the run, and
it's exactly what the Phase 4 sweep is pointed at. The report names every one.

**Does answering the questions mean I can skip the cross-model sweep?**
No. Your answer makes a row *declared*, not *correct*. The sweep hunts for what
neither of you noticed was a choice — that set doesn't shrink because you
answered the questions that were asked.

**Why is the ledger written *before* the code, not after?**
Because a ledger written after the fact contains the assumptions you still
remember making, which are the harmless ones. The dangerous ones are the ones you
stopped seeing as decisions — those only survive if the row is written at the
moment the choice is live.

---

## 12. See also

- [`SKILL.md`](SKILL.md) — the full contract
- [`../shared-references/acceptance-gate.md`](../shared-references/acceptance-gate.md) — why the build loop may self-terminate but may not self-acquit
- [`../shared-references/reviewer-independence.md`](../shared-references/reviewer-independence.md) — why the sweep gets paths, not summaries
- [`../shared-references/effort-contract.md`](../shared-references/effort-contract.md) — the effort / assurance axes

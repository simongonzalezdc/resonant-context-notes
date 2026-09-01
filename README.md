# Context engineering for local-model agents — measured

Notes for people building agent harnesses on their own hardware: a 27B-class
reasoning model served by llama.cpp or anything like it, driving a tool-using
loop, tokens slow, prefill slower. Everything below is an adaptation of
[KyaniteLabs/context-kit](https://github.com/KyaniteLabs/context-kit) (MIT) —
that repo is the instrument. It holds the code, the experiments, and the
rig every number was measured on. These notes restate its measured results
for harness builders, with every number's validity label intact, so you can
decide what to adopt without re-reading a whole repo.

Labels · Styles · Re-reads · Metrics · Honest measurement · Integration
laws · The honest gap · Verify it on your rig · Provenance

---

## Read the labels first

Every number in the canonical repo — and every number in these notes — carries
one of three validity labels:

- **CLEAN** — server-API-measured, or deterministic bytes/arithmetic; no known
  confound.
- **DIRECTIONAL** — small n and/or a known confound (thermal drift, single
  run, harness window). The *ordering* is claimed; the *magnitude* is soft.
- **ARITHMETIC** — deterministic math on measured inputs (bytes saved,
  tok/s to time-to-first-token).

A DIRECTIONAL number is an arrow, not a promise. It is enough to decide what
to try next on your rig; it is not enough to cite as a guaranteed win.

## Steer the style, never cap the budget

The problem: a thinking model burns most of its wall time reasoning in
patterns the task does not need — restating the question, narrating steps,
re-deriving facts it already knows. A system prompt that steers the *style*
of that internal reasoning cuts the waste without touching budgets,
quantization, or hardware.

Two styles, fusable into one:

- **caveman** — think in short telegraphic fragments, 3–8 words each; high
  confidence means decide and move.
- **ponytail** — lazy-senior-dev judgment. Before any solution, run the
  ladder: *does this need to exist at all? → does the stdlib already do it?
  → one line? → only then the minimum code that works.* Stop at the first
  rung that holds.
- **fused** — both at once. This is the default the canonical harness ships.

Measured on the canonical rig (27B reasoning model on llama.cpp, unified
memory; thinking ON, hits straight against the server API with no harness in
the path, so attribution is clean). The takeaway: about a third off both
tokens and time, with nothing lost in correctness.

| Arm | tokens/task | time/task | correct |
|---|---|---|---|
| baseline band (n=8) | 151–313 | 8.6–15.0 s | — |
| fused style (n=3) | 117–143 | 5.8–6.8 s | 15/15 |

≈ **-36% reasoning tokens, ≈ -33% task time, sustained, 15/15 correct.**

`Provenance: KyaniteLabs/context-kit README, style-steering experiment. VALIDITY: DIRECTIONAL (n=3 styled runs vs n=8 baseline band, same 5-task battery).`

Three laws if you adopt this:

1. **Style steers verbosity; the server-side budget stays the only hard
   backstop.** Nothing truncates a thought mid-stream. (The canonical rig
   also tested an estimate-then-think nudge — a pre-call size estimate that
   adjusts reasoning — and rejected it: the estimator showed zero
   discrimination between task scales.)
2. **Byte-stable position.** The style text must lead the leading system
   message, identical on every request, or you break the prompt cache. On
   local hardware that costs real seconds — see [prefill arithmetic](#prefill-pays-the-bills)
   below.
3. **Exempt lanes.** Creative work (the tokens are the product) and
   instant/no-think turns get no style.

### The cap verdict

Do not cap the thinking budget to save tokens — but do set one as a backstop,
because an uncapped model can think itself to death against the output
ceiling. The canonical repo ran the same 50 difficulty-enriched Omni-MATH
problems under no cap, a 1024-token cap, and a 512-token cap (same problems
per cell, cell order rotated, exact McNemar on paired flips):

- All three arms scored statistically indistinguishably (uncapped vs 512,
  p=0.73).
- The uncapped arm hit the output ceiling mid-think on **26 of 50** problems;
  the capped arms almost never died (1 of 50).
- Median wall time: **367 s uncapped vs 49 s at the 512 cap.**
- The hard band (difficulty >= 5) scored **16% at every budget** —
  capability-bound, not token-bound.

Refined law: **style steering stays the default for cutting verbosity; a
tight server-side budget is the backstop that also keeps the arm alive.**
The canonical harness default is now a 1024-token budget, 512 when turn speed
matters. One caveat carried from design review: the capped cells measure
*forced* early termination (a conclude-now injection), so if anything the
true cap cost is lower than measured.

`Provenance: KyaniteLabs/context-kit README, cap experiment (2026-08-18), paired Omni-MATH cells, exact McNemar.`

### The regime condition

The -36%/-33% win is real but **regime-conditional, not universal**. It was
measured where the model's thinking effort defaults to high — the regime
where a model overthinks and there is something to cut. Re-measured at low
thinking effort (n=15 per arm, interleaved, fixed fingerprinted harness), the
same fused style measured **+65% wall time** — a full inversion — because at
low effort there is nothing to cut and the style text costs more than it
saves.

So route **per session, not per turn**: apply the style only to high-effort
sessions. That also keeps the byte-stable prefix law intact, since toggling a
style per turn would break the prompt cache mid-session.

`Provenance: KyaniteLabs/context-kit README, re-baseline experiment (2026-08-15/16). VALIDITY: DIRECTIONAL (harness-lane n=15/arm, interleaved).`

### A number not to quote

A much bigger **-51% wall-time** figure appears in the canonical logs. It is
a *stack delta* (n=1; style, tool payload, and system prompt all changed at
once), valid at most as a whole-stack number and invalid as a style number.
Never cite it as style steering. The -36%/-33% above is the clean
attribution.

## Stop the re-reads

The problem: agents explore by reading whole files, then read them again, and
re-run the same commands. Every repeated byte is re-prefilled on the next
request — and re-prefill is where local agents lose real seconds (the
arithmetic is quantified below). The canonical kit attacks this in three
layers, biggest first.

1. **Symbol-level reads ("munch").** Instead of a whole-file read tool, give
   the loop a `read_symbol` tool: stdlib `ast` parses a Python file and
   returns one function or class — signature, docstring line, source span,
   optional body — instead of the entire file. On the canonical exploration
   task this replaced a 13,917 B whole-file read with a 276 B symbol read:
   **-98% tool-result bytes**, and prompt bytes fell **53.5 KB → 4.9 KB
   (-91%)** for the same task. Wall time on that task fell 48.6 s → 22.6 s —
   treat that one as direction only. A miss lists candidate symbol names, so
   the model retries cheaply instead of falling back to whole-file reads.
   Python-only; the idea generalizes (the canonical credits trace it to
   tree-sitter-based prior art).

   `Provenance: KyaniteLabs/context-kit README + kit/munch.py. Bytes: VALIDITY CLEAN (deterministic file-vs-stub, n=1). Wall time: VALIDITY DIRECTIONAL (n=1, harness window later found to distort tool calls).`

2. **mtime-validated read cache.** A repeat read of an *unchanged* file
   (`mtime_ns` + size both match) returns the cached bytes with zero
   re-execution. Three laws keep it safe: read-only tools only (never cache
   anything with side effects); the full cached bytes still flow back to the
   model (savings come at projection, layer 3 — not from truncation here);
   and validate before every hit, stamping mtime *after* execution so a file
   modified mid-read is caught next time.

3. **Projection-time dedup ("diet").** Identical repeated tool outputs
   collapse to a stub ("unchanged output — identical to the earlier result at
   event N") when history is projected into the model's view. The log keeps
   everything forever; only the view slims. Three 5 KB repeats of one command
   project to ~5.1 KB instead of 15 KB — **-66%** on that class of repetition.
   The canonical kit adds two more view passes in the same spirit: budget
   elision (oldest tool bodies slim first, a short preview survives) and
   long-session compaction (old bodies stub once the log is long; the recent
   working window stays full-fidelity).

   `Provenance: KyaniteLabs/context-kit README + kit/diet.py. VALIDITY: ARITHMETIC (deterministic bytes).`

A fourth move for big outputs: **progressive disclosure**. Tool outputs over
a size limit become a head+tail preview with the full body spilled to a
content-addressed file whose path is named in the preview, readable back in
slices. Guard it with a read-back counter: if the model re-reads spilled
bytes for more than ~15% of spills, the diet is negative-sum on that output
class — raise the limit or stop spilling it.

`Provenance: KyaniteLabs/context-kit docs/PATTERNS.md, "Progressive disclosure".`

### Prefill pays the bills

Decode gets the headlines; prefill costs the time. At the canonical rig's
~390 tok/s prefill ceiling, every 10,000 tokens you do not re-send saves
about **26 s of time-to-first-token**. That arithmetic is why the
prefix-stability law and the whole diet exist: late-turn prefill is where
local-agent pain lives.

`Provenance: KyaniteLabs/context-kit README + docs/PATTERNS.md. VALIDITY: ARITHMETIC (deterministic division on the measured prefill ceiling).`

## Decode tok/s is the wrong headline metric

Same rig, same server, three configurations — and the headline number moves
for reasons that have nothing to do with the work. Judge a config by
seconds-per-correct-task, not by decode speed:

| Configuration | Decode tok/s | What the number actually is |
|---|---|---|
| Q4-class quant, cold prompt, ~30k ctx | 59.7 | **CLEAN** — n=3 median, thermally uncoupled |
| Q3-class quant, cold prompt, 128k ctx | 63–64 | **DIRECTIONAL** — the +5.5% over Q4 is the same scale as thermal swing; ordering holds, magnitude soft |
| Q4-class quant, warm repeat prompts | 148–163 | n-gram cache artifact — 2.4× the cold number; the label *is* the claim |

A 2.4× swing from prompt warmth alone; a full quant rung buys ~5%, which is
inside thermal noise. Meanwhile style steering cut total task *time* by ~33%
with zero hardware change (DIRECTIONAL, above). On local agent hardware:
**tokens-not-needed beats tokens-per-second.**

`Provenance: KyaniteLabs/context-kit README, decode-comparison table, all server-API-measured.`

## Measure honestly

The canonical instrument is a five-task battery: arithmetic reasoning, strict
JSON emission, a tool-call shape check, a small coding task, and a classic
trap riddle — all auto-graded, single iteration, thinking ON, hit directly
against the server's OpenAI-compatible API with no harness in the path. On
the canonical rig it read **7.6–7.7 s per correct task at ~170 completion
tokens** mean. It measures seconds-per-correct-task and
tokens-per-correct-task, because faster tokens don't help if the tokens are
dumber and you need more of them.

`Provenance: KyaniteLabs/context-kit README + kit/instruments/tpt_battery.py. VALIDITY: DIRECTIONAL (n=2).`

Method rules, each one learned in the canonical repo by publishing a number
that later had to be walked back:

- **n>=3 per arm, one thermal window.** Back-to-back runs on fan-cooled
  unified-memory silicon drift with temperature. Interleave arms and log a
  temperature reading, or your +5% is weather.
- **Count refusals per arm.** A config that makes the model refuse tool calls
  inflates its opponent's numbers invisibly.
- **Inspect every FAIL.** Graders under-count: a position-anchored grader
  failed a *correct* riddle answer because a code fence preceded it;
  budget-truncation zero-content runs are measurement artifacts, not model
  failures. Strip fences before grading; eyeball FAIL content before quoting
  pass counts.
- **Label n on every number you keep.** One unlabeled n=1 becomes a headline
  within a week. A single-sample instrument is pilot-class: it can choose the
  next experiment, never an adoption decision.
- **Audit the server's defaults before trusting your control arm.** The
  canonical repo ran an "uncapped" cell that silently inherited the server's
  2048 reasoning-budget default. The tell: six problems with byte-identical
  reasoning lengths across cells. Control arms should send explicit overrides
  instead of omitting fields.
- **Persist the full raw trace per row** — reasoning and final content, not
  just counts and grades. A completed 69-row run stored counts only, and when
  failure-mode analysis was needed later, the evidence was gone. Counts are
  for dashboards; rows are for autopsies.
- **Measure at the source.** A client-side token estimate said a change cut
  ~10% of prompt tokens; the server's own usage count said **-41.5%** — the
  estimate was blind to a ~2,100-token tool payload riding every request
  (n=180-run paired A/B). Wire truth lives in the server's usage numbers.

`Provenance: KyaniteLabs/context-kit README, "Method rules" list — each rule records the incident that produced it.`

## Integration laws, distilled

The canonical `docs/PATTERNS.md` writes up the harness patterns behind the
kit, each with the measurement that justifies it. Distilled to one line each:

1. **Append-only event log; context is derived, never mutated.** The session
   is a log; the model-visible context is computed from it per request. Trim
   becomes a projection policy, and a bad policy is a one-line revert, not
   data loss. Crash recovery, audit, and forking are replays.
2. **Idempotent effect ledger.** Side-effecting tools carry an idempotency
   key per (turn, step): attempted is recorded before execution, committed
   only after a clean result, and a committed key replays recorded bytes.
   Retries and crash-restarts cannot double-apply.
3. **Reconcile, never guess.** A turn without its completion marker gets an
   "ambiguous" reconciliation note before anything new runs — the model is
   told to verify state, never to blindly re-run a half-applied side effect.
4. **Prefix-stability law.** The leading system message is byte-identical for
   the session; anything that varies lands after the stable bytes. Every
   variation is re-prefill (see the arithmetic above).
5. **Read cache validated by mtime.** Layer 2 above.
6. **Diet at projection.** Layer 3 above — elide, dedup, compact, spill; the
   input log is never mutated.
7. **Verify-after-edit evidence.** An edit tool returns the changed region
   plus a parse verdict, not "ok". A local model that must look at its work
   stops declaring success over broken edits — fix it with infrastructure,
   not vibes.
8. **Repair-loop breaker.** Three or more consecutive steps touching the same
   file means the model is cycling; inject a diff-first note and reset the
   tracker. Each loop iteration re-pays prefill.
9. **Bounded turns, forced final answer.** A turn caps at N steps with a
   budget note a few steps before the cap, then one final no-tools request.
   Unbounded loops on local hardware are effectively hangs.
10. **Telemetry that cannot break the task.** Every measurement side channel
    is wrapped so its failure is swallowed and logged. An instrument that can
    fail a task measures nothing.

Adoption order by measured pay-off, from the canonical repo: styles first
(-36% tokens / -33% time, DIRECTIONAL), then munch (-91% prompt bytes on
exploration, CLEAN structural), then diet (-66% on repeat-heavy suites,
ARITHMETIC), then the ledger and recovery patterns once your harness runs
long unattended tasks — and the instruments from day zero, because no number
above survives contact with your rig unverified.

`Provenance: KyaniteLabs/context-kit docs/PATTERNS.md and its "Applying the kit" order.`

## The honest gap

- **One rig class.** Every number comes from the canonical rig: a 27B-class
  reasoning model, llama.cpp, unified-memory silicon. No Apple-silicon,
  Snapdragon, or Intel row is measured yet in the canonical repo's community
  table. Your silicon may sit differently; the labels tell you how much to
  trust each number until you re-measure.
- **The style win is regime-conditional.** High-effort thinking only — at low
  effort it measured +65% wall (a full inversion). Route per session.
- **Munch is Python-only.** The canonical implementation uses stdlib `ast`.
  The idea ports to other languages; the code does not.
- **Local-only.** Nothing here was measured against frontier APIs. The kit
  was built for local constraints (prefill cost, thermal drift, output
  ceilings); some rules may matter less elsewhere.
- **Small n throughout.** The style and battery arms run n=2–15 over a few
  nights; even the CLEAN byte counts come from single suites (n=1) — they are
  deterministic, but on one codebase. The cap experiment is the only paired
  statistical test (p=0.73). Everything else is labeled ordering, not
  magnitude.

## Verify it on your rig

Zero dependencies, Python stdlib only — adapt, don't install. The canonical
repo is the instrument; clone it and point the battery at your running
server:

```sh
git clone https://github.com/KyaniteLabs/context-kit
cd context-kit

# baseline battery (5 auto-graded tasks, seconds/tokens per correct task)
python3 kit/instruments/tpt_battery.py 8080 baseline

# style A/B: fused style vs no-style baseline
python3 kit/instruments/tpt_style.py 8080 baseline -
python3 kit/instruments/tpt_style.py 8080 fused fused
```

Run **n>=3 per arm in one thermal window**, interleave arms, count refusals,
and inspect every FAIL before believing a delta. The canonical README's
"Community results" table takes rig rows by PR — your numbers belong there.

## Provenance, credits, license

All measurements and the code they describe are
[KyaniteLabs/context-kit](https://github.com/KyaniteLabs/context-kit) (MIT) —
that repo's README, `docs/PATTERNS.md`, `docs/CREDITS.md`, and
`kit/instruments/` are the source of every claim above; nothing here
re-measures or re-brands them. The style prompts themselves stand on prior
art credited there: **caveman** (JuliusBrussee, MIT) and **ponytail**
(DietrichGebert, MIT); architecture spine credit DeepSeek Harness; the
local-model infrastructure patterns (verify-after-edit, gates) credit
community Pi setups; every instrument measures against llama.cpp.

These notes are an adaptation written for local-agent builders, published
under the same MIT license (see `LICENSE`). If a number here ever disagrees
with the canonical repo, the canonical repo wins — flag an issue.

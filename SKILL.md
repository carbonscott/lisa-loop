---
name: lisa-loop
description: >-
  Prompt generator for ralph-loop that adds lab-notebook as
  cross-iteration state. Each iteration can recall prior results,
  do work, and log progress. Use when you need a stateful iterative
  loop with structured memory. Trigger on: stateful loop, lisa-loop,
  iterative task with memory.
user-invocable: true
argument-hint: --store <path-to-lab-notebook> <task description>
---

# /lisa-loop — Stateful Loop

Lisa-loop is a prompt generator for ralph-loop. It builds a structured
prompt that gives each iteration access to a `lab-notebook` store —
adding queryable, persistent state on top of ralph-loop's existing
file and git-based state. The prompt is static, but the notebook data
changes each iteration, making the loop behave dynamically.

## Step 1: Parse Arguments

From `$ARGUMENTS`, extract:
- `--store <path>` — **required**. Path to a lab-notebook store.
- Everything else — the task description.

Edge cases:
- `--store` is missing entirely → ask: "Which lab-notebook store should this loop use? Provide the path."
- `--store` is present but no path follows it → ask: "You specified --store but no path. Which store?"
- Path is relative → resolve to absolute using the current working directory before proceeding. All paths in the generated prompt must be absolute.

If the task description is empty or too vague to act on, ask: "What should the loop iterate on?"

**Probe the store**: Run:
```bash
LAB_NOTEBOOK_DIR="<path>" lab-notebook schema
```
- If it **errors** → the store isn't initialized. Tell the user:
  "That store isn't initialized. Run `LAB_NOTEBOOK_DIR=<path> lab-notebook init` first."
- If it **succeeds** → parse the output for:
  - **Entry types** from the `type` line (e.g., `one of: experiment, baseline, distillation`)
  - **Schema fields** and their SQL types (lines marked `-- schema field`)

Pick the most appropriate entry type for the LOG phase based on the task and the available types. If unsure, ask briefly.

## Step 2: Define the Stop Condition

Ask: "What does 'done' look like?"

The answer becomes natural language embedded in the prompt. Examples:
- "All 5 loss functions have been tested and results logged"
- "Validation loss has stopped improving for 2 consecutive iterations"
- "The bug is fixed and tests pass"

The completion promise is always `"DONE"` — it's a mechanical signal. The semantics live in this description.

## Step 3: Optional — Workflow Shape

Ask these sub-questions in order. Each is independently skippable. If the user skips all three, omit all optional sections from the generated prompt — the generated prompt has no optional sections.

### Step 3a — Phases

> Does this task have distinct stages? List them in order, or skip.

If provided, the user lists ordered stages (e.g., "Launch → Monitor → Eval → Summary"). Each phase has implicit entry/exit criteria derived from the task. These go into a `## Phases` section in the generated prompt.

### Step 3b — Categories

> Should iterations be classified? Name the categories, or skip.

If provided, the user names iteration types (e.g., "heartbeat, progress, anomaly, completion"). If the store schema has a suitable field (e.g., `change_type`), use it as a schema flag in LOG. Otherwise, use `--extra category=<chosen>` and query via `json_extract(extra,'$.category')` in RECALL SQL. The ASSESS phase checks category distribution.

### Step 3c — Pacing (only if categories were provided in 3b)

> How should the loop pace itself during quiet periods?
> Default: `30s, 30s, 60s, 60s, then double each time (cap 1800s)`
> Customize, or "default"?

The user can provide their own sequence (e.g., `10s, 10s, 30s, then double, cap 600s`) or accept the default. Then ask:

> Which category means "nothing interesting happened"? (This triggers backoff escalation.)

Infer if obvious from context (e.g., "heartbeat" in a monitoring task). This is the **low-activity category** — consecutive entries of this category escalate the sleep duration through the backoff sequence.

If no category naturally maps to "nothing happened," skip pacing entirely.

## Step 4: Choose Context Name

Auto-suggest a context slug derived from the task description (e.g., "run 5 training experiments" → `training-experiments`). Present it:

> **Context**: `training-experiments` — OK, or different name?

The context name scopes all entries from this loop. It's the key for future queries, so it should be intentional and descriptive.

## Step 5: Set Safety Cap

Default to `--max-iterations 10`. Mention it:

> **Max iterations**: 10 (safety cap). Adjust?

## Step 6: Assemble, Confirm, Launch

Build the prompt using the template below, filling in all placeholders from steps 1-5. Sections marked *(if configured)* are only included when the user provided values in step 3. If skipped, omit them entirely — the prompt should have no optional sections. Present the full assembled prompt to the user for review:

> Here's the assembled lisa-loop prompt:
>
> *(show the filled-in prompt)*
>
> Ready to launch, or adjust?

**After the user confirms, immediately invoke `/ralph-loop:ralph-loop` with the assembled prompt, `--max-iterations N`, and `--completion-promise "DONE"`. No preamble, no summary, no "launching now" text. Just call the Skill tool.**

---

## Prompt Template

Fill placeholders from steps 1-5. The 5 iteration phases are guidance, not rigid script — the agent adapts based on what it finds. Sections marked *(if configured)* are only included when the user provided values in step 3; omit them entirely otherwise.

```
## State Store
Notebook: <absolute-path-from-store-argument>
Context: <context-slug>
Available entry types: <comma-separated list from lab-notebook schema output>
Available fields: <comma-separated list from lab-notebook schema output, with types>

## Goal
<task description>

## Phases *(if configured)*
<numbered list of phases from step 3>
Determine current phase by querying notebook entries. Transition to the next
phase when its exit criteria are met. The stop condition applies after the
final phase.

## Stop Condition
<natural language from step 2 — when this is true, output <promise>DONE</promise>>

## Pacing *(if configured)*

Backoff sequence: <user-supplied or default: 30s, 30s, 60s, 60s, then double, cap 1800s>
Low-activity category: <from step 3c, e.g., "heartbeat">

Algorithm:
1. In RECALL, count consecutive entries of the low-activity category
   from the most recent entries.
2. Walk the backoff sequence by that count (0 consecutive → first value,
   1 consecutive → second value, etc.).
3. After the explicit sequence is exhausted, each additional consecutive
   low-activity entry doubles the previous sleep duration, up to the cap.
4. Any non-low-activity entry resets the count to 0.
5. Sleep at the end of CHECK (after logging), before the next iteration.

Example for sequence `30, 30, 60, 60, cap 1800`:
count 0→30s, 1→30s, 2→60s, 3→60s, 4→120s, 5→240s, 6→480s, 7→960s, 8+→1800s

## Each Iteration

1. RECALL — Query the notebook for prior entries in this context:
   ```
   LAB_NOTEBOOK_DIR="<path>" lab-notebook search "<context>" --context <context>
   ```
   If no entries exist, this is iteration 1 — start fresh.
   For structured queries (counts, comparisons):
   ```
   LAB_NOTEBOOK_DIR="<path>" lab-notebook sql "SELECT ts, type, <schema-fields>, substr(content,1,120) FROM entries WHERE context='<context>' ORDER BY ts DESC LIMIT 10"
   ```
   *(if pacing)* Count consecutive entries of the low-activity category
   from the top of the results to determine the current backoff position.

2. ASSESS — Based on recalled state: what has been done, what remains,
   what should this iteration focus on?
   *(if categories)* Check recent entry categories to understand loop trajectory.
   *(if phases)* Determine which phase we're in based on logged entries.

3. EXECUTE — Do the work for this iteration.

4. LOG — Record what you did and learned:
   ```
   LAB_NOTEBOOK_DIR="<path>" lab-notebook emit \
     --context <context> --type <entry-type> \
     [--<schema-field> value ...] [--extra key=value ...] \
     "concise summary of what happened this iteration"
   ```
   Use schema-defined fields as top-level flags (e.g., `--status keep
   --change_type progress`). Use `--extra` only for undeclared fields.
   If the logging command fails, note the error and continue —
   don't let logging failures block the main work.
   *(if categories)* Classify this iteration as one of: <categories>.
   Use the schema field if available (e.g., `--change_type heartbeat`),
   otherwise `--extra category=<chosen>`.

5. CHECK — Evaluate the stop condition. If met, output <promise>DONE</promise>.
   Otherwise, summarize remaining work (the next iteration will see this
   in the notebook via step 1).
   *(if pacing)* Compute sleep duration: walk the backoff sequence by
   the consecutive low-activity count from RECALL. If past the end of
   the explicit sequence, double the last value (capped). Execute:
   `sleep <computed-seconds>`
```

---

## Stateful Mechanisms

The generated prompt leverages all available cross-iteration state. The last four are provided by ralph-loop; lisa-loop adds the first.

| Mechanism | How | Role |
|-----------|-----|------|
| **Notebook/logbook entries** | `lab-notebook search/sql` | Primary — structured, queryable, persists beyond the loop |
| **Files on disk** | Read modified files | Secondary — see your own prior work |
| **Git history** | `git log`, `git diff` | Secondary — what changed between iterations |
| **Iteration counter** | Shown in ralph-loop system message | Secondary — know which iteration you're on |
| **Test/build results** | Run tests, check output | Secondary — backpressure from verification |

---

## Template Reference (Hand-Crafting)

To build a lisa-loop prompt manually instead of using interactive setup:

1. Pick your state store (any lab-notebook path)
2. Pick a context name (all entries share this key)
3. Write a prompt following the 5-phase pattern above
4. Invoke directly:
   ```
   /ralph-loop:ralph-loop "<your prompt>" --max-iterations 10 --completion-promise "DONE"
   ```

**Key principles for hand-crafted prompts:**
- Always include a RECALL phase at the top — this is what makes the loop stateful
- Always include a LOG phase — this is what feeds the next iteration's RECALL
- Handle iteration 1 explicitly ("if no entries, start fresh")
- Include command templates for `lab-notebook` — the agent needs the CLI syntax
- Make the stop condition concrete and evaluable, not vague
- Add `--max-iterations` as a safety cap even if you have a completion promise
- The agent must output `<promise>DONE</promise>` (with XML tags), not bare `DONE`. The `--completion-promise "DONE"` flag tells ralph-loop what text to look for; the `<promise>` wrapper is required by ralph-loop's stop hook

$ARGUMENTS

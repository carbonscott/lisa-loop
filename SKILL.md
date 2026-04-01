---
name: recall-loop
description: >-
  Stateful iterative loop with structured memory via lab-notebook.
  Each iteration recalls prior results, does work, and logs progress.
  Use when you need a loop with cross-iteration memory. Trigger on:
  stateful loop, recall-loop, iterative task with memory.
user-invocable: true
argument-hint: --store <path-to-lab-notebook> <task description>
---

# /recall-loop — Stateful Loop

Recall-loop builds a structured prompt where each iteration has access to
a lab-notebook store for queryable, persistent state. The loop runs on
ralph-wiggum (a while-true loop driver).

## Protocol

This skill uses the base iteration protocol defined in:
[loop-protocols/recall-execute-log.md](../loop-protocols/recall-execute-log.md)

Read that file for the full RECALL -> EXECUTE -> LOG cycle details.

## Setup

### Step 1: Parse Arguments

From `$ARGUMENTS`, extract:
- `--store <path>` — **required**. Path to a lab-notebook store.
- Everything else — the task description.

Edge cases:
- `--store` missing → ask: "Which lab-notebook store should this loop use?"
- `--store` present but no path → ask: "You specified --store but no path."
- Relative path → resolve to absolute before proceeding.

If the task description is empty or too vague, ask: "What should the loop
iterate on?"

**Probe the store**:
```bash
LAB_NOTEBOOK_DIR="<path>" lab-notebook schema
```
- If it **errors** → tell the user to initialize it first.
- If it **succeeds** → parse the output for entry types and schema fields.

Pick the most appropriate entry type for the LOG phase.

### Step 2: Define the Stop Condition

Ask: "What does 'done' look like?"

The answer becomes natural language embedded in the prompt. Examples:
- "All 5 loss functions have been tested and results logged"
- "The bug is fixed and tests pass"

### Step 3: Choose Context Name

Auto-suggest a context slug from the task description (e.g., "run 5
training experiments" → `training-experiments`). Present it:

> **Context**: `training-experiments` — OK, or different name?

### Step 4: Set Safety Cap

Default to `--max-iterations 10`. Mention it:

> **Max iterations**: 10 (safety cap). Adjust?

### Step 5: Assemble, Confirm, Launch

Build the prompt using the template below, filling in values from
steps 1-4. Present the assembled prompt for review:

> Here's the assembled recall-loop prompt:
>
> *(show the filled-in prompt)*
>
> Ready to launch, or adjust?

**After the user confirms:**

1. Write the prompt to `/tmp/recall-loop-<context-slug>.md` for reference.
2. Invoke `/ralph-loop:ralph-loop` with the full prompt text and flags:
   `--max-iterations N --completion-promise 'DONE'`

---

## Prompt Template

Fill placeholders from steps 1-4.

```
## State Store
Notebook: <absolute-path>
Context: <context-slug>
Available entry types: <from schema output>
Available fields: <from schema output>

## Goal
<task description>

## Stop Condition
<natural language from step 2>
When this is true, output <promise>DONE</promise> to exit the loop.

## Each Iteration

1. RECALL — Query the notebook for prior work:
   ```
   LAB_NOTEBOOK_DIR="<path>" lab-notebook sql "SELECT ts, type, substr(content,1,200) FROM entries WHERE context='<context>' ORDER BY ts DESC LIMIT 10"
   ```
   If no entries exist, this is the first iteration — start fresh.

2. EXECUTE — Do the work for this iteration, informed by recalled context.

3. LOG — Record what happened:
   ```
   LAB_NOTEBOOK_DIR="<path>" lab-notebook emit \
     --context <context> --type <entry-type> \
     [--<schema-field> value ...] [--extra key=value ...] \
     "concise summary of what happened"
   ```

After LOG, evaluate the stop condition. If done, output <promise>DONE</promise>.
```

$ARGUMENTS

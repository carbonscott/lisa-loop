# Multi-Dimensional Cursors in Lisa-Loop

## When to Use

You have multiple items (issues, experiments, datasets, etc.) and each item
requires a multi-phase workflow. Your instinct is nested loops — an outer
loop over items and an inner loop over phases. Lisa-wiggum supports this
natively with `--dim` flags that define the cursor dimensions. The hook
flattens the nesting into one loop with mechanical cursor advancement.

## Pattern

```
nested (conceptual):              lisa-wiggum (actual):

for item in 1..N:                 single loop with --dim flags:
  for phase in A..D:                cursor position in system message
    do work                         do work for current position
    if phase fails: retry           PHASE COMPLETE → advance outer dim
                                    PHASE FAILED   → advance retry dim
                                    cursor exhausted → loop ends
```

The cursor position is injected into the system message by the stop hook.
Each iteration reads its position from the header (e.g.,
`Cursor: item=3, phase=execute, retry=0`), does one unit of work, and
signals the outcome. The hook advances the cursor mechanically.

## How `--dim` Flags Work

Each `--dim` flag defines one dimension of the cursor. Order matters —
dimensions are listed outer-to-inner. The innermost dimension is the
retry/failure axis.

```bash
/lisa-wiggum:lisa-loop --prompt-file task.md \
  --store /path/to/notebook --context my-repo/fix-issues \
  --dim item 101 102 103 \
  --dim phase plan review execute test \
  --dim retry 0 1 2 \
  --on-exhaust skip \
  --max-iterations 50 --completion-promise "DONE"
```

This creates a 3D cursor: `(item, phase, retry)`.

### Signal-driven advancement

| Signal | Effect |
|--------|--------|
| `PHASE COMPLETE` | Advances second-to-innermost dimension (e.g., `phase` when `retry` is innermost), resets all inner dimensions |
| `PHASE FAILED` | Advances innermost dimension (retry) |
| Neither | Cursor unchanged; same position reruns next iteration |
| Retry exhausted | Resets innermost, advances second-to-innermost (hardcoded skip behavior) |
| All dimensions exhausted | Loop ends (cursor complete) |
| `<promise>DONE</promise>` | Loop ends immediately |

*Note: `--on-exhaust skip` is stored in the state file but currently not read by the hook — the skip behavior is hardcoded. The flag is reserved for future extension.*

### Examples

Success at `(item=101, phase=plan, retry=0)`:
  -> advances to `(item=101, phase=review, retry=0)`

Failure at `(item=101, phase=execute, retry=0)`:
  -> advances to `(item=101, phase=execute, retry=1)`

Failure at `(item=101, phase=execute, retry=2)` (retry exhausted, on_exhaust=skip):
  -> advances to `(item=101, phase=test, retry=0)`

Success at `(item=101, phase=test, retry=0)` (last phase):
  -> advances to `(item=102, phase=plan, retry=0)`

## Notebook Conventions

### Context naming

Use the pattern `<project>/<loop-name>`:
```
my-repo/fix-all-issues
training/sweep-architectures
```

All entries from the loop share this context.

### Entry types

These entry types are useful for tracking work, but they are not required
for cursor advancement (the cursor is mechanical):

| Entry type   | When to emit | Purpose |
|-------------|--------------|---------|
| `milestone` | A unit of work completes successfully | Records what was accomplished |
| `decision`  | A plan is formed, a review is done, a choice is made | Records reasoning |
| `dead-end`  | A work attempt fails | Records what went wrong |
| `observation` | Intermediate findings | Context for later iterations |

### Tagging entries with cursor position

The system message provides an `emit` command template with `--tags`
pre-filled from the current cursor position. Use it so entries are
queryable by position:

```bash
LAB_NOTEBOOK_DIR="/path/to/notebook" lab-notebook emit \
  --context my-repo/fix-issues --type milestone \
  --tags "item=101,phase=plan" \
  "Planned approach: refactor DataLoader for streaming"
```

Query entries for a specific position:
```bash
LAB_NOTEBOOK_DIR="/path/to/notebook" lab-notebook sql \
  "SELECT ts, type, tags, substr(content,1,200) FROM entries
   WHERE context='my-repo/fix-issues' AND tags LIKE '%item=101%'
   ORDER BY ts DESC LIMIT 10"
```

## Prompt Template for Multi-Dimensional Work

The prompt for a multi-dimensional loop is simpler than the old
flattening approach because the cursor handles all transition logic.

```
## State Store
Notebook: <absolute-path>
Context: <context-slug>
Available entry types: <from lab-notebook schema>

## Goal
<task description — what items, what phases, what success looks like>

## Phases (per item)
A. PLAN — <description>
B. REVIEW — <description>
C. EXECUTE — <description>
D. TEST — <description>

## Stop Condition
<natural language — when this is true, output <promise>DONE</promise>>

## Each Iteration

1. RECALL — Read your current cursor position from the system message
   header (e.g., "Cursor: item=101, phase=plan, retry=0").
   Query the notebook for context on the current item and phase:
   ```
   LAB_NOTEBOOK_DIR="<path>" lab-notebook sql \
     "SELECT ts, type, tags, substr(content,1,200) FROM entries
      WHERE context='<ctx>' AND tags LIKE '%item=<current>%'
      ORDER BY ts DESC LIMIT 10"
   ```

2. ASSESS — Based on cursor position and notebook entries:
   what does this phase require? Any prior dead-ends to learn from?

3. EXECUTE — Do the work for the current (item, phase).

4. LOG — Record the outcome using the emit template from the system
   message (it has --tags pre-filled):
   ```
   LAB_NOTEBOOK_DIR="<path>" lab-notebook emit \
     --context <ctx> --type <milestone|decision|dead-end> \
     --tags "<cursor-tags>" \
     "<summary of what happened>"
   ```
   (`--tags` requires a `tags` field in the store schema — check `lab-notebook schema`.)

5. CHECK — Signal the outcome:
   - Work succeeded → include "PHASE COMPLETE"
   - Work failed → include "PHASE FAILED"
   - All items complete → output <promise>DONE</promise>
   - Work still in progress → include neither signal; same position reruns
```

## Worked Example: 9 GitHub Issues

### Scenario

Solve GitHub issues #101 through #109. Each issue goes through:
plan -> adversarial review -> execute -> smoke test.

### Invocation

```bash
/lisa-wiggum:lisa-loop --prompt-file /tmp/lisa-loop-fix-9-issues.md \
  --store /data/lab-notebook \
  --context my-repo/fix-9-issues \
  --dim item 101 102 103 104 105 106 107 108 109 \
  --dim phase plan review execute test \
  --dim retry 0 1 2 \
  --on-exhaust skip \
  --max-iterations 50 --completion-promise "DONE"
```

### Prompt (written to `/tmp/lisa-loop-fix-9-issues.md`)

```
## State Store
Notebook: /data/lab-notebook
Context: my-repo/fix-9-issues
Available entry types: observation, decision, dead-end, question, milestone

## Goal
Solve GitHub issues #101 through #109 sequentially, each through a full
plan-review-execute-test workflow.

## Phases (per item)
A. PLAN — Fetch the issue with /gh-skills. Read all relevant code. Write a
   complete, self-contained implementation plan. Log plan as a decision entry.
B. REVIEW — Adopt an adversarial reviewer mindset. Critique the plan for:
   weaknesses, missed edge cases, interactions with other code, test gaps.
   For each critique, either defend with evidence or acknowledge and revise.
   Log revised decisions.
C. EXECUTE — Implement the revised plan. Commit with a message referencing
   the issue number.
D. TEST — Run smoke tests: single GPU, DDP, FSDP. If any fail, diagnose
   and fix. Repeat until all three pass.

## Stop Condition
All 9 issues solved — the cursor will exhaust naturally when the last item's
last phase succeeds. Output <promise>DONE</promise> when complete.

## Each Iteration

1. RECALL — Read cursor position from the system message header
   (e.g., "Cursor: item=103, phase=review, retry=0").
   Query the notebook for context on the current item:
   ```
   LAB_NOTEBOOK_DIR="/data/lab-notebook" \
     lab-notebook sql \
     "SELECT ts, type, tags, substr(content,1,200) FROM entries
      WHERE context='my-repo/fix-9-issues'
        AND tags LIKE '%item=<current_item>%'
      ORDER BY ts DESC LIMIT 10"
   ```

2. ASSESS — Based on cursor position and recalled entries:
   - Which issue am I on? Which phase?
   - Any prior dead-ends to learn from?
   - What does this phase require?

3. EXECUTE — Do the work for the current phase of the current issue.

4. LOG — Record the outcome using the emit template from the system message:
   ```
   LAB_NOTEBOOK_DIR="/data/lab-notebook" \
     lab-notebook emit --context my-repo/fix-9-issues --type milestone \
     --tags "<cursor-tags-from-system-message>" \
     "<summary of what happened>"
   ```
   On failure, use `--type dead-end` instead.

5. CHECK — Signal the outcome:
   - Phase work succeeded → include "PHASE COMPLETE"
   - Phase work failed → include "PHASE FAILED"
   - All 9 issues complete → output <promise>DONE</promise>
```

### Example system message mid-run

After completing issue #101 and starting #102's review phase, the system
message header would show:

```
Lisa iteration 7 | Cursor: item=102, phase=review, retry=0

## Traversal
1: item=101,phase=plan,retry=0 -> success
2: item=101,phase=review,retry=0 -> success
3: item=101,phase=execute,retry=0 -> success
4: item=101,phase=test,retry=0 -> success
5: item=102,phase=plan,retry=0 -> success
6: item=102,phase=review,retry=0 -> none
```

The agent reads this, sees it needs to do adversarial review for issue #102,
queries the notebook for #102's plan entries, and proceeds.

## Generalizing to Deeper Nesting

Add more `--dim` flags. For example, a 4-level cursor:

```bash
--dim dataset mnist cifar imagenet \
--dim model resnet vgg transformer \
--dim phase train eval \
--dim retry 0 1
```

Each additional dimension is just another `--dim` flag. The stop hook
handles all the transition logic mechanically. `PHASE COMPLETE` always
advances the second-to-innermost dimension — here `phase`, not `dataset`
or `model`. `PHASE FAILED` advances `retry` (innermost). When `retry` is
exhausted, the hook overflows to `phase` (second-to-innermost).

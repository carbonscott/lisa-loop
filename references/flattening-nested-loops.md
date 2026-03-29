# Flattening Nested Loops in Lisa-Loop

## When to Use

You have multiple items (issues, experiments, datasets, etc.) and each item
requires a multi-phase workflow. Your instinct is nested loops — an outer
loop over items and an inner loop over phases. Ralph-loop only supports one
active loop (single state file), so you flatten the nesting into one loop
with a multi-dimensional cursor tracked in lab-notebook.

## Pattern

```
nested (impossible):              flattened (works):

for item in 1..N:                 single lisa-loop:
  for phase in A..D:                read (item, phase) from notebook
    do work                         do work for (item, phase)
    if phase fails: retry           advance cursor:
                                      phase failed       → same item, same phase
                                      phase succeeded     → same item, next phase
                                      last phase passed   → next item, first phase
                                      last item completed → LOOP_COMPLETE
```

Lab-notebook replaces both loop variables. Each iteration is stateless —
it reconstructs its position entirely from notebook queries.

## Notebook Conventions

### Context naming

Use the pattern `<project>/<loop-name>`:
```
my-repo/fix-all-issues
training/sweep-architectures
```

All entries from the loop share this context.

### Entry types for cursor tracking

Use these entry types to track progress:

| Entry type   | When to emit | Purpose |
|-------------|--------------|---------|
| `milestone` | A phase completes successfully for an item | Marks phase completion — this is what the cursor reads |
| `decision`  | A plan is formed, a review is done, a choice is made | Records reasoning for the current phase |
| `dead-end`  | A phase attempt fails | Records what went wrong, prevents repeating the same failure |
| `observation` | Intermediate findings during a phase | Context for later phases |

### Content format for milestones (critical)

Milestone entries MUST follow this exact format so queries can parse them:

```
Item #<N>: <PHASE_NAME> COMPLETE — <one-line summary>
```

Examples:
```
Item #3: PLAN COMPLETE — will refactor DataLoader to support streaming
Item #3: REVIEW COMPLETE — revised plan to handle edge case in batch collation
Item #3: EXECUTE COMPLETE — committed refactor in 2 files, 47 lines changed
Item #3: TEST COMPLETE — single-gpu, ddp, fsdp all pass
```

When all phases for an item pass, emit one final milestone:
```
Item #3: ALL PHASES COMPLETE
```

## Query Recipes

Replace `<STORE>` with the absolute notebook path and `<CTX>` with the context.

### Count completed items

```bash
LAB_NOTEBOOK_DIR="<STORE>" lab-notebook sql \
  "SELECT COUNT(DISTINCT substr(content, 7, instr(substr(content,7),':')-1))
   FROM entries
   WHERE context='<CTX>' AND type='milestone'
     AND content LIKE '%ALL PHASES COMPLETE%'"
```

### Find current item and last completed phase

```bash
LAB_NOTEBOOK_DIR="<STORE>" lab-notebook sql \
  "SELECT ts, substr(content,1,120) FROM entries
   WHERE context='<CTX>' AND type='milestone'
   ORDER BY ts DESC LIMIT 5"
```

Read the results top-down:
- If the most recent milestone is `Item #N: ALL PHASES COMPLETE` and N is the last item, all work is done — skip to CHECK and emit DONE.
- If the most recent milestone is `Item #N: ALL PHASES COMPLETE` and N is not the last item, the current item is N+1, starting at the first phase.
- If the most recent milestone is `Item #N: <PHASE> COMPLETE`, the current item is N, and the next phase follows `<PHASE>`.
- If there are no milestones, this is iteration 1 — start at item 1, phase 1.

### Check for dead-ends on current item and phase

```bash
LAB_NOTEBOOK_DIR="<STORE>" lab-notebook search "Item #<N>: <PHASE> FAILED" --context <CTX> --type dead-end
```

## Transition Logic Template

Drop this into the `## Each Iteration` section of a lisa-loop prompt.
Replace placeholders with your specific items and phases.

```
## Items
<ITEMS — numbered list, e.g.:>
1. GitHub issue #101 — fix data loader race condition
2. GitHub issue #102 — add retry logic to API client
3. GitHub issue #103 — update schema migration

## Phases (per item)
<PHASES — ordered list, e.g.:>
A. PLAN — read the issue, understand the code, write an implementation plan
B. REVIEW — adversarially critique the plan, revise if needed
C. EXECUTE — implement the revised plan, commit
D. TEST — run smoke tests, fix failures until all pass

## Each Iteration

1. RECALL — Query the notebook to find where you are:
   ```
   LAB_NOTEBOOK_DIR="<STORE>" lab-notebook sql \
     "SELECT ts, substr(content,1,120) FROM entries
      WHERE context='<CTX>' AND type='milestone'
      ORDER BY ts DESC LIMIT 10"
   ```
   Parse the most recent milestones to determine:
   - current_item: which item number you are working on
   - current_phase: which phase to execute next
   - If no milestones exist → current_item=1, current_phase=A
   - If the most recent milestone is "Item #N: ALL PHASES COMPLETE" and N
     is the last item → all work is done, skip to CHECK and emit DONE

   Also check for dead-ends on the current item and phase:
   ```
   LAB_NOTEBOOK_DIR="<STORE>" lab-notebook search "Item #<current_item>: <current_phase> FAILED" \
     --context <CTX> --type dead-end
   ```

2. ASSESS — Decide what to do this iteration:
   - If the dead-end search returned results for the current phase: adjust approach based on what failed
   - Otherwise: proceed with the phase's defined work

3. EXECUTE — Do the work for (current_item, current_phase).

4. LOG — Record the outcome:
   - On success:
     ```
     LAB_NOTEBOOK_DIR="<STORE>" lab-notebook emit \
       --context <CTX> --type milestone \
       "Item #<N>: <PHASE> COMPLETE — <summary>"
     ```
   - If this was the last phase for this item, also emit:
     ```
     LAB_NOTEBOOK_DIR="<STORE>" lab-notebook emit \
       --context <CTX> --type milestone \
       "Item #<N>: ALL PHASES COMPLETE"
     ```
   - On failure:
     ```
     LAB_NOTEBOOK_DIR="<STORE>" lab-notebook emit \
       --context <CTX> --type dead-end \
       "Item #<N>: <PHASE> FAILED — <what went wrong>"
     ```

5. CHECK — Evaluate stop condition:
   - Count items with "ALL PHASES COMPLETE" milestones
   - If count == total items → output <promise>DONE</promise>
   - Otherwise → summarize: "Completed <X>/<TOTAL>. Next: Item #<N>, phase <P>."
```

## Worked Example: 9 GitHub Issues

### Scenario

Solve GitHub issues #101 through #109. Each issue goes through:
plan → adversarial review → execute → smoke test (single GPU, DDP, FSDP).

### Complete prompt (ready for ralph-loop)

```
## State Store
Notebook: /lustre/orion/lrn091/proj-shared/cwang31/lab-notebook
Context: my-repo/fix-9-issues
Available entry types: observation, decision, dead-end, question, milestone

## Goal
Solve GitHub issues #101 through #109 sequentially, each through a full
plan-review-execute-test workflow.

## Items
1. Item #101
2. Item #102
3. Item #103
4. Item #104
5. Item #105
6. Item #106
7. Item #107
8. Item #108
9. Item #109

## Phases (per item)
A. PLAN — Fetch the issue with /gh-skills. Read all relevant code. Write a
   complete, self-contained implementation plan. Do NOT enter plan mode
   (operator is hands-off). Log plan as a decision entry.
B. REVIEW — Adopt an adversarial reviewer mindset. Critique the plan for:
   weaknesses, missed edge cases, interactions with other code, test gaps.
   For each critique, either defend with evidence or acknowledge and revise.
   Log revised decisions.
C. EXECUTE — Implement the revised plan. Commit with a message referencing
   the issue number.
D. TEST — Run smoke tests: single GPU, DDP, FSDP. If any fail, diagnose
   and fix. Repeat until all three pass.

## Stop Condition
All 9 issues have "ALL PHASES COMPLETE" milestone entries → <promise>DONE</promise>

## Each Iteration

1. RECALL — Query the notebook:
   ```
   LAB_NOTEBOOK_DIR="/lustre/orion/lrn091/proj-shared/cwang31/lab-notebook" \
     lab-notebook sql \
     "SELECT ts, substr(content,1,120) FROM entries
      WHERE context='my-repo/fix-9-issues' AND type='milestone'
      ORDER BY ts DESC LIMIT 15"
   ```
   Determine current_item and current_phase from the most recent milestones.
   If no milestones → start at Item #101, phase PLAN.
   If the most recent milestone is "Item #109: ALL PHASES COMPLETE",
   all work is done — skip to CHECK and emit DONE.

   Check for dead-ends on current item:
   ```
   LAB_NOTEBOOK_DIR="/lustre/orion/lrn091/proj-shared/cwang31/lab-notebook" \
     lab-notebook search "Item #<current>: <current_phase> FAILED" \
     --context my-repo/fix-9-issues --type dead-end
   ```

2. ASSESS — Based on recalled state:
   - Which issue am I on? Which phase?
   - Any prior dead-ends to learn from?
   - What does this phase require?

3. EXECUTE — Do the work for the current phase of the current issue.

4. LOG — Record the outcome:
   On phase success:
   ```
   LAB_NOTEBOOK_DIR="/lustre/orion/lrn091/proj-shared/cwang31/lab-notebook" \
     lab-notebook emit --context my-repo/fix-9-issues --type milestone \
     "Item #<N>: <PHASE> COMPLETE — <summary>"
   ```
   When all 4 phases pass for an issue:
   ```
   LAB_NOTEBOOK_DIR="/lustre/orion/lrn091/proj-shared/cwang31/lab-notebook" \
     lab-notebook emit --context my-repo/fix-9-issues --type milestone \
     "Item #<N>: ALL PHASES COMPLETE"
   ```
   On failure:
   ```
   LAB_NOTEBOOK_DIR="/lustre/orion/lrn091/proj-shared/cwang31/lab-notebook" \
     lab-notebook emit --context my-repo/fix-9-issues --type dead-end \
     "Item #<N>: <PHASE> FAILED — <what went wrong>"
   ```

5. CHECK — Count completed issues:
   ```
   LAB_NOTEBOOK_DIR="/lustre/orion/lrn091/proj-shared/cwang31/lab-notebook" \
     lab-notebook sql \
     "SELECT COUNT(DISTINCT substr(content, 7, instr(substr(content,7),':')-1))
      FROM entries
      WHERE context='my-repo/fix-9-issues' AND type='milestone'
        AND content LIKE '%ALL PHASES COMPLETE%'"
   ```
   If count == 9 → <promise>DONE</promise>
   Otherwise → "Completed <X>/9. Next: Issue #<N>, phase <P>."
```

### Example notebook state mid-run (after completing 2 issues, mid-way through 3rd)

```
ts                    type        content
2026-03-28T21:20:00   decision    Item #101: plan is to refactor the DataLoader...
2026-03-28T21:20:30   milestone   Item #101: PLAN COMPLETE — refactor DataLoader for streaming
2026-03-28T21:21:00   decision    Item #101: review found edge case in batch collation, revised
2026-03-28T21:21:30   milestone   Item #101: REVIEW COMPLETE — added batch boundary handling
2026-03-28T21:23:00   milestone   Item #101: EXECUTE COMPLETE — committed in abc1234
2026-03-28T21:25:00   milestone   Item #101: TEST COMPLETE — single-gpu, ddp, fsdp pass
2026-03-28T21:25:01   milestone   Item #101: ALL PHASES COMPLETE
2026-03-28T21:26:00   decision    Item #102: plan is to add retry logic to API client...
2026-03-28T21:26:30   milestone   Item #102: PLAN COMPLETE — exponential backoff in api_client.py
2026-03-28T21:27:00   milestone   Item #102: REVIEW COMPLETE — no revisions needed
2026-03-28T21:28:00   milestone   Item #102: EXECUTE COMPLETE — committed in def5678
2026-03-28T21:29:00   dead-end    Item #102: TEST FAILED — DDP test hangs on barrier sync
2026-03-28T21:30:00   milestone   Item #102: TEST COMPLETE — fixed barrier, all pass
2026-03-28T21:30:01   milestone   Item #102: ALL PHASES COMPLETE
2026-03-28T21:31:00   decision    Item #103: plan is to update schema migration...
2026-03-28T21:31:30   milestone   Item #103: PLAN COMPLETE — add nullable column, backfill
```

An agent reading this state at the next iteration would:
1. See 2 "ALL PHASES COMPLETE" entries → 2 items done
2. See `Item #103: PLAN COMPLETE` as the most recent milestone → current item is #103, next phase is REVIEW
3. Proceed with adversarial review of item #103's plan

## Generalizing to Deeper Nesting

The same pattern extends to any depth. For 3 levels (item, phase, attempt):

```
Cursor: (item=N, phase=P, attempt=A)

Transitions:
  attempt failed, retries left     → (N, P, A+1)
  attempt succeeded                → (N, next_phase, 1)
  last phase succeeded             → (N+1, first_phase, 1)
  last item, last phase succeeded  → LOOP_COMPLETE
```

Each level needs its own **completion signal** in the notebook. The query
reconstructs all dimensions of the cursor from the most recent entries.
The prompt's transition logic is the only thing that changes — the
notebook conventions and query patterns stay the same.

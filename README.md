# detection-volume-test

Generates a large, predictable number of `Source-Code-Overwritten` detections,
for testing the asynchronous suppression-rule backfill.

## How volume actually works here

`HardenRunnerDetections` is keyed:

| Key | Attribute | Value |
| --- | --- | --- |
| HASH | `workflow_id` | `<owner>-<repo>-<workflow path, / replaced by ->` |
| RANGE | `detection` | `Source-Code-Overwritten-<file basename>` |

Neither key carries `run_id`, `job_id` or a timestamp. That drives everything
about this repo's shape:

- **Re-running a workflow does not add detections.** It rewrites the same rows.
  Running everything twice still leaves 5,500 rows.
- **A matrix does not add detections either** — parallel jobs touching the same
  file collide into a single row. So this repo deliberately does not use one.
- **`File` is the basename**, so basenames must be globally unique. They are:
  `module-00000.js` … `module-01099.js`.

```
total detections = workflows x distinct filenames = 5 x 1100 = 5500
```

A second hard limit lives in `correlate.go`:

```go
if processedCount > 11 {
    continue   // at most 11 file-overwrite detections per step
}
```

so each step touches exactly 11 files and the file list is chunked across 100
steps per workflow. Packing more files into a step silently wastes them.

## Preconditions

1. The repo must be onboarded to the StepSecurity app in the target tenant.
2. **The tenant must already have at least one detection rule.** Overwrite
   detections are only recorded when `len(rules) > 0` (`correlate.go`), so
   create a throwaway rule that matches nothing before the first run, or no
   detections will appear at all.

## Usage

All workflows are `workflow_dispatch` only — nothing runs until you ask.

```bash
# one workflow -> 1,100 detections
gh workflow run generate-01.yml

# everything -> 5,500 detections
for i in 01 02 03 04 05; do gh workflow run generate-$i.yml; done
```

Each job is ~100 short steps, so expect a few minutes per workflow.

## Scaling up

Edit the constants at the top of `generate.py` and re-run it:

```python
WORKFLOWS = 5              # more workflow files -> more workflow_id partitions
FILES_PER_WORKFLOW = 1100  # more distinct basenames -> more sort keys
```

`WORKFLOWS x FILES_PER_WORKFLOW` is the detection count. Raising `WORKFLOWS` is
cheaper in CI time than raising `FILES_PER_WORKFLOW`, because workflows run in
parallel while steps within a job are serial.

```bash
python3 generate.py && git add -A && git commit -m "chore: regenerate" && git push
```

## Testing the backfill

1. Run the workflows, wait for detections to appear in the console.
2. Create a suppression rule for `source_code_overwritten` scoped to this repo,
   with `file: "*"` so it matches all of them.
3. Watch it:
   - the create request should return immediately rather than hanging ~30s
   - worker logs: `starting fan-out` → `fanned out per-org ops` →
     `starting org walk` → `completed org walk` with `scanned` / `suppressed`
   - if a walk exceeds its budget you should see
     `time budget reached, enqueued continuation` followed by
     `starting org walk slice=2 resumed=true`

To re-test from a clean slate, delete the rule and the detections, then re-run
the workflows — the same rows are rewritten, so the counts stay stable.

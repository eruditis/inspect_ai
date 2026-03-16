# Design: log_file_info Header Fallback for Custom .eval Filenames

**Issue:** eruditis/inspect_ai#1
**Date:** 2026-03-16
**Status:** Approved
**Target:** Upstream PR to UKGovernmentBEIS/inspect_ai

## Problem

`log_file_info()` in `src/inspect_ai/log/_file.py` parses `task` and `task_id` from `.eval` filenames by positional underscore splitting. It expects the format `{ISO-timestamp}_{task}_{task_id}.eval`. Files not matching this pattern get garbled metadata, causing Inspect View to hide most logs via deduplication on incorrect `task_id` values.

This is a general correctness bug for any `.eval` file produced outside of Inspect's native recorder, not specific to any one project.

## Solution

Conditional header fallback: if filename parsing fails, read `header.json` from the `.eval` ZIP to get correct `task` and `task_id`.

### Design Principles

- No behavior change for native Inspect logs (fast path unchanged)
- Header read only when filename parse fails
- Uses existing `read_eval_log(..., header_only=True)` infrastructure
- Handles local, S3, and GCS filesystems transparently
- No client-side changes required
- No API schema changes

## Branching Strategy

| Branch | Purpose |
|--------|---------|
| `main` | Pure mirror of `UKGovernmentBEIS/inspect_ai:main`. Never diverge. |
| `eruditis-main` | Integration branch. Feature branches merge here for testing. |
| `fix/log-file-info-header-fallback` | Feature branch off `main`. Clean, upstream-ready. |

**Flow:**
1. Branch `fix/log-file-info-header-fallback` from `main`
2. Implement and test on that branch
3. Merge into `eruditis-main` for integration testing with real eval logs
4. Submit PR from feature branch to `UKGovernmentBEIS/inspect_ai:main`
5. Once merged upstream, sync `main` and `eruditis-main`

## Code Change

**Single file:** `src/inspect_ai/log/_file.py`, function `log_file_info()` (lines 707-734).

### Current Logic

```python
def log_file_info(info: FileInfo) -> EvalLogInfo:
    basename = os.path.splitext(info.name)[0]
    parts = basename.split("/").pop().split("_")
    if len(parts) == 1:
        task = ""
        task_id = ""
        suffix = None
    elif len(parts) == 2:
        task = parts[1]
        task_id = ""
        suffix = None
    else:
        last_idx = 3 if len(parts) > 3 else 2
        task = parts[1]
        part3 = parts[last_idx].split("-")
        task_id = part3[0]
        suffix = task_id[2] if len(part3) > 1 else None
    return EvalLogInfo(
        name=info.name, type=info.type, size=info.size,
        mtime=info.mtime, task=task, task_id=task_id, suffix=suffix,
    )
```

This splits on underscores, assumes `parts[0]` is a timestamp, `parts[1]` is task, etc. No validation that `parts[0]` is actually a timestamp.

### Proposed Logic

```python
def log_file_info(info: FileInfo) -> EvalLogInfo:
    basename = os.path.splitext(info.name)[0]
    parts = basename.split("/").pop().split("_")

    # Attempt native filename parse (requires timestamp prefix)
    task, task_id, suffix = _try_parse_filename(parts)

    if task is None:
        # Filename parse failed — fallback to reading eval log header
        task, task_id, suffix = _try_read_header(info.name)

    return EvalLogInfo(
        name=info.name, type=info.type, size=info.size,
        mtime=info.mtime, task=task or "", task_id=task_id or "", suffix=suffix,
    )


def _try_parse_filename(parts: list[str]) -> tuple[str | None, str | None, str | None]:
    """Parse task/task_id from filename parts. Returns (None, None, None) if
    the filename does not match the expected {timestamp}_{task}_{id} pattern."""
    if len(parts) < 2:
        return None, None, None

    # Validate that parts[0] looks like an ISO timestamp
    if not re.match(r"^\d{4}-\d{2}-\d{2}T\d{2}", parts[0]):
        return None, None, None

    # Existing parse logic for valid native filenames
    if len(parts) == 2:
        return parts[1], "", None
    else:
        last_idx = 3 if len(parts) > 3 else 2
        task = parts[1]
        part3 = parts[last_idx].split("-")
        task_id = part3[0]
        suffix = part3[1] if len(part3) > 1 else None
        return task, task_id, suffix


def _try_read_header(name: str) -> tuple[str | None, str | None, str | None]:
    """Attempt to read task/task_id from the eval log header.
    Only called when filename parsing fails. Returns (None, None, None)
    on any error so callers degrade gracefully."""
    try:
        log = read_eval_log(name, header_only=True)
        return log.eval.task, log.eval.task_id, None  # suffix not in header
    except Exception as e:
        logger.debug(f"Failed to read header from {name}: {e}")
        return None, None, None
```

### Key Design Decisions

1. **Timestamp validation gates the fast path.** We check that `parts[0]` matches an ISO timestamp pattern before trusting the filename parse. This is the missing validation that currently lets any filename through the parser.

2. **`read_eval_log` for the fallback.** This reuses Inspect's own infrastructure — handles ZIP reading, remote filesystems (S3/GCS), header-only parsing, and all edge cases. No raw `zipfile` code needed. Note: `read_eval_log` is synchronous. We must verify that `log_file_info` is never called from an async context (e.g., trio). If it is, we may need to use `read_eval_log_async` instead. Initial analysis: `log_file_info` is called from `log_files_from_ls` which is called from both sync (`list_eval_logs`) and async (`list_eval_logs_async`) paths. The async path in `common.py:385` calls `log_files_from_ls` directly (not awaited), so it runs synchronously within the async function. This means `read_eval_log` (sync) should be safe here, but we must confirm during implementation and add a comment.

3. **Extracted helper functions.** `_try_parse_filename` and `_try_read_header` keep `log_file_info` clean and testable.

4. **Graceful degradation with logging.** If both filename parsing and header reading fail, we return `task=""`, `task_id=""`. The viewer shows these as "unknown" rather than crashing. The `_try_read_header` except block logs at `debug` level so failures are diagnosable:
   ```python
   except Exception as e:
       logger.debug(f"Failed to read header from {name}: {e}")
       return None, None, None
   ```

5. **Suffix behavior change (separate concern).** The original code has `suffix = task_id[2] if len(part3) > 1 else None` which takes the 3rd *character* of `task_id`, not the suffix portion after the dash. This is almost certainly a bug in the original. Our proposed code corrects this to `suffix = part3[1] if len(part3) > 1 else None` (the actual suffix string after the dash). This is a pre-existing bug fix and should be called out in the PR as a separate concern from the fallback feature. Consider splitting into a separate commit for reviewer clarity.

6. **Suffix not available from headers.** When the header fallback is used, `suffix` is always `None` because `header.json` does not contain suffix information. This is an accepted limitation — suffix is only used for native Inspect re-scoring workflows (e.g., `-scored` suffix), which always use native filenames.

## Testing Strategy

Tests use TDD red/green/simplify. All tests in the inspect_ai repo.

### Unit Tests

| Test | Input | Expected |
|------|-------|----------|
| Native filename parses correctly | `2026-03-15T11-51-45_g2a-simlex999_abc123.eval` | `task="g2a-simlex999"`, `task_id="abc123"`, no header read |
| Custom filename triggers header fallback | `hf_e5_small_g2a.eval` with `header.json` containing `{"eval": {"task": "g2a_simlex999", "task_id": "xyz"}}` | `task="g2a_simlex999"`, `task_id="xyz"` |
| Corrupt file degrades gracefully | `bad_name.eval` (not a valid ZIP) | `task=""`, `task_id=""`, no exception raised |
| Partial header fields | Header with `task` but no `task_id` | `task="onlytask"`, `task_id` is whatever Pydantic default `EvalLog.eval.task_id` provides (likely `""`) — test must verify actual behavior |
| Determinism | Same file called twice | Identical results |

### Integration Test

Create a temp directory with both native-named and custom-named `.eval` files. Call `list_eval_logs(tmpdir)`. Verify all entries have correct `task`/`task_id`.

### Test Implementation

Tests create real temp `.eval` files (ZIP archives with `header.json` entries) using `zipfile.ZipFile` and `tempfile`. The `FileInfo` objects are constructed from the temp file paths.

## Performance

| Scenario | Path | Overhead per log |
|----------|------|-----------------|
| Native filename (no fallback) | Fast path | Zero — identical to current behavior |
| Custom filename, local disk | Header read | ~2-5 ms (ZIP open + JSON parse) |
| Custom filename, missing header | Failed read | ~1-2 ms (open fails quickly) |
| Custom filename, S3/remote | Header read | ~20-100 ms (network request) |

The header read only triggers for non-conforming filenames. Native Inspect logs see zero change. For typical usage (tens of logs, not thousands), the overhead is negligible.

## What This Does NOT Fix

These are tracked as separate issues:

- **eruditis/inspect_ai#3** — WorkQueue stalls after first batch of 6 eval logs (client-side bug)
- **eruditis/inspect_ai#4** — task_id deduplication from garbled filenames (likely resolved by this fix but needs verification)

## Downstream Impact

**No changes needed in grounding-measure-core:**
- `EvalLogFactory` already writes correct `header.json` with proper `task`/`task_id`
- Filenames don't need to change
- `bundle_and_deploy.py` needs no changes (static bundles already read headers via `write_log_listing()`)

**Verification after merge:**
1. `uv run inspect view start --log-dir benchmarks/results/` — all 85 eval files show correct tasks
2. `uv run inspect view bundle --log-dir benchmarks/results/ --output-dir /tmp/test_bundle/` — verify `listing.json` has correct metadata
3. Optional Playwright screenshot verification

## PR Strategy

**Title:** "Make log_file_info robust to non-standard filenames by falling back to header parsing"

**Positioning:** General correctness fix for non-native eval logs, not project-specific.

**Key points for reviewers:**
- No behavior change for native logs
- Header read only when filename parse fails
- Uses existing `read_eval_log` header-only path
- Maintains performance in common case
- Removes reliance on client repair logic

## Risk & Rollback

- **Low risk:** Change confined to one function with extracted helpers
- **Failure mode:** If header read fails, degrades to old behavior (`task=""`)
- **Rollback:** Revert single commit to restore original `log_file_info()`

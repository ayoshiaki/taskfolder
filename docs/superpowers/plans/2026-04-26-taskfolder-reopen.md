# taskfolder reopen Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Add a `taskfolder reopen <task_id>` subcommand that moves a task's folder from `ARCHIVE_DIR` back to `TODO_DIR` and rewrites the task's annotation to point at the new location.

**Architecture:** Mirror of the existing `archive_folder` function in [taskfolder](../../../taskfolder), implemented as a new `reopen_folder` function plus a new `reopen` argparse subcommand and dispatch branch. Annotation rewrite is performed via `task <id> denotate <old>` followed by `task <id> annotate <new>` after the filesystem move.

**Tech Stack:** Python 3.7+, Taskwarrior CLI (`task`), argparse, `pathlib`, `subprocess`. No tests framework — project has no automated test infrastructure; verification is manual per the spec.

**Spec:** [docs/superpowers/specs/2026-04-26-taskfolder-reopen-design.md](../specs/2026-04-26-taskfolder-reopen-design.md)

---

## Task 1: Add `reopen_folder` function

**Files:**
- Modify: [taskfolder](../../../taskfolder) — insert new function after `archive_folder` (currently ends at line 285).

- [ ] **Step 1: Open the file and locate the insertion point**

Open [taskfolder](../../../taskfolder) and scroll to the end of `archive_folder`. The function ends at line 285 with `logging.error(f"Error moving folder: {e}")`. The next function `list_folders` starts at line 288. Insert the new function between them, separated by two blank lines on each side (matching existing style).

- [ ] **Step 2: Add the `reopen_folder` function**

Insert this function after `archive_folder` and before `list_folders`:

```python
def reopen_folder(number: str) -> None:
    """Moves the linked folder from archive back to the todo directory."""
    info = load_task_info(number)
    if not info:
        logging.error(f"Could not load task {number}")
        return

    folder_path_old = _extract_folder_from_task(info)
    if not folder_path_old:
        logging.warning(f"No folder annotation found for task {number}")
        return

    folder_name = folder_path_old.name
    folder_path_from = ARCHIVE_DIR / folder_name
    folder_path_to = TODO_DIR / folder_name

    # Ensure todo directory exists
    if not TODO_DIR.exists():
        TODO_DIR.mkdir(parents=True, exist_ok=True)

    if not folder_path_from.exists():
        logging.warning(f"Folder not found in archive: {folder_path_from}")
        return

    if folder_path_to.exists():
        logging.error(f"Folder already exists in todo: {folder_path_to}")
        return

    try:
        folder_path_from.rename(folder_path_to)
        logging.info(f"Moved folder back to todo: {folder_path_to}")
    except OSError as e:
        logging.error(f"Error moving folder: {e}")
        return

    # Rewrite annotation: remove old, add new. Move-first ordering means a
    # taskwarrior failure here leaves the filesystem correct; the user is
    # informed via the log and can re-annotate manually.
    try:
        subprocess.run(
            ["task", number, "denotate", str(folder_path_old)], check=True
        )
    except subprocess.CalledProcessError as e:
        logging.error(f"Failed to remove old annotation {folder_path_old}: {e}")

    try:
        subprocess.run(
            ["task", number, "annotate", str(folder_path_to)], check=True
        )
        logging.info(f"Reopened folder: {folder_path_to}")
    except subprocess.CalledProcessError as e:
        logging.error(f"Failed to add new annotation {folder_path_to}: {e}")
```

- [ ] **Step 3: Verify the file still parses**

Run: `python3 -c "import ast; ast.parse(open('taskfolder').read())"`
Expected: no output, exit code 0.

- [ ] **Step 4: Verify the help still works (existing CLI must be intact)**

Run: `./taskfolder --help`
Expected: existing subcommands listed (`create`, `open`, `archive`, `get`, `addurl`, `openurl`, `list`). `reopen` is NOT yet registered — that's the next task.

- [ ] **Step 5: Stage but do not commit yet**

The CLI plumbing in Task 2 is part of the same logical change; commit them together at the end of Task 2.

```bash
git add taskfolder
```

---

## Task 2: Wire up the `reopen` subcommand in `main()`

**Files:**
- Modify: [taskfolder:342-385](../../../taskfolder#L342-L385) — `main()` function.

- [ ] **Step 1: Add the subparser**

Locate the `archive` subparser block in `main()` (currently lines 354-355):

```python
    p_archive = subparsers.add_parser("archive", help="Archive folder for task")
    p_archive.add_argument("number", type=str, help="Task ID or UUID")
```

Immediately after it, insert:

```python
    p_reopen = subparsers.add_parser("reopen", help="Move folder back from archive to todo")
    p_reopen.add_argument("number", type=str, help="Task ID or UUID")
```

- [ ] **Step 2: Add the dispatch branch**

In the dispatch chain at the bottom of `main()`, find the `archive` branch (currently lines 376-377):

```python
    elif args.command == "archive":
        archive_folder(args.number)
```

Immediately after it, insert:

```python
    elif args.command == "reopen":
        reopen_folder(args.number)
```

- [ ] **Step 3: Verify the file still parses**

Run: `python3 -c "import ast; ast.parse(open('taskfolder').read())"`
Expected: no output, exit code 0.

- [ ] **Step 4: Verify `reopen` is now in `--help`**

Run: `./taskfolder --help`
Expected: output includes `reopen` in the subcommand list.

Run: `./taskfolder reopen --help`
Expected: usage shows `reopen number` with the help text "Move folder back from archive to todo".

- [ ] **Step 5: Commit**

```bash
git add taskfolder
git commit -m "$(cat <<'EOF'
feat: add reopen command to move folder from archive to todo

Mirror of `archive` that moves the task's folder back into TODO_DIR and
rewrites the task annotation (denotate + annotate) so subsequent
`taskfolder open` resolves the new location.

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 3: Update CLAUDE.md

**Files:**
- Modify: [CLAUDE.md:11-19](../../../CLAUDE.md#L11-L19) — **Key Commands** code block.

- [ ] **Step 1: Add the new command line**

In the **Key Commands** section, find this line:

```
./taskfolder archive <task_id>   # Move folder to archive
```

Immediately after it, insert:

```
./taskfolder reopen <task_id>    # Move folder back from archive to todo
```

The result should be:

```bash
./taskfolder create <task_id>    # Create folder and annotate task
./taskfolder open <task_id>      # Open the linked folder
./taskfolder archive <task_id>   # Move folder to archive
./taskfolder reopen <task_id>    # Move folder back from archive to todo
./taskfolder get <task_id>       # Print folder path
./taskfolder list                # List all folders and tasks
./taskfolder list -a             # List archived folders and tasks
./taskfolder addurl <task_id> <url>   # Add URL to task
./taskfolder openurl <task_id>   # Open URL(s) from task
```

- [ ] **Step 2: Commit**

`CLAUDE.md` is currently untracked (it appears in `git status` as `??`). Adding it makes it part of the repo for the first time.

```bash
git add CLAUDE.md
git commit -m "$(cat <<'EOF'
docs: track CLAUDE.md and document reopen command

Co-Authored-By: Claude Opus 4.7 (1M context) <noreply@anthropic.com>
EOF
)"
```

---

## Task 4: Manual verification

This task is the spec's manual test plan, executed end-to-end.

**Prerequisites:** Taskwarrior installed; `~/work/todo` and `~/work/archive` writable (or `TASKFOLDER_TODO_DIR` / `TASKFOLDER_ARCHIVE_DIR` set to writable test paths). To avoid contaminating the real task list, optionally point Taskwarrior at a temporary data dir for this run by exporting `TASKDATA=$(mktemp -d)` before starting; restore by unsetting it after.

- [ ] **Step 1: Happy path round-trip**

```bash
TASK_ID=$(task add "test taskfolder reopen" | sed -n 's/.*Created task \([0-9]*\)\..*/\1/p')
./taskfolder create "$TASK_ID"
./taskfolder archive "$TASK_ID"
./taskfolder reopen "$TASK_ID"
```

Expected:
- After `create`: a folder appears in `$TASKFOLDER_TODO_DIR` (default `~/work/todo`) named `<YYYYMMDD>-<8 hex chars>`.
- After `archive`: that folder is now in `$TASKFOLDER_ARCHIVE_DIR` (default `~/work/archive`).
- After `reopen`: that folder is back in `$TASKFOLDER_TODO_DIR`. Logs show `Moved folder back to todo:` and `Reopened folder:`.

Verify the annotation was rewritten:

```bash
task "$TASK_ID" export | python3 -c "import json,sys; t=json.load(sys.stdin)[0]; [print(a['description']) for a in t.get('annotations',[])]"
```

Expected: exactly one annotation, and it contains the path under the todo dir, not the archive dir.

Verify `open` works:

```bash
./taskfolder open "$TASK_ID"
```

Expected: the folder opens in the system file manager. No "Folder path annotated but does not exist" warning.

- [ ] **Step 2: Task without folder annotation**

```bash
TASK_ID2=$(task add "test reopen no folder" | sed -n 's/.*Created task \([0-9]*\)\..*/\1/p')
./taskfolder reopen "$TASK_ID2"
```

Expected: log line `[WARNING] No folder annotation found for task <id>`, exit cleanly, no filesystem changes.

- [ ] **Step 3: Folder not present in archive**

Use the task from Step 1, which now has its folder back in todo:

```bash
./taskfolder reopen "$TASK_ID"
```

Expected: log line `[WARNING] Folder not found in archive: <path>`, no filesystem changes, no annotation changes.

Verify the annotation count is still 1:

```bash
task "$TASK_ID" export | python3 -c "import json,sys; t=json.load(sys.stdin)[0]; print(len(t.get('annotations',[])))"
```

Expected: `1`.

- [ ] **Step 4: Collision in todo**

Set up a collision: archive the task again, then create a colliding folder in todo:

```bash
./taskfolder archive "$TASK_ID"
FOLDER_NAME=$(task "$TASK_ID" export | python3 -c "import json,sys,re; t=json.load(sys.stdin)[0]; print(re.search(r'\d{8}-[0-9a-fA-F]{8}', t['annotations'][0]['description']).group(0))")
mkdir -p "${TASKFOLDER_TODO_DIR:-$HOME/work/todo}/$FOLDER_NAME"
./taskfolder reopen "$TASK_ID"
```

Expected: log line `[ERROR] Folder already exists in todo: <path>`. The archive folder is unchanged. The (empty) folder you created in todo is unchanged.

Cleanup:

```bash
rmdir "${TASKFOLDER_TODO_DIR:-$HOME/work/todo}/$FOLDER_NAME"
./taskfolder reopen "$TASK_ID"
```

Expected: this time the reopen succeeds.

- [ ] **Step 5: Empty archived folder**

Find a task whose folder you can put in archive while empty. The simplest is:

```bash
TASK_ID3=$(task add "test reopen empty" | sed -n 's/.*Created task \([0-9]*\)\..*/\1/p')
./taskfolder create "$TASK_ID3"
FOLDER3=$(./taskfolder get "$TASK_ID3")
mv "$FOLDER3" "${TASKFOLDER_ARCHIVE_DIR:-$HOME/work/archive}/"
./taskfolder reopen "$TASK_ID3"
```

(Note: we move the empty folder by hand because `./taskfolder archive` deletes empty folders rather than moving them — see [taskfolder:273-279](../../../taskfolder#L273-L279).)

Expected: the empty folder is moved back into todo. Annotation is rewritten.

- [ ] **Step 6: Cleanup**

Mark the test tasks done and remove their folders:

```bash
for id in "$TASK_ID" "$TASK_ID2" "$TASK_ID3"; do
  task "$id" done 2>/dev/null || true
done
```

Optionally `rm -rf` the test folders. If you used a temp `TASKDATA`, unset it.

- [ ] **Step 7: Record results**

Confirm in the response that all five test scenarios passed. If any failed, stop and investigate before declaring the work complete.

---

## Self-Review Notes

Spec coverage check (matching against [the spec](../specs/2026-04-26-taskfolder-reopen-design.md)):

- §Behavior steps 1–9 → Task 1 Step 2 (function body covers all nine).
- §CLI changes → Task 2.
- §Documentation → Task 3.
- §Manual test plan items 1–5 → Task 4 Steps 1–5 respectively.
- §Risks (denotate failure / stale annotation surfacing latent bug) → handled by the move-first / log-and-continue ordering in Task 1 Step 2 and the verification in Task 4 Step 1.

No placeholders, no "TBD", every code block contains the actual code. Function name `reopen_folder` and CLI command `reopen` are consistent across all tasks.

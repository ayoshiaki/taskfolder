# taskfolder reopen — Design

**Date:** 2026-04-26
**Status:** Approved for implementation

## Goal

Add a `taskfolder reopen <task_id>` command that moves a task's folder from `ARCHIVE_DIR` back to `TODO_DIR`, undoing a previous `archive`. The task's annotation must end up pointing at the new location so that subsequent `taskfolder open <task_id>` works without manual intervention.

## Motivation

`taskfolder archive` exists but has no inverse. Reopening a task today requires moving the folder by hand and re-annotating the task. A first-class `reopen` makes that round-trip symmetric and removes a manual recovery step when a previously archived task becomes active again.

## Scope

In scope:
- New `reopen_folder(number)` function in [taskfolder](../../../taskfolder).
- New `reopen` subcommand in the argparse CLI.
- Annotation rewrite: remove the old folder annotation and add the new one (option C from brainstorming).
- Update the **Key Commands** section of [CLAUDE.md](../../../CLAUDE.md).

Out of scope (intentional):
- Updating `archive_folder` to also rewrite annotations. The same staleness exists there but is treated as a separate follow-up.
- Updating `_extract_folder_from_task` to prefer paths that exist on disk.
- Any test infrastructure (the project has none today).

## Behavior

```
./taskfolder reopen <task_id>
```

1. Load task info via `load_task_info(number)`. If missing, log error and return.
2. Extract the existing folder path from the task's annotations via `_extract_folder_from_task(info)`. If absent, log a warning ("No folder annotation found for task <id>") and return.
3. Derive `folder_name = folder_path_old.name`. Use only the basename — the annotated path may point to any historical location, but the basename is the stable identifier.
4. Compute `folder_path_from = ARCHIVE_DIR / folder_name` and `folder_path_to = TODO_DIR / folder_name`.
5. Ensure `TODO_DIR` exists: `TODO_DIR.mkdir(parents=True, exist_ok=True)`.
6. Validate:
   - If `folder_path_from` does not exist → log warning `"Folder not found in archive: <path>"` and return.
   - If `folder_path_to` already exists → log error `"Folder already exists in todo: <path>"` and return. **Do not overwrite.**
7. Move the folder: `folder_path_from.rename(folder_path_to)`. On `OSError`, log error and return.
8. Rewrite the task annotation:
   1. `subprocess.run(["task", number, "denotate", str(folder_path_old)], check=True)` — remove the stale annotation.
   2. `subprocess.run(["task", number, "annotate", str(folder_path_to)], check=True)` — add the new one.
   - If denotate fails (`CalledProcessError`): log error, but still attempt annotate. The folder will have moved correctly; the task may end up with two folder annotations. This is acceptable degradation and is logged.
   - If annotate fails: log error. The folder has moved but the task points at a stale location. The user is informed via the log and can re-annotate manually.
9. Log a final success line on the happy path: `"Reopened folder: <folder_path_to>"`.

### Order rationale

`move → denotate → annotate`. Moving first ensures that even if Taskwarrior commands fail, the filesystem state is correct. Denotate before annotate avoids leaving the task with two folder annotations on the happy path; on failure we still attempt annotate so the task at minimum gains a valid pointer.

## CLI changes

In `main()`, add:

```python
p_reopen = subparsers.add_parser("reopen", help="Move folder back from archive to todo")
p_reopen.add_argument("number", type=str, help="Task ID or UUID")
```

And the dispatch branch:

```python
elif args.command == "reopen":
    reopen_folder(args.number)
```

## Documentation

Update the **Key Commands** section of [CLAUDE.md](../../../CLAUDE.md) by adding:

```
./taskfolder reopen <task_id>   # Move folder back from archive to todo
```

immediately after the `archive` line so related operations are grouped.

## Manual test plan

No automated tests exist in the project; verify by hand:

1. **Happy path round-trip**
   - Create a task: `task add "test reopen"`.
   - `./taskfolder create <id>` → folder appears in `~/work/todo/<YYYYMMDD-uuid>`.
   - `./taskfolder archive <id>` → folder moves to `~/work/archive/...`.
   - `./taskfolder reopen <id>` → folder is back in `~/work/todo/...`, task export shows a single folder annotation pointing at the new path, `./taskfolder open <id>` opens it.
2. **Task without folder annotation**
   - On a task that has no folder annotation, `./taskfolder reopen <id>` logs the "No folder annotation found" warning and exits cleanly.
3. **Folder not present in archive**
   - On a task whose annotated folder is not in `ARCHIVE_DIR`, `./taskfolder reopen <id>` logs "Folder not found in archive: ..." and does not modify annotations.
4. **Collision in todo**
   - Manually create `~/work/todo/<same-name>` so it collides with the archived folder, then run `./taskfolder reopen <id>`. Expect the "Folder already exists in todo" error and no move.
5. **Empty archived folder**
   - Archive an empty folder (currently `archive` deletes empty folders rather than moving them, so set this up by manually placing an empty folder in `ARCHIVE_DIR` whose name matches a task's annotation), then `reopen`. Expect the empty folder to be moved back into `TODO_DIR`.

## Risks and mitigations

- **Denotate matching.** `task <id> denotate <text>` matches the annotation containing `<text>`. We pass the full path string. If a user has manually edited the annotation in a way that changes the exact path string, denotate will fail — handled by the "log and continue" path above.
- **Stale annotation surfaces a known latent bug.** `_extract_folder_from_task` returns the first matching annotation, ordered by entry timestamp ascending. If denotate fails and annotate succeeds, the task ends up with the old (stale) annotation first — meaning `taskfolder open` will still try the old path. This is the existing behavior for `archive` too and is out of scope here. Documented as a follow-up.

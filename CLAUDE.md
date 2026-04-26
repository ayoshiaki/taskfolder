# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Project Overview

`taskfolder` is a command-line tool that manages folders linked to Taskwarrior tasks. It's a single-file Python script that creates uniquely-named folders (`YYYYMMDD-UUID` format) for tasks and annotates them in Taskwarrior. The tool also handles URL management, folder archiving, and listing.

## Key Commands

Run the script directly (it's executable):
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

No build process required - it's a standalone Python script.

## Architecture

This is a single-file application (taskfolder:1-388) with a functional architecture:

### Core Data Flow
1. **Task Information Retrieval** (`load_task_info()` at :32): Executes `task <id> export` and parses JSON to get task data from Taskwarrior
2. **Annotation Parsing** (`_extract_folder_from_task()` at :49, `_extract_urls_from_task()` at :59): Extracts folder paths and URLs from task annotations using regex patterns
3. **File System Operations**: Creates/moves folders in configurable TODO_DIR and ARCHIVE_DIR locations

### Key Patterns
- **Configuration via Environment**: TODO_DIR and ARCHIVE_DIR are set via `TASKFOLDER_TODO_DIR` and `TASKFOLDER_ARCHIVE_DIR` environment variables, defaulting to `~/work/todo` and `~/work/archive`
- **Regex Extraction**: Uses `FOLDER_PATH_PATTERN` (:24) to find folder paths in annotations and `URL_PATTERN` (:28) for URLs
- **Cross-Platform Support**: `open_path()` (:70) and `open_url()` (:88) handle macOS/Linux/Windows differences
- **Taskwarrior Integration**: All task operations use subprocess calls to the `task` command with JSON export/import

### Folder Naming Convention
Folders use the format `YYYYMMDD-UUID` where:
- `YYYYMMDD` is the creation date
- `UUID` is the first segment of the task's UUID (before the first hyphen)

### URL Management
- URLs are stored as task annotations
- Multiple URLs per task supported with interactive selection on open
- Duplicate URL prevention built-in

## Important Constraints

- Requires Taskwarrior installed and configured
- Python 3.7+ required for type hints
- Folder paths are stored in task annotations, so changing directory config requires updating existing task annotations
- The tool relies on pattern matching annotations - manual annotation edits must preserve the folder path format

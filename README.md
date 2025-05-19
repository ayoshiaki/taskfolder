# Taskfolder

`taskfolder` is a command-line tool to manage folders associated with
[Taskwarrior](https://taskwarrior.org/) tasks.

Each task receives a uniquely named folder for organizing related documents and
files. The tool ensures consistency, supports automatic folder creation and
annotation, and also provides archiving and listing features.

## Features

- Create folders named `YYYYMMDD-UUID` for individual Taskwarrior tasks
- Annotate tasks with the corresponding folder path
- Open task folders from the command line
- Move folders to an archive location
- List all existing folders with task IDs and descriptions

## Requirements

- Python 3.7 or later
- Taskwarrior 2.x or 3.x
- Works on macOS, Linux, and Windows (via WSL)

## Installation

Clone the repository and make the script executable:

```bash
git clone https://github.com/youruser/taskfolder.git
cd taskfolder
chmod +x taskfolder
```

Optionally move it to your PATH:

```bash
sudo mv taskfolder /usr/local/bin/
```

## Configuration

By default, folders are created under:

- `~/work/todo` (for active folders)
- `~/work/archive` (for archived folders)

You can override these using environment variables:

```bash
export TASKFOLDER_TODO_DIR="$HOME/path/to/todo"
export TASKFOLDER_ARCHIVE_DIR="$HOME/path/to/archive"
```

## Usage

```bash
taskfolder create <task_id>     # Create and link a folder to a task
taskfolder open <task_id>       # Open the folder linked to a task
taskfolder archive <task_id>    # Move the folder to the archive
taskfolder get <task_id>        # Print the folder path linked to the task
taskfolder list                 # List folders and associated tasks
```

## Example

```bash
task add "Prepare project report"
taskfolder create 12
taskfolder open 12
```

## License

MIT License

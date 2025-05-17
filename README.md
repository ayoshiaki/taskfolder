# taskfolder

A simple CLI utility to manage folders linked to [Taskwarrior](https://taskwarrior.org/) tasks.  
It automatically creates, opens, archives, and lists task-specific folders on your local filesystem, while annotating them in Taskwarrior for reference.

## Features

- Compatible with Taskwarrior 2.x and 3.x
- Creates a task-specific folder using the task's UUID
- Annotates the task with the folder path
- Opens folders via system call
- Moves completed folders to an archive directory
- Lists all folders and shows their linked task descriptions

## Requirements

- Python 3.7+
- [Taskwarrior](https://taskwarrior.org/) installed and configured
- macOS, Linux, or Windows (with minor adaptation)

## Folder Structure

- Task folders are created in: `~/work/todo/`
- Archived folders are moved to: `~/work/archive/`
- Folder names follow the pattern: `YYYYMMDD-XXXXXXXX` (date + shortened UUID)

## Usage

```bash
./taskfolder create <task_id>   # Create and annotate folder for a task
./taskfolder open <task_id>     # Open folder linked to a task
./taskfolder get <task_id>      # Print folder path linked to a task
./taskfolder archive <task_id>  # Move folder to archive if not empty
./taskfolder list               # List all folders and linked descriptions


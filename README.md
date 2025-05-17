# TaskFolder Manager

A lightweight Python CLI tool to manage local folders linked to [Taskwarrior](https://taskwarrior.org/) tasks. Useful for organizing documents and resources associated with each task.

## Features

- **create**: Creates a folder based on the task’s UUID and annotates the task with its path.
- **open**: Opens the folder linked to the task (searches in `todo/` and `archive/` directories).
- **archive**: Moves the task’s folder from the `todo/` directory to the `archive/` directory.
- **get**: Prints the path of the folder associated with a given task.
- **list**: Lists all existing folders in `todo/` along with their respective task descriptions.

## Requirements

- Python 3.6 or higher  
- [Taskwarrior](https://taskwarrior.org/)
- macOS (uses `open`) — see note below for Linux support  
- Directory structure:
  - `/Users/yoshiaki/work/todo/`
  - `/Users/yoshiaki/work/archive/`

## Installation

Clone the repository:

```bash
git clone https://github.com/your-username/taskfolder-manager.git
cd taskfolder-manager


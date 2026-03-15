# Unix-Based Treasure Hunt System

A filesystem-driven treasure hunt management system written in C for Unix/Linux environments. The project demonstrates core systems programming concepts including process management, inter-process communication, signals, pipes, binary file I/O, and directory traversal.

---

## Overview

The system allows users to create and manage **treasure hunts** — each hunt is a directory on the filesystem containing a binary `.dat` file that stores structured treasure records. Users interact with the system through an interactive **Treasure Hub** CLI, which delegates operations to a background **Monitor** process or spawns child processes as needed. A separate **Score Calculator** program aggregates per-user scores for any given hunt.

---

## Architecture

The system is composed of four cooperating programs:

```
┌─────────────────────────────────────────────────────┐
│                   Treasure Hub                      │
│         (Interactive CLI — treasure_hub)            │
│                                                     │
│  ┌──────────────┐         ┌───────────────────────┐ │
│  │   Commands   │─SIGUSR1─▶   Monitor Process     │ │
│  │  (via file)  │◀──pipe──│   (./monitor)         │ │
│  └──────────────┘         └──────────┬────────────┘ │
│                                      │ fork/exec    │
│                            ┌─────────▼────────────┐ │
│                            │  Treasure Manager    │ │
│                            │  (./treasure_manager)│ │
│                            └──────────────────────┘ │
│                                                     │
│  ┌──────────────────────────────────────────────┐   │
│  │         Score Calculator (fork/exec)         │   │
│  │         (./score_calculator <hunt_id>)       │   │
│  └──────────────────────────────────────────────┘   │
└─────────────────────────────────────────────────────┘
```

### Treasure Hub (`treasure_hub.c`)

The central interactive controller. It:
- Reads commands from `stdin` in a read-eval loop.
- Starts and manages the monitor process via `fork`/`exec`.
- Forwards monitor commands by writing to a shared file (`.monitor_cmd`) and signaling the monitor with `SIGUSR1`.
- Reads monitor output through a pipe using a dedicated `pthread` reader thread.
- Spawns independent `score_calculator` child processes (one per hunt) for `calculate_score`, collecting output via pipes in order.
- Handles `SIGCHLD` to detect monitor process termination.

### Monitor (`monitor.c`)

A long-running background process that:
- Sits in an infinite loop waiting for signals.
- On `SIGUSR1`: reads the command from `.monitor_cmd` and dispatches to `list_hunts`, `list_treasures`, or `view_treasure`.
- On `SIGTERM`: sleeps for 5 seconds and exits cleanly.
- For `list_treasures` and `view_treasure`, forks a child and `exec`s `./treasure_manager` to perform the actual operation.
- Writes all output to its `stdout`, which is piped back to the hub.

### Treasure Manager (`treasure_manager.c`, `operations.c`)

A standalone command-line utility for direct hunt and treasure manipulation. It:
- Parses command-line arguments and dispatches to the appropriate operation.
- Reads treasure input interactively from `stdin`, or parses a pre-formatted multi-line string if `stdin` has data ready (detected via `select`).
- Performs all I/O using low-level POSIX calls (`open`, `read`, `write`, `close`).
- Appends a log entry to `<hunt_id>/logged_hunt.txt` after each operation.
- Maintains a symbolic link `logged_hunt--<hunt_id>` pointing to each hunt's log file.

### Score Calculator (`score_calculator.c`)

A self-contained program that:
- Opens the `.dat` file for a specified hunt.
- Reads all treasure records and aggregates `value` totals per `user_name`.
- Prints a formatted score table to `stdout`.
- Uses dynamic memory allocation (`realloc`) to grow the user-score array as needed.

---

## Data Model

### Hunt

A hunt is a **directory** on the filesystem named by its `hunt_id`. It contains:

| File | Description |
|---|---|
| `<hunt_id>.dat` | Binary file; a flat array of `Treasure_t` structs |
| `logged_hunt.txt` | Append-only operation log with timestamps |

A symbolic link `logged_hunt--<hunt_id>` is created in the working directory pointing to the hunt's log file.

### Treasure Record

Each treasure is stored as a fixed-size binary struct:

```c
typedef struct treasure {
    char   treasure_id[20];   // Unique identifier within the hunt
    char   user_name[20];     // Owner/submitter username
    double coordinateX;       // Geographic or map X coordinate
    double coordinateY;       // Geographic or map Y coordinate
    char   clue[128];         // Text clue associated with the treasure
    int    value;             // Point value of the treasure
} Treasure_t;
```

Records are appended sequentially. There is no index; operations scan the file linearly.

---

## Project Structure

```
.
├── treasure_hub.c       # Interactive hub — main entry point for the user
├── monitor.c            # Background monitor process
├── monitor_state.c/.h   # Shared monitor state (pid, running state)
├── treasure_manager.c   # CLI tool for hunt/treasure management
├── operations.c/.h      # Implementation of add/list/view/remove operations
├── treasure.c/.h        # Treasure struct definition and input parsing
├── score_calculator.c   # Standalone score aggregation program
├── customs.c/.h         # Shared utility functions and common includes
└── makefile             # Build configuration
```

### Key Shared State

`monitor_state.h` defines a globally accessible `Monitor_t` struct:

```c
typedef struct {
    pid_t          pid;    // PID of the running monitor process
    monitor_state  state;  // OFFLINE | RUNNING | SHUTTING_DOWN
} Monitor_t;

extern Monitor_t *monitor_ex;
```

This is used by the hub to track whether the monitor is running and to send signals to it.

---

## How the System Works

1. The user launches `./treasure_hub`.
2. The hub displays a `>` prompt and waits for input.
3. **`start_monitor`** forks a child, which `exec`s `./monitor`. A pipe is set up; the monitor's `stdout` is redirected into the write end. A reader thread in the hub drains the read end and prints to the terminal.
4. Commands like `list_hunts`, `list_treasures <hunt_id>`, and `view_treasure <hunt_id> <treasure_id>` are sent to the monitor by:
   - Writing the command string to `.monitor_cmd`.
   - Sending `SIGUSR1` to the monitor process.
5. The monitor's signal handler reads `.monitor_cmd`, parses the command, and executes it — optionally by `fork`/`exec`-ing `treasure_manager`.
6. **`calculate_score`** is handled entirely by the hub: it scans the current directory for hunt subdirectories, forks a `score_calculator` child per hunt, and reads output from each child's pipe sequentially.
7. **`stop_monitor`** sends `SIGTERM` to the monitor, closes the pipe read end (causing the reader thread to hit EOF and exit), and detaches the reader thread.

---

## Commands

### Treasure Hub Commands

| Command | Requires Monitor | Description |
|---|---|---|
| `help` | No | Display available commands |
| `start_monitor` | No | Launch the monitor background process |
| `list_hunts` | Yes | List all hunts and their treasure counts |
| `list_treasures <hunt_id>` | Yes | List all treasures in a hunt |
| `view_treasure <hunt_id> <treasure_id>` | Yes | Display full details of a treasure |
| `calculate_score` | No | Aggregate and print per-user scores for all hunts |
| `stop_monitor` | Yes | Send SIGTERM to the monitor and shut it down |
| `exit` | No | Exit the hub (monitor must be stopped first) |

### Treasure Manager Commands

```
Usage: treasure_manager <command> [arguments]
```

| Command | Arguments | Description |
|---|---|---|
| `--help` | — | Display usage information |
| `--add` | `<hunt_id>` | Add a new treasure to the specified hunt |
| `--list` | `<hunt_id>` | List all treasure IDs in the hunt |
| `--view` | `<hunt_id> <treasure_id>` | Print full details of a specific treasure |
| `--remove_treasure` | `<hunt_id> <treasure_id>` | Remove a treasure by ID |
| `--remove_hunt` | `<hunt_id>` | Delete the entire hunt directory and its symlink |

---

## Compilation

Use the provided `makefile`:

```bash
make
```

This produces four executables in the project directory:

```
./treasure_hub
./monitor
./treasure_manager
./score_calculator
```

All four executables must be present in the same working directory at runtime.

To clean build artifacts:

```bash
make clean
```

---

## Usage Examples

### Starting the hub

```bash
./treasure_hub
```

### Starting the monitor and listing hunts

```
> start_monitor
Monitor started successfully.
> list_hunts
Hunt: hunt001 | Treasures: 3
Hunt: hunt002 | Treasures: 1
```

### Adding a treasure interactively

```bash
./treasure_manager --add hunt001
Give treasure id: t42
Give treasure username: alice
Give treasure coordX: 44.5
Give treasure coordY: 26.1
Give treasure clue: Under the old oak tree
Give treasure value: 150
```

### Adding a treasure via piped input

```bash
printf "t43\nbob\n12.3\n45.6\nBehind the waterfall\n200\n" | ./treasure_manager --add hunt001
```

### Viewing a treasure

```bash
./treasure_manager --view hunt001 t42
```

### Calculating scores from the hub

```
> calculate_score

Hunt: hunt001
alice: 300
bob: 200
```

### Removing a treasure and a hunt

```bash
./treasure_manager --remove_treasure hunt001 t42
./treasure_manager --remove_hunt hunt001
```

---

## Logging and Filesystem Layout

After adding treasures to `hunt001` and `hunt002`, the working directory will contain:

```
.
├── treasure_hub
├── monitor
├── treasure_manager
├── score_calculator
├── .monitor_cmd                    ← Temporary command file used for IPC
├── logged_hunt--hunt001            ← Symlink → hunt001/logged_hunt.txt
├── logged_hunt--hunt002            ← Symlink → hunt002/logged_hunt.txt
├── hunt001/
│   ├── hunt001.dat                 ← Binary treasure records
│   └── logged_hunt.txt             ← Timestamped operation log
└── hunt002/
    ├── hunt002.dat
    └── logged_hunt.txt
```

Each log entry follows the format:

```
[DD-MM-YYYY HH:MM:SS] OPERATION: DETAILS
```

Example:

```
[14-06-2025 10:23:41] Add hunt: Added treasure with ID - t42
[14-06-2025 10:25:02] List hunt: Listed hunt with ID - hunt001
```

The symlinks are removed automatically when `--remove_hunt` is called.

---

## Concepts Demonstrated

| Concept | Where Used |
|---|---|
| Binary file I/O | `.dat` files storing `Treasure_t` structs |
| Directory manipulation | `opendir`, `readdir`, `mkdir`, `rmdir` |
| Process creation | `fork` + `execv`/`execlp`/`execlp` throughout |
| Pipes | Hub↔Monitor output forwarding; hub↔score_calculator |
| POSIX signals | `SIGUSR1` (command trigger), `SIGTERM` (shutdown), `SIGCHLD` (reap) |
| `sigaction` | Signal handler registration in both hub and monitor |
| Threads (`pthread`) | Reader thread in hub draining monitor pipe |
| Symbolic links | `symlink`/`unlink` for log file shortcuts |
| `select` for I/O detection | Non-blocking stdin check in `add()` |
| Dynamic memory | `realloc`-based growing array in score calculator |
| `stat` | File size and modification time in `list()` |
| Low-level I/O only | All I/O via `read`/`write`; no `stdio` for data paths |

---

## Limitations

- **No concurrency control**: The `.dat` files have no locking. Concurrent writes from multiple `treasure_manager` instances would produce a corrupted file.
- **Fixed-size fields**: `treasure_id`, `user_name`, and `clue` are fixed-length char arrays. Input that exceeds these sizes is silently truncated.
- **Linear scan**: All lookups (view, remove, duplicate check) scan the entire `.dat` file sequentially — O(n) per operation.
- **Single monitor instance**: The hub tracks only one monitor PID. Calling `start_monitor` twice is blocked, but state recovery after a crash is not implemented.
- **`.monitor_cmd` race condition**: There is no synchronization between the hub writing to `.monitor_cmd` and the monitor reading it beyond the `SIGUSR1` signal delivery order.
- **No persistence of hub state**: The `monitor_ex` state is in-process memory. Restarting the hub loses track of any previously running monitor.
- **`handle_term` blocks for 5 seconds**: The monitor's `SIGTERM` handler unconditionally calls `sleep(5)` before exiting, which may delay hub shutdown.
- **`ERROR_BUFFER_SIZE` is 20 bytes**: The error buffer in `abandonCSTM()` is very small and may truncate longer `strerror` messages.

---

## Possible Improvements

- **File locking** (`flock` or `fcntl`) on `.dat` files to prevent concurrent corruption.
- **Index file or in-memory hash map** to speed up lookups in large hunts.
- **Structured IPC** replacing the `.monitor_cmd` file approach with a proper named pipe (FIFO) or socket to eliminate the file-based race condition.
- **Graceful monitor restart** — detect a crashed monitor via `SIGCHLD` and allow re-launch without restarting the hub.
- **Variable-length fields** using a record-length prefix or a separate index, to support longer IDs, names, and clues.
- **Configurable hunt root** — currently the system operates only in the current working directory. A base path argument would make the tool more flexible.
- **`calculate_score` via monitor** — currently bypasses the monitor entirely; routing it through the monitor would make the architecture more consistent.
- **Input validation on hub commands** — the hub passes the raw input string to handlers; malformed arguments (e.g., missing `treasure_id` for `view_treasure`) can cause undefined behavior in the monitor.

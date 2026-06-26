# MaqOS — Maqsad Operating System

A fully simulated desktop operating system built from scratch in C++17 with SFML. MaqOS implements real OS internals — process creation via `fork()`/`exec()`, a three-level multilevel queue scheduler, POSIX semaphore-protected memory management, pipe-based IPC, deadlock detection, and kernel/user mode privilege separation — wrapped in a Windows 11-inspired graphical desktop environment running on Linux.

---

## What It Does

MaqOS boots from the command line with configurable hardware parameters (RAM, HDD, CPU cores), runs an animated splash screen, presents a login screen (default: `admin` / `admin`), then launches a full SFML desktop. Every application runs as a real child process spawned via `fork()` + `exec()`, with a two-pipe IPC handshake negotiating RAM and HDD allocation before the child process is allowed to execute. A background scheduler thread manages a three-level process queue with priority aging, a deadlock detector runs a DFS on the Resource Allocation Graph every 5 seconds, and a live system monitor dashboard shows real-time RAM and CPU usage on the desktop.

---

## System Architecture

MaqOS follows a four-layer architecture:

| Layer | Components | Role |
|---|---|---|
| **User Mode** | 19 apps — games, file ops, media, utilities, shell | Child processes launched via fork/exec |
| **Kernel Mode** | Process manager, scheduler, memory manager, sync primitives | Core OS services |
| **HAL** | Hardware Abstraction Layer | Translates kernel calls to simulated hardware |
| **Simulated Hardware** | RAM, HDD, CPU cores (user-supplied at boot), system log | User-configured limits |

Boot sequence: `startup.cpp` → login → `main.cpp` (with RAM, HDD, Cores as args)  
Shutdown: SIGTERM to all processes → free resources → close log → exit window

---

## Features

- Animated boot splash with progress bar, then lock/login screen with masked password field
- SFML desktop at 1280×720 (resizable) with custom wallpaper, glassmorphism taskbar, and Start Menu
- Real-time System Monitor dashboard (RAM bar in green, CPU/thread bar in orange)
- 19 applications arranged in a 6-per-row icon grid — left-click launches, right-click minimizes/restores
- Windows 11-style Start Menu with search bar, app grid, Set Wallpaper (via Zenity file dialog), and shutdown
- F1 help overlay listing all keyboard shortcuts and OS features
- Kernel Mode / User Mode toggle in the taskbar system tray (green dot = User, red dot = Kernel)
- Kernel Panel (`K`) accessible in Kernel Mode — force-kill processes, adjust priorities, view resource allocation matrix
- Per-process resource allocation enforced: apps denied launch if RAM or HDD is insufficient
- Full system event log (`maqos_system.log`) with timestamped, mutex-protected writes

---

## Installed Applications (19)

| Category | Application | Queue | RAM | HDD | Priority |
|---|---|---|---|---|---|
| Productivity | Calculator | Q1 Interactive | 80 MB | 5 MB | 2 |
| Productivity | Notepad (pthread auto-save every 5s) | Q1 Interactive | 100 MB | 20 MB | 2 |
| Productivity | Browser (URL bar + shortcut tiles) | Q2 Background | 200 MB | 50 MB | 4 |
| System | Task Manager (reads real `/proc`) | Q0 System | 120 MB | 5 MB | 1 |
| System | Clock (auto-started on boot) | Q0 System | 40 MB | 2 MB | 0 |
| System | Calendar (auto-started on boot) | Q0 System | 60 MB | 5 MB | 0 |
| System | Stopwatch | Q1 Interactive | 50 MB | 2 MB | 2 |
| File Management | File Create | Q2 Background | 70 MB | 10 MB | 4 |
| File Management | File Delete | Q2 Background | 70 MB | 5 MB | 4 |
| File Management | File Info (via `stat()`) | Q2 Background | 60 MB | 5 MB | 4 |
| File Management | File Copy | Q2 Background | 150 MB | 50 MB | 4 |
| File Management | File Move (Cut) | Q2 Background | 150 MB | 50 MB | 4 |
| Media | Audio Player | Q2 Background | 100 MB | 30 MB | 4 |
| Media | Video Player (libVLC) | Q1 Interactive | 300 MB | 100 MB | 2 |
| Games | Chess (move validation + check detection) | Q1 Interactive | 150 MB | 15 MB | 2 |
| Games | Snake (high-score persistence) | Q1 Interactive | 100 MB | 8 MB | 2 |
| Games | Minesweeper (recursive reveal + flags) | Q1 Interactive | 120 MB | 10 MB | 2 |
| Games | Tic-Tac-Toe | Q1 Interactive | 80 MB | 8 MB | 2 |
| Games | Number Guess | Q1 Interactive | 60 MB | 5 MB | 2 |

---

## OS Concepts Implemented

| Concept | Implementation |
|---|---|
| Process creation | `fork()` + `exec()` with two-pipe IPC handshake for RAM/HDD negotiation before exec |
| Process states | READY → RUNNING → MINIMIZED / BLOCKED (I/O) → TERMINATED |
| CPU scheduling | 3-level MLQ: Q0 FCFS (system), Q1 Priority + Round Robin 2s quantum (interactive), Q2 FCFS (background) |
| Round Robin | Detached thread simulates 2s quantum expiry for Q1, logs `RR QUANTUM EXPIRED` |
| Priority scheduling | `std::priority_queue` (min-heap) for Q1, ordered by `ProcessEntry.priority` |
| Priority aging | `wait_ticks` counter; every 10 ticks → priority decremented by 1 (starvation prevention) |
| Context switching | `context_switch()` logs each transition; SIGSTOP/SIGCONT used for preemption |
| Semaphores | `sem_t ram_sem` (binary, protects RAM pool), `sem_t core_sem` (counting, limits concurrent processes to core count) |
| Mutex / critical sections | `std::mutex` for process table, scheduler queues, log file, RAM counter |
| Memory management | Per-process RAM + HDD tracked in `ProcessEntry`; atomic allocation/free on termination |
| Deadlock detection | Resource Allocation Graph + DFS cycle detection, background pthread every 5s |
| IPC — pipes | Anonymous pipes (`req_pipe`, `ack_pipe`) for parent-child resource request/ACK before exec |
| IPC — signals | SIGSTOP (minimize/block), SIGCONT (restore/resume), SIGTERM (close), SIGKILL (kernel mode admin) |
| Kernel / user mode | Privilege toggle; Kernel Mode auto-launches Kernel Panel with elevated admin access |
| I/O interrupt simulation | Key `I` → BLOCKED (SIGSTOP); detached thread auto-resolves after 3s (SIGCONT) |
| Child reaping | `waitpid(WNOHANG)` every frame prevents zombie processes |
| Multithreading | pthreads for scheduler daemon, deadlock monitor, Notepad auto-save |
| File system ops | Create, read, delete, copy, move — full POSIX file API |
| System logging | Thread-safe singleton logger (`logger.h`) writing timestamped `[INFO/WARN/ERROR/DEBUG]` entries to both `maqos_system.log` and stdout |
| Task Manager | Reads real `/proc/<pid>/stat` and `/proc/meminfo` for live CPU and memory metrics |

---

## Tech Stack

- **Language:** C++17
- **Graphics & windowing:** SFML 2.x (Graphics, Window, System, Audio)
- **Video playback:** libVLC / ffplay
- **Browser:** GTKmm 3 / `xdg-open`
- **Concurrency:** POSIX pthreads, `std::mutex`, `std::condition_variable`, `<semaphore.h>`
- **Platform:** Linux (Ubuntu/Debian — uses POSIX APIs and `/proc` filesystem throughout)
- **Build:** GNU Make (auto-installs all dependencies on first build)

---

## How to Run

**Platform:** Linux only (Ubuntu/Debian recommended)

**Step 1 — Build** (auto-installs all dependencies):

```bash
make all
```

Required packages installed automatically: `build-essential`, `libsfml-dev`, `ffmpeg`, `vlc`, `zenity`, `pkg-config`, `libgtkmm-3.0-dev`

**Step 2 — Launch**

Full boot sequence (recommended — animated splash → login → desktop):

```bash
./bin/startup
```

Direct launch with custom hardware (skip boot splash):

```bash
./bin/main <RAM_GB> <HDD_GB> <Cores>
# Example: ./bin/main 4 256 8
```

| Parameter | Effect |
|---|---|
| `RAM_GB` | Sets simulated RAM pool in MB. Binary semaphore protects allocation. |
| `HDD_GB` | Sets simulated HDD pool in MB. Atomic counter tracks usage. |
| `Cores` | Initialises counting semaphore — limits max concurrent processes. |

**Default login:** `admin` / `admin`

**Clean build artifacts:**

```bash
make clean
```

---

## Runtime Controls

| Input | Action |
|---|---|
| Left-click desktop icon | Launch app as new process (fork + exec) |
| Right-click desktop icon | Minimize (SIGSTOP) or restore (SIGCONT) running instance |
| Left-click taskbar app | Minimize or restore that process |
| Right-click taskbar app | Send SIGTERM — close the app |
| Start Button | Open / close Start Menu |
| Start Menu → Power button | Initiate MaqOS shutdown sequence |
| Start Menu → Set Wallpaper | Open Zenity file dialog to choose wallpaper |
| Kernel Mode button | Toggle User Mode ↔ Kernel Mode |
| Key `I` | Simulate I/O interrupt on first running process (BLOCKED for 3s) |
| Key `F1` | Toggle on-screen instruction guide overlay |
| Window resize | Desktop auto-scales wallpaper and SFML view |

---

## Repository Structure

```
MaqOS/
├── Makefile
├── README.md
├── .gitignore
├── src/
│   ├── main.cpp          # Desktop shell, scheduler, IPC, resource manager
│   ├── logger.h          # Thread-safe singleton logger
│   ├── startup.cpp       # Boot splash + login screen
│   ├── shutdown.cpp
│   ├── K.cpp             # Kernel Panel (elevated privileges)
│   ├── task_manager.cpp
│   ├── notepad.cpp
│   ├── calculator.cpp
│   ├── browse.cpp
│   ├── clock.cpp
│   ├── simple_calendar.cpp
│   ├── stopWatch.cpp
│   ├── file_browser.cpp
│   ├── File_copy.cpp
│   ├── file_cut.cpp
│   ├── fileCreate.cpp
│   ├── fileDelete.cpp
│   ├── fileInfo.cpp
│   ├── localDisk.cpp
│   ├── audioplayer.cpp
│   ├── VideoPlayer.cpp
│   ├── snake.cpp
│   ├── minesweeper.cpp
│   ├── chess.cpp
│   ├── ticTacToc.cpp
│   ├── numberGuess.cpp
│   ├── gif_loader.cpp
│   ├── file.cpp
│   └── test.cpp
├── assets/
│   ├── fonts/
│   └── icons/ & images/
└── docs/
    └── MaqOS_Project_Report.pdf
```

---

## Team

| Roll No. | Name |
|---|---|
| F24-3122 | Junaid Haider |
| F24-3062 | Taha Rehan |
| F24-3097 | Muhammad Abdullah |

---

## What We Learned

MaqOS pushed all three of us to stop treating OS theory as abstract and implement it against real constraints. Writing a scheduler that actually decides which process runs next — with three queue levels, Round Robin preemption, and aging so background tasks don't starve indefinitely — is a completely different problem from understanding the algorithm on paper. The IPC pipe handshake was one of the most concrete moments: every single app launch involves a real kernel-managed communication channel negotiating resources before `exec()` replaces the child's memory space. Getting all of it thread-safe simultaneously — the scheduler pthread, the deadlock monitor, the SFML render loop, and up to 19 concurrent child processes, all sharing the same process table and resource pools through carefully chosen mutexes, semaphores, and condition variables — was where the real learning happened. The deadlock detector running DFS on a live Resource Allocation Graph every 5 seconds made cycle detection feel like a real system responsibility rather than an exam question.

# Local Storage vs Session Storage

## Core Difference: Data Persistence

Both `localStorage` and `sessionStorage` are Web Storage APIs used by browsers to store key-value data on the client side.

The primary difference is **how long the data survives**.

| Feature | localStorage | sessionStorage |
|----------|-------------|----------------|
| Persistence | Remains until explicitly deleted | Removed when the tab/window closes |
| Scope | Shared across all tabs of the same origin | Limited to a single browser tab |
| Survives Browser Restart | ✅ Yes | ❌ No |
| Accessible From Other Tabs | ✅ Yes | ❌ No |

---

## localStorage

Stores data permanently (until manually removed).

### Example Use Cases

- Theme preference (Dark/Light Mode)
- Language preference
- Remembering user settings
- Caching non-sensitive data

### Lifecycle

```text
User visits website
        ↓
Data stored in localStorage
        ↓
Browser closed
        ↓
Browser reopened
        ↓
Data still exists
```

---

## sessionStorage

Stores data only for the lifetime of the current browser tab.

### Example Use Cases

- Multi-step forms
- Temporary UI state
- Shopping cart data during a session
- Unsaved draft information

### Lifecycle

```text
User opens tab
        ↓
Data stored in sessionStorage
        ↓
Tab closed
        ↓
Data automatically removed
```

---

## Rule of Thumb

- Use **localStorage** when data should survive browser restarts.
- Use **sessionStorage** when data should exist only during the current browsing session.

---

# Compiled Language vs Interpreted Language

## Core Difference

The fundamental difference lies in **when and how source code is translated into machine code** that the CPU can execute.

| Compiled Language | Interpreted Language |
|------------------|----------------------|
| Entire program translated before execution | Translated during execution |
| Generates machine code/binary | Executes source code through an interpreter |
| Faster runtime performance | Generally slower |
| Compilation step required | No separate compilation step |
| Errors found mostly at compile time | Errors often found at runtime |

---

## Compiled Languages

A compiler converts the entire source code into machine code before execution.

### Flow

```text
Source Code
      ↓
Compiler
      ↓
Machine Code / Binary
      ↓
CPU Executes
```

### Examples

- C
- C++
- Rust
- Go

### Analogy

Imagine translating an entire English book into Marathi before giving it to a reader.

The translation happens once, and reading becomes very fast afterward.

---

## Interpreted Languages

An interpreter reads and executes code line by line during runtime.

### Flow

```text
Source Code
      ↓
Interpreter
      ↓
Line-by-Line Execution
```

### Examples

- Python
- JavaScript
- Ruby
- PHP

### Analogy

Imagine having a translator sitting beside you who translates each sentence as you read it.

No upfront translation is needed, but execution is slower.

---

## Modern Reality

The distinction is no longer completely black and white.

Examples:

- Python compiles to bytecode (`.pyc`) before interpretation.
- Java compiles to bytecode and then uses the JVM.
- JavaScript engines use JIT (Just-In-Time) compilation.

Therefore:

> Compiled vs Interpreted is better viewed as a spectrum rather than two completely separate categories.

---

# Multiplexing

## Definition

Multiplexing is a technique that combines multiple independent data streams into a single transmission channel.

Its purpose is to maximize utilization of a shared communication medium.

---

## Why Multiplexing Exists

Without multiplexing:

```text
User A → Cable 1
User B → Cable 2
User C → Cable 3
```

Multiple physical connections are required.

With multiplexing:

```text
User A
User B
User C
   ↓
Multiplexer
   ↓
Single Cable
   ↓
Demultiplexer
   ↓
Destination
```

Multiple signals share the same physical medium.

---

## Benefits

- Better bandwidth utilization
- Reduced infrastructure cost
- Fewer cables and devices
- Higher transmission efficiency

---

## Real-World Examples

### Network Connections

A single network cable can carry:

- Web traffic
- Video calls
- File downloads
- SSH sessions

simultaneously.

### HTTP/2 Multiplexing

HTTP/1.1:

```text
Request 1 → Wait
Request 2 → Wait
Request 3 → Wait
```

HTTP/2:

```text
Request 1
Request 2
Request 3
      ↓
Single TCP Connection
```

Multiple requests share one connection.

---

## Types of Multiplexing

### Time Division Multiplexing (TDM)

Each signal gets a time slot.

```text
A → B → C → A → B → C
```

### Frequency Division Multiplexing (FDM)

Each signal gets a frequency range.

Used in:

- Radio
- Television broadcasting

### Wavelength Division Multiplexing (WDM)

Used in fiber optics.

Multiple wavelengths (colors of light) share a single fiber cable.

---

# Daemon

## Definition

A **daemon** is a long-running background process that operates independently of user interaction.

Daemons typically provide system-level services such as:

- Web servers
- Database servers
- SSH services
- Schedulers

---

## What Does "Daemonize" Mean?

Daemonization is the process of transforming a normal interactive program into a daemon.

---

## Problem Without Daemonization

When a program is started from a terminal:

```text
Terminal
   ↓
Application
```

The application becomes tied to that terminal session.

If the terminal closes:

```text
Terminal Closed
      ↓
SIGHUP Signal
      ↓
Application Stops
```

---

## Goal of Daemonization

Detach the process from:

- Terminal session
- User login session
- Interactive shell

Result:

```text
Terminal
    ✖ Closed

Daemon
    ✔ Continues Running
```

---

## Common Linux Daemons

| Service | Daemon |
|----------|---------|
| SSH Server | `sshd` |
| Cron Scheduler | `crond` |
| Web Server | `nginx` |
| Database | `mysqld` |
| Docker Engine | `dockerd` |

---

## Characteristics of a Daemon

- Runs in background
- No user interface
- Starts during boot (often)
- Managed by `systemd`
- Provides continuous services

---

## Example

Starting Nginx:

```bash
sudo systemctl start nginx
```

Process:

```text
nginx
  ↓
Daemonized
  ↓
Listens on Port 80
  ↓
Serves Requests Continuously
```

---

# nohup (No Hang Up)

## Definition

`nohup` is a Linux/Unix utility that allows a process to continue running even after the terminal is closed or the user logs out.

---

## Problem

Normally:

```bash
python app.py
```

Process is attached to the terminal.

```text
Terminal Closed
      ↓
SIGHUP
      ↓
Process Killed
```

---

## Solution

Run the process using:

```bash
nohup python app.py &
```

Result:

```text
Terminal Closed
      ↓
No SIGHUP Effect
      ↓
Process Continues Running
```

---

## What nohup Does

### Ignores SIGHUP

Normally:

```text
SIGHUP
   ↓
Terminate Process
```

With `nohup`:

```text
SIGHUP
   ↓
Ignored
```

---

### Redirects Output

By default:

```text
stdout
stderr
    ↓
nohup.out
```

unless explicitly redirected.

Example:

```bash
nohup python app.py > app.log 2>&1 &
```

---

## Common Use Cases

### Long Running Scripts

```bash
nohup backup.sh &
```

### Data Processing Jobs

```bash
nohup python process_data.py &
```

### Temporary Server Execution

```bash
nohup flask run &
```

---

## Difference Between nohup and Daemon

| Feature | nohup | Daemon |
|----------|--------|---------|
| Ignores SIGHUP | ✅ | ✅ |
| Runs in Background | ✅ | ✅ |
| Fully Detached from Terminal | ❌ | ✅ |
| Managed by systemd | ❌ | ✅ |
| Intended for Services | ❌ | ✅ |
| Quick Long-Running Tasks | ✅ | ❌ |

---

## Rule of Thumb

- Use **nohup** for temporary long-running commands.
- Use a **daemon/service** for production workloads.
- Modern Linux systems typically use **systemd services** instead of manually running processes with `nohup`.
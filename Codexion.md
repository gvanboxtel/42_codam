*This project has been created as part of the 42 curriculum by gvan-box.*

# Codexion

## Description

Codexion is a concurrency project in C, a variation on the classic **dining philosophers** problem. A number of coders sit around a table in a co-working hub. Between each pair of neighbours lies one USB dongle, so there are as many dongles as coders. To compile, a coder needs **both** the dongle on their left and the one on their right. Each coder repeats a cycle:

```
take two dongles → compile → release dongles → debug → refactor → ...
```

A coder who has not started compiling within `time_to_burnout` milliseconds of their previous compile (or of the start of the simulation) **burns out**, and the simulation stops. It also stops when every coder has compiled the required number of times.

Each coder is a POSIX thread, and coders do not communicate with each other. On top of the classic problem, Codexion adds:

- **Dongle cooldown:** a released dongle cannot be taken again until `dongle_cooldown` ms have passed.
- **Fair arbitration:** when several coders want the same dongle, a scheduler decides who gets it: `fifo` (first request first) or `edf` (Earliest Deadline First, where the deadline is `last_compile_start + time_to_burnout`).
- A **hand-written priority queue** (binary heap) for the scheduling; no standard library queue is allowed.
- A **monitor thread** that must report a burnout within 10 ms of it happening.

## Instructions

### Compilation

```bash
make          # builds ./codexion with cc -Wall -Wextra -Werror -pthread
make clean    # remove object files
make fclean   # remove object files and the binary
make re       # rebuild
```

### Usage

```bash
./codexion number_of_coders time_to_burnout time_to_compile time_to_debug \
           time_to_refactor number_of_compiles_required dongle_cooldown scheduler
```

| Argument | Meaning |
|---|---|
| `number_of_coders` | Number of coders, and of dongles |
| `time_to_burnout` | ms a coder can go without starting a compile |
| `time_to_compile` | ms a compile takes (while holding two dongles) |
| `time_to_debug` | ms spent debugging |
| `time_to_refactor` | ms spent refactoring |
| `number_of_compiles_required` | The simulation ends when every coder has compiled this often |
| `dongle_cooldown` | ms a dongle stays unavailable after release |
| `scheduler` | `fifo` or `edf` |

All arguments are mandatory. Negative numbers, non-integers and unknown schedulers are rejected.

### Example

```bash
./codexion 4 800 200 200 200 5 10 edf
```

```
0 1 has taken a dongle
1 1 has taken a dongle
1 1 is compiling
201 1 is debugging
401 1 is refactoring
402 2 has taken a dongle
403 2 has taken a dongle
403 2 is compiling
...
```

Every state change is printed as `timestamp_in_ms coder_number message`.

## Blocking cases handled

**Deadlock.** A deadlock needs all four Coffman conditions: mutual exclusion, hold-and-wait, no preemption and circular wait. Dongles are exclusive by nature, and a coder cannot take a dongle from someone else, so the first and third conditions always hold. The solution breaks **circular wait** and **hold-and-wait**: a coder only takes their dongles when the scheduler grants them both, so no coder sits holding one dongle while waiting for the other in a cycle around the table.

**Starvation.** Every request goes into the dongle's priority queue, and the scheduler always serves the head of that queue. With `fifo`, a waiting coder can only be overtaken by requests that arrived earlier, so they are served after a bounded wait. With `edf`, the coder closest to burning out is served first, which keeps every coder alive whenever the parameters make that possible. Equal deadlines are broken by request order, so the result is deterministic.

**Cooldown.** Each dongle stores the time it was released. It only becomes available again when `release_time + dongle_cooldown` has passed; until then, waiting coders wait on the dongle's condition variable with a timeout instead of busy-looping.

**One coder.** With a single coder there is only one dongle on the table. The coder takes it but can never compile, and burns out after `time_to_burnout` ms, as expected.

**Precise burnout detection.** A separate monitor thread checks every coder's last compile start in short intervals, far below 10 ms, and prints the burnout message within the 10 ms limit. Sleeps in the coder threads are split into small steps so threads notice quickly that the simulation has ended.

**Log serialization.** All output goes through one function that holds a print mutex. It also checks that the simulation is still running, so no message is printed after a burnout has been reported.

## Thread synchronization mechanisms

| Primitive | Protects | Used by |
|---|---|---|
| `pthread_mutex_t` per dongle | The dongle's state (owner, release time) and its waiting queue | Coders taking and releasing dongles |
| `pthread_cond_t` per dongle | Waking waiting coders when a dongle is released or its cooldown ends | Coders waiting for a dongle |
| `pthread_mutex_t` for printing | stdout, so lines never interleave | All coders and the monitor |
| `pthread_mutex_t` for coder state | Each coder's last compile start and compile count | The coder (writes) and the monitor (reads) |
| `pthread_mutex_t` for the simulation flag | Whether the simulation is still running | The monitor (sets it) and everyone else (reads it) |

**Taking dongles.** A coder locks the dongle's mutex, adds a request to its priority queue and waits on the condition variable (`pthread_cond_timedwait`) until they are at the head of the queue, the dongle is free and its cooldown has passed. Releasing a dongle updates its state and calls `pthread_cond_broadcast`, so the waiting coders re-check their condition. Because the check and the wait happen under the same mutex, a release can never be missed between them.

**Coder ↔ monitor.** A coder updates `last_compile_start` under its own mutex at the moment it starts compiling. The monitor reads the same field under that mutex, so it never sees a half-written value. When the monitor detects a burnout, or sees that every coder has reached the required number of compiles, it sets the shared stop flag under its mutex. The coders check this flag between actions and leave their loop, after which the main thread joins all threads and frees all memory.

**Race condition example.** Without the per-coder mutex, the monitor could read `last_compile_start` at the same moment a coder writes it and decide that a coder who just started compiling has burned out. Locking both the read and the write makes them happen one after the other.

## Resources

- Dijkstra (1965), the dining philosophers problem, as described in [Wikipedia: Dining philosophers problem](https://en.wikipedia.org/wiki/Dining_philosophers_problem).
- Coffman, Elphick & Shoshani (1971), *System Deadlocks*, ACM Computing Surveys: the four conditions for deadlock.
- [POSIX Threads Programming (LLNL)](https://hpc-tutorials.llnl.gov/posix/)
- `man pthread_mutex_lock`, `man pthread_cond_wait`, `man pthread_cond_timedwait`, `man gettimeofday`.
- [Earliest deadline first scheduling](https://en.wikipedia.org/wiki/Earliest_deadline_first_scheduling)
- Cormen et al., *Introduction to Algorithms*, chapter 6: binary heaps and priority queues.

### AI usage

This README was written with Claude (Anthropic), based on the project subject.

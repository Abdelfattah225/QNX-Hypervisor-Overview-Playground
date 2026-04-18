# QNX Hypervisor - Running Guest Code

## Table of Contents

- [Where Do Guests Run?](#where-do-guests-run)
- [Guest Exits and Guest Entrances](#guest-exits-and-guest-entrances)
- [Causes of Guest Exits](#causes-of-guest-exits)
- [Minimizing Guest Exits](#minimizing-guest-exits)
- [CPU Virtualization Support](#cpu-virtualization-support)
- [CPU Privilege Levels](#cpu-privilege-levels)
- [Virtual CPUs (vCPUs)](#virtual-cpus-vcpus)
- [Guest Scheduling and Priorities](#guest-scheduling-and-priorities)
- [CPU Clusters](#cpu-clusters)
- [Performance Tuning Strategy](#performance-tuning-strategy)
- [Summary](#summary)
- [Interview Questions & Answers](#interview-questions--answers)

---

## Where Do Guests Run?

Guests execute **directly on the physical CPU** (bare metal):

- **Most instructions** → execute directly on the CPU, not emulated
- **Some instructions** → execute with added **virtualization support** (minimal overhead)
- **Few instructions** → **trapped and handled** (guest exit)

Traps occur when:
- Guest executes a **privileged instruction** it's not permitted to run (e.g., `SMC` on ARM)
- Guest accesses an **address configured for a vdev** (virtual device)

---

## Guest Exits and Guest Entrances

### The Trap Cycle

When a guest does something that needs to be intercepted, a **guest exit / guest entrance** cycle occurs:

```
┌──────────────────────────────────────────────────────────────┐
│                      TRAP CYCLE                               │
│                                                              │
│  1. Guest running directly on CPU                            │
│     ↓                                                        │
│  2. Guest does something that triggers a trap                │
│     ↓                                                        │
│  3. GUEST EXIT — virtualization hardware stops the guest     │
│     ↓                                                        │
│  4. qvm starts running instead                               │
│     ↓                                                        │
│  5. qvm saves guest state (registers, etc.)                  │
│     ↓                                                        │
│  6. qvm handles the trap (calls vdev code, etc.)             │
│     ↓                                                        │
│  7. qvm restores guest state                                 │
│     ↓                                                        │
│  8. GUEST ENTRANCE — jumps back into guest code              │
│     ↓                                                        │
│  9. Guest running directly on CPU again                      │
└──────────────────────────────────────────────────────────────┘
```

### Timing

- The **guest exit + guest entrance** (saving/restoring state) takes **single-digit microseconds**
- Most of the time is spent in step 6: **handling the trap** (vdev code execution)
- The handling time depends on how fast the vdev code is

### Terminology

> **Important:** In practice, you will hear **"guest exit"** not "trap." When someone says "there was a guest exit," they mean the **entire cycle** happened — guest exit, trap handling, and guest entrance. Get used to this terminology.

---

## Causes of Guest Exits

### Guest-Initiated Causes

| Cause | Description |
|---|---|
| **Accessing a vdev address** | Guest accesses an address programmed into the virtualization hardware for a virtual device |
| **Accessing invalid memory** | Guest tries to access memory outside its guest physical address space |
| **Inter-Processor Interrupts (IPIs)** | Guest kernel initiates IPIs to manage its virtual CPUs |
| **Privileged system registers** | Guest accesses certain privileged system registers |
| **Higher exception level requests** | Guest executes instructions like `SMC` that require a higher privilege level |

### Host-Initiated Causes

| Cause | Description |
|---|---|
| **Hardware interrupts** | A hardware interrupt occurs and must be routed through the host to the guest |
| **vCPU preemption** | A higher-priority host thread preempts a vCPU thread |

> **Note:** The guest kernel also generates guest exits — not just application code. Kernels manage CPUs, initiate IPIs, and perform other operations that can trigger traps.

---

## Minimizing Guest Exits

### Why Minimize?

Each guest exit takes time (even if the handling is quick). **Frequent guest exits** reduce the time the guest spends executing at bare-metal speed. Minimizing guest exits gives the guest **more CPU time**.

### Example: Emulated vs. Para-Virtualized Serial Port

**Emulated vdev (`vdev ser8250`) — Normal I/O, 1 byte at a time:**

```
Guest driver reads byte 1 → GUEST EXIT → emulate → GUEST ENTRANCE
Guest driver reads byte 2 → GUEST EXIT → emulate → GUEST ENTRANCE
Guest driver reads byte 3 → GUEST EXIT → emulate → GUEST ENTRANCE
Guest driver reads byte 4 → GUEST EXIT → emulate → GUEST ENTRANCE

Result: 4 guest exits for 4 bytes
```

**Para-virtualized vdev (`vdev virtio-console`) — VirtIO, multiple bytes at a time:**

```
Guest VirtIO driver reads all bytes from virt queue → GUEST EXIT → handle → GUEST ENTRANCE

Result: 1 guest exit for all bytes
```

> **Key Insight:** By simply changing the configuration (switching from an emulation vdev to a VirtIO vdev and using the corresponding driver), you can dramatically reduce guest exits for the **same work**.

### Using Kernel Event Logs for Tuning

`qvm` logs every guest exit and guest entrance to the **kernel event log** in the host:

```
1. Capture kernel event log using tracelogger
2. Load log into IDE System Profiler perspective
3. Analyze guest exit frequency
4. Reconfigure (e.g., switch vdev types)
5. Capture new log
6. Compare — fewer guest exits = better
```

---

## CPU Virtualization Support

Some instructions execute directly on the CPU with **added virtualization support** — no guest exit needed.

### Example: x86 Read Time Stamp Counter (RDTSC)

**Without virtualization support:**
```
Guest executes RDTSC → GUEST EXIT → qvm adds offset → GUEST ENTRANCE
                       (overhead!)
```

**With virtualization support:**
```
1. qvm programs the offset value into the virtualization hardware (once, at setup)
2. Guest executes RDTSC → CPU executes instruction + hardware adds offset automatically
   (NO guest exit! Minimal overhead!)
```

### Key Points

- CPU manufacturers (Intel, AMD, ARM) are **continually adding** more virtualization support instructions
- These eliminate guest exits for common operations
- `qvm` programs the virtualization hardware at setup time
- At runtime, the CPU handles everything with **negligible overhead**
- Examples include: timestamp counters, certain control register accesses, page table walks

---

## CPU Privilege Levels

### ARM Exception Levels vs. x86 Rings

| Level | ARM | x86 |
|---|---|---|
| **Least privileged** | EL0 | Ring 3 |
| | EL1 | Ring 2, Ring 1 |
| **Most privileged** | EL3 | Ring 0 |

> Note: ARM numbers go up for more privilege; x86 numbers go **down** for more privilege.

### Guest Privilege

- Guests run at a **lower privilege level** than the hypervisor microkernel (EL0 or EL1)
- When a guest executes an instruction requiring **higher privilege** (e.g., `SMC`):
  1. Virtualization hardware detects insufficient privilege
  2. **Guest exit** occurs
  3. Appropriate privilege level is set
  4. `qvm` runs and handles the instruction
  5. **Guest entrance** — guest resumes

### SMC Instruction Example (ARM)

```
Guest executes SMC (Secure Monitor Call — requests TrustZone service)
  ↓
Virtualization hardware: "Guest doesn't have privilege for this!"
  ↓
GUEST EXIT → set privilege level → run qvm
  ↓
qvm handles it (call vdev, handle itself, or hand off to host)
  ↓
GUEST ENTRANCE → guest resumes
```

---

## Virtual CPUs (vCPUs)

### What Are vCPUs?

Each virtual CPU is a **thread** belonging to the `qvm` process. These threads are called **vCPU threads**.

```
┌─────────────────────────────────────────────────────────┐
│                       HOST (Normal QNX)                   │
│                                                          │
│  ┌──────────────────────────────────┐                    │
│  │         qvm (QNX guest)          │                    │
│  │                                  │                    │
│  │  Threads:                        │                    │
│  │  - main thread                   │                    │
│  │  - resource manager thread       │                    │
│  │  - vdev threads (networking,etc) │                    │
│  │  - vCPU0 ←── runs guest code    │                    │
│  └──────────────────────────────────┘                    │
│                                                          │
│  ┌──────────────────────────────────┐                    │
│  │         qvm (Linux guest)        │                    │
│  │                                  │                    │
│  │  Threads:                        │                    │
│  │  - main thread                   │                    │
│  │  - resource manager thread       │                    │
│  │  - vdev threads                  │                    │
│  │  - vCPU0 ←── runs guest code    │                    │
│  │  - vCPU1 ←── runs guest code    │                    │
│  └──────────────────────────────────┘                    │
│                                                          │
│  procnto, drivers, io-sock, etc.                         │
└─────────────────────────────────────────────────────────┘
```

### Critical Understanding

> **The guest gets CPU time while its vCPU threads are scheduled.**

This is the single most important concept for understanding guest execution:

- vCPU threads **execute the guest code** (the kernel image, ramdisk, etc. loaded via the config file)
- When a vCPU thread is **RUNNING** → the guest is running
- When a vCPU thread is **READY** (preempted) → the guest is stopped
- When a vCPU thread handles a guest exit → it temporarily runs vdev code instead of guest code

### vCPU Thread Dual Role

```
vCPU thread execution:

  ┌─────────────────┐     Guest Exit      ┌──────────────────┐
  │  Running Guest   │ ──────────────────→ │  Running vdev    │
  │  Code            │                     │  Code            │
  │  (bare metal)    │ ←────────────────── │  (trap handling) │
  └─────────────────┘   Guest Entrance     └──────────────────┘

The same thread alternates between running guest code and vdev code.
```

### Configuration

```
# Each "cpu" line creates one vCPU thread
cpu         # creates vCPU0
cpu         # creates vCPU1

# The guest kernel will think it has 2 physical CPUs
```

### vCPU Count Rules

| Rule | Explanation |
|---|---|
| **Per guest: vCPUs ≤ physical CPUs** | Having more vCPUs than physical CPUs for a single guest **degrades performance** — vCPU threads must time-slice, causing more preemptions (more guest exits) |
| **Total across all guests: may exceed physical CPUs** | Multiple guests each needing multiple CPUs can result in total vCPUs > physical CPUs — this is normal |
| **More vCPUs ≠ more physical CPUs** | vCPUs are just threads. You can't get more parallelism than you have physical cores |

**Example (4 physical CPUs):**
```
QNX guest:     2 vCPUs  ← OK (2 ≤ 4)
Linux guest:   2 vCPUs  ← OK (2 ≤ 4)
Android guest: 2 vCPUs  ← OK (2 ≤ 4)
                         Total: 6 vCPUs > 4 physical CPUs (acceptable across guests)

BAD: One guest with 5 vCPUs on 4 physical CPUs
     → 5th vCPU thread can never run simultaneously with the others
     → More preemptions, more guest exits, worse performance
```

---

## Guest Scheduling and Priorities

### Two Independent Priority Domains

Guest priorities and host priorities are **completely unrelated**:

```
┌──────────────────────┐    ┌──────────────────────┐
│    QNX Guest          │    │    Linux Guest        │
│                       │    │                       │
│  Thread priorities:   │    │  Thread priorities:   │
│  0 (lowest) to 255   │    │  0 (highest) to ...   │
│  (QNX convention)     │    │  (Linux convention)   │
│                       │    │  (REVERSED!)          │
│  These have NOTHING   │    │  These have NOTHING   │
│  to do with host      │    │  to do with host      │
│  priorities           │    │  priorities            │
└──────────┬────────────┘    └──────────┬────────────┘
           │                            │
    ┌──────┴──────┐              ┌──────┴──────┐
    │ vCPU thread │              │ vCPU thread │
    │ priority: 21│              │ priority: 10│
    │ (HOST prio) │              │ (HOST prio) │
    └─────────────┘              └─────────────┘
           │                            │
           └──────────┬─────────────────┘
                      ▼
              HOST QNX Scheduling
         (priority-driven, preemptive)
```

### Priority Compression

> **Think of guest priorities as being "compressed" into the vCPU thread priority.**

**Example:**
- Linux guest has a **high-priority** thread (from Linux's perspective)
- Its vCPU thread runs at **priority 10** in the host
- QNX guest has a **low-priority** thread (from QNX's perspective)
- Its vCPU thread runs at **priority 21** in the host

**Result:** The QNX guest's low-priority thread gets CPU time **before** the Linux guest's high-priority thread, because the QNX guest's vCPU thread has higher host priority (21 > 10).

### Configuring vCPU Priority

```
# In .qvmconf file:
cpu
  sched 21         # This vCPU thread runs at host priority 21

cpu
  sched 10         # This vCPU thread runs at host priority 10
```

### What Determines When a Guest Runs?

| Factor | Impact |
|---|---|
| **vCPU thread priority** | Higher priority = more likely to run |
| **Other host thread priorities** | Higher-priority host threads preempt vCPU threads |
| **Scheduling algorithm** | Round-robin at same priority level |
| **How long high-priority threads run** | CPU hogs at high priority starve lower-priority vCPU threads |
| **Cluster assignment** | Determines which physical CPUs the vCPU can run on |

---

## CPU Clusters

### What Are Clusters?

A **cluster** is a named set of physical CPUs. Configured in the QNX host's startup code.

### Default Clusters in QNX 8

For a 4-core system:

```
Cluster "all"    → CPU 0, 1, 2, 3    (all cores)
Cluster "cpu0"   → CPU 0              (individual-core cluster)
Cluster "cpu1"   → CPU 1              (individual-core cluster)
Cluster "cpu2"   → CPU 2              (individual-core cluster)
Cluster "cpu3"   → CPU 3              (individual-core cluster)

Total: 5 default clusters
```

By default, vCPU threads use **qvm's thread 1's cluster**, which is typically the **all-cores cluster**.

### Custom Clusters (e.g., ARM big.LITTLE)

```
# In startup code:
Cluster "bigcpus"    → CPU 2, 3    (high-performance cores)
Cluster "littlecpus" → CPU 0, 1    (low-power cores)
```

### Configuring vCPU Cluster Assignment

```
# In .qvmconf file:
cpu
  cluster bigcpus     # This vCPU thread runs only on big cores
```

### Cluster Rules and Best Practices

| Rule | Reason |
|---|---|
| **Don't put multiple vCPUs of one guest on the same individual-core cluster** | One CPU can only run one thread at a time — vCPUs would have to take turns, defeating the purpose of multiple vCPUs |
| **Steer vCPUs away from clusters running CPU-hogging worker threads** | Worker threads at high priority will starve vCPU threads |
| **Use big-core clusters for performance-critical guests** | Gives the guest access to high-performance cores |
| **Use little-core clusters for low-priority guests** | Keeps them out of the way of critical workloads |

**Bad Configuration Example:**
```
# 4 physical CPUs, 2 vCPUs for Linux guest
cpu
  cluster cpu0        # vCPU0 can ONLY run on CPU 0
cpu
  cluster cpu0        # vCPU1 can ONLY run on CPU 0  ← BAD!

# Both vCPUs fight for one physical CPU — they take turns
# Guest thinks it has 2 CPUs but effectively has < 1
```

**Good Configuration Example:**
```
cpu
  cluster bigcpus     # vCPU0 can run on any big core
cpu
  cluster bigcpus     # vCPU1 can run on any big core

# Both vCPUs can run simultaneously on different big cores
# Kernel has scheduling flexibility
```

### Clusters Help With Spinlocks and Interrupts

| Problem | What Happens | How Clusters Help |
|---|---|---|
| **Spinlocks** | Guest kernel holds a spinlock (tight loop, ~tens of μs). If vCPU thread is preempted mid-spinlock, the spinlock is held for ms instead of μs | Assign vCPU to a cluster where it won't be preempted by CPU hogs |
| **High-frequency interrupts** | Guest needs to handle interrupts frequently. If vCPU thread doesn't run often enough, interrupts are delayed | Assign vCPU to a cluster with available CPU time |

---

## Performance Tuning Strategy

### Ordered Steps (When Guest Isn't Meeting Timing Deadlines)

```
┌──────────────────────────────────────────────────────────┐
│           PERFORMANCE TUNING PRIORITY ORDER               │
│                                                          │
│  1. CHECK CODE QUALITY (First!)                          │
│     • Is any thread hogging the CPU at high priority?    │
│     • Can high-priority code be split into:              │
│       - Brief high-priority work                         │
│       - Hand off to lower-priority thread for heavy work │
│                                                          │
│  2. ARRANGE PRIORITIES                                   │
│     • Adjust vCPU thread priorities                      │
│     • Adjust ALL host thread priorities                  │
│     • Ensure proper priority inversion prevention        │
│                                                          │
│  3. CONFIGURE CLUSTERS (Last Resort)                     │
│     • Assign vCPUs to specific clusters                  │
│     • Avoid clusters with CPU-hogging worker threads     │
│     • Use big/little core clusters strategically         │
│                                                          │
│  ALSO: Minimize Guest Exits                              │
│     • Use VirtIO vdevs instead of emulation where possible│
│     • Analyze kernel event logs with System Profiler     │
│     • Reconfigure and re-measure                         │
└──────────────────────────────────────────────────────────┘
```

---

## Summary

```
┌─────────────────────────────────────────────────────────────────┐
│                  RUNNING GUEST CODE SUMMARY                      │
│                                                                  │
│  • Guests run when their vCPU threads are scheduled              │
│  • vCPU threads = normal QNX threads in the qvm process          │
│  • Scheduling based on QNX priority-driven preemptive model      │
│                                                                  │
│  • Guest code executes directly on CPU (bare metal speed)        │
│  • Some instructions have CPU virtualization support (minimal)   │
│  • Some operations cause GUEST EXITS (trapped & handled)         │
│                                                                  │
│  • vCPU threads alternate between guest code and vdev code       │
│  • Guest priorities ≠ host priorities (completely independent)   │
│  • Guest priorities are "compressed" into vCPU thread priority   │
│                                                                  │
│  • Minimize guest exits for better performance                   │
│  • Use kernel event logs to analyze and tune                     │
│  • Per-guest vCPUs ≤ physical CPUs                               │
│  • Tune: code quality → priorities → clusters (in that order)    │
└─────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions & Answers

### Fundamentals

---

**Q1: Explain the guest exit / guest entrance cycle. What happens during each phase?**

<details>
<summary>Answer</summary>

The cycle occurs when a guest does something that needs to be intercepted:

1. **Guest running:** Guest code executes directly on the physical CPU at bare-metal speed. The hypervisor is not involved.

2. **Trigger:** The guest does something that the virtualization hardware is watching for — accessing a vdev address, executing a privileged instruction, etc.

3. **Guest Exit:** The virtualization hardware **stops the guest**. Control is transferred away from guest code.

4. **State Save:** `qvm` **saves the guest's state** (registers, program counter, etc.).

5. **Handling:** `qvm` performs the necessary work — typically calling into **vdev code** (a `.so` shared object). This is where most of the time is spent.

6. **State Restore:** `qvm` **restores the guest's state**.

7. **Guest Entrance:** `qvm` jumps back into the guest code, and the guest resumes executing directly on the CPU.

The guest exit + entrance (state save/restore) takes **single-digit microseconds**. The handling time depends on the complexity of the vdev code.

</details>

---

**Q2: What are the common causes of guest exits?**

<details>
<summary>Answer</summary>

**Guest-initiated causes:**
- **Accessing a vdev-configured address:** The virtualization hardware traps any access to addresses programmed for virtual devices
- **Accessing invalid memory:** Guest tries to access a guest physical address that isn't mapped
- **Inter-Processor Interrupts (IPIs):** The guest kernel sends IPIs to manage its virtual CPUs
- **Privileged system register access:** Guest accesses restricted registers
- **Privileged instructions:** e.g., `SMC` on ARM (Secure Monitor Call) requires a higher exception level

**Host-initiated causes:**
- **Hardware interrupts:** Must be routed through the host to the guest, requiring a guest exit
- **vCPU thread preemption:** A higher-priority host thread preempts the vCPU thread — the guest stops until the vCPU thread is rescheduled

</details>

---

**Q3: Why is minimizing guest exits important? Give a practical example.**

<details>
<summary>Answer</summary>

Each guest exit involves overhead: stopping the guest, saving state, handling the trap, restoring state, and re-entering the guest. While each exit may be quick, **frequent exits** reduce the time the guest spends executing at bare-metal speed.

**Practical example — Serial port reading:**

**Emulated vdev (4 bytes, 1 at a time):**
- 4 I/O operations → 4 guest exits → 4 trap handlings → 4 guest entrances
- Significant cumulative overhead

**VirtIO vdev (4 bytes via virt queue):**
- 1 VirtIO operation → 1 guest exit → 1 trap handling → 1 guest entrance
- Same data transferred, **75% fewer guest exits**

The fix was purely a **configuration change** — switch from `vdev ser8250` to `vdev virtio-console` and use the appropriate VirtIO driver in the guest.

**Tuning approach:** Use `tracelogger` to capture kernel event logs (qvm logs every guest exit/entrance), load them into the IDE System Profiler, analyze the frequency, reconfigure, and re-measure.

</details>

---

### vCPUs and Scheduling

---

**Q4: What is a vCPU thread? How does it relate to guest execution?**

<details>
<summary>Answer</summary>

A **vCPU thread** is a normal QNX thread that belongs to the `qvm` process. Each virtual CPU configured for a guest creates one vCPU thread.

**Critical relationship:** The guest gets CPU time **only when its vCPU threads are scheduled to run** by the QNX host scheduler. The vCPU thread literally **executes the guest code** — the kernel image and applications loaded into the guest's memory.

The vCPU thread has a **dual role:**
- **Normally:** Executes guest code directly on the physical CPU
- **On guest exit:** Switches to executing vdev code (trap handling), then returns to guest code on guest entrance

From the **guest's perspective**, each vCPU thread represents a physical CPU core. If you configure 2 vCPUs, the guest kernel thinks it has 2 physical cores.

</details>

---

**Q5: Why shouldn't you configure more vCPUs than physical CPUs for a single guest?**

<details>
<summary>Answer</summary>

vCPUs are just threads — they need physical CPUs to run on. If a guest has more vCPU threads than there are physical CPUs:

1. **Not all vCPU threads can run simultaneously** — some must wait
2. **Time-slicing occurs** — vCPU threads take turns, adding scheduling overhead
3. **More preemptions** — each preemption causes a guest exit (overhead)
4. **No performance gain** — you can't achieve more parallelism than you have physical cores

**Example:** 4 physical CPUs, 5 vCPU threads for one guest:
- Only 4 vCPU threads can run at any time
- The 5th must wait, getting preempted/scheduled repeatedly
- The guest thinks it has 5 cores but effectively has less than 4 due to overhead

**Rule:** For a single guest, vCPUs ≤ physical CPUs. However, the **total** across all guests may exceed physical CPUs — that's unavoidable with multiple guests.

</details>

---

**Q6: Explain the concept of "priority compression" in the QNX Hypervisor.**

<details>
<summary>Answer</summary>

Guest thread priorities and host thread priorities are **completely independent and unrelated**. A high-priority thread inside a guest doesn't automatically get high priority on the host.

**What matters is the vCPU thread's host priority.** All threads inside a guest — regardless of their guest-internal priority — only run when the vCPU thread is scheduled. The guest's internal priorities are "compressed" into the single vCPU thread priority on the host.

**Example:**
- Linux guest: Has a thread at **highest Linux priority**. Its vCPU thread runs at host priority **10**.
- QNX guest: Has a thread at **lowest QNX priority**. Its vCPU thread runs at host priority **21**.

Result: The QNX guest's low-priority thread runs **before** the Linux guest's high-priority thread, because host priority 21 > 10.

**Implication:** To give a guest more CPU time, you must adjust the **vCPU thread's host priority**, not the priorities inside the guest.

</details>

---

**Q7: How does QNX scheduling determine when a guest runs?**

<details>
<summary>Answer</summary>

Guest execution is governed entirely by QNX's **priority-driven, preemptive scheduling**:

1. vCPU threads are normal QNX threads with assigned priorities
2. They compete with **all other host threads** (drivers, procnto, other qvm processes, etc.)
3. The QNX scheduler selects threads based on **priority** (highest priority runs first) and **scheduling algorithm** (round-robin for same-priority threads)

**When does a guest run?**
- When its vCPU thread is in the **RUNNING** state
- This happens when no higher-priority thread needs the same physical CPU

**When does a guest stop?**
- vCPU thread is **preempted** by a higher-priority thread → guest exit, vCPU enters READY state
- vCPU thread encounters a **guest exit** → switches to running vdev code temporarily
- vCPU thread blocks (rare for vCPU threads)

**If a guest isn't getting enough time:**
- Check for high-priority CPU hogs in the host
- Adjust vCPU thread priority
- Adjust other host thread priorities
- Consider cluster assignments

</details>

---

### CPU Clusters

---

**Q8: What are CPU clusters in QNX 8 and how do they relate to the hypervisor?**

<details>
<summary>Answer</summary>

A **cluster** is a named set of physical CPUs, configured in the QNX host's startup code. Threads can be assigned to run only on CPUs within a specific cluster.

**Default clusters** (4-core system):
- `all` → CPUs 0, 1, 2, 3
- `cpu0` → CPU 0 (individual-core cluster)
- `cpu1` → CPU 1
- `cpu2` → CPU 2
- `cpu3` → CPU 3

**Custom clusters** (e.g., ARM big.LITTLE):
- `bigcpus` → CPUs 2, 3 (high-performance cores)
- `littlecpus` → CPUs 0, 1 (low-power cores)

**Hypervisor relevance:** You can assign vCPU threads to specific clusters, controlling which physical CPUs a guest can run on:

```
cpu
  cluster bigcpus    # This vCPU runs only on big cores
```

By default, vCPU threads use the **all-cores cluster**. Custom cluster assignment is a **last resort** tuning technique.

</details>

---

**Q9: Why should you never assign two vCPU threads of the same guest to the same individual-core cluster?**

<details>
<summary>Answer</summary>

An individual-core cluster contains **only one physical CPU**. If two vCPU threads are restricted to that single CPU:

- Only **one thread can run at a time** on that CPU
- The two vCPU threads must **take turns** (time-slice)
- The guest thinks it has 2 CPUs but effectively has **less than 1** (due to scheduling overhead)
- Each time-slice switch causes a **preemption** → guest exit → overhead
- **Completely defeats the purpose** of having multiple vCPUs

**Correct approach:** Assign both vCPU threads to a cluster with **multiple CPUs** (e.g., `bigcpus` with CPUs 2 and 3), so they can run **simultaneously** on different cores.

</details>

---

### CPU Virtualization Support

---

**Q10: What is "added virtualization support" and how does it differ from a guest exit?**

<details>
<summary>Answer</summary>

"Added virtualization support" means the CPU handles virtualization-related work **as part of executing the instruction** — without a guest exit. The hypervisor programs the virtualization hardware once (at setup), and subsequent executions are handled entirely in hardware.

**Example — x86 RDTSC (Read Time Stamp Counter):**

Without hardware support:
- Guest executes RDTSC → guest exit → qvm adds offset → guest entrance (slow)

With hardware support:
- qvm programs the offset value into the CPU's virtualization registers (once)
- Guest executes RDTSC → CPU adds offset automatically → no guest exit (fast)

**Key difference from guest exit:**
- Guest exit: Guest is **stopped**, qvm runs, overhead is significant
- Virtualization support: Instruction completes **inline** with minimal overhead, no context switch

CPU manufacturers continuously add more such support, reducing the need for guest exits.

</details>

---

**Q11: What are the CPU privilege levels on ARM and x86, and how do they affect guest execution?**

<details>
<summary>Answer</summary>

**ARM Exception Levels:**
- EL0: Least privileged (user applications)
- EL1: OS kernel
- EL2: Hypervisor
- EL3: Most privileged (Secure Monitor / TrustZone)

**x86 Rings:**
- Ring 3: Least privileged (user applications)
- Ring 0: Most privileged (OS kernel)
- (VMX root/non-root for hypervisor)

**Guest execution:** Guests run at a **lower privilege level** than the hypervisor (EL0/EL1 on ARM). When a guest executes an instruction requiring higher privilege (e.g., `SMC` on ARM for TrustZone):

1. Virtualization hardware detects insufficient privilege
2. Guest exit occurs
3. Privilege level is elevated appropriately
4. qvm handles the instruction
5. Guest entrance — guest resumes at its normal privilege level

This ensures the hypervisor maintains control and guests cannot bypass security boundaries.

</details>

---

### Scenario-Based Questions

---

**Q12: A Linux guest on the QNX Hypervisor is experiencing high latency in handling network interrupts. What could be the causes and how would you investigate?**

<details>
<summary>Answer</summary>

**Possible causes:**

1. **vCPU thread preemption:** Higher-priority host threads are preempting the vCPU thread, preventing the guest from handling interrupts promptly
2. **Excessive guest exits:** Too many guest exits from other sources (emulated vdevs, IPIs) consuming time
3. **Low vCPU thread priority:** The vCPU thread priority is too low relative to other host threads
4. **Cluster contention:** vCPU thread is on a cluster with CPU-hogging worker threads
5. **Interrupt routing overhead:** The interrupt must travel through the host before reaching the guest

**Investigation steps:**

1. **Capture kernel event log** using `tracelogger`
2. **Analyze in System Profiler** — look for:
   - Frequency of guest exits
   - Duration of guest exits
   - How often vCPU thread is in READY state (preempted)
   - Which higher-priority threads are preempting the vCPU
3. **Check priorities:** Are any host threads at higher priority than the vCPU thread that shouldn't be?
4. **Check for CPU hogs:** Any thread running for extended periods at high priority?

**Fixes (in order):**
1. Fix any code quality issues (CPU hogs at high priority)
2. Adjust priorities — raise vCPU thread priority or lower offending thread priorities
3. Switch emulation vdevs to VirtIO vdevs where possible (reduce guest exits)
4. Assign vCPU threads to a dedicated cluster away from worker threads

</details>

---

**Q13: You have a 4-core ARM board with big.LITTLE architecture (2 big cores, 2 little cores). You need to run a safety-critical QNX guest and a Linux infotainment guest. Design the vCPU and cluster configuration.**

<details>
<summary>Answer</summary>

**Cluster setup (in startup code):**
```
Cluster "bigcpus"    → CPU 2, 3    (Cortex-A76, high-performance)
Cluster "littlecpus" → CPU 0, 1    (Cortex-A55, low-power)
```

**QNX safety-critical guest:**
```
cpu
  sched 63              # High host priority
  cluster bigcpus       # Run on big cores for performance
cpu
  sched 63
  cluster bigcpus
```
- 2 vCPUs on big cores — safety-critical needs deterministic, high-performance execution
- High priority ensures it gets CPU time when needed

**Linux infotainment guest:**
```
cpu
  sched 20              # Lower host priority
  cluster littlecpus    # Run on little cores
cpu
  sched 20
  cluster littlecpus
```
- 2 vCPUs on little cores — infotainment is less time-critical
- Lower priority so it doesn't interfere with safety-critical guest

**Rationale:**
- Safety-critical guest gets the **big cores** and **higher priority** — it will never be starved by the infotainment guest
- Infotainment guest gets the **little cores** — isolated from the safety-critical workload
- No vCPU contention between guests since they're on separate clusters
- If infotainment needs more power occasionally, you could adjust to share cores, but isolation is preferred for safety

</details>

---

**Q14: Explain what happens at the thread level when a guest's vCPU thread is preempted by a higher-priority host thread.**

<details>
<summary>Answer</summary>

Step-by-step at the thread level:

1. **vCPU thread is RUNNING** — executing guest code directly on a physical CPU
2. **Higher-priority host thread becomes ready** (e.g., a driver thread wakes up)
3. **QNX scheduler performs preemption:**
   - The virtualization hardware triggers a **guest exit**
   - Guest state is saved
   - vCPU thread moves from **RUNNING → READY** state
4. **Higher-priority thread moves from READY → RUNNING** — it now executes on that physical CPU
5. **Guest is frozen** — no guest code executes while the vCPU thread is in READY state
6. **Higher-priority thread finishes or blocks** — moves to BLOCKED or exits
7. **QNX scheduler re-examines READY queue** — vCPU thread is the highest-priority READY thread
8. **vCPU thread moves from READY → RUNNING**
9. **Guest state is restored** — guest entrance occurs
10. **Guest resumes** — continues executing from where it was interrupted

**Impact:** The guest was frozen for the entire duration of the preemption. If the guest was in a spinlock or handling a time-sensitive interrupt, this delay can cause timing violations.

</details>

---

**Q15: A developer asks: "Can I improve my guest's performance by adding more vCPUs?" What is your response?**

<details>
<summary>Answer</summary>

**Not necessarily — and it could make things worse.**

Adding more vCPUs only helps if:
- The guest workload is **actually parallelizable**
- The number of vCPUs is **≤ physical CPUs** for that guest
- There are **available physical CPUs** not being saturated by other guests or host threads

**When more vCPUs hurt performance:**
- **More vCPUs than physical CPUs:** vCPU threads must time-slice, causing extra preemptions. Each preemption is a guest exit — overhead increases, net performance decreases.
- **Contention with other guests:** More vCPU threads competing for the same physical CPUs means more scheduling overhead.
- **Increased IPI overhead:** More virtual CPUs in the guest means the guest kernel sends more Inter-Processor Interrupts, each causing a guest exit.

**Better approaches first:**
1. Profile the guest workload — is it actually CPU-bound and parallelizable?
2. Check for guest exits — minimize them (switch to VirtIO, reduce vdev overhead)
3. Check host priorities — ensure vCPU threads aren't being starved
4. Check cluster assignments — ensure vCPUs are on appropriate cores

Only add vCPUs if profiling shows the guest genuinely needs more parallel execution capacity and there are physical CPUs available.

</details>

---

**Q16: How would you use the QNX IDE System Profiler to diagnose guest performance issues?**

<details>
<summary>Answer</summary>

**Step-by-step process:**

1. **Capture the kernel event log:**
   ```bash
   tracelogger -f /tmp/trace.kev
   ```
   Run the system under the problematic workload, then stop tracelogger.

2. **Load into IDE System Profiler:** Open the `.kev` file in the Momentics IDE's System Profiler perspective.

3. **Look for guest exits:**
   - `qvm` logs every **guest exit** and **guest entrance** as kernel events
   - Identify the **frequency** of guest exits — are there too many?
   - Identify the **duration** between exit and entrance — is handling slow?

4. **Look for preemptions:**
   - Find vCPU threads in the timeline view
   - See when they're in RUNNING vs. READY state
   - Identify which **higher-priority threads** are preempting them
   - Check how long the preemptions last

5. **Correlate with guest behavior:**
   - Map guest exits to specific vdev accesses
   - Identify if a particular emulated vdev is causing frequent exits
   - Consider switching to VirtIO

6. **Iterate:**
   - Reconfigure (change vdev types, adjust priorities, assign clusters)
   - Capture a new log
   - Compare — look for fewer guest exits, shorter preemptions, more vCPU RUNNING time

</details>

---

**Q17: What is the SMC instruction and how does the QNX Hypervisor handle it?**

<details>
<summary>Answer</summary>

**SMC (Secure Monitor Call)** is an ARM instruction that requests a service from the **Secure Monitor** (TrustZone) at exception level EL3. It's used by software to access secure services like cryptographic operations, secure boot, etc.

**How the hypervisor handles it:**

1. The guest executes the `SMC` instruction
2. The virtualization hardware detects that the guest **doesn't have sufficient privilege** (guests run at EL0/EL1, SMC requires EL3)
3. A **guest exit** occurs — the guest is stopped
4. The **appropriate privilege level is set** by the hardware
5. **`qvm` runs** and handles the SMC:
   - It may call a vdev to emulate the secure service
   - It may handle it directly
   - It may forward it to something else in the host
   - It may actually forward a real SMC to TrustZone if configured to do so
6. Once handling is complete, **guest entrance** occurs — the guest resumes

The guest doesn't know the SMC was intercepted — it receives the expected result as if it had called TrustZone directly (in full virtualization mode).

</details>

---

*These notes are based on the QNX Hypervisor Running Guest Code lesson. Related topics (Virtual Devices, Interrupts, Time, Safety) are covered in separate lessons.*
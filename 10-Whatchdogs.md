# QNX Hypervisor - Watchdogs

## Table of Contents

- [Overview](#overview)
- [How Watchdogs Work](#how-watchdogs-work)
- [Watchdog Timeout Actions](#watchdog-timeout-actions)
- [Detecting and Handling qvm Death](#detecting-and-handling-qvm-death)
- [Quick Reference Summary](#quick-reference-summary)
- [Interview Questions & Answers](#interview-questions--answers)

---

## Overview

The QNX Hypervisor provides **watchdog virtual devices** for guests to monitor process health.

| Watchdog vdev | Target Platform | Implementation |
|---|---|---|
| **vdev-wdt-sp805.so** | ARM (SP805 chip emulation) | 100% software emulation |
| **vdev-wdt-ib700.so** | x86 (IB700 chip emulation) | 100% software emulation |

> **No physical watchdog chip is needed.** These vdevs are complete software emulations of the respective watchdog timer chips.

---

## How Watchdogs Work

### The "I'm Still Alive" Pattern

```
┌──────────────────────────────────────────────────────┐
│                      GUEST                            │
│                                                      │
│  ┌──────────────────────────────┐                    │
│  │ Monitored Process            │                    │
│  │                              │                    │
│  │  while (running) {           │                    │
│  │    do_work();                │                    │
│  │    write(watchdog_register); │ ← "I'm alive!"    │
│  │  }                           │                    │
│  └──────────────┬───────────────┘                    │
└─────────────────┼────────────────────────────────────┘
                  │ I/O trapped and emulated
                  ▼
┌──────────────────────────────────────────────────────┐
│                   HOST (qvm)                          │
│                                                      │
│  ┌──────────────────────────────┐                    │
│  │ vdev-wdt-sp805.so            │                    │
│  │                              │                    │
│  │ Timer resets each time       │                    │
│  │ the register is written      │                    │
│  │                              │                    │
│  │ If timer expires →           │                    │
│  │   "Process hasn't checked    │                    │
│  │    in! Something is wrong!"  │                    │
│  └──────────────────────────────┘                    │
└──────────────────────────────────────────────────────┘
```

### Step-by-Step

1. A process in the guest **periodically writes** to a watchdog register ("I'm still alive")
2. This I/O is **trapped and emulated** by the watchdog vdev (it's an emulation-type vdev)
3. Each write **resets** the watchdog timer
4. If the process **fails to write** within the timeout period:
   - The process is assumed to be **stuck, crashed, or hung**
   - The watchdog vdev takes a **configured action**

---

## Watchdog Timeout Actions

When the watchdog timer expires (process didn't "kick" it in time), you can configure the response:

| Action | Description |
|---|---|
| **Generate a core dump** | Creates a dump file of `qvm`'s memory, registers, and stacks. Useful for debugging — send to QNX engineering for analysis. Does NOT kill `qvm`. |
| **Generate an interrupt** | Signals an interrupt to the guest. A handler in the guest can then take corrective action (restart the process, log the error, enter safe state, etc.) |
| **Kill `qvm`** | Terminates the `qvm` process immediately. Guest stops executing. Can be detected and handled by a host monitoring process. |

```
Watchdog timer expires (process didn't check in)
  │
  ├──→ Option 1: Core dump (debug, qvm continues)
  │
  ├──→ Option 2: Interrupt to guest (guest handles it)
  │
  └──→ Option 3: Kill qvm (guest stops, host can restart)
```

---

## Detecting and Handling qvm Death

If the watchdog (or anything else) kills `qvm`, you can detect and handle it:

| Method | Description |
|---|---|
| **POSIX child process detection** | If `qvm` is a child of your monitoring process, detect child death via `waitpid()` or `SIGCHLD` |
| **QNX proc manager events** | Subscribe to process death notifications for any process using QNX-specific APIs |
| **HAM (High Availability Manager)** | QNX utility for monitoring and automatically restarting processes |
| **SLM (System Launch and Monitor)** | QNX utility for launching and monitoring system processes |
| **Custom monitoring code** | Write your own monitoring process using any detection method above |

### Recovery After qvm Death

```
Monitor detects qvm death
  ↓
Option A: Restart qvm (reboot the guest)
  → qvm /data/guests/linux/linux.qvmconf
  → Guest boots fresh
  → Drivers must reset hardware on startup (DMA caveat)
  
Option B: Log and alert (stay in safe state)
  → Record the failure
  → Alert operator / safety system
  → Don't restart (per safety requirements)
```

---

## Quick Reference Summary

```
┌────────────────────────────────────────────────────────────────────┐
│                    WATCHDOGS SUMMARY                                │
│                                                                    │
│  AVAILABLE WATCHDOG VDEVS                                          │
│  • vdev-wdt-sp805.so (ARM, SP805 chip emulation)                  │
│  • vdev-wdt-ib700.so (x86, IB700 chip emulation)                  │
│  • Both are 100% software emulation — no physical chip needed      │
│                                                                    │
│  HOW IT WORKS                                                      │
│  • Guest process periodically writes to watchdog register          │
│  • Write is trapped and emulated by watchdog vdev                  │
│  • Each write resets the timer                                     │
│  • If timer expires → process is stuck/dead                        │
│                                                                    │
│  TIMEOUT ACTIONS                                                   │
│  • Core dump of qvm (for debugging, qvm continues)                │
│  • Generate interrupt to guest (guest handles it)                  │
│  • Kill qvm (guest stops immediately)                              │
│                                                                    │
│  HANDLING QVM DEATH                                                │
│  • POSIX waitpid() / SIGCHLD                                      │
│  • QNX proc manager events                                        │
│  • HAM (High Availability Manager)                                 │
│  • SLM (System Launch and Monitor)                                 │
│  • Custom monitoring code                                          │
│  • Can restart qvm to reboot the guest                             │
└────────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions & Answers

### Fundamentals

---

**Q1: What watchdog vdevs does the QNX Hypervisor provide and how are they implemented?**

<details>
<summary>Answer</summary>

The QNX Hypervisor provides two watchdog virtual devices:

1. **`vdev-wdt-sp805.so`** — emulates the ARM SP805 watchdog timer chip
2. **`vdev-wdt-ib700.so`** — emulates the x86 IB700 watchdog timer chip

Both are **100% software emulation** — no physical watchdog chip is required. They are emulation-type vdevs loaded as shared objects by `qvm`. When a guest process writes to the watchdog register, the I/O is trapped and emulated by the vdev, which resets its internal timer. If the timer expires without being reset, the configured action is taken.

</details>

---

**Q2: How does the watchdog mechanism work in a hypervisor guest?**

<details>
<summary>Answer</summary>

1. A **guest process** periodically writes to a watchdog register (e.g., a memory-mapped I/O address) to signal "I'm still alive"
2. Since this is an **emulation-type vdev**, the I/O access is **trapped** by the virtualization hardware
3. A **guest exit** occurs, and `qvm` calls the watchdog vdev's handler
4. The vdev **resets its internal timer** — the process is alive
5. **Guest entrance** resumes the guest
6. If the process **fails to write** within the configured timeout (because it's stuck, crashed, or hung), the timer expires
7. The vdev takes the **configured action**: core dump, generate interrupt, or kill `qvm`

The guest process and its driver are completely unaware that the watchdog is virtual — they perform normal I/O as if talking to a real watchdog chip.

</details>

---

**Q3: What are the three actions a watchdog vdev can take when its timer expires?**

<details>
<summary>Answer</summary>

1. **Generate a core dump:** Creates a dump file of `qvm`'s memory, registers, and stacks. `qvm` continues running — the guest is NOT killed. The dump file can be sent to QNX engineering for analysis. Most useful for debugging.

2. **Generate an interrupt:** Signals a virtual interrupt to the guest. A handler in the guest can then take corrective action — restart the failed process, log an error, enter a safe state, or escalate. The guest stays running.

3. **Kill `qvm`:** Terminates the `qvm` process immediately. The guest stops executing entirely. This is the most drastic action. A monitoring process in the host can detect the death and optionally restart `qvm`.

</details>

---

### Detection and Recovery

---

**Q4: How can you detect that a `qvm` process has been killed by a watchdog (or any other cause)?**

<details>
<summary>Answer</summary>

Multiple detection methods are available:

1. **POSIX child process detection:** If `qvm` was spawned as a child of a monitoring process, use `waitpid()` or handle `SIGCHLD` to detect when the child terminates.

2. **QNX proc manager events:** Use QNX-specific APIs to subscribe to process death notifications. You can be notified when any process (including `qvm`) dies, without needing a parent-child relationship.

3. **HAM (High Availability Manager):** A QNX utility specifically designed for monitoring processes and taking configured recovery actions (restart, notify, escalate) when they die.

4. **SLM (System Launch and Monitor):** A QNX utility for launching and monitoring system processes, including automatic restart capabilities.

5. **Custom monitoring code:** Write your own process that periodically checks if `qvm` is running, or uses any of the above mechanisms.

Once death is detected, the monitor can **restart `qvm`** with the same configuration file to reboot the guest, or enter a Design Safe State.

</details>

---

**Q5: In a production system, how would you combine watchdog vdevs with the DSS concepts from the previous lesson?**

<details>
<summary>Answer</summary>

**Architecture:**

```
┌──────────────────────────────────────────────────────────┐
│                        GUEST                              │
│                                                          │
│  ┌──────────────────┐    ┌───────────────────────────┐   │
│  │ Critical Process  │───→│ Periodically kicks        │   │
│  │ (monitored)       │    │ watchdog register         │   │
│  └──────────────────┘    └───────────────────────────┘   │
│                                                          │
│  Watchdog vdev (vdev-wdt-sp805)                          │
│  • Configured to KILL QVM on timeout                     │
└──────────────────────────────────────────────────────────┘
                          │
                          │ qvm killed
                          ▼
┌──────────────────────────────────────────────────────────┐
│                        HOST                               │
│                                                          │
│  ┌──────────────────────────┐                            │
│  │ HAM / Monitor Process     │                            │
│  │ • Detects qvm death       │                            │
│  │ • Attempts: LOCAL DSS     │                            │
│  │   → Try restart qvm (N times)                         │
│  │   → If repeated failures → GLOBAL DSS (reboot)        │
│  │ • Logs all events         │                            │
│  └──────────────────────────┘                            │
└──────────────────────────────────────────────────────────┘
```

**Flow:**
1. Critical guest process periodically kicks the watchdog
2. If the process hangs → watchdog expires → kills `qvm` (**Local DSS**)
3. Host monitor detects `qvm` death → attempts restart
4. If restart succeeds → guest recovers (drivers reset hardware)
5. If restart fails repeatedly → escalate to **Global DSS** (reboot system)
6. All events are logged for post-incident analysis

This combines watchdog monitoring (guest-level health check) with DSS (system-level safety response) and HAM (process-level recovery).

</details>

---

*These notes are based on the QNX Hypervisor Watchdogs lesson. Related topics (Design Safe State, Safety, Virtual Devices) are covered in separate lessons.*
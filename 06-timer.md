# QNX Hypervisor - Time Management

## Table of Contents

- [Overview](#overview)
- [Virtualized Free-Running Hardware Counter](#virtualized-free-running-hardware-counter)
- [Counter Offset Configuration](#counter-offset-configuration)
- [Available Timers](#available-timers)
- [Managing Time of Day](#managing-time-of-day)
- [Stolen Time](#stolen-time)
- [Quick Reference Summary](#quick-reference-summary)
- [Interview Questions & Answers](#interview-questions--answers)

---

## Overview

The QNX Hypervisor provides guests with timing capabilities:

| Component | Description |
|---|---|
| **Virtualized free-running counter** | Hardware counter with configurable offset |
| **Emulated timer vdevs** | Various timer chip emulations (x86) |
| **Generic Timer** | ARM's built-in timer (ARM) |
| **Time-of-day management** | Guest's own responsibility |
| **Stolen time tracking** | Tracks time vCPU didn't run guest code |

---

## Virtualized Free-Running Hardware Counter

A free-running counter that continuously increments from boot time.

### Platform Details

| Platform | Counter Name | Access Method |
|---|---|---|
| **x86** | TSC (Time Stamp Counter) | `RDTSC` instruction (Read Time Stamp Counter) |
| **ARM** | Virtual Counter | Read `CNTVCT_EL0` register (Exception Level 0) |

### The Boot Time Problem

```
Timeline:
═══════════════════════════════════════════════════════
│                                                     │
▼                                                     ▼
Host boots                    Guest boots
Counter = 0                   Counter = ???

  ← Host running for some time → ← Guest starts here
```

When the host boots, the counter starts at **0**. But the guest boots **later** — so what value should the guest see?

**Two options:**
1. Counter appears as **0 when the guest boots** (default)
2. Counter **matches the host's counter** at all times

This is controlled by an **offset** that `qvm` applies to the counter.

---

## Counter Offset Configuration

### How the Offset Works

```
Guest counter value = Hardware counter value - Offset
```

### Default Behavior (Guest sees 0 at boot)

When the guest boots, `qvm` **grabs the current hardware counter value** and saves it as the offset.

```
At guest boot time:
  Hardware counter = 5,000,000 (host has been running for a while)
  Offset = 5,000,000 (qvm saves current value)
  
  Guest reads counter: 5,000,000 - 5,000,000 = 0  ✓

Later:
  Hardware counter = 8,000,000
  Guest reads counter: 8,000,000 - 5,000,000 = 3,000,000
  (Guest thinks 3,000,000 ticks have passed since it booted)
```

### Match Host Counter (Offset = 0)

Configure in `.qvmconf`:
```
set counter-offset 0
```

```
At guest boot time:
  Hardware counter = 5,000,000
  Offset = 0
  
  Guest reads counter: 5,000,000 - 0 = 5,000,000
  (Same value the host would see!)

Later:
  Hardware counter = 8,000,000
  Guest reads counter: 8,000,000 - 0 = 8,000,000
  (Always matches host)
```

### Configuration Options Summary

| Configuration | Offset Value | Guest Counter at Boot | Guest Counter Later |
|---|---|---|---|
| **Default** (no config) | Current counter value at boot | **0** | Time since guest boot |
| `set counter-offset 0` | 0 | Same as host | **Always matches host** |
| `set counter-offset ~0` | Same as default | **0** | Time since guest boot |
| `set counter-offset <value>` | Custom value | Custom | Custom (rarely useful) |

> **Practical choices:** Either leave as default (guest counter starts at 0) or set offset to 0 (guest counter matches host). Other values are valid but rarely useful.

### Implementation (No Guest Exit!)

As covered in the Running Guest Code lesson, this offset is handled by **CPU virtualization support**:
- `qvm` programs the offset value into the virtualization hardware **once** at setup
- When the guest reads the counter, the CPU **automatically applies the offset**
- **No guest exit occurs** — minimal overhead

---

## Available Timers

### ARM Timers

| Timer | Description | Notes |
|---|---|---|
| **Generic Timer** | ARM's built-in timer system | Available directly to guests. Includes the virtualized free-running counter. Multiple timer types available (physical, virtual, hypervisor). |

### x86 Timers

| Timer | vdev Name | Notes |
|---|---|---|
| **MC146818** (RTC) | `vdev mc146818` | Real-time clock chip emulation. Load `.so` via config. |
| **8254** (PIT) | `vdev 8254` | Programmable Interval Timer emulation. Load `.so` via config. |
| **HPET** | `vdev hpet` | High Precision Event Timer emulation. Load `.so` via config. |
| **LAPIC Timer** | Built into `qvm` | Local APIC timer. **No separate `.so` needed** — automatically available. |

### Configuration Example (x86)

```
# In .qvmconf — load timer vdevs as needed:
vdev mc146818        # RTC timer
vdev 8254            # PIT timer
vdev hpet            # High Precision Event Timer
# LAPIC timer is built into qvm — no config needed
```

> You can load **all** of them if your guest needs multiple timer sources.

---

## Managing Time of Day

### Guest Responsibility

**Time of day (system clock / date-time) is the guest's responsibility.** Guest OS kernels (Linux, QNX) track the current date and time — this is true whether or not a hypervisor is involved.

### Synchronizing Time with the Host or External Sources

If you need the guest's time of day to be accurate or synchronized:

| Method | How It Works |
|---|---|
| **Networking** | Write a host process that uses **NTP** (Network Time Protocol) or **PTP** (Precision Time Protocol) to get accurate time. Guest communicates with this host process via sockets (using VirtIO networking). |
| **Shared Memory** | Host process gets the time and writes it to shared memory. Guest reads from shared memory. |
| **GPS** | Host process reads time from a GPS receiver. Communicates to guest via networking or shared memory. |
| **Pass-through** | If a device has a time register, use the `pass` option to map it into the guest's physical address space. Guest process reads the register directly. Read-only access doesn't need to be exclusive. |
| **Other sources** | Atomic clocks, radio time signals, etc. — anything the host can access. |

```
┌──────────────────┐        ┌──────────────────┐
│      GUEST        │        │       HOST        │
│                   │        │                   │
│  ┌─────────────┐  │        │  ┌─────────────┐  │
│  │ Time sync   │  │ Socket │  │ Time source  │  │
│  │ client      │◄─┼────────┼──│ process      │  │
│  └─────────────┘  │  or    │  │ (NTP/PTP/GPS)│  │
│                   │ Shmem  │  └──────┬──────┘  │
└──────────────────┘        │         │          │
                             │  ┌──────┴──────┐  │
                             │  │ Time Source  │  │
                             │  │ (NTP server, │  │
                             │  │  GPS, atomic │  │
                             │  │  clock, etc.)│  │
                             │  └─────────────┘  │
                             └──────────────────┘
```

---

## Stolen Time

### Definition

**Stolen time** is the time that a vCPU thread **did not run guest code** because:
1. The vCPU thread was **preempted** by a higher-priority host thread
2. A **guest exit** occurred and the vCPU thread was executing emulation/vdev code

> **Origin:** Stolen time is an ARM concept, but QNX makes it available for x86 as well.

### Important: Does NOT Affect Guest Timers

Stolen time is tracked **independently** from guest timers. The virtualized counters continue counting regardless of stolen time. Stolen time is purely an accounting metric.

### Stolen Time Scenarios

#### Scenario 1: Preemption

```
┌─────────────────────────────────────────────────────────┐
│  vCPU Thread (Priority 10)                               │
│                                                         │
│  ██████████░░░░░░░░░░░░░░░██████████████████            │
│  Running    READY (stolen) Running                       │
│  guest      ←───────────→  guest                         │
│  code       Driver thread  code                          │
│             (Priority 20)                                │
│             preempts vCPU                                │
│                                                         │
│  Stolen time = duration vCPU was in READY state          │
└─────────────────────────────────────────────────────────┘
```

#### Scenario 2: Guest Exit (Emulation)

```
┌─────────────────────────────────────────────────────────┐
│  vCPU Thread                                             │
│                                                         │
│  ██████████▓▓▓▓▓▓▓▓▓▓▓▓▓▓██████████████████            │
│  Running    Guest exit     Running                       │
│  guest      (emulation/    guest                         │
│  code       vdev code)     code                          │
│             ←───────────→                                │
│             Stolen time                                  │
└─────────────────────────────────────────────────────────┘
```

### Exception: Halt / WFI Instructions

If the guest exit was caused by a **halt instruction** (x86) or **WFI instruction** (ARM — Wait For Interrupt), it is **NOT counted as stolen time**.

**Why?** These instructions indicate the guest's CPU is **idle** — the system has nothing to do. Idle time is not "stolen" — it's just unused.

```
Kernel idle loop:
  x86: HLT instruction  → guest exit → NOT stolen time
  ARM: WFI instruction  → guest exit → NOT stolen time
```

### Why Emulation Time Is Included

Strictly speaking, ARM's definition of stolen time only includes **preemption time** (not emulation time). However, QNX includes emulation time as well because:

```
What if during emulation, the vCPU thread gets preempted?

  vCPU thread running vdev code (guest exit)
    → Higher priority thread preempts vCPU
    → This IS preemption and SHOULD be stolen time
    → But tracking "preemption during emulation" is complex

Solution: Count ALL emulation time as stolen time
  → Simpler implementation
  → Slightly over-counts, but more practical than the alternative
```

### Stolen Time Per vCPU

Stolen time is tracked **per individual vCPU thread**:

```
┌──────────────┐   ┌──────────────┐
│  vCPU0        │   │  vCPU1        │
│               │   │               │
│  stolen: 15ms │   │  stolen: 42ms │
│  (preempted   │   │  (more guest  │
│   less)       │   │   exits)      │
└──────────────┘   └──────────────┘
```

---

## Quick Reference Summary

```
┌────────────────────────────────────────────────────────────────────┐
│                TIME MANAGEMENT SUMMARY                              │
│                                                                    │
│  VIRTUALIZED COUNTER                                               │
│  • x86: TSC (RDTSC instruction)                                    │
│  • ARM: Virtual Counter (CNTVCT_EL0 register)                      │
│  • Offset applied by CPU hardware (no guest exit!)                 │
│  • Default: guest counter = 0 at guest boot                       │
│  • set counter-offset 0: guest counter matches host                │
│                                                                    │
│  TIMERS                                                            │
│  • ARM: Generic Timer (built-in)                                   │
│  • x86: MC146818, 8254, HPET (vdev .so files)                     │
│  • x86: LAPIC timer (built into qvm, automatic)                   │
│                                                                    │
│  TIME OF DAY                                                       │
│  • Guest's own responsibility                                      │
│  • Sync via: NTP/PTP, GPS, shared memory, pass-through             │
│                                                                    │
│  STOLEN TIME                                                       │
│  • Time vCPU didn't run guest code                                 │
│  • Includes: preemption + emulation (guest exits)                  │
│  • Excludes: HLT/WFI (guest idle)                                  │
│  • Does NOT affect guest timers                                    │
│  • Tracked per vCPU thread                                         │
│  • ARM concept, also available on x86                              │
└────────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions & Answers

### Fundamentals

---

**Q1: What timing capabilities does the QNX Hypervisor provide to guests?**

<details>
<summary>Answer</summary>

The hypervisor provides:

1. **Virtualized free-running hardware counter:**
   - x86: TSC (Time Stamp Counter), accessed via `RDTSC` instruction
   - ARM: Virtual Counter, accessed via `CNTVCT_EL0` register
   - Configurable offset to adjust what value the guest sees at boot time

2. **Emulated timer vdevs (x86):**
   - MC146818 (RTC) — `vdev mc146818`
   - 8254 (PIT) — `vdev 8254`
   - HPET — `vdev hpet`
   - LAPIC Timer — built into `qvm`, automatic

3. **Generic Timer (ARM):** Built-in ARM timer available directly to guests

4. **Stolen time tracking:** Per-vCPU accounting of time not spent running guest code

**Not provided:** Time-of-day management — this is the guest OS kernel's responsibility.

</details>

---

**Q2: Explain the counter offset mechanism. Why is it needed?**

<details>
<summary>Answer</summary>

**The problem:** The hardware counter starts at 0 when the **host** boots. But a guest boots **later** — so the counter is already at some large value when the guest starts. If the guest expects its counter to start at 0 (as any freshly booted OS would), the raw counter value is wrong.

**The solution:** `qvm` applies an **offset** to the counter value that the guest sees:

```
Guest counter value = Hardware counter value - Offset
```

**Default behavior:** When the guest boots, `qvm` captures the current hardware counter value and uses it as the offset. This makes the guest counter appear to be 0 at boot time.

**Alternative (`set counter-offset 0`):** Setting the offset to 0 means the guest always sees the same counter value as the host — useful for time synchronization across guests and host.

**Implementation:** The offset is programmed into the **CPU virtualization hardware** once at setup. When the guest reads the counter, the CPU applies the offset automatically — **no guest exit occurs**, so overhead is negligible.

</details>

---

**Q3: What are the two practical options for counter offset configuration, and when would you use each?**

<details>
<summary>Answer</summary>

| Option | Configuration | Guest Counter at Boot | Use Case |
|---|---|---|---|
| **Default** | No configuration needed | **0** (appears freshly booted) | Standard use — guest OS expects counter to start at 0. Most common for Linux/QNX guests. |
| **Offset = 0** | `set counter-offset 0` | Same as host counter | When you need the guest's counter to **match the host's counter** at all times. Useful for time correlation between guests and host, debugging, or coordinated timing. |

**Rarely used:** Any other offset value (e.g., `set counter-offset ~0` reverts to default). Custom values are technically valid but not practically useful.

</details>

---

### Timers

---

**Q4: Compare the timer options available on ARM vs. x86 for hypervisor guests.**

<details>
<summary>Answer</summary>

**ARM:**
- **Generic Timer** — ARM's built-in timer system, available directly to guests
  - Includes the virtualized free-running counter (`CNTVCT_EL0`)
  - Multiple timer types (physical, virtual, hypervisor timers)
  - No vdev loading needed — it's part of the ARM architecture

**x86:**
- **TSC** — virtualized free-running counter (via `RDTSC`)
- **MC146818** — RTC chip emulation (`vdev mc146818` — load `.so`)
- **8254** — PIT emulation (`vdev 8254` — load `.so`)
- **HPET** — High Precision Event Timer (`vdev hpet` — load `.so`)
- **LAPIC Timer** — built into `qvm`, automatically available (no `.so` needed)

**Key difference:** ARM has a unified, architecture-level timer (Generic Timer). x86 has multiple legacy timer chips that must be individually emulated via vdevs, plus the LAPIC timer which is built into `qvm`.

</details>

---

**Q5: Why is the LAPIC timer different from other x86 timer vdevs?**

<details>
<summary>Answer</summary>

The LAPIC (Local Advanced Programmable Interrupt Controller) timer is **built into `qvm`** — it's statically linked, not a separate `.so` file. You don't need to configure it in the `.qvmconf`; it's **automatically available** to guests.

This is similar to how interrupt controller vdevs and some timer vdevs are built into `qvm` (as discussed in the Virtual Devices lesson). The LAPIC is so fundamental to x86 operation (every CPU core has one) that it makes sense to include it by default.

Other x86 timer chips (MC146818, 8254, HPET) are optional and loaded as separate `.so` shared objects via `vdev` lines in the configuration file, since not every guest may need them.

</details>

---

### Time of Day

---

**Q6: How does a guest manage its time of day? How can it synchronize with external time sources?**

<details>
<summary>Answer</summary>

**Time of day is entirely the guest's responsibility.** The guest OS kernel (Linux or QNX) maintains the system clock, just as it would on bare metal. The hypervisor provides timers/counters but does not manage what the actual date/time is.

**Synchronization methods:**

1. **Networking (NTP/PTP):** A host process runs an NTP/PTP client to get accurate time. The guest communicates with this process via TCP/IP sockets (using VirtIO networking) to synchronize its clock.

2. **Shared memory:** A host process writes the current time to shared memory. Guest reads it periodically.

3. **GPS:** Host process reads time from a GPS receiver, communicates to guest via networking or shared memory.

4. **Pass-through:** If a hardware device has a time register, use the `pass` option to map it into the guest's physical address space. The guest reads the register directly. Since it's read-only, exclusive access isn't strictly necessary.

5. **Any accessible time source:** Atomic clocks, radio time signals, etc. — as long as the host can access the source, it can relay the time to guests.

</details>

---

### Stolen Time

---

**Q7: What is stolen time and why is it tracked?**

<details>
<summary>Answer</summary>

**Stolen time** is the duration during which a vCPU thread was **not executing guest code**. It includes:

1. **Preemption time:** The vCPU thread was preempted by a higher-priority host thread and sat in the READY state
2. **Emulation time:** A guest exit occurred and the vCPU thread was executing vdev/emulation code instead of guest code

**Why track it?**
- **Performance analysis:** High stolen time indicates the guest isn't getting enough CPU time
- **Guest OS awareness:** Some guest OSes (especially Linux) can read stolen time and adjust their scheduling/accounting accordingly — they know not to charge application processes for time that was "stolen" by the hypervisor
- **Debugging:** Helps identify whether guest performance issues are caused by the guest itself or by host contention

**Important:** Stolen time does **not** affect guest timers. The virtualized counters continue incrementing regardless — stolen time is purely an accounting/diagnostic metric tracked per vCPU thread.

</details>

---

**Q8: Why are HLT and WFI instructions excluded from stolen time?**

<details>
<summary>Answer</summary>

**HLT** (x86) and **WFI** (ARM — Wait For Interrupt) are instructions that guest kernels execute when the CPU is **idle** — there's no work to do, so the kernel tells the CPU to wait for an interrupt.

These cause a guest exit, but they are **not counted as stolen time** because:

1. **The guest chose to be idle** — it's not that something external "stole" the time; the guest had nothing to do
2. **It's expected behavior** — every OS does this when idle
3. **It would distort the metric** — counting idle time as "stolen" would make stolen time misleading. You'd see high stolen time even when the guest simply has no work to do.

Stolen time should reflect time the guest **wanted to run but couldn't** (due to preemption or emulation overhead), not time the guest **voluntarily gave up**.

</details>

---

**Q9: Why does QNX count emulation time as stolen time, even though ARM's definition only includes preemption?**

<details>
<summary>Answer</summary>

ARM's original definition of stolen time only counts **preemption** (vCPU thread in READY state because a higher-priority thread is running). QNX also counts **emulation time** (guest exit for trap handling) for a practical reason:

**The complexity problem:**
During a guest exit (emulation), the vCPU thread is running vdev code. If a higher-priority thread **preempts the vCPU during emulation**, that preemption IS stolen time by ARM's definition. But tracking whether a preemption happened specifically "during emulation" versus "during guest code execution" would require complex bookkeeping in the microkernel.

**QNX's approach:** Count **all** emulation time as stolen time.
- Simpler implementation
- Slightly **over-counts** (emulation time that wasn't preempted is included)
- But avoids the complexity of distinguishing preemption-during-emulation from regular emulation
- In practice, emulation time is relatively short, so the over-counting is minimal

This is a pragmatic engineering tradeoff — simplicity over perfect accuracy.

</details>

---

### Scenario-Based Questions

---

**Q10: A guest OS reports inaccurate time after running for several hours, drifting from the actual wall-clock time. What could be causing this and how would you fix it?**

<details>
<summary>Answer</summary>

**Possible causes:**

1. **No time synchronization configured:** The guest is relying solely on its internal timers, which can drift over time due to:
   - Clock frequency inaccuracies
   - Accumulated rounding errors
   - Guest exits affecting timer interrupt delivery

2. **Counter offset misconfiguration:** If the guest counter doesn't align with reality, time calculations may be off from the start.

3. **Excessive stolen time:** If the guest experiences heavy preemption, timer interrupts may be delivered late, causing the guest's time tracking to fall behind.

4. **Timer interrupt frequency issues:** The guest may not be handling its tick timer frequently enough due to vCPU scheduling issues.

**Fixes:**

1. **Set up NTP/PTP synchronization:** Configure a host process with NTP/PTP access, and have the guest periodically synchronize its clock via networking or shared memory. This is the most robust solution.

2. **Use `set counter-offset 0`:** If the guest needs to correlate its counter with the host, set the offset to 0 so counters match.

3. **Reduce stolen time:** Ensure vCPU threads have sufficient priority and aren't being excessively preempted (check priorities, clusters, CPU hogs).

4. **Pass-through an RTC:** If the board has a battery-backed real-time clock, use `pass` to map it into the guest's address space so it can read accurate wall-clock time directly.

</details>

---

**Q11: You're running two guests on the QNX Hypervisor. Guest A needs its counter to start at 0 (default). Guest B needs its counter to match the host. How do you configure this?**

<details>
<summary>Answer</summary>

Each guest has its own `.qvmconf` configuration file, and the counter offset is configured **per guest**:

**Guest A (default — counter starts at 0):**
```
# guest_a.qvmconf
system guest_a
ram 256M
cpu
cpu
# No counter-offset configuration needed — default behavior
# Counter will appear as 0 when Guest A boots
```

**Guest B (counter matches host):**
```
# guest_b.qvmconf
system guest_b
ram 512M
cpu
cpu
set counter-offset 0    # Offset = 0, guest counter always matches host
```

**Result:**
- Guest A boots → its counter reads 0, then counts up from there
- Guest B boots → its counter reads the same value as the host's counter
- Both guests can coexist with different offset configurations

</details>

---

**Q12: A developer notices high stolen time for a vCPU thread. What does this indicate and how should they investigate?**

<details>
<summary>Answer</summary>

**What high stolen time indicates:**
The vCPU thread is spending significant time **not executing guest code** — either being preempted or handling guest exits.

**Investigation steps:**

1. **Capture kernel event log** with `tracelogger`
2. **Analyze in System Profiler:**
   - How much stolen time is from **preemption** vs. **emulation**?
   - Which higher-priority threads are preempting the vCPU?
   - How frequently are guest exits occurring?

3. **If preemption-dominated:**
   - Identify the preempting threads — are they CPU hogs?
   - Raise vCPU thread priority or lower offending thread priorities
   - Check cluster assignments — move vCPU to a less contended cluster
   - Review if high-priority threads can be split (brief high-priority work + handoff to lower-priority thread)

4. **If emulation-dominated:**
   - Identify which vdevs are causing frequent guest exits
   - Consider switching from emulated vdevs to **VirtIO vdevs** (fewer guest exits per operation)
   - Consider using **pass-through** for frequently accessed devices

5. **Check for HLT/WFI:**
   - If the guest is mostly idle, some guest exits are expected and not counted as stolen time
   - High stolen time on an idle guest suggests preemption issues, not workload issues

**Remember:** Stolen time doesn't affect guest timers — the counters keep incrementing. But high stolen time means the guest is getting less effective CPU time, which impacts real-time performance, interrupt handling, and spinlock behavior.

</details>

---

**Q13: Compare the virtualized free-running counter on ARM and x86. How is the offset applied on each platform?**

<details>
<summary>Answer</summary>

| Aspect | x86 | ARM |
|---|---|---|
| **Counter name** | TSC (Time Stamp Counter) | Virtual Counter |
| **Access method** | `RDTSC` instruction | Read `CNTVCT_EL0` register |
| **Offset mechanism** | Extended/virtualization hardware applies offset | Stage 2 timer offset register (`CNTVOFF_EL2`) |
| **Who programs offset** | `qvm` programs it once at guest setup | `qvm` programs it once at guest setup |
| **Guest exit required?** | **No** — CPU applies offset in hardware | **No** — CPU applies offset in hardware |
| **Overhead** | Negligible | Negligible |
| **Default offset** | Current counter value at guest boot → guest sees 0 | Current counter value at guest boot → guest sees 0 |
| **Counter-offset 0** | Guest counter matches host | Guest counter matches host |

Both platforms work the same way conceptually:
1. `qvm` programs the offset value into the virtualization hardware once
2. Every subsequent counter read by the guest has the offset applied automatically by the CPU
3. No guest exit occurs — this is a prime example of "added virtualization support"

</details>

---

*These notes are based on the QNX Hypervisor Time Management lesson. Related topics (Running Guest Code, Virtual Devices, Interrupts) are covered in separate lessons.*
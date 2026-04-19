# QNX Hypervisor - Design Safe State (DSS)

## Table of Contents

- [Overview](#overview)
- [Local DSS (Guest Failure)](#local-dss-guest-failure)
- [Global DSS (Host Failure)](#global-dss-host-failure)
- [DMA Caveat](#dma-caveat)
- [Recovery Options](#recovery-options)
- [Quick Reference Summary](#quick-reference-summary)
- [Interview Questions & Answers](#interview-questions--answers)

---

## Overview

A **Design Safe State (DSS)** is a state that the system moves to when an **internal error occurs** (software or hardware) that the system **was not designed to handle**.

In other words: something unexpected happened, and you don't know how to recover from it cleanly. What do you do?

There are **two types** of DSS in the QNX Hypervisor context:

| DSS Type | Trigger | Action |
|---|---|---|
| **Local DSS** | A **guest** fails | Terminate the `qvm` process for that guest |
| **Global DSS** | Something in the **host** fails | Terminate all guests and **reboot** the system |

---

## Local DSS (Guest Failure)

**Scenario:** Something goes wrong inside a guest that you didn't account for and can't recover from.

**Action:** Terminate the `qvm` process.

```
┌──────────────────────┐   ┌──────────────────────┐
│   Guest A (FAILED)    │   │   Guest B (OK)        │
│                       │   │                       │
│   ❌ Error occurred    │   │   ✅ Still running     │
└──────────┬───────────┘   └───────────────────────┘
           │
           ▼
┌──────────────────────────────────────────────────────┐
│                    HOST                                │
│                                                      │
│  qvm (Guest A) ← KILL THIS PROCESS                  │
│  qvm (Guest B) ← Unaffected                         │
│  procnto, drivers, etc. ← Unaffected                │
└──────────────────────────────────────────────────────┘
```

### Why This Works

- `qvm` is **just a normal QNX process**
- You can terminate it like any other process (`kill`, signal, etc.)
- No `qvm` → no vCPU threads → **guest code stops executing**
- **Theoretically** does not affect the host or other guests

---

## Global DSS (Host Failure)

**Scenario:** Something in the host fails unexpectedly — something you can't recover from cleanly.

**Action:** Terminate all guests and **reboot** the entire system.

```
┌──────────────────────┐   ┌──────────────────────┐
│   Guest A             │   │   Guest B             │
│   ← Terminate         │   │   ← Terminate         │
└──────────────────────┘   └──────────────────────┘
                    │               │
┌───────────────────┴───────────────┴──────────────────┐
│                    HOST (FAILED)                       │
│                                                      │
│   ❌ Unrecoverable error                              │
│   → Kill all qvm processes                           │
│   → REBOOT the system                                │
└──────────────────────────────────────────────────────┘
```

### Steps

1. Terminate all guest `qvm` processes
2. Reboot the host (the host is just QNX — reboot the system)
3. Everything comes back up fresh

---

## DMA Caveat

> **Every time** guest isolation is discussed, this caveat must be mentioned.

If a guest is terminated while a driver inside it was doing **pass-through DMA**:

```
Guest running → Driver doing DMA to hardware → Guest KILLED abruptly
                                                    ↓
                                          DMA hardware left in BAD STATE
                                                    ↓
                                          Restart guest → Driver runs
                                                    ↓
                                          Driver tries to use DMA → FAILS
                                          (hardware is in unknown state)
```

### Solution

Write the driver so that **on every startup**, it **resets the DMA hardware/device**:

```
driver_init() {
    reset_device();          // Put device in known good state
    initialize_dma();        // Set up DMA from scratch
    // Now proceed normally
}
```

**Don't assume** the hardware is in a good state when the driver starts — **always reset it**.

---

## Recovery Options

The DSS actions described above are **suggestions with no recovery attempts**:

| DSS | Default Action | Recovery Option |
|---|---|---|
| **Local** | Kill `qvm`, guest stops | Detect `qvm` death → **restart `qvm`** → reboot the guest |
| **Global** | Kill all guests, reboot | Reboot → everything comes up fresh |

### Optional Recovery for Local DSS

Since `qvm` is just a process:
- You can use a **monitoring process** (e.g., QNX HAM — High Availability Manager) to detect when `qvm` dies
- The monitor can **restart `qvm`** with the same configuration file
- The guest reboots from scratch
- **But**: ensure drivers handle the DMA caveat (reset hardware on startup)

```
Monitor detects qvm death
  → Restart: qvm /data/guests/linux/linux.qvmconf
  → Guest boots fresh
  → Drivers reset hardware on init
  → System recovers
```

> Recovery is **optional** — it's your design choice. The DSS is the safe fallback when recovery isn't attempted.

---

## Quick Reference Summary

```
┌────────────────────────────────────────────────────────────────────┐
│              DESIGN SAFE STATE (DSS) SUMMARY                       │
│                                                                    │
│  LOCAL DSS (Guest Failure)                                         │
│  • Trigger: Unrecoverable error in a guest                        │
│  • Action: Kill the qvm process for that guest                    │
│  • Effect: Guest stops, host and other guests unaffected*          │
│  • *DMA caveat: hardware may be left in bad state                 │
│                                                                    │
│  GLOBAL DSS (Host Failure)                                         │
│  • Trigger: Unrecoverable error in the host                       │
│  • Action: Kill all guests + reboot the system                    │
│  • Effect: Full system restart                                    │
│                                                                    │
│  DMA CAVEAT                                                        │
│  • Guest killed during DMA → hardware in bad state                │
│  • Solution: Drivers must reset hardware on every startup          │
│                                                                    │
│  RECOVERY (Optional)                                               │
│  • Local: Monitor qvm, restart on death                           │
│  • Global: Reboot brings everything up fresh                      │
│  • Recovery is YOUR design choice, not automatic                  │
└────────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions & Answers

### Fundamentals

---

**Q1: What is a Design Safe State (DSS) in the context of the QNX Hypervisor?**

<details>
<summary>Answer</summary>

A **Design Safe State (DSS)** is a predefined state that the system transitions to when an **internal error occurs** (software or hardware) that the system **was not designed to handle**. It's the fallback action for unexpected, unrecoverable errors.

In the QNX Hypervisor, there are two types:

1. **Local DSS:** Triggered by a guest failure. The action is to **terminate the `qvm` process** for that guest, stopping the guest without (theoretically) affecting the host or other guests.

2. **Global DSS:** Triggered by a host failure. The action is to **terminate all guests and reboot** the entire system.

These are default safe actions — no recovery is attempted. Recovery (such as restarting `qvm`) is an optional enhancement that the developer can implement.

</details>

---

**Q2: What is the difference between a local DSS and a global DSS?**

<details>
<summary>Answer</summary>

| Aspect | Local DSS | Global DSS |
|---|---|---|
| **Trigger** | Guest failure (unrecoverable) | Host failure (unrecoverable) |
| **Scope** | Affects only one guest | Affects entire system |
| **Action** | Kill the `qvm` process for the failed guest | Kill all `qvm` processes + reboot |
| **Impact on other guests** | None (theoretically) | All guests terminated |
| **Impact on host** | None | Host reboots |
| **Recovery** | Optional: restart `qvm` | System comes up fresh after reboot |

Local DSS is a **surgical** response — only the affected guest is stopped. Global DSS is a **complete reset** — everything comes down and restarts.

</details>

---

**Q3: Why can you simply "kill" a guest in the QNX Hypervisor?**

<details>
<summary>Answer</summary>

Because `qvm` is **just a normal QNX process** running in the host. There's nothing special about it from the host OS perspective — it can be terminated like any other process using standard signals or kill commands.

When `qvm` is terminated:
- Its **vCPU threads** are destroyed → guest code stops executing
- Memory allocated for the guest is freed
- Virtual devices are cleaned up
- The guest effectively ceases to exist

Since each guest has its **own `qvm` process**, killing one `qvm` only stops that specific guest. Other `qvm` processes (other guests) and the rest of the host continue running unaffected.

This is a direct benefit of the QNX Hypervisor's architecture — building on top of normal QNX process management.

</details>

---

### DMA and Safety

---

**Q4: What is the DMA caveat when terminating a guest, and how do you mitigate it?**

<details>
<summary>Answer</summary>

**The caveat:** If a guest is terminated while a driver inside it was performing **pass-through DMA** to physical hardware, the DMA controller/device may be left in an **inconsistent or bad state**. This happens because the DMA operation was interrupted mid-transaction — hardware registers, buffers, and state machines may be corrupted.

**Why it matters:**
- If you restart the guest, the driver runs again
- The driver tries to use the DMA hardware, which is in a bad state
- Operations fail or produce incorrect results
- This means guest termination **can** have side effects on hardware, even though the guest's memory is isolated

**Mitigation — the only reliable solution:**
```c
driver_init() {
    reset_device();       // Reset the DMA hardware completely
    initialize_dma();     // Set up DMA from a known good state
    // Don't assume hardware is in a good state
    // Always force it into a known state
}
```

Write every driver so that it **resets the device/DMA hardware on startup**, never assuming the hardware is in a clean state. This ensures that even after an abrupt guest termination, restarting the guest will bring the hardware back to a working state.

Additionally, using an **IOMMU** can help restrict the DMA access scope, limiting potential damage from runaway DMA operations.

</details>

---

**Q5: Does killing a guest's `qvm` process guarantee no impact on other guests or the host?**

<details>
<summary>Answer</summary>

**Theoretically yes, practically almost.**

**What IS isolated:**
- Guest memory is isolated via MMU Stage 2 page tables — killing `qvm` frees that memory
- Other guests' `qvm` processes are independent and continue running
- The host OS (`procnto`, drivers, etc.) is not affected by a single `qvm` termination

**What is NOT guaranteed:**
- **DMA hardware state:** If the guest was doing pass-through DMA, the hardware may be left in a bad state. This can affect any subsequent user of that hardware (another guest or the host).
- **Shared resources:** If the guest was sharing a device with another guest or the host (via shared memory, networking, or a controlling driver), the abrupt termination may leave shared state inconsistent.
- **Interrupt masking:** If a pass-through interrupt was being handled by the guest at the time of termination, the interrupt may remain masked until properly cleared.

**Bottom line:** Memory isolation is enforced by the MMU. But hardware state and shared resource state are **not automatically cleaned up** by killing `qvm`. Proper driver design (reset on startup) and system design (monitoring, recovery) are necessary.

</details>

---

### Recovery

---

**Q6: How would you implement automatic recovery after a local DSS (guest failure)?**

<details>
<summary>Answer</summary>

**Approach:**

1. **Monitoring:** Use a monitoring process in the host (e.g., QNX HAM — High Availability Manager, or a custom watchdog process) that watches for `qvm` process death.

2. **Detection:** When the monitor detects that a `qvm` process has terminated unexpectedly, it triggers recovery.

3. **Cleanup:** Ensure any shared resources (shared memory, networking connections) are cleaned up or reset.

4. **Restart:**
```bash
# Monitor detects qvm death, restarts:
qvm /data/guests/linux/linux.qvmconf
```

5. **Driver safety:** All drivers in the guest must **reset their devices on startup** to handle the DMA caveat.

6. **Application state:** The guest boots from scratch. If the application needs to resume state:
   - Use persistent storage (flash partition) to checkpoint state
   - Use shared memory with the host to preserve critical data across restarts
   - Design the application with restart/recovery in mind

**Example architecture:**
```
┌────────────────────────────────────────────────────┐
│                    HOST                              │
│                                                    │
│  ┌──────────────────┐   ┌───────────────────────┐  │
│  │  Monitor Process  │──→│  Detects qvm death    │  │
│  │  (HAM or custom)  │   │  → Restart qvm        │  │
│  └──────────────────┘   │  → Log the event       │  │
│                          │  → Alert if repeated   │  │
│                          └───────────────────────┘  │
│                                                    │
│  qvm (Guest A) ← Monitored, auto-restart           │
│  qvm (Guest B) ← Monitored, auto-restart           │
└────────────────────────────────────────────────────┘
```

> Recovery is entirely optional and depends on your system's safety requirements. For some systems, the correct DSS action is to stay in the safe state (guest stopped) and require manual intervention.

</details>

---

**Q7: In a safety-certified system, would you prefer a local DSS or attempt recovery? What are the tradeoffs?**

<details>
<summary>Answer</summary>

**It depends on the safety requirements and certification level.**

**Arguments for staying in DSS (no recovery):**
- **Simpler to certify:** The behavior is deterministic — error → stop. No complex recovery logic to verify.
- **Predictable:** The system is in a known, safe state. No risk of recovery introducing new errors.
- **Required by some standards:** Some safety standards (e.g., ISO 26262 for automotive) may require that the system enter a safe state and stay there until manually cleared.
- **DMA safety:** Restarting might encounter the DMA bad-state problem, creating a new failure mode.

**Arguments for recovery:**
- **Availability:** Some systems (e.g., medical devices, autonomous vehicles) can't simply stop — they need to resume operation.
- **User experience:** Infotainment or non-critical guests crashing and not restarting is a poor experience.
- **Redundancy:** If the guest was a redundant component, restarting maintains system redundancy.

**Tradeoffs:**

| Factor | Stay in DSS | Attempt Recovery |
|---|---|---|
| Safety certification | Easier | More complex (recovery logic must be certified) |
| System availability | Lower | Higher |
| Predictability | Very predictable | Recovery may introduce new failure modes |
| DMA risk | None (guest stays dead) | Must handle DMA reset correctly |
| Implementation | Simple | Complex (monitoring, restart, state management) |

**Typical approach in practice:**
- **Safety-critical guests:** Stay in DSS or enter a degraded-but-safe mode
- **Non-critical guests:** Attempt automatic recovery (restart `qvm`)
- **Always log the event** for post-incident analysis
- **Rate-limit restarts** — if a guest keeps crashing, stop retrying

</details>

---

*These notes are based on the QNX Hypervisor Design Safe State (DSS) lesson. Related topics (Safety, Which QNX to Run On, Architecture) are covered in separate lessons.*
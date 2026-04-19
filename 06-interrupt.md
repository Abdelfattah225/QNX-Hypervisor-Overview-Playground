# QNX Hypervisor - Interrupts

## Table of Contents

- [Overview](#overview)
- [Virtual Interrupt Controllers](#virtual-interrupt-controllers)
- [Guest-Side Interrupts](#guest-side-interrupts)
- [Pass-Through Interrupts](#pass-through-interrupts)
- [Interrupt Latency and Shared Interrupts](#interrupt-latency-and-shared-interrupts)
- [Quick Reference Summary](#quick-reference-summary)
- [Interview Questions & Answers](#interview-questions--answers)

---

## Overview

Interrupts in the QNX Hypervisor can be handled in two places:

| Location | Description |
|---|---|
| **Host-side** | Hardware interrupt handled entirely in the host (normal QNX interrupt handling) |
| **Guest-side** | Interrupt is signaled to the guest, which handles it using its own kernel and drivers |

Guests need a **virtual interrupt controller** to identify and manage interrupts — just as a bare-metal OS needs a physical interrupt controller.

---

## Virtual Interrupt Controllers

### Platform Details

| Platform | Controller | Implementation | Notes |
|---|---|---|---|
| **x86** | IO-APIC | `vdev-ioapic.so` (separate shared object) | Must include `.so` on target and configure in `.qvmconf` |
| **ARM** | GIC (Generic Interrupt Controller) | **Built into `qvm`** (no separate `.so`) | No `.so` file needed, but still configure in `.qvmconf` |

### Why Are They Needed?

Guest code includes a **kernel** (Linux or QNX), and kernels perform interrupt-related operations:

- Configure interrupt priorities
- Map interrupt lines to specific CPUs
- **Identify (ID) interrupts** — determine which interrupt fired
- Look up handlers and wake threads
- Mask/unmask interrupts

All of these operations require communication with an interrupt controller. In a guest, this is a **virtual** interrupt controller — but the guest kernel doesn't know it's virtual.

```
┌───────────────────────────────────────────┐
│                  GUEST                     │
│                                           │
│  ┌─────────┐    ┌──────────────────────┐  │
│  │ Driver  │◄──►│    Guest Kernel      │  │
│  └─────────┘    │  - configure intrs   │  │
│                 │  - ID interrupts     │  │
│                 │  - mask/unmask       │  │
│                 └──────────┬───────────┘  │
└────────────────────────────┼──────────────┘
                             │ Normal I/O (trapped)
                             ▼
              ┌──────────────────────────┐
              │  Virtual Interrupt       │
              │  Controller              │
              │  (vdev-ioapic or vdev-gic)│
              │  inside qvm              │
              └──────────────────────────┘
```

---

## Guest-Side Interrupts

### Example: Fully Emulated Device (No Physical Hardware)

A vdev emulates an entire device (e.g., SP805 watchdog timer). When the emulated device needs to generate an interrupt:

```
┌───────────────────────────────────────────┐
│                  GUEST                     │
│                                           │
│  ┌─────────┐    ┌──────────────────────┐  │
│  │ Driver  │◄──►│    Guest Kernel      │  │
│  │ (WDT)  │    │  "Oh, an interrupt!" │  │
│  └─────────┘    │  → ID it via vdev   │  │
│                 │  → call handler      │  │
│                 └──────────────────────┘  │
└───────────────────────────────────────────┘
                        ▲
                        │ Guest Entrance
                        │ (kernel sees interrupt pending)
┌───────────────────────┼───────────────────┐
│                  HOST (qvm)               │
│                                           │
│  ┌──────────────────────────────┐         │
│  │  vdev-wdt-sp805.so           │         │
│  │                              │         │
│  │  Calls guest_signal_intr()   │         │
│  │  → signals interrupt to guest│         │
│  └──────────────────────────────┘         │
│                                           │
│  ┌──────────────────────────────┐         │
│  │  Virtual Interrupt Controller │         │
│  │  (vdev-ioapic / vdev-gic)    │         │
│  └──────────────────────────────┘         │
└───────────────────────────────────────────┘
```

### Step-by-Step Flow

1. **vdev code decides** an interrupt should occur (e.g., watchdog timer expired)
2. **vdev calls `guest_signal_intr()`** — signals the interrupt to the guest
3. **Guest entrance** occurs — guest kernel starts running
4. **Guest kernel detects** an interrupt is pending
5. **Guest kernel IDs the interrupt** by communicating with the virtual interrupt controller (this is normal I/O that gets trapped → guest exit → vdev handles → guest entrance)
6. **Guest kernel looks up handler** — wakes up the appropriate driver/IST/thread
7. **Driver handles the interrupt** — does whatever is needed
8. **Done** — entirely handled within the guest

> This is a **guest-side interrupt** — the interrupt is generated and handled entirely within the guest/vdev ecosystem. No physical hardware is involved.

---

## Pass-Through Interrupts

### When a Guest Handles a Real Hardware Interrupt

A real hardware interrupt needs to be routed from the physical hardware, through the host, to the guest.

```
┌───────────────────────────────────────────────────────┐
│                      GUEST                             │
│                                                       │
│  ┌──────────┐    ┌────────────────────────────────┐   │
│  │ Driver   │◄──►│       Guest Kernel              │   │
│  │ (serial) │    │  5. ID interrupt via virtual    │   │
│  │          │    │     interrupt controller        │   │
│  │          │    │  6. Call driver handler          │   │
│  └──────────┘    └────────────────────────────────┘   │
└───────────────────────────────────────────────────────┘
                            ▲
                            │ 4. Guest Entrance
                            │    (kernel sees interrupt)
┌───────────────────────────┼───────────────────────────┐
│                      HOST                              │
│                                                       │
│  ┌──────────────────┐  ┌──────────────────────────┐   │
│  │  qvm              │  │  Virtual Interrupt       │   │
│  │                   │  │  Controller              │   │
│  │ 3. IST runs,     │  │  (vdev-ioapic/vdev-gic)  │   │
│  │    signals intr   │  └──────────────────────────┘   │
│  │    to guest       │                                 │
│  └────────┬──────────┘                                 │
│           │ 2. Kernel schedules                        │
│           │    qvm's IST                               │
│  ┌────────┴──────────────────────────────────────┐     │
│  │  procnto (microkernel)                        │     │
│  │  1. Receives hardware interrupt               │     │
│  │     Masks the interrupt                       │     │
│  └────────┬──────────────────────────────────────┘     │
└───────────┼────────────────────────────────────────────┘
            │
┌───────────┼────────────────────────────────────────────┐
│  HARDWARE │                                            │
│           │                                            │
│  ┌────────┴─────────┐    ┌──────────────────────────┐  │
│  │ Interrupt        │◄───│ Serial Port Device       │  │
│  │ Controller (HW)  │    │ (generates interrupt)    │  │
│  └──────────────────┘    └──────────────────────────┘  │
└────────────────────────────────────────────────────────┘
```

### Step-by-Step Flow

1. **Hardware interrupt fires** — serial port device triggers the actual hardware interrupt controller
2. **Host microkernel (`procnto`) receives interrupt** — masks the interrupt, looks up its table, finds that `qvm` registered an IST (Interrupt Service Thread) for this interrupt
3. **Microkernel schedules `qvm`'s IST** — the IST runs in the `qvm` process
4. **`qvm` signals the interrupt to the guest** — sets up the environment so the guest kernel will know an interrupt occurred. Guest entrance happens.
5. **Guest kernel IDs the interrupt** — communicates with the virtual interrupt controller (trapped I/O → guest exit → vdev handles → guest entrance)
6. **Guest driver handles the interrupt** — does whatever work is needed
7. **Interrupt is unmasked** — only after the guest has fully completed handling

### Configuration

```
# In .qvmconf — use pass option for interrupt:
pass intr gic:48    # ARM example
pass intr ioapic:4  # x86 example
```

When `qvm` reads this configuration:
- It attaches an **Interrupt Service Thread (IST)** to that interrupt number on the host
- When the interrupt fires, the host routes it through `qvm` to the guest

### Critical: Interrupt Remains Masked Until Guest Completes

```
Timeline:
═══════════════════════════════════════════════════════════
│                                                         │
▼                                                         ▼
Interrupt fires              Guest finishes handling
→ MASKED immediately         → UNMASKED

  ← Interrupt stays masked for entire duration →
  ← Much longer than host-only handling! →
```

> **The actual hardware interrupt remains masked until the guest has fully handled it.** This is longer than if the interrupt were handled purely in the host.

---

## Interrupt Latency and Shared Interrupts

### Guest-Side Handling Is Slower

Handling an interrupt in a guest involves **many more steps** than handling it in the host:

**Host-only interrupt handling:**
```
1. Interrupt fires
2. Kernel masks interrupt
3. Kernel schedules IST
4. IST handles interrupt
5. Unmask interrupt
→ Done (few steps, fast)
```

**Guest pass-through interrupt handling:**
```
1. Interrupt fires
2. Host kernel masks interrupt
3. Host kernel schedules qvm's IST
4. qvm's IST signals interrupt to guest
5. Guest entrance
6. Guest kernel IDs interrupt (trap → guest exit → vdev → guest entrance)
7. Guest driver handles interrupt
8. Unmask interrupt
→ Done (many steps, slower)
```

### Impact on Shared Interrupts

If two devices share the same interrupt line:
- **Device A** → handled by a driver in the **guest**
- **Device B** → handled by a driver in the **host**

```
Device A fires interrupt
  → Interrupt MASKED
  → Routed through host → qvm → guest
  → Guest handles it (takes longer)
  → Interrupt UNMASKED

During this entire time, Device B's interrupt is also masked!
  → Device B's driver must WAIT until the guest finishes
  → Increased latency for Device B
```

> **Modern boards** have many interrupt lines, so sharing is less common than in the past (old boards might have had only 8 interrupts). But when sharing does occur, be aware of this latency impact.

---

## Quick Reference Summary

```
┌────────────────────────────────────────────────────────────────────┐
│                    INTERRUPTS SUMMARY                               │
│                                                                    │
│  VIRTUAL INTERRUPT CONTROLLERS                                     │
│  • x86: vdev-ioapic.so (separate .so, must include on target)     │
│  • ARM: GIC built into qvm (no .so needed)                        │
│  • Both must be configured in .qvmconf                             │
│  • Guest kernel talks to virtual controller (doesn't know it's    │
│    virtual)                                                        │
│                                                                    │
│  GUEST-SIDE INTERRUPTS                                             │
│  • vdev code calls guest_signal_intr()                             │
│  • Guest kernel IDs interrupt via virtual controller               │
│  • Entirely within guest/vdev — no physical hardware               │
│                                                                    │
│  PASS-THROUGH INTERRUPTS                                           │
│  • Configured with pass option (pass intr ...)                     │
│  • NOT truly pass-through — routed through host!                   │
│  • Flow: HW → host kernel → qvm IST → guest                       │
│  • HW interrupt masked until guest completes handling              │
│  • Slower than host-only handling (more steps)                     │
│                                                                    │
│  SHARED INTERRUPTS                                                 │
│  • If guest and host share an interrupt line:                      │
│    - Interrupt masked until guest finishes (longer)                │
│    - Host driver for same interrupt must wait                      │
│  • Modern boards have many interrupts — sharing less common        │
└────────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions & Answers

### Fundamentals

---

**Q1: How are virtual interrupt controllers provided to guests on ARM vs. x86?**

<details>
<summary>Answer</summary>

| Platform | Virtual Controller | Implementation |
|---|---|---|
| **x86** | IO-APIC | `vdev-ioapic.so` — a **separate shared object** that must be included on the target filesystem and configured in the `.qvmconf` file |
| **ARM** | GIC (Generic Interrupt Controller) | **Built into `qvm`** — no separate `.so` file needed. Still must be configured in the `.qvmconf` file |

Both serve the same purpose: acting as the interrupt controller that the guest kernel communicates with. The guest kernel doesn't know it's talking to a virtual controller — it does normal I/O to identify interrupts, configure priorities, map interrupts to CPUs, etc., just as it would with real hardware. Those I/O operations are trapped and handled by the virtual interrupt controller vdev.

</details>

---

**Q2: Why does a guest need a virtual interrupt controller?**

<details>
<summary>Answer</summary>

Guest code includes a **kernel** (Linux or QNX), and kernels perform many interrupt-related operations:

- **Configure interrupt priorities** at boot time
- **Map interrupt lines to specific CPUs**
- **Identify (ID) which interrupt fired** when an interrupt occurs
- **Look up interrupt handlers** and wake appropriate threads
- **Mask and unmask interrupts**

All of these operations require communicating with an interrupt controller chip. Since the guest doesn't have direct access to the physical interrupt controller (the host owns it), a **virtual interrupt controller** is provided. The guest kernel talks to it using normal I/O, which gets trapped and emulated — the kernel has no idea it's not real hardware.

</details>

---

**Q3: What is the difference between host-side and guest-side interrupt handling?**

<details>
<summary>Answer</summary>

**Host-side:** The interrupt is handled entirely within the host QNX system. A normal QNX driver handles it directly — the microkernel receives the interrupt, schedules an IST, the IST handles it, done. No guest involvement.

**Guest-side:** The interrupt is ultimately handled by a driver running inside a guest. This can happen in two ways:

1. **Signaled by a vdev** (fully emulated): A vdev's code decides an interrupt should occur and calls `guest_signal_intr()`. The guest kernel sees the interrupt on the next guest entrance, IDs it via the virtual interrupt controller, and calls the appropriate handler. No physical hardware is involved.

2. **Pass-through (real hardware):** A physical hardware interrupt fires, is received by the host microkernel, routed through `qvm`'s IST, signaled to the guest, and then the guest kernel handles it. The hardware interrupt remains masked until the guest completes handling.

Guest-side handling involves **more steps and is slower** than host-side handling.

</details>

---

### Guest-Side Interrupts

---

**Q4: Walk through how a fully emulated vdev generates and delivers an interrupt to a guest.**

<details>
<summary>Answer</summary>

Using the SP805 watchdog timer vdev as an example:

1. **vdev code determines** an interrupt should occur (e.g., watchdog timer has expired without being "kicked")
2. **vdev calls `guest_signal_intr()`** — this is a QNX hypervisor API function that signals a virtual interrupt to the guest
3. **On the next guest entrance**, the guest kernel detects that an interrupt is pending
4. **Guest kernel IDs the interrupt** by communicating with the virtual interrupt controller:
   - This involves normal I/O to the virtual controller's addresses
   - These accesses are **trapped** (guest exit)
   - The virtual interrupt controller vdev handles the request and returns the interrupt ID
   - Guest entrance resumes
5. **Guest kernel looks up its handler table** — finds which driver/IST is registered for that interrupt
6. **Guest kernel wakes the handler** — the driver handles the interrupt (e.g., takes corrective action for watchdog expiry)
7. **Done** — entirely handled within the guest, no physical hardware involved

The guest driver and kernel behave exactly as they would on bare metal — they don't know the interrupt was virtual.

</details>

---

### Pass-Through Interrupts

---

**Q5: Explain the complete flow of a pass-through interrupt from hardware to guest handler.**

<details>
<summary>Answer</summary>

1. **Configuration:** In the guest's `.qvmconf`, the interrupt is specified with the `pass` option (e.g., `pass intr gic:48`). When `qvm` starts, it registers an **IST (Interrupt Service Thread)** for that interrupt number with the host microkernel.

2. **Hardware interrupt fires:** A device (e.g., serial port) triggers the physical interrupt controller.

3. **Host microkernel receives it:** `procnto` handles the interrupt, **masks it**, and looks up which process registered for it — finds `qvm`'s IST.

4. **`qvm`'s IST is scheduled:** The microkernel schedules the IST to run.

5. **`qvm` signals the interrupt to the guest:** Sets up the guest's environment so the kernel will know an interrupt occurred.

6. **Guest entrance:** Guest kernel starts running and detects the pending interrupt.

7. **Guest kernel IDs the interrupt:** Communicates with the virtual interrupt controller (trapped I/O → guest exit → vdev handles → guest entrance).

8. **Guest driver handles the interrupt:** The appropriate driver/IST in the guest performs the actual work.

9. **Interrupt unmasked:** Only after the guest has fully completed handling, the hardware interrupt is unmasked.

**Key points:**
- Despite being called "pass-through," the interrupt is **NOT truly passed through** — it goes through the host kernel and `qvm`
- The hardware interrupt stays **masked** for the entire duration
- This is **slower** than host-only interrupt handling due to the additional steps

</details>

---

**Q6: Why are "pass-through" interrupts not truly passed through?**

<details>
<summary>Answer</summary>

Despite being configured using the `pass` option (hence the name "pass-through"), interrupts are **not directly delivered to the guest**. The interrupt follows a multi-step path:

```
Physical hardware → Hardware interrupt controller → Host microkernel (procnto)
  → qvm's IST → Signal to guest → Guest kernel → Guest driver
```

The host microkernel **must** be involved because:
1. The physical interrupt controller is managed by the host
2. The host needs to **mask** the interrupt to prevent re-entry
3. `qvm` needs to **set up the guest environment** so the guest kernel knows an interrupt occurred
4. The virtual interrupt controller in `qvm` must be updated
5. A guest entrance must be triggered

The term "pass-through" refers to the **configuration method** (`pass` option), not the actual delivery mechanism. A more accurate term might be "routed interrupt" — but "pass-through interrupt" is the established terminology.

</details>

---

### Performance and Shared Interrupts

---

**Q7: Why is guest-side interrupt handling slower than host-side? What are the implications?**

<details>
<summary>Answer</summary>

**Host-only handling (5 steps):**
1. Interrupt fires
2. Kernel masks interrupt
3. Kernel schedules IST
4. IST handles interrupt
5. Unmask interrupt

**Guest pass-through handling (8+ steps):**
1. Interrupt fires
2. Host kernel masks interrupt
3. Host kernel schedules `qvm`'s IST
4. `qvm`'s IST runs, signals interrupt to guest
5. Guest entrance occurs
6. Guest kernel IDs interrupt (requires trapped I/O → guest exit → vdev → guest entrance)
7. Guest driver handles interrupt
8. Interrupt unmasked

**Additional overhead:**
- Multiple **guest exits/entrances** during the process
- Context switches between host and guest
- Virtual interrupt controller emulation

**Implications:**
- **Higher interrupt latency** for guest-handled interrupts
- The hardware interrupt is **masked longer** — other devices sharing the same interrupt line must wait
- **Not suitable for ultra-low-latency requirements** — consider handling critical interrupts in the host instead
- If a device requires real-time interrupt response, it may be better to run its driver in the host and communicate results to the guest via shared memory or networking

</details>

---

**Q8: What happens when a guest and the host share the same interrupt line? Why is this problematic?**

<details>
<summary>Answer</summary>

When two devices share the same interrupt line, one handled by a **guest driver** and one by a **host driver**:

1. **Device A** (guest-side) fires the interrupt
2. The interrupt is **masked** at the hardware level
3. The interrupt goes through the full pass-through path: host kernel → `qvm` IST → signal to guest → guest entrance → guest IDs interrupt → guest driver handles it
4. Only when the **guest finishes handling** is the interrupt **unmasked**

**During this entire time:**
- **Device B's** interrupt (host-side) is also masked because it shares the same line
- Device B's driver **cannot receive its interrupt** until the guest finishes
- This creates **increased latency** for Device B — potentially much longer than normal

**The problem compounds because:**
- Guest-side handling already takes longer than host-side
- The mask-until-guest-completes behavior extends the masked period
- If the guest's vCPU thread is preempted during handling, it takes even longer

**Mitigation:**
- **Avoid sharing interrupt lines** between guest-handled and host-handled devices when possible
- Modern boards have many interrupt lines, making sharing less necessary
- For latency-critical devices, handle them in the host rather than the guest
- Check your board's interrupt assignments and reconfigure if needed

</details>

---

**Q9: How would you decide whether to handle an interrupt in the host or in a guest?**

<details>
<summary>Answer</summary>

| Factor | Handle in Host | Handle in Guest |
|---|---|---|
| **Latency requirement** | Low latency needed (real-time) | Latency tolerant |
| **Shared interrupt line** | Shares line with guest-handled device | Dedicated interrupt line |
| **Device ownership** | Device is used by host or shared | Device is exclusively used by the guest |
| **Driver availability** | QNX driver available for host | Driver exists for guest OS (Linux/QNX) |
| **Complexity** | Simple, fewer steps | More complex path |
| **Performance** | Faster (fewer steps, no guest exit overhead) | Slower (multiple guest exits/entrances) |

**Decision flow:**
```
Is the device latency-critical?
  → YES: Handle in host, communicate results to guest via shmem/sockets
  → NO: Does the device share an interrupt line with a host device?
         → YES: Handle in host to avoid blocking shared line
         → NO: Is the device exclusively used by one guest?
                → YES: Handle in guest (pass-through)
                → NO: Handle in host with controlling driver
```

**Best practice:** For real-time or safety-critical systems, minimize guest-side interrupt handling. Handle critical interrupts in the host and relay information to guests through shared memory or VirtIO.

</details>

---

### Scenario-Based Questions

---

**Q10: You're designing a system where a QNX guest handles a CAN bus device and a Linux guest handles an infotainment audio device. Both devices share the same interrupt line. How would you handle this?**

<details>
<summary>Answer</summary>

**Problem:** CAN bus is safety-critical and latency-sensitive. Audio is throughput-sensitive but more latency-tolerant. Sharing the interrupt line means one will block the other while handling.

**Recommended approach:**

**Option 1 (Best): Handle CAN in the host**
```
┌─────────────────┐    ┌─────────────────┐
│  QNX Guest       │    │  Linux Guest     │
│  CAN application │    │  Audio driver    │
│  (reads from     │    │  (pass-through   │
│   shared memory) │    │   interrupt)     │
└────────┬────────┘    └────────┬────────┘
         │ shmem                │ pass-through intr
┌────────┴──────────────────────┴────────────────┐
│                    HOST                         │
│  CAN driver (handles interrupt directly)        │
│  → Writes data to shared memory                 │
│  → Fast interrupt handling, unmasked quickly     │
│                                                 │
│  Audio interrupt routed to Linux guest           │
│  → Acceptable latency for audio                  │
└─────────────────────────────────────────────────┘
```

- CAN interrupt handled in host → fast, interrupt unmasked quickly
- Audio interrupt pass-through to Linux guest → longer handling time is acceptable
- CAN data communicated to QNX guest via shared memory
- Shared interrupt line impact is minimized because CAN handling is fast

**Option 2: Reconfigure hardware interrupt assignments**
- If the board allows, reassign devices to **separate interrupt lines**
- Then both can be pass-through to their respective guests without blocking each other

**Option 3 (Avoid): Both as pass-through**
- CAN in QNX guest, audio in Linux guest, both pass-through
- The audio guest could block CAN handling (or vice versa) due to shared line
- **Not recommended** for safety-critical CAN

</details>

---

**Q11: A developer reports that pass-through interrupts in their guest are arriving late. What could be the causes?**

<details>
<summary>Answer</summary>

**Possible causes:**

1. **vCPU thread preemption:** The vCPU thread has low priority and is being preempted by higher-priority host threads. Even after `qvm`'s IST signals the interrupt, the guest can't process it until the vCPU thread runs.

2. **`qvm`'s IST priority:** The interrupt service thread in `qvm` that receives the hardware interrupt may have low priority, delaying the routing to the guest.

3. **Shared interrupt line:** Another device on the same interrupt line is being handled (possibly in another guest), keeping the interrupt masked.

4. **Excessive guest exits:** The guest is experiencing many other guest exits (from vdevs, IPIs, etc.) that delay when the interrupt can be processed.

5. **High stolen time:** The vCPU thread is spending too much time in READY state or handling emulation, reducing available time for interrupt processing.

6. **Guest-internal scheduling:** Inside the guest, the interrupt handler thread may have low priority relative to other guest threads.

**Investigation steps:**
1. Capture kernel event log → analyze guest exits and vCPU READY time
2. Check vCPU thread priority and `qvm` IST priority
3. Check for shared interrupt lines
4. Measure stolen time for the vCPU
5. Check guest-internal thread priorities

**Fixes:**
1. Raise vCPU thread priority
2. Raise `qvm`'s IST priority
3. Eliminate interrupt sharing if possible
4. Reduce guest exits (switch to VirtIO vdevs)
5. Assign vCPU to a dedicated cluster

</details>

---

**Q12: Compare `guest_signal_intr()` (guest-side interrupt) with the pass-through interrupt mechanism. When would you use each?**

<details>
<summary>Answer</summary>

| Aspect | `guest_signal_intr()` (Guest-Side) | Pass-Through Interrupt |
|---|---|---|
| **Trigger source** | vdev code (software) | Physical hardware device |
| **Physical hardware** | None required | Real device generates the interrupt |
| **Configuration** | Part of vdev implementation | `pass intr` in `.qvmconf` |
| **Host kernel involvement** | None — vdev directly signals guest | Full — host kernel receives, masks, routes via IST |
| **Latency** | Lower (fewer steps) | Higher (more steps through host) |
| **Use case** | Emulated devices with no real hardware | Real hardware devices whose interrupts the guest handles |
| **Interrupt masking** | N/A (no real hardware to mask) | Hardware interrupt masked until guest completes |
| **Examples** | Watchdog timer emulation, VirtIO device notifications | Serial port, network card, GPIO |

**When to use each:**
- **`guest_signal_intr()`:** When writing custom vdevs that emulate devices, or when VirtIO back-end code needs to notify the guest of an event
- **Pass-through:** When the guest needs to handle interrupts from real physical hardware that has been assigned to it

</details>

---

*These notes are based on the QNX Hypervisor Interrupts lesson. Related topics (Virtual Devices, Running Guest Code, Safety/IOMMUs) are covered in separate lessons.*
# QNX Hypervisor - Safety Issues and Solutions (Recap)

## Table of Contents

- [Spatial Isolation](#spatial-isolation)
- [Temporal Isolation](#temporal-isolation)
- [CPU Privileges](#cpu-privileges)
- [DMA Security](#dma-security)
- [Quick Reference Summary](#quick-reference-summary)
- [Interview Questions & Answers](#interview-questions--answers)

---

## Spatial Isolation

Guests **cannot harm each other or the host** — enforced by memory isolation.

### How It Works

- Each guest runs in its own **virtual address space**
- The MMU's **Stage 2 page tables** (programmed by `qvm`) restrict guest memory access
- A guest **cannot write outside** its allocated address space
- If a guest crashes, it's just the `qvm` process dying — it doesn't crash the kernel or harm anything else

```
┌─────────────────┐  ┌─────────────────┐  ┌───────────────┐
│ Guest A          │  │ Guest B          │  │    HOST       │
│ (own address     │  │ (own address     │  │               │
│  space)          │  │  space)          │  │               │
│                  │  │                  │  │               │
│ Cannot access →  │  │ ← Cannot access  │  │  Protected    │
│ Guest B or Host  │  │ Guest A or Host  │  │  from guests  │
└─────────────────┘  └─────────────────┘  └───────────────┘
         │                    │                     │
         └────────────────────┴─────────────────────┘
                    MMU enforces isolation
```

### The DMA Caveat (Always Mentioned)

If a guest crashes while doing **pass-through DMA**:
- DMA hardware may be left in a **bad state**
- Restarting the guest → driver tries to use DMA → fails

**Solutions:**
1. **Driver reset:** Write drivers to always reset the device on startup
2. **Reboot:** Full system reboot puts hardware in a good state
3. **IOMMU/IOMPU:** Secure DMA access to limit damage scope

### QNX Microkernel Protection

- Each guest is just a `qvm` process in its **own virtual address space**
- The QNX microkernel model **protects from errant processes**
- A crashing `qvm` doesn't bring down the host or other guests
- Drivers, file systems, and networking run as separate processes — one failing doesn't cascade

---

## Temporal Isolation

Guests can be guaranteed **sufficient time to run** through proper priority management.

### How It Works

- Guest execution depends on **vCPU thread scheduling**
- vCPU threads are normal QNX threads with configurable **priorities**
- Arrange priorities so guests meet their **timing deadlines**

```
Priority arrangement:
  Higher priority → Safety-critical host threads (run first when needed)
  ↓
  Medium priority → Safety-critical guest vCPU threads
  ↓
  Lower priority  → Non-critical guest vCPU threads
```

### Configuration

```
# In .qvmconf:
cpu
  sched 63     # High-priority vCPU for safety-critical guest

cpu
  sched 10     # Lower-priority vCPU for non-critical guest
```

### Key Points

- Lower-priority things **won't preempt** your safety-critical guests
- Higher-priority things **will preempt** — but that's intentional (they're more critical)
- It's the **vCPU thread priorities** that matter, not guest-internal priorities
- Proper priority arrangement ensures temporal isolation between guests

---

## CPU Privileges

- Guests run at a **lower privilege level** than the hypervisor microkernel
- Privileged instructions are **trapped and handled** by `qvm`
- Guests cannot escalate privileges or bypass the hypervisor
- ARM: Exception Levels (EL0/EL1 for guests, EL2 for hypervisor)
- x86: Rings (Ring 3 for guests, Ring 0 for hypervisor, VMX root)

---

## DMA Security

DMA operations can be secured through:

| Mechanism | Function |
|---|---|
| **IOMMU** | Address **translation** + protection for DMA operations |
| **IOMPU** | **Protection only** (no translation) for DMA operations |

These prevent guests from using DMA to access memory outside their allocated range.

---

## Quick Reference Summary

```
┌────────────────────────────────────────────────────────────────────┐
│              SAFETY ISSUES AND SOLUTIONS SUMMARY                   │
│                                                                    │
│  SPATIAL ISOLATION                                                 │
│  • MMU Stage 2 page tables enforce memory boundaries               │
│  • Guest can't access other guests' or host's memory               │
│  • Guest crash = qvm process death (no cascade)                    │
│  • DMA caveat: pass-through DMA may leave HW in bad state          │
│  • Solutions: driver reset on startup, IOMMU, or reboot            │
│                                                                    │
│  TEMPORAL ISOLATION                                                │
│  • vCPU thread priorities determine guest execution time            │
│  • Configure priorities in .qvmconf (sched option)                 │
│  • Proper priority arrangement = timing deadlines met              │
│  • Higher-priority = preempts (intentional for critical tasks)     │
│                                                                    │
│  CPU PRIVILEGES                                                    │
│  • Guests at lower privilege than hypervisor                       │
│  • Privileged instructions trapped and handled                     │
│                                                                    │
│  DMA SECURITY                                                      │
│  • IOMMU: address translation + protection                        │
│  • IOMPU: protection only                                          │
└────────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions & Answers

---

**Q1: What are the main safety mechanisms provided by the QNX Hypervisor?**

<details>
<summary>Answer</summary>

The QNX Hypervisor provides four main safety mechanisms:

1. **Spatial Isolation:** Each guest runs in its own virtual address space, enforced by MMU Stage 2 page tables. Guests cannot access each other's memory or the host's memory. A guest crash (qvm death) doesn't affect other components.

2. **Temporal Isolation:** Guest execution time is controlled by vCPU thread priorities. By properly arranging priorities, you ensure safety-critical guests get sufficient CPU time and aren't starved by non-critical workloads.

3. **CPU Privileges:** Guests run at lower privilege levels than the hypervisor. Privileged instructions are trapped and handled by `qvm`, preventing guests from bypassing security boundaries.

4. **DMA Security:** IOMMU (translation + protection) or IOMPU (protection only) can restrict DMA operations, preventing guests from using DMA to access memory outside their allocated range.

**Caveat:** Spatial isolation has a limitation with pass-through DMA — if a guest crashes mid-DMA, hardware may be left in a bad state. Drivers must reset devices on startup.

</details>

---

**Q2: Explain how spatial isolation and temporal isolation work together to provide safety in the QNX Hypervisor.**

<details>
<summary>Answer</summary>

**Spatial isolation** ensures that guests are **memory-separated**:
- Each guest's memory is bounded by MMU page tables
- A misbehaving guest cannot corrupt another guest's data or the host's memory
- This prevents one faulty component from causing data corruption in another

**Temporal isolation** ensures that guests get **predictable execution time**:
- vCPU thread priorities control when each guest runs
- Safety-critical guests are given higher priority → they preempt non-critical work
- Non-critical guests cannot monopolize the CPU at the expense of critical guests

**Together they provide:**
- **Space:** Guest A's bug can't corrupt Guest B's memory
- **Time:** Guest A's CPU-intensive workload can't starve Guest B of CPU time
- **Independence:** Each guest operates as if it has its own dedicated hardware
- **Safety certification:** Both forms of isolation support independent certification of guest components

Without temporal isolation, a guest could monopolize the CPU even if memory is isolated. Without spatial isolation, a guest could corrupt other guests' data even if it has sufficient CPU time. Both are needed for a complete safety architecture.

</details>

---

**Q3: A safety auditor asks: "Can you guarantee that a guest failure will never affect the host?" How do you respond?**

<details>
<summary>Answer</summary>

**Mostly yes, with one important caveat:**

**What IS guaranteed:**
- Guest memory is isolated by the MMU — a guest cannot write to host memory or other guests' memory
- `qvm` is just a process — if it crashes, the QNX microkernel continues running. The process death doesn't crash the kernel.
- Other `qvm` processes (other guests) are unaffected
- CPU privileges prevent guests from executing privileged instructions that could affect the host

**The DMA caveat:**
- If a guest was performing **pass-through DMA** when it crashed, the DMA hardware may be left in an **inconsistent state**
- This is a hardware state issue, not a memory isolation issue
- The bad hardware state could affect subsequent users of that hardware (which could include host components)

**Mitigations for the caveat:**
1. **IOMMU:** Restricts DMA to the guest's allocated memory range — even if DMA goes wrong, it can only affect the guest's memory
2. **Driver design:** All drivers reset hardware on startup, ensuring a known good state
3. **Unity guests (if no IOMMU):** 1:1 address mapping to avoid DMA targeting wrong memory
4. **System reboot (Global DSS):** If the state is truly unrecoverable, reboot puts everything back to a clean state

**Honest answer to the auditor:** Memory isolation is hardware-enforced by the MMU. DMA is the exception — but with an IOMMU and proper driver design, the risk is mitigated. We document this caveat and our mitigation strategy.

</details>

---

**Q4: How do you achieve temporal isolation for a safety-critical QNX guest running alongside a non-critical Linux guest?**

<details>
<summary>Answer</summary>

**Configuration approach:**

```
# Safety-critical QNX guest (.qvmconf):
cpu
  sched 63           # High priority
  cluster bigcpus    # Run on high-performance cores
cpu
  sched 63
  cluster bigcpus

# Non-critical Linux guest (.qvmconf):
cpu
  sched 10           # Low priority
  cluster littlecpus # Run on low-power cores
cpu
  sched 10
  cluster littlecpus
```

**How this achieves temporal isolation:**

1. **Priority separation:** QNX guest vCPU threads (priority 63) will always preempt Linux guest vCPU threads (priority 10) when both compete for the same CPU
2. **Cluster separation:** By assigning guests to different clusters (big vs. little cores), they don't even compete for the same CPUs
3. **Combined effect:** The QNX guest gets guaranteed execution time on high-performance cores. The Linux guest runs on separate, lower-performance cores and cannot interfere.

**Additional measures:**
- Ensure no host threads run at priority > 63 unless they are truly more critical
- Monitor stolen time for the QNX guest to verify it's getting sufficient CPU
- Split any high-priority CPU-hogging host threads into brief high-priority work + lower-priority continuation

</details>

---

*These notes are based on the QNX Hypervisor Safety Issues and Solutions lesson. Related topics (DSS, Watchdogs, Interrupts, Running Guest Code) are covered in separate lessons.*
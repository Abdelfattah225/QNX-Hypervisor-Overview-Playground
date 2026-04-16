

# QNX Hypervisor Architecture

## Table of Contents

- [What is the QNX Hypervisor?](#what-is-the-qnx-hypervisor)
- [Hypervisor vs. Machine Emulator](#hypervisor-vs-machine-emulator)
- [Supported Guests and Hardware](#supported-guests-and-hardware)
- [Architecture Under the Covers](#architecture-under-the-covers)
- [Guest Configuration](#guest-configuration)
- [How VMs Work](#how-vms-work)
- [Memory Architecture](#memory-architecture)
- [Unity Guests](#unity-guests)
- [Interview Questions & Answers](#interview-questions--answers)

---

## What is the QNX Hypervisor?

The QNX Hypervisor is **software that runs multiple operating systems on the same SoC** (System on Chip) — the same chip, the same board.

**Example:** QNX, Linux, and Android can all run simultaneously on a single board managed by the hypervisor.

**Why use it?**
- Run multiple OSes on one board
- Eliminates the need for multiple physical boards (e.g., in a vehicle)
- **Saves on hardware costs**

**Conceptual Layout:**

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│  QNX OS  │  │  Linux   │  │ Android  │
│ (Guest)  │  │ (Guest)  │  │ (Guest)  │
└────┬─────┘  └────┬─────┘  └────┬─────┘
     │             │              │
┌────┴─────────────┴──────────────┴─────┐
│            QNX Hypervisor             │
├───────────────────────────────────────┤
│    Hardware (CPU, Memory, Devices)    │
└───────────────────────────────────────┘
```

---

## Hypervisor vs. Machine Emulator

### Machine Emulator (e.g., QEMU)

- Emulates a **different architecture** (e.g., ARM on x86 hardware)
- **Every single instruction** must be emulated
- One ARM instruction might require 10, 100, or 1000 x86 instructions
- **Result: Very slow**

```
┌─────────────────────┐
│   ARM OS (Guest)    │
├─────────────────────┤
│   ARM Emulator      │  ← Every instruction emulated
├─────────────────────┤
│  x86 Hardware       │
└─────────────────────┘
```

### Hypervisor (e.g., QNX Hypervisor)

- **Most instructions execute directly on the CPU** — at bare metal speed
- The CPU may do a little extra work for virtualization support (negligible overhead)
- **Only some instructions** get trapped and emulated
- Guests generally think they're running on normal hardware

**Key Tradeoff:** The guest OS architecture **must match** the physical CPU architecture. Running an ARM OS requires ARM hardware.

| Feature | Emulator | Hypervisor |
|---|---|---|
| Instruction execution | All emulated | Most run directly on CPU |
| Performance | Slow | Near bare-metal speed |
| Architecture match required? | No | **Yes** |
| Example | QEMU | QNX Hypervisor, VMware |

---

## Supported Guests and Hardware

### Guest Operating Systems

| Guest OS | Notes |
|---|---|
| **QNX 8** | Fully supported |
| **Linux (Ubuntu 20.04)** | Fully supported |
| **Android** | Supported (from hypervisor's perspective, Android = Linux) |
| Older QNX versions | Possible — contact QNX sales |

> **Important Note:** From the hypervisor's point of view, **Linux and Android are identical** — they're both Linux kernels. However, from a customer perspective, Android has different software, different sources, and a different installation procedure.

### Hardware Platforms

- **x86-64** (64-bit only)
- **ARMv8 using AArch64** (64-bit only)

### QAVF (QNX Advanced Virtualization Framework)

A separate product that provides:
- **Graphics sharing** between multiple guests
- **Audio**
- **Touch screen** support
- **Virtual sockets**
- **File systems**
- And more

---

## Architecture Under the Covers

### Core Design Principle

The QNX Hypervisor is built on top of **normal QNX**. It consists of:

- **One `qvm` process per guest** — each is a normal QNX process
- **Standard QNX underneath** — `procnto` (process manager + microkernel), drivers, `io-sock`, and other standard processes

```
┌─────────────────────────────────────────────────┐
│                    THE HOST                       │
│                                                   │
│  ┌──────────┐  ┌──────────┐                       │
│  │   qvm    │  │   qvm    │   ← Normal QNX       │
│  │(QNX Guest)│ │(Linux    │     processes!        │
│  │          │  │ Guest)   │                       │
│  └──────────┘  └──────────┘                       │
│                                                   │
│  ┌─────────────────────────────────────────────┐ │
│  │  procnto (microkernel + process manager)    │ │
│  │  drivers, io-sock, random, etc.             │ │
│  │           Normal QNX                        │ │
│  └─────────────────────────────────────────────┘ │
└─────────────────────────────────────────────────┘
```

> **Key Insight:** The `qvm` processes are just **normal QNX processes**. The "hypervisor" is simply normal QNX plus `qvm` instances. Together, all of this is called **the host**.

### Demo Walkthrough (Raspberry Pi)

1. **Boot QNX** on Raspberry Pi via serial terminal → normal QNX running
2. **SSH into the Pi** → run `qvm` with a QNX guest configuration file
3. **SSH again** → run another `qvm` with a Linux guest configuration file
4. Result: **Two `qvm` processes** visible on the host, each managing a guest

```
Host QNX:  Shows two qvm processes running
SSH #1:    QNX guest booted — can run `pidin` and other QNX commands
SSH #2:    Linux guest booted — `uname -a` confirms Linux (built with Yocto)
```

---

## Guest Configuration

Each guest is configured via a **`.qvmconf` configuration file** that describes the virtual machine.

### Configuration File Structure

```
# Give the virtual machine a name
system name_of_guest

# Define RAM: size and starting guest physical address
ram 512M,0x80000000
# If no address specified, guest physical address starts at 0

# Define number of virtual CPUs
cpu 2

# Load guest OS images
load /path/to/kernel_image
load /path/to/ramdisk_image

# Linux kernel command line (if applicable)
cmdline "console=ttyAMA0 root=/dev/ram rw"

# Virtual devices (vdev)
vdev pl011
  # Defines a virtual PL011 serial port for the guest

# Passthrough mapping (pass)
pass 0xFE000000,0x3F000000,0x1000000
#    host_phys    guest_phys   size
```

### Key Configuration Options

| Option | Purpose |
|---|---|
| `system` | Name of the virtual machine |
| `ram` | Amount and location of guest RAM |
| `cpu` | Number of virtual CPUs assigned to the guest |
| `load` | Path to guest OS image files to load |
| `cmdline` | Kernel command line (for Linux guests) |
| `vdev` | Virtual device definitions (serial, console, networking, block devices, etc.) |
| `pass` | Passthrough memory mapping — maps host physical address directly into guest address space |

### Passthrough (`pass`) Explained

```
pass <host_physical_addr>,<guest_physical_addr>,<size>
```

- Creates a **direct mapping** from host physical memory to guest physical address space
- The guest accesses the device memory **directly** — bypassing the hypervisor
- Common use case: **video memory**, device registers
- Memory mapped via `pass` is **NOT zero-filled** (unlike `ram`)

### Example: QNX Guest Config (Raspberry Pi)

```
system qnx_guest
ram 128M,0x...         # QNX doesn't usually need a lot of RAM
cpu 2
load /data/guests/qnx/qnx_boot_image.ifs
pass ...               # Passthrough mappings
vdev ser               # Virtual serial port
vdev console           # Console
vdev shmem             # Shared memory
vdev net               # Networking
```

### Example: Linux Guest Config (Raspberry Pi)

```
system linux_guest
ram 400M,0x...         # Linux gets more RAM
cpu 2
load /data/guests/linux/kernel_image
load /data/guests/linux/ramdisk_image
cmdline "..."          # Linux kernel command line
vdev pl011             # Virtual serial port
vdev blk               # Block device
vdev console
vdev shmem
vdev net
```

---

## How VMs Work

### Instruction Execution Model

```
┌────────────────────────────────────────────────────┐
│              Guest Instructions                     │
├──────────────┬──────────────────┬──────────────────┤
│  MOST        │  SOME            │  FEW             │
│  Execute     │  CPU adds minor  │  Trapped &       │
│  directly    │  virtualization  │  emulated by     │
│  on CPU      │  overhead        │  qvm             │
│  (bare metal │  (negligible)    │                  │
│   speed)     │                  │                  │
└──────────────┴──────────────────┴──────────────────┘
```

- **Most instructions** → execute directly on the CPU at bare-metal speed
- **Some instructions** → CPU does a little extra virtualization work (negligible overhead)
- **Few instructions** → trapped and handled by `qvm` (e.g., accessing certain addresses, `SMC` instruction on ARM)

> **Guest Perspective:** Guests generally believe they are running on **normal hardware** and don't know there's a hypervisor underneath.

**Exception: Para-virtualization** — In this case, the guest **knows** it's running on a hypervisor and cooperates with it. (Covered in a separate lesson.)

---

## Memory Architecture

### Two-Layer Address Translation

The hypervisor introduces a **second layer** of address translation via the MMU (Memory Management Unit):

```
┌─────────────────────────────┐
│     Guest Process           │
│  (Virtual Addresses)        │
└──────────┬──────────────────┘
           │  Stage 1 / Normal Page Tables
           │  (Set up by guest OS)
           ▼
┌─────────────────────────────┐
│  Guest Physical Addresses   │
│  (What the guest thinks     │
│   is physical memory)       │
└──────────┬──────────────────┘
           │  Stage 2 (ARM) / Extended Page Tables (x86)
           │  (Set up by qvm)
           ▼
┌─────────────────────────────┐
│  Host Physical Addresses    │
│  (Actual physical memory    │
│   on the board)             │
└─────────────────────────────┘
```

### How It Works

1. **`qvm` reads the configuration file** → sees the `ram` option
2. **`qvm` asks `procnto`** (process manager/memory manager) to allocate memory
3. **`qvm` programs the MMU** with Stage 2 / Extended Page Tables to create the guest-physical → host-physical mapping
4. **Guest OS boots** and creates its own Stage 1 page tables (virtual → guest physical) — the hypervisor doesn't care about this layer
5. **At runtime:** CPU resolves addresses through **both layers** of page tables automatically via the MMU

### Key Properties of Guest Memory

| Property | Detail |
|---|---|
| **Guest physical memory is contiguous** | From the guest's perspective, its memory is one contiguous block |
| **Host physical memory may NOT be contiguous** | The actual allocated memory on the host can be scattered |
| **Guest cannot access outside its address space** | MMU enforces protection — just like normal operation without a hypervisor |
| **RAM is zero-filled** | Memory allocated via the `ram` option is zeroed out (takes time for large allocations) |
| **Pass memory is NOT zero-filled** | Memory mapped via `pass` is not zeroed — can be faster at boot |
| **Total guest memory limited by host** | Must account for host OS, other guests, and host processes |

### Platform Terminology

| Concept | x86 Term | ARM Term |
|---|---|---|
| Guest's perceived physical addresses | **Guest Physical Address (GPA)** | **Intermediate Physical Address (IPA)** |
| Actual physical addresses on the board | **Host Physical Address (HPA)** | **Physical Address (PA)** |
| Second-layer page tables | **Extended Page Tables (EPT)** | **Stage 2 Page Tables** |

> **QNX documentation uses x86 terminology** (Guest Physical Address, Host Physical Address) because it's clearer.

### Performance Trick: Using `pass` for RAM

You can use **typed memory** in the QNX host to set aside RAM, then use the `pass` option to map it into the guest physical address space. This memory is **not zero-filled**, resulting in **faster guest boot times**.

```
# Instead of:
ram 512M        # Zero-filled, slower boot

# You can:
# 1. Set up typed memory in QNX host
# 2. Use pass to map it into guest address space
pass <typed_mem_host_addr>,<guest_addr>,<size>   # Not zero-filled, faster boot
```

---

## Unity Guests

A **Unity Guest** has a **one-to-one mapping** between guest physical addresses and host physical addresses.

```
Guest Physical Address    Host Physical Address
0x00000000           →    0x00000000
0x10000000           →    0x10000000
```

### Key Points

- The MMU is **still involved** — page tables still need to be programmed
- It's just a **1:1 mapping** instead of a remapped one
- **Use case:** When you **don't have an IOMMU** (I/O Memory Management Unit), a Unity guest may be necessary (covered in the Safety lesson)
- **Complex to configure** — contact QNX engineering for help
- **Not the normal case** — normally, guest and host physical addresses are different

---

## Quick Reference Summary

```
┌─────────────────────────────────────────────────────────────┐
│                     QNX HYPERVISOR                           │
│                                                             │
│  • Not an emulator — most instructions run at bare-metal    │
│  • One qvm process per guest (normal QNX process)           │
│  • Host = normal QNX + qvm instances                        │
│  • Guest configured via .qvmconf files                      │
│  • Two-layer MMU translation (Stage 1 + Stage 2)            │
│  • Supports: QNX 8, Linux, Android on x86-64 & ARMv8       │
│  • Guest memory isolated — MMU enforces protection          │
│  • Pass option for direct device memory access              │
│  • Para-virtualization = guest knows about hypervisor       │
│  • Unity guest = 1:1 address mapping (special case)         │
└─────────────────────────────────────────────────────────────┘
```

---

## Interview Questions & Answers

### Fundamentals

---

**Q1: What is the QNX Hypervisor and why would you use it?**

<details>
<summary>Answer</summary>

The QNX Hypervisor is software that allows multiple operating systems (e.g., QNX, Linux, Android) to run simultaneously on the same SoC/board. The primary motivation is **hardware consolidation** — instead of having separate physical boards for each OS in a system (e.g., in a vehicle), you can run them all on one board, which **reduces hardware cost, weight, power consumption, and complexity**.

</details>

---

**Q2: What is the fundamental difference between a hypervisor and a machine emulator like QEMU?**

<details>
<summary>Answer</summary>

- **Emulator:** Every single guest instruction is emulated in software. The guest architecture can differ from the host (e.g., ARM guest on x86 host). This makes emulators **very slow** — one guest instruction may require hundreds of host instructions.
- **Hypervisor:** Most guest instructions execute **directly on the physical CPU** at bare-metal speed. Only a small number of privileged or sensitive instructions are trapped and emulated. The tradeoff is that the **guest OS architecture must match the host CPU architecture**.

</details>

---

**Q3: Can you run an ARM guest on x86 hardware using the QNX Hypervisor? Why or why not?**

<details>
<summary>Answer</summary>

**No.** Because a hypervisor executes most guest instructions directly on the physical CPU, the guest OS architecture must match the host hardware. If the hardware is x86, only x86 guests can run. If you need to run an ARM guest on x86 hardware, you would need an **emulator** (like QEMU), not a hypervisor.

</details>

---

**Q4: What guest operating systems does the QNX Hypervisor support?**

<details>
<summary>Answer</summary>

- **QNX 8** (fully supported)
- **Linux** (Ubuntu 20.04)
- **Android** (from the hypervisor's perspective, Android is just Linux)
- **Older QNX versions** (possible, requires consultation with QNX)

Supported hardware architectures: **x86-64** and **ARMv8 (AArch64)**, 64-bit only.

</details>

---

**Q5: Why does the QNX Hypervisor treat Android and Linux as the same thing?**

<details>
<summary>Answer</summary>

From the hypervisor's perspective, Android **is** Linux — it uses the Linux kernel. The hypervisor doesn't care about the userspace differences. However, from a **customer/integration** perspective, Android is different: it comes from different sources (AOSP), has different software stacks, different build systems, and different installation procedures.

</details>

---

### Architecture

---

**Q6: Describe the internal architecture of the QNX Hypervisor. What is the `qvm` process?**

<details>
<summary>Answer</summary>

The QNX Hypervisor runs on top of **standard QNX** (with `procnto`, drivers, and other normal processes). For each guest virtual machine, there is **one `qvm` process** — which is just a normal QNX process. The `qvm` process acts as the virtual machine manager for its guest. It reads a configuration file, allocates resources (memory, CPUs), sets up MMU mappings, loads guest images, and handles trapped instructions.

Together, normal QNX plus all `qvm` instances form what is called **the host**.

</details>

---

**Q7: What is the "host" in the context of the QNX Hypervisor?**

<details>
<summary>Answer</summary>

The **host** refers to everything that is not a guest: the standard QNX operating system (including `procnto`, the microkernel, process manager, drivers, networking stacks, etc.) plus all the `qvm` processes managing the guests. The host is essentially **normal QNX** with `qvm` instances running as regular processes.

</details>

---

**Q8: What is `procnto` and what role does it play in the hypervisor architecture?**

<details>
<summary>Answer</summary>

`procnto` is the QNX **microkernel combined with the process manager**. In the hypervisor context, it plays the same role as in any QNX system — it manages processes, threads, memory, and scheduling. When `qvm` needs to allocate memory for a guest, it requests it from `procnto`'s memory manager. `procnto` is part of the host.

</details>

---

**Q9: Is the QNX Hypervisor a Type 1 or Type 2 hypervisor? Explain.**

<details>
<summary>Answer</summary>

This is nuanced. The QNX Hypervisor could be considered a **Type 1 (bare-metal) hypervisor** because QNX itself runs directly on the hardware without another OS underneath. However, architecturally, it behaves somewhat like a **Type 2 hypervisor** since the `qvm` processes run as user-space processes on a full QNX operating system. In practice, QNX literature treats it as a **Type 1 hypervisor** because the host OS (QNX) is a lightweight RTOS running directly on bare metal, and the `qvm` processes leverage hardware virtualization extensions directly.

</details>

---

### Configuration

---

**Q10: How do you configure a guest virtual machine in the QNX Hypervisor?**

<details>
<summary>Answer</summary>

Each guest is configured via a **`.qvmconf` configuration file**. This text file describes:
- **`system`** — name of the VM
- **`ram`** — amount of guest RAM and starting guest physical address
- **`cpu`** — number of virtual CPUs
- **`load`** — paths to guest OS image files (kernel, ramdisk, boot image)
- **`cmdline`** — kernel command line (for Linux guests)
- **`vdev`** — virtual devices (serial ports, consoles, block devices, networking, shared memory)
- **`pass`** — passthrough memory mappings for direct device access

You start a guest by running `qvm` and passing the path to its configuration file.

</details>

---

**Q11: What is the `pass` option in the QNX Hypervisor configuration, and when would you use it?**

<details>
<summary>Answer</summary>

The `pass` (passthrough) option creates a **direct mapping** from a host physical address to a guest physical address, allowing the guest to access device memory **directly without going through the hypervisor**. 

Syntax: `pass <host_physical_addr>,<guest_physical_addr>,<size>`

**Use cases:**
- Mapping **video/GPU memory** into the guest address space
- Mapping **device registers** for direct hardware access
- Mapping pre-allocated **typed memory** for faster boot (since `pass` memory is not zero-filled)

The guest accesses the mapped region as if it were normal memory at the guest physical address, but the actual access goes directly to the host physical address.

</details>

---

**Q12: What is a `vdev` in the QNX Hypervisor configuration?**

<details>
<summary>Answer</summary>

`vdev` stands for **virtual device**. It defines a virtualized hardware device that the guest will see. Examples include:
- **`pl011`** — a virtual ARM PL011 serial port
- **`ser`** — a virtual serial device
- **`console`** — a virtual console
- **`blk`** — a virtual block device (disk)
- **`net`** — a virtual network device
- **`shmem`** — shared memory device

These virtual devices are emulated by `qvm`. When the guest accesses the addresses associated with a vdev, those accesses are **trapped** and handled by the `qvm` process, which emulates the expected hardware behavior.

</details>

---

**Q13: How do you start a guest VM on the QNX Hypervisor?**

<details>
<summary>Answer</summary>

You run the `qvm` command and provide the path to the guest's configuration file:

```bash
qvm /data/guests/linux/linux-rpi.qvmconf
```

This starts a new `qvm` process that:
1. Reads the configuration file
2. Allocates memory from the host
3. Programs the MMU with guest-to-host address mappings
4. Loads the guest OS images into memory
5. Sets up virtual devices
6. Starts executing the guest OS

Each guest requires its own `qvm` process.

</details>

---

### Memory Architecture

---

**Q14: Explain the two-layer address translation in the QNX Hypervisor.**

<details>
<summary>Answer</summary>

The QNX Hypervisor uses **two layers of MMU page table translation**:

1. **Stage 1 (set up by the guest OS):** Translates **guest virtual addresses → guest physical addresses**. This is the normal page table mapping that any OS creates for its processes. The hypervisor doesn't manage this.

2. **Stage 2 / Extended Page Tables (set up by `qvm`):** Translates **guest physical addresses → host physical addresses**. This mapping is what the hypervisor controls to give each guest its own isolated view of physical memory.

On **ARM**, these are called **Stage 2 page tables**. On **x86**, they're called **Extended Page Tables (EPT)**.

At runtime, the CPU's MMU resolves both layers automatically — the guest process uses a virtual address, the MMU walks Stage 1 to get the guest physical address, then walks Stage 2 to get the actual host physical address. This all happens in hardware with minimal overhead.

</details>

---

**Q15: What is the difference between guest physical addresses and host physical addresses?**

<details>
<summary>Answer</summary>

- **Guest Physical Address (GPA):** The address the guest OS believes is the physical address. This is what the guest sees as its physical memory. It's typically contiguous starting from 0 (or a configured offset).

- **Host Physical Address (HPA):** The actual physical address on the real hardware. The memory backing a guest may be non-contiguous on the host.

The **MMU** translates GPAs to HPAs via Stage 2 / Extended Page Tables. The guest never sees or knows about HPAs — it only works with GPAs.

**ARM terminology note:** ARM calls GPAs "Intermediate Physical Addresses (IPA)" and HPAs simply "Physical Addresses (PA)". QNX documentation uses the x86 terminology (GPA/HPA) for clarity.

</details>

---

**Q16: Is guest physical memory contiguous on the host? Why or why not?**

<details>
<summary>Answer</summary>

**No, not necessarily.** From the **guest's perspective**, its physical memory is contiguous (e.g., 0x00000000 through 0x08000000 for 128MB). However, when `qvm` requests memory from the host's process manager (`procnto`), the allocated host physical memory **may be scattered** across different physical locations. This doesn't matter because the **MMU's Stage 2 page tables** create the mapping that makes it appear contiguous to the guest.

</details>

---

**Q17: How does the QNX Hypervisor ensure memory isolation between guests?**

<details>
<summary>Answer</summary>

Memory isolation is enforced by the **MMU**. Each guest's `qvm` process programs the Stage 2 / Extended Page Tables to map only the memory that belongs to that guest. If a guest tries to access an address outside of its mapped range, the MMU will **fault** — just as it would in a normal OS when a process accesses memory it doesn't own. The guest cannot see or access another guest's memory or the host's memory (unless explicitly configured via passthrough).

</details>

---

**Q18: Why is memory allocated via the `ram` option zero-filled, and what are the performance implications?**

<details>
<summary>Answer</summary>

When `qvm` allocates memory via the `ram` configuration option, it requests memory from the QNX process manager, which **zero-fills** all allocated memory. This is a security/safety measure to prevent data leakage from previously used memory. However, zero-filling large amounts of memory (e.g., 512MB) **takes time** and can slow down guest boot.

**Workaround:** Use **typed memory** on the host to reserve RAM, then use the `pass` option to map it into the guest address space. Memory mapped via `pass` is **not zero-filled**, resulting in faster boot times. (Of course, you must ensure the memory content is acceptable for the guest.)

</details>

---

**Q19: What limits the amount of memory you can assign to a guest?**

<details>
<summary>Answer</summary>

Guest memory is limited by the **total physical memory available on the host board**. You must account for:
- Memory used by the **host QNX OS** itself (`procnto`, drivers, etc.)
- Memory used by **other guest VMs**
- Memory used by the **`qvm` processes** themselves
- Any other host processes consuming memory

The `ram` option in the configuration file lets you specify how much memory a guest gets, but the total across all guests and the host cannot exceed the physical RAM on the board.

</details>

---

### Instruction Execution & Trapping

---

**Q20: How does the QNX Hypervisor handle guest instructions? What gets trapped?**

<details>
<summary>Answer</summary>

There are three categories of guest instruction execution:

1. **Most instructions** execute **directly on the physical CPU** at bare-metal speed — no hypervisor involvement.

2. **Some instructions** execute on the CPU but the CPU does a **small amount of extra work** for virtualization support (e.g., checking permissions, translating through an extra page table layer). The overhead is **negligible**.

3. **A few instructions** are **trapped** — execution is paused, control is transferred to `qvm`, which emulates the expected behavior. Examples:
   - Accessing certain memory-mapped I/O addresses (virtual device registers)
   - Privileged instructions like **SMC** (Secure Monitor Call) on ARM
   - Instructions that would affect the physical hardware state

The key insight is that the vast majority of instructions run at full speed, which is why hypervisors have near-native performance.

</details>

---

**Q21: What is para-virtualization and how does it differ from full virtualization?**

<details>
<summary>Answer</summary>

- **Full virtualization:** The guest OS believes it's running on real hardware. It has **no knowledge** of the hypervisor. All sensitive operations are transparently trapped and emulated.

- **Para-virtualization:** The guest OS is **aware** that it's running on a hypervisor. It is modified to cooperate with the hypervisor, making explicit calls (hypercalls) instead of executing instructions that would need to be trapped. This can be **more efficient** because it avoids the overhead of trapping and emulating certain operations.

In the QNX Hypervisor context, most guests run with full virtualization (they don't know about the hypervisor), but para-virtualization is available as an option for better performance in certain scenarios.

</details>

---

### Unity Guests

---

**Q22: What is a Unity guest and when would you use one?**

<details>
<summary>Answer</summary>

A **Unity guest** has a **1:1 (identity) mapping** between guest physical addresses and host physical addresses — meaning GPA 0x10000000 maps directly to HPA 0x10000000.

The MMU is still involved (page tables must still be programmed), but the addresses are the same on both sides.

**When to use it:**
- When the hardware **lacks an IOMMU** (I/O Memory Management Unit). Without an IOMMU, DMA-capable devices use host physical addresses directly. If the guest physical addresses don't match host physical addresses, DMA operations would go to the wrong memory locations. A Unity guest ensures addresses match.

**Caveats:**
- Complex to configure
- Not the normal case — typically guest and host physical addresses differ
- Usually requires assistance from QNX engineering

</details>

---

**Q23: What is an IOMMU and why is it relevant to hypervisor memory management?**

<details>
<summary>Answer</summary>

An **IOMMU** (I/O Memory Management Unit) is a hardware component that provides address translation for **DMA (Direct Memory Access)** operations performed by I/O devices. It translates device-visible addresses to actual physical addresses.

**Relevance to hypervisors:**
- Without an IOMMU, DMA devices use **host physical addresses** directly. If a guest programs a DMA device with a guest physical address (which differs from the host physical address), the DMA will access the **wrong memory** — potentially corrupting other guests' or the host's memory.
- **With an IOMMU:** DMA addresses are translated, so the guest can safely program DMA devices with guest physical addresses.
- **Without an IOMMU:** You may need a **Unity guest** (1:1 address mapping) so that guest physical addresses equal host physical addresses, making DMA safe.

</details>

---

### Scenario-Based Questions

---

**Q24: You have an 8-core ARM board with 4GB of RAM. You need to run a QNX safety-critical guest, a Linux infotainment guest, and the host QNX. How would you plan the resource allocation?**

<details>
<summary>Answer</summary>

**CPU allocation:**
- Host QNX: Needs at least 1-2 cores for `procnto`, drivers, `qvm` processes, and other host services
- QNX safety-critical guest: 2 cores (for real-time performance and redundancy)
- Linux infotainment guest: 3-4 cores (infotainment workloads tend to be heavier)
- Exact allocation depends on workload profiling

**Memory allocation (4GB total):**
- Host QNX: ~512MB (for `procnto`, drivers, `qvm` overhead, networking, etc.)
- QNX safety-critical guest: ~512MB (QNX typically needs less RAM)
- Linux infotainment guest: ~2.5GB (graphics, multimedia, Android automotive workloads are memory-hungry)
- Reserve ~512MB buffer for overhead and dynamic allocation

**Configuration approach:**
- Each guest gets its own `.qvmconf` file
- Use `cpu` option to assign cores
- Use `ram` option for memory (or `pass` with typed memory for faster boot on the safety guest)
- Use `vdev` for virtual devices; use `pass` for GPU/display memory passthrough to the infotainment guest
- Consider IOMMU availability for safe DMA handling

</details>

---

**Q25: A guest is trying to access a memory address and gets a fault. What could be the possible causes in a hypervisor environment?**

<details>
<summary>Answer</summary>

Possible causes:

1. **Guest virtual address not mapped (Stage 1 fault):** The guest OS hasn't mapped that virtual address in its own page tables — this is a normal page fault handled by the guest OS itself.

2. **Guest physical address not mapped (Stage 2 fault):** The guest is trying to access a guest physical address that `qvm` hasn't mapped in the Stage 2 / Extended Page Tables. The guest is trying to access memory outside its allocated range — the MMU blocks it.

3. **Permission violation:** The MMU mapping exists but the access type is wrong (e.g., writing to read-only memory, executing from non-executable memory).

4. **Intentional trap:** The address belongs to a **virtual device (vdev)**. The access is deliberately trapped so `qvm` can emulate the device behavior and return the expected result.

5. **Misconfigured passthrough:** A `pass` mapping might be missing or incorrect, so a device memory access that should work fails.

6. **Attempting to access another guest's memory:** The MMU will fault because there's no mapping to that address in the current guest's Stage 2 tables.

</details>

---

**Q26: How would you optimize guest boot time on the QNX Hypervisor?**

<details>
<summary>Answer</summary>

Strategies to optimize guest boot time:

1. **Use `pass` with typed memory instead of `ram`:** Memory allocated via `ram` is zero-filled, which is slow for large allocations. By reserving memory using typed memory on the host and mapping it via `pass`, you avoid zero-fill overhead.

2. **Minimize guest image size:** Use stripped-down kernels and minimal ramdisk images. For QNX, use a minimal `mkifs` build file. For Linux, use a minimal Yocto configuration.

3. **Reduce virtual device count:** Only configure the `vdev` entries that are actually needed. Each virtual device adds startup overhead.

4. **Optimize CPU allocation:** Give the guest enough CPUs to parallelize its boot process.

5. **Use passthrough for critical devices:** Instead of virtual devices that require trapping and emulation, pass through hardware directly where possible for faster initialization.

6. **Pre-load guest images:** If possible, keep guest images in fast storage or pre-loaded memory regions.

</details>

---

**Q27: From the host QNX command line, how can you tell which guests are currently running?**

<details>
<summary>Answer</summary>

Since each guest is managed by a **`qvm` process** (which is a normal QNX process), you can use standard QNX process listing commands:

```bash
pidin | grep qvm
```

or

```bash
pidin -p qvm
```

Each `qvm` process corresponds to one running guest. You'll see one `qvm` process per active guest. The process list will show the PID, memory usage, and other standard process information for each `qvm` instance.

</details>

---

**Q28: What is QAVF and what problem does it solve?**

<details>
<summary>Answer</summary>

**QAVF** (QNX Advanced Virtualization Framework) is a separate product that provides **shared I/O resources** between multiple guests running on the QNX Hypervisor. It solves the problem of how multiple guests can share:

- **Graphics/GPU** — display sharing between guests (e.g., instrument cluster + infotainment)
- **Audio** — audio routing between guests
- **Touch screens** — touch input routing
- **Virtual sockets** — inter-guest communication
- **File systems** — shared file access

Without QAVF, guests are isolated and cannot easily share these types of I/O resources. QAVF is particularly important for **digital cockpit** scenarios in automotive where multiple guests need to share a single display, audio system, and touch interface.

</details>

---

**Q29: Explain what happens step-by-step when you execute `qvm /data/guests/linux/linux.qvmconf`.**

<details>
<summary>Answer</summary>

Step-by-step:

1. **Process creation:** QNX starts a new `qvm` process (a normal QNX user-space process).

2. **Parse configuration:** `qvm` reads and parses `linux.qvmconf`, extracting all VM parameters (RAM, CPUs, images, vdevs, passthrough mappings, etc.).

3. **Memory allocation:** `qvm` requests memory from `procnto`'s memory manager based on the `ram` option (e.g., 400MB). The returned memory is **zero-filled**.

4. **MMU programming:** `qvm` programs the **Stage 2 page tables** (ARM) or **Extended Page Tables** (x86) in the MMU to create the guest-physical → host-physical address mapping.

5. **Passthrough setup:** For any `pass` entries, `qvm` creates direct mappings from host physical addresses to guest physical addresses in the Stage 2 tables.

6. **Image loading:** `qvm` loads the guest kernel image and ramdisk image into the allocated guest memory at the appropriate addresses.

7. **Virtual device initialization:** `qvm` sets up all configured virtual devices (`vdev` entries), preparing trap handlers for their address ranges.

8. **vCPU setup:** `qvm` configures the specified number of virtual CPUs, sets up their initial register state, and points the instruction pointer to the kernel entry point.

9. **Guest execution begins:** `qvm` starts executing the guest. The CPU enters guest mode, and the Linux kernel begins its normal boot sequence — it doesn't know it's virtualized.

10. **Runtime management:** `qvm` continues running, handling any trapped instructions, virtual device accesses, and other hypervisor events as the guest runs.

</details>

---

**Q30: Compare how the QNX Hypervisor handles memory mapping on ARM vs. x86 architectures.**

<details>
<summary>Answer</summary>

| Aspect | ARM (ARMv8/AArch64) | x86 (x86-64) |
|---|---|---|
| **Guest → Host translation** | **Stage 2 page tables** | **Extended Page Tables (EPT)** |
| **Guest → Guest-physical** | Stage 1 page tables (set up by guest OS) | Normal page tables (set up by guest OS) |
| **Guest physical address term** | Intermediate Physical Address (IPA) | Guest Physical Address (GPA) |
| **Host physical address term** | Physical Address (PA) | Host Physical Address (HPA) |
| **Hardware support** | ARM Virtualization Extensions (VHE/non-VHE) | Intel VT-x / AMD-V |
| **RAM layout quirks** | Typically contiguous | May have gaps (ROM between RAM regions due to legacy PC architecture) |

Functionally, both work the same way — two layers of address translation in the MMU, set up by `qvm` (Stage 2/EPT) and the guest OS (Stage 1/normal). The QNX documentation uses x86 terminology (GPA/HPA) for clarity regardless of platform.

</details>

---

*These notes and interview questions are based on the QNX Hypervisor Architecture lesson. Topics like para-virtualization, VirtIO, IOMMUs, and safety considerations are covered in subsequent lessons.*

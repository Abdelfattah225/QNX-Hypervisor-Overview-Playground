

# QNX Hypervisor - Configuring Devices for Guests

## Table of Contents

- [Overview: Three Types of Devices](#overview-three-types-of-devices)
- [Emulated Devices (Emulation vdevs)](#emulated-devices-emulation-vdevs)
- [Para-Virtualized Devices (VirtIO vdevs)](#para-virtualized-devices-virtio-vdevs)
- [Pass-Through Devices](#pass-through-devices)
- [Front-End and Back-End Terminology](#front-end-and-back-end-terminology)
- [Quick Reference Summary](#quick-reference-summary)
- [Interview Questions & Answers](#interview-questions--answers)

---

## Overview: Three Types of Devices

When configuring devices for a guest in the QNX Hypervisor, there are **three types**:

| Type | Category | Guest Driver Behavior | Guest Awareness |
|---|---|---|---|
| **Emulated** | vdev (virtual device) | Normal I/O | **Unaware** of hypervisor |
| **Para-Virtualized** | vdev (virtual device) | VirtIO API | **Aware** of hypervisor |
| **Pass-Through** | Direct hardware access | Normal I/O directly to hardware | N/A — bypasses hypervisor |

> **Terminology:** Both emulated and para-virtualized devices are called **vdevs** (virtual devices). The term "vdev" will be used extensively.

```
┌─────────────────────────────────────────────────────┐
│                   Device Types                       │
│                                                     │
│   ┌──────────────────────────────────┐              │
│   │        vdevs (Virtual Devices)   │              │
│   │                                  │              │
│   │  ┌────────────┐ ┌─────────────┐  │  ┌────────┐ │
│   │  │ Emulated   │ │Para-Virtual.│  │  │Pass-   │ │
│   │  │ (Normal IO)│ │ (VirtIO)    │  │  │Through │ │
│   │  └────────────┘ └─────────────┘  │  └────────┘ │
│   └──────────────────────────────────┘              │
└─────────────────────────────────────────────────────┘
```

---

## Emulated Devices (Emulation vdevs)

### How They Work

Emulated devices present **virtual hardware** to the guest that is emulated in software. The guest driver performs **normal I/O** — it has no idea there's a hypervisor underneath.

### Mechanism Step-by-Step

1. In the `.qvmconf` file, you specify a `vdev` entry (e.g., `vdev wdt-sp805`)
2. `qvm` constructs a filename from this: **`vdev-wdt-sp805.so`**
3. `qvm` calls `dlopen()` to load this shared object (DLL) into its memory
4. `qvm` reads the `loc` option (address) and programs it into the **virtualization hardware**
5. The virtualization hardware is now watching for any guest access to that address
6. When the guest driver accesses that address (normal I/O):
   - The **virtualization hardware traps** the access
   - The **guest is stopped**
   - **`qvm` is notified**
   - `qvm` **calls into the loaded `.so` code** to emulate the operation
   - The emulated result is returned
   - **Guest execution resumes**

### Architecture Diagram

```
┌─────────────────────────────────────────────┐
│                  GUEST                        │
│  ┌──────────────────────────┐                │
│  │  Normal Driver           │                │
│  │  (does normal I/O)       │                │
│  │  Unaware of hypervisor   │                │
│  └────────────┬─────────────┘                │
└───────────────┼──────────────────────────────┘
                │ Access to device address → TRAPPED
                ▼
┌─────────────────────────────────────────────┐
│                  HOST (qvm)                  │
│                                              │
│  ┌──────────────────────┐                    │
│  │  vdev-*.so           │──── (optional) ──→ │ QNX Driver → Actual HW
│  │  (emulation code)    │    IPC              │
│  │  loaded via dlopen() │                    │
│  └──────────────────────┘                    │
└─────────────────────────────────────────────┘
```

### Two Variants

**Variant 1: Pure emulation (no real hardware)**
- The `.so` code emulates the device entirely in software
- No actual hardware underneath

**Variant 2: Emulation backed by real hardware**
- The `.so` code performs **IPC (inter-process communication)** with a normal QNX driver on the host
- That host driver talks to the actual physical hardware
- The guest driver still does normal I/O — it doesn't know about the intermediary

### Built-in vdevs

Some emulation vdevs are **statically linked into `qvm`** — no separate `.so` file needed:

- **Interrupt controller chip** vdevs
- **Timer** vdevs

You still configure them in the `.qvmconf` file (to set interrupt numbers, addresses, etc.), but you don't need to supply a `.so` file.

### Configuration Example

```
# In .qvmconf file:
vdev wdt-sp805
  loc 0x1C0F0000        # Address the virtualization HW watches
  intr gic:32           # Interrupt configuration

# qvm constructs filename: vdev-wdt-sp805.so
# qvm does dlopen("vdev-wdt-sp805.so")
# qvm programs 0x1C0F0000 into virtualization hardware
```

---

## Para-Virtualized Devices (VirtIO vdevs)

### How They Work

Para-virtualized devices use the **VirtIO API** instead of normal I/O. The guest driver is **aware** that it's running on a hypervisor and uses a special communication protocol.

### What is VirtIO?

- An API/standard created by the **Linux community**
- Has a **standards committee** — QNX closely follows the standard
- Uses **virt queues** (virtual queues) for data transfer between guest and host
- Not normal I/O — it's a specialized, efficient communication protocol designed for virtualized environments

### Mechanism Step-by-Step

1. In the `.qvmconf` file, specify a VirtIO vdev (e.g., `vdev virtio-blk`)
2. `qvm` constructs filename: **`vdev-virtio-blk.so`**
3. `qvm` calls `dlopen()` to load the shared object
4. The guest driver uses the **VirtIO API** (writes to virt queues, reads from virt queues)
5. The virtualization hardware **traps** the VirtIO operations
6. **Guest is stopped**, `qvm` is notified
7. `qvm` calls into the loaded `.so` code to handle the VirtIO request
8. The `.so` code may perform **IPC** with a host driver that talks to actual hardware
9. Guest execution resumes

### Architecture Diagram

```
┌─────────────────────────────────────────────┐
│                  GUEST                        │
│  ┌──────────────────────────┐                │
│  │  VirtIO Driver           │                │
│  │  (uses VirtIO API)       │                │
│  │  AWARE of hypervisor     │                │
│  └────────────┬─────────────┘                │
└───────────────┼──────────────────────────────┘
                │ VirtIO operations → TRAPPED
                ▼
┌─────────────────────────────────────────────┐
│                  HOST (qvm)                  │
│                                              │
│  ┌──────────────────────────┐                │
│  │  vdev-virtio-*.so        │── (optional) → │ QNX Driver → Actual HW
│  │  (handles VirtIO API)    │   IPC          │
│  │  (virt queues, etc.)     │                │
│  └──────────────────────────┘                │
└─────────────────────────────────────────────┘
```

### Configuration Example

```
# In .qvmconf file:
vdev virtio-blk
  # Additional options...

# qvm constructs filename: vdev-virtio-blk.so
# qvm does dlopen("vdev-virtio-blk.so")
```

### Why "Para-Virtualized"?

- **"Para"** comes from Greek meaning **"adjacent to"**
- The driver logic is split: part is in the **guest** (VirtIO driver) and part is in the **host** (the `.so` backend)
- They are "adjacent to" each other, working together across the hypervisor boundary

### Emulated vs. Para-Virtualized Comparison

| Aspect | Emulated vdev | Para-Virtualized vdev |
|---|---|---|
| Guest driver type | Normal driver, normal I/O | Special VirtIO driver |
| Guest awareness | **Unaware** of hypervisor | **Aware** of hypervisor |
| `.so` naming | `vdev-*.so` | `vdev-virtio-*.so` (by convention) |
| Trap mechanism | Address-based trap | VirtIO operation trap |
| Communication | Normal I/O emulation | VirtIO API (virt queues) |
| Performance | May be slower (full emulation) | Can be more efficient (designed for virtualization) |
| Driver availability | Use existing/standard drivers | Need VirtIO-compatible drivers |

---

## Pass-Through Devices

### How They Work

Pass-through devices allow the guest driver to access **actual physical hardware directly**, bypassing the hypervisor entirely.

### Mechanism

- Creates a **mapping in the MMU's page tables** from host physical address to guest physical address
- The guest driver accesses the device memory directly — no trapping, no emulation
- The guest driver has **exclusive access** to that hardware

### Architecture Diagram

```
┌─────────────────────────────────────────────┐
│                  GUEST                        │
│  ┌──────────────────────────┐                │
│  │  Normal Driver           │                │
│  │  (talks directly to HW)  │                │
│  └────────────┬─────────────┘                │
└───────────────┼──────────────────────────────┘
                │ DIRECT ACCESS (no trapping!)
                │ MMU maps guest addr → host addr
                ▼
┌─────────────────────────────────────────────┐
│           ACTUAL HARDWARE                    │
│  (video memory, device registers, etc.)      │
└─────────────────────────────────────────────┘

  ← Hypervisor is BYPASSED →
```

### Configuration

#### Memory Pass-Through

```
# In .qvmconf file:
pass 0xFE000000,0x3F000000,0x01000000
#    host_phys   guest_phys  size

# Creates MMU mapping:
#   Guest sees device at 0x3F000000
#   Actually located at 0xFE000000 on real hardware
#   Size: 16MB (0x01000000)
```

#### Interrupt Pass-Through

```
# You can also pass through interrupts:
pass intr gic:48
```

> **Important Note on Interrupt Pass-Through:** Despite being called "pass-through" (because they use the `pass` option), interrupts are **NOT truly passed through**. There is processing in between — the host routes interrupts to the guest. It's called pass-through only because of the configuration syntax. See the Interrupts lesson for details.

### Exclusive Access Rule

| Rule | Detail |
|---|---|
| **Only one guest** should access the hardware at a time | Not enforced by the hypervisor — it's your responsibility |
| **The host shouldn't touch it either** | Nothing prevents it, but you shouldn't do it |
| **If sharing is needed** | Use an intermediary process for synchronization (covered in Shared Devices lesson) |

> **Warning:** Nothing in the hypervisor **prevents** multiple guests or the host from mapping the same device memory. It's a configuration discipline issue. If two things access the same hardware without synchronization, behavior will be unpredictable.

### DMA and IOMMUs

When using pass-through with DMA-capable devices:
- An **IOMMU** provides address translation for DMA operations
- An **IOMPU** provides protection only (no translation)
- See the **Safety lesson** for details on IOMMUs

---

## Front-End and Back-End Terminology

This terminology appears in QNX documentation:

| Term | Meaning | Location |
|---|---|---|
| **Front-End** | The driver running in the **guest** | Guest side |
| **Back-End** | Everything else needed to support that driver | Host side (`.so` code, host drivers, etc.) |

### Examples

**Emulation vdev with real hardware:**
```
Front-End: Guest driver (normal I/O)
Back-End:  vdev-*.so (in qvm) + Host QNX driver + Actual hardware
           └── Two components ──┘
```

**Emulation vdev without real hardware:**
```
Front-End: Guest driver (normal I/O)
Back-End:  vdev-*.so (in qvm)
           └── One component ──┘
```

**Para-virtualized vdev:**
```
Front-End: Guest VirtIO driver
Back-End:  vdev-virtio-*.so (in qvm) + possibly Host QNX driver
```

---

## Quick Reference Summary

```
┌────────────────────────────────────────────────────────────────────┐
│                DEVICE CONFIGURATION SUMMARY                        │
│                                                                    │
│  EMULATED VDEV                                                     │
│  • Guest driver does normal I/O                                    │
│  • Guest is UNAWARE of hypervisor                                  │
│  • Access trapped → qvm calls vdev-*.so to emulate                 │
│  • Some vdevs built into qvm (interrupt controllers, timers)       │
│                                                                    │
│  PARA-VIRTUALIZED VDEV                                             │
│  • Guest driver uses VirtIO API                                    │
│  • Guest is AWARE of hypervisor                                    │
│  • Access trapped → qvm calls vdev-virtio-*.so to handle           │
│  • VirtIO standard from Linux community                            │
│                                                                    │
│  PASS-THROUGH                                                      │
│  • Guest driver accesses hardware DIRECTLY                         │
│  • MMU mapping: host physical → guest physical                     │
│  • Hypervisor is BYPASSED                                          │
│  • Only one guest should access at a time (not enforced)           │
│  • Interrupt "pass-through" is NOT truly pass-through              │
└────────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions & Answers

### Fundamentals

---

**Q1: What are the three types of devices you can configure for a guest in the QNX Hypervisor?**

<details>
<summary>Answer</summary>

1. **Emulated devices (emulation vdevs):** Hardware is emulated in software. The guest driver does normal I/O and is **unaware** of the hypervisor. Access to device addresses is trapped by the virtualization hardware, and `qvm` calls into a loaded `.so` shared object to emulate the operation.

2. **Para-virtualized devices (VirtIO vdevs):** The guest driver uses the **VirtIO API** instead of normal I/O. The guest is **aware** it's running on a hypervisor. VirtIO operations are trapped and handled by `qvm` through a `vdev-virtio-*.so` shared object.

3. **Pass-through devices:** The guest driver accesses **actual physical hardware directly**, bypassing the hypervisor. A mapping is created in the MMU page tables from host physical address to guest physical address. The guest should have **exclusive access** to the hardware.

</details>

---

**Q2: What is a vdev? What are the two types?**

<details>
<summary>Answer</summary>

A **vdev** (virtual device) is a virtualized hardware device presented to a guest. There are two types:

1. **Emulated vdev:** The guest driver does normal I/O, which is trapped and emulated by `qvm` using a loaded `.so` shared object. The guest is unaware of the hypervisor.

2. **Para-virtualized vdev:** The guest driver uses the VirtIO API. The guest knows it's on a hypervisor. VirtIO operations are trapped and handled by `qvm` through a VirtIO-specific `.so` file.

Both types involve trapping and processing by `qvm`, but they differ in the **guest driver's behavior** (normal I/O vs. VirtIO) and **awareness** of the hypervisor.

</details>

---

### Emulated Devices

---

**Q3: Walk through what happens when a guest driver accesses an emulated device address.**

<details>
<summary>Answer</summary>

Step-by-step:

1. During `qvm` startup, it reads the `.qvmconf` file and sees a `vdev` entry (e.g., `vdev wdt-sp805`)
2. `qvm` constructs the filename `vdev-wdt-sp805.so` and loads it via `dlopen()`
3. `qvm` reads the `loc` option (the device address) and programs it into the **virtualization hardware**
4. The virtualization hardware is now **watching** for any guest access to that address
5. When the guest driver (doing normal I/O) accesses that address:
   - The **virtualization hardware traps** the access
   - The **guest is stopped** (paused)
   - **`qvm` is notified** of the trap
   - `qvm` **calls into the `.so` code** to emulate the instruction
   - The `.so` code may optionally do **IPC** with a host driver that talks to real hardware
   - The emulated result is produced
   - **Guest execution resumes**

The guest driver never knows its I/O was intercepted and emulated.

</details>

---

**Q4: How does `qvm` know which shared object to load for a vdev?**

<details>
<summary>Answer</summary>

`qvm` constructs the filename from the `vdev` line in the configuration file:

- Config line: `vdev wdt-sp805` → filename: **`vdev-wdt-sp805.so`**
- Config line: `vdev virtio-blk` → filename: **`vdev-virtio-blk.so`**

The pattern is: `vdev-<name>.so`

`qvm` then calls `dlopen()` to load this shared object (DLL) into its process memory. Once loaded, `qvm` calls into this code whenever the guest triggers a trap related to that virtual device.

</details>

---

**Q5: Which vdevs are built into `qvm` and don't need a separate `.so` file?**

<details>
<summary>Answer</summary>

Some vdevs are **statically linked** into `qvm`'s code:

- **Interrupt controller chip** vdevs
- **Timer** vdevs

These don't require a separate `.so` file to be supplied. However, you still **configure them** in the `.qvmconf` file because you may need to set parameters like interrupt numbers, addresses, and other settings. You just don't need to worry about providing the `.so` file — it's already built into `qvm`.

</details>

---

### Para-Virtualized Devices

---

**Q6: What is VirtIO and where does it come from?**

<details>
<summary>Answer</summary>

**VirtIO** is a standardized API for virtualized I/O, created by the **Linux community**. Key points:

- It provides an efficient communication protocol between guest drivers and the hypervisor
- It uses **virt queues** (virtual queues) for data transfer
- There is a **standards committee** that maintains and evolves the specification
- QNX closely follows the VirtIO standard and keeps up with its versions
- It replaces normal I/O with a protocol specifically designed for virtualized environments
- VirtIO drivers are widely available in Linux kernels and are also implemented for QNX

The name "VirtIO" stands for **Virtual I/O**.

</details>

---

**Q7: Why would you choose a para-virtualized device over an emulated device?**

<details>
<summary>Answer</summary>

**Performance and efficiency.** 

With an **emulated device**, every single I/O operation by the guest driver must be trapped and the entire hardware behavior must be emulated instruction by instruction. This can involve many traps for a single logical I/O operation (e.g., writing to multiple registers to set up a DMA transfer).

With a **para-virtualized (VirtIO) device**, the guest driver is designed to communicate efficiently with the hypervisor. Instead of emulating individual hardware register accesses, VirtIO uses **virt queues** to batch and transfer data. This can result in:

- **Fewer traps** per I/O operation
- **Higher throughput** for data-intensive operations (block devices, networking)
- **Lower overhead** because the protocol is designed for virtualization

The tradeoff is that you need a **VirtIO-compatible driver** in the guest, whereas emulated devices work with existing standard drivers.

</details>

---

**Q8: What does "para" mean in para-virtualization?**

<details>
<summary>Answer</summary>

"Para" comes from the **Greek word meaning "adjacent to."** The concept is that the driver logic is split across two sides:

- **Guest side:** The VirtIO driver in the guest
- **Host side:** The back-end `.so` code in `qvm` (and possibly a host driver)

These two parts are "adjacent to" each other — working together across the hypervisor boundary. The term dates back to when hypervisor concepts were first developed. While the etymology doesn't perfectly describe modern usage (emulated vdevs also have a similar split), the key distinction is that the **guest driver is aware** of the hypervisor and uses the VirtIO API.

</details>

---

**Q9: How does a para-virtualized vdev differ from an emulated vdev in terms of the trapping mechanism?**

<details>
<summary>Answer</summary>

Both types involve trapping, but what triggers the trap differs:

**Emulated vdev:**
- `qvm` programs a **specific address** into the virtualization hardware
- When the guest driver accesses that **address** (normal I/O — port I/O, memory-mapped I/O), the trap occurs
- The trap is **address-based**

**Para-virtualized vdev:**
- The guest driver uses the **VirtIO API** (writes to virt queues, signals the hypervisor)
- VirtIO operations trigger the trap
- The trap is based on **VirtIO protocol operations** rather than raw address access
- The back-end `.so` code handles VirtIO protocol semantics (queue management, data transfer)

In both cases, the guest is stopped, `qvm` is notified, and the `.so` code handles the operation. The difference is in what the guest driver does (normal I/O vs. VirtIO API) and how the back-end interprets it.

</details>

---

### Pass-Through Devices

---

**Q10: How does pass-through work for memory-mapped devices?**

<details>
<summary>Answer</summary>

Pass-through creates a **direct mapping in the MMU's page tables** from a host physical address to a guest physical address:

```
pass <host_physical_addr>,<guest_physical_addr>,<size>
```

For example: `pass 0xFE000000,0x3F000000,0x01000000`

- The device exists at host physical address `0xFE000000`
- The guest sees it at guest physical address `0x3F000000`
- 16MB of address space is mapped

When the guest driver accesses `0x3F000000`, the MMU translates it directly to `0xFE000000` on the real hardware. **No trapping occurs**, **no emulation**, and **`qvm` is not involved** in the data path. The guest accesses the hardware at near bare-metal speed.

This is used for devices where performance is critical (e.g., GPU/video memory, high-speed I/O).

</details>

---

**Q11: Why should only one guest have pass-through access to a device at a time?**

<details>
<summary>Answer</summary>

Because with pass-through, the guest driver talks **directly** to the hardware without any intermediary. If two guests (or a guest and the host) both access the same hardware simultaneously:

- There is **no synchronization** between them
- They could issue **conflicting commands** to the device
- Device state could become **corrupted**
- Data could be **lost or garbled**

**Important:** The hypervisor does **NOT enforce** exclusive access. Nothing prevents you from configuring the same pass-through in multiple guests or having the host also access the hardware. It's a **configuration discipline** responsibility.

If you need multiple guests to share a device, you should use an **intermediary process** that handles synchronization — this is covered in the Shared Devices lesson.

</details>

---

**Q12: Are pass-through interrupts truly "passed through" to the guest?**

<details>
<summary>Answer</summary>

**No.** Despite being called "pass-through interrupts" (because they are configured using the `pass` option in the `.qvmconf` file), interrupts are **NOT truly passed through**. 

What actually happens:
1. The hardware interrupt fires
2. The **host** receives and processes the interrupt first
3. The host **routes** the interrupt to the appropriate guest
4. The guest then handles it

There is processing and routing done by the host in between. The term "pass-through interrupt" is a misnomer based on the configuration syntax (`pass intr ...`), not on the actual behavior. The Interrupts lesson covers this in detail.

</details>

---

**Q13: What is the role of an IOMMU in the context of pass-through devices?**

<details>
<summary>Answer</summary>

When a pass-through device performs **DMA (Direct Memory Access)**, the device uses physical addresses to read/write memory. Without proper translation or protection:

- The device might access **any physical memory** on the host, including other guests' memory or the host's memory
- This is a **security and safety risk**

An **IOMMU** (I/O Memory Management Unit) solves this by:
- **Translating** device DMA addresses (guest physical) to actual host physical addresses
- **Restricting** which memory regions the device can access
- **Protecting** other guests and the host from errant DMA operations

An **IOMPU** provides only the **protection** aspect (no translation).

Without an IOMMU/IOMPU, you may need to use a **Unity guest** (1:1 address mapping) so that the addresses the device uses match the actual physical addresses.

</details>

---

### Comparison & Design Questions

---

**Q14: When would you choose each device type (emulated, para-virtualized, pass-through) for a guest?**

<details>
<summary>Answer</summary>

| Device Type | Choose When... |
|---|---|
| **Emulated** | You want to use **existing, unmodified drivers** in the guest. The guest OS doesn't need any changes. Good for simple devices (serial ports, watchdog timers). Performance is acceptable for low-bandwidth devices. |
| **Para-Virtualized (VirtIO)** | You need **better performance** for data-intensive I/O (block storage, networking). You can use/install VirtIO drivers in the guest. Most Linux kernels already include VirtIO drivers. Good for high-throughput use cases. |
| **Pass-Through** | You need **maximum performance** with direct hardware access (GPU, high-speed networking). Only **one guest** needs the device. You have IOMMU support for safe DMA. The device doesn't need to be shared. |

**Typical automotive digital cockpit example:**
- **Instrument cluster (QNX guest):** Pass-through for the display controller (safety-critical, needs direct access)
- **Infotainment (Linux guest):** VirtIO for block storage and networking (high throughput needed), pass-through for GPU
- **Simple serial console:** Emulated vdev (low bandwidth, simplicity)

</details>

---

**Q15: A guest needs to access a network interface. Compare implementing this as an emulated device, a VirtIO device, and a pass-through device.**

<details>
<summary>Answer</summary>

**Emulated NIC:**
- Guest runs a **standard NIC driver** (e.g., for an emulated Intel e1000)
- Every register access is **trapped and emulated** by `qvm` via a `vdev-*.so`
- The `.so` may do IPC with a host NIC driver to send/receive actual packets
- **Performance:** Lowest — many traps per packet (driver touches many registers)
- **Compatibility:** Highest — any OS with an e1000 driver works unmodified
- **Use case:** Low-traffic control plane networking

**VirtIO NIC:**
- Guest runs a **VirtIO network driver** (`virtio-net`)
- Uses virt queues to batch packet data efficiently
- **Fewer traps** per packet compared to emulation
- The `.so` handles VirtIO protocol and may do IPC with host networking
- **Performance:** Good — designed for efficient virtualized I/O
- **Compatibility:** Need VirtIO driver (standard in Linux, available for QNX)
- **Use case:** General-purpose networking with good throughput

**Pass-Through NIC:**
- Guest driver accesses the **physical NIC directly**
- MMU mapping created for NIC registers and DMA regions
- **No hypervisor involvement** in the data path
- **Performance:** Best — bare-metal speed
- **Drawback:** Only one guest can use this NIC; need IOMMU for safe DMA
- **Use case:** High-performance networking where one guest needs dedicated access

</details>

---

**Q16: You're configuring a QNX Hypervisor system with two guests. Guest A needs a serial port for debug logging, and Guest B needs high-speed block storage. What device types would you recommend and why?**

<details>
<summary>Answer</summary>

**Guest A — Serial port for debug logging:**
- **Recommendation: Emulated vdev**
- Serial ports are **low bandwidth** — the performance overhead of trapping and emulation is negligible
- Use a standard serial emulation (e.g., `vdev pl011` for ARM or `vdev 8250` for x86)
- The guest can use its **standard serial driver** without modification
- Simple to configure and maintain

**Guest B — High-speed block storage:**
- **Recommendation: Para-virtualized (VirtIO) vdev** — specifically `vdev virtio-blk`
- Block storage involves **large data transfers** where emulation overhead would be significant
- VirtIO uses **virt queues** to batch I/O operations, reducing the number of traps
- VirtIO block drivers are **standard in Linux kernels** and available for QNX
- If even higher performance is needed and the guest can have exclusive access to the storage controller, consider **pass-through** — but VirtIO is usually sufficient and more flexible

</details>

---

**Q17: What is the difference between the front-end and back-end of a virtual device?**

<details>
<summary>Answer</summary>

- **Front-end:** The **driver running inside the guest**. For emulated vdevs, it's a normal driver doing normal I/O. For para-virtualized vdevs, it's a VirtIO driver.

- **Back-end:** **Everything else** needed to support that driver — the `.so` code loaded by `qvm`, any host-side processes, and the host driver that communicates with actual hardware (if applicable).

The back-end can have:
- **One component:** Just the `.so` code in `qvm` (pure emulation, no real hardware)
- **Two or more components:** The `.so` code in `qvm` + a host QNX driver that communicates with actual hardware via IPC

</details>

---

**Q18: Can you modify or create custom vdev shared objects? How does the plugin architecture work?**

<details>
<summary>Answer</summary>

Yes, the vdev architecture is **plugin-based**:

1. `qvm` reads the `vdev` line from the configuration file
2. It constructs a filename: `vdev-<name>.so`
3. It calls `dlopen()` to dynamically load the shared object
4. It uses defined entry points in the `.so` to handle trapped device accesses

This means you can:
- **Create custom vdevs** by implementing the required interface in a new `.so` file
- **Replace existing vdevs** with custom implementations
- **Add new virtual hardware** that doesn't correspond to any real device

The `.so` acts like a **DLL/plugin** — `qvm` loads it at runtime based on the configuration. This provides a flexible, modular architecture for device virtualization.

</details>

---

**Q19: In a scenario where two guests both need access to the same GPU, how would you handle this? Why can't you just use pass-through for both?**

<details>
<summary>Answer</summary>

You **cannot** simply use pass-through for both guests because:

- Pass-through gives **direct, unsynchronized** hardware access
- Two guests writing to the same GPU registers simultaneously would cause **conflicts and corruption**
- The hypervisor does **not enforce** exclusive access — it would silently allow both, but the GPU would malfunction

**Solutions:**

1. **Intermediary process:** Create a process (in the host or a dedicated guest) that manages GPU access. Both guests communicate with this intermediary, which serializes and coordinates GPU operations.

2. **QAVF (QNX Advanced Virtualization Framework):** This product specifically addresses shared graphics — it provides a framework for sharing GPU/display resources between multiple guests safely.

3. **Time-slicing:** One guest uses the GPU at a time, with software managing the switching (complex to implement).

4. **SR-IOV (if hardware supports it):** Some GPUs support Single Root I/O Virtualization, which presents multiple virtual functions that can each be passed through to different guests independently.

The Shared Devices lesson covers this topic in detail.

</details>

---

**Q20: Explain the complete data flow when a Linux guest writes data to a VirtIO block device that is backed by real storage hardware on the host.**

<details>
<summary>Answer</summary>

Complete data flow:

1. **Application in Linux guest** issues a `write()` system call
2. **Linux kernel (guest)** processes the write through its VFS and block layer
3. **VirtIO block driver (guest front-end)** places the write request into a **virt queue** (a shared memory ring buffer) and signals the hypervisor (e.g., writes to a specific register)
4. **Virtualization hardware traps** the signal/notification
5. **Guest is paused**; control transfers to `qvm`
6. **`qvm` invokes `vdev-virtio-blk.so`** (the back-end), which reads the request from the virt queue
7. **`vdev-virtio-blk.so` does IPC** (inter-process communication) with a **host QNX block device driver**
8. **Host QNX driver** sends the write command to the **actual storage hardware** (e.g., eMMC, SSD)
9. **Hardware completes the write** and signals completion
10. **Host QNX driver** notifies `vdev-virtio-blk.so` via IPC
11. **`vdev-virtio-blk.so`** updates the virt queue with the completion status
12. **`qvm` injects a virtual interrupt** into the guest
13. **Guest resumes**; the VirtIO driver processes the completion from the virt queue
14. **Linux kernel** propagates the completion back to the application

This involves two traps minimum (notification + completion interrupt) versus potentially dozens of traps for an emulated block device where every register access would be trapped individually.

</details>

---

*These notes are based on the QNX Hypervisor Configuring Devices lesson. Related topics (Shared Devices, Interrupts, Safety/IOMMUs, Running Guests) are covered in separate lessons.*

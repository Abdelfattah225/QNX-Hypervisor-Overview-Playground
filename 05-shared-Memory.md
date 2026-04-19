# QNX Hypervisor - Sharing Devices Between Guests and Host

## Table of Contents

- [Overview](#overview)
- [Non-Overlapping Memory Regions](#non-overlapping-memory-regions)
- [Shared Device via Controlling Driver](#shared-device-via-controlling-driver)
- [Shared Flash/Block Storage](#shared-flashblock-storage)
- [Complex Sharing (Vendor-Specific)](#complex-sharing-vendor-specific)
- [Quick Reference Summary](#quick-reference-summary)
- [Interview Questions & Answers](#interview-questions--answers)

---

## Overview

A **shared device** is hardware that is shared between:
- Multiple **guests**, or
- **Guests and the host**

Sharing devices is primarily a **configuration exercise** — using techniques already covered (pass-through, shared memory, networking, VirtIO vdevs).

---

## Non-Overlapping Memory Regions

The simplest case: two guests need the **same device** but access **different, non-overlapping regions** of its memory.

```
┌──────────────┐     ┌──────────────┐
│   Guest A     │     │   Guest B     │
│               │     │               │
│  Pass-through │     │  Pass-through │
│  driver       │     │  driver       │
└──────┬───────┘     └──────┬───────┘
       │                     │
       ▼                     ▼
┌──────────────────────────────────────┐
│          Device Memory                │
│                                      │
│  ┌──────────────┐ ┌──────────────┐   │
│  │ Region A     │ │ Region B     │   │
│  │ (Guest A)    │ │ (Guest B)    │   │
│  └──────────────┘ └──────────────┘   │
│      No overlap!                     │
└──────────────────────────────────────┘
```

### Configuration

- Use the **`pass`** option in each guest's `.qvmconf` to map different regions:

```
# Guest A .qvmconf:
pass 0xFE000000,0x3F000000,0x00800000   # Maps first half of device memory

# Guest B .qvmconf:
pass 0xFE800000,0x3F000000,0x00800000   # Maps second half of device memory
```

### Key Points

- **Simple** — just configuration using `pass` option
- Guests use **pass-through drivers** (direct hardware access)
- Memory regions must **not overlap**
- No intermediary or synchronization needed since regions are independent

---

## Shared Device via Controlling Driver

When multiple guests need to access the **same device** through **overlapping or shared resources**, use a **controlling driver** in the host.

```
┌──────────────┐   ┌──────────────┐   ┌──────────────┐
│   Guest A     │   │   Guest B     │   │     HOST      │
│               │   │               │   │               │
│  ┌──────────┐ │   │  ┌──────────┐ │   │  ┌──────────┐ │
│  │ Process  │ │   │  │ Process  │ │   │  │ Process  │ │
│  └────┬─────┘ │   │  └────┬─────┘ │   │  └────┬─────┘ │
└───────┼───────┘   └───────┼───────┘   └───────┼───────┘
        │                   │                   │
        │   Shared Memory + Socket Communication │
        │                   │                   │
        └───────────┬───────┴───────────────────┘
                    ▼
        ┌───────────────────────┐
        │  Controlling Driver   │  ← Single point of control
        │  (Host Process)       │     Everything goes through it
        └───────────┬───────────┘
                    ▼
        ┌───────────────────────┐
        │   Physical Device     │
        └───────────────────────┘
```

### How It Works

1. A **single driver** in the host has exclusive access to the device
2. Guest processes communicate with this driver using:
   - **Shared memory** — for large data transfers (megabytes)
   - **Socket communication** — for notifications and control messages ("data is ready," "read this," "write that")
3. The driver serializes access and manages the device

### Communication Flow Example

```
Guest A wants to write large data to device:
  1. Guest A writes data to shared memory
  2. Guest A sends socket message to driver: "New data at offset X, size Y"
  3. Driver reads data from shared memory
  4. Driver writes data to physical device

Guest B wants to read from device:
  1. Guest B sends socket message to driver: "Read data from device"
  2. Driver reads from device, writes to shared memory
  3. Driver sends socket notification to Guest B: "Data ready"
  4. Guest B reads data from shared memory
```

### Advantages

| Advantage | Detail |
|---|---|
| **Controlled access** | Single driver manages all device access — no conflicts |
| **Fault isolation** | Driver is just a host process. If it dies, it doesn't crash the host |
| **Monitoring** | Another host process can monitor the driver and handle failures |
| **Flexible communication** | Can use shared memory, sockets, or both depending on needs |

### Configuration

Uses techniques from the Communication lesson:
- `vdev shmem` in guest configs for shared memory
- `vdev virtio-net` for socket communication
- `vdevpeer-net` in host `io-sock` for guest-to-host networking

> **Note:** You don't have to use both shared memory and sockets. If data volumes are small, sockets alone may be sufficient.

---

## Shared Flash/Block Storage

Multiple guests need to access the **same flash/eMMC/SD card** storage.

```
┌──────────────────┐         ┌──────────────────┐
│    QNX Guest      │         │   Linux Guest     │
│                   │         │                   │
│  ┌──────────────┐ │         │  ┌──────────────┐ │
│  │ Process      │ │         │  │ Process      │ │
│  │ open/read/   │ │         │  │ open/read/   │ │
│  │ write files  │ │         │  │ write files  │ │
│  └──────┬───────┘ │         │  └──────┬───────┘ │
│         │         │         │         │         │
│  ┌──────┴───────┐ │         │  ┌──────┴───────┐ │
│  │ devb-virtio  │ │         │  │ Linux VirtIO │ │
│  │ driver       │ │         │  │ block driver │ │
│  └──────┬───────┘ │         │  └──────┬───────┘ │
│         │         │         │         │         │
│  ┌──────┴───────┐ │         │  ┌──────┴───────┐ │
│  │vdev virtio-  │ │         │  │vdev virtio-  │ │
│  │blk           │ │         │  │blk           │ │
│  └──────┬───────┘ │         │  └──────┴───────┘ │
└─────────┼─────────┘         └─────────┼─────────┘
          │                             │
          └──────────┬──────────────────┘
                     ▼
┌─────────────────────────────────────────────┐
│                   HOST                       │
│                                             │
│  ┌───────────────────────────────────────┐  │
│  │  devb-sdmmc driver                    │  │
│  │  (from board support package)         │  │
│  └──────────────────┬────────────────────┘  │
└─────────────────────┼───────────────────────┘
                      ▼
┌─────────────────────────────────────────────┐
│  Flash / eMMC / SD Card                      │
│                                             │
│  ┌──────────────┐  ┌──────────────┐         │
│  │ Partition 1  │  │ Partition 2  │         │
│  │ (QNX Guest)  │  │ (Linux Guest)│         │
│  └──────────────┘  └──────────────┘         │
└─────────────────────────────────────────────┘
```

### Configuration Steps

| Where | What |
|---|---|
| **Host** | Run flash driver (e.g., `devb-sdmmc` from BSP for Renesas R-Car) |
| **Each guest `.qvmconf`** | Add `vdev virtio-blk` |
| **QNX guest** | Run `devb-virtio` driver |
| **Linux guest** | Use Linux VirtIO block driver (included in Linux) |
| **Flash storage** | **Partition** the flash — one partition per guest |

### Critical Rule: Use Different Partitions

> **Block regions must be unique.** Each guest must write to its own partition to avoid data corruption.

- Partition the flash/eMMC before use
- Assign each guest a **different partition**
- Since partitions don't overlap, there's no danger of simultaneous writes to the same data

### After Configuration

Guest processes can use standard file I/O:
```c
int fd = open("/path/to/file", O_RDWR);
read(fd, buf, size);
write(fd, buf, size);
close(fd);
```

---

## Complex Sharing (Vendor-Specific)

For more complex device sharing (especially graphics), solutions are typically **board/chip vendor-specific**:

| Vendor/Platform | Sharing Technology |
|---|---|
| **Intel x86** | Intel GVT-g (graphics virtualization) |
| **Renesas H3** | PowerVR (graphics sharing) |
| **Qualcomm** | Hardware Abstraction Layer for chipsets |

### Key Points

- Complex sharing (especially GPU/graphics) gets **complex very quickly**
- Usually requires working with **QNX engineering** directly
- **QAVF** (QNX Advanced Virtualization Framework) addresses some of these needs (graphics, audio, touch sharing)
- Solutions are highly hardware-dependent

---

## Quick Reference Summary

```
┌────────────────────────────────────────────────────────────────────┐
│              SHARED DEVICES SUMMARY                                │
│                                                                    │
│  NON-OVERLAPPING MEMORY                                            │
│  • Simplest case — just pass-through configuration                 │
│  • Each guest maps different region of device memory               │
│  • No synchronization needed                                       │
│                                                                    │
│  CONTROLLING DRIVER (HOST)                                         │
│  • Single driver in host manages all device access                 │
│  • Guests communicate via shared memory + sockets                  │
│  • Driver serializes access — prevents conflicts                   │
│  • Fault-isolated — driver crash doesn't crash host                │
│                                                                    │
│  SHARED FLASH/BLOCK STORAGE                                        │
│  • Host runs flash driver, guests use vdev virtio-blk              │
│  • MUST use different partitions per guest                          │
│  • Standard file I/O after configuration                           │
│                                                                    │
│  COMPLEX SHARING                                                   │
│  • Vendor-specific (Intel GVT-g, PowerVR, Qualcomm HAL)           │
│  • Usually requires QNX engineering support                        │
│  • QAVF for graphics/audio/touch sharing                           │
└────────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions & Answers

### Fundamentals

---

**Q1: What is a shared device in the context of the QNX Hypervisor?**

<details>
<summary>Answer</summary>

A **shared device** is physical hardware that is accessed by **multiple guests** and/or by **guests and the host** simultaneously. Since multiple entities need to use the same hardware, proper configuration and access management is required to prevent conflicts and data corruption.

Sharing approaches include:
- **Non-overlapping memory regions** — each guest uses a different part of the device memory via pass-through
- **Controlling driver in the host** — a single driver manages all access, guests communicate via shared memory and sockets
- **Partitioned block storage** — each guest gets its own partition on flash/eMMC
- **Vendor-specific solutions** — for complex devices like GPUs (Intel GVT-g, PowerVR, etc.)

</details>

---

**Q2: How do you share a device between two guests when they need non-overlapping memory regions?**

<details>
<summary>Answer</summary>

This is the **simplest sharing scenario**. Each guest uses pass-through to map a **different, non-overlapping region** of the device memory into its guest physical address space.

**Configuration:**
```
# Guest A: maps first half of device memory
pass 0xFE000000,0x3F000000,0x00800000

# Guest B: maps second half of device memory  
pass 0xFE800000,0x3F000000,0x00800000
```

Since the memory regions don't overlap:
- No synchronization is needed
- No intermediary driver is required
- Each guest's pass-through driver accesses its own region directly
- It's purely a **configuration exercise** using the `pass` option

</details>

---

**Q3: When multiple guests need to share the same device through overlapping resources, what approach should you use?**

<details>
<summary>Answer</summary>

Use a **controlling driver in the host**. This is a single host process that has exclusive access to the physical device. All guests communicate with this driver rather than accessing the hardware directly.

**How it works:**
1. A driver process in the host manages all device access
2. Guest processes communicate with this driver using:
   - **Shared memory** for large data transfers (megabytes)
   - **Socket communication** for notifications and control messages
3. The driver **serializes** all access to the device, preventing conflicts

**Advantages:**
- Single point of control prevents conflicting hardware access
- **Fault isolation** — if the driver dies, it doesn't crash the host (it's just a process)
- A monitoring process can detect and handle driver failures
- Flexible — can use shared memory, sockets, or both

</details>

---

**Q4: How do multiple guests safely share flash storage (eMMC/SD card)?**

<details>
<summary>Answer</summary>

**Architecture:**
- Host runs the flash driver (e.g., `devb-sdmmc` from the board BSP)
- Each guest loads a `vdev virtio-blk` in its configuration
- QNX guests run `devb-virtio`; Linux guests use the built-in VirtIO block driver

**Safety: Use different partitions.**
- Partition the flash/eMMC storage beforehand
- Assign **one partition per guest**
- Since partitions don't overlap, there's no risk of simultaneous writes to the same data
- After configuration, guest processes use standard file I/O (`open`, `read`, `write`)

**Critical rule:** Block regions must be unique — if two guests write to the same partition, data corruption will occur.

</details>

---

### Design & Architecture

---

**Q5: Why is the controlling driver approach preferred for shared device access rather than having multiple guests pass-through to the same device?**

<details>
<summary>Answer</summary>

Direct pass-through to the same device from multiple guests is problematic because:

1. **No synchronization:** Pass-through gives direct, unsynchronized hardware access. Two guests could issue conflicting commands simultaneously, corrupting device state.

2. **No arbitration:** There's no mechanism to ensure orderly access — one guest might interrupt another's multi-step operation.

3. **Not enforced:** The hypervisor doesn't prevent multiple guests from mapping the same device memory — it's the developer's responsibility.

The **controlling driver** approach solves all of these:
- **Serialization:** All requests go through one driver, which processes them in order
- **Arbitration:** The driver can implement priority, queuing, or fairness policies
- **Fault isolation:** The driver is just a host process — if it crashes, the host continues running
- **Monitoring:** Other host processes can detect and restart a failed driver
- **Flexibility:** Can use shared memory for performance, sockets for control

</details>

---

**Q6: In a shared device scenario with a controlling driver, why might you use both shared memory and socket communication rather than just one?**

<details>
<summary>Answer</summary>

Each method has different strengths:

**Shared memory:**
- Best for **large data** transfers (megabytes)
- Very fast — direct memory access, no protocol overhead
- But provides no built-in notification mechanism (you don't know when new data arrives)

**Socket communication:**
- Best for **small control messages** and notifications
- Has built-in connection management and protocol guarantees (TCP)
- But has overhead (protocol headers, checksums, socket buffers) that's unnecessary for bulk data

**Combined approach example:**
```
1. Guest writes 10MB of data to shared memory
2. Guest sends small socket message: "Data ready at offset X, size 10MB"
3. Host driver receives socket notification
4. Host driver reads data directly from shared memory (fast!)
5. Host driver writes data to physical device
```

This gives you the **throughput** of shared memory with the **notification capability** of sockets. Using sockets alone for 10MB would be slower; using shared memory alone would require polling or a separate notification mechanism.

> **Note:** You don't always need both. For small data volumes, sockets alone are perfectly fine.

</details>

---

**Q7: What happens if a controlling driver process in the host crashes? How does the system handle it?**

<details>
<summary>Answer</summary>

Since the controlling driver is **just a normal QNX process** in the host:

1. **The host doesn't crash** — QNX's microkernel architecture provides fault isolation. The driver process dies, but `procnto` and everything else continues running.

2. **Guests lose access** to the shared device — any pending operations will fail, and new requests won't be served until the driver is restarted.

3. **Recovery options:**
   - Another host process can **monitor** the driver (e.g., using QNX's HAM — High Availability Manager)
   - Upon detecting the driver's death, the monitor can **restart** it
   - Guests may need to **reconnect** and re-establish communication
   - Depending on the device, state may need to be reinitialized

4. **Guest impact is limited** to the functionality provided by that specific driver — other guests and other devices are unaffected.

This is a significant advantage of the QNX microkernel architecture over monolithic approaches where a driver crash could bring down the entire system.

</details>

---

**Q8: Why is complex device sharing (like GPU) typically vendor-specific?**

<details>
<summary>Answer</summary>

Complex devices like GPUs have:

1. **Proprietary hardware interfaces:** GPU register sets, memory management, command queues, and rendering pipelines are unique to each vendor/chip
2. **Complex state machines:** GPUs maintain extensive internal state that must be carefully managed when switching between contexts (guests)
3. **Performance sensitivity:** Naive sharing approaches (like a simple intermediary) would severely degrade GPU performance
4. **Hardware-assisted virtualization:** Some GPUs offer built-in virtualization features that are chip-specific:
   - **Intel GVT-g:** Provides GPU virtualization for Intel integrated graphics
   - **PowerVR:** Used on Renesas boards for graphics sharing
   - **Qualcomm HAL:** Hardware abstraction for Qualcomm chipsets
5. **Driver complexity:** GPU drivers are some of the most complex software components — sharing requires deep integration with vendor-specific driver stacks

For these reasons, complex sharing usually requires:
- Working with **QNX engineering** directly
- Using vendor-specific solutions
- Potentially using **QAVF** (QNX Advanced Virtualization Framework) for graphics, audio, and touch sharing

</details>

---

**Q9: Design a shared device architecture for a scenario where three guests all need to write log data to the same serial port.**

<details>
<summary>Answer</summary>

**Problem:** Three guests need to write to one physical serial port. Direct pass-through from all three would cause garbled output.

**Solution: Controlling driver approach**

```
┌──────────┐  ┌──────────┐  ┌──────────┐
│ Guest A   │  │ Guest B   │  │ Guest C   │
│ (Logger)  │  │ (Logger)  │  │ (Logger)  │
└─────┬─────┘  └─────┬─────┘  └─────┬─────┘
      │              │              │
      │     Socket communication    │
      └──────────┬───┴──────────────┘
                 ▼
     ┌────────────────────────┐
     │  Log Multiplexer       │  ← Host process
     │  (Controlling Driver)  │     Serializes access
     │  - Receives messages   │     Adds timestamps/prefixes
     │  - Queues log entries  │     
     │  - Writes to serial    │
     └───────────┬────────────┘
                 ▼
     ┌────────────────────────┐
     │  Physical Serial Port   │
     └────────────────────────┘
```

**Implementation:**
1. **Host:** Run a "log multiplexer" process with exclusive access to the serial port
2. **Each guest `.qvmconf`:** Configure `vdev virtio-net` for networking
3. **Guest processes:** Send log messages via TCP/IP sockets to the multiplexer
4. **Multiplexer:** Queues messages, optionally adds guest ID/timestamp prefix, writes complete lines atomically to the serial port

**Why sockets and not shared memory?**
- Log messages are typically small (< 1KB)
- Socket overhead is negligible for small messages
- Sockets provide natural message boundaries (line-by-line)
- Simpler implementation than shared memory ring buffers

**Why not just pass-through?**
- Three writers to one serial port = garbled, interleaved characters
- No synchronization possible with pass-through
- The multiplexer ensures atomic, ordered log output

</details>

---

**Q10: Compare the three shared device approaches. When would you use each?**

<details>
<summary>Answer</summary>

| Approach | When to Use | Complexity | Performance |
|---|---|---|---|
| **Non-overlapping memory regions** | Device has naturally separable regions (e.g., different framebuffer areas, different register banks). Each guest only needs a subset. | **Low** — just `pass` configuration | **Best** — direct hardware access |
| **Controlling driver** | Multiple guests need the same device resources. Need synchronization, arbitration, or shared access to overlapping regions. | **Medium** — requires host driver + communication setup | **Good** — adds one layer of indirection |
| **Partitioned block storage** | Multiple guests need flash/eMMC but can use separate partitions. | **Medium** — partition setup + VirtIO configuration | **Good** — each guest has direct partition access |
| **Vendor-specific** | Complex devices (GPU, DSP) where generic approaches are insufficient. | **High** — requires vendor-specific solutions, QNX engineering | **Varies** — optimized for specific hardware |

**Decision flow:**
```
Can guests use separate, non-overlapping regions?
  → YES: Use pass-through with non-overlapping regions (simplest)
  → NO: Do guests need the same block storage?
           → YES: Partition the storage + VirtIO
           → NO: Is it a complex device (GPU)?
                    → YES: Vendor-specific solution / QAVF
                    → NO: Use controlling driver in host
```

</details>

---

*These notes are based on the QNX Hypervisor Shared Devices lesson. Related topics (Virtual Devices, Communication, Pass-Through, Safety) are covered in separate lessons.* 
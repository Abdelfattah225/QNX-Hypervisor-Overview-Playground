# QNX Hypervisor - Inter-Guest and Guest-to-Host Communication

## Table of Contents

- [Overview](#overview)
- [Shared Memory Communication](#shared-memory-communication)
- [Networking Communication](#networking-communication)
  - [Guest-to-Guest (Peer-to-Peer)](#guest-to-guest-peer-to-peer)
  - [Guest-to-Host](#guest-to-host)
  - [Guest-to-External World](#guest-to-external-world)
- [Combining Methods](#combining-methods)
- [Quick Reference Summary](#quick-reference-summary)
- [Interview Questions & Answers](#interview-questions--answers)

---

## Overview

Guests can communicate with each other and with the host using two primary methods:

| Method | Use Case | Mechanism |
|---|---|---|
| **Shared Memory** | High-throughput data transfer, low-latency | Memory-mapped regions shared between guests/host with interrupt notification |
| **Networking** | Standard socket-based communication | TCP/IP over VirtIO networking vdevs |

These can also be **combined** — shared memory for bulk data, networking for notifications.

---

## Shared Memory Communication

### Architecture

```
┌─────────────────┐   ┌─────────────────┐   ┌─────────────────┐
│   QNX Guest      │   │   Linux Guest    │   │      HOST       │
│                  │   │                  │   │                 │
│  ┌────────────┐  │   │  ┌────────────┐  │   │  ┌───────────┐ │
│  │  Process   │  │   │  │  Process   │  │   │  │  Process  │ │
│  │  (user)    │  │   │  │  (user)    │  │   │  │           │ │
│  └─────┬──────┘  │   │  └─────┬──────┘  │   │  └─────┬─────┘ │
│        │         │   │        │         │   │        │       │
│  ┌─────┴──────┐  │   │  ┌─────┴──────┐  │   │        │       │
│  │ vdev-shmem │  │   │  │ Kernel     │  │   │        │       │
│  │ (.so)      │  │   │  │ Module     │  │   │        │       │
│  └─────┬──────┘  │   │  └─────┬──────┘  │   │        │       │
└────────┼─────────┘   └────────┼─────────┘   └────────┼───────┘
         │                      │                      │
         └──────────┬───────────┴──────────────────────┘
                    ▼
        ┌───────────────────────┐
        │    SHARED MEMORY      │
        │  (accessible by all)  │
        └───────────────────────┘
```

### Setup Requirements

| Component | Requirement |
|---|---|
| **Each guest config** | Must include `vdev shmem` in the `.qvmconf` file |
| **QNX guest** | Relatively simple — user-space process can access shared memory |
| **Linux guest** | **More complex** — requires a **kernel module** for interrupt handling and physical memory mapping. User-space process communicates with the kernel module, which manages the shared memory |
| **Host process** | Accesses shared memory directly |

### Notification Feature

When one process writes to shared memory, the other participants can be **notified via an interrupt**:

```
Process A writes data to shared memory
  ↓
vdev-shmem triggers interrupt notification
  ↓
Process B (in another guest) receives interrupt → reads new data
Process C (in host) receives interrupt → reads new data
```

> **This is why a Linux kernel module is needed** — interrupt handling in Linux must be done in kernel space, not user space. The kernel module handles the interrupt and notifies the user-space process.

### Sample Code Resources

| Platform | Source |
|---|---|
| **QNX guest** | Hypervisor Board Support Packages (BSPs) — downloadable from QNX Software Center |
| **Linux kernel module** | QNX public documentation / web resources for Linux kernel module samples |
| **Documentation** | QNX docs → Using Virtual Devices → Networking and Shared Memory → Shared Memory section |

---

## Networking Communication

All networking communication uses **VirtIO networking** (`vdev virtio-net`) and standard **TCP/IP socket-level** communication.

### Guest-to-Guest (Peer-to-Peer)

```
┌──────────────────────┐          ┌──────────────────────┐
│      QNX Guest        │          │     Linux Guest       │
│                       │          │                       │
│  ┌─────────────────┐  │          │  ┌─────────────────┐  │
│  │ Process          │  │          │  │ Process          │  │
│  │ (socket comm)    │  │          │  │ (socket comm)    │  │
│  └────────┬────────┘  │          │  └────────┬────────┘  │
│           │           │          │           │           │
│  ┌────────┴────────┐  │          │  ┌────────┴────────┐  │
│  │ Network Stack   │  │          │  │ Network Stack   │  │
│  │ (io-sock)       │  │          │  │ (Linux native)  │  │
│  │ + virtio-net    │  │          │  │ + virtio-net    │  │
│  └────────┬────────┘  │          │  └────────┬────────┘  │
│           │           │          │           │           │
│  ┌────────┴────────┐  │          │  ┌────────┴────────┐  │
│  │ vdev virtio-net │  │          │  │ vdev virtio-net │  │
│  └────────┬────────┘  │          │  └────────┬────────┘  │
└───────────┼───────────┘          └───────────┼───────────┘
            │                                  │
            └──────────┬───────────────────────┘
                       ▼
              Direct peer-to-peer
              (no host networking needed!)
```

#### Configuration Requirements

| Where | What |
|---|---|
| **Each guest `.qvmconf`** | Add `vdev virtio-net` |
| **QNX 8 guest** | Run `io-sock` with VirtIO networking driver |
| **QNX 7.1 guest** | Run `io-pkt` with appropriate VirtIO module |
| **Linux guest** | Configure Linux network stack with `virtio-net` driver (included in Linux) |
| **Host** | **No networking stack needed** for pure guest-to-guest |

#### Key Points

- Communication is done via **normal TCP/IP sockets** — no special API needed
- Once configured, standard socket code works (even basic code from a Google search!)
- **No host networking stack is required** for peer-to-peer guest communication
- Called **"peer-to-peer"** because guests are peers communicating directly

---

### Guest-to-Host

```
┌──────────────────────┐          ┌──────────────────────┐
│      QNX Guest        │          │     Linux Guest       │
│                       │          │                       │
│  ┌─────────────────┐  │          │  ┌─────────────────┐  │
│  │ Process          │  │          │  │ Process          │  │
│  └────────┬────────┘  │          │  └────────┬────────┘  │
│           │           │          │           │           │
│  ┌────────┴────────┐  │          │  ┌────────┴────────┐  │
│  │ Network Stack   │  │          │  │ Network Stack   │  │
│  │ + virtio-net    │  │          │  │ + virtio-net    │  │
│  └────────┬────────┘  │          │  └────────┬────────┘  │
│           │           │          │           │           │
│  ┌────────┴────────┐  │          │  ┌────────┴────────┐  │
│  │ vdev virtio-net │  │          │  │ vdev virtio-net │  │
│  └────────┬────────┘  │          │  └────────┬────────┘  │
└───────────┼───────────┘          └───────────┼───────────┘
            │                                  │
            └──────────┬───────────────────────┘
                       ▼
┌─────────────────────────────────────────────────────────┐
│                        HOST                              │
│                                                         │
│  ┌─────────────────────────────────────────┐            │
│  │  io-sock                                │            │
│  │  + vdevpeer-net module                  │            │
│  └──────────────────┬──────────────────────┘            │
│                     │                                   │
│  ┌──────────────────┴──────────────────────┐            │
│  │  Host Process (socket communication)     │            │
│  └──────────────────────────────────────────┘            │
└─────────────────────────────────────────────────────────┘
```

#### Additional Configuration (Beyond Peer-to-Peer)

| Where | What |
|---|---|
| **Host** | Run `io-sock` with the **`vdevpeer-net`** module |
| **Host process** | Use normal socket communication to talk to guests |

#### Key Points

- Everything from peer-to-peer setup **plus** host networking stack with `vdevpeer-net`
- Communication is **fairly efficient** — internally it's essentially `memcpy()` operations
- Standard TCP/IP sockets between guests and host

---

### Guest-to-External World

```
┌──────────────────────┐
│      Guest            │
│  ┌─────────────────┐  │
│  │ Process          │  │
│  └────────┬────────┘  │
│  ┌────────┴────────┐  │
│  │ Network Stack   │  │
│  │ + virtio-net    │  │
│  └────────┬────────┘  │
│  ┌────────┴────────┐  │
│  │ vdev virtio-net │  │
│  └────────┬────────┘  │
└───────────┼───────────┘
            │
            ▼
┌─────────────────────────────────────────────────────────┐
│                        HOST                              │
│                                                         │
│  ┌─────────────────────────────────────────┐            │
│  │  io-sock                                │            │
│  │  + vdevpeer-net module                  │            │
│  │  + devs-genet driver (or appropriate    │            │
│  │    Ethernet hardware driver)            │            │
│  └─────┬──────────────────────┬────────────┘            │
│        │                      │                         │
│   vdevpeer-net           devs-genet                     │
│   interface              interface                      │
│        │                      │                         │
│        └──────┬───────────────┘                         │
│               ▼                                         │
│        ┌──────────────┐                                 │
│        │ Bridge       │  ← Connects the two interfaces  │
│        │ Interface    │                                  │
│        └──────┬───────┘                                 │
└───────────────┼─────────────────────────────────────────┘
                │
                ▼
        ┌───────────────┐
        │   Ethernet    │
        │   Hardware    │
        │   (Physical)  │
        └───────┬───────┘
                │
                ▼
          External World
```

#### Additional Configuration (Beyond Guest-to-Host)

| Where | What |
|---|---|
| **Host `io-sock`** | Load the **Ethernet hardware driver** (e.g., `devs-genet`) in addition to `vdevpeer-net` |
| **Host networking** | Configure a **bridge interface** between the `vdevpeer-net` interface and the Ethernet hardware driver interface |

#### Data Flow (Guest Process → External World)

```
Guest process → socket API → network stack → vdev virtio-net
  → vdevpeer-net module → bridge → devs-genet driver → Ethernet hardware
  → External network
```

---

## Combining Methods

For optimal performance, you can **combine shared memory and networking**:

```
┌───────────────────────────────────────────────────────────────┐
│  COMBINED APPROACH                                            │
│                                                               │
│  Shared Memory: For BULK DATA transfer (megabytes)            │
│  • Fast — direct memory access                                │
│  • No protocol overhead                                       │
│                                                               │
│  Socket Communication: For NOTIFICATIONS / CONTROL            │
│  • "Data is ready" signals                                    │
│  • Small control messages                                     │
│  • Leverages existing TCP/IP infrastructure                   │
│                                                               │
│  Example flow:                                                │
│  1. Guest process writes large dataset to shared memory       │
│  2. Guest process sends small socket message to host:         │
│     "Data ready at offset X, size Y"                          │
│  3. Host process reads data directly from shared memory       │
└───────────────────────────────────────────────────────────────┘
```

---

## Quick Reference Summary

```
┌────────────────────────────────────────────────────────────────────┐
│              COMMUNICATION METHODS SUMMARY                         │
│                                                                    │
│  SHARED MEMORY (vdev-shmem)                                        │
│  • Configure vdev shmem in each guest's .qvmconf                   │
│  • QNX guest: user-space process (simple)                          │
│  • Linux guest: kernel module required (interrupt handling)         │
│  • Interrupt-based notification when data is written               │
│  • Best for: large data transfers                                  │
│                                                                    │
│  NETWORKING (vdev virtio-net)                                      │
│  • Guest-to-Guest: vdev virtio-net in each guest, no host stack    │
│  • Guest-to-Host: + io-sock with vdevpeer-net in host              │
│  • Guest-to-World: + Ethernet driver + bridge in host              │
│  • Standard TCP/IP socket communication                            │
│  • Internally efficient (memcpy-based)                             │
│                                                                    │
│  COMBINED                                                          │
│  • Shared memory for bulk data                                     │
│  • Sockets for notifications/control                               │
└────────────────────────────────────────────────────────────────────┘
```

---

## Interview Questions & Answers

### Fundamentals

---

**Q1: What are the two primary methods for inter-guest and guest-to-host communication in the QNX Hypervisor?**

<details>
<summary>Answer</summary>

1. **Shared Memory** — using the `vdev-shmem` virtual device. Guests and the host share a memory region. Writing to the shared memory can trigger interrupt-based notifications to other participants. Best for high-throughput, low-latency data transfer.

2. **Networking** — using the `vdev virtio-net` virtual device with standard TCP/IP socket communication. Supports guest-to-guest (peer-to-peer), guest-to-host, and guest-to-external-world communication. Uses VirtIO networking and standard socket APIs.

These methods can also be **combined** — shared memory for bulk data transfer and sockets for control/notification messages.

</details>

---

**Q2: How does shared memory communication work between guests in the QNX Hypervisor?**

<details>
<summary>Answer</summary>

1. Each guest's `.qvmconf` configuration file must include the **`vdev shmem`** virtual device
2. `qvm` sets up a shared memory region accessible by all configured participants (guests and host)
3. Processes in each guest can read/write to this shared memory
4. When one process writes data, others can be **notified via an interrupt** (provided by `vdev-shmem`)

**Platform-specific requirements:**
- **QNX guest:** Relatively simple — user-space process can directly work with shared memory
- **Linux guest:** Requires a **kernel module** because:
  - Interrupt handling must be done in kernel space
  - Physical memory mapping must be done in kernel space
  - The kernel module then notifies the user-space process of new data
- **Host:** Process accesses shared memory directly

</details>

---

**Q3: Why does a Linux guest require a kernel module for shared memory communication, while a QNX guest doesn't?**

<details>
<summary>Answer</summary>

The shared memory mechanism relies on **interrupt-based notifications** (provided by `vdev-shmem`) and **physical memory mapping**.

In Linux:
- **Interrupt handling** can only be done in **kernel space** — user-space processes cannot directly register and handle hardware interrupts
- **Physical memory mapping** (mapping guest physical memory into a process's address space) requires kernel-level operations
- Therefore, a **kernel module** is needed to handle interrupts and memory mapping, and then communicate with the user-space process through a kernel interface (e.g., device file, ioctl, etc.)

In QNX:
- The architecture allows user-space processes to handle these operations more directly due to QNX's microkernel design, making the implementation simpler

</details>

---

### Networking

---

**Q4: How do you set up guest-to-guest (peer-to-peer) networking communication?**

<details>
<summary>Answer</summary>

**Configuration steps:**

1. **Each guest's `.qvmconf`:** Add `vdev virtio-net`

2. **Inside each guest — configure networking stack:**
   - QNX 8 guest: Run `io-sock` with VirtIO networking driver
   - QNX 7.1 guest: Run `io-pkt` with VirtIO module
   - Linux guest: Configure Linux network stack with `virtio-net` driver (included in Linux kernel)

3. **No host networking stack needed** for pure guest-to-guest communication

4. **Application code:** Use standard TCP/IP socket APIs (connect, send, recv, etc.)

**Key point:** Once configured, it's just normal socket communication — no special hypervisor-aware API needed at the application level.

</details>

---

**Q5: What additional configuration is needed for guest-to-host networking compared to guest-to-guest?**

<details>
<summary>Answer</summary>

Beyond the guest-to-guest setup, you need:

1. **Host networking stack:** Run `io-sock` in the host with the **`vdevpeer-net`** module loaded
2. This creates a network interface on the host side that connects to the guests' VirtIO network interfaces

The `vdevpeer-net` module is the host-side counterpart to the guests' `vdev virtio-net`. It allows the host's networking stack to communicate with the guest networking through the VirtIO mechanism.

Communication is **fairly efficient** — internally it's essentially `memcpy()` operations.

</details>

---

**Q6: How do you enable a guest to communicate with the external world (outside the board)?**

<details>
<summary>Answer</summary>

Beyond the guest-to-host setup, you need:

1. **Ethernet hardware driver:** Load the appropriate driver in `io-sock` (e.g., `devs-genet` for Genet Ethernet hardware)
2. **Bridge interface:** Configure a **network bridge** between:
   - The `vdevpeer-net` interface (connects to guests)
   - The Ethernet hardware driver interface (connects to physical network)

**Data flow:**
```
Guest process → socket API → guest network stack → vdev virtio-net
  → vdevpeer-net (host) → bridge interface → Ethernet driver → Physical NIC
  → External network
```

The bridge allows traffic to flow between the virtual network (guests) and the physical network (external world).

</details>

---

**Q7: Is a host networking stack required for all guest networking scenarios?**

<details>
<summary>Answer</summary>

**No.** It depends on the communication pattern:

| Scenario | Host Networking Stack Required? |
|---|---|
| **Guest-to-Guest (peer-to-peer)** | **No** — guests communicate directly through VirtIO |
| **Guest-to-Host** | **Yes** — need `io-sock` with `vdevpeer-net` module |
| **Guest-to-External World** | **Yes** — need `io-sock` with `vdevpeer-net` + Ethernet driver + bridge |

For pure peer-to-peer guest communication, only the guests need networking configured. The host networking stack is only needed when the host itself or the external network is a communication endpoint.

</details>

---

### Design & Architecture

---

**Q8: When would you choose shared memory over networking for inter-guest communication, and vice versa?**

<details>
<summary>Answer</summary>

**Choose shared memory when:**
- Transferring **large amounts of data** (megabytes or more)
- **Low latency** is critical — shared memory avoids protocol overhead
- **High throughput** is needed — direct memory access is faster than socket protocol processing
- You need **zero-copy** semantics — data is written once and read directly

**Choose networking when:**
- You want to use **standard TCP/IP APIs** — portable, well-understood
- Communication involves **small messages** or control signals
- You need **protocol guarantees** (TCP reliability, ordering)
- You want to **reuse existing socket-based code** without modification
- You may need to communicate with the **external world** later

**Best practice — combine both:**
- Shared memory for **bulk data** transfer
- Socket communication for **notifications** ("data is ready at offset X, size Y")

</details>

---

**Q9: Describe a real-world automotive scenario where you would combine shared memory and networking for inter-guest communication.**

<details>
<summary>Answer</summary>

**Scenario: Camera feed from safety guest to infotainment guest**

A QNX safety-critical guest processes rear-view camera data. The Linux infotainment guest needs to display it on the center console screen.

**Design:**
1. **Shared memory:** The QNX guest writes camera frame data (e.g., 1920x1080 frames, several MB each) to a shared memory region. The Linux guest reads frames directly — no data copying through sockets.

2. **Socket communication (or shared memory interrupt):** After writing a new frame, the QNX guest sends a small notification message (frame number, timestamp, offset) to the Linux guest via either:
   - A TCP/IP socket message, or
   - The `vdev-shmem` interrupt notification mechanism

3. **Linux guest:** The kernel module handles the notification interrupt, signals the display application, which reads the new frame from shared memory and renders it.

**Why this design?**
- Camera frames are **too large** for efficient socket transfer (protocol overhead, copying)
- Shared memory provides **near-zero-latency** data access
- Notification can be lightweight (small socket message or interrupt)
- The QNX guest maintains control over frame production rate

</details>

---

**Q10: Explain the role of `vdev-shmem`, `vdev virtio-net`, and `vdevpeer-net`. How are they different?**

<details>
<summary>Answer</summary>

| Component | Type | Where It Runs | Purpose |
|---|---|---|---|
| **`vdev-shmem`** | Virtual device (.so) | Loaded by `qvm` for each guest | Sets up **shared memory** between guests/host. Provides interrupt-based notification when data is written. |
| **`vdev virtio-net`** | Virtual device (.so) | Loaded by `qvm` for each guest | Provides a **virtual network interface** in the guest using VirtIO. Guest network stack talks to this. |
| **`vdevpeer-net`** | Networking module | Loaded by `io-sock` in the **host** | **Host-side counterpart** to guests' `vdev virtio-net`. Creates a host network interface that connects to the guest virtual network. |

**Relationships:**
- `vdev-shmem` = shared memory path (no networking involved)
- `vdev virtio-net` (guest) ↔ `vdevpeer-net` (host) = networking path
- For guest-to-guest only, `vdevpeer-net` is not needed
- For guest-to-host or guest-to-world, `vdevpeer-net` is required

</details>

---

**Q11: How would you configure a system where a QNX guest, a Linux guest, and a host process all need to communicate with each other AND with an external Ethernet network?**

<details>
<summary>Answer</summary>

**Guest configuration (each guest's `.qvmconf`):**
```
vdev virtio-net
  # peer configuration for networking
vdev shmem
  # if shared memory is also needed
```

**QNX guest (inside guest):**
```bash
io-sock # with VirtIO networking driver
ifconfig virt0 192.168.1.10   # assign IP
```

**Linux guest (inside guest):**
```bash
# Configure virtio-net driver (included in kernel)
ifconfig eth0 192.168.1.20    # assign IP
# If using shared memory: load kernel module for shmem
```

**Host configuration:**
```bash
# Start networking stack with both vdevpeer and Ethernet driver
io-sock -d vdevpeer-net -d devs-genet

# Configure interfaces
ifconfig vdevpeer0 192.168.1.1    # virtual peer interface
ifconfig genet0 10.0.0.100        # physical Ethernet interface

# Create bridge between virtual and physical networks
brconfig bridge0 add vdevpeer0 add genet0 up
```

**Result:**
- QNX guest ↔ Linux guest: Peer-to-peer via VirtIO networking
- QNX guest ↔ Host: Via VirtIO → vdevpeer-net
- Linux guest ↔ Host: Via VirtIO → vdevpeer-net
- Any guest ↔ External: Via VirtIO → vdevpeer-net → bridge → genet → physical Ethernet
- Host ↔ External: Via genet → physical Ethernet

</details>

---

**Q12: What makes the networking communication in the QNX Hypervisor "fairly efficient"?**

<details>
<summary>Answer</summary>

The networking path between guests and host is efficient because:

1. **No physical network hardware involved** (for guest-to-guest and guest-to-host): Data doesn't traverse an actual NIC — it stays in memory

2. **Internally uses `memcpy()`**: The data transfer between the VirtIO vdev and `vdevpeer-net` is essentially memory copies — no serialization/deserialization, no wire protocol encoding

3. **VirtIO is designed for virtualization efficiency**: Uses virt queues that can batch multiple packets, reducing guest exits

4. **Shared address space**: The `qvm` process, which manages the VirtIO vdev, and the host networking stack are all in the same host memory space, so data transfers are local memory operations

However, for **maximum performance** with large data transfers, shared memory is still faster because it avoids the TCP/IP protocol stack overhead entirely (no headers, no checksums, no socket buffers, no protocol processing).

</details>

---

**Q13: A developer is confused about why their socket code works for guest-to-guest communication but not for guest-to-host. What's the likely issue?**

<details>
<summary>Answer</summary>

The most likely issue is that the **host networking stack is not configured with the `vdevpeer-net` module**.

For **guest-to-guest** communication, no host networking is needed — the VirtIO vdevs handle the peer-to-peer path directly.

For **guest-to-host** communication, the host needs:
1. `io-sock` running with the **`vdevpeer-net`** module loaded
2. The `vdevpeer` interface configured with an **IP address** on the same subnet as the guests

**Debugging checklist:**
- Is `io-sock` running on the host? (`pidin | grep io-sock`)
- Was `vdevpeer-net` loaded as a module? (check `io-sock` command line arguments)
- Is the `vdevpeer` interface assigned an IP address? (`ifconfig`)
- Are the guest and host IP addresses on the same subnet?
- Can you ping between guest and host?

</details>

---

**Q14: What is the role of a bridge interface in the host networking configuration?**

<details>
<summary>Answer</summary>

A **bridge interface** connects two or more network interfaces at Layer 2 (data link layer), allowing traffic to flow between them as if they were on the same network segment.

In the QNX Hypervisor context, it connects:
- The **`vdevpeer-net` interface** (virtual, connects to guests)
- The **Ethernet hardware driver interface** (physical, connects to external network)

**Without a bridge:** Guest traffic can reach the host (via `vdevpeer-net`) but cannot reach the external network, because the virtual and physical interfaces are separate, unconnected networks.

**With a bridge:** Packets from guests flow through `vdevpeer-net` → bridge → Ethernet driver → physical network, and vice versa. The bridge forwards packets between the two interfaces transparently.

**Configuration:**
```bash
brconfig bridge0 add vdevpeer0 add genet0 up
```

</details>

---

*These notes are based on the QNX Hypervisor Inter-Guest and Guest-to-Host Communication lesson. Related topics (Virtual Devices, Shared Devices, Interrupts) are covered in separate lessons.*
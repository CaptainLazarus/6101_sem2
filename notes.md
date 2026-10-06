# Space Networks — Initial Reading Notes

Paper: *Revisiting RDMA Reliability for Lossy Fabrics* (SIGCOMM 2025).

Reference material for later:
[Networking and RDMA Reference](networking_reference.md) contains the field
map, sources, mental models, competing positions, and diagnostic questions.
[DCP Reference](dcp_reference.md) contains the paper-specific map, assumptions,
evaluation evidence, and corrections to earlier explanations.

## Core model

The paper studies reliable data transfer over a high-speed data-centre network.
The final RDMA operation should deliver the exact data, even if packets are
dropped inside a **lossy fabric**. A **lossless fabric** instead tries to prevent
packet drops inside the network.

RDMA (Remote Direct Memory Access) lets one machine read or write a registered
memory region on another machine over a network, with minimal per-packet CPU
and kernel involvement. An **RNIC** is the RDMA-capable Network Interface Card
that performs much of this work. This paper focuses on **RoCEv2**, RDMA carried
over UDP/IP and Ethernet.

## Hardware flow

```text
Host memory → PCIe → RNIC → Ethernet/fibre link → switches → RNIC → PCIe → host memory
```

Switches contain forwarding logic, buffers, and logical queues. Buffers are
physical memory for temporarily holding packets; queues organize those packets
for scheduling. The **fabric** means the network infrastructure: switches,
links, forwarding, and queueing behaviour. It may be lossy or lossless.

### RDMA transfer walkthrough scope

We are studying the process of sending data, not the initialization itself.
For our walkthrough of the SRNIC conceptual model, assume setup is complete:

- Both applications (the actual software programs on the two machines) are
  running, and the sender's data is already in RAM.
- The RNIC connection and its queues have been created.
- The sender's data buffer and a sufficiently large receiver buffer have been
  allocated and registered with their RNICs.
- The receiver has posted a RECEIVE WQE identifying its available buffer.

The receiver application prepares memory based on its own logic or an
application-level exchange; the RNIC does not automatically discover the file
size and allocate a buffer. The conceptual diagram does not explain this
coordination.

Step 1 starts the transfer walkthrough: the sender posts a SEND WQE asking its
RNIC to send data from the existing buffer. In step 4, the receiving RNIC reads
the already-posted RECEIVE WQE, locates the existing destination buffer, and
writes the data there using DMA. It does not allocate that buffer at step 4.
For this walkthrough the receive instruction is ready beforehand; technically,
it must be available before the incoming SEND is processed, not necessarily
before the sender posts its SEND WQE.

Reference: [RDMA_Detailed.pdf](RDMA_Detailed.pdf), Figure 3 and §3.1–3.2
(PDF page 5, printed page 4). These are walkthrough assumptions, not a claim
that the figure specifies the full initialization procedure.

## Reliability vocabulary

- **Go-Back-N:** after a loss, resend the missing packet and later packets.
- **Selective Repeat:** resend only the missing packet(s). Hardware versions
  often use simplified tracking. Detailed explanation deferred at the user's
  request while reading the Introduction.
- **RTO:** Retransmission Timeout. If an expected acknowledgement does not
  arrive before a timer expires, assume loss and retransmit. This is slower and
  less precise than an immediate loss notification.
- **PFC:** Priority-based Flow Control. When a switch queue crosses a congestion
  threshold, it sends PAUSE upstream for the associated priority; transmission
  resumes when allowed. This helps prevent buffer-overflow drops.

### Flows queues and PFC priorities

Many flows share a limited number of queues at an output port. Traffic is
assigned priority classes; several flows may share the same class and queue.
In DCP, the switch's data and control queues are tied to specific outgoing
(egress) ports. Multiple incoming ports can feed a queue for the same outgoing
port; a queue need not correspond to one incoming–outgoing port pair. This is
explicit in the paper's §4.1 and §5, not a property of every queue data structure.
PFC pauses a priority on a particular link, so every flow using that priority
on the link can be paused, including flows not causing the congestion.

```text
Many flows → shared priority class / queue
          → PFC pauses that priority on the link
          → all those flows are affected
```

The rationale is implementation simplicity and a fast local reaction, rather
than identifying and signaling individual end-to-end flows. This does not mean
that PFC makes network operation simple overall. Per-flow congestion control
can provide more precise rate regulation; PFC is the coarse local safeguard.

## Path and switch vocabulary

- **ECMP:** Equal-Cost Multi-Path. A switch hashes packet-header fields such as
  addresses, ports, and protocol to assign a flow to one of several paths.
- **ECMP hash collision:** multiple flows map to the same path, potentially
  creating a hotspot while other paths are underused.
- **Packet-level load balancing:** choose a path per packet rather than per
  flow. This can use paths more evenly but can cause out-of-order arrival.
- **Packet trimming:** under congestion, remove a packet's payload but preserve
  its header. The header becomes a compact control signal identifying data that
  needs retransmission.

## Hardware offloading

Hardware offloading means moving work from general-purpose CPU/software into
specialized hardware. An RNIC may handle packet parsing and formation, memory
placement, acknowledgements, retransmissions, and packet-state tracking. The
CPU still configures the RNIC, submits operations, and receives completions.

## Terms still to learn

Data plane and control plane; DCP; DCP-Switch; DCP-RNIC; lossless control plane;
header-only retransmission; bitmap-free packet tracking; P4; FPGA; and SOTA.

## Abbreviation glossary

The full wording of an abbreviation is called its expansion or full form.
This list covers the abbreviations in our notes and discussion so far;
extend it as we encounter more terms.

| Abbreviation | Expansion / full form |
|---|---|
| RDMA | Remote Direct Memory Access |
| NIC | Network Interface Card |
| RNIC | RDMA-capable Network Interface Card |
| RoCE | RDMA over Converged Ethernet |
| RoCEv1 / RoCEv2 | RoCE version 1 / version 2 |
| iWARP | Internet Wide Area RDMA Protocol |
| PFC | Priority-based Flow Control |
| GBN | Go-Back-N |
| SR | Selective Repeat |
| RNIC-GBN | An RNIC using Go-Back-N recovery |
| RNIC-SR | An RNIC using Selective Repeat recovery |
| RTO | Retransmission Timeout |
| ECMP | Equal-Cost Multi-Path |
| LB | Load Balancing |
| AR | Adaptive Routing |
| OOO | Out of Order |
| HoL | Head of Line |
| DP | Data Plane |
| CP | Control Plane (in DCP's terminology) |
| HO | Header Only |
| PSN | Packet Sequence Number |
| QP | Queue Pair |
| QPN | Queue Pair Number |
| DCP | The proposed architecture's name; the paper does not explicitly provide a full-form expansion |
| DCP-Switch | The switch component of DCP |
| DCP-RNIC | The RNIC component of DCP |
| P4 | Programming Protocol-independent Packet Processors |
| FPGA | Field-Programmable Gate Array |
| SOTA | State of the Art |
| CPU | Central Processing Unit |
| PCIe | Peripheral Component Interconnect Express |
| DMA | Direct Memory Access |
| TCP | Transmission Control Protocol |
| UDP | User Datagram Protocol |
| IP | Internet Protocol |
| ACK | Acknowledgement |
| MTU | Maximum Transmission Unit |
| FCT | Flow Completion Time |
| JCT | Job Completion Time |
| DC | Data Centre |
| CC | Congestion Control |
| DCQCN | Data Center Quantized Congestion Notification |
| IRN | Improved RoCE NIC |
| CX5 | ConnectX-5, a commercial Mellanox RNIC used as a comparison baseline |
| MP-RDMA | Multi-Path RDMA |
| AI | Artificial Intelligence |
| LUT | Look-Up Table |
| BRAM | Block Random-Access Memory |
| SRAM | Static Random-Access Memory |
| Gb/s / Gbps | Gigabits per second |
| Mpps | Million packets per second |
| KB / MB / GB | Kilobytes / megabytes / gigabytes |
| μs / ms | Microseconds / milliseconds |

Sources: the local paper and our reading notes; the
[P4 language specification](https://p4.org/wp-content/uploads/sites/53/p4-spec/docs/P4-16-v1.2.4.html)
confirms the origin of the P4 name.

## Questions to revisit

1. What are the causes of data loss? Which causes does DCP handle, and which
   causes does it not handle?
2. Is DCP-RNIC an RNIC or a separate technology based on RNICs?
   Answer: RNIC is a hardware category; DCP-RNIC is a particular RNIC design.
   The authors modify an RNIC-GBN baseline and prototype it on an FPGA.
3. What is a control plane, particularly in DCP? (Pending.)
4. What are P4 and FPGA, and how are they used in the prototype?
   P4 discussed: a language for programming packet processing; the authors
   use it for DCP-Switch. FPGA explanation is next.

## Today's deadlines — 5 October 2026

All times are Singapore time. Planned at 11:04 AM, with lunch in about ten
minutes; assume roughly 45 minutes for lunch and return around noon.
Continue explanations one term at a time, with the user choosing what to expand.

| Deadline | Deliverable |
|---|---|
| 11:15 AM | Clarify FPGA and finish the abstract's terminology as far as time permits |
| Noon | Return from lunch and resume reading |
| 1:30 PM | Understand background and motivation: RDMA versus sockets, PFC, loss recovery, reordering |
| 3:00 PM | Explain DCP's mechanisms from memory using a concrete packet example |
| 4:00 PM | Understand main evaluation results; record limitations and answer deferred questions |
| 5:30 PM | Complete the slide deck using presentation_draft.md; roughly ten minutes of background and motivation |
| 6:30 PM | Finish a timed 20-minute rehearsal and revise unclear or overlong slides |
| 7:00 PM | Final PPT/PDF, backup, and short Q&A preparation |

## RDMA application evidence

These papers are cited in the opening sentence of DCP's Introduction.
Reference numbers below belong to the DCP paper. Results describe the
specified workloads, not universal RDMA speedups.

| Reference and paper | RDMA application | Measured evidence and conditions |
|---|---|---|
| [16] [Empowering Azure Storage with RDMA](https://www.usenix.org/system/files/nsdi23-bai.pdf), NSDI 2023 | Compute-to-storage communication and communication within storage clusters | §8.2, Figures 9–10: up to 34.5% lower host-domain CPU utilization than TCP in an 8-KB I/O test-cluster experiment. Separately, monitoring of test VMs across Azure regions found 23.8% lower read latency and 15.6% lower write latency for 1-MB I/O requests. |
| [23] [When Cloud Storage Meets RDMA](https://www.usenix.org/system/files/nsdi21-gao.pdf), NSDI 2021 | Alibaba's Pangu distributed storage | §3.3, Figure 3: FIO virtual-disk writes with 16-KB blocks, 8 jobs, and depth 8; all-RDMA traffic had approximately half the average BlockServer request latency of all-TCP traffic. TCP tail latency was over 10 times larger. The test enabled the paper's kernel TCP optimizations. |
| [22] [RDMA over Ethernet for Distributed AI Training at Meta Scale](https://engineering.fb.com/wp-content/uploads/2024/08/sigcomm24-final246.pdf), SIGCOMM 2024 | Communication between servers doing distributed GPU training | Documents real production RoCE deployments. Useful as an application example; not a clean RDMA-versus-TCP benchmark. |

The other references in the same citation list are transport or NIC designs,
not three additional application deployments:

- [41] [TIMELY](https://research.google/pubs/timely-rtt-based-congestion-control-for-the-datacenter/), SIGCOMM 2015: congestion control based on round-trip delay.
- [42] [Revisiting Network Support for RDMA](https://arxiv.org/abs/1806.08159), SIGCOMM 2018: IRN, efficient loss recovery without PFC.
- [53] [SRNIC](https://www.usenix.org/system/files/nsdi23-wang-zilong.pdf), NSDI 2023: RNIC connection scalability and the benchmark below.

Storage request latency includes application/storage effects. Do not equate
it with the latency of transferring a small message over the network.

## RDMA and TCP benchmark numbers

[SRNIC §6.2, Figure 10](https://www.usenix.org/system/files/nsdi23-wang-zilong.pdf):
one connection, directly connected 100-Gbps interfaces, 1024-byte RoCE MTU.

| Transport and hardware | 64-byte message latency | Throughput for messages larger than 4 KB | Reported CPU utilization |
|---|---|---|---|
| TCP sockets | About 24 μs | Up to 37 Gbps | Around 100% |
| RDMA with ConnectX-6 RNIC | About 1.16 μs | About 97 Gbps | Below 5% |

TCP was limited by a single CPU core. Its multithreaded experiment in §6.1
reached 81–96 Gbps. Do not label the CPU percentages as whole-server usage.
The single-connection results imply approximately 21 times lower small-message
latency and 2.6 times higher throughput in that experiment, not universally.

## Network and connection scalability

[SRNIC §§1–2.2](https://www.usenix.org/system/files/nsdi23-wang-zilong.pdf):
network scalability concerns performance and operation as the fabric grows;
PFC's network-wide effects are a limitation. Connection scalability concerns
one RNIC maintaining performance with many active queue pairs; limited
on-chip state and cache misses are limitations. These are related but distinct.

# Project Context

## Current task

Tomorrow we need to give a **20-minute presentation** on:

> Wenxue Li et al., “Revisiting RDMA Reliability for Lossy Fabrics,” ACM
> SIGCOMM 2025.

The professor’s requirements are:

- make the presentation self-contained;
- explain how RDMA works;
- explain RDMA’s performance benefits and how it differs from conventional
  socket-based communication;
- explain how existing RDMA networks handle congestion and packet loss;
- explain the limitations that motivate this work;
- explain DCP’s key design ideas and the main evaluation results;
- discuss our perspective: assumptions, limitations, and questions;
- use diagrams and concrete examples to explain the mechanisms;
- spend roughly the first half on background and motivation.

Do not make the presentation automatically. The user will make the slides,
with help understanding the paper and possibly later help planning or checking
the presentation.

## How to continue learning

The agreed method for unfamiliar papers is incremental and source-driven:

1. Build a bounded source set: target paper, relevant background material, and
   a small number of foundational sources.
2. Identify the core mental models experts share.
3. Map major disagreements and the strongest argument on each side.
4. Record assumptions, limitations, and open questions.
5. Generate diagnostic questions that distinguish understanding from memorized
   facts.
6. Answer them from memory and the sources.
7. Correct each wrong or incomplete answer against the source material.
8. Build both a field map and a paper-specific map.

When starting from zero, define one term at a time and do not front-load a
dense explanation. The user prefers short, focused explanations and will say
when to move to the next term.

## Conceptual foundation established

The paper is about sending data from one point to another over a high-speed,
wired data-centre network. The final RDMA operation should be exact and
reliable even when packets may be dropped inside a **lossy fabric**.

RDMA means **Remote Direct Memory Access**: one machine can read from or write
to a registered memory region on another machine over a network, with minimal
per-packet CPU and kernel involvement.

An **RNIC** is an RDMA-capable Network Interface Card/Controller. It is the
hardware that performs much of the RDMA work. RDMA is the communication
technology/model; the RNIC is the hardware implementation that makes it fast.

The paper concerns **RoCEv2**, a specific technology for transporting RDMA
over UDP/IP and Ethernet. It is not the umbrella name for all RDMA transports.
Other examples are InfiniBand, RoCEv1, and iWARP.

## Hardware flow

```text
Server A / host memory
        ↓ PCIe
      RNIC A
        ↓ Ethernet or fibre link
  switches with forwarding logic,
  buffers, queues, and scheduling
        ↓ Ethernet or fibre links
      RNIC B
        ↓ PCIe
Server B / remote host memory
```

PCIe is the internal server connection between the host system and RNIC.
Ethernet/fibre links connect RNICs to switches and switches to one another.
Buffers and queues are components inside switches, not separate devices:
buffers are physical packet memory; queues are logical lines of waiting
packets.

The **fabric** means the network infrastructure: switches, links, forwarding,
and queueing behaviour. It can be lossless or lossy.

## Reliability terms

- **Go-Back-N:** if packet 3 is lost, resend packet 3 and later packets.
- **Selective Repeat:** resend only the missing packet(s). Existing RNIC
  implementations use simplified, hardware-friendly versions.
- **RTO:** Retransmission Timeout. If an expected acknowledgement does not
  arrive before a timer expires, assume loss and retransmit. It is slower and
  less precise than an immediate loss notification.
- **PFC:** Priority Flow Control. If a switch queue fills, it sends PAUSE
  upstream; after the queue drains, it sends RESUME. PFC tries to make the
  fabric lossless by preventing drops.

## Path and switch terms

- **ECMP:** Equal-Cost Multi-Path. A switch hashes packet-header fields such
  as addresses, ports, and protocol to assign a flow to one of several paths.
- **ECMP hash collision:** different flows map to the same path. Collisions are
  unavoidable; the performance problem is that large flows can create hotspots
  while other paths are underused.
- **Packet-level load balancing:** choose a path per packet rather than per
  flow. This can use paths more evenly but can cause out-of-order arrival.
- **Packet trimming:** under congestion, remove a packet’s payload but preserve
  its header. The header becomes a compact control signal identifying data that
  needs retransmission.
- **Hardware offloading:** move networking work from general-purpose CPU and
  software into specialized hardware such as the RNIC. The CPU still configures
  the RNIC, submits operations, and receives completions.

## Core tension

“Lossless RDMA” means the network fabric tries to prevent packet loss, usually
with PFC. “Lossy RDMA” means the fabric may drop packets while the endpoints
recover them. Both aim for a complete, exact final data transfer.

The basic idea “detect missing packets and retransmit them” is straightforward.
The practical difficulty is doing it at high speed while handling packet
reordering, congestion, slow or false timeouts, hardware memory limits, and
low RNIC processing overhead.

## Terms still to learn from the abstract

- data plane versus control plane;
- DCP transport architecture;
- DCP-Switch and DCP-RNIC;
- lossless control plane;
- header-only retransmission;
- bitmap-free packet tracking;
- P4 programmable switch;
- FPGA prototype;
- SOTA (state of the art).

## Current repository

`pldi16.pdf` is the paper. `notes.md` contains the initial vocabulary notes.
This file is the handoff context for continuing in a new conversation.

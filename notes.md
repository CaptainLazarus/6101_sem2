# Space Networks — Initial Reading Notes

Paper: *Revisiting RDMA Reliability for Lossy Fabrics* (SIGCOMM 2025).

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

## Reliability vocabulary

- **Go-Back-N:** after a loss, resend the missing packet and later packets.
- **Selective Repeat:** resend only the missing packet(s). Hardware versions
  often use simplified tracking.
- **RTO:** Retransmission Timeout. If an expected acknowledgement does not
  arrive before a timer expires, assume loss and retransmit. This is slower and
  less precise than an immediate loss notification.
- **PFC:** Priority Flow Control. When a switch queue fills, it sends PAUSE
  upstream; after the queue drains, it sends RESUME. This helps create a
  lossless fabric.

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

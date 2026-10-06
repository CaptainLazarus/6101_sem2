# DCP Reference

A compact map of *Revisiting RDMA Reliability for Lossy Fabrics*, grounded in
the [local paper](pldi16.pdf). Use alongside the broader
[Networking and RDMA Reference](networking_reference.md).
Section numbers below refer to the paper; interpretations are labeled.

## Compact paper map

| Part | Paper-specific content | Source |
|---|---|---|
| Problem | Existing lossy-RDMA recovery can mistake reordering for loss and depend on timeouts for some losses; PFC-based fabrics have operational and scaling costs. | §2 |
| Goals | Independence from PFC; compatibility with packet-level load balancing; fast ordinary loss recovery without RTO; low hardware memory and processing overhead. | §3 |
| Architecture | Switches preserve trimmed headers; RNICs use those headers for recovery and extend packet reception and tracking. | §4 |
| Main assumption | Header-only packets normally survive and loss notification does not cause spurious retransmissions. | §4.2 and §4.5 |
| Prototype | P4 programmable switch plus an FPGA RNIC derived from an RNIC-GBN baseline; the cited hardware is EdgeCore AS9516 and AMD Alveo U250. | §5, references 2 and 4 |
| Evidence | Directly connected prototype benchmark; 16-RNIC, two-switch 100 Gb/s testbed; larger NS3 simulations. | §§5–6 |
| Limits | Conditional header delivery, fallback timeouts, congestion-control dependence, modified hardware and headers, and unresolved deployment questions for counting. | §4.5, §6.3, and §7 |

## Packet journey

For a congestion drop handled by DCP:

```text
Sender RNIC sends [header + payload]
                    |
Switch finds the data queue above its threshold
                    |
Switch removes payload and preserves a header-only packet
                    |
Protected control queue forwards the header to the receiver
                    |
Receiver rewrites addressing and sends the header back
                    |
Sender identifies the missing packet and schedules retransmission
                    |
Receiver places the retransmitted payload in application memory
```

The header identifies the missing data; it is not a replacement for that data.
The sender fetches the data from host memory, using a retransmission queue and
batched fetches to reduce PCIe overhead. [§4.1 and §4.3](pldi16.pdf)

## How the design connects to the goals

| Goal | Mechanism | Qualification |
|---|---|---|
| Avoid PFC dependence | Trim payloads and schedule the smaller header-only packets separately. | Header-queue protection depends on traffic and scheduling conditions. |
| Tolerate packet-level paths | Recover from explicit loss notifications and put out-of-order payloads directly in the appropriate memory location. | Requires RDMA header extensions and modified RNIC reception logic. |
| Avoid ordinary RTO recovery | Returned header-only packets identify precisely what to retransmit, including tail packets. | A coarse-grained timeout remains for failures or missing notifications. |
| Fit hardware budgets | Batch host-memory fetches and use message-level counters instead of receiver packet bitmaps. | Relies on counting and retry-round assumptions; retains other state and host-memory queues. |

These mappings follow [§§4.2–4.5](pldi16.pdf). DCP supports packet-level load
balancing; it does not require one unique load-balancing algorithm. Its
prototype adds adaptive routing, while rate control is a separate mechanism.

## Evidence to cite carefully

| Experiment | Result | What it establishes |
|---|---|---|
| Controlled 0.01%–5% loss in the hardware testbed | 1.6×–72× improvement in loss-recovery efficiency over CX5, measured through goodput | Recovery performance under this loss-injection setup; DCP trims where CX5 loses packets. |
| Collective jobs in the testbed | Up to 33% lower AllReduce and 42% lower AllToAll job completion time | DCP with adaptive routing versus CX5 with ECMP; the comparison changes both endpoint behavior and routing. |
| Long-running flow across a 10 km optical link | Around 85 Gb/s on 100 Gb/s links | A specific long-distance hardware validation, not a universal file-transfer measurement. |
| Severe incast, Table 5 | One tested setting loses 0.16% of header-only packets without congestion control; tested settings with congestion control lose none | The protected path is robust in these tests but not an unconditional no-loss guarantee. |

Results and qualifications come from [§6.1 and §6.3](pldi16.pdf). The headline
1.6× and 2.1× improvements are not interchangeable with all these individual
measurements. Distinguish goodput, flow completion, and job completion; do not
read exact latency values from Figure 8 without adequate plot resolution.

## Assumptions and limitations

1. **Lossless is qualified.** Footnote 1 says header-only loss is very rare,
   rather than impossible. §4.2 gives a scheduling condition involving the
   data/header size ratio and fan-in; outside that condition, the theoretical
   guarantee does not hold. Table 5 measures a nonzero-loss case.
2. **ACKs are not all protected.** §4.2 puts header-only packets in the control
   queue, while DCP ACK packets can be dropped when the data queue exceeds
   its threshold. Our earlier explanation grouped these too broadly.
3. **Timeouts remain.** §4.5 describes a fallback for lost headers and
   link/switch failures. This is not proof that a permanently broken path or
   failed host can be repaired by the transport.
4. **Counting needs careful semantics.** Without duplicates, packet counts
   establish message completion. Fallback retransmissions introduce retry
   rounds so older and newer arrivals are not simply counted together.
5. **Recovery does not regulate all load.** §6.3 shows poor tail behavior
   without congestion control under severe load. §7 leaves congestion control
   combined with packet-level routing as future work.
6. **Deployment requires cooperation.** Modified switches, RNICs, and packet
   headers must agree. Our inference: compatibility and rollout are important
   practical questions; the prototype is not evidence of a completed commercial
   deployment.

## Deferred question about causes of loss

Keep the user's original question open for a dedicated discussion:
**What causes data loss, and which causes does DCP handle?**

Source-supported distinctions for that discussion:

| Event | What the paper supports |
|---|---|
| Congestion drop at a DCP switch | Main fast path: trim, return the header, retransmit. §4.1 |
| Reordering with no loss | Accept reordered data and avoid inferring loss solely from order. §4.4 |
| Missing header-only notification or link/switch failure | Coarse-grained fallback timeout. §4.5 |
| Physical corruption, silent host-memory corruption, or failed application | This source set does not establish a complete DCP-specific treatment of these failures. Do not infer protection from the word reliability alone. |

The last row is an evidence boundary, not a claim that every such error is
unrecoverable. Also distinguish a transient failure with a usable route from
a permanent loss of connectivity.

## Perspective and questions to investigate

These are our questions, not claimed experimental conclusions:

- How much of a particular improvement comes from recovery versus routing?
- What happens with tiny payloads or fan-in beyond the scheduling bound?
- How robust is arrival counting to duplicates or errors outside the modeled
  retry behavior?
- What can be deployed incrementally with existing RNICs and switches?
- Which guarantees concern packet delivery, RDMA completion, or application
  correctness?

The broader reference contains diagnostic questions and source-checked keys.
User recall and corrections should be recorded after an actual discussion;
they have not been simulated here.

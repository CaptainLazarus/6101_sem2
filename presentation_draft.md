# Revisiting RDMA Reliability for Lossy Fabrics

## Rough 20-minute presentation draft

Paper: Wenxue Li et al., ACM SIGCOMM 2025

Drafting convention: claims attributed to the paper include section/figure references where useful. The diagrams below describe the intended visuals; the editable slide deck supplies the artwork.

---

## Slide 1 — Title and motivating question (0:30)

**Title:** Revisiting RDMA Reliability for Lossy Fabrics

**On-slide points**

- RDMA is fast because the RNIC handles most networking work.
- But high-speed RDMA networks are traditionally designed to be lossless.
- Can we allow packet loss while keeping RDMA fast and hardware-offloaded?

**Speaker notes**

Today we are discussing a transport architecture called DCP. The high-level question is whether an RDMA network really needs to prevent every data-packet loss inside the fabric. The paper argues that we can instead make the data plane lossy, as long as the switch and RNIC cooperate on a lightweight, reliable control plane.

The paper focuses on RoCEv2 over Ethernet, not every possible RDMA technology.

**Diagram suggestion:** A single question in the middle: “Lossless fabric?” on the left versus “Fast recovery at endpoints?” on the right.

---

## Slide 2 — What conventional communication looks like (1:00)

**On-slide points**

- Application calls `send()` / `recv()`.
- Data normally travels through the kernel networking stack.
- The kernel copies or manages buffers, protocols, interrupts, and sockets.
- Reliability is commonly handled by a software transport such as TCP.

**Speaker notes**

Start with the familiar model. A process writes to a socket. The operating system and kernel networking stack process the data, maintain transport state, and eventually put bytes on the network. At the receiver, the reverse path delivers data to the receiving process.

This model is general and convenient, but every packet may involve CPU work, kernel data structures, interrupts, and memory movement. For very high-speed data-centre workloads, those overheads can become a bottleneck.

We should be careful not to say that TCP always copies data multiple times or that sockets cannot use zero-copy; the point is that the conventional abstraction places more work in the host software path than RDMA does.

**Diagram suggestion:** `Application → socket → kernel TCP/IP stack → NIC → switches → NIC → kernel → application`.

---

## Slide 3 — What RDMA changes (1:00)

**On-slide points**

- Remote Direct Memory Access: one host reads/writes a registered memory region on another host.
- The application posts a work request to an RNIC.
- RNIC hardware forms packets, moves data using DMA, handles acknowledgements/retransmissions, and places data in memory.
- CPU involvement is mainly setup, posting work, and completion handling.

**Speaker notes**

RDMA changes the data path rather than simply making TCP faster. The application registers memory and posts a work request to a queue pair. The RNIC translates that request into packets and uses DMA to move data directly between host memory and the network interface.

The remote RNIC can place incoming data directly into the intended memory region. This is why RDMA can offer high throughput and low CPU overhead. The RNIC is not magic: the CPU still creates or posts work requests and eventually handles completion events, but per-packet processing is largely offloaded.

**Diagram suggestion:** `Host A memory → PCIe → RNIC A → Ethernet switches → RNIC B → PCIe → Host B memory`, with the CPU shown mostly above the path rather than inside it.

---

## Slide 4 — RDMA vocabulary and the unit of work (0:45)

**On-slide points**

- RNIC: RDMA-capable network interface/controller.
- QP: Queue Pair; a send queue and a receive queue organize operations.
- WQE: Work Queue Element describing an operation.
- PSN: Packet Sequence Number.
- MSN: Message Sequence Number.
- FCT/JCT: flow/job completion time.

**Speaker notes**

We need a small vocabulary to understand the design. A queue pair is the endpoint state used by RDMA. Completion queues are separate and can be shared by queue pairs. A work request becomes one or more packets. Packet sequence numbers identify individual packets; message sequence numbers identify larger messages or requests.

The distinction matters because DCP tries to avoid tracking every packet with a bitmap. It instead uses message-level counting at the receiver, relying on its retransmission design to preserve an exactly-once property in the normal case.

**Diagram suggestion:** One large message divided into packets labelled PSN 10, 11, 12, 13, with the whole message labelled MSN 7.

---

## Slide 5 — Why RDMA fabrics have traditionally been lossless (1:00)

**On-slide points**

- Original RDMA designs assumed reliable/lossless fabrics.
- Traditional RoCEv2 RNICs use Go-Back-N-style recovery.
- If packet 3 is lost, the sender may resend packet 3 and later packets.
- Packet loss can therefore cause a large throughput penalty.

**Speaker notes**

RDMA was originally designed for lossless InfiniBand-style environments. In the paper’s terminology, traditional RNICs use a Go-Back-N retransmission mechanism. Go-Back-N is hardware-friendly, but it is inefficient when packets really disappear: a loss can force retransmission of subsequent packets that had already arrived.

Because of this, operators often use Ethernet mechanisms to prevent loss before it happens. The most important one here is Priority-based Flow Control, or PFC.

**Diagram suggestion:** Timeline of packets 1–5 where packet 3 disappears and packets 3–5 are retransmitted.

---

## Slide 6 — PFC: making Ethernet look lossless (1:05)

**On-slide points**

- A congested switch queue sends PAUSE upstream.
- Upstream traffic stops until RESUME.
- This protects packets from being dropped.
- But PAUSE is coarse-grained and can spread through the network.

**Speaker notes**

PFC is a hop-by-hop backpressure mechanism. When a queue crosses a threshold, the switch asks upstream senders to pause the relevant priority. When the queue drains, it resumes traffic.

This can prevent packet drops, but it does not make congestion disappear. The paper lists head-of-line blocking, congestion spreading or PFC storms, deadlock, operational complexity, and deployment-distance limitations. A failure or persistent pause can have effects beyond the original congested queue.

The paper also emphasizes buffer headroom. The switch needs enough buffer for packets already in flight after a pause. Its table estimates that commodity switch buffers support only kilometre-scale lossless lengths in some configurations, not an easy path to long-haul communication.

**Diagram suggestion:** Three switches in a line. A full queue at the last switch sends PAUSE arrows backwards, blocking unrelated traffic.

---

## Slide 7 — Why consider a lossy fabric? (0:45)

**On-slide points**

- Remove dependence on PFC.
- Avoid PFC’s blocking, storms, deadlocks, and buffer/headroom constraints.
- Permit more flexible deployment, including longer-distance links.
- But endpoints must recover losses efficiently.

**Speaker notes**

The goal is not to make the final RDMA operation unreliable. “Lossy” describes the fabric: an individual data packet may be dropped. The RDMA operation must still deliver the correct data.

This shifts the problem. We trade switch-side prevention for endpoint-side detection and retransmission. For that to work at 100-Gbps-class rates, recovery must be precise, fast, and implementable in RNIC hardware.

**Diagram suggestion:** Split the network into a lossy data path and a thin reliable control path.

---

## Slide 8 — Existing lossy-RDMA approach: RNIC selective repeat (1:15)

**On-slide points**

- SOTA RNIC-SR designs improve on Go-Back-N.
- Receiver sends selective acknowledgements (SACKs).
- Sender tracks acknowledged/missing packets, often with a bitmap.
- This can retransmit only selected packets.

**Speaker notes**

The paper uses IRN as a representative selective-repeat design. When packets arrive out of order, the receiver sends a SACK containing cumulative and selective information. The sender uses that information and a bitmap to decide what to retransmit.

This is much better than blindly going back to the first loss. However, the paper identifies two problems. First, the mechanism interprets out-of-order arrival as evidence related to loss. Second, some losses do not produce a SACK that can trigger fast recovery and therefore fall back to a retransmission timeout.

**Diagram suggestion:** Receiver sees PSN 1, 3, 4. It sends SACK information for 3 and 4; sender infers 2 is missing.

---

## Slide 9 — Problem 1: packet-level load balancing creates reordering (1:25)

**On-slide points**

- ECMP hashes a flow onto one path.
- Hash collisions can overload one path while others are underused.
- Packet-level load balancing spreads packets across paths.
- Different path delays naturally create out-of-order arrivals.
- The analyzed IRN/RNIC-SR design can mistake reordering for loss and retransmit unnecessarily.

**Speaker notes**

ECMP is flow-level: a flow is assigned to one path based on a hash. It avoids reordering, but unrelated large flows can collide on the same path. Packet-level load balancing can use the fabric more evenly and adapt to congestion or failures, but packets may arrive in a different order.

The paper’s key criticism is that IRN’s SACK-driven logic does not cleanly distinguish actual loss from normal out-of-order delivery under packet-level load balancing. In an NS-3 WebSearch experiment, the authors report spurious retransmissions even when they observed no packet loss. They report that the retransmission ratio could reach 100%; roughly 50%, 80%, and 90% of small, medium, and large flows respectively experienced spurious retransmissions in their reported experiment.

**Diagram suggestion:** Packet 1 uses short path; packet 2 uses long path; packet 3 arrives before packet 2. Mark “reordered, not lost.”

---

## Slide 10 — Problem 2: RTOs hurt tail latency (1:15)

**On-slide points**

- Tail packet loss may produce no later packet to trigger a SACK.
- Retransmitted packets can be lost again during the same congestion episode.
- These cases fall back to retransmission timeouts.
- RTOs are slow and inflate tail completion time.

**Speaker notes**

The second issue is that selective repeat is not always fast. If the final packet of a flow is lost, there may be no later out-of-order packet to generate the signal that starts recovery. Also, if a retransmitted packet is lost again, the sender may need an RTO.

The paper reports excessive timeouts for IRN under both background and incast traffic, with even more timeouts under adaptive routing. In contrast, the DCP flows in that experiment experienced no timeout. We should present this as the paper’s experiment, not as a universal guarantee for every deployment.

**Diagram suggestion:** A flow ending at packet 5; packet 5 is dropped; no packet arrives after it, so the sender waits for an RTO.

---

## Slide 11 — DCP’s four design goals (0:30)

**On-slide points**

1. Independent of PFC.
2. Compatible with packet-level load balancing.
3. Fast retransmission for any lost packet.
4. Hardware-oriented: low memory and processing overhead.

**Speaker notes**

These requirements organize the paper. DCP is not just a new acknowledgement format. It is a co-design between a programmable switch, called DCP-Switch, and an RNIC implementation, called DCP-RNIC.

The central trick is to let the data plane be lossy but preserve a lossless control plane. The switch converts a congested data packet into a small header-only notification, and the RNIC uses that notification to request the exact missing packet.

**Diagram suggestion:** Four boxes around “DCP.”

---

## Slide 12 — DCP overview: lossy data plane, lossless control plane (1:25)

**On-slide points**

- Normal data packet enters DCP-Switch.
- If data queue is below threshold: forward full packet.
- If congested: trim payload, retain header and PSN, mark as HO packet.
- Put HO packet in a control queue.
- WRR scheduling prioritizes the control queue.

**Speaker notes**

DCP-Switch separates the packet’s payload from the information needed for recovery. Under normal conditions, the full data packet follows the data queue. Under congestion, the switch removes the payload and keeps a 57-byte header in the control queue. This is called a header-only, or HO, packet.

The switch uses weighted round-robin scheduling so the control queue gets enough service to remain effectively lossless, while the data queue still makes progress. The paper calls this a lossless control plane. The switch does not need to keep a large amount of per-flow state for this mechanism; the implementation is based on packet trimming and queue scheduling.

Important nuance: the paper says HO loss is very rare under the intended control-plane operation, but not impossible under severe conditions or failures. DCP therefore has a coarse timeout fallback.

**Diagram suggestion:** Full packet → congested switch → “header + PSN” control packet; payload path is dropped, control path continues.

---

## Slide 13 — HO-based precise retransmission (1:10)

**On-slide points**

- Receiver gets an HO packet carrying the missing packet’s identity.
- Receiver swaps source/destination fields and sends the HO back.
- Sender reads the PSN and retransmits exactly that packet.
- Retransmission queue lets congestion control regulate recovery rate.

**Speaker notes**

The header-only packet is effectively an in-network loss notification. At the receiver, the RNIC turns it around by swapping address and queue-pair fields and sends it to the original sender. The sender now knows the precise PSN that needs retransmission.

This avoids inferring loss from out-of-order arrival and avoids waiting for an RTO for ordinary data losses. DCP-RNIC stores retransmission entries, such as message and packet sequence numbers, in a per-QP retransmission queue in host memory. It fetches retransmission entries in batches, reducing PCIe transactions. The retransmission queue also allows the congestion-control module to limit the recovery rate so recovery traffic does not worsen congestion.

**Diagram suggestion:** `Switch trims P2 → receiver sends HO(P2) backwards → sender retransmits P2`.

---

## Slide 14 — Order-tolerant reception and bitmap-free tracking (1:25)

**On-slide points**

- DCP allows packets to arrive out of order.
- Header extensions identify the destination memory address/order context.
- Receiver writes packets directly to their correct memory locations.
- Instead of packet bitmap tracking, count arrivals per message.
- Compare arrival count with the expected packet count to detect completion.

**Speaker notes**

Packet-level load balancing is useful only if the RNIC can tolerate reordering. DCP extends RDMA headers so each packet has enough information for the receiver to place it directly. For writes, the remote memory address information is included in all packets rather than only the first. For two-sided operations, a Send Sequence Number helps match packets to the correct receive work request.

The paper then proposes bitmap-free packet tracking. Because the control plane should cause only genuinely lost packets to be retransmitted, the receiver expects exactly one successfully delivered copy of each packet in the normal case. It can count packets for a message and declare completion when the count reaches the expected packet count. At 400 Gb/s and 10 microseconds RTT, Table 3 reports 32 bytes per QP for DCP tracking versus 320 bytes for a BDP-sized bitmap; for 10,000 QPs it reports 0.3 MB versus 3 MB. These are tracking-state costs, not total RNIC memory.

This is an appealing hardware idea, but it depends strongly on the control-plane and exactly-once assumptions. The paper itself acknowledges substantial real-world deployment challenges.

**Diagram suggestion:** Packets 1, 3, 2 arrive; each is written directly into its memory slot; a message counter reaches 3/3.

---

## Slide 15 — Failure fallback and prototype (1:00)

**On-slide points**

- If the HO/control plane fails, DCP falls back to a coarse-grained timeout.
- `sRetryNo` / receiver retry state prevents confusing timeout rounds.
- DCP-Switch prototype: P4 programmable switch.
- DCP-RNIC prototype: FPGA, 100-Gbps Ethernet, 300-MHz clock.

**Speaker notes**

DCP does not claim that the control plane can never fail. If an HO packet is lost because of a link or switch failure, the sender tracks the smallest unacknowledged message and eventually retransmits all packets in that message after a timeout. Retry numbers in the headers allow the receiver to count packets from the current retry round correctly.

The authors implemented switch behavior in P4 and a fully functional RNIC prototype on an FPGA. Section 5 and Table 4 report relative resource increases over their RNIC-GBN baseline: approximately 1.7% more lookup tables (LUTs), 0.4% more registers, and 1.1% more block RAM (BRAM). These are relative FPGA resource counts, not CPU utilization or percentage points of FPGA capacity.

**Diagram suggestion:** Normal HO fast path above; broken control path below leading to timeout fallback.

---

## Slide 16 — Evaluation setup and headline results (1:05)

**On-slide points**

- Testbed: two P4 switches, 16 FPGA RNICs, 100-Gbps links.
- Simulations: two-layer CLOS, 256 servers, 100-Gbps links.
- Baselines include CX5/RNIC-GBN, PFC, IRN, and MP-RDMA.
- Paper headline: 1.6×–72× better loss-recovery efficiency than CX5 across tested loss rates.

**Speaker notes**

The evaluation has both a hardware testbed and NS-3 simulations. The testbed uses two P4 switches and FPGA-based RNICs. The authors compare the prototype with a Mellanox ConnectX-5 RNIC for several direct tests.

For loss recovery, they inject loss rates from 0.01% to 5%. The paper reports a 1.6× to 72× improvement in goodput or loss-recovery efficiency compared with CX5 over those tested rates. The wide range depends on the loss rate, so we should show the actual figure rather than reduce it to one representative number.

They also report stable goodput with adaptive routing and roughly 85 Gbps over a real 10-km optical link in the testbed. The 100-km and 1000-km scenarios are simulations, not cross-country hardware deployments.

**Diagram suggestion:** Small table with “testbed / simulation / workload / metric.”

---

## Slide 17 — Evaluation: general and AI workloads (1:15)

**On-slide points**

- WebSearch workload: DCP improves tail FCT over IRN and MP-RDMA in reported settings.
- At load 0.3: about 5% lower tail FCT than IRN and 16% lower than MP-RDMA.
- At load 0.5: about 10% lower than IRN and 12% lower than MP-RDMA.
- AI collectives: DCP reduces JCT substantially in AllReduce/AllToAll experiments.

**Speaker notes**

In the large-scale WebSearch simulations, DCP is compared with PFC, IRN, and MP-RDMA. The paper reports lower P95 flow completion time than IRN and MP-RDMA at both tested loads. The exact values are around 5% and 16% at load 0.3, and 10% and 12% at load 0.5, respectively.

For synchronized AI workloads, a single straggling flow can delay the whole collective. The paper reports average AllReduce JCT reductions of 38%, 44%, and 61% relative to MP-RDMA, IRN, and PFC, respectively. For AllToAll, it reports 5%, 45%, and 46% reductions relative to those baselines.

These results are from the paper’s modeled topology and configurations; they are evidence for DCP’s design, not proof that the same percentages hold in every cluster.

**Diagram suggestion:** Two bar charts: P95 FCT for WebSearch; JCT for AllReduce/AllToAll.

---

## Slide 18 — Evaluation: cross-DC and congestion behavior (0:45)

**On-slide points**

- PFC needs buffering/headroom for in-flight packets over long paths.
- DCP is designed to tolerate lossy data traffic without that same PFC dependence.
- Simulations test 100-km and 1000-km-style propagation delays.
- DCP+CC performs best under high load in the paper’s experiments.

**Speaker notes**

The paper evaluates larger propagation delays corresponding to cross-data-centre scenarios. This is important because the paper’s critique of PFC includes distance and headroom constraints. The reported plots show DCP’s advantage increasing in some cross-DC cases, especially at the tail.

The authors also evaluate congestion control and severe incast. They report that DCP’s lossless control plane remains robust under severe incast and that DCP combined with congestion control outperforms the comparisons at high load.

We should avoid claiming that DCP automatically solves wide-area networking. The paper is still about RDMA-style communication and its assumptions about one RDMA domain, timing, switch support, and control-plane behavior need to be discussed.

**Diagram suggestion:** Same topology at low versus high propagation delay, with PFC headroom highlighted.

---

## Slide 19 — Our critique and open questions (1:05)

**On-slide points**

- DCP is a switch/RNIC co-design, so deployment requires coordinated hardware support.
- The “lossless control plane” is an assumption maintained by queue priority, not an absolute guarantee.
- HO packets can still be lost; fallback becomes coarse and expensive.
- Bitmap-free counting depends on exactly-once delivery and message-level metadata.
- Evaluation is promising but does not establish production-scale deployment by itself.

**Speaker notes**

Our main positive assessment is that DCP attacks the actual interaction between load balancing and reliability. It does not simply add another bitmap or another timeout. It changes the signal: the switch explicitly identifies a packet whose payload was removed.

The main concern is complexity of the ecosystem. DCP requires switch pipeline behavior, queue scheduling, packet formats, and RNIC changes to agree. A network operator cannot deploy only one half of the design.

Second, “lossless control plane” should be phrased carefully. WRR can prioritize HO packets, but severe congestion, failures, or implementation limits can still violate the assumption. Table 5 reports 0.16% HO loss in one extreme incast configuration without congestion control, versus no observed HO loss in the tested configurations with congestion control. DCP then falls back to a timeout. Headers are not guaranteed to arrive in order, and ordinary ACK packets use the data queue.

Third, packet counting is elegant but fragile if duplicate packets, delayed packets from old retry rounds, malformed metadata, or multiple failures violate exactly-once behavior. The paper adds retry numbering, but this deserves more stress testing.

Finally, the testbed and simulations are substantial, but we should ask how DCP behaves with heterogeneous vendors, larger numbers of QPs, arbitrary failures, changing routes during retransmission, and real production workloads.

**Diagram suggestion:** A two-column “strengths / questions” slide.

---

## Slide 20 — Takeaways and questions (0:20)

**On-slide points**

- RDMA moves fast-path networking work into RNIC hardware.
- PFC makes RDMA fabrics lossless, but creates operational and scaling costs.
- The analyzed RNIC selective-repeat design struggles with reordering and timeout cases.
- DCP uses a lossless control plane to recover losses in a lossy data plane.
- The key idea: trim payloads, preserve precise loss notifications, and keep recovery hardware-friendly.

**Speaker notes**

The paper’s central contribution is a reliability architecture, not just a retransmission algorithm. DCP lets packet-level load balancing and lossy forwarding coexist with RDMA by separating data and control roles.

The three questions I would leave the audience with are:

1. Is a small, reliable control plane easier to operate than a globally lossless fabric?
2. How robust is bitmap-free counting under repeated failures and retries?
3. What is the realistic path from P4/FPGA prototypes to heterogeneous production RNICs and switches?

**Closing line:** DCP reframes reliability: instead of preventing every data loss in the network, make loss explicit, cheap to report, and fast to repair.

---

## Optional backup slide — Glossary / anticipated questions

**Why not just use TCP?**

TCP provides a general software transport, but the paper’s target is RDMA’s low CPU overhead and direct memory placement. A software recovery mechanism may sacrifice the offload advantage.

**Does lossy mean incorrect data?**

No. It means the network may drop individual data packets. The RDMA operation remains reliable through retransmission.

**Why not use a larger bitmap?**

Bitmaps can be memory-expensive for many active QPs and may require packet-level lookup work. DCP’s message-level counting relies on its exactly-once recovery property.

**What happens if the HO packet is lost?**

DCP uses a coarse-grained timeout fallback and retransmits the unacknowledged message. This is a recovery path, not the normal fast path.

**Is DCP already deployable?**

The paper presents P4 and FPGA prototypes and argues the switch behavior is feasible in ASICs. Production deployment across vendors remains an open practical question.

---

## Timing check

The main slides sum to approximately 20 minutes:

- Background and motivation (Slides 1–10): approximately 10 minutes.
- DCP design (Slides 11–15): approximately 5.5 minutes.
- Evaluation and critique (Slides 16–20): approximately 4.5 minutes.

If the class expects more DCP detail, shorten Slides 2–4 and expand Slides 12–15 using the suggested packet walk-through. If the audience is unfamiliar with networking, keep Slides 2–5 and use the glossary as a backup.

## Source anchors in the paper

- Abstract and Section 1: motivation, goals, DCP overview, headline results.
- Section 2.1: PFC operation, lossless-RDMA limitations, buffer/distance discussion.
- Section 2.2: RNIC-SR, ECMP collision problem, spurious retransmission and RTO experiments.
- Section 3: requirements R1–R4.
- Section 4: DCP-Switch, HO retransmission, order-tolerant reception, bitmap-free tracking.
- Section 5: P4/FPGA implementation and resource usage.
- Section 6 and Figures 8–15: testbed, simulation, workload, cross-DC, and congestion results.

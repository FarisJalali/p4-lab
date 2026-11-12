# Week 5 Assignment — Put a Limit on the Tunnel

**Student ID (last 4 digits only): 1945**

## 1. Analysis: the Loop

I created a forwarding loop between switches s1 and s2 by modifying the tunnel forwarding table on s2. I changed the entry for tunnel destination ID 3 (handle 2) so that packets would be forwarded through port 1, back towards s1.

The command used was:

`sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"`

I then sent a tunnelled packet from h1 with destination IP `10.45.0.3` and tunnel destination ID 3:

`h1 python3 send.py 10.45.0.3 "loop" --tunnel 3`

The initial packet-processing counts were:

- s1: 504
- s2: 502

After another 5 seconds, the counts increased to:

- s1: 671
- s2: 669

Both switches processed an additional 167 packets during the measurement period. This showed that the forwarding loop continued because the tunnel forwarding action did not decrease any hop counter.

**Evidence:** `assignment/evidence/loop-before.txt`

For normal IPv4 forwarding, the `ipv4_forward()` action decreases the IPv4 TTL by 1. However, the program does not explicitly check whether TTL has reached zero before forwarding.

Since TTL is an 8-bit field, subtracting 1 from zero can wrap the value to 255. Therefore, the existing IPv4 forwarding logic does not reliably prevent an IPv4 forwarding loop.

## 2. Design: the Hop Limit

I decided to add a new field called `hop_limit` to the custom tunnel header. This field limits how many times a tunnelled packet can be forwarded through switches.

### Field Size and Position

I chose an 8-bit field (`bit<8>`) and placed it after `dst_id` in `myTunnel_t`.

The updated tunnel header consists of:

- `proto_id`: 16 bits
- `dst_id`: 16 bits
- `hop_limit`: 8 bits

The total tunnel header size is 40 bits (5 bytes).

I chose 8 bits because it provides values from 0 to 255 while adding only one byte of overhead. Placing it after the existing fields keeps the original tunnel destination information in the same position.

### Initial Hop Limit

I chose an initial hop limit of 8, which is sufficient for the small network used in this lab.

The sender uses 8 by default, while allowing another value to be specified through the `--hops` argument.

### Decrement and Drop Behaviour

The hop limit is checked in the ingress pipeline whenever a valid tunnel header is received.

If the hop limit is greater than 1, the switch decreases it by 1 and applies the tunnel forwarding table.

If the hop limit is 1 or 0, the packet is dropped without being forwarded.

With an initial value of 8, a packet can be forwarded through seven switches before the next switch drops it, assuming every switch uses the updated program.

This ensures that a forwarding loop eventually stops rather than continuing indefinitely.

### Changes Required for Compatibility

The P4 parser and deparser must use the updated tunnel header definition so that the hop-limit field is correctly extracted and emitted.

The host applications also required modifications. `send.py` includes the new field when constructing packets, while `receive.py` extracts and displays the received hop limit.

All components must agree on the header layout to prevent incorrect packet interpretation.

## 3. Implementation

I modified three files to implement the hop limit: `tunnel.p4`, `send.py`, and `receive.py`.

### tunnel.p4

First, I added an 8-bit `hop_limit` field to the `myTunnel_t` header, after `dst_id`.

I then modified the `MyIngress.apply()` block to check the hop limit before forwarding a tunnelled packet.

The implementation follows these rules:

- If `hop_limit <= 1`, the packet is dropped using `drop()`.
- Otherwise, the switch subtracts 1 from `hop_limit` and applies `myTunnel_exact`.
- Packets without a tunnel header continue using the normal IPv4 forwarding table.

I used a check-before-decrement approach so that packets with a hop limit of zero cannot wrap around to 255.

The parser extracts the updated tunnel header and the deparser emits it, so the modified value is carried to the next switch.

### send.py

I updated the Scapy `MyTunnel` class by adding `ByteField("hop_limit", 8)`.

I also added a new command-line argument, `--hops`, with a default value of 8. This allows the sender to choose the starting hop limit without modifying the Python code.

The sender validates that the hop-limit value is between 0 and 255 and includes it when constructing tunnelled packets.

For example:

`h1 python3 send.py 10.45.0.3 "hello" --tunnel 2 --hops 4`

This sends a tunnelled packet with destination ID 2 and an initial hop limit of 4.

### receive.py

I updated the receiver's Scapy header definition to match the new 5-byte tunnel header.

I also modified the packet display function to print the remaining hop limit when a tunnelled packet arrives.

This allowed me to verify that the switches were actually decrementing the field rather than simply forwarding packets unchanged.

### Implementation Considerations

One important issue was ensuring that the field order matched across all three files. If the sender, switches, and receiver interpret the header differently, packet parsing can fail.

I also kept `tunnel_v1.p4` unchanged as a copy of the original working program so that it could be used during the mixed-rollout experiment.

The updated P4 program compiled successfully and both switches loaded six forwarding-table entries each.

## 4. Evidence: It Works

I performed several tests to verify that the new hop limit works without breaking normal forwarding.

### 4.1 Tunnelled Delivery to h2 and h3

I sent two tunnelled packets with an initial hop limit of 8.

Commands:

`h1 python3 send.py 10.45.0.3 "delivery h2" --tunnel 2 --hops 8`

`h1 python3 send.py 10.45.0.3 "delivery h3" --tunnel 3 --hops 8`

The results were:

| Destination | Initial Hop Limit | Received Hop Limit | Result |
|---|---:|---:|---|
| h2 | 8 | 6 | Delivered |
| h3 | 8 | 6 | Delivered |

Both packets crossed two switches, s1 and s2, so their hop limits decreased from 8 to 6.

The IPv4 TTL remained at 64 because tunnel forwarding does not modify the IPv4 TTL.

Both receivers successfully displayed the remaining hop-limit value.

### 4.2 Normal IPv4 Forwarding

I also verified that the changes did not break ordinary IPv4 forwarding.

Command:

`h1 ping -c 2 h3`

Result:

- 2 packets transmitted
- 2 packets received
- 0% packet loss
- Reply TTL: 62

This confirmed that the normal IPv4 forwarding path continued working.

### 4.3 Low Hop Limit Test

I sent another tunnelled packet using an initial hop limit of 2.

Command:

`h1 python3 send.py 10.45.0.3 "low hops" --tunnel 2 --hops 2`

The message did not appear in either receiver's log.

Based on the implementation, s1 decremented the hop limit from 2 to 1. When s2 received the packet, its ingress logic checked the value and dropped the packet because the limit was 1.

This result was consistent with the intended drop behaviour.

**Evidence:** `assignment/evidence/delivery.txt`

### 4.4 Forwarding Loop After Implementing the Hop Limit

I repeated the forwarding-loop experiment from Section 1 with the updated program.

I modified s2's tunnel destination 3 entry to forward packets back towards s1.

Command:

`sh ./p4cli s2 "table_modify MyIngress.myTunnel_exact MyIngress.myTunnel_forward 2 1"`

I then sent a packet with an initial hop limit of 8:

`h1 python3 send.py 10.45.0.3 "loop after" --tunnel 3 --hops 8`

The packet-processing counts were:

| Switch | First Count | After 5 Seconds |
|---|---:|---:|
| s1 | 1271 | 1271 |
| s2 | 1268 | 1268 |

The counts did not increase during the second measurement, showing that the loop was no longer active.

The s2 logs also confirmed the drop condition:

`Condition "hdr.myTunnel.hop_limit <= 1" is true`

`Action MyIngress.drop`

`Dropping packet at the end of ingress`

With an initial hop limit of 8, the expected sequence is seven successful forwarding operations followed by a drop on the eighth switch visit.

The drop occurred at s2, which is consistent with the expected sequence because the packet started at s1 and alternated between both switches.

This demonstrates that the new hop-limit mechanism prevents a forwarding loop from continuing indefinitely.

**Evidence:** `assignment/evidence/loop-after.txt`

## 5. Evaluation

### 5(a) Mixed Rollout — Prediction (Before Testing)

Before running the experiment, I predicted that the old switch (s2) might not correctly process packets forwarded by the new switch (s1) because their tunnel header formats were different.

The new program uses a 5-byte tunnel header containing an 8-bit hop limit, while the old program expects a 4-byte header.

Because the new hop-limit field is placed after `dst_id`, I expected s2 to still recognise the tunnel destination ID. However, it might interpret the additional hop-limit byte as part of the IPv4 header, causing incorrect packet parsing or delivery problems.

For the forwarding loop, I predicted that s2 would not decrement the hop limit because it runs the old program. The new switch (s1) should still decrease the hop limit whenever the packet passes through it. Therefore, I expected the loop might eventually stop, although the exact behaviour depended on how the two versions handled the packet format.

### 5(a) Mixed Rollout — Actual Results

I tested a mixed network where s1 used the updated `tunnel.p4` and s2 used the original `tunnel_v1.p4`.

The command used was:

`python3 topo.py --id 1945 --s2-program tunnel_v1.p4`

Both programs compiled successfully. However, when I sent a tunnelled packet to h2, the receiver did not report the message.

I used tcpdump to capture the Ethernet frame at h2. The capture showed the bytes:

`0002 0745`

These bytes represent:

- `0002`: Tunnel destination ID 2
- `07`: Remaining hop limit after s1 decremented 8 to 7
- `45`: Beginning of the original IPv4 header, representing IPv4 version 4 and header length 5

This confirmed my prediction about the header mismatch.

The old switch expects a 4-byte tunnel header and therefore interprets the additional hop-limit byte as part of IPv4.

The packet can still be forwarded using the tunnel destination ID because the destination field is in the same location in both versions. However, the IPv4 header is parsed at the wrong byte offset, so the receiver cannot decode the resulting packet correctly.

The tcpdump capture confirmed that the Ethernet frame physically reached h2 despite the receiver not reporting the UDP message.

I also tested the forwarding loop by changing s2's tunnel destination 3 to port 1.

The packet-processing counts remained unchanged during a second measurement five seconds later:

- s1: 1281 to 1281
- s2: 1277 to 1277

The logs showed that s1 eventually evaluated `hdr.myTunnel.hop_limit <= 1` as true and dropped the packet.

During the mixed-loop experiment, the s1 logs showed seven consecutive false evaluations followed by a true evaluation on the eighth visit to s1.

This result was consistent with s1 decrementing the hop limit once per visit while the old s2 forwarded packets without decrementing it.

Compared with the fully updated network, the mixed network required more switch visits before the packet was dropped. Under this alternating path, the packet made seven successful forwarding visits to s1 and seven to s2 before being dropped on its eighth visit to s1, making 15 switch visits in total.

The experiment confirmed that a mixed network could still stop this loop, but it also demonstrated that the two header versions were not fully compatible.

**Evidence:** `assignment/evidence/mixed.txt`

### 5(b) Safe Rollout Strategy

Updating a packet-header format across an existing network can be risky because not every switch can be upgraded simultaneously.

Based on my experiment, I would avoid sending the new header format until all switches on the intended path can understand it.

I considered two approaches.

**Option 1: Use a new EtherType**

One approach is to assign a different EtherType to the updated tunnel header.

The original header would continue using `0x1212`, while the updated format would use a separate EtherType.

This would allow upgraded switches to distinguish between the old and new header formats immediately.

The main advantage is that the parser can identify the correct header format without guessing its length.

However, old switches would not recognise the new EtherType. Therefore, upgraded switches would need to support both formats, and the new format should only be enabled on paths where every required device supports it.

**Option 2: Add explicit header versioning**

Another approach is to include a version field in the tunnel header.

For example, version 1 could represent the original header, while version 2 could represent the format containing the hop limit.

An upgraded parser could use this field to select the correct header structure.

This would allow both packet formats to coexist during migration.

However, the old switches do not understand a newly introduced version field, so versioning alone would not solve compatibility. The old devices would still need to be upgraded before versioned packets could safely traverse them.

**Comparison and Recommendation**

I prefer using a new EtherType with a staged deployment.

First, I would upgrade the switches and receivers so they can understand both old and new tunnel formats.

Next, I would test the new format on a small network path while keeping the original format as the default.

After confirming compatibility, I would gradually enable the new format on additional paths.

Finally, I would remove support for the old format only after all necessary devices had been upgraded.

This approach reduces the risk of packet corruption and provides a clear rollback option if problems occur.

The mixed-rollout experiment showed why this is important: the switches still forwarded the packet, but the receiver could not correctly interpret its contents.

### 5(c) Data Plane vs Control Plane

Loop protection can be implemented in both the data plane and the control plane, but they address different parts of the problem.

**Data-Plane Protection**

The hop limit implemented in this assignment is a data-plane mechanism.

Each updated switch checks the hop limit when processing a tunnelled packet. If the value is too low, the packet is dropped.

The main advantage is that the protection happens during packet forwarding without needing to communicate with a controller.

It can stop packets from circulating indefinitely even when a forwarding loop has already formed.

However, it does not prevent a loop from being created. It only limits how long a packet can remain in the loop.

It also depends on switches correctly enforcing the hop limit. My mixed-rollout experiment demonstrated that an old switch could forward a packet without decreasing the counter.

**Control-Plane Protection**

In Week 3, the `check_paths()` function was used to examine forwarding paths.

A control-plane path check can identify forwarding loops or invalid paths from the network configuration before traffic is sent.

This is useful because it can detect a problem before packets waste network bandwidth.

However, a control-plane check depends on the forwarding information available when the check is performed.

If a forwarding table changes afterward, or the actual network behaviour differs from the expected configuration, the earlier check may no longer represent the active forwarding paths.

**Comparison**

The control plane tries to prevent incorrect forwarding paths from being installed or used, while the data plane limits the damage when an unexpected forwarding loop occurs.

For example, a control-plane check could detect a configured cycle between s1 and s2 before traffic is sent.

If a loop still occurs because of a runtime table change, the data-plane hop limit can stop the affected packet after a bounded number of forwarding operations.

Real networks benefit from using both mechanisms because they provide complementary protection against forwarding loops.

### 5(d) Hardware Cost

Adding a hop limit introduces some additional processing and packet overhead.

**Header Overhead**

The original tunnel header was 4 bytes, while the updated header is 5 bytes.

This adds one byte to every tunnelled packet, slightly increasing the transmitted data size.

The percentage overhead is larger for small packets than for large packets.

**Processing Overhead**

Each updated switch must perform a few additional operations:

1. Extract the 8-bit hop-limit field during parsing.
2. Compare the value against 1.
3. Drop the packet if the value is too low.
4. Otherwise, decrement the value by 1.
5. Emit the updated field when forwarding.

These operations are relatively simple and do not require complicated calculations.

On a programmable hardware switch, the comparison and subtraction may be implemented using pipeline arithmetic and logic resources.

However, the extra field and operations consume some parser, packet-header storage, and pipeline resources.

**Effect on Line-Rate Performance**

Hardware switches use forwarding pipelines designed to process large numbers of packets.

The additional comparison and decrement may fit within available pipeline stages without reducing throughput, depending on the hardware architecture.

However, if the pipeline is already using most of its available resources, the change could create resource constraints or require additional stages.

Therefore, I would not assume that the modification has zero performance cost without testing it on the target hardware.

**Conclusion**

I consider the additional overhead reasonable because it requires only one extra header byte and a small number of simple operations.

The benefit is that tunnelled packets can no longer circulate indefinitely when the hop limit is correctly enforced.

## AI Use

I used **ChatGPT (OpenAI)** to assist with this assignment.

ChatGPT helped me understand the P4 laboratory instructions, interpret BMv2 switch logs, and understand how the parser, ingress, and deparser work together.

It also helped me design the 8-bit hop-limit mechanism, suggest changes to `tunnel.p4`, `send.py`, and `receive.py`, and troubleshoot issues encountered during implementation.

I also used ChatGPT to help organise and draft sections of this report based on my experimental results.

I applied the code changes in GitHub Codespaces, ran the experiments, inspected the actual results, and checked whether the implementation behaved as expected.

The experimental evidence included in this assignment comes from my own lab runs using student ID suffix 1945.

## Evidence Files

| File | What It Shows |
|---|---|
| `evidence/loop-before.txt` | Packet-processing counts increasing continuously before implementing the hop limit |
| `evidence/delivery.txt` | Successful tunnelled delivery to h2 and h3, received hop limits, IPv4 ping, and low-hop test |
| `evidence/loop-after.txt` | Loop termination after implementing the hop limit, including the s2 drop evidence |
| `evidence/mixed.txt` | Mixed-version deployment, packet capture, forwarding-loop results, and s1 drop logs |
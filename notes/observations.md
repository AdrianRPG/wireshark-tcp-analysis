# TCP Analysis Observations

## Lab Context

This analysis was completed using Wireshark and the `tcp-wireshark-trace1-1` packet capture associated with the TCP lab from *Computer Networking: A Top-Down Approach*.

The trace captures the transfer of an approximately 150 KB text file to the `gaia.cs.umass.edu` web server using an HTTP POST request over TCP.

The analysis focused on:

- TCP connection establishment
- Sequence and acknowledgment numbers
- Application-data segmentation
- Round-trip time (RTT)
- Estimated RTT
- Receiver-advertised flow control
- TCP window scaling
- Retransmission behavior
- Acknowledgment behavior
- Throughput
- TCP congestion control and slow start

---

## Network Endpoints

The TCP connection was established between the following endpoints:

| Role | IP Address | TCP Port |
|---|---|---:|
| Client | `192.168.86.68` | `55639` |
| Server | `128.119.245.12` | `80` |

The client used ephemeral TCP port `55639`, while the remote HTTP server communicated through TCP port `80`.

The connection can therefore be represented as:

```text
Client
192.168.86.68:55639
        |
        | TCP / HTTP
        v
gaia.cs.umass.edu
128.119.245.12:80
```

---

## TCP Three-Way Handshake

The connection begins with the standard TCP three-way handshake:

```text
Client                                  Server
192.168.86.68                           128.119.245.12

      SYN
      ---------------------------------->

                     SYN, ACK
      <----------------------------------

      ACK
      ---------------------------------->
```

### Client SYN

The client initiated the TCP connection using a segment with the SYN flag enabled.

Raw initial sequence number:

```text
4236649187
```

Wireshark displayed:

```text
Flags: SYN
```

The SYN segment also advertised support for Selective Acknowledgment:

```text
SACK_PERM
```

This indicates that Selective Acknowledgments could be used during the TCP session.

### Server SYN-ACK

The server responded with a TCP SYN-ACK segment.

Raw server sequence number:

```text
1068969752
```

Raw acknowledgment number:

```text
4236649188
```

The acknowledgment value is one greater than the client's initial sequence number:

```text
4236649187 + 1 = 4236649188
```

This occurs because a TCP SYN consumes one sequence number even though it does not carry an application payload.

The server segment had both TCP flags enabled:

```text
SYN = 1
ACK = 1
```

---

## Sequence and Acknowledgment Numbers

TCP sequence numbers identify positions in the byte stream rather than individual packets.

The client's SYN used the following raw sequence number:

```text
4236649187
```

Because the SYN consumes one sequence number, the first application-data byte began at:

```text
4236649188
```

The server acknowledged the SYN using:

```text
ACK = 4236649188
```

This value represents the sequence number of the next byte the server expects to receive from the client.

---

## HTTP POST and TCP Segmentation

The application-layer HTTP POST message contained the uploaded text file.

Because the file was significantly larger than the amount of data that could fit inside a single TCP segment, the HTTP message was divided across many TCP segments before transmission.

An important observation during the analysis was that the packet Wireshark associates with the fully reassembled HTTP POST is not necessarily the TCP segment containing the beginning of the HTTP request.

The first TCP segment containing the ASCII string:

```text
POST
```

was packet:

```text
4
```

The payload contained the hexadecimal byte sequence:

```text
50 4f 53 54
```

These values correspond to the ASCII characters:

```text
50 = P
4f = O
53 = S
54 = T
```

Therefore:

```text
50 4f 53 54
P  O  S  T
```

This demonstrates the distinction between:

- An individual TCP segment carrying part of an application message
- Wireshark's reassembly of many TCP segments into a complete HTTP message

---

## First Data-Carrying TCP Segment

The first TCP segment containing the beginning of the HTTP POST was packet `4`.

Its relative sequence number was:

```text
1
```

Its raw sequence number was:

```text
4236649188
```

The TCP payload size was:

```text
1448 bytes
```

The observed TCP header size was:

```text
32 bytes
```

Therefore, at the TCP layer:

```text
TCP header  =   32 bytes
TCP payload = 1448 bytes
-----------------------
Total       = 1480 bytes
```

The IPv4 header added another:

```text
20 bytes
```

giving a complete IPv4 packet size of:

```text
1480 + 20 = 1500 bytes
```

This can be summarized as:

| Component | Size |
|---|---:|
| TCP payload | `1448 bytes` |
| TCP header | `32 bytes` |
| TCP header + payload | `1480 bytes` |
| IPv4 header | `20 bytes` |
| IPv4 packet | `1500 bytes` |

Wireshark displayed `Len=1448` for these data-carrying TCP packets, corresponding to the TCP payload length.

---

## Round-Trip Time Analysis

Round-Trip Time (RTT) measures the amount of time between transmitting a TCP segment and receiving the acknowledgment associated with that data.

### First Data-Carrying Segment

The first data-carrying TCP segment was transmitted at:

```text
0.024047 s
```

The corresponding ACK was received at:

```text
0.052671 s
```

The measured RTT was therefore:

```text
RTT = ACK arrival time - segment transmission time
```

```text
RTT = 0.052671 - 0.024047
```

```text
RTT = 0.028624 s
```

Equivalent to:

```text
28.624 ms
```

---

## Second Data-Carrying Segment RTT

The second client-to-server data segment was transmitted at:

```text
0.024048 s
```

Its corresponding acknowledgment was received at:

```text
0.052676 s
```

Therefore:

```text
RTT = 0.052676 - 0.024048
```

```text
RTT = 0.028628 s
```

Equivalent to:

```text
28.628 ms
```

The first two measured RTT values were therefore very similar:

| Sample | RTT |
|---|---:|
| First segment | `28.624 ms` |
| Second segment | `28.628 ms` |

---

## Estimated RTT

TCP does not normally base timing decisions on only one RTT measurement.

Instead, an EstimatedRTT can be calculated using an exponentially weighted moving average:

```text
EstimatedRTT =
(1 - α)(Previous EstimatedRTT)
+ α(SampleRTT)
```

For this analysis:

```text
α = 0.125
```

The first measured RTT was used as the initial estimate:

```text
Previous EstimatedRTT = 0.028624 s
```

The second RTT sample was:

```text
SampleRTT = 0.028628 s
```

Therefore:

```text
EstimatedRTT
= (1 - 0.125)(0.028624)
+ (0.125)(0.028628)
```

```text
EstimatedRTT
= (0.875)(0.028624)
+ (0.125)(0.028628)
```

```text
EstimatedRTT
= 0.0286245 s
```

This is approximately:

```text
28.6245 ms
```

---

## TCP Receiver Flow Control

TCP implements receiver-side flow control using the receive-window field.

The server advertises how much additional data it is prepared to receive, preventing the sender from overwhelming its receive buffer.

The connection also negotiated TCP Window Scaling, allowing the effective receive window to exceed the maximum value that could otherwise be represented by the original 16-bit TCP window field.

The server used a window scaling factor of:

```text
128
```

### Initial Advertised Windows

The first observed server acknowledgments showed:

| ACK Number | Window Size Value | Scale Factor | Calculated Window |
|---:|---:|---:|---:|
| `1449` | `249` | `128` | `31872 bytes` |
| `2897` | `272` | `128` | `34816 bytes` |
| `4345` | `295` | `128` | `37760 bytes` |
| `5793` | `317` | `128` | `40576 bytes` |

The minimum raw Window Size Value among these ACKs was:

```text
249
```

Applying the negotiated scaling factor:

```text
249 × 128 = 31872 bytes
```

Therefore, the smallest effective receive window observed among these initial ACKs was:

```text
31872 bytes
```

---

## Receiver Buffer Analysis

Approximately twenty 1500-byte packets would require:

```text
20 × 1500 = 30000 bytes
```

The minimum scaled receive window was:

```text
31872 bytes
```

Since:

```text
31872 > 30000
```

the receiver had sufficient available buffer space for more than twenty approximately 1500-byte packets.

Therefore, receiver-buffer limitations did not throttle the sender during the initial portion of the transfer.

---

## TCP Acknowledgment Behavior

The initial data-carrying TCP segments each contained:

```text
1448 bytes
```

The corresponding acknowledgment numbers included:

```text
1449
2897
4345
5793
```

The differences between consecutive ACK values were:

```text
2897 - 1449 = 1448
```

```text
4345 - 2897 = 1448
```

```text
5793 - 4345 = 1448
```

This shows that these acknowledgments advanced the expected byte position by:

```text
1448 bytes
```

TCP acknowledgments are cumulative and indicate the sequence number of the next byte expected by the receiver.

---

## Retransmission Analysis

No TCP retransmissions were observed in the analyzed trace.

One Wireshark display filter that can be used to identify retransmissions is:

```text
tcp.analysis.retransmission || tcp.analysis.fast_retransmission
```

The client-to-server sequence numbers were also examined for repeated or overlapping data ranges.

The data sequence numbers continued progressing forward without evidence of previously transmitted payload ranges being sent again.

This provided additional evidence that retransmissions did not occur during the analyzed transfer.

---

## TCP Conversation Statistics

Wireshark's:

```text
Statistics → Conversations → TCP
```

view reported the connection between:

```text
192.168.86.68:55639
```

and:

```text
128.119.245.12:80
```

The conversation statistics included approximately:

```text
Packets A → B: 109
Bytes A → B:   161 kB
Packets B → A: 71
Bytes B → A:   5 kB
Duration:      0.1927 s
Bits/s A → B:  6667 kbps
Bits/s B → A:  227 kbps
```

Wireshark therefore reported approximately:

```text
6.667 Mb/s
```

for the client-to-server direction over the full conversation interval used by the Conversations view.

---

## Application Data Throughput

Throughput can also be calculated manually as:

```text
Throughput = Data transferred / Transfer time
```

The first TCP segment carrying the beginning of the POST was transmitted at approximately:

```text
0.024047 s
```

The final segment associated with the reassembled HTTP POST appeared at approximately:

```text
0.147682 s
```

The approximate transfer interval was therefore:

```text
0.147682 - 0.024047
```

```text
= 0.123635 s
```

The reassembled HTTP data was approximately:

```text
153425 bytes
```

An approximate application-data throughput is therefore:

```text
153425 / 0.123635
```

```text
≈ 1.24 million bytes per second
```

or approximately:

```text
1.24 MB/s
```

Expressed in bits per second:

```text
1.24 MB/s × 8
≈ 9.9 Mb/s
```

This value differs from the Wireshark Conversations statistic because the calculations use different byte counts and measurement intervals.

This illustrates an important networking-analysis principle:

> Throughput measurements should always identify exactly what traffic is being counted and what time interval is being measured.

---

## TCP Congestion Control

The TCP Time-Sequence Graph (Stevens) can be used to visualize how sequence numbers increase over time.

During the beginning of the transfer, groups or "fleets" of TCP segments were observed around:

```text
t ≈ 0.025 s
t ≈ 0.053 s
t ≈ 0.082 s
t ≈ 0.100 s
```

The approximate number of segments transmitted in successive fleets followed the pattern:

```text
3 → 6 → 12 → 24
```

This pattern is consistent with TCP's **slow start** behavior.

During slow start, the sender's congestion window increases rapidly as acknowledgments arrive, allowing progressively larger groups of TCP segments to be transmitted.

The approximate doubling behavior:

```text
3
6
12
24
```

illustrates this exponential increase.

---

## Fleet Periodicity and RTT

The groups of transmitted segments also appeared at roughly periodic intervals.

The spacing between these fleets was of the same general magnitude as the measured TCP RTT.

Measured RTT values were approximately:

```text
28.6 ms
```

This behavior can be explained by TCP's acknowledgment-driven transmission process:

```text
Send group of segments
        ↓
Segments reach receiver
        ↓
ACKs return to sender
        ↓
Congestion window increases
        ↓
Sender transmits a larger group
```

As a result, new groups of segments become eligible for transmission after acknowledgments return from the receiver.

---

## Key Technical Findings

This analysis produced the following key observations:

1. The client established a TCP connection from `192.168.86.68:55639` to `128.119.245.12:80`.

2. The connection used the standard TCP three-way handshake:
   ```text
   SYN → SYN-ACK → ACK
   ```

3. The client's raw initial sequence number was:
   ```text
   4236649187
   ```

4. The server acknowledged the SYN using:
   ```text
   4236649188
   ```

5. Selective Acknowledgment support was advertised during connection establishment.

6. The HTTP POST request was segmented across many TCP segments.

7. Packet `4` contained the beginning of the HTTP POST request.

8. The ASCII string `POST` was verified directly from:
   ```text
   50 4f 53 54
   ```

9. The initial TCP data segments carried:
   ```text
   1448 bytes
   ```
   of TCP payload.

10. The first measured RTT was:
    ```text
    28.624 ms
    ```

11. The second measured RTT was:
    ```text
    28.628 ms
    ```

12. The resulting EstimatedRTT was:
    ```text
    28.6245 ms
    ```

13. The minimum observed raw receiver window value was:
    ```text
    249
    ```

14. After window scaling, this represented:
    ```text
    31872 bytes
    ```

15. Receiver buffer availability did not throttle the sender during the initial transfer.

16. No TCP retransmissions were observed.

17. Initial ACK values advanced by approximately:
    ```text
    1448 bytes
    ```

18. The Time-Sequence Graph showed behavior consistent with TCP slow start.

19. The observed transmission fleets approximately followed:
    ```text
    3 → 6 → 12 → 24
    ```

20. Throughput values varied depending on the traffic scope and time interval used in the calculation.

---

## Skills Demonstrated

This lab provided hands-on experience with:

- Wireshark
- Packet capture analysis
- TCP/IP
- HTTP over TCP
- TCP three-way handshake
- TCP flags
- Sequence numbers
- Acknowledgment numbers
- TCP payload inspection
- Hexadecimal and ASCII packet analysis
- TCP segmentation
- Application-layer reassembly
- Round-Trip Time analysis
- Estimated RTT calculations
- TCP receive windows
- Window scaling
- TCP flow control
- Retransmission analysis
- TCP throughput analysis
- TCP congestion control
- TCP slow start
- Time-Sequence Graph analysis
- Network troubleshooting methodology

---

## Academic Context

This analysis originated from coursework completed for:

**CNT 4713 – Net-Centric Computing**  
**Florida International University**

The project repository documents my own packet analysis, technical observations, calculations, and interpretation of TCP behavior. The packet analysis is presented for portfolio purposes to demonstrate practical understanding of TCP behavior and network protocol analysis.

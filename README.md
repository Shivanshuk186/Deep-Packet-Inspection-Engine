# DPI Engine — Deep Packet Inspection System

A C++17-based Deep Packet Inspection (DPI) engine for analyzing PCAP network captures, parsing network protocols, tracking flows, extracting TLS SNI/HTTP host information, classifying application traffic, applying blocking rules, and generating filtered PCAP output.

The project includes both a **single-threaded implementation** for straightforward analysis and a **multi-threaded DPI pipeline** designed to process larger captures using load balancers, fast-path workers, thread-safe queues, and flow-aware packet distribution.

---

## Table of Contents

1. [What is DPI?](#1-what-is-dpi)
2. [Networking Background](#2-networking-background)
3. [Project Overview](#3-project-overview)
4. [File Structure](#4-file-structure)
5. [The Journey of a Packet — Simple Version](#5-the-journey-of-a-packet--simple-version)
6. [The Journey of a Packet — Multi-threaded Version](#6-the-journey-of-a-packet--multi-threaded-version)
7. [Deep Dive: Each Component](#7-deep-dive-each-component)
8. [How SNI Extraction Works](#8-how-sni-extraction-works)
9. [How Blocking Works](#9-how-blocking-works)
10. [Building and Running](#10-building-and-running)
11. [Understanding the Output](#11-understanding-the-output)
12. [Extending the Project](#12-extending-the-project)

---

# 1. What is DPI?

**Deep Packet Inspection (DPI)** is a network traffic analysis technique that examines packet headers and, where available, application-layer information.

A traditional network filter may primarily use:

* Source IP
* Destination IP
* Source port
* Destination port
* Transport protocol

A DPI system can inspect deeper protocol information to identify traffic characteristics and classify connections.

### Typical Uses

* **Network monitoring** — Analyze traffic patterns and protocols
* **Enterprise networking** — Apply application/domain-based traffic policies
* **Security systems** — Detect suspicious or unwanted traffic
* **Traffic management** — Classify and control network usage
* **Research and education** — Study network protocols and packet processing

### What the DPI Engine Does

```text
Input PCAP
    │
    ▼
┌───────────────────────┐
│      DPI Engine       │
│                       │
│  • Parse packets      │
│  • Track flows        │
│  • Inspect protocols  │
│  • Extract SNI/Host   │
│  • Classify traffic   │
│  • Apply rules        │
│  • Generate statistics│
└───────────┬───────────┘
            │
            ▼
     Filtered PCAP
```

The engine operates on **PCAP files**, making it possible to analyze previously captured network traffic without requiring live packet capture.

---

# 2. Networking Background

## The Network Stack

Network communication involves multiple protocol layers:

```text
┌─────────────────────────────────────────────────────────┐
│ Layer 7: Application    │ HTTP, TLS, DNS               │
├─────────────────────────────────────────────────────────┤
│ Layer 4: Transport      │ TCP, UDP                     │
├─────────────────────────────────────────────────────────┤
│ Layer 3: Network        │ IPv4                         │
├─────────────────────────────────────────────────────────┤
│ Layer 2: Data Link      │ Ethernet / MAC addresses     │
└─────────────────────────────────────────────────────────┘
```

The DPI engine processes these layers from the lower-level packet representation toward application-layer information.

---

## A Packet's Structure

A typical Ethernet + IPv4 + TCP packet can be visualized as:

```text
┌──────────────────────────────────────────────────────────────┐
│ Ethernet Header                                              │
│ Destination MAC | Source MAC | EtherType                    │
├──────────────────────────────────────────────────────────────┤
│ IPv4 Header                                                  │
│ Source IP | Destination IP | Protocol | TTL | ...           │
├──────────────────────────────────────────────────────────────┤
│ TCP Header                                                   │
│ Source Port | Destination Port | Seq | ACK | Flags | ...    │
├──────────────────────────────────────────────────────────────┤
│ Application Payload                                          │
│ HTTP / TLS / other application data                         │
└──────────────────────────────────────────────────────────────┘
```

The exact size of the IP and TCP headers can vary because optional fields may be present. The common minimum sizes are:

* Ethernet header: **14 bytes**
* IPv4 header: **20 bytes**
* TCP header: **20 bytes**

---

## The Five-Tuple

A network flow can be identified using five values:

| Field            | Example          | Purpose             |
| ---------------- | ---------------- | ------------------- |
| Source IP        | `192.168.1.100`  | Sender              |
| Destination IP   | `172.217.14.206` | Receiver            |
| Source Port      | `54321`          | Source endpoint     |
| Destination Port | `443`            | Destination service |
| Protocol         | `TCP (6)`        | Transport protocol  |

Conceptually:

```text
(Source IP,
 Destination IP,
 Source Port,
 Destination Port,
 Protocol)
```

### Why is the Five-Tuple Important?

The DPI engine uses it as the key for flow state.

```text
Five-Tuple
     │
     ▼
┌───────────────┐
│   Flow Table  │
└───────────────┘
     │
     ▼
   Flow State
```

This allows packets belonging to the same flow to share information such as:

* Detected SNI
* Application classification
* Blocked state
* Flow statistics

---

## What is SNI?

**Server Name Indication (SNI)** is a TLS extension that can contain the hostname a TLS client is attempting to connect to.

For example:

```text
Client
   │
   │ TLS Client Hello
   │ SNI = www.example.com
   ▼
Server
```

The DPI engine uses the SNI hostname, when available, as an application/domain classification signal.

### Important Limitation

SNI visibility should **not** be assumed for every modern HTTPS connection. TLS deployments can use privacy mechanisms such as **Encrypted Client Hello (ECH)**, and traffic can use protocols such as **QUIC/HTTP/3**.

Therefore, SNI-based classification is one inspection technique rather than a universal method of identifying encrypted traffic.

---

# 3. Project Overview

## Architecture

```text
                    Input PCAP
                       │
                       ▼
                ┌──────────────┐
                │  PCAP Reader │
                └──────┬───────┘
                       │
                       ▼
                ┌──────────────┐
                │ Packet Parser│
                └──────┬───────┘
                       │
                       ▼
                 Five-Tuple
                       │
                       ▼
                  Flow Table
                       │
                       ▼
              TLS / HTTP Inspection
                       │
                       ▼
                Classification
                       │
                       ▼
                Rule Evaluation
                  /          \
                 ▼            ▼
              DROP         FORWARD
                              │
                              ▼
                        Output PCAP
```

---

## Two Implementations

| Version                  | File                   | Purpose                                            |
| ------------------------ | ---------------------- | -------------------------------------------------- |
| Simple / Single-threaded | `src/main_working.cpp` | Straightforward packet-processing pipeline         |
| Multi-threaded           | `src/dpi_mt.cpp`       | Parallel packet processing using LB and FP workers |

The single-threaded version is useful for understanding the core processing pipeline. The multi-threaded version extends the same concepts into a concurrent architecture.

---

# 4. File Structure

```text
packet_analyzer/
│
├── include/
│   ├── pcap_reader.h
│   ├── packet_parser.h
│   ├── sni_extractor.h
│   ├── types.h
│   ├── rule_manager.h
│   ├── connection_tracker.h
│   ├── load_balancer.h
│   ├── fast_path.h
│   ├── thread_safe_queue.h
│   └── dpi_engine.h
│
├── src/
│   ├── pcap_reader.cpp
│   ├── packet_parser.cpp
│   ├── sni_extractor.cpp
│   ├── types.cpp
│   ├── main_working.cpp
│   └── dpi_mt.cpp
│
├── generate_test_pcap.py
├── test_dpi.pcap
├── output.pcap
└── README.md
```

### Component Overview

| Component               | Responsibility                                |
| ----------------------- | --------------------------------------------- |
| `pcap_reader`           | Reads PCAP headers and packet data            |
| `packet_parser`         | Parses Ethernet, IPv4, TCP and UDP headers    |
| `sni_extractor`         | Extracts TLS SNI and HTTP Host information    |
| `types`                 | Defines core structures and application types |
| `rule_manager`          | Maintains traffic blocking rules              |
| `connection_tracker`    | Tracks flow state                             |
| `load_balancer`         | Distributes packets across workers            |
| `fast_path`             | Performs DPI and rule evaluation              |
| `thread_safe_queue`     | Synchronizes communication between threads    |
| `dpi_engine`            | Coordinates the multi-threaded pipeline       |
| `main_working.cpp`      | Single-threaded implementation                |
| `dpi_mt.cpp`            | Multi-threaded implementation                 |
| `generate_test_pcap.py` | Generates sample PCAP test data               |

---

# 5. The Journey of a Packet — Simple Version

The single-threaded implementation in `main_working.cpp` processes packets sequentially.

```text
PCAP
 │
 ▼
Read Packet
 │
 ▼
Parse Ethernet
 │
 ▼
Parse IPv4
 │
 ▼
Parse TCP / UDP
 │
 ▼
Create Five-Tuple
 │
 ▼
Find / Create Flow
 │
 ▼
Inspect TLS / HTTP
 │
 ▼
Classify Application
 │
 ▼
Apply Rules
 │
 ├── BLOCK → Drop
 │
 └── ALLOW → Output PCAP
```

---

## Step 1: Read PCAP File

```cpp
PcapReader reader;

reader.open("capture.pcap");
```

The reader:

1. Opens the PCAP file in binary mode.
2. Reads the global header.
3. Validates the capture metadata.
4. Prepares the reader for packet-by-packet processing.

### PCAP Layout

```text
┌────────────────────────────┐
│ Global Header (24 bytes)   │
├────────────────────────────┤
│ Packet Header (16 bytes)   │
│ Packet Data (variable)     │
├────────────────────────────┤
│ Packet Header (16 bytes)   │
│ Packet Data (variable)     │
├────────────────────────────┤
│ ... more packets ...       │
└────────────────────────────┘
```

---

## Step 2: Read Each Packet

```cpp
while (reader.readNextPacket(raw)) {
    // raw.data contains packet bytes
    // raw.header contains packet metadata
}
```

For every packet:

1. Read the packet header.
2. Read `incl_len` bytes of packet data.
3. Return the packet to the processing pipeline.
4. Return `false` when the PCAP has been exhausted.

---

# Step 3: Parse Protocol Headers

```cpp
PacketParser::parse(raw, parsed);
```

A typical Ethernet + IPv4 + TCP packet can be represented as:

```text
[0 ... 13]    Ethernet Header
[14 ...]      IPv4 Header
[...]         TCP Header
[...]         Application Payload
```

After parsing, the packet contains information such as:

```text
Source MAC
Destination MAC
Source IP
Destination IP
Source Port
Destination Port
Protocol
TCP/UDP information
Payload
```

---

## Ethernet Header

Common Ethernet header fields:

```text
Bytes 0-5:    Destination MAC
Bytes 6-11:   Source MAC
Bytes 12-13:  EtherType
```

For example:

```text
EtherType = 0x0800
```

indicates IPv4.

---

## IPv4 Header

Important fields include:

```text
Version
IHL
TTL
Protocol
Source IP
Destination IP
```

The protocol field identifies the transport protocol:

```text
6  → TCP
17 → UDP
```

---

## TCP Header

Important TCP fields include:

```text
Source Port
Destination Port
Sequence Number
Acknowledgment Number
Data Offset
Flags
```

TCP flags include:

```text
SYN
ACK
FIN
RST
PSH
URG
```

---

# Step 4: Create Five-Tuple and Look Up Flow

```cpp
FiveTuple tuple;

tuple.src_ip = parseIP(parsed.src_ip);
tuple.dst_ip = parseIP(parsed.dest_ip);

tuple.src_port = parsed.src_port;
tuple.dst_port = parsed.dst_port;
tuple.protocol = parsed.protocol;

Flow& flow = flows[tuple];
```

The flow table concept is:

```text
unordered_map<FiveTuple, Flow>
```

If the tuple already exists:

```text
Existing Flow
```

If it does not exist:

```text
Create New Flow
```

This gives every connection persistent state while the capture is being processed.

---

# Step 5: Extract SNI

For TLS traffic, the engine attempts to inspect the Client Hello.

Conceptually:

```cpp
if (pkt.tuple.dst_port == 443 &&
    pkt.payload_length > 5) {

    auto sni = SNIExtractor::extract(
        payload,
        payload_length
    );

    if (sni) {
        flow.sni = *sni;
        flow.app_type = sniToAppType(*sni);
    }
}
```

The SNI extractor:

1. Checks the TLS record.
2. Checks for a Client Hello.
3. Navigates through the Client Hello fields.
4. Iterates through TLS extensions.
5. Finds extension type `0x0000`.
6. Extracts the hostname.

Example:

```text
TLS Client Hello
      │
      ▼
SNI Extension
      │
      ▼
www.example.com
      │
      ▼
Application Classification
```

---

# Step 6: Check Blocking Rules

```cpp
if (rules.isBlocked(
        tuple.src_ip,
        flow.app_type,
        flow.sni)) {

    flow.blocked = true;
}
```

Rules can be based on:

* Source IP
* Application type
* Domain/SNI

Example:

```cpp
blocked_ips
blocked_apps
blocked_domains
```

---

# Step 7: Forward or Drop

```cpp
if (flow.blocked) {

    dropped++;

} else {

    forwarded++;

    output.write(packet_header);
    output.write(packet_data);
}
```

The output PCAP contains packets that pass the configured filtering rules.

---

# Step 8: Generate Report

After processing, flow and packet statistics can be summarized.

For example:

```text
Total Packets: 1000
Forwarded:      850
Dropped:        150

Application Breakdown:

HTTPS       500
DNS         100
YouTube      80
Facebook     40
Unknown     280
```

---

# 6. The Journey of a Packet — Multi-threaded Version

The multi-threaded implementation introduces parallel processing.

## Architecture

```text
                         Reader Thread
                              │
                              ▼
                     ┌────────────────┐
                     │ Hash FiveTuple │
                     └───────┬────────┘
                             │
                 ┌───────────┴───────────┐
                 ▼                       ▼
          ┌─────────────┐         ┌─────────────┐
          │ LB0 Thread  │         │ LB1 Thread  │
          │ Load Balancer│        │ Load Balancer│
          └──────┬──────┘         └──────┬──────┘
                 │                       │
            ┌────┴────┐             ┌────┴────┐
            ▼         ▼             ▼         ▼
          FP0       FP1            FP2       FP3
        Fast Path Fast Path      Fast Path Fast Path
            │         │             │         │
            └─────────┴──────┬──────┴─────────┘
                             ▼
                       Output Queue
                             │
                             ▼
                    Output Writer Thread
                             │
                             ▼
                       Output PCAP
```

---

## Why This Design?

### Load Balancers

Distribute packets between Fast Path workers.

### Fast Paths

Perform the actual packet inspection, flow tracking, classification and rule evaluation.

### Flow-Aware Hashing

Packets belonging to the same five-tuple are mapped consistently to the same Fast Path.

This is important because each Fast Path maintains flow state.

---

## Why Same Flow → Same Fast Path?

Consider:

```text
Connection:
192.168.1.100:54321
        ↓
142.250.185.206:443
```

Without flow-aware distribution:

```text
Packet 1 → FP0
Packet 2 → FP3
Packet 3 → FP1
Packet 4 → FP2
```

Flow state would be distributed across multiple workers.

With hashing:

```text
hash(FiveTuple) % num_fps
```

the packets are consistently routed:

```text
Packet 1 → FP2
Packet 2 → FP2
Packet 3 → FP2
Packet 4 → FP2
```

FP2 can therefore maintain the state of that flow locally.

---

## Step 1: Reader Thread

```cpp
while (reader.readNextPacket(raw)) {

    Packet pkt = createPacket(raw);

    size_t lb_idx =
        hash(pkt.tuple) % num_lbs;

    lbs_[lb_idx]->queue().push(pkt);
}
```

The reader:

1. Reads the packet.
2. Parses enough information to obtain the flow key.
3. Hashes the five-tuple.
4. Selects a Load Balancer.
5. Pushes the packet into that LB's queue.

---

## Step 2: Load Balancer Thread

```cpp
void LoadBalancer::run() {

    while (running_) {

        auto pkt = input_queue_.pop();

        size_t fp_idx =
            hash(pkt.tuple) % num_fps_;

        fps_[fp_idx]->queue().push(pkt);
    }
}
```

The Load Balancer:

```text
Input Queue
     │
     ▼
Pop Packet
     │
     ▼
Hash Five-Tuple
     │
     ▼
Select Fast Path
     │
     ▼
Push Packet
```

---

## Step 3: Fast Path Thread

```cpp
void FastPath::run() {

    while (running_) {

        auto pkt = input_queue_.pop();

        Flow& flow = flows_[pkt.tuple];

        classifyFlow(pkt, flow);

        if (rules_->isBlocked(
                pkt.tuple.src_ip,
                flow.app_type,
                flow.sni)) {

            stats_->dropped++;

        } else {

            output_queue_->push(pkt);
        }
    }
}
```

The Fast Path is responsible for:

```text
Packet
  ↓
Flow Lookup
  ↓
Classification
  ↓
Rule Evaluation
  ↓
DROP / FORWARD
```

---

## Step 4: Output Writer Thread

```cpp
void outputThread() {

    while (running_ || output_queue_.size() > 0) {

        auto pkt = output_queue_.pop();

        output_file.write(packet_header);
        output_file.write(pkt.data);
    }
}
```

A dedicated writer avoids having multiple processing threads directly coordinate writes to the same output file.

---

# Thread-Safe Queue

The multi-threaded architecture uses a producer-consumer queue.

Conceptually:

```cpp
template<typename T>
class TSQueue {

    std::queue<T> queue_;
    std::mutex mutex_;
    std::condition_variable not_empty_;

    // push / pop operations
};
```

### Push

```cpp
push(item)
```

The producer:

1. Locks the queue.
2. Adds the item.
3. Notifies a waiting consumer.

### Pop

```cpp
pop()
```

The consumer:

1. Locks the queue.
2. Waits if the queue is empty.
3. Retrieves an item.
4. Removes it from the queue.

### Why a Condition Variable?

Without a condition variable, a consumer might continuously poll:

```cpp
while (queue.empty()) {
    // busy waiting
}
```

This wastes CPU.

A condition variable allows the consumer to sleep until work becomes available.

---

# 7. Deep Dive: Each Component

## `pcap_reader.h / pcap_reader.cpp`

### Purpose

Reads packet captures from PCAP files.

### PCAP Global Header

```cpp
struct PcapGlobalHeader {

    uint32_t magic_number;
    uint16_t version_major;
    uint16_t version_minor;
    uint32_t snaplen;
    uint32_t network;
};
```

### Packet Header

```cpp
struct PcapPacketHeader {

    uint32_t ts_sec;
    uint32_t ts_usec;
    uint32_t incl_len;
    uint32_t orig_len;
};
```

### Main Operations

```text
open()
   ↓
readNextPacket()
   ↓
readNextPacket()
   ↓
...
   ↓
close()
```

---

# `packet_parser.h / packet_parser.cpp`

### Purpose

Converts raw packet bytes into structured protocol information.

```cpp
bool PacketParser::parse(
    const RawPacket& raw,
    ParsedPacket& parsed
);
```

Conceptually:

```text
Raw Bytes
    │
    ▼
Ethernet Parser
    │
    ▼
IPv4 Parser
    │
    ├── TCP Parser
    │
    └── UDP Parser
    │
    ▼
Parsed Packet
```

---

## Network Byte Order

Network protocols use **network byte order**, which is big-endian.

The host machine may use a different byte order.

Common conversion functions:

```cpp
ntohs()
ntohl()
```

Example:

```cpp
uint16_t port =
    ntohs(network_port);
```

```cpp
uint32_t value =
    ntohl(network_value);
```

---

# `sni_extractor.h / sni_extractor.cpp`

### Purpose

Extract application-layer host information from supported traffic.

For TLS:

```cpp
std::optional<std::string>
SNIExtractor::extract(
    const uint8_t* payload,
    size_t length
);
```

Processing:

```text
TLS Record
    ↓
Client Hello
    ↓
Extensions
    ↓
SNI Extension
    ↓
Hostname
```

For HTTP, the corresponding logic can inspect the `Host` header.

---

# `types.h / types.cpp`

Defines core data structures.

## FiveTuple

```cpp
struct FiveTuple {

    uint32_t src_ip;
    uint32_t dst_ip;

    uint16_t src_port;
    uint16_t dst_port;

    uint8_t protocol;

    bool operator==(
        const FiveTuple& other
    ) const;
};
```

---

## AppType

```cpp
enum class AppType {

    UNKNOWN,
    HTTP,
    HTTPS,
    DNS,

    GOOGLE,
    YOUTUBE,
    FACEBOOK,

    // Additional application types
};
```

---

## SNI → Application Classification

Application classification is signature-based.

For example:

```cpp
if (sni.find("youtube") != std::string::npos)
    return AppType::YOUTUBE;

if (sni.find("facebook") != std::string::npos)
    return AppType::FACEBOOK;
```

This is **deterministic signature matching**, not machine-learning-based classification.

---

# `connection_tracker`

Maintains state associated with network flows.

Conceptually:

```text
FiveTuple
    │
    ▼
Flow Table
    │
    ├── SNI
    ├── Application Type
    ├── Blocked State
    └── Statistics
```

---

# `rule_manager`

Stores and evaluates traffic rules.

Supported rule categories include:

```text
Source IP
Application
Domain
```

The rule manager answers the basic question:

```text
Should this flow be blocked?
```

---

# `load_balancer`

Responsible for distributing packets between Fast Path workers.

```text
Packet
  │
  ▼
Hash FiveTuple
  │
  ▼
Fast Path Selection
```

---

# `fast_path`

Performs the core DPI processing:

```text
Receive Packet
      ↓
Flow Lookup
      ↓
Inspect
      ↓
Classify
      ↓
Apply Rules
      ↓
DROP / FORWARD
```

---

# `thread_safe_queue`

Provides synchronized communication between processing stages.

Core concurrency primitives include:

```text
std::mutex
std::lock_guard
std::unique_lock
std::condition_variable
```

The queue implements the producer-consumer pattern.

---

# `dpi_engine`

Coordinates the complete multi-threaded system.

Responsibilities include:

* Starting worker threads
* Connecting queues
* Configuring Load Balancers
* Configuring Fast Paths
* Coordinating shutdown
* Managing processing flow

---

# 8. How SNI Extraction Works

## TLS Handshake

A simplified TLS connection looks like:

```text
┌──────────┐                              ┌──────────┐
│  Client  │                              │  Server  │
└────┬─────┘                              └────┬─────┘
     │                                         │
     │ ─────── Client Hello ─────────────────► │
     │          SNI: example.com               │
     │                                         │
     │ ◄────── Server Hello ────────────────── │
     │                                         │
     │ ◄──────── TLS Handshake ───────────────► │
     │                                         │
     │ ◄══════ Encrypted Application Data ════► │
```

The DPI engine attempts to obtain the hostname from the Client Hello when the SNI is available.

---

# TLS Client Hello Structure

A simplified TLS record:

```text
Byte 0:      Content Type = 0x16
Bytes 1-2:   Record Version
Bytes 3-4:   Record Length

Handshake:

Byte 5:      Handshake Type = 0x01
Bytes 6-8:   Handshake Length

Client Hello:

Client Version
Random
Session ID
Cipher Suites
Compression Methods
Extensions
```

The SNI extension has:

```text
Extension Type: 0x0000
Extension Length
SNI List Length
SNI Type
SNI Length
SNI Value
```

Example:

```text
SNI Extension
      │
      ▼
www.example.com
```

---

# SNI Extraction Flow

```text
Raw TCP Payload
      │
      ▼
Check TLS Record
      │
      ▼
Check Client Hello
      │
      ▼
Skip Client Hello Fields
      │
      ▼
Read Extensions
      │
      ▼
Find Extension 0x0000
      │
      ▼
Extract Hostname
```

A simplified implementation looks like:

```cpp
std::optional<std::string>
SNIExtractor::extract(
    const uint8_t* payload,
    size_t length
) {

    // Validate TLS record

    if (payload[0] != 0x16)
        return std::nullopt;

    // Validate Client Hello

    if (payload[5] != 0x01)
        return std::nullopt;

    // Navigate through the Client Hello

    // Read extensions

    // Search for extension type 0x0000

    // Extract hostname

    return hostname;
}
```

### Important Production Considerations

A robust DPI implementation should account for:

* TCP segmentation
* TCP stream reassembly
* IP fragmentation
* TLS variations
* Missing SNI
* Encrypted Client Hello (ECH)
* QUIC/HTTP/3
* IPv6

The simplified parser should therefore be viewed as a protocol-analysis implementation rather than a universal HTTPS inspection mechanism.

---

# 9. How Blocking Works

## Rule Types

| Rule Type | Example        | Effect                                |
| --------- | -------------- | ------------------------------------- |
| IP        | `192.168.1.50` | Blocks matching source traffic        |
| App       | `YouTube`      | Blocks classified application traffic |
| Domain    | `tiktok`       | Blocks matching SNI/domain patterns   |

---

## Blocking Flow

```text
Packet Arrives
      │
      ▼
┌──────────────────────────────┐
│ Source IP blocked?           │
└──────────────┬───────────────┘
               │
          Yes ─┴─► DROP
               │ No
               ▼
┌──────────────────────────────┐
│ Application blocked?         │
└──────────────┬───────────────┘
               │
          Yes ─┴─► DROP
               │ No
               ▼
┌──────────────────────────────┐
│ SNI/domain blocked?          │
└──────────────┬───────────────┘
               │
          Yes ─┴─► DROP
               │ No
               ▼
            FORWARD
```

---

# Flow-Based Blocking

Blocking is maintained at the **flow level**.

Example:

```text
TCP Connection

Packet 1: SYN
    → No SNI yet
    → Forward

Packet 2: SYN-ACK
    → No SNI yet
    → Forward

Packet 3: ACK
    → No SNI yet
    → Forward

Packet 4: TLS Client Hello
    → SNI detected
    → Application identified
    → Rule matches
    → Mark flow as BLOCKED
    → Drop

Packet 5: Encrypted data
    → Flow already BLOCKED
    → Drop

Packet 6: Encrypted data
    → Flow already BLOCKED
    → Drop
```

### Why Flow State?

Application identification may only become possible after inspecting a later packet such as a TLS Client Hello.

Once the flow is classified as blocked, subsequent packets can be handled consistently using the stored flow state.

---

# 10. Building and Running

## Prerequisites

* **macOS/Linux**
* **C++17-compatible compiler**
* **g++** or **clang++**
* Python 3 for generating test PCAP data

The C++ implementation does not require external C++ libraries.

---

## Build Commands

### Simple Version

```bash
g++ -std=c++17 -O2 -I include -o dpi_simple \
    src/main_working.cpp \
    src/pcap_reader.cpp \
    src/packet_parser.cpp \
    src/sni_extractor.cpp \
    src/types.cpp
```

### Multi-threaded Version

```bash
g++ -std=c++17 -pthread -O2 -I include -o dpi_engine \
    src/dpi_mt.cpp \
    src/pcap_reader.cpp \
    src/packet_parser.cpp \
    src/sni_extractor.cpp \
    src/types.cpp
```

---

## Running

### Basic Usage

```bash
./dpi_engine test_dpi.pcap output.pcap
```

### With Blocking Rules

```bash
./dpi_engine test_dpi.pcap output.pcap \
    --block-app YouTube \
    --block-app TikTok \
    --block-ip 192.168.1.50 \
    --block-domain facebook
```

### Configure Threads

For the multi-threaded implementation:

```bash
./dpi_engine input.pcap output.pcap --lbs 4 --fps 4
```

This creates:

```text
4 Load Balancer threads
×
4 Fast Path workers per configuration
```

Refer to the implementation's command-line handling for the exact worker topology used by the current version.

---

## Creating Test Data

```bash
python3 generate_test_pcap.py
```

This creates:

```text
test_dpi.pcap
```

which can then be processed by the DPI engine.

---

# 11. Understanding the Output

A typical processing report contains sections such as:

```text
╔══════════════════════════════════════════════════════════════╗
║                 DPI ENGINE                                  ║
╠══════════════════════════════════════════════════════════════╣
║ Load Balancers:  2    Fast Paths:  4                         ║
╚══════════════════════════════════════════════════════════════╝

[Rules] Blocked app: YouTube
[Rules] Blocked IP: 192.168.1.50

[Reader] Processing packets...
[Reader] Done reading packets

╔══════════════════════════════════════════════════════════════╗
║                  PROCESSING REPORT                            ║
╠══════════════════════════════════════════════════════════════╣
║ Total Packets:             ...                               ║
║ Total Bytes:               ...                               ║
║ TCP Packets:               ...                               ║
║ UDP Packets:               ...                               ║
╠══════════════════════════════════════════════════════════════╣
║ Forwarded:                 ...                               ║
║ Dropped:                   ...                               ║
╠══════════════════════════════════════════════════════════════╣
║ APPLICATION BREAKDOWN                                         ║
╠══════════════════════════════════════════════════════════════╣
║ HTTPS                      ...                               ║
║ Unknown                    ...                               ║
║ YouTube                    ...                               ║
║ DNS                        ...                               ║
║ Facebook                   ...                               ║
╚══════════════════════════════════════════════════════════════╝
```

---

## What Each Section Means

| Section               | Meaning                                |
| --------------------- | -------------------------------------- |
| Configuration         | Worker/thread configuration            |
| Rules                 | Active filtering rules                 |
| Total Packets         | Packets read from the input PCAP       |
| Total Bytes           | Captured packet bytes processed        |
| Forwarded             | Packets written to the output PCAP     |
| Dropped               | Packets filtered by configured rules   |
| Thread Statistics     | Work processed by individual workers   |
| Application Breakdown | Classification results                 |
| Detected SNIs         | Hostnames discovered during inspection |

---

# 12. Extending the Project

## 1. Add More Application Signatures

Application signatures can be extended in `types.cpp`.

Example:

```cpp
if (sni.find("twitch") != std::string::npos)
    return AppType::TWITCH;
```

For larger rule sets, more efficient pattern-matching approaches can be considered instead of repeatedly scanning every configured domain.

---

## 2. Add Bandwidth Throttling

Instead of simply dropping traffic, a future traffic-control layer could delay packets belonging to selected flows.

Conceptually:

```cpp
if (shouldThrottle(flow)) {

    std::this_thread::sleep_for(
        std::chrono::milliseconds(10)
    );
}
```

---

## 3. Add Live Statistics

A dedicated statistics thread could periodically report:

```text
Packets/sec
Bytes/sec
Flows
Dropped packets
Forwarded packets
Queue depth
Application distribution
```

Example:

```cpp
void statsThread() {

    while (running) {

        printStats();

        std::this_thread::sleep_for(
            std::chrono::seconds(1)
        );
    }
}
```

---

## 4. Add QUIC / HTTP/3 Support

Modern HTTP/3 uses QUIC over UDP.

A future implementation could add:

```text
UDP
  │
  ▼
QUIC
  │
  ▼
HTTP/3 inspection
```

This would complement the current TCP/TLS-oriented inspection pipeline.

---

## 5. Add IPv6 Support

The current parser focuses on IPv4.

IPv6 support would require:

* IPv6 header parsing
* IPv6 address representation
* Protocol dispatch
* IPv6-aware flow keys
* Updated rule handling

---

## 6. Add TCP Stream Reassembly

TLS Client Hello data may span multiple TCP segments.

A more robust DPI pipeline can therefore implement:

```text
TCP Segment 1 ─┐
TCP Segment 2 ─┼──► TCP Stream Reassembly
TCP Segment 3 ─┘
                         │
                         ▼
                   TLS Parser
```

This makes application-layer inspection more reliable.

---

## 7. Improve Flow Management

For large captures or long-running systems, flow state can be bounded using mechanisms such as:

* Idle timeouts
* FIN/RST cleanup
* LRU eviction
* Maximum flow-table size

---

## 8. Improve Rule Matching

The current domain matching approach can use substring matching.

For larger rule sets, alternatives include:

```text
Hash-based exact matching
Trie
Aho-Corasick
Suffix matching
```

The appropriate structure depends on whether rules require exact, prefix, suffix, or multi-pattern matching.

---

# Architecture Summary

The complete system can be summarized as:

```text
                       ┌──────────────┐
                       │  Input PCAP  │
                       └──────┬───────┘
                              │
                              ▼
                       ┌──────────────┐
                       │ PCAP Reader  │
                       └──────┬───────┘
                              │
                              ▼
                       ┌──────────────┐
                       │Packet Parser │
                       └──────┬───────┘
                              │
                              ▼
                        Five-Tuple
                              │
                              ▼
                         Flow State
                              │
                              ▼
                    TLS / HTTP Inspection
                              │
                              ▼
                     SNI / Hostname
                              │
                              ▼
                    Application Type
                              │
                              ▼
                       Rule Manager
                       /          \
                      ▼            ▼
                   BLOCK         ALLOW
                      │             │
                      │             ▼
                      │       Output Queue
                      │             │
                      │             ▼
                      │       Output Writer
                      │             │
                      └──────► Output PCAP
```

---

# Key Technical Concepts

This project demonstrates practical implementation of:

### Networking

* Ethernet frame parsing
* IPv4 parsing
* TCP/UDP parsing
* TCP flow identification
* Five-tuples
* PCAP file format
* Network byte order

### Deep Packet Inspection

* TLS Client Hello inspection
* SNI extraction
* HTTP Host extraction
* Application signature matching
* Domain-based classification

### Stateful Processing

* Flow tables
* Per-flow state
* Flow-based blocking
* Application classification state

### Concurrent Programming

* `std::thread`
* `std::mutex`
* `std::lock_guard`
* `std::unique_lock`
* `std::condition_variable`
* Producer-consumer queues
* Multi-stage processing pipelines
* Flow-aware packet distribution

### Systems Programming

* Binary file I/O
* Raw byte parsing
* Protocol offsets
* Network/host byte-order conversion
* Hash-based routing
* Concurrent data processing

---

# Summary

The DPI Engine is a C++17 network traffic analysis system that processes PCAP captures through a structured inspection pipeline:

```text
PCAP
 ↓
Packet Parsing
 ↓
Five-Tuple Generation
 ↓
Flow Tracking
 ↓
TLS / HTTP Inspection
 ↓
SNI / Host Extraction
 ↓
Application Classification
 ↓
Rule Evaluation
 ↓
Forward / Drop
 ↓
Filtered PCAP + Statistics
```

The multi-threaded implementation extends this pipeline with:

```text
Reader
   ↓
Load Balancers
   ↓
Fast Path Workers
   ↓
Output Queue
   ↓
Writer
```

Flow-aware hashing ensures packets belonging to the same five-tuple can remain associated with the same processing worker, allowing flow state to be maintained locally while enabling parallel processing across independent flows.

The project provides a practical demonstration of **network protocol parsing, deep packet inspection, flow tracking, signature-based traffic classification, stateful filtering, producer-consumer concurrency, and multi-threaded packet-processing architecture**.
#   D e e p - P a c k e t - I n s p e c t i o n - E n g i n e  
 
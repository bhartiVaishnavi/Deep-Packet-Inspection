# DPI Engine - Deep Packet Inspection System

A C++17 project that reads saved network captures (PCAP files), identifies which application each connection belongs to (YouTube, Facebook, GitHub, etc.), applies blocking rules, and generates a traffic report.

It comes in two versions: a simple single-threaded one for understanding the core ideas, and a multi-threaded one that processes packets in parallel using a pipeline of worker threads.

> **Scope:** This is an offline learning project. It analyzes recorded PCAP files, not live traffic, and it does not decrypt HTTPS. It only uses information that is visible in plaintext, such as the domain name sent during the TLS handshake.

---

## Table of Contents

1. [Why This Project](#why-this-project)
2. [Key Features](#key-features)
3. [Tech Stack and Concepts Used](#tech-stack-and-concepts-used)
4. [How It Works](#how-it-works)
5. [Multi-threaded Architecture](#multi-threaded-architecture)
6. [How Blocking Works](#how-blocking-works)
7. [Project Structure](#project-structure)
8. [Build and Run](#build-and-run)
9. [Sample Output](#sample-output)
10. [Challenges and Design Decisions](#challenges-and-design-decisions)
11. [Future Improvements](#future-improvements)
12. [What I Learned](#what-i-learned)

---

## Why This Project

Most traffic today is HTTPS, so the content of a connection is encrypted. However, the first message of the TLS handshake (the Client Hello) still contains the destination domain name in plaintext (SNI, Server Name Indication). Networks, ISPs and enterprise firewalls use this to recognize and manage applications.

I built this project to understand that process from the ground up: reading raw bytes, parsing protocols, tracking connections, and scaling the work across threads.

---

## Key Features

- Reads and writes standard **PCAP** capture files (as saved by Wireshark/tcpdump)
- Parses **Ethernet, IPv4, TCP and UDP** headers from raw bytes, handling network byte order
- Tracks connections using the **5-tuple** (source IP, destination IP, source port, destination port, protocol)
- Identifies applications from the **TLS SNI** and the **HTTP Host header**
- Rule-based blocking by **IP address, application or domain**
- Connection-level (flow-based) decisions, so one verdict applies to every packet of that connection
- Writes only the allowed packets to a new PCAP file
- Prints a **report**: packet counts, forwarded/dropped totals, per-thread statistics, application breakdown and detected domains
- Configurable number of load-balancer and worker threads from the command line

---

## Tech Stack and Concepts Used

| Area | Details |
|------|---------|
| Language | C++17 (STL, `std::optional`, structured bindings, hash maps) |
| Concurrency | `std::thread`, `std::mutex`, `std::condition_variable`, producer-consumer pattern |
| Networking | TCP/IP model, Ethernet/IP/TCP/UDP headers, 5-tuple flows, TLS handshake, SNI, HTTP |
| Data handling | Binary file I/O, byte-order conversion (big-endian vs little-endian), bounds-checked parsing |
| Tools | g++ (MSYS2 on Windows, clang on macOS), Wireshark for verifying results, Python for generating test captures |
| Design | Modular components, separate single-threaded and multi-threaded implementations |

---

## How It Works

```
input.pcap → Read → Parse → Track Flow → Classify → Apply Rules → Forward / Drop → output.pcap
                                                                        ↓
                                                                  Traffic Report
```

1. **Read:** The PCAP reader validates the file header and reads packets one by one.
2. **Parse:** Each packet is decoded layer by layer (Ethernet → IP → TCP/UDP) into a structured form.
3. **Track:** A flow table, keyed by the 5-tuple, groups packets into connections and stores per-connection state.
4. **Classify:** For HTTPS, the TLS Client Hello is inspected to read the SNI. For HTTP, the Host header is used. The domain is then mapped to an application such as YouTube or GitHub.
5. **Decide:** The connection is checked against the active blocking rules.
6. **Output:** Allowed packets are written to the output PCAP. Blocked packets are dropped. A summary report is printed at the end.

---

## Multi-threaded Architecture

```
Reader → Load Balancers → Fast Path workers → Output Writer → output.pcap
```

| Stage | Responsibility |
|-------|----------------|
| Reader | Reads packets from the PCAP file and hands them to the load balancers |
| Load Balancers | Distribute packets across the fast-path workers |
| Fast Path workers | Track flows, classify applications and apply rules |
| Output Writer | Collects forwarded packets and writes the output file |

**Key design points:**
- Stages communicate through **thread-safe queues**, so threads wait efficiently instead of busy-looping.
- Packets are assigned to workers by **hashing the 5-tuple**. Every packet of a connection reaches the same worker, so each worker can keep its own flow table without sharing state or locking it.
- The number of load balancers and workers is configurable, so the pipeline can be scaled up or down.

---

## How Blocking Works

| Rule Type | Example | Effect |
|-----------|---------|--------|
| IP | `192.168.1.50` | Blocks traffic from that source |
| Application | `YouTube` | Blocks all connections classified as that app |
| Domain | `tiktok` | Blocks any connection whose domain contains that text |

Blocking is **flow-based**. The application cannot be identified until the TLS Client Hello arrives, so the first packets of a connection (the TCP handshake) are forwarded. Once the connection is identified as blocked, that packet and every later packet of the same connection are dropped.

---

## Project Structure

```
packet_analyzer/
├── include/                 # Header files
├── src/
│   ├── main_working.cpp     # Simple single-threaded version
│   ├── dpi_mt.cpp           # Multi-threaded version
│   └── ...                  # PCAP reading, packet parsing, SNI extraction, helpers
├── generate_test_pcap.py    # Creates sample test traffic
├── test_dpi.pcap            # Sample capture
└── README.md
```

Main components: PCAP reader, packet parser, SNI/Host extractor, rule manager, connection tracker, load balancer, fast-path worker and thread-safe queue.

---

## Build and Run

Run all commands from the project folder.

### Windows (CMD)

**Requirements:** Windows 10/11, MSYS2 GCC (C++17), Python 3 (only for generating the sample file)

**Build:**

```bat
g++ -std=c++17 -pthread -O2 -I include -o dpi_engine.exe src/dpi_mt.cpp src/pcap_reader.cpp src/packet_parser.cpp src/sni_extractor.cpp src/types.cpp
```

**Generate the sample PCAP (optional):**

```bat
python generate_test_pcap.py
```

**Run:**

```bat
dpi_engine.exe test_dpi.pcap output.pcap
```

**Run with blocking rules:**

```bat
dpi_engine.exe test_dpi.pcap output.pcap --block-app YouTube --block-ip 192.168.1.50 --block-domain facebook
```

**Set thread counts:**

```bat
dpi_engine.exe test_dpi.pcap output.pcap --lbs 2 --fps 2
```

### macOS (Terminal)

**Requirements:** Xcode Command Line Tools (provides `g++`/clang with C++17 support), Python 3 (only for generating the sample file)

**Install the compiler tools (one time):**

```bash
xcode-select --install
```

**Build:**

```bash
g++ -std=c++17 -pthread -O2 -I include -o dpi_engine src/dpi_mt.cpp src/pcap_reader.cpp src/packet_parser.cpp src/sni_extractor.cpp src/types.cpp
```

**Generate the sample PCAP (optional):**

```bash
python3 generate_test_pcap.py
```

**Run:**

```bash
./dpi_engine test_dpi.pcap output.pcap
```

**Run with blocking rules:**

```bash
./dpi_engine test_dpi.pcap output.pcap --block-app YouTube --block-ip 192.168.1.50 --block-domain facebook
```

**Set thread counts:**

```bash
./dpi_engine test_dpi.pcap output.pcap --lbs 2 --fps 2
```

---

## Sample Output

```
Total Packets:   77
Forwarded:       69
Dropped:          8

THREAD STATISTICS
  LB0 dispatched:   53
  LB1 dispatched:   24

APPLICATION BREAKDOWN
HTTPS     39  50.6%
Unknown   16  20.8%
YouTube    4   5.2%  (BLOCKED)
DNS        4   5.2%
Facebook   3   3.9%

[Detected Domains]
  www.youtube.com  -> YouTube
  www.facebook.com -> Facebook
  github.com       -> GitHub
```

You can open `output.pcap` in Wireshark to confirm that the blocked traffic is missing.

---

## Challenges and Design Decisions

- **Identifying apps inside encrypted traffic:** Only the handshake is readable, so classification happens on the first data packet of a connection and the result is stored for the rest of the flow.
- **Keeping a connection on one thread:** Consistent hashing of the 5-tuple avoids shared flow tables and lock contention between workers.
- **Safe parsing of untrusted bytes:** Every header and length field is bounds-checked before it is read, since captured packets can be truncated or malformed.
- **Endianness:** Network byte order is converted to host byte order when reading multi-byte fields.
- **Clean shutdown of the pipeline:** The output stage keeps running until all queues are drained, so no forwarded packets are lost.

---

## What I Learned

- How packets are structured and parsed at the byte level
- How connection tracking and flow-based decisions work
- How DPI classifies traffic that is encrypted
- Designing a multi-threaded pipeline with thread-safe queues and avoiding shared state
- Structuring a C++ project into small, testable components
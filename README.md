# NetSpector

**Developed by:** Priyanshu Raj

NetSpector is a high-performance, multi-threaded Deep Packet Inspection (DPI) engine written in modern C++ (C++17). It is designed to ingest raw network traffic captures (PCAP), perform Layer 7 payload analysis, and apply granular, stateful filtering rules based on Application indicators, Domain names (SNI), and IP addresses. 

Unlike standard firewalls that restrict filtering to Layer 3/Layer 4 headers, NetSpector reconstructs network flows and parses application-layer payloads (such as TLS Client Hello messages and HTTP Host headers) to make intelligent routing and blocking decisions.

## Core Features

*   **Custom PCAP Parser:** Reads and writes PCAP files natively without relying on heavy external dependencies like `libpcap`.
*   **Multi-Threaded Architecture:** Utilizes a highly concurrent Producer-Consumer model. 
    *   **Load Balancers (LBs):** Distribute incoming packets using consistent hashing (5-tuple).
    *   **Fast Paths (FPs):** Dedicated worker threads that maintain stateful connection flow tables and perform the heavy lifting of payload inspection.
*   **Deep Payload Extraction:**
    *   **TLS/HTTPS:** Parses TLS records to extract the Server Name Indication (SNI) in plaintext during the initial Client Hello.
    *   **HTTP:** Extracts the `Host` header from raw plaintext HTTP requests.
    *   **DNS:** Parses DNS queries to identify domain resolution requests.
*   **Granular Traffic Filtering:** Dynamically blocks traffic at the flow level based on:
    *   Source/Destination IP Addresses
    *   Application Types (e.g., YouTube, Facebook, Netflix)
    *   Domain Names (Substring matching against extracted SNIs)
*   **Detailed Analytics:** Generates comprehensive telemetry reports on traffic distribution, dropped packets, and application usage breakdowns.

## System Architecture

NetSpector operates on a pipelined, multi-threaded model designed for high throughput:

1.  **Ingestion:** The main thread reads the input PCAP file, validates headers, and extracts the 5-tuple (Src IP, Dst IP, Src Port, Dst Port, Protocol).
2.  **Load Balancing:** Packets are hashed based on their 5-tuple and routed to one of the Load Balancer (LB) threads.
3.  **Fast Path Processing:** The LB thread routes the packet to a specific Fast Path (FP) thread. Because of consistent hashing, all packets belonging to the same TCP/UDP flow are guaranteed to hit the same FP thread, allowing the engine to safely maintain stateful flow tracking without constant mutex locking.
4.  **Inspection & Enforcement:** The FP thread decodes the payload, extracts application data (like SNI), and checks it against the `RuleManager`. If a block rule matches, the flow is marked as blocked and all subsequent packets for that flow are dropped.
5.  **Egress:** Forwarded packets are pushed to a thread-safe output queue and written to a sanitized output PCAP file.

## Build Instructions

NetSpector is built using CMake and requires a C++17 compatible compiler (GCC, Clang, or MSVC).

```bash
# Clone the repository and navigate to the project root
git clone <your-repo-url>
cd NetSpector

# Create a build directory
mkdir build && cd build

# Generate build files and compile with Release optimizations (-O3)
cmake -DCMAKE_BUILD_TYPE=Release ..
cmake --build .
```

## Usage

NetSpector is a command-line tool. You provide an input PCAP file (containing raw traffic) and specify the filtering rules. The engine will produce an output PCAP file containing only the permitted traffic.

### Basic Syntax
```bash
./netspector <input.pcap> <output.pcap> [options]
```

### Filtering Options
*   `--block-ip <ip>`: Block all traffic originating from this source IP.
*   `--block-app <app>`: Block a specific application profile (e.g., `YouTube`, `Facebook`, `TikTok`).
*   `--block-domain <dom>`: Block any traffic where the SNI or Host header contains this substring.
*   `--lbs <n>`: Override the number of Load Balancer threads (default: 2).
*   `--fps <n>`: Override the number of Fast Path threads per LB (default: 2).

### Examples

**Filter out social media and a specific IP:**
```bash
./netspector capture.pcap filtered.pcap --block-app YouTube --block-domain facebook --block-ip 192.168.1.50
```

**Run with maximum concurrency (4 LBs, 4 FPs per LB = 16 worker threads):**
```bash
./netspector high_volume.pcap output.pcap --lbs 4 --fps 4
```

## Generating Test Data

If you need a PCAP file to test the engine, you can generate synthetic traffic using the included Python script. This script crafts mock TLS, HTTP, and DNS packets.

```bash
python3 generate_test_pcap.py
./netspector test_dpi.pcap result.pcap --block-app Netflix
```

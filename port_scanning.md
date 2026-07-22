# Architecture & Methodology: Port Scanning Module

## 1. Process & Methodology

In a professional bug bounty and penetration testing workflow, **Port Scanning** represents the critical transition from passive asset discovery to active engagement. Once a target's infrastructure has been mapped via subdomain enumeration and DNS resolution, the objective shifts to identifying live hosts and the services exposed on their network interfaces.

A mature methodology rarely relies on a single pass. Instead, it utilizes a **tiered scanning approach**:
1. **Broad Sweep (Discovery Pass):** Rapid, asynchronous SYN scanning across all 65,535 ports on all resolved IP addresses and CIDR ranges to identify "open" or "filtered" ports. This prioritizes speed and raw connectivity over deep inspection.
2. **Deep Inspection (Service Enumeration):** Targeted, stateful TCP connect scans directed *only* at the ports discovered in the first pass. This stage gathers service banners, determines software versions, and executes localized vulnerability scripts.
3. **Continuous Delta Scanning:** In ongoing bug bounties, infrastructure changes rapidly. Professional workflows continuously scan known assets for state changes (e.g., a newly exposed development port) rather than rescanning the entire surface blindly.

## 2. Current Tools & Ecosystem

The current ecosystem is highly specialized, with tools generally favoring either pure speed or deep analytical capability:

*   **naabu:** Written in Go, it is the modern standard for the initial broad sweep. It utilizes SYN-based scanning and integrates seamlessly with other Project Discovery tools, outputting easily parseable JSON.
*   **masscan:** An asynchronous, internet-scale scanner capable of transmitting millions of packets per second. It is the go-to tool for scanning massive CIDR blocks but lacks application-layer awareness.
*   **nmap:** The undisputed industry standard for deep inspection. While too slow and resource-intensive for initial sweeps across massive scopes, its OS fingerprinting, service detection, and Nmap Scripting Engine (NSE) are unparalleled for detailed analysis.
*   **rustscan:** A hybrid utility designed specifically to solve the speed vs. depth problem by rapidly scanning for open ports and automatically piping those exact ports into Nmap for deep scanning.
*   **unicornscan:** An asynchronous, user-land TCP/IP stack scanner highly effective for unusual scan types, such as stateless TCP and UDP scanning, which are often missed by standard SYN sweeps.

## 3. Where Current Tools Lack

While individual tools are powerful, building an automated, highly scalable pipeline exposes several critical architectural shortcomings:

*   **Non-Standardized Data Structures:** Pipelining `masscan` (custom list/XML) into `nmap` (Grepable/XML) into HTTP probers requires fragile bash scripting (`awk`/`sed`) or complex Python wrappers. There is no universally agreed-upon JSON schema for a "Port Object".
*   **Lack of State Tracking:** Existing tools are stateless point-in-time execution engines. They do not natively know what a target's attack surface looked like yesterday. Tracking "newly opened ports" requires external database management.
*   **Resource and Network Exhaustion:** Tools like `masscan` and `naabu` can easily overwhelm the host machine's network stack (filling NAT state tables or exhausting ephemeral ports) or trigger immediate IP bans from cloud WAFs/IPS, leading to massive false-negative rates.
*   **Monolithic Execution Bottlenecks:** Most tools are designed to run on a single machine. Internet-scale scanning on large scopes (e.g., sweeping all ASNs belonging to a Fortune 500 company) takes too long on a single VPS and lacks native distributed-worker support.
*   **Rigid Handoffs:** Hardcoded wrappers (like Rustscan) lack the flexibility to route specific findings to non-Nmap tools (e.g., sending port 3306 directly to a MySQL brute-forcer, or port 80 to `httpx`).

## 4. Proposed Tool Workflow & Architecture

Our framework's Port Scanning module will act as a **distributed orchestration layer** rather than reinventing the raw packet-crafting wheel. It will consist of an asynchronous dispatcher, a tiered execution engine, and a normalization pipeline.

### Core Architecture Components

1.  **Ingestion & Deduplication Layer:**
    *   Receives resolved IPs, ASNs, and CIDR blocks from the DNS resolution phase.
    *   Translates hostnames to IPs and deduplicates them to prevent redundant scanning of load-balanced domains pointing to the same IP.
2.  **Tier 1: Distributed Fast-Scan Engine (Discovery):**
    *   Wraps a high-speed asynchronous engine (e.g., `naabu` or custom raw-socket Go routines).
    *   Executes full 65k port SYN sweeps.
3.  **Event Router (Pub/Sub):**
    *   As open ports are discovered in Tier 1, they are immediately pushed to a message broker (e.g., Redis, RabbitMQ, or Go Channels).
4.  **Tier 2: Stateful Deep-Scan Engine (Fingerprinting):**
    *   Worker nodes consume discovered IP/Port combinations from the Event Router.
    *   Executes deep inspection (Nmap service detection, TLS certificate parsing, banner grabbing) on those specific ports.
5.  **State Management Database:**
    *   Results are upserted into a graph or relational database. The engine compares current open ports against the historical baseline to trigger "New Asset" alerts.

## 5. Enhancements & Plugin Integration

To exceed the capabilities of existing pipelines, the new module will implement the following advanced features:

### Standardized JSON Output Schema
Every scan result, regardless of the underlying engine used, will be normalized into a strict, unified JSON schema. This ensures seamless integration with downstream modules (HTTP Probing, Brute Forcing).

```json
{
  "asset_id": "ip_192.168.1.100",
  "ip": "192.168.1.100",
  "port": 8443,
  "protocol": "tcp",
  "state": "open",
  "service": {
    "name": "https",
    "product": "nginx",
    "version": "1.18.0",
    "tls": true
  },
  "metadata": {
    "discovery_timestamp": "2026-07-22T14:32:00Z",
    "last_seen": "2026-07-22T14:32:00Z"
  }
}
```

### Distributed & Cloud-Native Scanning
*   **Grid Scanning:** The orchestrator will chunk massive IP ranges and distribute them across ephemeral serverless functions (AWS Lambda/Fargate) or a fleet of cheap VPS nodes via cloud-init.
*   **IP Rotation:** To bypass aggressive rate limiting and IPS blocks, scanning traffic will be routed through rotating proxy pools or multi-cloud exit nodes.

### Smart Auto-Throttling & Evasion
*   **Dynamic Rate Limiting:** The engine will monitor packet drop rates and ICMP unreachable responses in real-time. If it detects network congestion or potential IDS throttling, it will automatically scale back the packets-per-second (PPS) rate.
*   **Smart Target Randomization:** IPs and ports will be scanned in a completely randomized, interleaved order to avoid triggering sequential port-scan detection heuristics.

### Event-Driven Plugin Architecture
The output of the Port Scanner will immediately trigger downstream plugins based on predefined routing rules, creating a truly continuous recon loop:
*   *Rule:* `if port in [80, 443, 8080, 8443] or service == "http"` ➔ Trigger **HTTP Probing Module** (`httpx`).
*   *Rule:* `if service == "ssh"` ➔ Trigger **Security Validation Module** (check for default credentials).
*   *Rule:* `if tls == true` ➔ Extract SSL/TLS certificates and feed the Subject Alternative Names (SANs) back into the **Subdomain Enumeration Module**.
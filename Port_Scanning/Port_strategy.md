
# Nmap-Like Tool Architecture & Features

> This document describes the major components, modules, and techniques that an advanced network scanner should implement. It focuses on architecture and workflow rather than implementation details.

---

# Overall Architecture

```
                   User Input
                        │
                        ▼
               Configuration Parser
                        │
                        ▼
                 Target Manager
                        │
                        ▼
               DNS Resolution Module
                        │
                        ▼
              Host Discovery Engine
                        │
                        ▼
             Network Information Module
                        │
                        ▼
               Port Scanning Engine
                        │
                        ▼
              Packet Analysis Engine
                        │
                        ▼
            Service Detection Engine
                        │
                        ▼
              OS Detection Engine
                        │
                        ▼
          Vulnerability Script Engine
                        │
                        ▼
          Result Correlation Engine
                        │
                        ▼
            Report Generation Module
```

---

# Core Modules

## 1. Configuration Manager

### Responsibilities

- Parse user options
- Validate configuration
- Manage scan profiles
- Load custom settings
- Configure scan behavior

---

## 2. Target Management

### Responsibilities

- Parse IP addresses
- Parse CIDR ranges
- Parse hostnames
- Remove duplicates
- Expand IP ranges
- Queue targets
- Prioritize scanning order

---

## 3. DNS Module

### Responsibilities

- Forward lookup
- Reverse lookup
- Resolver selection
- Multiple DNS support
- DNS caching
- Timeout handling

---

## 4. Host Discovery Engine

### Purpose

Determine whether a host is alive before performing expensive scans.

### Techniques

- ICMP Discovery
- TCP Discovery
- UDP Discovery
- ARP Discovery
- IPv6 Discovery
- Passive Discovery

---

## 5. Packet Engine

### Responsibilities

- Packet creation
- Packet serialization
- Raw socket communication
- TCP packet generation
- UDP packet generation
- ICMP generation
- IPv6 packet generation
- Fragmentation
- Packet checksum generation

---

## 6. Scan Engine

### Responsibilities

- Manage scan jobs
- Handle concurrency
- Retry failed probes
- Scan scheduling
- Adaptive timing

### Scan Types

- SYN
- TCP Connect
- ACK
- FIN
- NULL
- Xmas
- UDP
- SCTP
- Idle
- Window
- Maimon
- IP Protocol

---

## 7. Port State Analyzer

### Responsibilities

Interpret packet responses.

### States

- Open
- Closed
- Filtered
- Unfiltered
- Open|Filtered
- Closed|Filtered

---

## 8. Service Detection

### Responsibilities

- Banner grabbing
- Protocol fingerprinting
- Service matching
- Version detection
- Product identification
- Vendor detection

---

## 9. OS Fingerprinting

### Responsibilities

Analyze TCP/IP stack characteristics.

### Indicators

- TTL
- TCP Window Size
- DF Bit
- TCP Options
- Sequence Numbers
- ICMP Behavior
- IPID Generation
- Timestamp behavior

---

## 10. Script Engine

### Responsibilities

Run modular scripts after service detection.

### Categories

- Discovery
- Enumeration
- Authentication
- Vulnerability Checks
- Default Credentials
- SSL Analysis
- SMB Enumeration
- HTTP Enumeration
- DNS Enumeration
- FTP Enumeration
- SNMP Enumeration

---

## 11. Performance Engine

### Responsibilities

- Thread Pool
- Worker Scheduling
- Adaptive Timing
- Rate Limiting
- Timeout Adjustment
- Retransmissions
- Packet Queue
- Connection Pool

---

## 12. Network Intelligence

Collect metadata.

### Information

- RTT
- Latency
- Hop Count
- Packet Loss
- MTU
- Route Path
- ASN
- Network Distance

---

## 13. Firewall Detection

### Responsibilities

Detect packet filtering.

### Indicators

- Silent Drop
- ICMP Unreachable
- TCP Reset
- Rate Limiting
- TTL Modification
- Response Delays

---

## 14. IDS Detection

### Indicators

- RST Injection
- Delayed Responses
- Packet Modification
- Sequence Anomalies
- Connection Termination

---

## 15. Evasion Engine

### Techniques

- Packet Fragmentation
- Decoys
- Source Port Selection
- Timing Randomization
- Random Payload
- Data Padding
- MAC Spoofing
- Idle Scan
- Packet Reordering

---

## 16. Fingerprint Database

Maintain databases for:

- Services
- OS
- Vendors
- Products
- Protocols
- Common Ports
- Signatures
- Script Matching

---

## 17. Result Correlation

Combine information from multiple engines.

### Merge

- Host Status
- Port Status
- Service
- Version
- OS Guess
- Script Results
- Network Information
- Timing Statistics

---

## 18. Reporting Engine

Generate reports.

### Formats

- Terminal
- JSON
- XML
- YAML
- CSV
- HTML
- Markdown
- SQLite

---

## 19. Logging System

### Responsibilities

- Debug Logs
- Error Logs
- Packet Logs
- Timing Logs
- Script Logs
- Scan History

---

## 20. Plugin Architecture

Allow third-party extensions.

### Plugin Types

- Scan Plugins
- Service Detection Plugins
- Fingerprinting Plugins
- Script Plugins
- Output Plugins
- Protocol Plugins

---

# Internal Workflow

```
Input
   │
   ▼
Configuration
   │
   ▼
Target Parsing
   │
   ▼
DNS Resolution
   │
   ▼
Host Discovery
   │
   ▼
Network Information
   │
   ▼
Packet Generation
   │
   ▼
Port Scanning
   │
   ▼
Packet Analysis
   │
   ▼
Port Classification
   │
   ▼
Service Detection
   │
   ▼
OS Fingerprinting
   │
   ▼
Script Execution
   │
   ▼
Correlation
   │
   ▼
Reporting
```

---

# Cross-Cutting Components

- Configuration Manager
- Thread Manager
- Task Scheduler
- Packet Queue
- Memory Manager
- Cache Manager
- Database Manager
- Plugin Loader
- Logging Framework
- Error Handler
- Statistics Collector
- Update Manager

---

# Supporting Databases

- Port Database
- Service Fingerprints
- OS Fingerprints
- Protocol Database
- Vendor Database
- NSE Script Database
- Common Ports
- Banner Signatures
- Vulnerability Signatures
- MAC Vendor Database
- ASN Database
- DNS Cache

---

# Major Techniques Used

- Raw Socket Programming
- TCP/IP Stack Fingerprinting
- Banner Grabbing
- Protocol Fingerprinting
- Active Network Scanning
- Passive Discovery
- Packet Crafting
- Adaptive Timing
- Parallel Scanning
- Multi-threading
- Asynchronous Networking
- Signature Matching
- Pattern Matching
- Stateful Packet Analysis
- Response Correlation
- Plugin-Based Architecture
- Event-Driven Processing
- Modular Pipeline Design
- Retry Algorithms
- Rate Control
- Output Normalization

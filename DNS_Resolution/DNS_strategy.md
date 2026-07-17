# DNS Resolution Engine

> Inspired by: dnsx, massdns, puredns, shuffledns, zdns

---

# Goal

Resolve discovered domains and subdomains into valid DNS records while filtering invalid, wildcard, parked, or unreachable entries. The DNS Resolution Engine transforms raw asset lists into verified targets that can be used by later stages such as HTTP probing, port scanning, and vulnerability assessment.

---

# Core Concept

After enumeration, thousands of discovered hostnames may be outdated, unresolved, duplicated, or protected by wildcard DNS.

The DNS Resolution Engine validates every hostname by querying DNS resolvers, collecting records, identifying wildcard responses, and enriching the data with DNS metadata.

Unlike subdomain enumeration, this engine focuses on **verification and intelligence gathering**, not discovery.

---

# Workflow

```
Input Hostnames
       │
       ▼
Normalize Targets
       │
       ▼
Load Resolver Pool
       │
       ▼
Resolve DNS Records
       │
       ▼
Detect Wildcards
       │
       ▼
Validate Responses
       │
       ▼
Collect DNS Metadata
       │
       ▼
Filter Invalid Hosts
       │
       ▼
Generate Output
```

---

# Core Modules

## Input Manager

Responsibilities

- Load hostnames
- Remove duplicates
- Normalize formatting
- Validate domain syntax

---

## Resolver Manager

Responsibilities

- Load public resolvers
- Load custom resolvers
- Rotate resolvers
- Health check resolvers
- Remove slow or dead resolvers

---

## Resolution Engine

Responsible for querying DNS records efficiently.

Supported Record Types

- A
- AAAA
- CNAME
- MX
- TXT
- NS
- SOA
- PTR
- CAA
- SRV

---

## Wildcard Detection Engine

Determine whether a DNS zone uses wildcard records.

Checks

- Random hostname generation
- Response comparison
- IP similarity
- CNAME similarity
- DNS fingerprint comparison

---

## Validation Engine

Verify DNS responses.

Checks

- NXDOMAIN
- SERVFAIL
- REFUSED
- Timeout
- Empty Answer
- Multiple Answers

---

## Metadata Collector

Collect additional information.

Examples

- TTL
- Resolver Used
- Response Time
- Record Count
- Record Type
- Authoritative Status
- CNAME Chain

---

## Filter Engine

Remove

- Duplicate records
- Wildcard results
- Invalid responses
- Empty records
- Broken CNAME chains

---

## Output Engine

Generate

- TXT
- JSON
- CSV
- SQLite
- NDJSON

---

# Techniques Used

- Parallel DNS Resolution
- Multi-Resolver Rotation
- Wildcard Detection
- Record Enumeration
- Response Validation
- DNS Caching
- Retry Logic
- Response Correlation

---

# Supported DNS Records

| Record | Purpose |
|---------|---------|
| A | IPv4 Address |
| AAAA | IPv6 Address |
| CNAME | Alias Resolution |
| MX | Mail Servers |
| NS | Name Servers |
| TXT | Verification & Metadata |
| SOA | Zone Information |
| PTR | Reverse DNS |
| SRV | Service Discovery |
| CAA | Certificate Authority Policy |

---

# Input

- Domains
- Subdomains
- Resolver List
- Configuration
- Record Types

---

# Output

- Resolved Hosts
- DNS Records
- Wildcard Status
- Response Time
- TTL
- Resolver Information
- Validation Status

---

# Design Goals

- Fast Resolution
- High Accuracy
- Modular Architecture
- Multi-threaded
- Wildcard Aware
- Low False Positives
- Scalable
- Resolver Independent

---

# Integration

Previous Engine

- Subdomain Enumeration

Next Engines

- HTTP Probing
- Port Scanning
- Technology Fingerprinting
- Web Crawling
- Vulnerability Scanning

---

# Internal Pipeline

```
Subdomains
     │
     ▼
Normalization
     │
     ▼
Resolver Selection
     │
     ▼
DNS Resolution
     │
     ▼
Record Collection
     │
     ▼
Wildcard Detection
     │
     ▼
Validation
     │
     ▼
Filtering
     │
     ▼
Metadata Collection
     │
     ▼
Structured Output
```

---

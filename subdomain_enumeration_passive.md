# Architecture & Methodology: Subdomain Enumeration (Passive)

## 1. Process & Methodology

In professional bug bounty hunting and penetration testing, **Passive Subdomain Enumeration** is the critical first step in defining a target's external attack surface. "Passive" strictly means that the target's own infrastructure is never directly queried or interacted with. Instead, the methodology relies on extracting information from third-party datasets, historical archives, and public internet infrastructure.

The core methodology involves querying multiple distinct data vectors:
*   **Certificate Transparency (CT) Logs:** Querying databases like `crt.sh` to find subdomains tied to issued SSL/TLS certificates.
*   **Public Datasets & Search Engines:** Scraping results from Google, Bing, Baidu, and specialized datasets like Project Discovery's `chaos`.
*   **Threat Intelligence & OSINT APIs:** Querying services like SecurityTrails, Censys, Shodan, and VirusTotal for historical DNS records.
*   **Code Repositories:** Searching GitHub, GitLab, and Bitbucket for hardcoded development, staging, or internal subdomains.
*   **Web Archives:** Extracting URLs from the Wayback Machine or Common Crawl (`gau`), which inherently leak valid subdomains.

The goal of this phase is absolute coverage and stealth. By gathering as many potential subdomains as possible without alerting the target, the attacker builds a massive foundational list that will be deduplicated, resolved, and probed in subsequent active phases.

---

## 2. Current Tools & Ecosystem

The current ecosystem is rich, but heavily fragmented. Based on the documented toolset, the ecosystem relies on a few heavy hitters and several specialized scripts:

*   **Aggregators:** Tools like `subfinder` and `findomain` are built for pure speed, concurrently polling dozens of free and paid API endpoints to aggregate lists quickly.
*   **Comprehensive Frameworks:** `amass (passive mode)` builds an extensive graph of relationships (ASNs, WHOIS, IP blocks) alongside standard scraping. It is exhaustive but resource-intensive.
*   **Lightweight / Focused CLI:** `assetfinder` focuses purely on high-yield sources like CT logs and Facebook's API, prioritizing Unix-philosophy chaining over features.
*   **Niche / Alternative Sources:** `github-subdomains` finds developer-leaked endpoints, while `gau` inadvertently extracts deep subdomains hidden within archived endpoint URLs. Web interfaces like `dnsdumpster` and `rapiddns` provide quick, visual, or historical references but are less suited for automated pipelines.

---

## 3. Where Current Tools Lack

While highly effective, current tools present several architectural bottlenecks when building a continuously running, highly scalable framework:

1.  **Lack of Standardized Output & Provenance:** Most tools output a flat text list of subdomains. When JSON is supported, the schemas differ wildly. Critical metadata is often lost, such as *which* source found the subdomain, *when* it was first seen, and *what* API key was used.
2.  **Poor State Tracking & Diffing:** Tools execute as stateless binaries. In a continuous monitoring scenario, operators want to know what subdomains are *new* today compared to yesterday. Existing tools require messy bash scripting (`comm`, `diff`, `anew`) to track state, which fails at scale.
3.  **API Key Fatigue & Rate Limit Mishandling:** Users must maintain dozens of API keys across `subfinder`, `amass`, etc. When a rate limit is hit, many tools silently drop the source or panic, rather than implementing intelligent queuing, backoff, or key-rotation logic.
4.  **Memory & Resource Bloat:** Frameworks like `amass` build massive local graph databases (e.g., Cayley) which can consume gigabytes of RAM and lock up during large target sweeps, making them hostile to lightweight containerized deployments.
5.  **Monolithic Architecture:** If a single API scraper breaks due to an HTML change, the user must wait for the core maintainers to release a new version of the entire binary, rather than just updating a standalone plugin.

---

## 4. Proposed Tool Workflow & Architecture

To resolve these issues, the new Passive Subdomain Enumeration module will be designed as a **decoupled, event-driven pipeline** emphasizing statefulness, speed, and modularity. 

### Core Architecture Components:

*   **Dispatcher (Control Plane):** Accepts the target scope (domains, ASNs, or organization names), manages API key pools, and distributes tasks to worker nodes/threads.
*   **Source Plugins (The Workers):** Lightweight, independent modules (e.g., Go plugins, Lua scripts, or WebAssembly modules) that map to specific sources (e.g., `crtsh_worker`, `securitytrails_worker`). If a scraping logic breaks, only the specific plugin is hot-reloaded.
*   **Normalization & Deduplication Engine:** An asynchronous channel that ingests raw results from all workers in real-time. It deduplicates subdomains on the fly and standardizes the schema.
*   **State Store (KV Database):** An embedded database (like SQLite or BadgerDB) or a remote Redis instance that maintains the historical state of the target's attack surface.
*   **Output Emitter:** Streams standardized JSON-lines to stdout, pushes to a message queue (Kafka/RabbitMQ) for the active resolution phase, or triggers webhooks (Slack/Discord) for newly discovered assets.

### Execution Workflow:
1. **Ingest:** Pipeline receives `target.com`.
2. **Fan-out:** Dispatcher checks API key quotas and spins up 30+ concurrent source plugins.
3. **Fetch & Yield:** Plugins query APIs, CT logs, and archives, streaming raw results back to the Normalization Engine as soon as they arrive (no waiting for all to finish).
4. **Enrich & Diff:** The Engine lowercases the domains, strips wildcards, and checks the State Store. If the subdomain is new, it is marked with `is_new: true`.
5. **Fan-in & Emit:** Data is flushed to the Output Emitter and immediately passed to the next framework phase (DNS Resolution).

---

## 5. Enhancements & Plugin Integration

To ensure this framework outperforms the existing ecosystem, we will implement the following advanced enhancements:

### A. Universal JSON Schema
Every plugin will adhere to a strict, standardized output format to make downstream pipelining flawless:
```json
{
  "subdomain": "api.dev.target.com",
  "root_domain": "target.com",
  "sources": ["SecurityTrails", "crt.sh", "github-subdomains"],
  "first_seen": "2026-07-22T08:00:00Z",
  "is_new": true,
  "confidence": "high",
  "metadata": {
    "github_repo": "target/internal-api",
    "ct_issuer": "Let's Encrypt"
  }
}
```

### B. Smart Rate Limiting & Key Rotation
Implement an intelligent API manager. If the `SecurityTrails` plugin hits a 429 Too Many Requests, the manager will:
1. Pause the specific worker (not the whole program).
2. Automatically rotate to a secondary API key if available.
3. Apply exponential backoff and re-queue the failed query.

### C. Distributed & Serverless Scanning
Unlike `amass` or `subfinder` which run on a single machine, our architecture will support distributed execution. Plugins can be deployed as AWS Lambda functions or distributed via gRPC to disposable VPS instances. This circumvents IP-based rate limiting from services like GitHub or Google.

### D. "Smart" Passive Filtering
Passive sources often return garbage (e.g., `*.target.com`, `www.-dev.target.com`, or domains from 2012 that no longer exist). The Normalizer will include a heuristic filter to flag malformed subdomains and segregate wildcard entries before they clog up the active resolution pipeline.

### E. Bring-Your-Own-Plugin (BYOP)
By utilizing an embedded scripting engine (like Starlark or Lua) or WebAssembly (Wasm), users can write a 10-line script to scrape a new, proprietary, or custom OSINT source without needing to recompile the core framework.
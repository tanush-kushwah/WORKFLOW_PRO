# Architecture & Methodology: Historical URL Discovery

## 1. Process & Methodology

In modern offensive security, bug bounty hunting, and penetration testing, mapping the current live attack surface is only half the battle. **Historical URL Discovery** is the practice of querying third-party archival services and threat intelligence datasets to retrieve a comprehensive list of URLs, endpoints, and parameters associated with a target domain over time. 

**Why it matters:**
*   **Forgotten Infrastructure:** Development, staging, or legacy endpoints are often unlinked from the main application but remain active on the server.
*   **Hidden Parameters:** Archives capture GET parameters (`?id=`, `?token=`, `?redirect=`) that may no longer be referenced in modern client-side code but still function on the backend, opening the door for SSRF, XSS, or IDOR.
*   **Sensitive Data Leaks:** Temporary files (e.g., `.sql`, `.bak`, `.env`, `.git`, `.tar.gz`) might have been captured by crawlers before being deleted or restricted.
*   **Passive Reconnaissance:** Because this data is pulled from third-party aggregators (like the Wayback Machine, Common Crawl, and AlienVault OTX), no traffic is sent to the target's servers, making this phase entirely stealthy.

**Workflow:**
1.  **Seeding:** Provide a root domain or a list of resolved subdomains.
2.  **Aggregation:** Query multiple external APIs and datasets concurrently.
3.  **Extraction & Parsing:** Parse the JSON/Text responses into raw URLs.
4.  **Filtering & Deduplication:** Remove noise (static assets) and identical URLs.
5.  **Downstream Pipelining:** Pass the filtered list to active HTTP probers, parameter fuzzers, or vulnerability scanners.

---

## 2. Current Tools & Ecosystem

The current ecosystem relies heavily on specialized, single-purpose CLI utilities. Based on the industry standard tools, the landscape looks like this:

*   **waybackurls (tomnomnom):** The grandfather of historical discovery. It elegantly and simply pulls all URLs archived by the Wayback Machine for a specific domain. Highly modular via Unix pipes.
*   **gau / GetAllUrls (lc):** The current standard for speed and breadth. It aggregates URLs from the Wayback Machine, Common Crawl, and AlienVault OTX, significantly expanding the dataset beyond just Internet Archive.
*   **gauplus (bp0lr):** An enhanced fork of `gau` that introduced proxy and worker support for better performance. It is now archived, yielding back to the actively maintained `gau`.
*   **waymore (xnl-h4ck3r):** A highly thorough tool designed for depth rather than raw speed. It implements deeper pagination, advanced filtering, and pulls responses directly from the Wayback Machine (not just the CDX API) to find hidden links.
*   **katana (Project Discovery):** Primarily a modern active crawler, but its passive mode efficiently queries historical sources and outputs highly structured JSON, bridging the gap between passive and active discovery.

---

## 3. Where Current Tools Lack

While existing tools are foundational, they exhibit several architectural shortcomings when scaled into an automated, continuous attack surface mapping framework:

1.  **Output Standardization & Metadata Loss:** Most tools output raw text lists of URLs. They discard valuable metadata returned by the source APIs (e.g., MIME type, HTTP status code at the time of archiving, timestamp, and response length).
2.  **Inefficient Memory & Deduplication:** Running `gau` against a massive target like `yahoo.com` can consume gigabytes of RAM and output millions of duplicate or near-duplicate URLs (e.g., the same endpoint with varying cache-buster parameters).
3.  **Lack of State Tracking (Diffing):** Current tools are stateless. If a continuous recon pipeline runs weekly, these tools fetch the entire historical dataset every time. There is no native way to say, *"Only show me URLs archived since the last scan."*
4.  **Poor Handling of Static Noise:** Security engineers usually don't care about `.jpg`, `.woff2`, or `.css` files unless they are looking for specific CVEs. Filtering requires piping to other tools (like `grep -v` or `gf`), which wastes initial processing time and API limits.
5.  **Brittle Error Handling:** APIs like Common Crawl and Wayback CDX frequently timeout or rate-limit aggressive queries. Simple tools often crash or silently drop data without built-in retry logic, backoff mechanisms, or proxy rotation.

---

## 4. Proposed Tool Workflow & Architecture

To resolve these limitations, our new framework's **Historical URL Discovery Module** will be built using a concurrent, modular, and state-aware architecture.

### Core Architecture Components:

1.  **Input & Config Manager:**
    *   Accepts domains, subdomains, or wildcard scopes.
    *   Loads API keys (for services like URLScan or VirusTotal) and proxy lists.
2.  **Source Provider Engine (Interfaces):**
    *   A plugin-based system where each provider (Wayback, CommonCrawl, OTX, URLScan) implements a standard `Provider` interface. 
    *   Includes built-in **smart rate-limiting** (e.g., Token Bucket algorithm) and **exponential backoff** to handle API temperaments.
3.  **Stream Normalizer & Enricher:**
    *   Instead of waiting for all URLs to download, data is processed in a continuous stream.
    *   **Enrichment:** Captures the timestamp, source provider, and original HTTP status code (if available) and structures it into a standard internal object.
4.  **Heuristic Filtering & Deduplicator:**
    *   **Extension Filtering:** Configurable drop-lists for static assets (fonts, images).
    *   **Parameter Normalization:** Deduplicates URLs that only differ by highly entropic, non-functional parameters (like Google Analytics UTM tags).
    *   **Bloom Filters:** Uses Redis-backed or in-memory Bloom Filters to deduplicate millions of URLs with minimal RAM usage.
5.  **State / Database Layer:**
    *   Saves the normalized data into a local SQLite/PostgreSQL database or key-value store.
    *   Tracks the `last_seen` and `first_seen` timestamps, enabling incremental diffing for continuous scanning operations.

### Data Flow Diagram:
```text
[ Target List ] 
      │
      ▼
[ Provider Engine ] ──────┬───► Wayback Machine CDX API
 (Goroutines/Async)       ├───► Common Crawl Index
                          ├───► AlienVault OTX API
                          └───► URLScan.io API
      │
      ▼ (Raw JSON Streams)
[ Stream Normalizer ]  ───► Converts disparate API schemas into Unified JSON Object
      │
      ▼
[ Heuristic Filter ]   ───► Drops .png, .css / Normalizes UTM parameters
      │
      ▼
[ Bloom Deduplicator ] ───► Compares against historical state; drops duplicates
      │
      ▼
[ State Tracker ]      ───► Tags "New", "Updated", or "Existing"
      │
      ▼
[ Standardized Output ]───► JSONL / Database Insert / Downstream Message Queue
```

---

## 5. Enhancements & Plugin Integration

To make this module a force multiplier in a professional pipeline, we will introduce several advanced enhancements:

### A. Standardized JSON Lines (JSONL) Output
Text lists are obsolete for complex pipelines. All output will be structured JSONL, allowing seamless integration with tools like `jq` and downstream databases.
```json
{
  "target": "api.example.com",
  "url": "https://api.example.com/v1/user/export?format=csv",
  "parameters": ["format"],
  "extension": "csv",
  "source": ["wayback", "otx"],
  "first_seen": "2021-04-12T14:22:01Z",
  "historical_status": 200,
  "is_new": true
}
```

### B. Smart Parameter Extraction (Integration with Parameter Discovery)
Instead of just passing URLs to a fuzzer, the module will have a built-in hook to extract all unique query parameters found across the historical dataset per endpoint. This automatically feeds into tools like `arjun` or `x8`, providing a highly targeted custom wordlist of parameters that the application has *actually used* in the past.

### C. "Liveness" Validation Plugin (Active Pivot)
Historical URLs are only useful if they (or the application paths) still exist. The module will feature an optional **Validator Plugin** that seamlessly pipelines the filtered output directly into an HTTP probing library (akin to `httpx`). 
*   *Action:* It fires a lightweight `HEAD` or `GET` request to the historical URL.
*   *Result:* Appends `"current_status": 404` or `"current_status": 200` to the JSON output, allowing security engineers to instantly filter for zombie endpoints that still return `200 OK`.

### D. Distributed Scanning & Microservices
For massive enterprise scopes, running on a single machine is a bottleneck. The module will support a **Pub/Sub mode** (via Redis or RabbitMQ). A master node distributes subdomains to worker nodes, and the workers stream the discovered, normalized URLs back to a central PostgreSQL database. This allows for infinite horizontal scaling akin to commercial Attack Surface Management (ASM) platforms.
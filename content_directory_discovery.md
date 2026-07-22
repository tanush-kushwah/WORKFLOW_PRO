# Architecture & Methodology Specification
**Phase:** Content & Directory Discovery
**Version:** 1.0.0
**Target Framework:** Next-Generation Modular Attack Surface Mapping (ASM)

---

## 1. Process & Methodology

In a professional penetration testing or bug bounty workflow, **Content and Directory Discovery** (often referred to as directory brute-forcing or web fuzzing) is the critical bridge between passive reconnaissance and active vulnerability scanning. After resolving live hosts and profiling their tech stacks, security engineers must map the application's hidden attack surface—unlinked directories, administrative panels, backup files, legacy API endpoints, and sensitive configuration files.

**Core Methodology:**
1. **Target Ingestion:** Live web servers (typically output from tools like `httpx`) are fed into the discovery engine.
2. **Wordlist Preparation:** Contextual wordlists are selected based on the technology stack (e.g., scanning for `.php.bak` on PHP servers, or `swagger.json` on API endpoints).
3. **Fuzzing & Request Generation:** High-speed HTTP GET/HEAD requests are fired against the target by appending wordlist entries to base URLs.
4. **Heuristic Calibration (Custom 404 Detection):** Before deep fuzzing, the engine establishes a baseline by requesting non-existent randomized paths to profile how the server handles "Not Found" errors (e.g., 200 OK soft-404s, standard 404s, or custom redirects).
5. **Response Analysis:** Responses are evaluated against the calibration baseline using metrics like HTTP status codes, content length, line counts, word counts, and response hashes.
6. **Recursive Discovery:** When a new directory is found (e.g., `301 Redirect` to `/api/v1/`), the tool automatically queues this new path for a recursive scan.

---

## 2. Current Tools & Ecosystem

The current ecosystem is mature but highly fragmented, relying on a mix of languages and approaches:

*   **ffuf (Go):** The current industry standard for raw speed and flexibility. Highly customizable filtering (by words, lines, size) but requires manual tuning for complex targets.
*   **feroxbuster (Rust):** Unrivaled in recursive directory brute-forcing. Built for speed and ease of use, making it excellent for deep, nested applications.
*   **dirsearch (Python):** Offers superior wordlist and extension handling out-of-the-box, with built-in payloads for specific vulnerabilities, though slower than Go/Rust alternatives.
*   **gobuster (Go):** A reliable, fast, standard brute-forcer with multiple modes (DNS, vhost, dir). Excellent for static environments but lacks the dynamic flexibility of `ffuf`.
*   **wfuzz (Python):** Highly flexible for all types of web fuzzing (headers, parameters, URLs) but suffers from performance bottlenecks compared to compiled languages.
*   **dirb (C):** A legacy tool that pioneered the space but is no longer actively maintained or suited for modern, high-concurrency workflows.

---

## 3. Where Current Tools Lack

Despite their speed, existing tools exhibit several critical shortcomings when integrated into a highly scalable, automated continuous recon framework:

1.  **Non-Standardized Output for Pipelining:** Every tool uses a different JSON schema. Piping `feroxbuster` output into `nuclei` or a custom database requires brittle intermediate parsing scripts (glue code).
2.  **State Management & Fault Tolerance:** If a massive 10-hour directory scan is interrupted, most tools lack robust pause/resume state tracking. They are designed for single-session terminal use, not distributed cloud deployments.
3.  **Dumb Calibration & WAF Triggers:** WAFs and modern CDNs frequently block rapid sequential brute-forcing. Current tools either die silently, generate thousands of false positives (due to WAF blocks returning 200 OK CAPTCHA pages), or require extreme manual throttling.
4.  **Inefficient Memory Usage on Deep Recursion:** Tools like `feroxbuster` can suffer from memory exhaustion when scanning large sites with infinite wildcard routing, as they attempt to queue an infinite loop of recursive directories.
5.  **Context-Blindness:** Current tools blindly fire standard wordlists. They do not dynamically generate wordlists based on the site's scraped JavaScript, domain name, or identified technologies.

---

## 4. Proposed Tool Workflow & Architecture

To resolve these bottlenecks, our new Content & Directory Discovery module will be architected as a **distributed, state-aware, and heuristic-driven** microservice.

### A. Architectural Components

*   **Ingestion & Tech-Stack Mapper (Input Node):**
    *   Consumes NDJSON streams from the HTTP probing phase.
    *   Reads technology tags (e.g., `tech:java`, `tech:iis`) to dynamically construct tailored wordlists on the fly.
*   **State & Job Manager (Control Plane):**
    *   Backed by Redis or a lightweight embedded KV store (like BadgerDB/RocksDB).
    *   Tracks all queued paths, preventing infinite recursion loops and enabling pause/resume functionality.
*   **Auto-Calibration Engine (The Heuristic Brain):**
    *   Runs *before* fuzzing begins. Sends 5-10 randomized, mathematically improbable requests.
    *   Captures baseline metrics: Status codes, content lengths, headers, and computes a **SimHash** (Similarity Hash) of the response body.
*   **Asynchronous HTTP Execution Engine (Worker Node):**
    *   Built in Rust (using `reqwest`/`tokio`) or Go (`fasthttp`).
    *   Implements connection pooling, HTTP/2 multiplexing, and jitter/rate-limiting to evade basic WAF rules.
*   **Smart Filter & Output Emitter (Data Sink):**
    *   Compares incoming successful hits against the calibration baseline using fuzzy hashing. If a page is 95% similar to the baseline custom 404, it is silently dropped.
    *   Outputs strictly in a standardized NDJSON schema specifically designed for pipeline ingestion.

### B. Data Flow Architecture
```text
[ httpx NDJSON ] -> (Ingestion Node) -> (Tech-Stack Analyzer)
                                              |
                                     (Dynamic Wordlist Builder)
                                              |
[ Job Queue / KV Store ] <==========> (Execution Engine) <--> [ Target Server ]
      |                                       |
(Infinite Recursion Protection)       (Heuristic Filter)
                                              |
                                      [ NDJSON Output Sink ] -> (Vulnerability Scanner / DB)
```

---

## 5. Enhancements & Plugin Integration

To ensure this tool outperforms existing open-source counterparts in an automated pipeline, it will incorporate the following advanced integrations:

### Dynamic Contextual Wordlists
Instead of relying solely on static text files (like `raft-large-directories`), the tool will feature a **Scraper Plugin**. When analyzing a target, it will parse previously collected JavaScript files and page text (using tools like `subjs` or `LinkFinder` outputs) to generate a custom wordlist specific to the target's internal naming conventions.

### Distributed & Multi-IP Scanning
The architecture will natively support gRPC-based clustering. A single master node will coordinate the state, while multiple ephemeral worker nodes (hosted on cheap VPS instances or serverless functions) pull tasks from the queue. This distributes the traffic across multiple IP addresses, drastically reducing the chance of IP bans and WAF blocks.

### Fuzzy Hashing for False Positive Eradication
By implementing algorithms like **SimHash** or **TLSH** (Trend Micro Locality Sensitive Hash), the tool will detect soft-404s and WAF challenge pages even if the content length changes slightly due to dynamic timestamps or reflection.

### Native Pipeline Integrations
*   **Nuclei Integration:** Discovered files/directories (e.g., `/.git/config`, `/api/user`) can be directly piped into Nuclei, automatically selecting only the templates relevant to that specific file path.
*   **ASM Dashboards:** Native webhooks will allow real-time reporting of high-value discoveries (e.g., `.env` files, `/admin` panels) directly to Slack, Discord, or vulnerability management platforms like Faraday and DefectDojo.
# HTTP Probing: Architecture & Methodology

## 1. Process & Methodology

In a professional bug bounty or penetration testing workflow, **HTTP Probing** (often referred to as web liveness checking) is the critical bridge between infrastructure discovery and web application testing. After enumerating subdomains, resolving IPs, and scanning for open ports, security practitioners are left with a massive list of potential targets. HTTP Probing answers the question: *"Which of these endpoints actually host web services, and what do they look like?"*

The standard methodology involves:
*   **Liveness Detection:** Attempting HTTP and HTTPS connections on standard (80, 443) and identified non-standard ports to confirm the presence of a web server.
*   **Metadata Extraction:** Capturing essential data points such as HTTP status codes, page titles, response headers, content length, and body hashes. 
*   **Routing & Redirect Resolution:** Following redirects to map the final destination of a service (e.g., mapping HTTP to HTTPS, or root domains to SSO login portals).
*   **Technology Fingerprinting:** Analyzing headers and response bodies to identify the underlying technology stack, Web Application Firewalls (WAFs), and Content Delivery Networks (CDNs).
*   **Wildcard & Catch-All Filtering:** Identifying patterns that indicate a catch-all DNS record or a default hosting provider page to filter out noise before passing data to heavy vulnerability scanners.

## 2. Current Tools & Ecosystem

The current ecosystem relies heavily on a few specialized, highly concurrent tools to handle thousands of hosts quickly:

*   **httpx (ProjectDiscovery):** The modern gold standard for HTTP probing. It is written in Go, extremely fast, and feature-rich. It handles liveness, status codes, tech detection (`-tech-detect`), CDN identification, and outputs highly structured JSON.
*   **httprobe (tomnomnom):** A minimalist, UNIX-philosophy tool designed to do one thing perfectly: take a list of domains and return a list of responding URLs. It is lightweight but lacks advanced metadata extraction.
*   **meg (tomnomnom):** A specialized tool designed to fetch many paths across many hosts efficiently without triggering rate limits on a single host. It is invaluable for mass-targeted endpoint probing rather than pure liveness checking.

## 3. Where Current Tools Lack

While existing tools are exceptionally fast, relying on them in automated, scaled pipelines reveals several critical shortcomings:

*   **Lack of State & Resumability:** Current tools operate as single-execution binaries. If a massive scan covering hundreds of thousands of hosts crashes or is interrupted, the progress is lost, and the scan must restart from scratch.
*   **Static Rate Limiting vs. Dynamic WAFs:** Tools typically rely on static concurrency flags. When hitting modern WAFs (Cloudflare, Akamai) or fragile infrastructure, they fail to adapt. They often trigger rate limits (HTTP 429) or IP bans (HTTP 403) without dynamically backing off, leading to incomplete data.
*   **Resource Management & Memory Spikes:** Running high-concurrency HTTP probes over massive datasets can cause local network interface exhaustion (file descriptor limits) and severe memory spikes, resulting in OOM (Out of Memory) kills.
*   **Poor Wildcard & Noise Handling:** While some tools support basic status code filtering, they struggle to automatically identify and group semantic duplicates (e.g., 500 subdomains all returning the exact same default IIS page or CDN error page).
*   **Non-Standard Output Across the Pipeline:** Pipelining smaller tools (like `httprobe` or `meg`) requires heavy `awk`/`sed` or `jq` parsing. Even when JSON is used, the schemas differ between tools, complicating database ingestion.

## 4. Proposed Tool Workflow & Architecture

To address these limitations, our new HTTP Probing module will be designed as a **stateful, asynchronous, and distributed microservice**.

### High-Level Architecture
1.  **Ingestion & Queueing Layer:**
    *   Accepts raw inputs (Domains, IPs, Port combos) from the DNS/Port Scanning phase via a message broker (e.g., RabbitMQ, Redis Streams) or direct API calls.
    *   Prioritizes targets based on previous phase confidence scores.
2.  **Adaptive Probing Engine:**
    *   Built on a high-performance asynchronous HTTP client (e.g., Go's `fasthttp` or Rust's `reqwest`).
    *   Implements a **Dynamic Backoff Controller**: Monitors HTTP 429 responses and connection resets, automatically dialing down concurrency per-ASN or per-domain.
3.  **Context Extraction & Analysis Layer:**
    *   Captures full request/response context: TLS certificates, HTTP/2 & HTTP/3 support, headers, body, and DOM elements.
    *   Generates Simhash/MinHash signatures of the response body to cluster visually and structurally identical pages.
4.  **State Management & Storage Database:**
    *   Maintains a persistent state file (or local SQLite/Postgres db) tracking `pending`, `in_progress`, `completed`, and `failed` jobs.
    *   Guarantees true resumability.
5.  **Output & Forwarding Layer:**
    *   Emits standardized JSON output conforming to a strict schema.
    *   Hooks directly into the next phases (Content Discovery, Vulnerability Scanning) via webhooks or message queues.

## 5. Enhancements & Plugin Integration

To significantly improve upon existing methods, the tool will feature the following enhancements:

*   **Fuzzy Hashing & Smart Deduplication:**
    Instead of passing 10,000 active endpoints to a fuzzer, the module will compute fuzzy hashes (e.g., Simhash) of response bodies. If 9,000 subdomains return the exact same parking page, they are grouped into a single cluster, and only one representative URL is passed down the pipeline.
*   **Distributed Scanning (Fleet Mode):**
    The architecture will natively support a master-worker configuration. A central controller can distribute chunks of the target list to ephemeral workers (VPS instances or serverless functions across multiple IP ranges/regions) to massively parallelize the scan and evade geographic IP blocking.
*   **Standardized JSON Telemetry:**
    Output will strictly adhere to an extensible JSON schema (e.g., `url`, `method`, `status_code`, `content_length`, `tls_data`, `tech_stack`, `fuzzy_hash`). This ensures seamless integration with UI dashboards (DefectDojo, Faraday) and downstream tools (Nuclei).
*   **Middleware Plugin System:**
    The engine will support WebAssembly (Wasm) or Lua plugins to manipulate requests dynamically:
    *   *Auth Plugins:* Automatically fetch and inject valid OAuth/JWT tokens into headers for authenticated surface mapping.
    *   *Evasion Plugins:* Inject headers like `X-Forwarded-For: 127.0.0.1` or `X-Custom-IP-Authorization` to probe for misconfigured access controls.
    *   *Tech Detection Plugins:* Pluggable regex and DOM-parsing rules (compatible with Wappalyzer databases) that execute instantly on the captured response stream.
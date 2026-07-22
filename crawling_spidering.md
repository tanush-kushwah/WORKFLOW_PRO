# Architecture & Methodology: Crawling and Spidering Phase

## 1. Process & Methodology

In a professional bug bounty or penetration testing workflow, the **Crawling / Spidering** phase acts as the bridge between broad infrastructure enumeration (horizontal recon) and targeted vulnerability discovery (vertical recon). Once subdomains, live hosts, and web services are identified, crawling systematically maps the internal structure of the target applications.

**Core Objectives:**
*   **Path & Endpoint Discovery:** Uncover unlinked or hidden directories, administrative interfaces, and legacy API routes.
*   **Asset Extraction:** Identify and download JavaScript files (for subsequent static analysis and secret extraction), CSS, and media.
*   **Parameter Gathering:** Harvest GET/POST parameters, form fields, and JSON payloads to feed into fuzzers (e.g., `ffuf`, `wfuzz`) and scanners.
*   **Contextual Reconnaissance:** Extract metadata such as developer comments, technology stacks, and integrated third-party services.

**The Workflow:**
Crawling is typically executed after HTTP probing (e.g., `httpx`). The inputs are confirmed live web servers. The process utilizes a hybrid approach: **Passive Spidering** (aggregating historical data via tools like `waybackurls` or `gau`) and **Active Crawling** (interacting with the live site). The active phase must navigate modern Single Page Applications (SPAs), requiring JavaScript execution and DOM rendering to uncover endpoints that static HTML parsing would miss. The output is a deduplicated list of URLs, files, and form definitions, serialized and piped into vulnerability scanners like `nuclei` or injection fuzzers.

---

## 2. Current Tools & Ecosystem

The current ecosystem offers a range of tools, balancing raw speed against deep application interaction:

*   **katana:** The current state-of-the-art by Project Discovery. It supports both passive data fetching and active, JS-aware crawling via headless browsers. It excels at form extraction and fits seamlessly into modern CLI pipelines.
*   **gospider:** A robust, highly concurrent Go-based crawler. It effectively parses `sitemap.xml` and `robots.txt` automatically and is heavily utilized for deep, multi-threaded site mapping.
*   **hakrawler:** A minimalist, fast crawler designed for the Unix philosophy. It is highly effective for quick link, JS file, and form extraction, operating primarily through static parsing.
*   **Photon:** Blurs the line between OSINT and web crawling. While it maps out endpoints, it also actively searches for emails, social media accounts, and exposed secrets, making it ideal for contextual engagements.
*   **crawley:** A lightweight crawler optimized for raw speed and basic link extraction, best used when executing massive, internet-scale sweeps where headless browsing is unfeasible.

---

## 3. Where Current Tools Lack

Despite the efficacy of the current toolset, integrating them into a highly scalable, automated, and continuous attack surface management (ASM) framework reveals critical shortcomings:

1.  **Memory Exhaustion & Lack of State:** Most current crawlers run completely in memory. When unleashed on massive enterprise applications (e.g., millions of products on an e-commerce site), they frequently suffer Out-Of-Memory (OOM) crashes. Furthermore, they lack resumability; if a crawler crashes at 95%, the process must restart from zero.
2.  **Inconsistent Output Schemas:** Tools output JSON, but the schemas differ wildly. Pipelining `hakrawler` output into a custom fuzzer requires different parsing logic than pipelining `katana` output, breaking seamless automation.
3.  **Crawler Traps & Infinite Loops:** Existing tools struggle with dynamic crawler traps (e.g., endless calendar components, infinite pagination, or dynamically generated URL directories). They often lack intelligent, heuristic-based deduplication beyond simple exact-match strings.
4.  **Inefficient Headless Execution:** Rendering DOMs for SPAs is resource-heavy. Tools like `katana` are excellent, but running headless Chromium at scale without intelligent DOM-mutation observers or resource-blocking (blocking images, fonts, analytics tracking) slows the pipeline dramatically.
5.  **Standalone Execution vs. Distributed Processing:** Current crawlers are designed as single-machine CLI utilities. They do not natively support distributed scanning across a fleet of worker nodes (e.g., via Redis or RabbitMQ queues).

---

## 4. Proposed Tool Workflow & Architecture

To address these limitations, our new Crawling / Spidering module will be built on a **Distributed, Event-Driven Architecture**, prioritizing state management, intelligent filtering, and standardized I/O.

### Core Architecture Components

*   **Ingestion Queue (Kafka / Redis / NATS):**
    *   Accepts verified live URLs from the HTTP Probing phase.
    *   Distributes URLs to a fleet of Crawler Workers to enable horizontal scaling across multiple VPS instances.
*   **Dual-Engine Crawler Workers:**
    *   **Engine A (Static / Fast):** Uses high-speed AST and regex parsing (similar to `hakrawler`). Processes standard HTML/text responses.
    *   **Engine B (Dynamic / Headless):** A lightweight, optimized headless browser (e.g., Playwright/Puppeteer via Go/Rust wrappers). Invoked *only* if Engine A detects SPA frameworks (React, Angular, Vue) via Wappalyzer-like signatures. Blocks all non-essential assets (CSS, images, tracking scripts) to minimize memory and bandwidth overhead.
*   **State & Deduplication Database (RocksDB / SQLite / Redis):**
    *   Tracks the state of the crawl on disk, allowing paused, stopped, or crashed jobs to resume instantly.
    *   Maintains a global hash set of normalized URLs to prevent revisiting the same endpoints.
*   **Heuristic Trap Detector:**
    *   Monitors URL path depth and query parameter entropy. Automatically aborts or blacklists paths that exhibit infinite recursive loops.
*   **Standardized Emitter (JSON-L):**
    *   Outputs results strictly to stdout (or a message broker) using a universal, heavily documented JSON schema.

### Data Flow Diagram

```text
[HTTP Probing Outputs] -> (Message Broker)
                                |
                                v
                      +-------------------+
                      | Dispatcher Router | --> (Target specific rate limiting)
                      +-------------------+
                                |
          +---------------------+---------------------+
          |                                           |
[Static Worker Engine]                      [Headless Worker Engine]
  (HTML, Text, XML)                           (SPAs, DOM Mutation)
          |                                           |
          +---------------------+---------------------+
                                |
                     [Heuristic Trap Filter]
                                |
                   [State / Deduplication DB]
                                |
                   [Standardized JSON Emitter] --> (To Fuzzers/Scanners)
```

---

## 5. Enhancements & Plugin Integration

To ensure our crawler exceeds the capabilities of existing ecosystem tools, the architecture will implement the following advanced features and plugin integrations:

### 1. Smart Parameter Canonicalization
Instead of treating `/?id=1` and `/?id=2` as distinct pages, the crawler will generate a structure signature (e.g., `/?id=<int>`). Once a threshold of structurally identical pages is reached, it ceases crawling that pattern to save time and resources, while passing the parameter key (`id`) to the fuzzing queue.

### 2. Standardized Extensible Output
All outputs will adhere to a strict JSON schema, ensuring downstream tools do not need custom parsers. 
```json
{
  "timestamp": "2026-07-22T10:00:00Z",
  "source_url": "https://target.com/app",
  "discovered_endpoint": "https://target.com/api/v1/users",
  "method": "GET",
  "parameters": ["id", "token"],
  "content_type": "application/json",
  "technologies": ["React", "Express"]
}
```

### 3. Modular Plugin System (The "Hook" Engine)
The crawler will expose an API for lightweight Lua or JavaScript plugins that execute during the crawl phase.
*   **Secret Extraction Plugin:** Uses high-entropy regex (similar to `SecretFinder` or `trufflehog`) on the fly to detect AWS keys or JWTs in extracted JS files without needing a separate pipeline step.
*   **WAF / Rate-Limit Evasion Plugin:** Integrates with proxy rotators. If a `429 Too Many Requests` or `403 Forbidden` (via Cloudflare/AWS WAF) is detected, the worker automatically pauses, rotates its IP, and resumes from the tracked state.
*   **Form Analytics Plugin:** Parses discovered `<form>` tags, identifies inputs, and translates them into ready-to-use Nuclei templates or JSON request bodies for downstream injection testing.

### 4. Cross-Phase Context Sharing
Integration with the **JavaScript Analysis** phase. Whenever a `.js` file is discovered, the crawler will instantly fork the URL to a dedicated JS parsing queue (mimicking `LinkFinder` logic) to extract deep API routes, subsequently feeding those new routes back into the main Crawling queue.
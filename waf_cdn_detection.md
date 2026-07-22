# Architecture & Methodology: WAF and CDN Detection

## 1. Process & Methodology

In modern offensive security workflows, WAF (Web Application Firewall) and CDN (Content Delivery Network) detection is a critical triage phase that occurs immediately after asset discovery and port scanning, but before intensive vulnerability scanning or fuzzing. 

The primary objectives of this phase are:
*   **Infrastructure Mapping:** Determining if the target resolves directly to an origin server or is masked behind edge infrastructure (e.g., Cloudflare, Fastly, Akamai).
*   **Scan Calibration:** WAFs and CDNs aggressively block high-rate or malformed traffic. Identifying them allows the overarching framework to dynamically adjust rate limits, thread counts, and timeout configurations, preventing IP bans.
*   **Payload Tailoring:** Knowing the specific WAF vendor enables the pipeline to deploy targeted evasion techniques and bypass payloads.
*   **Origin Hunting:** If a CDN is detected, the workflow can fork a parallel process to hunt for origin IP leaks (e.g., checking historical DNS, Shodan, or Censys for matching SSL certificates).

The methodology typically follows a layered approach:
1.  **DNS & IP Profiling:** Checking A records and CNAMEs against known CDN IP ranges (CIDR blocks).
2.  **Passive HTTP Probing:** Sending a benign `GET` request and analyzing the HTTP response headers (e.g., `Server`, `X-Powered-By`, `X-Cache`) and cookies.
3.  **Active Heuristic Probing:** Sending intentional, safe anomalous requests (e.g., basic cross-site scripting or SQL injection payloads) to trigger a block page, then fingerprinting the resulting error response.

## 2. Current Tools & Ecosystem

The current ecosystem relies heavily on a few specialized, standalone tools:

*   **wafw00f:** The industry standard for WAF fingerprinting. It operates primarily via active heuristic probing, sending standard requests followed by known malicious payloads. It compares the responses against a massive signature database of block pages, headers, and cookies to identify the WAF vendor.
*   **cdncheck:** A lightweight, highly efficient tool (typically built in Go) used to determine if an IP address belongs to a known CDN or Cloud provider. It operates passively by comparing the target IP against frequently updated lists of provider CIDR ranges.

## 3. Where Current Tools Lack

While effective individually, integrating these tools into a highly scalable, automated attack surface mapping framework presents several challenges:

*   **Non-Standardized Output and Pipelining:** `wafw00f` was originally designed as a standalone Python CLI tool. While it supports JSON output, error handling and output streams can occasionally break pipeline chaining. 
*   **Redundant Probing (Lack of State):** Existing tools evaluate targets in isolation. In a large attack surface, thousands of subdomains may point to the same Cloudflare IP. Current tools will redundantly send HTTP probes to all of them, wasting bandwidth and increasing the risk of getting blacklisted.
*   **Binary Identification without Context:** Current tools report *what* the WAF is, but provide no actionable feedback loop to the orchestrator regarding *how* to handle it (e.g., suggested rate limits, required header permutations).
*   **High False Positive Rates on Edge Cases:** Custom WAF rules or stripped HTTP headers often cause signature-based tools to fail or misidentify the firewall.

## 4. Proposed Tool Workflow & Architecture

To address these shortcomings, the new WAF/CDN Detection module must be designed as a state-aware, highly concurrent microservice within the broader framework.

### Module Architecture

1.  **Input Ingestion (NdJSON):** The module accepts a stream of resolved assets (URLs or IP addresses) from the DNS/Port Scanning phase via standard input.
2.  **Shared State Cache:** An in-memory key-value store (e.g., Redis or native memory map) tracks processed IP addresses to deduplicate network requests. 
3.  **Phase 1: Local CIDR & CNAME Resolution (Zero-Packet):**
    *   Compare the target IP against a locally cached, auto-updating database of CDN/Cloud CIDR ranges (similar to `cdncheck`).
    *   Analyze the CNAME records for known vendor domains (e.g., `*.cloudfront.net`).
    *   *Decision Gate:* If identified as a CDN, tag the asset and proceed to Phase 2 for WAF analysis.
4.  **Phase 2: Passive HTTP Fingerprinting:**
    *   Send a standard `GET /` request.
    *   Extract and hash the response headers, cookies, and body.
    *   Match against a localized signature set.
5.  **Phase 3: Active Heuristic Probing (Configurable):**
    *   If Phase 2 yields low confidence, send a calibrated anomalous request (e.g., `GET /?id=1' OR '1'='1`).
    *   Capture the block page structure, status code (e.g., 403, 406), and headers.
6.  **Data Structuring & Output:** Emit a highly structured JSON object detailing the findings, confidence score, and routing tags.

### Data Model (JSON Output)
```json
{
  "target": "https://api.example.com",
  "ip": "192.0.2.1",
  "cdn": {
    "detected": true,
    "provider": "Cloudflare"
  },
  "waf": {
    "detected": true,
    "provider": "Cloudflare Bot Management",
    "confidence": "high",
    "trigger_method": "active_probe_xss"
  },
  "recommended_action": "throttle",
  "timestamp": "2026-07-22T14:30:00Z"
}
```

## 5. Enhancements & Plugin Integration

The new module will introduce several advanced capabilities to out-perform legacy tools:

*   **Dynamic Feedback Loop (Smart Orchestration):** The module will append a `recommended_action` tag to its output. Downstream modules (like directory bruteforcers or vulnerability scanners) will parse this tag to automatically apply specific rate limits (e.g., max 5 requests/second) or rotate User-Agents to avoid triggering the identified WAF.
*   **Stateful Deduplication:** If `sub1.example.com` and `sub2.example.com` resolve to the exact same CDN IP, the module will only execute the active HTTP WAF probe once, copying the result to all associated subdomains. This drastically reduces scan time and noise.
*   **Origin Discovery Integration:** When a CDN is detected, the module will automatically trigger an asynchronous "Origin Hunt" plugin. This plugin will query historical DNS databases, Shodan, and Censys to find the true backend IP, attempting to bypass the WAF entirely.
*   **Distributed Probing Engine:** For environments with aggressive IP banning, the active heuristic probing phase can be configured to route requests through rotating proxy pools or serverless functions (e.g., AWS Lambda), ensuring the core framework IP remains unblocked.
*   **Machine Readable Signatures:** Transitioning away from hardcoded Python regex rules to a YAML-based signature format (similar to Nuclei templates) for WAF block pages, allowing the community to easily contribute and update WAF signatures without modifying the source code.
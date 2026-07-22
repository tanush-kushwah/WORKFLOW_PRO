# Architecture & Methodology: Subdomain Takeover Detection Module

## 1. Process & Methodology

Subdomain takeover occurs when a DNS record (typically a CNAME) points to a third-party service (e.g., AWS S3, GitHub Pages, Heroku, Azure) that has been deprovisioned, deleted, or abandoned by the original owner, while the DNS routing remains active. This allows an attacker to register the abandoned resource on the third-party provider and effectively hijack the subdomain, leading to reputational damage, phishing, cookie stealing, or cross-site scripting (XSS).

In a professional bug bounty or penetration testing workflow, Subdomain Takeover Detection is typically phase three, executed immediately after Subdomain Enumeration and DNS Resolution. The methodology follows a strict sequence:

1.  **Ingestion:** A list of previously resolved, live (and sometimes dead) subdomains is ingested.
2.  **DNS Verification:** The module queries the target subdomain for `CNAME`, `A`, or `AAAA` records to identify if traffic is being routed to a known third-party cloud provider or SaaS platform.
3.  **HTTP Probing & Fingerprinting:** If a target points to a third-party service, an HTTP/HTTPS GET request is sent. The HTTP response (status code, headers, and body) is analyzed against a database of known "unclaimed resource" signatures (e.g., AWS's `NoSuchBucket`, GitHub's `There isn't a GitHub Pages site here`).
4.  **Edge-case Checking:** Advanced checks resolve `NXDOMAIN` statuses on the provider side to confirm the resource is truly available for registration.
5.  **Reporting:** High-confidence takeovers are flagged and escalated for manual verification and proof-of-concept (PoC) generation.

## 2. Current Tools & Ecosystem

The ecosystem currently relies on a mix of dedicated takeover scanners and generalized template scanners:

*   **subzy & tko-subs:** Dedicated, fast CLI tools written in Go. They are purpose-built for this task, taking a list of subdomains and running them concurrently against a hardcoded or locally stored JSON file of takeover signatures.
*   **nuclei (takeover templates):** A highly flexible, generalized vulnerability scanner. The `takeovers` template pack uses YAML-based DSL to define DNS and HTTP checks. It has become the industry standard due to the ease of updating signatures.
*   **can-i-take-over-xyz:** Not a scanning tool, but the canonical community-driven repository documenting which services are vulnerable to takeover, how to verify them, and their specific HTTP/DNS fingerprints. This repository acts as the upstream source of truth for almost all takeover tools.

## 3. Where Current Tools Lack

While existing tools are effective, they present several architectural and operational bottlenecks when integrated into a highly scalable, automated enterprise or bug bounty pipeline:

*   **High False Positive Rates:** Many tools rely solely on simple string matching in the HTTP response body. If a WAF or custom 404 page mirrors the signature string inadvertently, a false positive is triggered.
*   **Lack of State & Retries:** Network jitter or temporary rate-limiting by cloud providers (like AWS or Cloudflare) often results in dropped connections. Existing tools usually fail open or drop the target, leading to missed takeovers.
*   **Rigid Signature Updates:** Dedicated tools (like `subzy` or `tko-subs`) often require recompilation, manual flag overrides, or pulling new JSON files to update signatures. They lack dynamic, automatic syncing with upstream sources like `can-i-take-over-xyz`.
*   **Poor Pipelining Integration:** Outputs are often non-standard or mix logging information with results. Integrating these tools into a continuous Bash/Go pipeline requires heavy use of `grep`, `awk`, or `jq` to normalize the data.
*   **Inefficient Resolution Phasing:** Many tools attempt HTTP requests before strictly verifying if the DNS CNAME actually points to a vulnerable provider, wasting bandwidth and time on internal or bare-metal IPs.

## 4. Proposed Tool Workflow & Architecture

To address these shortcomings, the new Subdomain Takeover Detection module should be designed as a microservice-ready CLI tool with a decoupled, event-driven architecture. 

### Core Components:

1.  **Input & Normalization Pipeline (Stdin/File):**
    *   Accepts raw subdomains, URLs, or structured JSON from upstream tools (like `dnsx` or `httpx`).
    *   Automatically strips protocols and trailing slashes to isolate the FQDN.

2.  **Two-Tier Analysis Engine:**
    *   **Tier 1: DNS Fast-Path (The Filter):** Before any HTTP requests are made, the tool asynchronously resolves the `CNAME`. It compares the CNAME target against a generalized regex map of known cloud providers (e.g., `.*\.s3\.amazonaws\.com`, `.*\.github\.io`). Subdomains failing this check are safely dropped to conserve resources.
    *   **Tier 2: HTTP Verification (The Prober):** For domains that pass Tier 1, the engine dials the endpoint (handling both HTTP and HTTPS). It captures the response body, headers, and status code.

3.  **Context-Aware Signature Matching:**
    *   Signatures are defined in a standardized JSON/YAML schema that requires *multiple* conditions to trigger a match (e.g., `Status Code == 404` AND `Body contains "NoSuchBucket"` AND `CNAME matches s3.amazonaws.com`).
    *   This boolean AND/OR logic drastically reduces false positives compared to flat string matching.

4.  **Standardized Output Formatter:**
    *   Outputs strictly in JSONL (JSON Lines) format by default, separating standard logging (stderr) from actionable results (stdout).

### System Architecture Diagram (Conceptual)
```text
[Upstream Pipeline] -> (stdin) -> [Input Buffer]
                                        |
                                [DNS Tier 1 Filter] <--- (Provider CNAME Regex)
                                        |
                                 [Worker Pool]
                                        |
                             [HTTP Tier 2 Prober] <--- (Retry/Timeout Logic)
                                        |
                            [Signature Matcher] <--- (Dynamic Signature DB)
                                        |
                                [JSONL Output] -> (stdout) -> [Downstream/DB]
```

## 5. Enhancements & Plugin Integration

To make this module superior to existing open-source alternatives, the following enhancements should be engineered into the core:

*   **Dynamic Signature Sync:** The tool should feature a `--update` flag or auto-sync mechanism that fetches the latest fingerprint definitions directly from the `can-i-take-over-xyz` repository and compiles them into its local database, ensuring zero-day takeover methods are immediately actionable.
*   **Distributed / Sharded Scanning:** For enterprise attack surface mapping (handling millions of subdomains), the tool should support integration with messaging queues (like Redis or RabbitMQ). A master node can shard the subdomain list, allowing multiple lightweight worker nodes to process the takeover checks concurrently from different geographic IP spaces (evading regional rate limits).
*   **Web-Hook & Notification Integrations:** Native support for Slack, Discord, or custom HTTP webhooks. Given the ephemeral nature of subdomain takeovers (where a vulnerable domain can be claimed by a competitor or malicious actor within minutes), real-time alerting is critical.
*   **"Claiming" SDK Hooks (Advanced):** While the core tool is a detector, it can expose an interface for user-defined Python or Go scripts. If a takeover is confirmed, the tool can trigger a specific script (e.g., `aws_claim_s3.py`) to automatically register the resource and upload a harmless PoC file, securing the vulnerability before an adversary finds it. 
*   **Smart Rate-Limit Backoff:** Implement adaptive rate limiting. If a specific cloud provider's edge network begins returning `429 Too Many Requests`, the tool should dynamically back off threads for that specific CNAME pattern while continuing to scan other providers at maximum speed.
# Architecture and Methodology: Parameter Discovery

## 1. Process & Methodology

Parameter discovery is a critical phase in modern attack surface mapping and web application security testing. As applications grow in complexity, developers frequently leave undocumented, hidden, or deprecated parameters in the codebase. These parameters may be used for debugging, feature flagging, internal API routing, or legacy support, and they often bypass standard input validation, making them prime targets for vulnerabilities like Server-Side Request Forgery (SSRF), Cross-Site Scripting (XSS), Insecure Direct Object References (IDOR), and SQL Injection (SQLi).

In a professional testing workflow, the parameter discovery phase typically follows endpoint discovery and crawling. The methodology involves two primary approaches:

*   **Passive Discovery:** Mining historical traffic, open-source intelligence (OSINT), and internet archives (e.g., Wayback Machine, Common Crawl, AlienVault OTX) to find parameters that were previously exposed or logged.
*   **Active Discovery (Bruteforcing & Fuzzing):** Sending crafted HTTP requests (GET, POST, JSON, XML) appending large wordlists of common parameter names to known endpoints. The response behavior is then analyzed to determine if the server processed the injected parameter. 

The core challenge in active discovery is distinguishing a true parameter response from dynamic page changes, WAF interventions, or generic server reflections. This requires advanced behavioral analysis and response diffing—measuring factors like HTTP status codes, content length, response timing, and DOM structure alterations.

## 2. Current Tools & Ecosystem

The current ecosystem for parameter discovery relies on a mix of highly specialized tools and flexible fuzzers:

*   **Arjun:** A staple in active parameter discovery, utilizing an advanced heuristic approach to identify parameters and bypass WAFs by packaging multiple parameters in a single request (e.g., chunking).
*   **x8:** Written in Rust, x8 is designed for extreme speed and accuracy, leveraging sophisticated response-diffing algorithms to eliminate false positives caused by dynamic content.
*   **param-miner:** A highly effective Burp Suite extension that automates the discovery of hidden parameters and headers during manual or automated proxy testing, specifically excelling in identifying cache poisoning vectors.
*   **ParamSpider:** Focuses on the passive methodology by mining historical archives for endpoints and parameters that the application has naturally used over time.
*   **gf (Grep Formatter):** A utility that acts as a filter, allowing practitioners to take massive lists of discovered URLs and sort them by potential vulnerability categories based on the parameter names present (e.g., filtering for `?url=` or `?redirect=` for SSRF/Open Redirect testing).

## 3. Where Current Tools Lack

While highly effective in isolation, existing tools exhibit several shortcomings when integrated into a continuous, highly scalable attack surface mapping framework:

*   **Lack of State and Context Awareness:** Most parameter bruteforcers treat every endpoint as a blank slate. They do not maintain state or learn from the application. If `?user_id=` is discovered on `/api/v1/users`, current tools do not automatically prioritize `user_id` when fuzzing `/api/v1/settings`.
*   **Inefficient Data Pipelining and Standardization:** Tools like Arjun and x8 have their own output formats. Integrating passive tools (ParamSpider) with active tools (x8) requires intermediate parsing scripts. A lack of standard JSON/NDJSON data contracts makes automated pipelining fragile.
*   **Handling of Complex Data Types:** Many tools struggle with nested JSON structures, GraphQL mutations, or multi-part form data. They excel at simple `?param=value` GET queries or flat POST bodies, but fail to discover parameters hidden deep within complex API request structures.
*   **False Positives in Modern SPAs:** Highly dynamic Single Page Applications (SPAs) and modern CDNs frequently alter response lengths and content dynamically (e.g., changing CSRF tokens or cache headers). This causes traditional length-diffing algorithms to produce high false-positive rates.
*   **Rate Limiting and Network Bottlenecks:** Active discovery is inherently noisy. Existing tools often lack distributed execution capabilities out of the box, making them susceptible to immediate blocking by modern Web Application Firewalls (WAFs).

## 4. Proposed Tool Workflow & Architecture

To address these shortcomings, the new Parameter Discovery module should be designed as a distributed, state-aware engine. The architecture is divided into the following core components:

### A. Ingestion and Context Engine
*   **Input Standardizer:** Accepts a standardized JSON schema containing endpoints, known headers, valid authentication tokens, and previously discovered parameters.
*   **Contextual Analyzer:** Analyzes the baseline behavior of the endpoint before any fuzzing begins. It records baseline response times, structural DOM hashes, and dynamic reflection points.

### B. Payload Generation Module
*   **Smart Wordlist Generator:** Instead of using a static wordlist, this module generates a dynamic list based on the target's context. It merges universal parameter lists (e.g., `id`, `debug`) with context-specific lists derived from passive discovery (Wayback machine) and JavaScript file analysis.
*   **Format Mutator:** Automatically translates the parameter list into all relevant structural formats: GET query strings, form-urlencoded POST bodies, flat JSON, nested JSON, and XML.

### C. Execution and Routing Engine
*   **Multiplexing & Chunking:** Implements Arjun-style parameter chunking (sending 50-100 parameters per request) to reduce network overhead. If a chunk triggers a behavioral change, it automatically splits and recurses (binary search) to isolate the valid parameter.
*   **Distributed Task Queue:** Built on a message broker (e.g., Redis or RabbitMQ) to distribute chunks across multiple worker nodes/proxies, avoiding single-IP rate limits and WAF bans.

### D. Heuristic Diffing Engine
*   **Multi-Factor Diffing:** Moves beyond simple content-length comparisons. The engine must compare:
    *   HTTP Status Codes and Headers
    *   Abstract Syntax Tree (AST) or structural DOM changes (ignoring dynamic text like timestamps)
    *   Reflected input analysis (tracking where and how the payload is reflected)
    *   Response timing anomalies (identifying boolean/time-based parameter execution)

### E. Output & Data Contract
*   **NDJSON Emitter:** Streams discovered parameters immediately as NDJSON, ensuring downstream vulnerability scanners (like Nuclei) can consume the data in real-time without waiting for the entire module to finish.

## 5. Enhancements & Plugin Integration

To ensure the module remains cutting-edge, several advanced enhancements should be implemented:

*   **Machine Learning / Statistical Anomaly Detection:** Implement a statistical anomaly detection model in the Heuristic Diffing Engine. By calculating the standard deviation of baseline responses, the tool can programmatically identify true parameter reflections even when WAFs or CDNs inject highly variable dynamic content.
*   **Continuous Feedback Loop (Integration with JS Analysis):** Integrate directly with the JavaScript Analysis phase. When a LinkFinder-style tool discovers a parameter in a client-side `.js` file, it should immediately push that parameter to the top of the queue for the Active Parameter Discovery module.
*   **WAF Evasion Plugin System:** Introduce a plugin architecture for request mutation. Plugins can automatically apply HTTP Request Smuggling techniques, character encoding (URL, Unicode, Hex), or header manipulation (e.g., `X-Forwarded-For`) to bypass edge protections during the discovery phase.
*   **Automated Type Inference:** When a parameter is discovered, the tool should perform a micro-fuzz to determine its accepted data type (integer, string, boolean, array). This enriched data (e.g., `{"parameter": "user_id", "type": "integer"}`) is passed to the next phase, drastically reducing the payload space required for subsequent vulnerability scanning.
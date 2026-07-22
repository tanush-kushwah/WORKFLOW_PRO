# Architecture & Methodology: CMS Scanners Phase

## 1. Process & Methodology

In a professional bug bounty or penetration testing workflow, the **CMS (Content Management System) Scanning** phase occurs after the initial reconnaissance, port scanning, HTTP probing, and technology fingerprinting phases. Once a target is identified as running a specific CMS (e.g., WordPress, Drupal, Joomla), dedicated scanning techniques are applied to extract granular, CMS-specific attack surface data.

The methodology typically follows a structured pipeline:
1. **CMS Verification:** Confirming the CMS and identifying its core version via meta tags, specific file hashes, or response headers.
2. **Component Enumeration:** Bruteforcing or passively extracting the names and versions of installed plugins, themes, modules, and extensions.
3. **User Enumeration:** Extracting usernames (e.g., via WP REST API, author archive pages, or predictable XML-RPC behaviors) to fuel subsequent password spraying or bruteforcing attacks.
4. **Vulnerability Correlation:** Cross-referencing the discovered core version and component versions against known CVEs and vulnerability databases.
5. **Misconfiguration Detection:** Searching for exposed debug logs, unauthenticated configuration files (e.g., `wp-config.php.bak`), enabled XML-RPC, or excessive directory listings.

This phase is highly critical because CMS platforms are heavily targeted, often poorly maintained by end-users, and rely on vast ecosystems of third-party plugins that frequently introduce critical vulnerabilities (RCE, SQLi, LFI).

---

## 2. Current Tools & Ecosystem

The current ecosystem relies heavily on specialized, monolithic tools tailored either to a single CMS or a small group of popular platforms. Based on our reference data, the landscape includes:

*   **Dedicated Single-CMS Scanners:**
    *   **wpscan:** The undisputed industry standard for WordPress. It features comprehensive plugin/theme enumeration, user extraction, and vulnerability mapping.
    *   **droopescan:** A scanner specifically focused on Drupal (with some Silverstripe support), adept at identifying core versions and installed modules.
    *   **joomscan:** An OWASP project dedicated to Joomla, focusing on core vulnerabilities and component detection.
*   **Multi-CMS Scanners:**
    *   **CMSeeK:** A detection and scanning tool that casts a wider net, supporting a massive range of different CMS platforms.
    *   **cmsmap:** A multi-CMS scanner prioritizing WordPress, Joomla, and Drupal, aiming to unify the functionality of individual tools into one interface.

---

## 3. Where Current Tools Lack

While effective in manual testing scenarios, the existing suite of CMS scanners presents significant friction when integrated into a highly scalable, automated attack surface mapping framework:

*   **Dependency & Environment Hell:** The ecosystem is fragmented across different languages and environments. `wpscan` requires Ruby, `droopescan` and `cmsmap` require Python, and `joomscan` relies on Perl. Maintaining these dependencies in a distributed scanning environment is an operational nightmare.
*   **Lack of Standardized Output:** Each tool produces completely different output structures. Parsing the results of `wpscan`, `joomscan`, and `droopescan` into a unified pipeline requires maintaining multiple brittle regex or JSON parsers.
*   **API Rate Limiting & Commercial Paywalls:** Tools like `wpscan` rely on live, paid API queries (WPVulnDB) to map vulnerabilities. In a continuous monitoring or bug bounty scenario across thousands of assets, this approach immediately hits rate limits and incurs massive costs.
*   **Monolithic & Single-Threaded Inefficiencies:** Many legacy scanners are not designed for asynchronous, internet-scale scanning. They operate sequentially and often hang on dead hosts or aggressive WAFs, bottlenecking the entire recon pipeline.
*   **No State Tracking (Diffing):** Current tools operate in a vacuum. They do not remember what plugins were installed yesterday versus today, making it impossible to alert purely on *new* attack surface changes.

---

## 4. Proposed Tool Workflow & Architecture

To build a modern, scalable CMS scanning module, we must abandon the "wrapper" approach (running legacy tools via subprocesses) and instead build a native, high-performance engine in a compiled language (e.g., Go or Rust). 

### Architecture Design

1. **Pre-Scan Routing (The Dispatcher):**
   *   The module consumes input from the Tech Fingerprinting phase (e.g., `httpx` or `webanalyze`).
   *   Instead of blindly firing CMS scanners at every web server, the dispatcher dynamically routes targets to specific CMS micro-modules based on verified fingerprints.
2. **Asynchronous Enumeration Engine:**
   *   A core HTTP engine handles all requests asynchronously, utilizing connection pooling, HTTP/2 multiplexing, and smart timeouts.
   *   It performs concurrent directory bruteforcing (for plugins/themes) using optimized, CMS-specific wordlists.
3. **Passive Extraction Subsystem:**
   *   Before sending any bruteforce traffic, the engine parses the homepage DOM, CSS, and JavaScript files to extract plugin/theme paths and versions passively using regular expressions, minimizing network noise.
4. **Local Vulnerability Correlation (Offline DB):**
   *   The framework will completely bypass live API rate limits. A background cron job will sync public vulnerability databases (NVD, OSV, GitHub Security Advisories) into a localized, optimized SQLite or DuckDB instance.
   *   Discovered components are queried against this local database in milliseconds.
5. **Standardized Unified Schema:**
   *   Regardless of whether the target is WP, Drupal, or Magento, the output is standardized into a single JSON schema:
     ```json
     {
       "target": "https://example.com",
       "cms": {"name": "WordPress", "version": "6.3.1"},
       "components": [
         {"type": "plugin", "name": "woocommerce", "version": "8.0.1", "vulnerabilities": ["CVE-XXXX-XXXX"]}
       ],
       "users": ["admin", "editor"],
       "misconfigurations": ["xmlrpc_enabled"]
     }
     ```

---

## 5. Enhancements & Plugin Integration

To ensure this new tool surpasses existing solutions, the following enhancements and integration capabilities will be built into the architecture:

*   **Template-Based Detection (Nuclei Integration):**
    Instead of hardcoding detection logic for new plugins, the scanner will consume YAML-based templates (similar to Project Discovery's `nuclei`). When a new critical CMS plugin vulnerability drops, users can simply drop a YAML template into the pipeline rather than waiting for a core software update.
*   **WAF Evasion & Smart Throttling:**
    The engine will feature global rate limiting, automatic jitter (randomized delays between requests), User-Agent rotation, and automatic back-off if WAF blocks (e.g., 403, 429 status codes, or Cloudflare captchas) are detected.
*   **Headless JS Analysis:**
    Modern CMS setups often use heavy caching or headless frontends (e.g., React/Next.js pulling from a WP REST API). The architecture will support optional headless browser integration (via CDP) to render pages and extract API endpoints or component fingerprints that are invisible to raw HTTP GET requests.
*   **Distributed Queue Architecture:**
    The engine will be designed to run as a worker node consuming from a message broker (RabbitMQ/Kafka/Redis). For a scope of 100,000 subdomains, the orchestrator can spin up 50 CMS scanner instances in Kubernetes, allowing the workload to scale horizontally.
*   **Stateful Diffing Engine:**
    Scans will be stored in a state database. Upon subsequent scans, the module will only push alerts to the user (via Slack/Discord/Webhook) if a *new* plugin is installed, a version is downgraded, or a *new* user is enumerated, dramatically reducing alert fatigue in continuous bug bounty monitoring.
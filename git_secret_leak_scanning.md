# Architecture and Methodology: Git / Secret Leak Scanning

## 1. Process & Methodology
In a professional bug bounty or penetration testing workflow, Git and Secret Leak Scanning is a critical phase for discovering hardcoded credentials, API keys, private keys, and sensitive configuration files. This phase often bridges the gap between external reconnaissance and internal exploitation. 

The core methodology follows these sequential steps:
1.  **Target Enumeration:** Identifying all code repositories associated with the target. This includes official GitHub/GitLab/Bitbucket organizations, personal repositories of developers (OSINT), and exposed `.git` directories on live web servers.
2.  **Data Acquisition:** Fetching the repository data. For shallow recon, this means checking the latest commit. For deep recon, it involves cloning the entire commit history, all branches, and pull requests to find secrets that were committed and subsequently "deleted" in later commits.
3.  **Static Analysis & Extraction:** 
    *   **Pattern Matching:** Using regular expressions to find known key formats (e.g., `AKIA...` for AWS, `xoxb-...` for Slack).
    *   **Entropy Analysis:** Calculating Shannon entropy to identify highly random strings that don't match specific regexes but are likely cryptographic keys or passwords.
    *   **File Metadata Analysis:** Scanning for sensitive filenames, extensions, or paths (e.g., `id_rsa`, `.env`, `config.bak`).
4.  **Validation:** Testing the extracted secrets against the respective provider's API to determine if they are active and what permissions they hold, effectively filtering out revoked keys and false positives.

## 2. Current Tools & Ecosystem
The current ecosystem relies heavily on a few specialized tools, each with distinct philosophies:

*   **Trufflehog:** The heavyweight champion for deep scanning. It effectively combines regex and entropy analysis across git history, S3 buckets, and other sources. Its modern iterations also include active verification of discovered keys.
*   **Gitleaks:** Widely considered the standard for fast, regex-based scanning. It is heavily utilized in DevSecOps pipelines but translates perfectly to offensive recon due to its speed and comprehensive default ruleset.
*   **Gitrob:** An older but conceptually important tool that focuses on identifying sensitive files by name and extension across an organization's public GitHub footprint, rather than solely scanning file contents.
*   **Git-secrets:** Originally developed by AWS to prevent commits containing secrets, it can be repurposed to scan existing repositories for AWS-specific credential leaks.

## 3. Where Current Tools Lack
While powerful, relying on standalone execution of these tools in an automated attack surface mapping pipeline presents several critical architectural challenges:

*   **Non-Standardized Outputs:** Each tool outputs data differently (varying JSON schemas, raw text, or console-only output). This makes chaining them into a unified pipeline (e.g., feeding results into a vulnerability manager or graph database) incredibly brittle.
*   **Lack of State Tracking:** Existing tools are largely stateless. If an organization has 500 repositories and a new scan is run a week later, tools typically re-clone and re-scan the entire history, wasting massive amounts of bandwidth, memory, and time. There is no native "resume from last checked commit" feature for external offensive recon.
*   **Disk & Memory Inefficiency:** Deep-cloning hundreds of large repositories to disk before scanning creates severe I/O bottlenecks and bloats storage.
*   **High False Positive Rates (Context Blindness):** Entropy-based scanners frequently flag test data, dummy API keys in documentation, or compiled binary blobs. The tools lack the contextual awareness to recognize a `tests/fixtures/` directory or a variable named `DUMMY_TOKEN`.
*   **Siloed Execution:** Tools designed for GitHub don't easily accept exposed `/.git/` directories found via web fuzzers (like `ffuf` or `dirsearch`) without manual intermediary steps (like using `git-dumper`).

## 4. Proposed Tool Workflow & Architecture
To solve these issues, our framework will implement a centralized **Secret Discovery Module (SDM)** designed for statefulness, in-memory processing, and modularity.

### Architecture Components:
1.  **Ingestion & Targeting Engine:**
    *   Accepts diverse inputs: GitHub/GitLab Org URLs, lists of developer usernames, or direct URLs to exposed `.git` web directories.
    *   Resolves all targets into a standardized URI format.
2.  **Smart Fetcher (In-Memory Git Client):**
    *   Instead of cloning to disk, the fetcher streams git objects (blobs, trees, commits) directly into memory using libraries like `go-git` or `libgit2`.
    *   Queries a local State Database (SQLite/Redis) to check the `latest_scanned_commit_hash` for a given repo. It only fetches and processes commits that occurred *after* this hash.
3.  **Multi-Engine Scanner:**
    *   **Regex Engine:** Uses a centralized, frequently updated YAML ruleset (merging rules from Gitleaks and Trufflehog).
    *   **Entropy Engine:** Evaluates strings but applies negative-weighting to strings found in known "safe" contexts.
    *   **File Profiler:** Flags sensitive configurations (`.env`, `*.pem`, `wp-config.php`).
4.  **Context & Heuristics Filter:**
    *   Drops matches found in paths containing `test`, `mock`, `example`, or `.min.js` unless explicitly overridden.
5.  **Active Validator (The Verifier):**
    *   An asynchronous worker pool that takes discovered secrets and safely attempts non-destructive authentication against the target APIs (AWS STS, GitHub API, Slack Auth) to append a `{"verified": true}` tag to the finding.

## 5. Enhancements & Plugin Integration
To ensure the new tool outpaces existing implementations, it will feature the following architectural enhancements:

*   **Standardized NDJSON Output:** All findings—regardless of whether they were found via regex, entropy, or filename—will be emitted as Newline Delimited JSON. This ensures seamless piping into data visualization tools, SIEMs, or subsequent exploit modules. 
    *   *Schema:* `{"timestamp", "target", "repo", "commit", "author", "secret_type", "snippet", "verified", "severity"}`
*   **Distributed Scanning & Worker Nodes:** The architecture will support a message broker (e.g., RabbitMQ or Redis Pub/Sub). The Ingestion Engine will push individual repository URIs to a queue, allowing multiple lightweight worker nodes to scan repositories concurrently.
*   **Web-Recon Plugin Integration:** The module will expose an internal API or socket. When the HTTP Probing or Content Discovery phases (using tools like `httpx` or `ffuf`) identify an open `/.git/` directory, they immediately send an event to this module. The Smart Fetcher will dynamically dump the remote git index and scan it on the fly.
*   **AI/ML False Positive Reduction:** Integration with a lightweight local LLM or a pre-trained Bayes classifier to evaluate the code snippet surrounding a high-entropy string. By analyzing variable names (e.g., `test_key` vs `prod_db_pass`), the system can assign a confidence score, dramatically reducing analyst fatigue.
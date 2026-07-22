# Phase: API & GraphQL Discovery
## Architecture & Methodology Document

---

### 1. Process & Methodology

In modern web applications, the presentation layer (frontend) and the data layer are heavily decoupled, communicating almost exclusively via APIs (REST, SOAP, gRPC) or GraphQL. For a security professional or bug bounty hunter, discovering these interfaces is a critical phase of Attack Surface Mapping (ASM), as APIs often expose undocumented administrative functions, deprecated endpoints, or verbose error messages that lead to direct exploitation (e.g., BOLA/IDOR, Mass Assignment, or Information Disclosure).

**The Standard Discovery Workflow:**
1. **Passive Discovery:** Extracting API routes and GraphQL endpoints from static sources. This involves parsing JavaScript files (using LinkFinder/JSFinder equivalents), inspecting mobile application binaries (APK/IPA decompilation), and searching Wayback Machine/OSINT data for historical endpoints (e.g., `/api/v1/`).
2. **Active Fingerprinting:** Probing known web servers for common API paths (e.g., `/api/`, `/graphql`, `/v1/graphql`, `/swagger/v1/swagger.json`) to identify the API schema or framework.
3. **GraphQL Introspection & Fuzzing:**
   * Checking if Introspection queries are enabled. If so, extracting the entire schema (Types, Queries, Mutations).
   * If Introspection is disabled, attempting field/mutation guessing (fuzzing) based on common naming conventions or partial schema leaks.
4. **Route Bruteforcing:** Using highly specialized, context-aware wordlists to discover hidden REST routes (e.g., `/api/v2/admin/users/export`). 
5. **Query & Payload Generation:** Automatically converting discovered schemas (Swagger/OpenAPI or GraphQL Introspection data) into valid, testable requests for vulnerability scanners to consume.

---

### 2. Current Tools & Ecosystem

The current ecosystem is fragmented between general-purpose web fuzzers and highly specialized API/GraphQL utilities:

* **kiterunner:** The gold standard for REST API route bruteforcing. Instead of standard directory fuzzing, it uses massive datasets of real-world API routes and swagger specifications, allowing it to guess exact path structures and HTTP methods simultaneously.
* **InQL & gqlspection:** Essential tools for GraphQL. They excel at taking an introspection query response and generating all possible queries, mutations, and subscriptions. InQL operates well within Burp Suite, while gqlspection is more CLI-friendly.
* **graphql-cop:** A specialized utility that scans specifically for common GraphQL misconfigurations (e.g., Introspection enabled, GraphiQL interface exposed, lack of query depth limiting, CSRF vulnerabilities).
* **graphql-voyager:** Purely a visualization tool. It takes an introspection schema and maps it into an interactive node graph, helping researchers visually understand the relationships between different data types.

---

### 3. Where Current Tools Lack

While effective individually, chaining these tools into a highly scalable, automated ASM pipeline reveals significant bottlenecks:

* **Non-Standardized Output & Pipelining Friction:** `kiterunner` outputs custom text formats, `InQL` generates local files, and `graphql-cop` outputs console text. Connecting a tool that discovers a GraphQL endpoint to a tool that parses its schema requires writing fragile middleware scripts.
* **Lack of State & Diffing:** Bug bounty is continuous. Existing tools do not natively track state. If an API adds a new `/v3/` endpoint or a new mutation to a GraphQL schema, running the tools again dumps the entire state rather than alerting on the *delta* (what changed since last week).
* **Memory & Performance Bottlenecks:** Tools written in Python (like `graphql-cop` or older scanners) struggle with asynchronous concurrency at the scale of thousands of subdomains. Parsing massive 50MB Swagger JSON files can also cause memory spikes in unoptimized parsers.
* **Context-Blind Fuzzing:** Standard fuzzing doesn't dynamically adapt. If a tool discovers `/api/v1/users/`, it should automatically prioritize testing `/api/v1/users/{id}` or `/api/v2/users/`, but most fuzzers strictly adhere to their static wordlist.

---

### 4. Proposed Tool Workflow & Architecture

To resolve these ecosystem limitations, the new API/GraphQL Discovery module should be designed as an asynchronous, state-aware engine (ideally in Go or Rust) that ingests raw endpoints and outputs enriched, weaponized request templates.

#### **High-Level Architecture**

1. **Input Ingestion & Deduplication Queue:**
   * Accepts targets via `stdin` (JSONL) from previous pipeline phases (e.g., active subdomains, HTTP probes, JS link extractors).
   * Normalizes paths to prevent redundant scanning (e.g., treating `api.target.com/v1` and `api.target.com/v1/` as the same seed).

2. **Heuristic Fingerprinter (The Router):**
   * Sends lightweight probes (OPTIONS, GET, POST with empty JSON) to detect the technology stack.
   * **Route Logic:** If it detects Swagger/OpenAPI, route to the *REST Engine*. If it detects GraphQL-specific error messages (e.g., `"Must provide query string"`), route to the *GraphQL Engine*.

3. **The REST Engine:**
   * Automatically attempts to fetch schema definitions (`swagger.json`, `openapi.yml`).
   * If schemas are missing, initiates smart route bruteforcing using an embedded Kiterunner-style engine (using HTTP method permutation and API-specific datasets).
   * Parses discovered parameters and paths.

4. **The GraphQL Engine:**
   * Attempts full introspection.
   * Runs misconfiguration checks (GraphiQL exposure, field suggestion leaks).
   * If introspection is disabled, utilizes a schema-guessing worker to brute-force common queries (e.g., `query { user { id } }`).

5. **Query Generator & State Tracker:**
   * Compiles discovered routes, queries, and mutations into valid HTTP request objects.
   * Compares the generated map against the local state database (e.g., SQLite or Redis) to identify newly deployed endpoints.

6. **Standardized Emitter (JSONL):**
   * Streams structured JSON output detailing the endpoint, HTTP method, required parameters, and data type (REST vs. GraphQL).

---

### 5. Enhancements & Plugin Integration

To ensure this module exceeds current open-source capabilities, the following enhancements should be integrated into the architecture:

* **Dynamic Schema Stitching:** When a schema is fragmented across microservices (e.g., Apollo Federation), the tool should automatically fetch and stitch the fragments into a single unified workspace.
* **Context-Aware Wordlist Generation:** Instead of using a static 50GB wordlist, the tool should dynamically generate wordlists based on the target. If the JS analysis phase found the keyword `payment_gateway`, the API fuzzer should heavily prioritize API routes containing `payment`, `gateway`, `stripe`, and `transaction`.
* **Automated Parameter Binding:** Integrate a mapping system that recognizes parameter names and injects valid mock data. For instance, if a discovered REST endpoint requires an `email` and `uuid`, the tool automatically binds a valid test email and format-compliant UUID to the generated query, rather than leaving it blank and triggering 400 Bad Request errors in downstream vulnerability scanners.
* **Native Nuclei Integration:** Output formats should be natively compatible with vulnerability scanners. The tool can automatically generate custom Nuclei YAML templates for newly discovered API routes on the fly, allowing the vulnerability scanning phase to test for authorization bypasses or SQLi immediately. 
* **Distributed Worker Nodes:** For massive scopes (e.g., scanning the entire ASNs of a Fortune 500 company), the engine should support a Pub/Sub model (like RabbitMQ or Redis Streams) where a central coordinator distributes the API fuzzing workload across multiple ephemeral cloud workers to bypass rate limiting and reduce scan times.
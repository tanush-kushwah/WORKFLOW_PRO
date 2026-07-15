# Recon & Attack Surface Mapping — Tool Reference with Summaries & Links

Each tool below is listed by category with a short description and a link — GitHub repo where one exists, otherwise the tool's official website. Where no reliable link is available, the Link column is left blank rather than dropping the tool.

---

## Subdomain Enumeration (Passive)

| Tool | Summary | Link |
|---|---|---|
| **subfinder** | Aggregates subdomains from dozens of passive sources (crt.sh, VirusTotal, Shodan, etc.) in one fast run. | https://github.com/projectdiscovery/subfinder |
| **amass (passive mode)** | OWASP's asset discovery tool; pulls from certificate transparency, WHOIS, scraping, and API sources. | https://github.com/owasp-amass/amass |
| **assetfinder** | Lightweight, fast subdomain finder using crt.sh and Facebook's CT API. | https://github.com/tomnomnom/assetfinder |
| **crt.sh** | Web-based certificate transparency log search; every SSL cert issued for a domain shows up here. | https://crt.sh |
| **chaos** | Project Discovery's dataset of subdomains collected from bug bounty programs. | https://github.com/projectdiscovery/chaos-client |
| **github-subdomains** | Searches GitHub code/repos for hardcoded references to subdomains. | https://github.com/gwen001/github-subdomains |
| **findomain** | Fast, cross-platform subdomain finder similar to subfinder. | https://github.com/Findomain/Findomain |
| **sublist3r** | Older but still useful passive subdomain enumerator. | https://github.com/aboul3la/Sublist3r |
| **censys-subdomain-finder** | Queries Censys's internet-wide scan data for certificates/hosts tied to a domain. | https://search.censys.io |
| **SecurityTrails API** | Commercial DNS history database; historical DNS records and subdomains. | https://securitytrails.com |
| **dnsdumpster** | Free web tool that visually maps DNS records, subdomains, and hosting providers. | https://dnsdumpster.com |
| **gau (subdomain extraction)** | Primarily pulls archived URLs; subdomains extracted from the URL list as a byproduct. | https://github.com/lc/gau |
| **rapiddns** | Free DNS lookup/subdomain tool with historical A-record data. | https://rapiddns.io |

---

## Subdomain Enumeration (Active)

| Tool | Summary | Link |
|---|---|---|
| **puredns** | Fast DNS bruteforcer/resolver built on massdns; handles wildcard filtering well. | https://github.com/d3mondev/puredns |
| **alterx** | Generates smart subdomain permutations based on patterns found in known subdomains. | https://github.com/projectdiscovery/alterx |
| **amass (active mode)** | Same tool as passive amass but adds bruteforce, zone transfers, reverse DNS sweeps. | https://github.com/owasp-amass/amass |
| **massdns** | Extremely fast raw DNS resolver; the underlying engine many other tools are built on. | https://github.com/blechschmidt/massdns |
| **shuffledns** | Wrapper around massdns; simplifies bruteforce + resolution + wildcard filtering. | https://github.com/projectdiscovery/shuffledns |
| **dnsgen** | Generates permutations/combinations from a list of known subdomains. | https://github.com/AlephNullSK/dnsgen |
| **gotator** | Similar permutation generator to alterx/dnsgen, with customizable pattern files. | https://github.com/Josue87/gotator |
| **ripgen** | Rust-based reimplementation of dnsgen, optimized for large-scale permutation runs. | https://github.com/resyncgg/ripgen |

---

## DNS Resolution

| Tool | Summary | Link |
|---|---|---|
| **dnsx** | Fast multi-purpose DNS resolver/toolkit; resolves, checks record types, outputs JSON. | https://github.com/projectdiscovery/dnsx |
| **puredns** | (see above) also serves as a resolution tool with wildcard detection. | https://github.com/d3mondev/puredns |
| **massdns** | Raw high-speed resolver, typically used as a backend engine. | https://github.com/blechschmidt/massdns |
| **shuffledns** | (see above) resolution + bruteforce combined. | https://github.com/projectdiscovery/shuffledns |
| **zdns** | Fast DNS lookup tool from the ZMap project, good for bulk structured DNS queries. | https://github.com/zmap/zdns |

---

## Port Scanning

| Tool | Summary | Link |
|---|---|---|
| **naabu** | Fast SYN-based port scanner; typically used for the initial broad sweep. | https://github.com/projectdiscovery/naabu |
| **nmap** | The industry-standard scanner — service/version detection, NSE scripts, OS fingerprinting. | https://github.com/nmap/nmap |
| **masscan** | Extremely fast internet-scale port scanner for huge IP ranges. | https://github.com/robertdavidgraham/masscan |
| **rustscan** | Fast port scanner that pipes results into nmap automatically. | https://github.com/RustScan/RustScan |
| **unicornscan** | Asynchronous scanner for unusual scan types (UDP, stateless TCP). | https://github.com/dneufeld/unicornscan |

---

## HTTP Probing

| Tool | Summary | Link |
|---|---|---|
| **httpx** | Core HTTP toolkit — liveness, status codes, titles, tech stack, redirects, CDN detection. | https://github.com/projectdiscovery/httpx |
| **httprobe** | Simple tool that just checks which hosts respond on HTTP/HTTPS. | https://github.com/tomnomnom/httprobe |
| **meg** | Fetches many paths across many hosts efficiently. | https://github.com/tomnomnom/meg |

---

## Tech Fingerprinting

| Tool | Summary | Link |
|---|---|---|
| **httpx** | Built-in `-tech-detect` flag identifies frameworks/CMS/servers as part of the probing pass. | https://github.com/projectdiscovery/httpx |
| **whatweb** | Dedicated fingerprinting tool with a large plugin/signature database. | https://github.com/urbanadventurer/WhatWeb |
| **wappalyzer** | Browser extension/CLI identifying CMS, JS frameworks, analytics tools, and more. | https://github.com/wappalyzer/wappalyzer |
| **webanalyze** | Go-based reimplementation of Wappalyzer's detection logic, usable at scale. | https://github.com/rverton/webanalyze |
| **retire.js** | Scans for known-vulnerable JavaScript library versions loaded on a page. | https://github.com/RetireJS/retire.js |

---

## Screenshots / Visual Recon

| Tool | Summary | Link |
|---|---|---|
| **gowitness** | Fast headless-Chrome screenshotting tool; JSON/SQLite metadata storage. | https://github.com/sensepost/gowitness |
| **aquatone** | Visual recon tool that groups screenshots by similarity. | https://github.com/michenriksen/aquatone |
| **eyewitness** | Screenshotting tool that also attempts default credential checks. | https://github.com/FortyNorthSecurity/EyeWitness |
| **webscreenshot** | Lightweight Python-based screenshot tool for smaller/simpler runs. | https://github.com/maaaaz/webscreenshot |

---

## Content & Directory Discovery

| Tool | Summary | Link |
|---|---|---|
| **ffuf** | Fast, flexible fuzzer for directories, files, parameters, and vhosts. | https://github.com/ffuf/ffuf |
| **feroxbuster** | Rust-based recursive content discovery tool. | https://github.com/epi052/feroxbuster |
| **dirsearch** | Python-based directory/file bruteforcer with good wordlist/extension handling. | https://github.com/maurosoria/dirsearch |
| **gobuster** | Fast Go-based bruteforcer supporting dir, DNS, and vhost modes. | https://github.com/OJ/gobuster |
| **dirb** | Older, simpler directory bruteforce tool; no actively-maintained canonical repo (originally hosted on SourceForge, bundled in Kali). | |
| **wfuzz** | Flexible fuzzing tool for URLs, parameters, and headers. | https://github.com/xmendez/wfuzz |

---

## Parameter Discovery

| Tool | Summary | Link |
|---|---|---|
| **arjun** | Bruteforces hidden GET/POST/JSON parameters on a specific endpoint. | https://github.com/s0md3v/Arjun |
| **paramspider** | Mines historical/archived URLs for parameters already used by the app. | https://github.com/devanshbatham/ParamSpider |
| **gf** | Pattern-matching tool that filters URLs by vuln-category regex patterns. | https://github.com/tomnomnom/gf |
| **x8** | Fast hidden parameter discovery tool with response-diff detection. | https://github.com/Sh1Yo/x8 |
| **param-miner** | Burp Suite extension that discovers hidden parameters and headers. | https://github.com/PortSwigger/param-miner |

---

## JavaScript Analysis

| Tool | Summary | Link |
|---|---|---|
| **subjs** | Extracts JavaScript file URLs from a list of live hosts. | https://github.com/lc/subjs |
| **LinkFinder** | Parses JS files for endpoint paths embedded in the code. | https://github.com/GerbenJavado/LinkFinder |
| **SecretFinder** | Scans JS for hardcoded secrets: API keys, tokens, AWS credentials, Firebase configs. | https://github.com/m4ll0k/SecretFinder |
| **JSFinder** | Combines subdomain and URL extraction directly from JS files. | https://github.com/Threezh1/JSFinder |
| **xnLinkFinder** | Actively maintained LinkFinder alternative with better false-positive filtering. | https://github.com/xnl-h4ck3r/xnLinkFinder |
| **getJS** | Simple tool to download all JS files from a target for offline analysis. | https://github.com/003random/getJS |
| **JSLeak** | Finds secrets, paths, and links in JS/source code, inspired by LinkFinder. | https://github.com/byt3hx/jsleak |
| **mantra** | Hunts down API key leaks in JS files and pages. | https://github.com/brosck/mantra |

---

## Historical URL Discovery

| Tool | Summary | Link |
|---|---|---|
| **waybackurls** | Pulls all URLs the Wayback Machine has archived for a domain. | https://github.com/tomnomnom/waybackurls |
| **gau** | Aggregates URLs from Wayback Machine, Common Crawl, and OTX. | https://github.com/lc/gau |
| **katana (passive mode)** | Project Discovery's crawler can also pull historical/passive URL data. | https://github.com/projectdiscovery/katana |
| **gauplus** | Enhanced fork of gau with proxy/worker support (archived — lc/gau is the actively maintained successor). | https://github.com/bp0lr/gauplus |
| **waymore** | Thorough Wayback Machine harvester with deeper pagination/filtering. | https://github.com/xnl-h4ck3r/waymore |

---

## Vulnerability Scanning

| Tool | Summary | Link |
|---|---|---|
| **nuclei** | Template-based vulnerability scanner covering CVEs, misconfigs, exposures, takeovers. | https://github.com/projectdiscovery/nuclei |
| **jaeles** | Flexible scanning engine with a custom signature format. | https://github.com/jaeles-project/jaeles |
| **nikto** | Classic web server scanner for outdated software and dangerous files. | https://github.com/sullo/nikto |

---

## Subdomain Takeover Detection

| Tool | Summary | Link |
|---|---|---|
| **subzy** | Checks subdomains against known takeover fingerprints. | https://github.com/PentestPad/subzy |
| **tko-subs** | Takeover-detection tool with a broad fingerprint database. | https://github.com/anshumanbh/tko-subs |
| **nuclei (takeover templates)** | Nuclei's dedicated takeover template pack. | https://github.com/projectdiscovery/nuclei-templates |
| **can-i-take-over-xyz** | Community-maintained reference list of takeover-vulnerable services. | https://github.com/EdOverflow/can-i-take-over-xyz |

---

## Cloud Recon

| Tool | Summary | Link |
|---|---|---|
| **cloud_enum** | Multi-cloud (AWS/Azure/GCP) enumeration via keyword bruteforce. | https://github.com/initstring/cloud_enum |
| **S3Scanner** | Scans hostnames/keywords for open or misconfigured AWS S3 buckets. | https://github.com/sa7mon/S3Scanner |
| **cloudbrute** | Bruteforces cloud storage/infra across multiple providers. | https://github.com/0xsha/CloudBrute |
| **GCPBucketBrute** | Discovers and checks permissions of Google Cloud Storage buckets. | https://github.com/RhinoSecurityLabs/GCPBucketBrute |
| **lazys3** | Simple, fast Ruby-based S3 bucket bruteforcer. | https://github.com/nahamsec/lazys3 |
| **slurp** | S3 bucket discovery tool, also checks CT logs for bucket name hints. Note: the original `bbb31/slurp` GitHub account was deleted and later re-registered by an unrelated party — use this maintained fork instead of any old blog links pointing to `bbb31/slurp`. | https://github.com/0xbharath/slurp |

---

## CMS Scanners

| Tool | Summary | Link |
|---|---|---|
| **wpscan** | The standard WordPress scanner — plugins, themes, users, known vulnerabilities. | https://github.com/wpscanteam/wpscan |
| **droopescan** | Scanner focused on Drupal (and some Silverstripe support). | https://github.com/droope/droopescan |
| **joomscan** | Dedicated Joomla vulnerability and component scanner (OWASP project). | https://github.com/OWASP/joomscan |
| **CMSeeK** | Multi-CMS detection and scanning tool supporting a wide range of platforms. | https://github.com/Tuhinshubhra/CMSeeK |
| **cmsmap** | Multi-CMS scanner focused on WordPress, Joomla, and Drupal. | https://github.com/Dionach/CMSmap |

---

## API / GraphQL Discovery

| Tool | Summary | Link |
|---|---|---|
| **kiterunner** | Bruteforces API routes using large route-definition wordlists. | https://github.com/assetnote/kiterunner |
| **graphql-cop** | Checks GraphQL endpoints for common misconfigurations. | https://github.com/dolevf/graphql-cop |
| **graphql-voyager** | Visualizes a GraphQL schema as an interactive graph. | https://github.com/graphql-kit/graphql-voyager |
| **InQL** | Burp Suite extension for GraphQL introspection, query generation, and testing. | https://github.com/doyensec/inql |
| **gqlspection** | Parses a GraphQL introspection schema and generates possible queries/mutations from it. | https://github.com/doyensec/GQLSpection |

---

## OSINT

| Tool | Summary | Link |
|---|---|---|
| **theHarvester** | Aggregates emails, subdomains, and employee names from public sources. | https://github.com/laramies/theHarvester |
| **holehe** | Checks whether an email address is registered on dozens of platforms. | https://github.com/megadose/holehe |
| **h8mail** | Searches known breach databases for an email address. | https://github.com/khast3x/h8mail |
| **Photon** | OSINT-focused crawler that extracts emails, URLs, files, and secrets. | https://github.com/s0md3v/Photon |
| **Sherlock** | Searches for a given username across hundreds of social platforms. | https://github.com/sherlock-project/sherlock |
| **Maltego** | Commercial graph-based OSINT platform for linking domains, people, infrastructure. | https://www.maltego.com |
| **recon-ng** | Modular OSINT framework with many built-in reconnaissance modules. | https://github.com/lanmaster53/recon-ng |
| **spiderfoot** | Automated OSINT tool that runs dozens of modules against a target. | https://github.com/smicallef/spiderfoot |

---

## Crawling / Spidering

| Tool | Summary | Link |
|---|---|---|
| **katana** | Project Discovery's crawler; passive and active modes, JS-aware crawling, form extraction. | https://github.com/projectdiscovery/katana |
| **hakrawler** | Fast, simple web crawler for pulling links, JS files, and forms. | https://github.com/hakluke/hakrawler |
| **gospider** | Fast crawler with sitemap/robots.txt parsing and concurrent crawling. | https://github.com/jaeles-project/gospider |
| **Photon** | (see OSINT above) also functions as a general-purpose crawler. | https://github.com/s0md3v/Photon |
| **crawley** | Lightweight crawler focused on simplicity and speed for basic link extraction. | https://github.com/s0rg/crawley |

---

## WAF / CDN Detection

| Tool | Summary | Link |
|---|---|---|
| **wafw00f** | Identifies which Web Application Firewall (if any) is protecting a target. | https://github.com/EnableSecurity/wafw00f |
| **cdncheck** | Detects whether a host is behind a CDN. | https://github.com/projectdiscovery/cdncheck |

---

## Fuzzing (General Purpose)

| Tool | Summary | Link |
|---|---|---|
| **ffuf** | (see Content Discovery above) also widely used for general parameter/header fuzzing. | https://github.com/ffuf/ffuf |
| **wfuzz** | (see Content Discovery above) flexible general-purpose fuzzer. | https://github.com/xmendez/wfuzz |
| **x8** | (see Parameter Discovery above) fuzzing-based hidden parameter finder. | https://github.com/Sh1Yo/x8 |
| **radamsa** | General-purpose fuzz-input generator for malformed test payloads; moved off GitHub to GitLab. | https://gitlab.com/akihe/radamsa |

---

## Git / Secret Leak Scanning

| Tool | Summary | Link |
|---|---|---|
| **trufflehog** | Scans git history (and other sources) for secrets using entropy analysis and regex. | https://github.com/trufflesecurity/trufflehog |
| **gitleaks** | Fast, widely-used git secret scanner, often run in CI as well as offensive recon. | https://github.com/gitleaks/gitleaks |
| **gitrob** | Scans an organization's GitHub repos for potentially sensitive files/filenames. | https://github.com/michenriksen/gitrob |
| **git-secrets** | Prevents/detects secrets being committed to git; also usable to scan existing repos. | https://github.com/awslabs/git-secrets |

---

## All-in-One Automation Frameworks

| Tool | Summary | Link |
|---|---|---|
| **reconftw** | Full end-to-end recon automation script covering passive/active enumeration through vuln scanning. | https://github.com/six2dez/reconftw |
| **Osmedeus** | Workflow-based recon automation framework, highly configurable pipeline stages. | https://github.com/j3ssie/Osmedeus |
| **Sn1per** | All-in-one recon/scanning platform combining many tools into a single workflow. | https://github.com/1N3/Sn1per |
| **Legion** | GUI-based automated recon/scanning tool for visual management of large scans. | https://github.com/GoVanguard/legion |
| **Faraday** | Collaborative vulnerability management platform for centralizing findings. | https://github.com/infobyte/faraday |
| **AttackSurfaceMapper** | Automates recon combining OSINT and active techniques to expand a target's attack surface. | https://github.com/superhedgy/AttackSurfaceMapper |

---

## Reporting / Data Management

| Tool | Summary | Link |
|---|---|---|
| **Faraday** | (see above) also serves as a reporting/data aggregation platform across engagements. | https://github.com/infobyte/faraday |
| **DefectDojo** | Vulnerability management and reporting platform for tracking findings over time. | https://github.com/DefectDojo/django-DefectDojo |
| **Notion / Obsidian** | Not security tools themselves, but commonly used for manual note-taking and methodology docs. | https://www.notion.com / https://obsidian.md |

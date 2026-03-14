# private_webscanner
# 🎯 Recon Automation Script — Full Guide

> **Script**: `recon.sh`
> **Usage**: `./recon.sh <target_domain> [resolvers_path]`
> **Example**: `./recon.sh example.com resolvers.txt`

---

## Table of Contents

- [How It Works](#how-it-works)
- [Interactive Controls](#interactive-controls)
- [Core Functions](#core-functions)
- [Wordlist Configuration](#wordlist-configuration)
- [Step 1 — Passive Subdomain Enumeration](#step-1--passive-subdomain-enumeration)
- [Step 2 — Active Subdomain Enumeration](#step-2--active-subdomain-enumeration)
- [Step 3 — Subdomain Fuzzing](#step-3--subdomain-fuzzing)
- [Step 4 — Infrastructure Discovery](#step-4--infrastructure-discovery)
- [Step 5 — Certificate Transparency](#step-5--certificate-transparency-crtsh)
- [Step 6 — Merge & Deduplicate](#step-6--merge--deduplicate)
- [Step 7 — Alive Host Detection](#step-7--alive-host-detection)
- [Step 8 — URL Discovery](#step-8--url-discovery)
- [Step 9 — Merge URLs](#step-9--merge-urls)
- [Step 10 — Extract Interesting Data](#step-10--extract-interesting-data)
- [Step 11 — Filter Live URLs](#step-11--filter-live-urls)
- [Step 12 — Extract Parameters](#step-12--extract-parameters)
- [Step 13 — JavaScript Discovery](#step-13--javascript-discovery)
- [Step 14 — JS Secret Discovery](#step-14--js-secret-discovery)
- [Step 15 — PHP Parameter Discovery](#step-15--php-parameter-discovery)
- [Step 16 — Nuclei Vulnerability Scan](#step-16--nuclei-vulnerability-scan)
- [Step 17 — Hidden Parameters](#step-17--hidden-parameter-discovery)
- [Step 18 — Port Discovery](#step-18--port-discovery)
- [Step 19 — Wayback Sensitive Files](#step-19--wayback-sensitive-file-enumerator)
- [Step 20 — Advanced FFUF](#step-20--advanced-ffuf)
- [Step 21 — Kiterunner API Discovery](#step-21--kiterunner-api-discovery)
- [Output Directory Structure](#output-directory-structure)

---

## How It Works

```
┌─────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Check Tool │───▶│  Run Command │───▶│ Show Preview │───▶│ Wait Input   │
│  Installed? │    │  & Log Output│    │ (10 lines)   │    │ Enter/S/Q    │
└─────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
      │ No                                                        │
      ▼                                                           ▼
  [SKIP tool]                                            [Next tool or Step Summary]
```

1. **Check** if the tool exists on the system (`command -v`)
2. **Run** the tool, pipe output to a file and terminal
3. **Preview** results: line count, file size, first 10 lines
4. **Pause** for user review — Enter to continue, `s` to skip next, `q` to quit
5. **Step Summary** at the end of each numbered step — consolidated file list with line counts

---

## Interactive Controls

| Key | Action |
|-----|--------|
| `Enter` | Continue to next tool |
| `s` | Skip the next tool (one-time skip) |
| `q` | Quit the entire script |

---

## Core Functions

### `check_tool(tool_name)`
Checks if a binary is available in `$PATH`. If found, prints its path in green. If missing, prints a red error and returns non-zero so the calling step skips that tool.

```bash
check_tool "subfinder"
# [✓] Tool 'subfinder' is installed: /usr/bin/subfinder
# — OR —
# [✗] Tool 'subfinder' is NOT installed. Skipping...
```

### `run_and_log(description, output_file, command...)`
The main execution wrapper. It:
1. Checks if the user flagged a skip (`should_skip`)
2. Prints the command being run
3. Executes the command via `eval`
4. Reports line count on success or warns on non-zero exit
5. Calls `show_file_preview()` to display results
6. Calls `pause_and_review()` to wait for user input

### `show_file_preview(filepath, max_lines=10)`
Displays a boxed preview of an output file:
- File path, total line count, file size (human-readable)
- First 10 lines of content
- Note if more lines remain

### `pause_and_review()`
Displays a prompt box and reads user input:
- `Enter` → proceed
- `s`/`S` → sets `SKIP_NEXT=true` (next tool is skipped)
- `q`/`Q` → exits the script

### `should_skip()`
Returns 0 (true) if the user pressed `s` at the last prompt. The skip flag is one-time — it auto-resets after one tool is skipped.

### `step_summary(step_num, step_name, files...)`
Prints a consolidated summary box at the end of each numbered step:
- Lists each output file with its line count
- `✓` = has data, `○` = empty
- Pauses for user review before moving to the next step

### `pick_wordlist(candidates...)`
Takes multiple wordlist paths, returns the first one that exists. Used to auto-select the best available wordlist for each stage.

### `check_wordlist(name, path)`
Reports whether a specific wordlist file exists with a labeled status message.

---

## Wordlist Configuration

Each stage uses a **purpose-specific wordlist**. These are configured at the top of the script:

| Variable | Default Path | Used By |
|----------|-------------|---------|
| `WL_DNS_SUBS` | `seclists/Discovery/DNS/subdomains-top1million-5000.txt` | amass brute, dnscan, dnsx, puredns |
| `WL_DNS_FUZZ` | `seclists/Discovery/DNS/subdomains-top1million-20000.txt` | ffuf subdomain fuzzing |
| `WL_DIR_LARGE` | `seclists/Discovery/Web-Content/raft-large-directories.txt` | ffuf directory brute |
| `WL_DIR_MEDIUM` | `seclists/Discovery/Web-Content/raft-medium-directories.txt` | ffuf recursive fuzzing |
| `WL_DIR_FILES` | `seclists/Discovery/Web-Content/raft-large-files.txt` | ffuf file discovery |
| `WL_DIR_COMMON` | `seclists/Discovery/Web-Content/common.txt` | extension fuzzing, fallback |
| `WL_PARAMS` | `seclists/Discovery/Web-Content/burp-parameter-names.txt` | ffuf `?FUZZ=value`, hidden params |
| `WL_CONTENT` | `seclists/Discovery/Web-Content/common.txt` | general fallback |

> [!TIP]
> Edit these variables at the top of `recon.sh` if your seclists are installed in a different path.

---

## Step 1 — Passive Subdomain Enumeration

**Goal**: Discover subdomains without directly contacting the target. Uses public databases, APIs, and certificate logs.

### Tools Used

#### subfinder
**What**: Fast passive subdomain finder by ProjectDiscovery. Queries 40+ sources (crt.sh, Shodan, VirusTotal, etc.).
```bash
subfinder -d example.com -all -recursive -o Subs01.txt
```
- `-all` → use all sources
- `-recursive` → also enumerate subdomains of discovered subdomains
- **Output**: `{target}_subfinder.txt`

#### assetfinder
**What**: Finds domains and subdomains related to a domain using various data sources (Facebook CT, certspotter, etc.).
```bash
echo example.com | assetfinder -subs-only > Subs02.txt
```
- `-subs-only` → only output subdomains, not the root domain
- **Output**: `{target}_assetfinder.txt`

#### amass (passive)
**What**: OWASP Amass — deep DNS enumeration using passive data collection from APIs, web scraping, and data sources.
```bash
amass enum -d example.com -o Subs03.txt
```
- **Output**: `{target}_amass_passive.txt`

#### amass (brute)
**What**: Same tool but in brute-force mode — tries subdomain names from a wordlist.
```bash
amass enum -d example.com -brute -w wordlist.txt -o Subs03.txt
```
- `-brute` → enables brute-force mode
- `-w` → DNS wordlist (`WL_DNS_SUBS`)
- **Output**: `{target}_amass_brute.txt`

#### subenum.sh
**What**: Custom wrapper script that combines multiple subdomain tools (wayback, crt, AbuseIPDB, Findomain, Subfinder, Amass, Assetfinder).
```bash
./subenum.sh -d example.com -u wayback,crt,abuseipdb,Findomain,Subfinder,Amass,Assetfinder -o Subs04.txt
```
- Must be in the current directory
- **Output**: `{target}_subenum.txt`

#### findomain
**What**: Cross-platform subdomain finder with certificate transparency and APIs support.
```bash
findomain -t example.com -u Subs05.txt
```
- `-t` → target domain
- `-u` → output file (unique results)
- **Output**: `{target}_findomain.txt`

#### chaos
**What**: ProjectDiscovery's Chaos dataset — pre-indexed subdomain data from mass internet scanning.
```bash
chaos -d example.com -o Subs06.txt
```
- **Requires**: `PDCP_API_KEY` environment variable
- **Output**: `{target}_chaos.txt`

#### github-subdomains
**What**: Scrapes GitHub for subdomains leaked in source code, configs, and documentation.
```bash
github-subdomains -d example.com -t YOUR_GITHUB_TOKEN
```
- **Requires**: `GITHUB_TOKEN` environment variable
- **Output**: `{target}_github_subdomains.txt`

---

## Step 2 — Active Subdomain Enumeration

**Goal**: Actively brute-force subdomains by resolving DNS names from a wordlist against the target.

**Wordlist**: `WL_DNS_SUBS` (DNS subdomain names)

### Tools Used

#### dnscan
**What**: Python DNS scanner that brute-forces subdomains and checks for zone transfers.
```bash
python dnscan.py -d example.com -w wordlist.txt -t 300
```
- `-t 300` → 300 threads for speed
- Must have `dnscan.py` in current directory
- **Output**: `{target}_dnscan.txt`

#### dnsx
**What**: Fast DNS toolkit by ProjectDiscovery. Resolves subdomains from a wordlist.
```bash
dnsx -silent -d example.com -w wordlist.txt
```
- `-silent` → minimal output, just results
- **Output**: `{target}_dnsx.txt`

#### puredns
**What**: Fast domain resolver and brute-forcer with massdns integration. Uses public resolvers to validate.
```bash
puredns bruteforce wordlist.txt example.com -r resolvers.txt
```
- `-r` → resolver file (auto-downloaded if missing)
- **Output**: `{target}_puredns.txt`

---

## Step 3 — Subdomain Fuzzing

**Goal**: Discover subdomains by fuzzing name patterns (prefixes, suffixes, variations).

**Wordlist**: `WL_DNS_FUZZ` (larger DNS wordlist for fuzzing)

### Tool: ffuf
**What**: Fast web fuzzer. Here used to discover subdomains via URL patterns and virtual host headers.

#### URL Pattern Fuzzing
```bash
ffuf -u https://FUZZ.example.com -w wordlist.txt
ffuf -u https://FUZZ-example.com -w wordlist.txt
ffuf -u https://example-FUZZ.com -w wordlist.txt
```
Tries subdomain name variations: `dev.example.com`, `dev-example.com`, etc.

#### Virtual Host Fuzzing
```bash
ffuf -u https://example.com -w wordlist.txt -H "Host: FUZZ.example.com"
```
Sends requests to the IP but changes the `Host` header — catches virtual hosts not visible in DNS.

- **Output**: `{target}_ffuf_fuzz_1.txt`, `_fuzz_2.txt`, `_fuzz_3.txt`, `_ffuf_vhost.txt`

---

## Step 4 — Infrastructure Discovery

**Goal**: Map the target's infrastructure — reverse DNS, IP ranges, related hosts.

### Tools Used

#### shodanspider + dnsx
**What**: Shodanspider queries Shodan for the target's infrastructure. Results are piped to dnsx for PTR (reverse DNS) resolution.
```bash
shodanspider target.com | dnsx -silent -resp-only -ptr
```
- `-ptr` → perform PTR (reverse DNS) lookup
- **Output**: `{target}_shodan_infra.txt`

#### magicrecon.sh
**What**: Automated recon framework that runs multiple discovery tools in sequence.
```bash
./magicrecon.sh -l targets.txt --all
```
- Must be in current directory
- **Output**: `{target}_magicrecon.txt`

---

## Step 5 — Certificate Transparency (crt.sh)

**Goal**: Find subdomains from SSL/TLS certificate logs — public records of every certificate ever issued.

### Tool: curl + jq
**What**: Queries crt.sh's JSON API for all certificates matching `%.example.com`, extracts domain names.
```bash
curl -s "https://crt.sh/?q=%25.example.com&output=json" \
  | jq -r '.[].name_value' \
  | sed 's/\*\.//g' \
  | sort -u
```
- Removes wildcard prefixes (`*.`)
- Deduplicates results
- Falls back to `grep` parsing if `jq` is not installed
- **Output**: `{target}_crtsh.txt`

---

## Step 6 — Merge & Deduplicate

**Goal**: Combine ALL subdomain results from Steps 1–5 into one clean list.

### Process
1. `cat` all `{target}_*.txt` files from the output directory
2. `sort -u` to deduplicate
3. If `anew` is available, use it for smarter deduplication
4. Report total unique count

- **Output**: `{target}_allsubs.txt`

---

## Step 7 — Alive Host Detection

**Goal**: Probe every subdomain to check which ones are actually responding (alive). Categorize by HTTP status code.

### Tool: httpx
**What**: Fast HTTP prober by ProjectDiscovery. Checks if hosts respond and extracts metadata.

```bash
cat AllSubs.txt | httpx -status-code -content-length -web-server -title -follow-redirects
```

#### Categorized Outputs

| File | Filter | Purpose |
|------|--------|---------|
| `{target}_allstatus.txt` | All responses | Complete view |
| `{target}_200ok.txt` | `-match-code 200` | Live, accessible hosts |
| `{target}_403forbidden.txt` | `-match-code 403` | → Try 403 bypass techniques |
| `{target}_errors.txt` | `-filter-code 400` | Client errors |
| `{target}_notfound.txt` | `-filter-code 404` | → Try wayback + fuzz |
| `{target}_alive_subs.txt` | Silent, URLs only | Clean list for next steps |

> [!IMPORTANT]
> **403** hosts → attempt bypass with headers/path manipulation later (Step 20)
> **404** hosts → check wayback for historical content, fuzz for hidden paths

---

## Step 8 — URL Discovery

**Goal**: Discover as many URLs as possible from the alive hosts — historical, crawled, and scraped.

### Tools Used

#### waybackurls
**What**: Fetches all known URLs from the Wayback Machine (web.archive.org) for a domain.
```bash
cat AliveSubs.txt | waybackurls > WB1.txt
```
- **Output**: `{target}_waybackurls.txt`

#### waymore
**What**: Enhanced Wayback Machine URL fetcher with filters for year range and result limits.
```bash
waymore -i AliveSubs.txt -mode U -l 1000 -from 2021 -oU WM1.txt
```
- `-mode U` → URL-only mode
- `-l 1000` → limit to 1000 results
- `-from 2021` → only from 2021 onwards
- **Output**: `{target}_waymore.txt`

#### gau (GetAllUrls)
**What**: Fetches URLs from Wayback, Common Crawl, OTX, and URLScan.
```bash
cat AliveSubs.txt | gau --threads 200 > GAU1.txt
```
- **Output**: `{target}_gau.txt`

#### gauplus
**What**: Enhanced gau with random user-agent rotation for stealth.
```bash
gauplus -t 200 -random-agent < AliveSubs.txt > GAU2.txt
```
- **Output**: `{target}_gauplus.txt`

#### hakrawler
**What**: Web crawler built for asset discovery — finds URLs, subdomains, and JS files.
```bash
cat AliveSubs.txt | hakrawler -subs -u -insecure > HK1.txt
```
- `-subs` → include subdomains
- `-u` → show unique URLs only
- `-insecure` → skip TLS verification
- **Output**: `{target}_hakrawler.txt`

#### katana
**What**: Next-gen web crawler by ProjectDiscovery with headless browser support and JS parsing.
```bash
katana -u AliveSubs.txt -jc -kf all -d 5 -headless -fx -aff -fs rdn -f url -silent
```
- `-jc` → crawl JavaScript files
- `-kf all` → keep all found forms
- `-d 5` → depth 5
- `-headless` → use headless Chrome
- **Output**: `{target}_katana.txt`

#### gospider
**What**: Fast web spider that discovers URLs from sitemaps, robots.txt, and inline JavaScript.
```bash
gospider -S AliveSubs.txt -t 20 -d 3 --js --sitemap --robots
```
- `-t 20` → 20 threads
- `-d 3` → depth 3
- **Output**: `{target}_gospider.txt`

#### paramspider
**What**: Mines parameters from web archives for a domain. Focuses on finding parameterized URLs.
```bash
paramspider -d example.com -o PS1.txt
```
- **Output**: `{target}_paramspider.txt`

#### x8
**What**: Hidden parameter discovery tool. Brute-forces parameter names against URLs.
```bash
x8 -u "AliveSubs.txt" -o X81.txt
```
- **Output**: `{target}_x8.txt`

---

## Step 9 — Merge URLs

**Goal**: Combine all URL discovery results into one deduplicated file.

### Process
1. Concatenate all URL files from Step 8
2. `sort -u` or `anew` for deduplication
3. Report total count

- **Output**: `{target}_allurls.txt`

---

## Step 10 — Extract Interesting Data

**Goal**: Categorize all URLs by type/purpose for targeted analysis.

### Process
Uses `grep -iE` regex patterns to filter the master URL list into categories:

| Category | Pattern | File | Why It Matters |
|----------|---------|------|----------------|
| JS files | `\.js(\?|$)` | `{target}_js.txt` | Secrets, API keys, endpoints |
| PHP files | `\.php(\?|$)` | `{target}_php.txt` | SQL injection targets |
| ASP files | `\.asp(\?|$)` | `{target}_asp.txt` | Legacy app targets |
| JSP files | `\.jsp(\?|$)` | `{target}_jsp.txt` | Java app targets |
| ASPX files | `\.aspx(\?|$)` | `{target}_aspx.txt` | .NET targets |
| JSPX files | `\.jspx(\?|$)` | `{target}_jspx.txt` | Java XML targets |
| Injection points | `=` | `{target}_injection.txt` | Any URL with parameters |
| API URLs | `.json\|.xml\|.graphql` | `{target}_api_urls.txt` | API endpoints |
| Backend files | `.php\|.asp\|.jsp\|.cgi` | `{target}_backend_files.txt` | Server-side code |
| Login flows | `login\|signin\|auth` | `{target}_login_flows.txt` | Auth bypass targets |
| File uploads | `upload\|file\|download` | `{target}_file_uploads.txt` | Upload vulns |
| Admin panels | `admin\|dashboard` | `{target}_admin_panels.txt` | Admin access |
| Sensitive files | `.env\|.bak\|.config\|.sql` | `{target}_sensitive_files.txt` | Data leaks |
| IDOR targets | `[0-9]{2,}` | `{target}_idor_targets.txt` | ID manipulation |
| Interesting endpoints | `redirect\|callback\|debug` | `{target}_interesting_endpoints.txt` | SSRF/open redirect |
| Cloud leaks | `aws\|s3\|bucket\|token` | `{target}_cloud_leaks.txt` | Cloud misconfig |

#### urinteresting (if available)
Automated interesting URL classifier that uses predefined rules to find noteworthy URLs.

---

## Step 11 — Filter Live URLs

**Goal**: From the full URL list, keep only URLs that actually respond.

### Tool: httpx
```bash
cat AllURLs.txt | httpx -status-code -content-length -silent > LiveURLs.txt
```
- **Output**: `{target}_liveurls.txt`

---

## Step 12 — Extract Parameters

**Goal**: Extract all parameterized URLs for injection testing.

### Process

#### qsreplace (parameter templating)
```bash
cat AllURLs.txt | grep "=" | qsreplace "FUZZ" | sort -u
```
Replaces all parameter values with `FUZZ` for fuzzing. Example:
`/page?id=123&name=test` → `/page?id=FUZZ&name=FUZZ`
- **Output**: `{target}_paramurls.txt`

#### Raw param extraction
```bash
grep "=" AllURLs.txt | sort -u
```
- **Output**: `{target}_params.txt`

#### Param name deduplication
```bash
grep '=' AllURLs.txt | sed 's/=[^&]*/=/g' | sort -u
```
Strips values, keeps only parameter structures. Example:
`/page?id=123` → `/page?id=`
- **Output**: `{target}_param_names.txt`

---

## Step 13 — JavaScript Discovery

**Goal**: Collect all JavaScript file URLs — JS files are goldmines for secrets, API endpoints, and internal logic.

### Process
1. **grep from AllURLs**: Filter for `.js` extension (exclude `.json`)
2. **Wayback CDX API**: Query Wayback Machine's index directly for historical JS files
```bash
curl -s "https://web.archive.org/cdx/search/cdx?url=*.example.com/*&filter=original:.*.js$"
```

- **Output**: `{target}_js_urls.txt`, `{target}_js_all.txt`, `{target}_wayback_js.txt`

---

## Step 14 — JS Secret Discovery

**Goal**: Scan JavaScript files for hardcoded secrets (API keys, tokens, passwords, endpoints).

### Tools Used

#### subjs
**What**: Extracts JS sources from HTML pages by fetching pages and finding `<script>` tags.
```bash
cat AllURLs.txt | subjs
```
- **Output**: `{target}_subjs.txt`

#### mantra
**What**: Hunts for API keys, tokens, and secrets in JS files using predefined patterns.
```bash
cat js_urls.txt | mantra
```
- **Output**: `{target}_mantra.txt`

#### jsecret
**What**: Finds sensitive information in JavaScript files using regex-based secret patterns.
```bash
cat js_urls.txt | jsecret
```
- **Output**: `{target}_jsecret.txt`

#### jsleak
**What**: Finds URLs, secrets, and sensitive data leaked in JavaScript files.
```bash
cat js_urls.txt | xargs -P 20 -I {} jsleak -s -l -k -e {}
```
- `-P 20` → 20 parallel processes
- `-s` → scan for secrets
- `-l` → scan for links
- `-k` → scan for keywords
- **Output**: `{target}_jsleak.txt`

#### Regex-Based Secret Grep
Downloads first 50 JS files and greps for:
```
api[_-]?key|token|secret|password
```
- **Output**: `{target}_regex_secrets.txt`

---

## Step 15 — PHP Parameter Discovery

**Goal**: Discover hidden parameters in PHP endpoints and test for SQL injection.

### Tools Used

#### arjun
**What**: HTTP parameter discovery tool — brute-forces parameter names against PHP URLs.
```bash
arjun -i php_urls.txt >> php_parameters.txt
```
- **Output**: `{target}_php_params.txt`

#### sqlmap
**What**: Automatic SQL injection detection and exploitation tool.
```bash
sqlmap -u "https://example.com/file.php?id=*" --dbs --banner --batch --random-agent
```
- `--batch` → non-interactive mode (auto-answers prompts)
- `--random-agent` → randomize User-Agent header
- `--dbs` → enumerate databases
- Only runs on the first 5 PHP URLs for safety
- **Output**: `{target}_sqlmap.txt`

> [!CAUTION]
> sqlmap is an exploitation tool. Only use it on targets you have **explicit authorization** to test.

---

## Step 16 — Nuclei Vulnerability Scan

**Goal**: Automated vulnerability scanning using community-maintained templates.

### Tool: nuclei
**What**: Template-based vulnerability scanner by ProjectDiscovery. Checks for known exposures, misconfigs, and CVEs.
```bash
nuclei -l alive_subs.txt -t /nuclei-templates/http/exposures -o results.txt
```
- Falls back to `-tags exposure` if template path doesn't exist
- **Output**: `{target}_nuclei_exposures.txt`

---

## Step 17 — Hidden Parameter Discovery

**Goal**: Find hidden/undocumented parameters on the target that may lead to vulnerabilities.

### Tools Used

#### arjun
```bash
arjun -i AllURLs.txt -o arjun.json
```
Brute-forces GET/POST parameter names against all discovered URLs.
- **Output**: `{target}_arjun.json`

#### paraminer
**What**: Burp Suite's param miner equivalent — discovers hidden, unlinked parameters.
```bash
paraminer -u https://example.com
```
- **Output**: `{target}_paraminer.txt`

#### ffuf (parameter fuzzing)
**What**: Fuzzes parameter names using `?FUZZ=value` pattern.
```bash
ffuf -u https://example.com/search?FUZZ=value -w burp-parameter-names.txt
```
- **Wordlist**: `WL_PARAMS` (burp-parameter-names.txt)
- **Output**: `{target}_ffuf_params.txt`

---

## Step 18 — Port Discovery

**Goal**: Discover open ports across all alive hosts.

### Tool: naabu
**What**: Fast port scanner by ProjectDiscovery optimized for mass scanning.
```bash
naabu -list AliveSubs.txt -p - -rate 2000 -o ports.txt
```
- `-p -` → scan ALL ports (1-65535)
- `-rate 2000` → 2000 packets/sec
- **Output**: `{target}_ports.txt`

---

## Step 19 — Wayback Sensitive File Enumerator

**Goal**: Find historical sensitive files (backups, configs, keys, dumps) from Wayback Machine.

### Tool: waybackurls + grep
```bash
waybackurls https://www.target.com | grep -iE "\.(xls|sql|bak|env|key|pem|git|config)..."
```

#### File Categories Searched

| Category | Extensions |
|----------|-----------|
| Documents | `.xls`, `.xlsx`, `.csv`, `.pdf`, `.doc`, `.docx`, `.pptx` |
| Databases | `.sql`, `.db`, `.dump`, `.dmp` |
| Backups | `.bak`, `.backup`, `.old`, `.tar.gz`, `.zip`, `.7z`, `.rar` |
| Configs | `.ini`, `.conf`, `.config`, `.env`, `.json`, `.xml`, `.yml`, `.yaml` |
| Secrets | `.pem`, `.key`, `.crt`, `.ssh` |
| Source control | `.git`, `.gitignore`, `.gitconfig`, `.gitmodules` |
| DevOps | `.gitlab-ci.yml`, `.travis.yml`, `.dockerignore`, `docker-compose` |
| IDE | `.vscode`, `.idea`, `.DS_Store` |

- **Output**: `{target}_wayback_sensitive.txt`, `{target}_wayback_devops.txt`

---

## Step 20 — Advanced FFUF

**Goal**: Comprehensive directory and file brute-forcing with multiple strategies.

### Tool: ffuf (multiple passes)

#### 1. Directory Fuzzing
```bash
ffuf -u https://example.com/FUZZ -w raft-large-directories.txt -t 2000
```
- **Wordlist**: `WL_DIR_LARGE`
- **Output**: `{target}_dirs.txt`

#### 2. Extension Fuzzing
```bash
ffuf -u https://example.com/FUZZ -w common.txt -e .html,.php,.asp,.aspx,.js,.json,.xml,.config,.bak,.old,.backup,.zip
```
- **Wordlist**: `WL_CONTENT` (common.txt — smaller, since extensions multiply the request count)
- **Output**: `{target}_dirs_ext.txt`

#### 3. Bypass Headers
```bash
ffuf -u https://example.com/FUZZ -w wordlist.txt -H "X-Forwarded-For: 127.0.0.1" -H "X-Forwarded-Host: 127.0.0.1"
```
Bypasses IP-based rate limiting and access controls.
- **Output**: `{target}_bypass.txt`

#### 4. Recursive Fuzzing
```bash
ffuf -u https://example.com/FUZZ -w raft-medium-directories.txt -recursion -recursion-depth 2
```
Discovers nested directories (e.g., `/admin/config/backup/`).
- **Wordlist**: `WL_DIR_MEDIUM` (medium-sized to avoid explosion with recursion)
- **Output**: `{target}_recursive.txt`

#### 5. File Discovery
```bash
ffuf -u https://example.com/FUZZ -w raft-large-files.txt -t 2000
```
- **Wordlist**: `WL_DIR_FILES`
- **Output**: `{target}_files.txt`

---

## Step 21 — Kiterunner API Discovery

**Goal**: Discover API endpoints using kiterunner's routing-aware brute-forcing.

### Tool: kr (kiterunner)
**What**: Context-aware API endpoint scanner. Uses pre-built route datasets instead of dumb wordlists.

#### API Routes Scan
```bash
kr scan https://api.example.com -A=apiroutes-260227:10000 -x 8 -j 15 -v info
```
Discovers paths like `/api/v1/users`, `/api/v2/auth/login`
- **Output**: `{target}_kr_api.txt`

#### Parameters Scan
```bash
kr scan https://api.example.com -A=parameters-260227:5000 -x 5 -j 10 -v info
```
Discovers endpoints with parameter patterns like `/api/users?user_id=1`
- **Output**: `{target}_kr_params.txt`

#### Directories Scan
```bash
kr scan https://api.example.com -A=directories-260227:8000 -x 6 -j 12 -v info
```
Discovers directories like `/uploads`, `/static`, `/internal`
- **Output**: `{target}_kr_dirs.txt`

---

## Output Directory Structure

```
{target}_recon_{timestamp}/
├── {target}_allsubs.txt           ← All unique subdomains (Step 6)
├── {target}_alive_subs.txt        ← Alive hosts only (Step 7)
├── {target}_allurls.txt           ← All unique URLs (Step 9)
├── {target}_liveurls.txt          ← Live URLs only (Step 11)
├── {target}_ports.txt             ← Open ports (Step 18)
├── {target}_recon.log             ← Full execution log
│
├── alive/                         ← HTTP status categorized (Step 7)
│   ├── {target}_allstatus.txt
│   ├── {target}_200ok.txt
│   ├── {target}_403forbidden.txt
│   ├── {target}_errors.txt
│   └── {target}_notfound.txt
│
├── urls/                          ← Per-tool URL results (Step 8)
│   ├── {target}_waybackurls.txt
│   ├── {target}_gau.txt
│   ├── {target}_katana.txt
│   └── ...
│
├── extracted/                     ← Categorized URLs (Step 10)
│   ├── {target}_js.txt
│   ├── {target}_php.txt
│   ├── {target}_admin_panels.txt
│   ├── {target}_cloud_leaks.txt
│   └── ...
│
├── params/                        ← Parameter data (Step 12)
│   ├── {target}_paramurls.txt
│   ├── {target}_params.txt
│   └── {target}_param_names.txt
│
├── js/                            ← JavaScript files (Step 13)
│   ├── {target}_js_urls.txt
│   └── {target}_wayback_js.txt
│
├── js_secrets/                    ← JS secrets (Step 14)
│   ├── {target}_mantra.txt
│   ├── {target}_jsleak.txt
│   └── {target}_regex_secrets.txt
│
├── php/                           ← PHP analysis (Step 15)
│   ├── {target}_php_params.txt
│   └── {target}_sqlmap.txt
│
├── vulns/                         ← Vulnerability scan (Step 16)
│   └── {target}_nuclei_exposures.txt
│
├── hidden_params/                 ← Hidden params (Step 17)
│   ├── {target}_arjun.json
│   └── {target}_ffuf_params.txt
│
├── sensitive/                     ← Sensitive files (Step 19)
│   ├── {target}_wayback_sensitive.txt
│   └── {target}_wayback_devops.txt
│
├── ffuf/                          ← FFUF results (Step 20)
│   ├── {target}_dirs.txt
│   ├── {target}_dirs_ext.txt
│   ├── {target}_bypass.txt
│   ├── {target}_recursive.txt
│   └── {target}_files.txt
│
└── kiterunner/                    ← API discovery (Step 21)
    ├── {target}_kr_api.txt
    ├── {target}_kr_params.txt
    └── {target}_kr_dirs.txt
```

---

> [!WARNING]
> **Legal Notice**: Only use this script on targets you have **explicit written authorization** to test. Unauthorized scanning is illegal in most jurisdictions.

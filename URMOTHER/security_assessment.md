# Security Assessment: ScrapeGraphAI

**Date:** 2026-06-20
**Assessed by:** Kilo (automated)
**Scope:** `/home/urmother/projects/Scrapegraph-ai`
**Exclusions:** None
**Tech stack:** Python 3.12+, LangChain, Playwright, Selenium, Plasmate, BeautifulSoup, Pydantic, LangChain-Ollama/OpenAI/Mistral/etc.
**Deployment model:** Python library (pip-installable package), also ship as Docker image. Consumers use it as a scraping pipeline builder; the library itself makes outbound HTTP calls on the user's behalf.

---

## Executive Summary

The highest-severity finding is a confirmed arbitrary code-execution path: LLM-generated Python code (influenced by scraped web content) is executed with `exec()` and `__builtins__` present in `generate_code_node.py:456`. This allows an attacker who controls a target URL to run arbitrary Python on the host running ScrapeGraphAI.

Additionally, user-supplied URLs are passed directly to `requests.get()` in multiple loaders with no SSRF guard, enabling requests to cloud metadata endpoints or internal services. An API-token-for-Scrape.do is leaked in the URL query string. The library also ships an opt-out telemetry stream that sends user prompts, scraped website content, LLM responses, and JSON schemas to an external endpoint by default.

Overall risk rating: **HIGH** (combination of CRITICAL code execution, HIGH SSRF, and HIGH credential exposure).

---

## Trust Boundary Map

| Boundary | Present | Notes |
|----------|---------|-------|
| External network (fetch, webhooks, upload) | Yes | `requests.get` in `fetch_node.py`, `scrape_do.py`, `research_web.py`; `urllib.request` telemetry in `telemetry.py:138`; Playwright/Chromium browser fetches. |
| Multi-tenant (shared process or DB) | No | Library runs in-process for a single caller; no multi-tenant DB concept. |
| Agent / LLM surface (tool input, source edits) | Yes | User-supplied URL/prompt/HTML flows through multiple nodes into LLM calls; LLM output is fed to `exec()` in `generate_code_node.py`. |
| Local file system (read/write/exec from input) | Yes | `fetch_node.py:172-221` reads local files (`pdf`, `csv`, `json`, `xml`, `md`) based on user `source`. `generate_code_node.py:456` executes generated code from LLM. |
| Cross-origin / embed (HTML/markdown render) | Yes | Scraped HTML is parsed with `Html2TextTransformer`, BeautifulSoup, and `convert_to_md`. Output is not rendered in a browser, but is passed to LLMs as context. |

---

## Automated Scan Results

| Tool | Command | Findings |
|------|---------|----------|
| `grep` (manual secret scan) | `grep -E "sk-|AKIA|ghp_|github_pat_|secret_key|access_key|password|postgres://|mysql://|mongodb://|redis://"` excluding `.env.example` and `tests/fixtures/` | None observed — no hardcoded real secrets found. |
| Dependency audit | Not available (`pip`/`uv` not installed in assessment environment). `uv.lock` present and pinned. | Blocked — cannot run `pip audit` or `cargo audit`; lockfile is present and version-locked via uv. |
| Linter | `uv run ruff` — blocked (uv not installed). | Blocked — assessment environment lacks runtime dependencies. |
| SAST | None configured. | None available. |

---

## Findings

### Finding-1 — Arbitrary code execution via LLM-generated code in `exec()`

| Field | Value |
|-------|-------|
| **Severity** | CRITICAL |
| **Threat class** | TC2 — Untrusted Input -> Code Execution |
| **Location** | `scrapegraphai/nodes/generate_code_node.py:456` |
| **Status** | Confirmed |
| **Impact** | An attacker who influences the target webpage (or user prompt and/or a referenced schema) can craft HTML/content that causes the LLM to emit arbitrary Python. That code is then executed with `exec(function_code, sandbox_globals)` on the host machine. `__builtins__` is included in the sandbox globals, allowing full system access, file I/O, and network calls. The generated code is looped up to 10 times, so even if one iteration's output is sanitized, prior iterations already ran. |
| **Remediation** | (1) Remove `__builtins__` from `sandbox_globals` and replace `exec()` with a restricted AST-walk evaluator or a separate sandboxed subprocess with an OS-level seccomp/AppArmor profile. (2) Run generated code in a throwaway container/VM with no network/filesystem access. (3) If full sandboxing is not achievable, gate this feature behind an explicit opt-in and document the risk prominently. |

**Evidence:**

```python
# scrapegraphai/nodes/generate_code_node.py:446-466
sandbox_globals = {
    "BeautifulSoup": BeautifulSoup,
    "re": re,
    "__builtins__": __builtins__,
}

old_stdout = sys.stdout
sys.stdout = StringIO()

try:
    exec(function_code, sandbox_globals)

    extract_data = sandbox_globals.get("extract_data")

    if not extract_data:
        raise NameError(
            "Function 'extract_data' not found in the generated code."
        )

    result = extract_data(self.raw_html)
    return True, result
```

The `function_code` originates from the LLM, which is fed the user prompt, scraped HTML (`self.raw_html = state["original_html"][0].page_content`), and the user-defined schema. Remote HTML is therefore an attacker-controlled input reaching `exec()`.

---

### Finding-2 — SSRF via user-supplied URL passed directly to `requests.get`

| Field | Value |
|-------|-------|
| **Severity** | HIGH |
| **Threat class** | TC3 — SSRF & Outbound Fetch Abuse |
| **Location** | `scrapegraphai/nodes/fetch_node.py:287-289` |
| **Status** | Confirmed |
| **Impact** | When `use_soup=True`, the `source` URL (user-controlled) is passed directly to `requests.get(source)` with only an optional client-side timeout. No SSRF guard, no metadata-IP blocklist, and no redirect-following restriction (`redirect: "manual"` is not set). An attacker can make the library send HTTP requests to `169.254.169.254`, `127.0.0.1:PORT`, internal cloud metadata endpoints, or other internal services, leaking credentials or PII. |
| **Remediation** | (1) Validate `source` against an allowlist or blocklist for internal ranges (`169.254.169.254`, `127.0.0.0/8`, `10.0.0.0/8`, `172.16.0.0/12`, `192.168.0.0/16`, `[::1]`). (2) Use `redirect="manual"` and handle redirects explicitly. (3) Add an `ssrfSafeFetch` wrapper used by every outbound request path. |

**Evidence:**

```python
# scrapegraphai/nodes/fetch_node.py:283-289
if self.use_soup:
    if self.timeout is None:
        response = requests.get(source)
    else:
        response = requests.get(source, timeout=self.timeout)
```

`source` is the user-provided target URL. The same unguarded pattern recurs in `docloaders/scrape_do.py:42-48`.

---

### Finding-3 — API token transmitted in URL query string

| Field | Value |
|-------|-------|
| **Severity** | HIGH |
| **Threat class** | TC1 — Credential Leakage & Secret Handling |
| **Location** | `scrapegraphai/docloaders/scrape_do.py:47-48` |
| **Status** | Confirmed |
| **Impact** | The Scrape.do API token is embedded in the URL query parameter `?token={token}`. This token is then visible in HTTP access logs on the Scrape.do side, in any intermediate proxy logs, in browser/network inspection tools, and in `Referer` headers if the response triggers further requests. A leaked token enables unauthorized scraping through the victim's account, potentially incurring cost and attribution risk. |
| **Remediation** | (1) Prefer the Scrape.do proxy endpoint that authenticates with the token in the proxy URL (`user:token@host:port`) rather than in the query string. (2) If the API endpoint must be used, pass the token in an `Authorization` header or at minimum reduce its exposure via a short-lived token if the service supports it. |

**Evidence:**

```python
# scrapegraphai/docloaders/scrape_do.py:31-48
encoded_url = urllib.parse.quote(target_url)
if use_proxy:
    ...
else:
    api_scrape_do_url = os.getenv("API_SCRAPE_DO_URL", "api.scrape.do")
    url = f"http://{api_scrape_do_url}?token={token}&url={encoded_url}"
    response = requests.get(url)
```

---

### Finding-4 — TLS certificate verification disabled for Scrape.do proxy path

| Field | Value |
|-------|-------|
| **Severity** | MEDIUM |
| **Threat class** | TC3 — SSRF & Outbound Fetch Abuse |
| **Location** | `scrapegraphai/docloaders/scrape_do.py:11, 42-44` |
| **Status** | Confirmed |
| **Impact** | `urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)` suppresses TLS errors globally. The proxy-mode `requests.get(..., verify=False)` call then silently accepts any certificate, enabling man-in-the-middle attacks on proxy communications and on scraped target content. |
| **Remediation** | (1) Remove `urllib3.disable_warnings(...)`. (2) Default `verify=True` for all `requests.get` calls and only opt-out with explicit user consent when importing the module. (3) If MITM is truly desired (e.g., debugging), gate it behind an explicit flag. |

**Evidence:**

```python
# scrapegraphai/docloaders/scrape_do.py:11, 42-44
urllib3.disable_warnings(urllib3.exceptions.InsecureRequestWarning)
...
response = requests.get(
    target_url, proxies=proxies, verify=False, params=params
)
```

---

### Finding-5 — Opt-out telemetry exfiltrates prompts, scraped content, LLM responses, and schemas

| Field | Value |
|-------|-------|
| **Severity** | MEDIUM |
| **Threat class** | TC1 — Credential Leakage & Secret Handling |
| **Location** | `scrapegraphai/telemetry/telemetry.py:58-60, 115-142` |
| **Status** | Confirmed |
| **Impact** | Telemetry is enabled by default (`g_telemetry_enabled = _check_config_and_environ_for_telemetry_flag(True, config)`). The payload sent to `https://sgai-oss-tracing.onrender.com/v1/telemetry` includes `user_prompt`, `json_schema`, `website_content`, and `llm_response` (see `_build_telemetry_payload`). For consumers processing proprietary data, internal knowledge-base content, or personally identifiable information scraped from the web, this is a confidentiality breach. Opt-out requires knowledge of `SCRAPEGRAPHAI_TELEMETRY_ENABLED=false`. |
| **Remediation** | (1) Change default to opt-in. (2) Document the exact fields sent in the README and package docstrings. (3) Never send the full scraped `website_content` or `llm_response`; prefer hashed/anonymized counts or model identifiers only. |

**Evidence:**

```python
# scrapegraphai/telemetry/telemetry.py:59
g_telemetry_enabled = _check_config_and_environ_for_telemetry_flag(True, config)
...
# scrapegraphai/telemetry/telemetry.py:115-122
return {
    "user_prompt": prompt,
    "json_schema": json_schema,
    "website_content": content,
    "llm_response": llm_response,
    "llm_model": llm_model or "unknown",
    "url": url,
}
```

---

### Finding-6 — Dynamic module loading via user-configurable backend name

| Field | Value |
|-------|-------|
| **Severity** | LOW |
| **Threat class** | TC6 — Insecure Deserialization & Data Integrity |
| **Location** | `scrapegraphai/docloaders/chromium.py:62` → `scrapegraphai/utils/__init__.py` → `scrapegraphai/utils/sys_dynamic_import.py:63` |
| **Status** | Needs verification |
| **Impact** | `ChromiumLoader.__init__` accepts a `backend` argument (default `playwright`, user-overridable via config) and passes it to `dynamic_import(backend)`, which calls `importlib.import_module(modname)`. If an attacker can control the graph config (e.g., via a JSON/YAML config file or API body), they could force loading of an arbitrary module, leading to arbitrary code execution at import time (e.g., `nt:start` on Windows, module-level `os.system` calls). |
| **Remediation** | (1) Validate `backend` against an allowlist (`playwright`, `selenium`). (2) Reject unknown backends with a clear error rather than importing them. |

**Evidence:**

```python
# scrapegraphai/docloaders/chromium.py:62
dynamic_import(backend, message)
...
# scrapegraphai/utils/sys_dynamic_import.py:59-64
def dynamic_import(modname: str, message: str = "") -> None:
    if modname not in sys.modules:
        try:
            import importlib
            module = importlib.import_module(modname)
            sys.modules[modname] = module
```

---

### Finding-7 — Lower-bound-only dependency ranges risk silent breakage / supply-chain attack surface

| Field | Value |
|-------|-------|
| **Severity** | LOW |
| **Threat class** | TC8 — Dependency & Supply Chain |
| **Location** | `pyproject.toml:14-37` |
| **Status** | Likely |
| **Impact** | `pyproject.toml` declares `langchain>=1.2.0`, `langchain-classic>=1.0.0`, `playwright>=1.57.0`, `undetected-playwright>=0.3.0`, `beautifulsoup4>=4.14.3`, `duckduckgo-search>=8.1.1`, etc. without upper pinning. A compromised or buggy major/minor release of any of these (especially tool-calling frameworks and browser-automation packages) could silently introduce RCE, SSRF, or parsing vulnerabilities into downstream projects. |
| **Remediation** | (1) Pin to known-good versions (e.g., `langchain==1.2.0`, `playwright==1.57.0`, `beautifulsoup4==4.14.3`) or use caretless range constraints tied to the lockfile (e.g., `>=1.2.0,<2.0.0`). (2) Add a `pip-audit` / `uv audit` step to CI. (3) Review transitive `postinstall` / `install` scripts in the dependency tree. |

**Evidence:**

```toml
# pyproject.toml:14-37 (partial)
dependencies = [
    "langchain>=1.2.0",
    "langchain-classic>=1.0.0",
    "langchain-openai>=1.1.6",
    "langchain-mistralai>=1.1.1",
    "langchain_community>=0.4.0",
    ...
    "playwright>=1.57.0",
    "undetected-playwright>=0.3.0",
    "duckduckgo-search>=8.1.1",
    ...
]
```

---

### Finding-8 — Unbounded `ConditionalNode` condition evaluation from graph config

| Field | Value |
|-------|-------|
| **Severity** | LOW |
| **Threat class** | TC6 — Insecure Deserialization & Data Integrity |
| **Location** | `scrapegraphai/nodes/conditional_node.py:86-107` |
| **Status** | Needs verification |
| **Impact** | `ConditionalNode._evaluate_condition` evaluates an arbitrary expression string via `simple_eval` with a namespace that includes the entire graph state plus user-defined functions (`len`). An attacker who controls the graph configuration JSON/YAML can embed a condition that performs information disclosure (e.g., reading internal state keys) or denial-of-service (complex expression). |
| **Remediation** | (1) Restrict `ConditionalNode` to a whitelist of safe expressions or boolean keys. (2) If dynamic conditions are required, run them in a restricted `simpleeval` context (no attribute access, limited operators). |

**Evidence:**

```python
# scrapegraphai/nodes/conditional_node.py:86-107
def _evaluate_condition(self, state: dict, condition: str) -> bool:
    eval_globals = self.eval_instance.functions.copy()
    eval_globals.update(state)

    try:
        result = simple_eval(
            condition,
            names=eval_globals,
            functions=self.eval_instance.functions,
            operators=self.eval_instance.operators,
        )
        return bool(result)
```

---

## Remediation Roadmap

| Priority | Finding | Recommended Action | Effort |
|----------|---------|-------------------|--------|
| P0 | Finding-1 (CRITICAL) | Remove `__builtins__` from `generate_code_node.py:446` `sandbox_globals` and migrate `exec()` to a restricted subprocess/container sandbox. Gate the ScriptCreatorGraph behind an explicit opt-in with prominent documentation. | L |
| P0 | Finding-2 (HIGH) | Introduce `ssrfSafeFetch()` validating URLs against blocklists for `169.254.169.254`, `127.0.0.0/8`, `10/8`, `172.16/12`, `192.168/16`, `[::1]`. Apply in `FetchNode.handle_web_source`, `scrape_do_fetch`, and `research_web.py`. | M |
| P1 | Finding-3 (HIGH) | Move Scrape.do token from URL query string to proxy auth (preferred) or `X-API-KEY` header. | S |
| P1 | Finding-5 (MEDIUM) | Flip telemetry default to opt-in. Strip `website_content` and `llm_response` from payload; send only model name and anonymized event counts. | S |
| P1 | Finding-4 (MEDIUM) | Remove global `urllib3.disable_warnings`, default `verify=True`, gate `verify=False` behind explicit user flag. | S |
| P2 | Finding-6 (LOW) | Allowlist backends in `ChromiumLoader` before calling `dynamic_import()`. | S |
| P2 | Finding-7 (LOW) | Add upper caps to `pyproject.toml` runtime deps and wire `pip-audit`/`uv audit` into CI `.github/workflows/dependency-review.yml`. | M |
| P2 | Finding-8 (LOW) | Restrict `ConditionalNode._evaluate_condition` to key-based branching or a safe-expression subset. | M |

---

## Appendix — Scanned Files

- `scrapegraphai/nodes/generate_code_node.py` — Script creator / LLM-code execution node (TC2)
- `scrapegraphai/nodes/fetch_node.py` — URL fetch node, `requests.get` (TC3)
- `scrapegraphai/docloaders/scrape_do.py` — Scrape.do API integration (TC1, TC3)
- `scrapegraphai/docloaders/chromium.py` — Playwright/Selenium loader with dynamic import (TC6)
- `scrapegraphai/docloaders/plasmate.py` — Plasmate CLI subprocess loader (TC2 via command args)
- `scrapegraphai/nodes/conditional_node.py` — `simple_eval` condition evaluator from graph config (TC6)
- `scrapegraphai/telemetry/telemetry.py` — Opt-out telemetry to external endpoint (TC1)
- `scrapegraphai/utils/research_web.py` — Search engine integrations: DuckDuckGo, Bing, SearXNG, Serper (TC3)
- `scrapegraphai/utils/sys_dynamic_import.py` — `importlib.import_module` / `spec_from_file_location` helper (TC6)
- `scrapegraphai/graphs/abstract_graph.py` — LLM configuration and provider dispatch (TC5 surface)
- `pyproject.toml` — Dependency specification without upper pins (TC8)

---

## Appendix — Reviewed & Dismissed Patterns

- `json.load(f)` in `fetch_node.py:214` (loading user-specified JSON files) — **dismissed**. Local file read; schema validation happens downstream via Pydantic in `generate_code_node.py` and output parsers. No network ingress path to trigger this without explicit user file selection.
- `simple_eval` usage (rather than built-in `eval`) — **dismissed as standalone finding**: included above as Finding-8 because the context is a developer-authored graph configuration, but bundled here because `eval_globals` re-includes full graph state, which weakens the isolation usually relied on with `simpleeval`.
- `urllib3.disable_warnings()` in `scrape_do.py:11` — **dismissed as standalone finding**: included above as Finding-4.
- `.env.example` files — **dismissed**: all are empty templates with placeholder variable names, no real values.

---

## Appendix — Tools & Methodology

Scanned with: manual static review (grep + file read), trust-boundary mapping, and TC1-TC8 threat-class analysis guided by the security-assessment skill framework.

Dependency audit and linter runs were attempted but blocked because `pip`, `uv`, and `pip-audit` are not installed in this assessment environment; the project does ship a pinned `uv.lock` and `pyproject.toml`.

Method: Pre-Assessment Setup → Trust Boundary Map → Automated Signals (blocked; noted) → Targeted Manual Review (TC1-TC8) → Structured Report.

# TRINETRA

**Threat reports in, verified intelligence out.** Give TRINETRA a threat report (URL, PDF or text). It extracts the indicators, maps attacker behaviour to MITRE ATT&CK® v20 with the exact sentence that proves each mapping, and tells you what to fix first.

[![License: Apache 2.0](https://img.shields.io/badge/license-Apache%202.0-black)](LICENSE)
[![MITRE ATT&CK v20](https://img.shields.io/badge/MITRE%20ATT%26CK-v20-black)](https://attack.mitre.org/resources/updates/)
[![STIX 2.1](https://img.shields.io/badge/STIX-2.1-black)](https://oasis-open.github.io/cti-documentation/)
![Python 3.11](https://img.shields.io/badge/python-3.11-black)

![TRINETRA report overview](docs/assets/report-overview.png)

Threat reports hide indicators in prose (often defanged, like `hxxp://evil[.]com`) and describe attacker behaviour that analysts map to ATT&CK by hand. Plain LLMs make this worse: they invent technique IDs and quote sentences that aren't in the report. TRINETRA's rule is simple: **nothing reaches you unless it is proven.**

## Quick start

You need Docker with Compose, Node.js 20+, and an API key for any LLM provider that LiteLLM supports.

```bash
git clone https://github.com/rohith-0099/trinetra.git && cd trinetra
cp .env.example .env                                         # add GEMINI_API_KEY (or LLM_API_KEY)
docker compose up -d --build                                 # backend + Postgres (pgvector)
docker compose exec backend python scripts/load_attack.py    # first run only: load ATT&CK v20
cd frontend && printf 'NEXT_PUBLIC_API_URL=http://localhost:8000\nNEXT_PUBLIC_USE_MOCKS=false\n' > .env.local
npm install && npm run dev
```

Open http://localhost:3000 and paste a report URL. The API docs are at http://localhost:8000/docs.

## What it does

- **Every technique comes with proof.** Each mapping shows the exact sentence from the report; click it and it's highlighted in the source.
- **Code rejects hallucinations.** A technique ID that isn't in ATT&CK v20, or a quote that isn't in the report, is thrown out and counted as "hallucination blocked".
- **Indicators without the noise.** It extracts IPs, domains, URLs, hashes, CVEs and more (including defanged ones) and filters noise with rules that each cite a public source (IANA, RFCs, Public Suffix List).
- **It ends in action.** You get a plain-English briefing, CVE facts from NVD and CISA KEV, detection gaps with validated Sigma rules, and a downloadable block list.
- **It exports to standard formats:** STIX 2.1, ATT&CK Navigator, Attack Flow JSON, Sigma YAML and a TXT block list.

<details>
<summary><b>All features</b></summary>

| | |
|---|---|
| **Indicator engine** | Extracts IPs, domains, URLs, hashes, emails, CVEs, file paths and registry keys, including defanged ones. Filters noise with sourced rules: private/reserved and documentation IPs (IANA, RFC 1918, RFC 5737), public DNS resolvers, names without a real public suffix (Public Suffix List), URL paths that look like domains (RFC 3986), and harmless hashes such as MD5("123456"). Filtered items stay visible with their reason and source. An AI labels each kept indicator's role (C2, payload…) with a quote. |
| **Evidence-grounded ATT&CK v20 mapping** | An AI lists attacker behaviours with exact quotes. Retrieval finds candidate techniques in ATT&CK v20. A verifier compares each quote with the official description. Then code checks: the ID must exist in v20, the quote must be in the report, the quote must describe an action (not a heading, a summary phrase or an ATT&CK table row), and confidence must clear the threshold. Rejected candidates are shown with the reason. |
| **Plain-English briefing** | The attack in order, step by step, each with "See proof". Then "What to do now" with priorities and a source per action. Written only from verified data: code drops any step or action that names a technique or CVE the analysis didn't prove. |
| **CVE facts** | For each CVE: CVSS score and severity, description, affected versions and patch/advisory links from the NVD CVE API 2.0; whether it is in the CISA Known Exploited Vulnerabilities catalog, with its due date and required action. Cached; a failed lookup says "unavailable" and shows nothing rather than guessing. |
| **Live pipeline** | Every stage streams to a canvas as it runs, badged as deterministic, AI, or AI + rules. |
| **Attack flow** | Observed techniques as a chain by ATT&CK tactic order; gaps between stages are marked "not established in report". Suggested techniques to hunt for come from a **co-occurrence model over ATT&CK v20 groups, campaigns and software**they are dashed and labelled as not observed. |
| **Disruption simulator** | Block a step and see what share of attack paths break, plus the ATT&CK mitigations for that technique. |
| **Detection readiness** | ATT&CK v20 detection strategies and log sources per technique; tick the telemetry you have to see gaps. Sigma rules are validated with pySigma and converted to Splunk SPL. |
| **Memory across reports** | Indicators that appeared in earlier reports are flagged, with conflicting claims. Re-analysing the same document gives one "already analysed" notice instead of an alert per indicator. |
| **Prompt-injection firewall** | Reports are untrusted input. Text that tries to instruct the AI is flagged and mapped to ATT&CK v20 T1698 / T1699. |
| **v20 migrator** | Revoked or deprecated IDs are remapped to their v20 replacements (for example T1070.001 → T1685.005). |
| **Exports** | STIX 2.1 bundle · ATT&CK Navigator layer · Attack Flow JSON · Sigma YAML · printable report (browser print view) · block list (TXT). |

</details>

## How it works

```mermaid
flowchart LR
    A[URL / PDF / Text] --> B[Ingest]
    B --> G[Injection<br/>firewall]
    G --> C[IOC engine<br/><i>deterministic</i>]
    G --> D[Behaviour agent<br/><i>AI</i>]
    C --> C2[IOC role agent<br/><i>AI</i>]
    C --> V[CVE lookup<br/>NVD + CISA KEV]
    D --> E[Retriever<br/>ATT&CK v20 vectors]
    E --> F[Verifier<br/>AI + code checks]
    C2 --> L[Linker]
    F --> L
    L --> M[v20 migrator]
    M --> P[Co-occurrence<br/>suggestions]
    P --> H[Attack flow]
    M --> J[Detections<br/>+ Sigma]
    M --> K[Memory]
    H & J & K & V --> BR[Briefing<br/>AI + code checks]
    BR --> X[Exports]
```

The pipeline runs on LangGraph; every stage publishes Server-Sent Events, so the UI renders stage by stage.

The pipeline runs on LangGraph, and every stage streams Server-Sent Events so the UI renders stage by stage.

**Design principles**
1. **No unproven output reaches the user.** IDs must exist in ATT&CK v20; quotes must exist in the report and describe an action.
2. **Deterministic first.** Code does what code can; the AI handles language.
3. **Every rule cites a public source**or says plainly that it is a project rule.
4. **Observed ≠ suggested.** Suggestions are visually and structurally separate and never enter exports by default.
5. **Untrusted input stays data.** Report text is never treated as instructions.

## Results

The evaluation ran on 31 Oct 2026 over 5 reports, with none skipped.

| System | Precision | Recall | F1 | Parent F1 | Avg. time / report |
|---|:---:|:---:|:---:|:---:|:---:|
| Plain LLM baseline | 0.418 | 0.283 | 0.337 | 0.505 | 3.9 s |
| **TRINETRA** | **0.500** | 0.241 | 0.326 | 0.472 | 210.7 s |

| Report | Answer-key techniques | Baseline TP / FP / FN | TRINETRA TP / FP / FN |
|---|:---:|:---:|:---:|
| CVE-2024-4577 PHP-CGI test report | 35 | 18 / 7 / 17 | 15 / 6 / 20 |
| AA26-281A PRC-linked actors | 15 | 2 / 17 / 13 | 5 / 7 / 10 |
| AA26-204A Russian Zimbra phishing | 36 | 5 / 15 / 31 | 5 / 8 / 31 |
| AA26-222A Gunra ransomware | 36 | 8 / 9 / 28 | 5 / 5 / 31 |
| AA26-293A SafePay ransomware | 23 | 8 / 9 / 15 | 5 / 9 / 18 |

**What this shows:** TRINETRA claims fewer techniques and more of them are right (precision 0.500 vs 0.418), and every one it claims comes with a quote from the report. It also misses more (recall 0.241 vs 0.283), so its F1 is slightly below the plain LLM's. It is much slower because every behaviour goes through retrieval and a verifier call.

<details>
<summary><b>How it is measured</b></summary>

- **Answer keys:** the ATT&CK technique tables published in four CISA joint advisories (AA26-293A SafePay, AA26-281A PRC-linked actors, AA26-222A Gunra, AA26-204A Russian Zimbra phishing) and the appendix of our CVE-2024-4577 test report. IDs are copied, mapped to v20, and never edited. Files: [`backend/app/eval/ground_truth/`](backend/app/eval/ground_truth/).
- **No peeking:** ATT&CK tables, appendices and reference sections are cut from each report, and every remaining technique or tactic ID is deleted, before either system sees it.
- **Baseline:** one LLM call asked to list the techniques, with no retrieval and no checks. **TRINETRA:** the full pipeline, observed techniques only.
- **Models:** both slots run on Gemini `gemini/gemini-3.1-flash-lite` (extract: behaviours, Sigma rules, baseline; verify: verifier, IOC roles, guard, briefing). The saved result records both.
- **Partial runs are never scored:** a report whose pipeline had any failed LLM call is skipped and listed on `/eval` with the reason. In this run none were skipped.

```bash
docker compose exec backend python scripts/run_eval.py --dry-run   # ingest + leakage check, no LLM
docker compose exec backend python scripts/run_eval.py
```

</details>

## Limitations

- **Speed and volume depend on your LLM key.** Both slots run on Gemini (`gemini/gemini-3.1-flash-lite`). We first ran the verify slot on Groq's free on-demand tier (8,000 tokens/minute, 200,000 tokens/day); one report then took about 13 minutes (793 s) and the first eval attempt exhausted Groq's daily quota, so the verify slot moved to Gemini. Groq remains an optional setting.
- **No silent fallback between providers.** If one is down or out of quota, its calls fail with the reason in the stage event; the other provider is never used in its place.
- **Recall is low.** On the 5-report eval TRINETRA found 24% of the answer-key techniques (the plain LLM 28%), with higher precision (50% vs 42%) and a slightly lower F1. It misses techniques that a report only implies, and preparation techniques (reconnaissance, infrastructure) that advisories list but barely describe.
- **AI steps can miss techniques** that a report describes only vaguely. Made-up IDs and quotes are blocked; missed techniques are not.
- **Indicator roles are AI judgements** backed by a quote, not lookups in threat-intel feeds.
- **Suggested techniques** come from co-occurrence in ATT&CK data, not from MITRE CTID's Technique Inference Engine, and are hypotheses for hunting.
- **Some filters are project rules** with no external standard (common legitimate domains, file names, reference links). The app labels them so, and filtered items stay visible.
- **The Map tab** appears only with an offline GeoIP database installed; IPs it cannot place are not shown.
- **Not deployed yet**run it locally as above.

## Screenshots

| Home | How it works |
|:---:|:---:|
| <img src="docs/assets/home.png" width="420" alt="TRINETRA home"/> | <img src="docs/assets/how-it-works.png" width="420" alt="TRINETRA how it works page"/> |

These screenshots were captured from a local build. The report shown was analysed before the briefing and CVE stages were added, so its briefing card says so.

## Usage

**In the app:** paste a URL, upload a PDF or paste text. Then watch the pipeline, read the briefing, check the evidence and export.

**Via the API:**
```bash
curl -X POST http://localhost:8000/api/analyses -H "Content-Type: application/json" \
  -d '{"url": "https://www.cisa.gov/news-events/cybersecurity-advisories/aa26-293a"}'
curl -N http://localhost:8000/api/analyses/<id>/events          # live stage events
curl http://localhost:8000/api/analyses/<id>/export/stix -o report.stix.json
```

<details>
<summary><b>API reference</b></summary>

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/api/analyses` | Start an analysis (`url`, `text`, or PDF upload) |
| `GET` | `/api/analyses` | Recent analyses |
| `GET` | `/api/analyses/{id}` | Full result |
| `GET` | `/api/analyses/{id}/events` | Live stage events (SSE) |
| `POST` | `/api/analyses/{id}/feedback` | Analyst accept / reject |
| `POST` | `/api/analyses/{id}/simulate` | Disruption simulation |
| `POST` | `/api/analyses/{id}/readiness` | Detection gap analysis |
| `GET` | `/api/analyses/{id}/export/{fmt}` | `stix` · `navigator` · `sigma` · `flow` |
| `GET` | `/api/analyses/{id}/geo` | Map points (`available: false` without a GeoIP database) |
| `GET` | `/api/graph` | Cross-report memory graph |
| `GET` | `/api/sources` | Every rule, its source, and the glossary |
| `GET` | `/api/eval` | Latest accuracy run (404 until one has run) |

</details>

<details>
<summary><b>Environment variables</b></summary>

| Variable | Description |
|---|---|
| `GEMINI_API_KEY` / `GROQ_API_KEY` | Keys for models named `gemini/...` and `groq/...` |
| `LLM_API_KEY` | Key for any other provider (no key at all = offline mock mode) |
| `LLM_EXTRACT_MODEL` / `LLM_VERIFY_MODEL` | LiteLLM model names; both default to `gemini/gemini-3.1-flash-lite` (a `groq/...` model also works) |
| `LLM_TPM_GEMINI` / `LLM_TPM_GROQ` | Tokens-per-minute limit per provider; calls wait instead of failing. Empty = none |
| `LLM_RPM_GEMINI` / `LLM_RPM_GROQ` | Requests-per-minute limit per provider. Empty = none |
| `LLM_CONCURRENCY` | Parallel LLM calls (default 8) |
| `LLM_TOKENS_PER_MINUTE` | Tokens-per-minute limit for providers without their own setting. 0 = off |
| `LLM_CACHE` | `true` caches LLM answers in Postgres; set `false` only to measure real latency |
| `MAPPING_CONFIDENCE_THRESHOLD` | Minimum verifier confidence (default `0.6`) |
| `FIRECRAWL_API_KEY` | Optional; URL/PDF ingestion falls back to local parsing |
| `NVD_API_KEY` | Optional; raises NVD rate limits |
| `DATABASE_URL` | Postgres connection string |
| `GEOIP_DB_PATH` | Optional DB-IP City Lite `.mmdb`; without it the Map tab is hidden |
| `N8N_WEBHOOK_URL` | Optional outbound alerts |
| `CORS_ORIGINS` | Frontend origin(s), comma-separated |

</details>

<details>
<summary><b>Backend without Docker</b></summary>

```bash
docker compose up -d db
cd backend
python3.11 -m venv .venv && source .venv/bin/activate
pip install -r requirements.txt
export DATABASE_URL=postgresql://trinetra:trinetra@localhost:5433/trinetra
uvicorn app.main:app --reload --port 8000 --workers 1   # 1 worker: live events live in memory
```
</details>

## Tech stack

| Layer | Technology |
|---|---|
| **Frontend** | Next.js 15 · TypeScript · Tailwind · shadcn/ui · React Flow · sigma.js / graphology |
| **Backend** | Python 3.11 · FastAPI · LangGraph · Pydantic v2 · Server-Sent Events |
| **AI** | Any LLM via LiteLLM · fastembed (`BAAI/bge-small-en-v1.5`) for retrieval |
| **Security data** | MITRE ATT&CK v20 (STIX) · NVD CVE API 2.0 · CISA KEV · iocsearcher · tldextract (Public Suffix List) · stix2 · pySigma |
| **Data** | PostgreSQL + pgvector · NetworkX |
| **Integrations** | Firecrawl (URL/PDF ingestion) · n8n webhook (optional alerts) |

## Roadmap

- [x] Defang-aware indicator extraction with sourced filters and role labels
- [x] Evidence-grounded ATT&CK v20 mapping with hard checks
- [x] Plain-English briefing and CVE facts (NVD, CISA KEV)
- [x] Live pipeline view
- [x] Attack flow with co-occurrence suggestions, and disruption simulator
- [x] Detection readiness with pySigma-validated Sigma rules
- [x] Memory across reports with conflicting-claim alerts
- [x] STIX 2.1, Navigator, Attack Flow and Sigma exports
- [x] Accuracy scoreboard run (5 reports, 31 Oct 2026)
- [ ] Hosted deployment
- [ ] Autonomous watch of CISA, CERT-In and vendor blogs
- [ ] Investigation copilot with cited answers

## Sources and methodology

- **In the app:** the **How it works** page lists the pipeline, every rule with its reason and source link, the data sources, and the limitations. Every filtered indicator, rejected technique and migration links its own source.
- **In the repo:** [`docs/DECISIONS.md`](docs/DECISIONS.md) records each fix: the problem we saw, the change, the files, and the public source that supports it. Links we could not open are marked UNVERIFIED and are never shown in the app.
- **One registry:** [`backend/app/sources.py`](backend/app/sources.py), served at `/api/sources`.

## Responsible use

- Indicators are always shown defanged and are never links.
- Report text is untrusted; injection attempts are flagged, and the verifier blocks injected fake IDs and quotes.
- Suggested techniques are hypotheses, not observations. Shared infrastructure is never presented as proof of attribution.

## Credits

- [MITRE ATT&CK®](https://attack.mitre.org/): technique data (v20), descriptions, detection strategies and mitigations
- [NIST National Vulnerability Database](https://nvd.nist.gov/): CVE records via the CVE API 2.0
- [CISA Known Exploited Vulnerabilities Catalog](https://www.cisa.gov/known-exploited-vulnerabilities-catalog)
- [CISA, Best Practices for MITRE ATT&CK Mapping](https://www.cisa.gov/sites/default/files/2023-01/Best%20Practices%20for%20MITRE%20ATTCK%20Mapping.pdf): the basis for the evidence rules
- [MITRE CTID Attack Flow](https://ctid.mitre.org/): export format
- [OASIS STIX 2.1](https://oasis-open.github.io/cti-documentation/) · [SigmaHQ / pySigma](https://github.com/SigmaHQ) · [Public Suffix List](https://publicsuffix.org/) · [iocsearcher](https://pypi.org/project/iocsearcher/) · [Firecrawl](https://github.com/firecrawl/firecrawl) · [LangGraph](https://github.com/langchain-ai/langgraph)
- [DB-IP](https://db-ip.com/db/download/ip-to-city-lite) IP to City Lite (CC BY 4.0), only if you install it for the optional map

*MITRE ATT&CK and ATT&CK are registered trademarks of The MITRE Corporation. TRINETRA is not affiliated with or endorsed by MITRE, NIST or CISA.*

## License

Apache License 2.0. See [`LICENSE`](LICENSE).

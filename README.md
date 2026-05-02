# Multi-Domain Support Triage Agent

A terminal-based support triage agent that handles support tickets across three ecosystems:
**HackerRank**, **Claude (Anthropic)**, and **Visa India**.

---

## Setup

### 1. Install dependency
```bash
pip install groq
```

### 2. Get a free Groq API key
- Go to [console.groq.com](https://console.groq.com)
- Sign up → API Keys → Create API Key
- Copy the key (starts with `gsk_...`)

### 3. Set your API key

**Windows PowerShell:**
```powershell
$env:GROQ_API_KEY = "gsk_your_key_here"
```

**Mac / Linux:**
```bash
export GROQ_API_KEY="gsk_your_key_here"
```

---

## Usage

### Process CSV tickets (main mode)
```bash
python agent.py --input support_tickets.csv --output output.csv
```

Reads every ticket from `support_tickets.csv`, processes it, writes results to `output.csv`.
A full run log is saved to `log.txt` automatically.

### Interactive mode
```bash
python agent.py
```

| Command | What it does |
|---|---|
| `demo` | Runs 8 pre-loaded example tickets |
| `corpus` | Browse all 37 knowledge base articles |
| `quit` | Exit |

---

## Output Fields

| Field | Description |
|---|---|
| `Issue` | Original ticket text |
| `Subject` | Original subject line |
| `Company` | Original company field |
| `Response` | User-facing answer grounded in the support corpus |
| `Product Area` | Specific support category (e.g. `billing`, `fraud_security`) |
| `Status` | `replied` or `escalated` |
| `Request Type` | `product_issue`, `feature_request`, `bug`, or `invalid` |
| `justification` | Internal reasoning for the decision |

---

## Approach Overview

### Pipeline

Every ticket goes through a 6-stage pipeline:

```
Ticket (issue + subject + company)
    │
    ├─ [1] Domain Detection
    │       Keyword scoring across HackerRank / Claude / Visa vocabularies.
    │       Company field used first if present and valid.
    │
    ├─ [2] Safety Checks
    │       Prompt injection detector (English + French + Spanish patterns)
    │       Harmful request detector (delete files, exploit code, etc.)
    │
    ├─ [3] Escalation Engine
    │       3-tier rule engine:
    │       CRITICAL — stolen card, identity theft, security vulnerability,
    │                  platform-wide outage, unauthorized transactions
    │       HIGH     — billing disputes, account compromise, fraud, locked out
    │       MEDIUM   — plagiarism flags, test cancellations, disqualifications
    │
    ├─ [4] Corpus Retrieval
    │       Tag (x4) + title (x2) + content (x1) keyword scoring.
    │       Top 4 articles injected into LLM context.
    │       Stopwords filtered to improve signal quality.
    │
    ├─ [5] LLM Response Generation
    │       Groq llama-3.3-70b with a strict system prompt.
    │       Grounded only in retrieved corpus — no outside knowledge.
    │       Returns JSON: status, product_area, response, justification, request_type.
    │
    └─ [6] Output Validation
            Ensures status and request_type are within allowed values.
            Falls back to escalated/product_issue on parse failure.
```

---

### Knowledge Base

37 articles across three domains, written from the official support pages:

| Domain | Articles | Areas Covered |
|---|---|---|
| HackerRank | 16 | Account management, assessments, billing, proctoring, candidate management, certifications, technical support, infosec/compliance |
| Claude (Anthropic) | 10 | Billing, usage limits, privacy/data, content policy, API/developer, account access, enterprise/LTI, security/bug bounty |
| Visa India | 11 | Fraud & security, disputes, card usage, international travel, EMI, contactless, cash advance, traveller's cheques |

---

### Safety Design

| Threat | How it's handled |
|---|---|
| Prompt injection | Regex patterns detect attempts to reveal internal rules or bypass instructions |
| Harmful requests | Patterns detect requests for destructive code (delete all files, exploits) |
| Hallucination | LLM strictly instructed to answer only from the provided corpus |
| Misleading company field | Domain re-inferred from issue text when company is None or ambiguous |
| Non-English tickets | Language detected; LLM replies in the same language |
| Out-of-scope tickets | Replied with request_type=invalid explaining scope limitation |

---

### Escalation Logic

The agent escalates when any of the following are true:
- A rule-engine keyword matches (e.g. `stolen card`, `identity theft`, `all requests are failing`)
- The LLM determines escalation is needed based on corpus and system prompt rules
- The ticket contains a harmful or injection payload

---

### Rate Limit Handling

- **8-second delay** between tickets by default (safe for Groq free tier)
- **Exponential backoff** on 429 errors: 10s → 20s → 40s → 80s → 160s
- **5 retry attempts** before falling back to a safe escalation response

Override delay if needed:
```bash
python agent.py --input support_tickets.csv --delay 12
```

---

## Files

| File | Description |
|---|---|
| `agent.py` | Main agent — all logic in one file |
| `requirements.txt` | Python dependencies |
| `output.csv` | Generated after running on support_tickets.csv |
| `log.txt` | Full run transcript, auto-generated |

---

## Model

**Groq** · `llama-3.3-70b-versatile` · Temperature: 0.1 · Max tokens: 900

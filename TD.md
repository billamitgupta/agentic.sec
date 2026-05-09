# Hackathon Plan: SecTriage — AI Security Incident Response Agent

> Generative UI Global Hackathon: Agentic Interfaces — Pune, May 10 2026
> Organized by AI Tinkerers Pune × Google DeepMind × CopilotKit

---

## 1. The Idea

### Elevator Pitch

**SecTriage** is an agentic security incident response console. You paste a raw alert — a Splunk log line, a CVE ID, a suspicious IP, a Wazuh alert — and the agent thinks through it, calls real security APIs, and renders a living incident dashboard: severity card, enrichment table, recommended actions. When it needs to do something destructive (block an IP, revoke a credential, isolate a host), it **pauses and asks for human sign-off** before executing.

No chat bubbles. No "here is a summary of your alert." The interface is the agent's output.

### Why It's Different

Most hackathon demos are productivity tools: to-do managers, CRM dashboards, note-takers. SecTriage is:

- **Stakes are real** — approving or rejecting "Block this IP" is a decision with consequences. Human-in-the-loop isn't a gimmick, it's mandatory.
- **Compound tool use** — CVE enrichment + IP reputation + WHOIS + action execution is a multi-hop workflow, not a single API call.
- **Non-obvious UI** — the agent decides what components to render based on what it finds. A clean CVE renders a patch card. A malicious C2 IP renders an isolation workflow. The UI structure itself encodes the agent's reasoning.

### The Demo Scenario (Golden Path)

```
User pastes:
  "ALERT: Outbound connection to 185.220.101.47:443 from prod-api-03.
   Process: python3 (pid 9821). User: www-data. Time: 03:14 UTC."

Agent:
  1. Classifies: C2 beaconing candidate (Tor exit node)
  2. Calls AbuseIPDB → renders IP Reputation Card (score: 97/100, 847 reports)
  3. Calls Shodan stub → renders Open Ports Table (443, 9001 — Tor signature)
  4. Calls internal asset lookup → renders Affected Asset Card (prod-api-03, env: production)
  5. Determines severity: CRITICAL
  6. Generates action buttons: [Block IP at firewall] [Isolate host] [Create incident ticket]
  7. User clicks "Isolate host" → INTERRUPT fires → Approval Modal appears
  8. User approves → agent executes, streams STATE_DELTA back → card updates to "Isolated"
```

---

## 2. Track & Judging Alignment

**Chosen Track: Declarative Generative UI (A2UI + AG-UI)**

| Judging Criterion | How SecTriage Hits It |
|---|---|
| **Dynamic Component Generation** | Agent emits different A2UI component trees based on alert type — malware vs CVE vs C2 render entirely different layouts |
| **Agentic Feedback Loops** | User approves/rejects actions via rendered buttons; agent continues from that signal |
| **Latency-Optimized Rendering** | A2UI components stream via SSE as tool calls complete — each card appears individually, not all at once |
| **Tool-Enabled Interfaces** | Action buttons directly trigger agent tool calls (firewall API, ticketing stub, isolation command) |

---

## 3. Prerequisites

### Accounts & API Keys to Set Up Before the Hackathon

| Service | What For | Free Tier? | Link |
|---|---|---|---|
| **Google AI Studio** | Gemini API key (default LLM) | Yes | aistudio.google.com |
| **AbuseIPDB** | IP reputation enrichment | Yes (1000 req/day) | abuseipdb.com/api |
| **NVD / NIST** | CVE lookup by ID | Yes, no key needed | nvd.nist.gov/developers |
| **Shodan** | IP open ports + banner grab | Free tier limited | account.shodan.io |
| **CopilotKit Intelligence** | Persistent threads (Postgres) | Free dev tier | via `npx @copilotkit/cli init` |
| **GitHub** | Clone starter kit | — | github.com |

> Minimum viable set: Gemini key + AbuseIPDB key. Everything else can be stubbed on the day.

### Software to Install Locally

```bash
# Node / npm
node --version   # >= 20
npm --version    # >= 10

# Python
python3 --version  # >= 3.11
pip install uv     # fast Python package manager used by the starter kit

# Docker (required for CopilotKit Intelligence / Postgres thread storage)
docker --version

# GitHub CLI (optional but useful)
brew install gh
```

### Environment Variables

Create `.env` in the root and `apps/agent/.env` (the starter kit uses both):

```env
# Root .env
GEMINI_API_KEY=your_key_here

# apps/agent/.env
GEMINI_API_KEY=your_key_here
ABUSEIPDB_API_KEY=your_key_here
NVD_API_KEY=optional_speeds_up_rate_limits
SHODAN_API_KEY=your_key_here_or_leave_blank_to_stub
```

---

## 4. Tech Stack

```
Starter Kit:   jerelvelarde/Generative-UI-Global-Hackathon-Starter-Kit
LLM:           Gemini 3.1 Flash-Lite (default) — one-line swap to Claude if needed
Agent:         LangChain Deep Agents (LangGraph under the hood)
Frontend:      Next.js 15 + CopilotKit v2 + A2UI renderer
Transport:     AG-UI protocol over SSE (CopilotKit manages this)
Components:    A2UI declarative component spec (Google DeepMind)
Thread Store:  CopilotKit Intelligence (Postgres-backed persistent threads)
MCP Server:    apps/mcp — deployable, runs natively in Claude/ChatGPT
```

---

## 5. Repository Structure

Starting from the cloned starter kit, here is the final structure after our customizations. Lines marked `[NEW]` are files we write. Lines marked `[MODIFY]` are starter files we edit. Everything else is untouched infrastructure.

```
Generative-UI-Global-Hackathon-Starter-Kit/
│
├── apps/
│   │
│   ├── agent/                          # Python LangGraph Deep Agent
│   │   └── src/
│   │       ├── agent.py                [MODIFY] swap leads logic → security triage logic
│   │       ├── runtime.py              [MODIFY] update system prompt for security context
│   │       ├── tools/
│   │       │   ├── abuseipdb.py        [NEW] AbuseIPDB IP reputation tool
│   │       │   ├── nvd_cve.py          [NEW] NVD CVE lookup tool
│   │       │   ├── shodan_lookup.py    [NEW] Shodan open ports tool (stub-safe)
│   │       │   ├── asset_lookup.py     [NEW] mock internal asset registry
│   │       │   └── action_executor.py  [NEW] firewall block / host isolation stubs
│   │       ├── a2ui_schemas.py         [NEW] A2UI component JSON schemas for security
│   │       └── notion_mcp.py           [MODIFY] disable / replace with security MCP tools
│   │
│   ├── frontend/                       # Next.js 15 + CopilotKit
│   │   └── app/
│   │       ├── page.tsx                [MODIFY] update canvas layout + add alert input
│   │       ├── components/
│   │       │   ├── AlertInput.tsx      [NEW] paste-box for raw alert text
│   │       │   ├── IncidentCard.tsx    [NEW] A2UI "card" renderer for incident metadata
│   │       │   ├── IpReputationCard.tsx [NEW] A2UI "card" for AbuseIPDB result
│   │       │   ├── CveCard.tsx         [NEW] A2UI "card" for CVE detail
│   │       │   ├── FindingsTable.tsx   [NEW] A2UI "table" for enrichment rows
│   │       │   ├── ActionButtons.tsx   [NEW] A2UI "button-group" triggering agent tools
│   │       │   ├── ApprovalModal.tsx   [NEW] fires on INTERRUPT — shows action + confirms
│   │       │   └── SeverityBadge.tsx   [NEW] color-coded badge (LOW / MED / HIGH / CRITICAL)
│   │       └── lib/
│   │           └── a2ui-registry.ts    [NEW] maps A2UI type strings → React components
│   │
│   ├── bff/                            # Backend-for-frontend (untouched)
│   │
│   └── mcp/                            # Deployable MCP server
│       └── src/
│           └── index.ts                [MODIFY] expose security tools as MCP tools
│                                                (lets the agent run in Claude/ChatGPT too)
│
├── .env.example                        [MODIFY] add our required keys
├── .claude/skills/                     # Pre-installed — gives Claude Code CopilotKit context
└── dev-docs/
    └── demo-prompts.md                 [MODIFY] add our golden path alert scenarios
```

---

## 6. Protocol Architecture

### How A2UI, AG-UI, and MCP Work Together in SecTriage

```
┌─────────────────────────────────────────────────────────────┐
│                      FRONTEND (Next.js)                      │
│                                                              │
│  AlertInput ──► CopilotKitProvider ──► Canvas               │
│                        │                  │                  │
│                   AG-UI SSE stream    A2UI Renderer          │
│                        │                  │                  │
│              (RUN_STARTED, TEXT,      IncidentCard           │
│               TOOL_CALL_START,        IpReputationCard       │
│               STATE_DELTA,            FindingsTable          │
│               INTERRUPT) ◄────────── ActionButtons          │
│                                       ApprovalModal          │
└─────────────────────┬───────────────────────────────────────┘
                      │ SSE (AG-UI events)
┌─────────────────────▼───────────────────────────────────────┐
│                   BFF (Next.js API route)                    │
│              /api/copilotkit — bridges to agent              │
└─────────────────────┬───────────────────────────────────────┘
                      │ HTTP
┌─────────────────────▼───────────────────────────────────────┐
│              AGENT (Python, LangGraph Deep Agent)            │
│                                                              │
│  Input: raw alert text                                       │
│  Step 1: classify alert type (IP/CVE/log/hash)               │
│  Step 2: call relevant tools → build enrichment context      │
│  Step 3: emit A2UI JSON → STATE_SNAPSHOT to frontend         │
│  Step 4: determine severity                                   │
│  Step 5: if HIGH/CRITICAL + destructive action               │
│           → emit INTERRUPT event                             │
│  Step 6: wait for user signal (approve/reject)               │
│  Step 7: execute action → emit STATE_DELTA                   │
│                                                              │
│  Tools registered:                                           │
│   - lookup_ip_reputation(ip)      → AbuseIPDB               │
│   - lookup_cve(cve_id)            → NVD API                  │
│   - lookup_ports(ip)              → Shodan                   │
│   - lookup_asset(hostname)        → internal stub            │
│   - execute_action(type, target)  → firewall/isolation stub  │
└─────────────────────┬───────────────────────────────────────┘
                      │ MCP (optional surface)
┌─────────────────────▼───────────────────────────────────────┐
│              MCP SERVER (apps/mcp)                           │
│   Exposes the same tools so the agent runs inside            │
│   Claude or ChatGPT's native generative UI surface           │
└─────────────────────────────────────────────────────────────┘
```

### AG-UI Event Flow for a Critical Alert

```
Agent                                    Frontend
  │  RUN_STARTED                            │
  │ ───────────────────────────────────►   │  spinner starts
  │  TEXT_MESSAGE_CONTENT ("Analyzing…")   │
  │ ───────────────────────────────────►   │  chat message appears
  │  TOOL_CALL_START (lookup_ip_reputation) │
  │ ───────────────────────────────────►   │  "Checking IP reputation…" badge
  │  TOOL_CALL_RESULT                      │
  │ ───────────────────────────────────►   │
  │  STATE_SNAPSHOT (A2UI: IpReputationCard)│
  │ ───────────────────────────────────►   │  card renders on canvas
  │  TOOL_CALL_START (lookup_ports)        │
  │ ───────────────────────────────────►   │  "Scanning ports…" badge
  │  STATE_DELTA (A2UI: FindingsTable row) │
  │ ───────────────────────────────────►   │  table row appends live
  │  STATE_DELTA (A2UI: ActionButtons)     │
  │ ───────────────────────────────────►   │  action buttons appear
  │  INTERRUPT ("Isolate host requested")  │
  │ ───────────────────────────────────►   │  ApprovalModal opens, agent pauses
  │                          [User clicks Approve]
  │  ◄──────────────────────────────────   │  resume signal sent
  │  TOOL_CALL_START (execute_action)      │
  │ ───────────────────────────────────►   │
  │  STATE_DELTA (status: "Isolated")      │
  │ ───────────────────────────────────►   │  card status badge updates
  │  RUN_FINISHED                          │
  │ ───────────────────────────────────►   │  spinner stops
```

---

## 7. A2UI Component Schemas

These are the JSON structures the agent emits. They map to our React renderers via `a2ui-registry.ts`.

### IncidentCard

```json
{
  "type": "card",
  "id": "incident-card",
  "props": {
    "title": "Security Incident #2026-0510-001",
    "severity": "CRITICAL",
    "source": "Wazuh",
    "timestamp": "2026-05-10T03:14:00Z",
    "summary": "Outbound C2 beaconing to known Tor exit node",
    "affected_asset": "prod-api-03",
    "status": "investigating"
  }
}
```

### IpReputationCard

```json
{
  "type": "card",
  "id": "ip-reputation",
  "props": {
    "ip": "185.220.101.47",
    "abuse_score": 97,
    "total_reports": 847,
    "last_reported": "2026-05-09T22:00:00Z",
    "categories": ["Tor Exit Node", "C2", "Port Scan"],
    "country": "DE",
    "isp": "Frantech Solutions"
  }
}
```

### FindingsTable

```json
{
  "type": "table",
  "id": "findings-table",
  "props": {
    "columns": ["Check", "Result", "Confidence"],
    "rows": [
      ["IP Reputation", "Malicious (97/100)", "HIGH"],
      ["Open Ports", "443, 9001 (Tor signature)", "HIGH"],
      ["Asset Environment", "Production", "—"],
      ["Process", "python3 / www-data", "MEDIUM"]
    ]
  }
}
```

### ActionButtons

```json
{
  "type": "button-group",
  "id": "action-buttons",
  "props": {
    "buttons": [
      { "label": "Block IP at Firewall", "action": "block_ip", "variant": "danger", "requires_approval": true },
      { "label": "Isolate Host", "action": "isolate_host", "variant": "danger", "requires_approval": true },
      { "label": "Create Ticket", "action": "create_ticket", "variant": "primary", "requires_approval": false },
      { "label": "Mark False Positive", "action": "dismiss", "variant": "secondary", "requires_approval": false }
    ]
  }
}
```

### ApprovalModal (rendered on INTERRUPT)

```json
{
  "type": "interrupt",
  "id": "approval-modal",
  "props": {
    "action": "isolate_host",
    "target": "prod-api-03",
    "description": "This will cut all network connectivity to the host. Services running on it will become unreachable.",
    "severity": "CRITICAL",
    "confirm_label": "Approve Isolation",
    "reject_label": "Cancel"
  }
}
```

---

## 8. Tool Integrations

### Tool 1: AbuseIPDB IP Reputation (`tools/abuseipdb.py`)

- **Endpoint:** `https://api.abuseipdb.com/api/v2/check`
- **Input:** IP address string
- **Output:** `{ abuse_score, total_reports, categories, isp, country, last_reported }`
- **Stub fallback:** if key missing, return hardcoded high-score result for demo

```python
@tool
def lookup_ip_reputation(ip: str) -> dict:
    """Check an IP address against AbuseIPDB for malicious activity reports."""
```

### Tool 2: NVD CVE Lookup (`tools/nvd_cve.py`)

- **Endpoint:** `https://services.nvd.nist.gov/rest/json/cves/2.0?cveId={id}`
- **Input:** CVE ID (e.g., `CVE-2024-12345`)
- **Output:** `{ cvss_score, severity, description, affected_products, published_date, patch_available }`
- **No API key required** (rate-limited without key, key speeds it up)

```python
@tool
def lookup_cve(cve_id: str) -> dict:
    """Look up CVE details from the National Vulnerability Database."""
```

### Tool 3: Shodan Port Lookup (`tools/shodan_lookup.py`)

- **Endpoint:** `https://api.shodan.io/shodan/host/{ip}`
- **Input:** IP address
- **Output:** `{ ports, banners, os, hostnames, tags }`
- **Stub fallback:** return `[443, 9001]` with Tor signature for demo if no key

```python
@tool
def lookup_ports(ip: str) -> dict:
    """Look up open ports and service banners for an IP via Shodan."""
```

### Tool 4: Asset Registry (`tools/asset_lookup.py`)

- **In-memory mock** (no external API needed)
- **Input:** hostname or IP
- **Output:** `{ hostname, environment, owner, criticality, last_seen }`
- Pre-seeded with `prod-api-03`, `dev-worker-01`, etc. for demo

```python
@tool
def lookup_asset(identifier: str) -> dict:
    """Look up internal asset metadata by hostname or IP."""
```

### Tool 5: Action Executor (`tools/action_executor.py`)

- **All stubbed** — logs to console, returns success/failure JSON
- Actions: `block_ip`, `isolate_host`, `create_ticket`, `revoke_credential`
- In a real setup: calls pfsense API, AWS SSM, Jira API
- INTERRUPT must be fired by the agent **before** calling this tool for `requires_approval: true` actions

```python
@tool
def execute_action(action_type: str, target: str, metadata: dict = {}) -> dict:
    """Execute a security response action. Only call after receiving user approval."""
```

### MCP Server Exposure (`apps/mcp/src/index.ts`)

Expose `lookup_ip_reputation`, `lookup_cve`, and `execute_action` as MCP tools. This lets the exact same agent run as a native surface in Claude or ChatGPT — bonus demo moment.

---

## 9. Agent Logic (Python, `apps/agent/src/agent.py`)

### System Prompt

```
You are SecTriage, an AI security incident response agent.

When given a raw security alert (log line, CVE ID, IP address, file hash, etc.):
1. Identify the alert type: [ip_threat | cve | malware | anomaly | unknown]
2. Call the relevant enrichment tools based on alert type
3. After each tool result, emit an A2UI component via state update:
   - IP results → IpReputationCard
   - CVE results → CveCard  
   - Asset results → IncidentCard update
   - All findings → FindingsTable rows
4. Determine severity: LOW / MEDIUM / HIGH / CRITICAL based on findings
5. Generate ActionButtons with appropriate response options
6. For any action with requires_approval: true AND severity HIGH or CRITICAL:
   STOP. Emit an INTERRUPT with action details. Wait for user approval.
7. Only after approval: call execute_action with the approved action.
8. Emit a STATE_DELTA to update the incident status.

Always emit A2UI JSON. Never respond with plain prose summaries.
```

### LangGraph Node Structure

```
START
  │
  ▼
classify_alert          ← determines alert type + extracts IOCs
  │
  ▼
enrich_parallel         ← calls 2-3 tools concurrently (LangGraph parallel branch)
  │    └── ip_reputation
  │    └── asset_lookup
  │    └── cve_lookup (if CVE in alert)
  ▼
render_components       ← emits A2UI STATE_SNAPSHOT with all cards + table
  │
  ▼
determine_severity      ← scores based on enrichment results
  │
  ▼
generate_actions        ← builds ActionButtons A2UI component
  │
  ├── [LOW/MEDIUM, no approval needed] ──► execute_action ──► update_state ──► END
  │
  └── [HIGH/CRITICAL, destructive] ──► emit INTERRUPT ──► wait_for_approval
                                                                │
                                          [Approved] ──────────► execute_action ──► update_state ──► END
                                          [Rejected] ──────────► log_rejection ──► END
```

---

## 10. Frontend Components

### `AlertInput.tsx`

A centered textarea above the canvas. On submit, calls `useCopilotAction` to send the alert text to the agent. Shows a pulsing border while agent is running.

### `a2ui-registry.ts`

Maps A2UI type strings to React components:

```typescript
export const registry: Record<string, React.ComponentType<any>> = {
  "card":         DynamicCard,       // routes to IncidentCard / IpReputationCard / CveCard by id
  "table":        FindingsTable,
  "button-group": ActionButtons,
  "interrupt":    ApprovalModal,
  "progress-bar": ProgressBar,       // from starter kit, keep as-is
};
```

### `ApprovalModal.tsx`

Triggered by AG-UI `INTERRUPT` event via CopilotKit's `useCopilotInterrupt` hook (or equivalent). Shows:
- Action name + target in bold
- Consequence description in muted text
- Severity badge (red for CRITICAL)
- Two buttons: Approve (red) / Cancel (gray)
- Blocks all other interaction while open

### `SeverityBadge.tsx`

```
LOW      → gray pill
MEDIUM   → yellow pill
HIGH     → orange pill
CRITICAL → red pill, pulsing animation
```

---

## 11. Build Timeline

The hackathon is a live build. This timeline assumes 2 people.

| Block | Time | Owner | Task | Done When |
|---|---|---|---|---|
| **0** | Pre-event | Both | Install deps, get API keys, clone kit, run `npm run dev` | Echo agent responds in browser |
| **1** | 0:00–0:45 | Both | Replace Notion tools with security tools skeleton; update system prompt | `npm run dev` boots, agent replies to "hello" |
| **2** | 0:45–1:30 | Person A | Implement `abuseipdb.py` + `nvd_cve.py` tools with real API calls | Tool returns real JSON for test IP |
| **2** | 0:45–1:30 | Person B | Build `IncidentCard.tsx` + `IpReputationCard.tsx` + `a2ui-registry.ts` | Cards render from hardcoded A2UI JSON |
| **3** | 1:30–2:30 | Both | Wire agent to emit real A2UI STATE_SNAPSHOT after tool calls | Paste alert → cards appear on canvas |
| **4** | 2:30–3:15 | Person A | Implement `FindingsTable.tsx` + STATE_DELTA streaming (rows appear one by one) | Table populates live during agent run |
| **4** | 2:30–3:15 | Person B | Implement INTERRUPT flow: `ApprovalModal.tsx` + agent pause/resume logic | Modal blocks, approve resumes agent |
| **5** | 3:15–4:00 | Both | Implement `ActionButtons.tsx` + `execute_action` stub | Clicking "Block IP" triggers modal then executes |
| **6** | 4:00–4:30 | Person B | Add `SeverityBadge`, status updates, polish canvas layout | Looks clean, no layout breaks |
| **6** | 4:00–4:30 | Person A | Wire Shodan stub + asset registry + test with 3 alert scenarios | All 3 golden path scenarios work end-to-end |
| **7** | 4:30–5:00 | Both | Expose 2 tools in `apps/mcp/` — bonus Claude/ChatGPT demo surface | `npm run dev:mcp` works, tools visible in MCP Inspector |
| **8** | 5:00–6:00 | Both | Demo prep: rehearse golden path, fix edge cases, confirm on-screen layout | Can demo without notes, under 3 minutes |

### Contingency Rules

- If blocked on INTERRUPT implementation: hardcode a 3-second pause + auto-modal instead
- If Shodan key unavailable: return stub `[443, 9001]` — nobody will notice in a demo
- If CopilotKit Intelligence (Postgres/Docker) won't start: remove persistent threads, run stateless — still fully demoable
- If any tool API is down: every tool has a hardcoded stub fallback, flip `USE_STUBS=true` in env

---

## 12. Demo Script (Golden Path — 3 Minutes)

**Setup:** Browser open, canvas empty, AlertInput focused.

**Step 1 (0:00–0:20) — Paste the alert**
> "Here's a real-world scenario. Our monitoring system fires an alert about suspicious outbound traffic."

Paste:
```
ALERT: Outbound connection to 185.220.101.47:443 from prod-api-03.
Process: python3 (pid 9821). User: www-data. Time: 03:14 UTC.
```

**Step 2 (0:20–0:50) — Watch the canvas come alive**
> "The agent isn't writing a text reply. It's building a UI. Watch what happens."

Point out: cards appearing, tool call badges flashing, table rows streaming in.

**Step 3 (0:50–1:20) — Point out the intelligence**
> "It already knows this IP — 847 abuse reports, classified as a Tor exit node and C2 server. And it found the asset: prod-api-03 is a production server. That combination pushed severity to CRITICAL."

**Step 4 (1:20–1:50) — Trigger the approval flow**
> "Now I click 'Isolate Host'."

Modal appears. Agent has paused.
> "This is the INTERRUPT event. The agent knows this action is irreversible. It stopped. It's waiting for me."

Approve.
> "I approve. And watch — the agent resumes and the status updates live."

Card badge flips to "Isolated."

**Step 5 (1:50–2:30) — Second scenario (optional if time allows)**
Paste a CVE alert: `"New critical vuln: CVE-2024-6387 (regreSSHion) detected on 14 hosts in our fleet."`
Show: CveCard renders, CVSS score, affected hosts table, patch action button (no INTERRUPT needed — non-destructive).

**Step 6 (2:30–3:00) — Close**
> "No chat. No summary. The interface is the agent's output. The INTERRUPT isn't a UX trick — in security, a human must be in the loop before you touch production. This is what agentic UI looks like when the stakes are real."

---

## 13. Quick Start (Day-Of Commands)

```bash
# 1. Clone the starter kit
git clone https://github.com/jerelvelarde/Generative-UI-Global-Hackathon-Starter-Kit
cd Generative-UI-Global-Hackathon-Starter-Kit

# 2. Init CopilotKit Intelligence (sets up Postgres thread storage)
npx @copilotkit/cli@latest init
# select "Intelligence" when prompted

# 3. Drop in env vars
cp .env.example .env
# edit .env: add GEMINI_API_KEY
cp apps/agent/.env.example apps/agent/.env
# edit apps/agent/.env: add GEMINI_API_KEY, ABUSEIPDB_API_KEY

# 4. Install and run
npm install
npm run dev
# → frontend:  http://localhost:3000
# → agent:     http://localhost:8000
# → pre-flight check will tell you if anything is missing

# 5. (Optional) Run MCP server too
npm run dev:full
```

---

## 14. Key Reference Links

| Resource | Link |
|---|---|
| Starter kit | github.com/jerelvelarde/Generative-UI-Global-Hackathon-Starter-Kit |
| A2UI spec + composer | a2ui.org · a2ui-composer.ag-ui.com |
| AG-UI protocol | ag-ui.com |
| CopilotKit docs | docs.copilotkit.ai |
| LangChain Deep Agents | github.com/langchain-ai/deepagents |
| AbuseIPDB API docs | docs.abuseipdb.com |
| NVD CVE API | nvd.nist.gov/developers |
| Shodan API | developer.shodan.io |
| Gemini API | aistudio.google.com |

---

## 15. What Wins This

> Everyone can build a to-do agent.
>
> Not everyone builds one that **pauses before isolating a production server** and demands human sign-off.
>
> The INTERRUPT is the demo. Everything else is set-up.
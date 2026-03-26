# O2C Intelligence Platform
### Graph-Based Order to Cash Data Explorer with AI Query Interface

> A context graph system built on real SAP Order-to-Cash data, featuring interactive graph visualization, an analytics dashboard, flow analysis, and a natural language query interface powered by Claude (Anthropic).

---

## 🔗 Live Demo
**[View Live Demo →](YOUR_DEMO_LINK_HERE)**

---

## 📸 Screenshots

> Graph Explorer · Analytics Dashboard · Flow Analysis · AI Chat Interface

---

## 🏗️ Architecture Overview

```
┌─────────────────────────────────────────────────────────────┐
│                    Single-Page Application                   │
│                   (Self-contained HTML/JS)                   │
├──────────────┬──────────────┬──────────────┬────────────────┤
│  Graph Tab   │ Analytics Tab│  Flows Tab   │   Chat Tab     │
│  (D3.js)     │ (SVG Charts) │ (Full Table) │ (Claude API)   │
├──────────────┴──────────────┴──────────────┴────────────────┤
│                    In-Memory Query Engine                     │
│              (JavaScript · Pre-computed Analytics)           │
├─────────────────────────────────────────────────────────────┤
│                    Embedded Dataset (JSON)                    │
│     SAP O2C · 19 Entity Types · ~1,600 Records Total        │
└─────────────────────────────────────────────────────────────┘
```

The entire application is a **single HTML file** with no backend, no server, and no external dependencies beyond D3.js and Google Fonts. All SAP data is embedded directly in the file at build time.

---

## 📂 Dataset

The application ingests the SAP O2C dataset consisting of 19 entity types:

| Entity | Records | Role |
|--------|---------|------|
| sales_order_headers | 100 | Core — order creation |
| sales_order_items | 167 | Detail — line items |
| outbound_delivery_headers | 86 | Core — shipment |
| outbound_delivery_items | 137 | Detail — delivery lines |
| billing_document_headers | 163 | Core — invoicing |
| billing_document_items | 200 | Detail — billing lines |
| billing_document_cancellations | 80 | Cancellation records |
| journal_entry_items_accounts_receivable | 123 | Accounting |
| payments_accounts_receivable | 120 | Payment receipts |
| business_partners | 8 | Master — customers |
| products | 69 | Master — materials |
| product_descriptions | 69 | Master — product names |
| plants | 44 | Master — locations |
| sales_order_schedule_lines | 179 | Scheduling |
| customer_company_assignments | 8 | Master |
| customer_sales_area_assignments | 28 | Master |
| business_partner_addresses | 8 | Master |
| product_plants | 3,036 | Plant assignments |
| product_storage_locations | 16,723 | Storage data |

---

## 🔗 Graph Model

### Node Types (7)

| Node Type | Color | Represents |
|-----------|-------|------------|
| Customer | Purple | Business partner / sold-to party |
| Sales Order | Blue | Purchase order header |
| Delivery | Green | Outbound delivery |
| Billing Doc | Amber | Invoice / billing document |
| Journal Entry | Orange | Accounting document |
| Payment | Red | Accounts receivable clearing |
| Product | Cyan | Material / SKU |

### Edge Types (Relationships)

| Relationship | From → To | Key Field |
|---|---|---|
| `ordered` | Customer → Sales Order | `soldToParty` |
| `delivered_via` | Sales Order → Delivery | `outbound_delivery_items.referenceSdDocument` |
| `billed_from` | Delivery → Billing Doc | `billing_document_items.referenceSdDocument` |
| `billed_to` | Customer → Billing Doc | `soldToParty` |
| `posted_to` | Billing Doc → Journal Entry | `journal_entry.referenceDocument` |
| `cleared_by` | Journal Entry → Payment | `payments.clearingAccountingDocument` |
| `contains` | Sales Order → Product | `sales_order_items.material` |

### Core Business Flow
```
Customer
  └─► Sales Order
           └─► Delivery (via delivery items)
                    └─► Billing Document (via billing items)
                               └─► Journal Entry (via referenceDocument)
                                          └─► Payment (via clearingAccountingDocument)
```

---

## 💾 Database / Storage Choice

**Decision: In-Memory JavaScript (embedded JSON)**

This was a deliberate architectural choice over alternatives like Neo4j, SQLite, or a REST API backend.

**Why in-memory:**
- **Zero infrastructure** — the entire app deploys as a single static file to any CDN, GitHub Pages, or Netlify with no server
- **Instant queries** — all data is pre-loaded in memory; lookups are O(1) via hash maps built at startup
- **Dataset size** — at ~1,600 meaningful records, in-memory is the right tradeoff; a graph DB would add overhead without benefit at this scale
- **Portability** — the HTML file itself is the database; easy to share, version, and demo

**Tradeoffs acknowledged:**
- Cannot handle dataset growth beyond ~50,000 records without switching to IndexedDB or a proper backend
- No persistence — filters/highlights reset on refresh
- For production scale, the right choice would be **Neo4j** (native graph queries with Cypher) or **PostgreSQL with recursive CTEs** for flow traversal

---

## 🤖 LLM Integration & Prompting Strategy

### Two-Layer Query Architecture

```
User Query
    │
    ▼
┌─────────────────────────────┐
│   Layer 1: Local Query      │
│   Engine (JavaScript)       │
│   • Keyword + regex match   │
│   • Direct data lookup      │
│   • Zero latency, no API    │
└──────────┬──────────────────┘
           │ No match found
           ▼
┌─────────────────────────────┐
│   Layer 2: Claude API       │
│   claude-sonnet-4-20250514  │
│   • Natural language        │
│   • Complex reasoning       │
│   • Schema-aware prompting  │
└─────────────────────────────┘
```

### Local Engine Handles
- Document trace queries (`trace billing doc 90504248`)
- Aggregation queries (`top products by billing volume`)
- Status queries (`cancelled billing documents`)
- Flow completeness analysis (`incomplete orders`)
- Financial summaries (`revenue by customer`)

### Claude Handles
- Free-form questions not matching patterns
- Multi-entity reasoning
- Comparative analysis
- Narrative explanations

### System Prompt Strategy

The system prompt given to Claude contains:

1. **Hard domain restriction** — "You ONLY answer questions about this O2C dataset. Reject all other queries with a specific message."
2. **Full schema** — all table names, key fields, and data types
3. **Relationship map** — how entities link to each other
4. **Live dataset metrics** — actual record counts and KPIs injected at runtime
5. **Currency context** — "All amounts are in INR, use ₹ symbol"
6. **Behavioral instruction** — "Be concise, analytical, and data-specific"

**Conversation memory** is maintained by passing the last 12 messages in every API call, enabling multi-turn context like:
> "Show me incomplete flows" → "Which of those have the highest value?" ← understands context

---

## 🛡️ Guardrails

### Layer 1 — Local Engine Guardrails
The local query engine only responds to known O2C query patterns. Any unrecognized query is passed to Claude, never answered with hallucinated data. Every answer is **grounded in actual dataset values** — no synthetic responses.

### Layer 2 — LLM Prompt Guardrails
The system prompt explicitly instructs Claude:

```
"You ONLY answer questions about this specific dataset.
Reject all off-topic queries with:
'This system is designed to answer questions related
to the Order to Cash dataset only.'"
```

This handles:
- General knowledge questions ("What is the capital of France?")
- Creative writing requests
- Code generation requests
- Anything outside the O2C domain

### Layer 3 — Data Grounding
Claude is given the real schema and key metrics. It cannot fabricate document IDs or amounts because the system prompt anchors its responses to actual data structures. The local engine always takes priority and returns exact data — Claude is only invoked for interpretive questions.

---

## ✨ Features

### Graph Explorer
- Force-directed D3.js graph with 7 node types, color-coded
- Drag, zoom, and pan navigation
- Click any node to inspect all metadata in a tooltip
- 4 preset views: Full Graph, SO→Delivery, Billing Flow, By Customer
- Node type filters (show/hide any entity type)
- Query results highlight matching nodes on the graph for 10 seconds

### Analytics Dashboard
- Live KPI header bar (Sales Orders, Active Invoices, Cancelled, Incomplete Flows, Revenue)
- O2C Flow Funnel with drop-off percentages at each stage
- Top 10 products by billing document volume (bar chart)
- Revenue by customer (bar chart)
- Billing document status donut chart (SVG)
- Payment analysis — billed vs. collected vs. outstanding
- Entity summary table with record counts

### Flow Analysis Tab
- Complete status table for all 100 sales orders
- Each row shows: customer, amount, deliveries, billing docs, journal entries, status
- Color-coded: Complete / Not Billed (Warning) / Not Delivered (Critical)
- Alert cards for broken flow counts

### AI Query Interface
- Natural language queries answered from live data
- Conversation memory across turns
- Graph highlighting linked to query results
- Graceful degradation — works without API key for common queries
- Strict guardrails rejecting off-topic prompts

---

## 🚀 How to Run

### Option 1: Open directly
```bash
# Just open the HTML file in any browser
open o2c_explorer_v2.html
```

### Option 2: Deploy to GitHub Pages
```bash
git init
git add o2c_explorer_v2.html
git commit -m "O2C Intelligence Platform"
git branch -M main
git remote add origin https://github.com/YOUR_USERNAME/o2c-graph-explorer.git
git push -u origin main
# Enable GitHub Pages in repo settings → Pages → main branch → / (root)
```

### Option 3: Deploy to Netlify
Drag and drop the HTML file at [netlify.com/drop](https://netlify.com/drop) — instant live URL.

---

## 🔑 API Key

To enable AI-powered queries, enter your Anthropic API key in the interface. The key is never stored — it only lives in the browser session.

Get a free key at [console.anthropic.com](https://console.anthropic.com).

The application works without an API key for all pre-built query patterns (top products, incomplete flows, billing traces, revenue analysis, etc.).

---

## 🛠️ Tech Stack

| Technology | Usage |
|---|---|
| D3.js v7 | Force-directed graph visualization |
| Vanilla JavaScript | Query engine, data processing, UI |
| HTML5 / CSS3 | Single-file application |
| Claude API (claude-sonnet-4-20250514) | Natural language query processing |
| Google Fonts (Orbitron + Share Tech Mono) | Typography |
| Python 3 | Build-time data processing and JSON embedding |

---

## 📊 Key Findings from the Dataset

- **49.1% cancellation rate** on billing documents (80 of 163)
- **83 complete O2C flows** out of 100 sales orders (83%)
- **14 orders never delivered** — potential stuck orders
- **3 orders delivered but not billed** — revenue recognition risk
- **Top product**: Destiny 100ml EDP — appears in 16 billing documents
- **Largest customer**: Nelson, Fitzpatrick and Jordan — ₹27,776.93 in SO value (72 orders)
- **Total dataset revenue**: ₹70,877.13 across all sales orders

---

## 📁 Repository Structure

```
o2c-graph-explorer/
├── o2c_explorer_v2.html     # Complete application (self-contained)
├── README.md                # This file
├── data/
│   └── sap-o2c-dataset.zip  # Original dataset (19 entity folders, JSONL format)
└── build/
    └── build_app.py         # Python script used to embed data into HTML
```

---

## 🤝 AI Tools Used

This project was built with assistance from **Claude (Anthropic)** via Claude.ai.

AI was used for:
- Architecture planning and data model design
- Graph construction logic
- Query engine pattern matching
- Analytics computation
- UI/UX design and CSS
- System prompt engineering for guardrails

Session logs are included in the `/logs` folder of this repository.

---

*Built for the Forward Deployed Engineer assignment — Graph-Based Data Modeling and Query System*

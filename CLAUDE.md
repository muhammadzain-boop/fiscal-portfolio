# FinanceAI Suite — Project Intelligence

## Identity & Role
You are the **FinanceAI Suite Agent** — the core intelligence powering this financial dashboard. You understand every component, every data flow, and every design decision in this project. When the user asks for changes, new features, or debugging help, you act with full authority and deep context. You never need to ask "what project is this?" — you already know everything about it.

---

## Project Overview
**FinanceAI Suite** is a single-file React web portal (`index.html`) for financial analysis, served via `python -m http.server 8000`. No Node.js, no npm, no build step. Everything runs in-browser using Babel Standalone + UMD CDN scripts.

**Live URL:** `http://localhost:8000`
**Main file:** `c:\Users\Hp\Desktop\Munshi\index.html`
**Source file:** `c:\Users\Hp\Desktop\Munshi\financial-agents (1).jsx` (original reference)

---

## Tech Stack
| Layer | Technology | How loaded |
|---|---|---|
| UI Framework | React 18 | `unpkg.com/react@18/umd/react.development.js` |
| DOM Rendering | ReactDOM 18 | `unpkg.com/react-dom@18/umd/react-dom.development.js` |
| Charts | Recharts 2 | `unpkg.com/recharts@2/umd/Recharts.js` |
| Excel I/O | SheetJS (xlsx) | `unpkg.com/xlsx/dist/xlsx.full.min.js` |
| JSX Transpiler | Babel Standalone | `unpkg.com/@babel/standalone/babel.min.js` |
| Peer Dep | prop-types 15 | `unpkg.com/prop-types@15/prop-types.min.js` (must load BEFORE recharts) |
| AI | Claude API | Direct browser fetch to `api.anthropic.com/v1/messages` |
| ERP | Odoo JSON-RPC | Direct browser fetch to user's Odoo instance |
| Server | Python built-in | `python -m http.server 8000` |

**Critical load order in `<head>`:**
```
react → react-dom → prop-types → recharts → xlsx → babel
```
prop-types MUST come before recharts or `Recharts is not defined` error occurs.

---

## Architecture — Single File Pattern
All code lives inside ONE `<script type="text/babel">` tag in `index.html`.

**Global destructuring at top of script (replaces ES module imports):**
```js
const { useState, useRef, useEffect } = React;
const { createRoot } = ReactDOM;
const { BarChart, Bar, XAxis, YAxis, CartesianGrid, Tooltip,
        ResponsiveContainer, Cell, PieChart, Pie, Legend } = Recharts;
// XLSX is already global as window.XLSX
```

**Never use `import` statements inside the babel script** — this will break Babel Standalone.

---

## Component Tree
```
App
├── Header (sticky, 58px)
│   ├── Tab switcher: "variance" | "costing"
│   ├── ⚙ API Key button → ApiKeyModal
│   ├── 📖 Commands button → VoiceHelp modal
│   └── VoiceMic (Web Speech API)
├── [tab === "variance"] → Variance Workflow
│   ├── StepIndicator (4 steps, auto-advances)
│   ├── Step 0: BudgetUploader
│   ├── Step 1: OdooConnect
│   ├── Step 2: FetchActualsPanel
│   └── Step 3: VarianceDashboard
│       ├── KPI cards (6 metrics)
│       ├── View toggle: table | chart
│       ├── Filter: all | over | favorable | unbudgeted
│       ├── BarChart + PieChart (Recharts)
│       ├── Data table with sort
│       └── AIBox (Claude analysis)
├── [tab === "costing"] → CostingAgent
│   ├── Odoo BOM loader (mrp.bom)
│   ├── BOM table (Material/Labor/Overhead)
│   ├── KPI cards (unit cost, selling price, margin)
│   ├── PieChart (cost split) + BarChart (top items)
│   ├── SubTabs: bom | overhead | chat
│   └── AIBox (Claude optimization)
├── VoiceToast (bottom center, auto-dismiss 3.5s)
├── VoiceHelp modal
└── ApiKeyModal
```

---

## Theme System
```js
const T = {
  bg: "#07090f",      // page background
  card: "#0f1520",    // card background
  card2: "#131d2e",   // nested card
  border: "#1a2744",  // subtle border
  border2: "#223060", // stronger border
  text: "#dde5f8",    // primary text
  muted: "#4e6080",   // secondary text
  dim: "#273550",     // very muted
  green: "#00d98b",   // success / primary action
  red: "#ff3d5a",     // error / over-budget
  yellow: "#ffba08",  // warning / unbudgeted
  blue: "#3b9eff",    // info / Odoo
  purple: "#8b6fff",  // accent
  orange: "#ff7849",  // accent
};
const PIE = [T.green, T.blue, T.yellow, T.purple, T.orange, T.red];
```
**All styles are inline** using the T object. No CSS classes except keyframe animations in `<style>` tag.

---

## Shared UI Components
| Component | Props | Purpose |
|---|---|---|
| `Btn` | `children, onClick, variant, disabled, style, title` | Button. Variants: `primary`(green), `secondary`, `danger`, `odoo`(dark red), `outline` |
| `KPI` | `label, value, sub, color, icon` | Metric card, flex:1 |
| `Badge` | `children, color` | Pill badge. Colors: green/red/yellow/blue |
| `Spinner` | `size=14` | CSS spinning loader |
| `AIBox` | `content, loading` | Claude AI response box, green glow |
| `StepIndicator` | `steps, current` | 4-step progress bar |

**Input style constant:**
```js
const inp = { background: "rgba(255,255,255,0.04)", border: `1px solid ${T.border2}`, borderRadius: 7, padding: "9px 13px", color: T.text, fontSize: 13, outline: "none", width: "100%", fontFamily: "inherit" };
```

---

## Claude AI Integration
```js
async function callClaude(system, user) {
  const apiKey = localStorage.getItem("claude_api_key") || "";
  if (!apiKey) return "⚠ No API key set. Click ⚙ Settings in the header.";
  const r = await fetch("https://api.anthropic.com/v1/messages", {
    method: "POST",
    headers: {
      "Content-Type": "application/json",
      "x-api-key": apiKey,
      "anthropic-version": "2023-06-01",
      "anthropic-dangerous-allow-browser": "true",  // required for browser calls
    },
    body: JSON.stringify({
      model: "claude-sonnet-4-20250514",
      max_tokens: 1200,
      system,
      messages: [{ role: "user", content: user }],
    }),
  });
  const data = await r.json();
  if (data.error) return `⚠ API Error: ${data.error.message}`;
  return data.content?.[0]?.text || "No response.";
}
```
**API key storage:** `localStorage.getItem("claude_api_key")` — never hardcoded, never sent to third parties.
**Model:** `claude-sonnet-4-20250514`

---

## Odoo Integration
**Class:** `OdooClient(url, db, username, password)`

| Method | Odoo endpoint | Purpose |
|---|---|---|
| `authenticate()` | `/web/session/authenticate` | Login, stores uid |
| `searchRead(model, domain, fields, limit)` | `/web/dataset/call_kw` | Generic read |
| `fetchActuals(dateFrom, dateTo, groupBy)` | `account.move.line` | Expense actuals |

**Variance agent queries `account.move.line`** filtered by:
- `date >= dateFrom`, `date <= dateTo`
- `parent_state = posted`
- `account_id.account_type in [expense, income, expense_direct_cost, expense_depreciation]`
- `company_id.active = true`

**Costing agent queries:**
- `mrp.bom` — list of Bills of Materials
- `mrp.bom.line` — BOM line items
- `product.product` — product standard prices

---

## Data Formatters
```js
const fmt$ = (n) => `$${Number(n||0).toLocaleString("en-US", {minimumFractionDigits:2, maximumFractionDigits:2})}`;
const fmtK = (n) => Math.abs(n) >= 1000 ? `$${(n/1000).toFixed(1)}K` : fmt$(n);
const fmtPct = (n) => `${Number(n||0).toFixed(1)}%`;
const sign = (n) => (n > 0 ? "+" : "");
```

---

## Variance Workflow State (in App component)
```js
const [workflowStep, setWorkflowStep] = useState(0); // 0-3
// Auto-advance rules:
useEffect(() => { if (budgetData && workflowStep === 0) setWorkflowStep(1); }, [budgetData]);
useEffect(() => { if (odooClient && workflowStep === 1) setWorkflowStep(2); }, [odooClient]);
useEffect(() => { if (actualsData && workflowStep < 3) setWorkflowStep(3); }, [actualsData]);
```

**Budget matching logic (FetchActualsPanel):**
1. Exact match (case-insensitive)
2. Keyword match (split on spaces/slashes/dashes, min 4-char words)
3. Unmatched → actual=0
4. Odoo items not in budget → added as `matchType: "unbudgeted"`

---

## Known Issues & Fixes Applied
| Issue | Root cause | Fix applied |
|---|---|---|
| `React is not defined` | Babel JSX uses `React.createElement` globally but ES imports don't expose globals | Switched from importmap/esm.sh to UMD CDN scripts |
| `Recharts is not defined` | Recharts UMD needs prop-types as external peer dep | Added `prop-types@15` CDN script BEFORE recharts |
| Claude API 401 | Missing `x-api-key` header in original code | Added key from localStorage + `anthropic-dangerous-allow-browser: true` |
| Blank page, no error | Babel fails silently | Added `window.onerror` handler + red error box + loading spinner |

---

## Development Workflow
```bash
# Start server
cd C:\Users\Hp\Desktop\Munshi
python -m http.server 8000

# Open browser
http://localhost:8000

# Edit file
# index.html — all code in one file

# Hard refresh after edits
Ctrl + Shift + R
```
**Never open index.html as file:// directly** — ES module restrictions break CDN imports. Always use the Python server.

---

## Coding Conventions
- **All styles inline** — use `style={{ ... }}` with T theme object
- **No CSS classes** for components — only for global animations (`@keyframes`)
- **Functional components only** — hooks, no class components
- **State co-located** — each agent manages its own state, App holds shared state (odooClient, tab)
- **No PropTypes** — not needed for internal components
- **Keep it single-file** — do not split into multiple files unless explicitly asked
- **Emoji icons in UI** — used for buttons and labels (📊 🔧 🤖 ⬇ etc.)

---

## Adding New Agents
To add a new tab/agent:
1. Add tab button in header: `[["variance","📊 Variance"], ["costing","🔧 Costing"], ["newagent","🆕 Label"]]`
2. Create `function NewAgent({ odooClient }) { ... }` component
3. Add `{tab === "newagent" && <NewAgent odooClient={odooClient} />}` in main content
4. Use existing shared components: `Btn`, `KPI`, `Badge`, `Spinner`, `AIBox`
5. Call `callClaude(system, user)` for AI features
6. Use `odooClient.searchRead(model, domain, fields)` for Odoo data

---

## User Preferences
- **Platform:** Windows 11, bash shell in VS Code / Cursor
- **Python:** Available (`python -m http.server`) — Node.js NOT installed
- **No build tools** — keep everything as single HTML file
- **Dark theme only** — never suggest light mode changes
- **Odoo ERP user** — familiar with Odoo models and JSON-RPC
- **Wants working code** — not boilerplate or placeholders

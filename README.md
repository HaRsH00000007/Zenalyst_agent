# 🛡️ Zenalyst — Deterministic AI Workforce for Financial Reconciliation

[![Live Demo](https://img.shields.io/badge/Live%20Demo-Vercel-black?logo=vercel)](https://multi-agent-finance.vercel.app)
[![Backend API](https://img.shields.io/badge/Backend-Render-46E3B7?logo=render)](https://multi-agent-finance-backend.onrender.com/docs)
[![License](https://img.shields.io/badge/License-Apache%202.0-blue)](LICENSE)
[![Python](https://img.shields.io/badge/Python-3.10+-blue?logo=python)](https://python.org)
[![React](https://img.shields.io/badge/React-18-61DAFB?logo=react)](https://react.dev)

> **"LLMs write code. Code runs the math. Never the other way around."**

Zenalyst is a production-grade, full-stack AI platform for large-scale financial CSV reconciliation and audit. It uses a **Sovereign Intelligence architecture** — AI agents write deterministic Python/Pandas code which is executed in a sandboxed environment — eliminating hallucinations and guaranteeing 100% mathematically accurate results.

Tested on real datasets with **313,000+ rows** and **67,000+ duplicates** removed in a single session.

---

## 🌐 Live Deployment

| Service | URL |
|---|---|
| 🖥️ Frontend (Vercel) | https://multi-agent-finance.vercel.app |
| ⚙️ Backend API (Render) | https://multi-agent-finance-backend.onrender.com/docs |
| 💻 Source Code | https://github.com/HaRsH00000007/Zenalyst_agent |

---

## 🏗️ System Architecture

```
                        ┌─────────────────────┐
                        │   CSV File Upload    │
                        └──────────┬──────────┘
                                   │
                                   ▼
                    ┌──────────────────────────────┐
                    │      ZenForce (Orchestrator)  │
                    │  Coordinates agents · Streams │
                    │  real-time thoughts via SSE   │
                    └────┬──────────────────┬───────┘
                         │                  │
               ┌─────────▼──────┐  ┌───────▼────────┐
               │   ZenRecon     │  │   ZenVault     │
               │  (Analyst)     │  │  (Auditor)     │
               │  3-Gate ML     │  │  Integrity     │
               │  Pipeline      │  │  Verification  │
               └─────────┬──────┘  └───────┬────────┘
                         │                  │
                         └────────┬─────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │       ZenView              │
                    │  (Visualization Agent)     │
                    │  LLM writes matplotlib     │
                    │  code · Sandbox executes   │
                    └─────────────┬──────────────┘
                                  │
                    ┌─────────────▼──────────────┐
                    │       ZenChat              │
                    │  (RAG Q&A Agent)           │
                    │  DataFrame-Augmented Gen   │
                    │  Grounded answers only     │
                    └────────────────────────────┘
```

### The Determinism Contract

```
LLM writes Pandas code  →  Sandbox executor runs it  →  Mathematically exact result
```

The LLM **never performs arithmetic** inside its context window. It only writes Python code. All calculations happen in the executor. This is what makes Zenalyst suitable for financial data.

---

## 🤖 Agent Descriptions

### ZenForce — Orchestrator
- Coordinates the full multi-agent pipeline end-to-end
- Streams real-time **thought signatures** to the frontend via Server-Sent Events (SSE)
- Type-dispatches generator events: `str` → forward as thought, `pd.DataFrame` → pass to next agent
- Optional Groq call at `temp=0.2` for coordination commentary

### ZenRecon — Data Analyst (3-Gate Pipeline)

| Gate | Name | What Happens |
|---|---|---|
| Gate 0 | EDA Audit | shape, dtypes, null counts, duplicates, describe, correlations — all hardcoded, no LLM |
| Gate 1 | LLM Cleaning | Groq writes Pandas cleaning code at `temp=0.0`; creates `CompositeKey = Date\|Amount\|Vendor_Slug` |
| Gate 2 | Deduplication | Hardcoded `drop_duplicates()` on CompositeKey; captures before/after row counts |

### ZenVault — Auditor
- Validates data integrity after cleaning
- Checks: residual nulls, CompositeKey presence, duplicate elimination completeness
- Produces structured audit report: `PASS / WARN / FAIL`

### ZenView — Visualization Agent
- LLM writes `matplotlib`/`seaborn` code grounded in real DataFrame metadata (`df.describe()`, `value_counts()`, dtypes)
- Executed in **safe Python sandbox** — `plt`, `sns`, `SAVE_PATH` are pre-injected
- LLM cannot call `plt.show()`, cannot import anything, cannot save to arbitrary paths
- Output PNG served via `/plot` endpoint

### ZenChat — RAG Q&A Agent
Context window = last 30 thought signatures + ZenVault audit JSON + `df.head(20).to_markdown()`
- Answers questions **grounded only in session data** — never invents numbers
- Powers the **AI Conclusion card** on the Visualize page with analyst-grade insights

---

## 🔒 Safe Execution Sandbox

The most critical component — makes LLM-generated code safe to run in production:

```python
# Allowlist: only these modules can be imported inside LLM-generated code
_ALLOWED_MODULES = {
    'pandas', 'numpy', 'matplotlib', 'seaborn',
    're', 'math', 'datetime', 'collections'
}

def _safe_import(name, *args, **kwargs):
    if name not in _ALLOWED_MODULES:
        raise ImportError(f"Module '{name}' is blocked in sandbox")
    return original_import(name, *args, **kwargs)
```

**Security properties:**
- Blocks `os`, `subprocess`, `socket`, `sys`, and all other dangerous modules
- Always operates on `df.copy()` — original data is never mutated
- Captures `stdout`/`stderr` via `contextlib.redirect_stdout` so `df.info()` streams to UI
- Visualization variant pre-injects `plt`, `sns`, `SAVE_PATH`; calls `plt.close('all')` in `finally` block
- Syntax validation runs before execution — malformed code fails fast

---

## 📁 Project Structure

```
Zenalyst_agent/
│
├── project-bolt-sb1-3ahzypkk/project/        # ── Frontend (React + TypeScript)
│   ├── src/
│   │   ├── components/
│   │   │   ├── UploadReconcile.tsx            # Page 1: CSV upload + live pipeline
│   │   │   ├── Visualize.tsx                  # Page 2: Charts + AI conclusion
│   │   │   ├── ZenChat.tsx                    # Page 3: RAG chat interface
│   │   │   ├── Layout.tsx                     # Navbar + routing shell
│   │   │   ├── FileUpload.tsx                 # Drag-and-drop upload
│   │   │   └── Terminal.tsx                   # Live thought stream terminal
│   │   ├── context/
│   │   │   └── SessionContext.tsx             # Global session state (React Context)
│   │   └── lib/
│   │       ├── api.ts                         # All API calls + SSE stream parsing
│   │       └── types.ts                       # TypeScript interfaces
│   ├── package.json
│   ├── vite.config.ts
│   └── tailwind.config.js
│
└── zenalyst-workforce/backend/                # ── Backend (FastAPI + Python)
    ├── main.py                                # FastAPI app — 5 endpoints + session state
    ├── agents/
    │   ├── orchestrator.py                    # ZenForce — agent coordinator
    │   ├── analyst.py                         # ZenRecon — 3-gate pipeline
    │   └── visualizer.py                      # ZenView — chart generation agent
    └── tools/
        └── executor.py                        # Safe Python execution sandbox
```

---

## 🔌 API Reference

| Method | Endpoint | Description |
|---|---|---|
| `POST` | `/reconcile` | Upload CSV → SSE stream of thoughts + final summary |
| `POST` | `/visualize` | Trigger ZenView → SSE stream → matplotlib PNG |
| `GET` | `/plot` | Serve the generated chart PNG |
| `POST` | `/ask` | ZenChat: answer questions grounded in session data |
| `GET` | `/health` | Backend status + session presence |

### SSE Event Format
```json
{ "type": "thought",    "data": "⚙️ ZenRecon [Gate 0] :: Running EDA..." }
{ "type": "summary",    "data": { "session_id": "ZF-...", "clean_rows": 313999, ... }}
{ "type": "viz_result", "data": { "success": true, "plot_path": "/tmp/zen_plot.png" }}
```

---

## 🖥️ Frontend Pages

### Page 1 — Reconcile
- Drag-and-drop CSV upload (any size)
- Live terminal showing ZenForce thought stream in real time via SSE
- Summary dashboard on completion:
  - Original Rows → Clean Rows → Duplicates Removed → Integrity Status
  - Residual Nulls + CompositeKey presence

### Page 2 — Visualize
All charts are built from `sessionData` already in React state — zero additional API calls required for rendering:

| Chart | What It Shows |
|---|---|
| 🔮 AI Conclusion | ZenChat generates a 3-sentence analyst conclusion from session metrics |
| 📊 Data Quality Gauge | `clean_rows / original_rows × 100%` as an arc gauge |
| 📊 Duplicate Rate Gauge | `duplicates / original_rows × 100%` with risk assessment |
| 📊 Row Breakdown Bar | Original vs Clean vs Duplicates — absolute count comparison |
| 🥧 Data Composition Donut | Proportion of clean / duplicates / residual nulls |
| ✅ Audit Checklist | 4 ZenVault checks with PASS/FAIL badges |
| 🔽 Reconciliation Funnel | Visual pipeline: Raw → Dedup → Clean → Status |
| 🤖 Raw Agent PNG | Collapsible: matplotlib chart from ZenView sandbox |

### Page 3 — ZenChat
- Full chat interface with starter question chips
- Every response tagged as **Grounded** (from session data) or **Ungrounded**
- Persistent conversation history within session

---

## ⚙️ Local Development Setup

### Prerequisites
- Node.js 18+
- Python 3.10+
- Groq API key — free at [console.groq.com](https://console.groq.com)

### 1. Clone the repo
```bash
git clone https://github.com/HaRsH00000007/Zenalyst_agent.git
cd Zenalyst_agent
```

### 2. Start the Backend
```bash
cd zenalyst-workforce/backend
pip install -r requirements.txt

# Create .env file
echo "GROQ_API_KEY=your_groq_key_here" > .env

python main.py
# API runs at: http://localhost:8000
# Swagger docs: http://localhost:8000/docs
```

### 3. Start the Frontend
```bash
cd project-bolt-sb1-3ahzypkk/project
npm install
npm run dev
# App runs at: http://localhost:5173
```

---

## 🚀 Production Deployment

### Backend → Render
1. Connect your GitHub repo to [render.com](https://render.com)
2. Set environment variable: `GROQ_API_KEY=your_key`
3. Build command: `pip install -r requirements.txt`
4. Start command: `python main.py`

### Frontend → Vercel
1. Connect your GitHub repo to [vercel.com](https://vercel.com)
2. Set root directory to: `project-bolt-sb1-3ahzypkk/project`
3. Add environment variable:
   - `VITE_API_URL` = `https://your-render-backend.onrender.com`
4. Deploy — Vercel auto-deploys on every `git push`

### Environment Variables Summary

| Variable | Where | Value |
|---|---|---|
| `GROQ_API_KEY` | Render (Backend) | Your Groq API key |
| `VITE_API_URL` | Vercel (Frontend) | Your Render backend URL |

---

## 🛠️ Tech Stack

### Frontend
| Tech | Version | Purpose |
|---|---|---|
| React | 18 | UI framework |
| TypeScript | 5 | Type safety |
| Vite | 5 | Build tool |
| Tailwind CSS | 3 | Styling |
| Framer Motion | 11 | Animations |
| Recharts | 2 | Native chart components |
| React Router | 6 | Client-side routing |
| Lucide React | — | Icons |

### Backend
| Tech | Version | Purpose |
|---|---|---|
| FastAPI | — | REST API + SSE streaming |
| Groq SDK | — | LLM inference (`llama-3.1-70b-versatile`) |
| Pandas | — | Data processing |
| NumPy | — | Numerical operations |
| Matplotlib + Seaborn | — | Chart generation |
| Python-dotenv | — | Environment config |

---

## 🧪 Example Usage

1. Upload a dirty financial CSV (any size — tested up to 380k rows)
2. Watch ZenForce orchestrate the pipeline live in the terminal
3. See your clean data metrics: rows retained, duplicates removed, integrity status
4. Go to Visualize — charts load instantly from session data
5. Read the AI Conclusion for analyst-grade insights
6. Ask ZenChat anything: *"Which vendors have the most transactions?"*

---

## 🤝 Contributing

Pull requests are welcome. For major changes please open an issue first.

1. Fork the repo
2. Create your feature branch: `git checkout -b feature/my-feature`
3. Commit your changes: `git commit -m 'Add my feature'`
4. Push to the branch: `git push origin feature/my-feature`
5. Open a Pull Request

---

## 📄 License

[Apache 2.0](LICENSE) — Dharmendra / HaRsH00000007

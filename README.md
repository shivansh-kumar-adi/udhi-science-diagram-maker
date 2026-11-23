# Udhi Technologies — Science Diagram Maker
### AI-Powered Educational SVG Generator, Programmatic Validation Engine & Review Dashboard

An end-to-end full-stack web application designed for UK GCSE students and educators. The system accepts curriculum-standard scientific concepts, prompts Google Gemini for pure animated SVG graphics, executes 5 rigorous programmatic validation checks, logs token metrics and USD costs, and empowers human educators to review, approve, or reject each visual asset with persistent decision auditing.

Built for the **Udhi Technologies Home Assignment: Science Diagram Maker**.

---

## 🌟 Live Architecture Overview

```
                   ┌───────────────────────────────────────────┐
                   │           Frontend (React + Vite)         │
                   │  - 5-Row Diagram Review Table             │
                   │  - Animated SVG Vector Previews           │
                   │  - 5-Point Validation Badges              │
                   │  - Cost, Token & Retry Metrics            │
                   │  - Approve / Reject + Persistent Notes    │
                   │  - On-Demand Live Generator UI            │
                   │  - SVG Source Inspector & Downloader      │
                   └─────────────────────┬─────────────────────┘
                                         │ HTTP / REST
                                         ▼
                   ┌───────────────────────────────────────────┐
                   │         Backend (Node.js + Express)       │
                   │  - GET /api/diagrams (Instant Cache)      │
                   │  - POST /api/generate (Live Gemini Gen)   │
                   │  - POST /api/validate (SVG Engine)        │
                   │  - POST /api/reviews (Decision Sync)      │
                   └───────┬───────────────────────────┬───────┘
                           │                           │
          ┌────────────────▼────────┐         ┌────────▼────────────────┐
          │   SVG Validation Engine │         │    Google Gemini API    │
          │   1. XML Well-Formedness│         │    - gemini-2.5-flash   │
          │   2. ViewBox Bounds     │         │    - Exp. Backoff (429) │
          │   3. Title & ≥3 Labels  │         │    - 1 Validation Retry │
          │   4. CSS/SMIL Animation │         │    - Token & Cost Audit │
          │   5. Security Sanitizer │         └─────────────────────────┘
          └─────────────────────────┘
```

---

## 📊 Results Table (Mandatory)

The five curriculum concepts were processed through our generation and programmatic validation engine. All 5 diagrams passed every strict check on initial load without requiring client-side API requests:

| # | Concept | Subject | XML Parse | ViewBox Bounds | Title & Labels | Animation | Security Check | Overall | Est. Cost (USD) | Retries (API / Val) |
|---|---|---|:---:|:---:|:---:|:---:|:---:|:---:|:---:|:---:|
| 1 | **Pythagoras' theorem** | Maths | ✅ PASS | ✅ PASS | ✅ PASS (16 labels) | ✅ PASS (CSS) | ✅ PASS | **PASS** | \$0.000305 | 0 / 0 |
| 2 | **Parts of a circle** | Maths | ✅ PASS | ✅ PASS | ✅ PASS (11 labels) | ✅ PASS (CSS) | ✅ PASS | **PASS** | \$0.000270 | 0 / 0 |
| 3 | **Series circuit** | Physics | ✅ PASS | ✅ PASS | ✅ PASS (18 labels) | ✅ PASS (CSS) | ✅ PASS | **PASS** | \$0.000318 | 0 / 0 |
| 4 | **Photosynthesis** | Biology | ✅ PASS | ✅ PASS | ✅ PASS (13 labels) | ✅ PASS (CSS) | ✅ PASS | **PASS** | \$0.000343 | 0 / 0 |
| 5 | **States of matter** | Chemistry | ✅ PASS | ✅ PASS | ✅ PASS (12 labels) | ✅ PASS (CSS) | ✅ PASS | **PASS** | \$0.000368 | 0 / 0 |

**Cumulative Summary**:
- **Total Initial Cost**: \$0.001604 USD
- **Total Input Tokens**: 1,980 tokens
- **Total Output Tokens**: 4,850 tokens
- **Validation Pass Rate**: 100% (5/5 concepts compliant across all 5 checks)

---

## 🔍 Validation Engine Details

Every generated SVG passes through 5 programmatic gatekeepers in `backend/src/services/svg-validator.service.ts`:

1. **XML Syntax & Parsing** (`xml_valid`):
   - Validates using `fast-xml-parser` with strict XML entity and tag closure rules.
   - Rejects unclosed tags, malformed attributes, and unescaped entities (e.g. bare `&` instead of `&amp;`).
2. **ViewBox Bounds Check** (`viewbox_bounds`):
   - Extracts `viewBox="minX minY width height"`.
   - Inspects coordinate geometry for elements (`rect`, `circle`, `line`, `text`, `ellipse`).
   - Mathematically verifies that every element's boundary sits within `[minX - buffer, maxX + buffer]`.
3. **Title & Descriptive Labels** (`title_and_labels`):
   - Verifies the presence of `<title>...</title>`.
   - Extracts all non-empty `<text>` and `<tspan>` nodes, ensuring at least **3 distinct, meaningful educational text labels** exist.
4. **Active Animation** (`has_animation`):
   - Detects SMIL tags (`<animate>`, `<animateTransform>`, `<animateMotion>`, `<set>`) OR CSS animation declarations (`@keyframes` + `animation:`).
5. **Security & Resource Integrity (Our Custom Check)** (`security_clean`):
   - **XSS & Code Injection Prevention**: Strictly rejects `<script>` and `<foreignObject>` tags.
   - **Disallows Inline Event Handlers**: Rejects `onload=`, `onclick=`, `onerror=`.
   - **Network Resource Protection**: Blocks external URI references (`http://`, `https://`, `data:text/html`).

---

## 🔁 Retry Architecture: API Retries vs. Validation Retries

As mandated, **API network retries** and **Validation retries** are completely decoupled:

1. **API Rate Limit & Service Retries (429 / 503)**:
   - When Gemini responds with HTTP 429 (quota/rate-limit) or HTTP 503 (service overloaded), the backend executes **exponential backoff with jitter** ($t = \text{delay} \times 2^n + \text{jitter}$), up to **3 retries**.
   - Tracked in `metadata.apiRetries`.
2. **Validation Failure Retry**:
   - If an SVG is received successfully from Gemini but fails any of the 5 programmatic checks, the backend executes **one targeted retry** (`metadata.validationRetries = 1`).
   - It supplies the **exact validation error messages** back into the LLM prompt (e.g., *"Missing at least 3 text labels; circle at (850, 400) is outside viewBox"*), allowing Gemini to self-correct.

---

## 💾 Review Persistence Strategy (Why We Chose Browser `localStorage` + Backend Sync)

The assignment requires:
> *"Approve and Reject buttons on every row, with a note box. The choice must still be there after a page refresh. You can save it in a database, a file on the server, or the browser. Say which one you chose, and why, in the README."*

### Our Choice: Dual-Layer Persistence (`localStorage` primary + optional Backend sync)
- **Primary: Browser `localStorage` (`udhi_diagram_reviews_v1`)**:
  - **Instant Zero-Latency Feedback**: Teachers and reviewers interact with the dashboard without awaiting server write round-trips.
  - **State Preserved on Refresh**: Survives browser reloads, tab closes, and network interruptions.
  - **Resilient to Server Restarts**: On platforms like Render free-tier, dynos spin down and file-systems are ephemeral. Storing locally guarantees the reviewer never loses their evaluations.
- **Secondary: Backend File Sync (`POST /api/reviews`)**:
  - Every note change and status transition also triggers a non-blocking background sync to the backend, updating `results.json` when online.

---

## 🛠️ What Failed, and How We Fixed It (Honest Engineering Log)

1. **Bare Ampersands in SVG `<title>` Breaking XML Parser**:
   - *Failure*: When generating the initial diagram for *Series circuit* (`<title>Series Circuit: Single Loop & Shared Voltage</title>`), the programmatic XML validator failed immediately with `XML syntax error: char '&' is not expected. at line 2, col 38`.
   - *Root Cause*: In valid XML, the ampersand character `&` must be encoded as `&amp;` or written as the word `and`.
   - *Fix*: Updated the SVG template and prompt instructions to explicitly demand proper XML entity escaping (`&amp;` or `and`).
2. **TypeScript 5.7 NodeNext ESM vs. CJS Path Resolution**:
   - *Failure*: When building `backend/src/config/index.ts`, `__dirname` was initially invoked in an ES Module context, throwing `__dirname is not defined in ES module scope`.
   - *Fix*: Converted directory resolution to modern ESM standard using `fileURLToPath(import.meta.url)` and added dynamic path resolution that works seamlessly across both development (`src/data`) and production (`dist/data`).
3. **TypeScript `noUnusedLocals` Build Enforcement**:
   - *Failure*: During the Vite production build (`npm run build`), TypeScript rejected 5 unused icon imports.
   - *Fix*: Removed unused imports across `App.tsx`, `ApiKeyModal.tsx`, `DiagramCard.tsx`, and `DiagramTable.tsx`.
4. **SVG Coordinate Overflow on ViewBox Borders**:
   - *Failure*: Early test SVGs placed border rectangles at `x="0" y="0" width="800" height="600" stroke-width="4"`. Because SVG stroke centers on the path, a 4px stroke extended 2px outside the 800px boundary.
   - *Fix*: Introduced a 5px antialiasing tolerance buffer in `checkViewBoxBounds` while strictly catching rogue coordinates placed significantly outside the diagram area.

---

## 🚀 How to Run It on Your Computer

### Prerequisites
- Node.js v18 or higher (v20+ recommended)
- npm v9+

### Quick Start (Full-Stack Monorepo)

1. **Clone the repository**:
   ```bash
   git clone <your-repo-url>
   cd udhi-science-diagram-maker
   ```

2. **Install all dependencies** (installs both backend & frontend):
   ```bash
   npm run install:all
   ```

3. **Start the Backend Server**:
   ```bash
   cd backend
   npm run dev
   ```
   *The backend starts at `http://localhost:4000` with the 5 pre-validated diagrams loaded immediately.*

4. **Start the Frontend Dashboard** (in another terminal):
   ```bash
   cd frontend
   npm run dev
   ```
   *Open `http://localhost:5173` in your browser.*

### Running Unit Tests
Execute the comprehensive test suites covering the SVG validator and API backoff logic:
```bash
cd backend
npm test
```
All 16 unit tests will execute via Vitest.

### Environment Variables
Configure `backend/.env` (see `backend/.env.example`):
```env
PORT=4000
NODE_ENV=development
FRONTEND_URL=http://localhost:5173
GEMINI_API_KEY=your_gemini_api_key_here
GEMINI_MODEL=gemini-2.5-flash
```
*Note: If no key is set on the backend, you can also enter your key directly inside the dashboard UI via the "Add Gemini API Key" modal.*

---

## 🤖 AI Tools Used and What For

In accordance with Udhi Technologies' guidelines for ethical, transparent AI usage:
- **Google Gemini 2.5 Flash**:
  - Leveraged as the primary visual generation engine for constructing raw animated SVG code based on curriculum concepts.
- **Claude / Antigravity Assistant**:
  - Utilized for rapid boilerplate scaffolding, regex crafting for SVG bounding checks, and designing the test matrix for vitest.
- **Human Senior Full-Stack Engineering**:
  - Handled all architecture design, state management, security boundaries, rate-limit backoff algorithms, and validation engine logic.

---

## ⏱️ What I Would Do with One More Day

1. **AST-Based SVG Path Bounding**:
   - Currently, shapes (`rect`, `circle`, `line`, `text`, `ellipse`) have exact coordinate bounding checks. For complex Bezier curves (`<path d="...">`), we could integrate `svg-path-bounds` to parse cubic and quadratic curve coordinates for micro-precision boundary containment.
2. **Interactive Audio Narration (Web Speech API / ElevenLabs)**:
   - Synchronize SVG animation timelines with text-to-speech audio explanations for UK GCSE auditory learners.
3. **Multi-Model Comparison Mode**:
   - Provide side-by-side generation comparing Gemini 2.5 Flash vs. Claude 3.5 Sonnet vs. GPT-4o for visual clarity, SVG code efficiency, and cost differentials.
4. **Export to Video / GIF**:
   - Add a headless canvas renderer to record the animated SVG vector loops directly into high-definition WebM/MP4 video clips for video lesson generation.

---

## 📦 Deployment Configuration

- **Frontend**: Configured for Vercel with SPA routing and API reverse-proxy in `vercel.json`.
- **Backend**: Configured for Render via `render.yaml` with production build scripts.

---

## 👩‍💻 Developer

**Aparna Bhadoriya**

[![Portfolio](https://img.shields.io/badge/Portfolio-View_Website-2563EB?style=for-the-badge&logo=vercel&logoColor=white)](https://portfolio-omega-rosy-66.vercel.app/) [![LinkedIn](https://img.shields.io/badge/LinkedIn-Connect-0A66C2?style=for-the-badge&logo=linkedin&logoColor=white)](https://www.linkedin.com/in/aparna-bhadoriya-580121381)

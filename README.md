# 🏥 ClearDischarge — AI-Powered Hospital Discharge Summary Translator

**INFO 7375 ST: Prompt Engineering & AI — Final Project | Northeastern University**  
**Authors:** Rahul Reddy & Rohan Reddy

Turns clinical discharge paperwork into plain-language patient action plans — powered by **5 core generative AI components**.

## What It Does

1. **Extracts** every medication, restriction, red-flag symptom, and follow-up appointment from a discharge summary
2. **Cross-checks** the medication list against the live OpenFDA drug interaction database
3. **Retrieves** condition-specific patient education from a RAG knowledge base
4. **Generates** a plain-language Patient Action Plan verified at ≤ Grade 8 reading level
5. **Creates** a visual medication schedule showing what to take and when

---

## 5 Core Generative AI Components

### 1. Prompt Engineering
- Structured extraction prompts with JSON schema enforcement
- Patient education generation prompt with readability constraints
- Automatic retry/simplify loop when reading level exceeds Grade 8
- Edge case handling: missing fields, abbreviations, varied formatting

### 2. RAG (Retrieval-Augmented Generation)
- Patient education knowledge base built from trusted medical sources
- ChromaDB vector store with sentence-transformer embeddings (`all-MiniLM-L6-v2`)
- Paragraph-level document chunking with 100-char overlap
- **Confidence-weighted relevance display** — shows users which education sources were most relevant to their specific diagnosis, with relevance percentages
- Covers: heart failure, diabetes, pneumonia, surgery recovery, hypertension, medication safety

### 3. Multimodal Integration
- **Input:** OCR for scanned/image-based discharge PDFs (pytesseract)
- **Output:** Visual medication schedule (matplotlib) — color-coded daily pill chart
- Automatic detection of scanned vs. text-based PDFs with OCR fallback
- **OCR post-processing:** Confidence scoring per page, noise removal (artifact lines, garbage characters), and low-quality scan warnings
- **Printable PDF export** of medication schedule alongside PNG
- **Alarm time suggestions** — recommended phone reminder times based on medication frequency

### 4. Synthetic Data Generation
- Generates diverse discharge summaries via Groq API
- 8 diagnosis categories: cardiac, respiratory, surgical, endocrine, neurological, renal, GI, orthopedic
- 3 complexity tiers: simple (2-4 meds), moderate (4-7), complex (7-12)
- 5 formatting styles: standard, abbreviations, minimal, narrative, mixed
- Auto-generated ground truth annotations for recall evaluation
- **Component 4 → 5 loop:** Augmentation templates generate additional PASS/FAIL training examples for the readability classifier

### 5. Fine-Tuning
- DistilBERT binary classifier fine-tuned on medical text readability
- Detects jargon complexity that formula-based metrics (Flesch-Kincaid) miss
- **100 training examples** (40 curated + 60 template-augmented via Component 4 loop)
- **Full evaluation metrics:** Precision, recall, F1 score, and confusion matrix on held-out validation set
- Serves as a dual validation gate alongside Flesch-Kincaid scoring

---

## Project Structure

```
cleardischarge/
├── .env                  ← API key (never commit)
├── .env.example          ← Template
├── app.py                ← Streamlit web UI (all 5 components)
├── extractor.py          ← Stage 1: Clinical data extraction
├── drug_checker.py       ← Stage 2: FDA drug interaction checker
├── rag_knowledge.py      ← Component 2: RAG knowledge base
├── generator.py          ← Stage 3: Action plan generator (+ RAG + ML)
├── multimodal.py         ← Component 3: OCR + visual schedule
├── synthetic_gen.py      ← Component 4: Synthetic data generation
├── readability_model.py  ← Component 5: Fine-tuned readability classifier
├── evaluate.py           ← Evaluation harness
├── requirements.txt      ← Python dependencies
├── knowledge_base/       ← RAG source documents
│   ├── heart_failure.txt
│   ├── diabetes.txt
│   ├── pneumonia.txt
│   ├── surgery_recovery.txt
│   ├── hypertension.txt
│   └── medication_safety.txt
├── webpage/              ← Project showcase web page
│   ├── index.html        ← Single-page project showcase
│   └── ClearDischarge_Documentation.pdf  ← Full project documentation
│   └── Video_Project_11  ← Full project demo
└── README.md
```

---

## Setup

### 1. Clone / download the project
```bash
cd cleardischarge
```

### 2. Create virtual environment
```bash
# Standard (Python 3.11/3.12)
python -m venv venv

# Python 3.13 on Windows (known ensurepip hang — use this instead)
python -m venv venv --without-pip
.\venv\Scripts\Activate.ps1
.\venv\Scripts\python.exe -m ensurepip --upgrade

# Mac/Linux activation
source venv/bin/activate
```

### 3. Install dependencies
```bash
pip install -r requirements.txt

# If pip installed to system Python instead of venv (check with pip --version),
# force venv's Python explicitly:
.\venv\Scripts\python.exe -m pip install -r requirements.txt
```

### 4. Install Tesseract OCR (optional — for scanned PDF support)
- **Windows:** Download from https://github.com/UB-Mannheim/tesseract/wiki
- **Mac:** `brew install tesseract`
- **Linux:** `sudo apt install tesseract-ocr`

### 5. Add your API key
Create a `.env` file:
```
GROQ_API_KEY=your_groq_key_here
```

### 6. Build the RAG knowledge base (first time only)
```bash
python rag_knowledge.py
```

### 7. Fine-tune the readability model (optional)
```bash
python readability_model.py --train --epochs 10
```

---

## Running the App

```bash
streamlit run app.py
```
Opens at http://localhost:8501

- **Paste Text tab** — paste any discharge summary → click Analyze
- **Upload PDF tab** — upload a PDF (text-based or scanned) → click Analyze
- **Load Sample** — loads a built-in CHF discharge summary for testing
- **Clear Results** — clears the current analysis so you can switch input methods
- **Accessibility sidebar** — adjust text size and toggle high-contrast mode; changes apply instantly without re-running the analysis

### Session State Behavior

Results persist across UI interactions (toggling settings, downloading files, expanding sections). The analysis pipeline only re-runs when you click "Analyze My Discharge." Use "Clear Results" to reset before analyzing a different document.

---

## Running Individual Components

```bash
# Test extraction (Prompt Engineering)
python extractor.py

# Test drug interaction checker (FDA API)
python drug_checker.py

# Test RAG knowledge base
python rag_knowledge.py

# Test full pipeline (extraction + RAG + drug check + generation)
python generator.py

# Test visual schedule generation (Multimodal)
python multimodal.py

# Generate synthetic test cases
python synthetic_gen.py --n 10

# Fine-tune readability classifier
python readability_model.py --train

# Test readability classifier
python readability_model.py --test
```

---

## Running the Evaluation

### With built-in samples (no API needed for data)
```bash
python evaluate.py --sample
```

### With synthetic data (Component 4 — generates cases via Claude)
```bash
python evaluate.py --synthetic --n 10
```

### With MIMIC-III (after PhysioNet access)
```bash
python evaluate.py --n 10
```

### Sample Evaluation Output
```
═══════════════════════════════════════════════════════════════════
  CLEARDISCHARGE EVALUATION RESULTS  (n=5 cases)
═══════════════════════════════════════════════════════════════════

  Medication Recall        : 1.00  (target ≥ 0.95)  ✅
  Restriction Recall       : 0.843  (target ≥ 0.90)  ✅
  Red Flag Recall          : 0.960  (target ≥ 0.90)  ✅

  Hallucination Rate       : 0.000  (target = 0.00)   ✅
  Avg Reading Grade Level  : 5.6   (target ≤ 8.0)    ✅
  % Cases Meeting Grade    : 100%
  Avg Simplification Retries: 0.4
  Total Drug Interactions  : 4

  ── Component Usage ──
  RAG Context Used         : 100% of cases
  ML Readability Model     : 0% of cases
```

---

## Evaluation Metrics

| Metric | What It Measures | Target |
|--------|-----------------|--------|
| Medication Recall | % of real medications correctly extracted | ≥ 0.95 |
| Hallucination Rate | % of extracted drugs NOT in source text | 0% |
| Restriction Recall | % of restrictions correctly extracted | ≥ 0.90 |
| Red Flag Recall | % of ER symptoms correctly extracted | ≥ 0.90 |
| Reading Grade Level | Flesch-Kincaid grade of output | ≤ Grade 8 |

---

## Tech Stack

| Component | Technology |
|-----------|-----------|
| LLM | LLaMA 3.3 70B via Groq (free, fast inference) |
| Drug Safety Data | OpenFDA API (api.fda.gov) — free, U.S. government |
| RAG Vector Store | ChromaDB (persistent, local) |
| RAG Embeddings | sentence-transformers (all-MiniLM-L6-v2) |
| Readability (formula) | textstat (Flesch-Kincaid) |
| Readability (ML) | Fine-tuned DistilBERT classifier |
| OCR | pytesseract + Pillow |
| Visual Output | matplotlib |
| PDF Parsing | pypdf |
| UI | Streamlit |
| Evaluation Data | MIMIC-III (PhysioNet) + synthetic generation via Groq |

---

## Cost

- Groq API: Free tier available (generous rate limits)
- OpenFDA: Free
- Everything else: Free
- Total for development: $0 (all free tiers)

---

## Accessibility

ClearDischarge is designed for patients of all abilities:

- **Adjustable text size:** Normal, Large, and Extra Large options in the sidebar, with word-count guidance (e.g., "Best for long reports — 500+ words") so patients know which size fits their document
- **High-contrast mode:** Toggle overrides Streamlit's CSS variables to enforce pure black background with white text across the entire app, sidebar, headings, labels, and all custom UI elements
- **Dark-mode-friendly defaults:** All custom alert boxes (warnings, drug interactions, success messages) use dark backgrounds with light text to match Streamlit's default dark theme
- **ARIA labels:** Custom UI elements include descriptive `role` and `aria-label` attributes for screen reader compatibility
- **Reading level enforcement:** All output verified at ≤ Grade 8 via dual-gate check (Flesch-Kincaid + ML classifier)
- **User onboarding:** First-time guide explains each step in plain language
- **Printable outputs:** Medication schedule available as both PNG and PDF for patients who prefer paper
- **Alarm suggestions:** Recommended phone reminder times based on medication frequency

---

## UX Features

- **Session state persistence:** Analysis results survive sidebar toggles, downloads, and expander clicks — the pipeline only re-runs on explicit "Analyze" clicks
- **PDF caching:** Uploaded PDFs are extracted once and cached by filename; switching tabs or toggling settings no longer re-processes the document
- **Clear Results button:** Appears alongside results, allowing users to reset before analyzing a different document
- **Pipeline performance display:** Expandable per-stage timing breakdown shows extraction, drug check, RAG retrieval, generation, and visual schedule durations

---

## Pipeline Performance

The app displays per-stage timing after each analysis:

| Stage | Typical Time | Notes |
|-------|-------------|-------|
| Extraction | 1–2s | Single LLM call via Groq |
| Drug Check | 0.5–2s | FDA API calls per drug pair |
| RAG Retrieval | 0.1–0.5s | Local ChromaDB vector search |
| Generation | 1–3s | LLM call + possible retry |
| Visual Schedule | 0.1–0.3s | Local matplotlib render |
| **Total** | **3–8s** | Varies by medication count |

Timing breakdown is visible in the "Pipeline Performance" expander in the app.

---

## Project Web Page & Documentation

The `webpage/` folder contains the project showcase and PDF documentation:

### Viewing the Web Page
```bash
# Option 1: Open directly in browser
open webpage/index.html        # Mac
start webpage/index.html       # Windows

# Option 2: Serve locally (for proper relative links)
cd webpage
python -m http.server 8000
# Then visit http://localhost:8000
```

The web page includes a project overview, interactive app mockup, 5-component breakdown, architecture diagram, tech stack, features grid, and a PDF documentation download button.

### PDF Documentation

The full project documentation (`webpage/ClearDischarge_Documentation.pdf`) covers system architecture with a pipeline diagram, implementation details for all 5 components, performance metrics and evaluation results, challenges and solutions, future improvements, and ethical considerations.

---

## Ethical Considerations

- **Not a medical device:** This tool is for informational purposes only
- **No patient data stored:** All processing happens in-session
- **Hallucination detection:** Extraction is verified against source text
- **Readability verified:** Dual-gate check (formula + ML) ensures accessibility
- **Drug interactions sourced:** All alerts cite FDA database or clinical references
- **Bias awareness:** Knowledge base covers common conditions; expansion needed for rare diseases
- **Privacy:** No data is sent to external services beyond the Groq API and OpenFDA

---

## Disclaimer

This tool is for informational purposes only. It is not a substitute for advice from a licensed medical professional. Always follow the instructions given by your healthcare team.

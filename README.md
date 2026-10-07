# 1. Project Name

# VYOM+
## The AI Trust Layer for GST Invoices

**Don't just read an invoice. Know whether you can trust it.**

> **AI proposes. Rules verify. Evidence explains. Humans resolve only what remains uncertain.**

**Team Phantom** | Hacktober Fest — Open Source AI Hackathon

---
 
---

# 2. Problem Statement

Handwritten bills, phone photographs, and scanned GST invoices are easy to misread. A single incorrect GSTIN digit, tax amount, quantity, rate, or total can become an accounting error.

Traditional OCR answers one question:

> "What text is visible?"

```
Invoice
   ↓
 OCR
   ↓
Text
   ↓
Accounting System
```

The problem is that **correctly reading text does not mean the financial record is correct.** A value can look perfectly readable and still produce an incorrect invoice total, tax calculation, or accounting entry.

The expensive part of invoice automation is not reading characters. It is dealing with the consequences of being wrong.

| Traditional OCR | Gap |
| --- | --- |
| Reads text | Does not know if the text is right |
| Returns values | Gives no reason to trust them |
| Leaves correctness to the user | Manual rechecking of every field |

---

# 3. Project Overview

VYOM+ converts messy invoices into structured, accounting-ready data, then **verifies the extracted information before it reaches an accounting workflow.**

Instead of asking *"Can we extract this document?"*, VYOM+ asks:

> **"Which extracted values are actually safe to trust?"**

### READ → VERIFY → RECOVER → EXPLAIN

| Stage | What happens |
| --- | --- |
| **READ** | PaddleOCR and Qwen2.5-VL independently read the invoice. |
| **VERIFY** | Their results are cross-checked against deterministic GST and arithmetic rules. |
| **RECOVER** | When something fails, VYOM+ goes back to the relevant invoice region and performs a targeted re-read. |
| **EXPLAIN** | Every important field receives a status — **Verified, Warning, or Needs Review** — with the reason and available evidence. |

VYOM+ treats **uncertainty as a first-class output.** Instead of "I think this is correct," it says:

- Here is the value.
- Here is the evidence.
- Here are the checks it passed.
- Here is what remains uncertain.

---

# 4. Proposed Solution

VYOM+ is an evidence-backed trust layer that sits between messy financial documents and automated accounting.

```
                         Invoice
                            │
              ┌─────────────┴─────────────┐
              ↓                           ↓
         PaddleOCR                  Vision Model
              │                           │
              └─────────────┬─────────────┘
                            ↓
                     Cross-Check
                            ↓
                 GST + Math Validation
                            ↓
                     Anything wrong?
                      /             \
                    No              Yes
                    ↓                ↓
                 VERIFY       Targeted Re-read
                                      ↓
                               Still uncertain?
                                /           \
                              No             Yes
                              ↓               ↓
                           VERIFY       HUMAN REVIEW
```

### Traditional OCR vs VYOM+

| Traditional OCR | VYOM+ |
| --- | --- |
| Reads text | Uses two independent readers |
| Returns values | Cross-checks important values |
| Leaves correctness to the user | Validates GST and arithmetic |
| | Re-reads suspicious regions |
| | Shows evidence behind important decisions |
| | Escalates only unresolved fields to a human |

**Traditional OCR extracts. VYOM+ verifies.**

### What Makes VYOM+ Different

#### 1. Two Readers, One Verdict

- **PaddleOCR**: text recognition and spatial coordinates
- **Qwen2.5-VL**: visual understanding, layout, handwriting, and field extraction

The first reading from each system is independent. If they disagree on an important field, that disagreement becomes a risk signal.

#### 2. Rules Verify What AI Reads

AI is used for perception. Deterministic rules decide whether the extracted information is internally consistent. VYOM+ checks:

- GSTIN structure
- GSTIN state code
- GSTIN check digit
- PAN pattern inside GSTIN
- Quantity × rate
- Tax calculations
- CGST / SGST consistency
- IGST consistency where applicable
- Line-item totals
- Invoice total reconciliation
- Date validity

*The model reads. The rule engine verifies.*

#### 3. Progressive Verification

VYOM+ does not immediately send every uncertain field to a human.

```
AI reads invoice
      ↓
Validation fails
      ↓
Find relevant invoice region
      ↓
Crop the region
      ↓
Targeted re-read
      ↓
Validate again
      ↓
       ┌───────────────┐
       │               │
     Pass            Still fail
       ↓               ↓
    Verify        Human Review
```

This keeps human involvement selective rather than constant. *The system knows when it doesn't know.*

#### 4. Evidence Graph

VYOM+ does not only return a value. It can preserve the evidence behind the decision.

Instead of:

```json
{
  "gstin": "27ABCDE1234F1Z5",
  "total": 1180
}
```

VYOM+ can represent:

```json
{
  "gstin": {
    "value": "27ABCDE1234F1Z5",
    "status": "verified",
    "evidence": {
      "page": 1,
      "region": [420, 115, 720, 170],
      "ocr": "27ABCDE1234F1Z5",
      "vision": "27ABCDE1234F1Z5",
      "gst_validation": true
    }
  }
}
```

In the interface:

```
GSTIN
27ABCDE1234F1Z5

VERIFIED

✓ OCR and Vision agree
✓ GSTIN check digit valid
✓ State code valid

TOTAL
₹12,480

NEEDS REVIEW
✗ Calculated total: ₹12,380
✗ Invoice total:     ₹12,480
⚠ Difference:        ₹100
```

The evidence chain:

```
Invoice Region
      ↓
Extracted Value
      ↓
Independent Reading
      ↓
Validation
      ↓
Decision
```

VYOM+ doesn't ask users to blindly trust the AI. It shows them why a value was trusted, or why it wasn't.

#### 5. Invoice Health

Instead of an unexplained AI confidence score, VYOM+ can produce an **Invoice Health** report from measurable verification signals.

```
┌──────────────────────────────────────┐
│          VYOM+ INVOICE HEALTH        │
│                                      │
│              82 / 100                │
│                                      │
│  Extraction Reliability   █████████░ │
│  GST Validation           ██████████ │
│  Arithmetic Consistency   ████████░░ │
│  Cross-Reader Agreement   █████████░ │
│  Document Quality         ██████░░░░ │
│                                      │
│  ⚠ 2 fields need review              │
└──────────────────────────────────────┘
```

**Why?**

```
✓ GSTIN is structurally valid
✓ Readers agree on GSTIN
✓ Tax structure is consistent

⚠ Invoice image is blurry
⚠ Total does not reconcile
```

Signals used: reader agreement, GST validation, arithmetic validation, re-read agreement, document quality, and historical supplier consistency where available.

The score is an **explanation layer**, not an AI-generated guess.

#### 6. Invoice Risk Radar

VYOM+ can surface invoice inconsistencies that deserve investigation. **It does not claim to prove fraud.**

```
┌──────────────────────────────────────┐
│          INVOICE RISK RADAR          │
│                                      │
│              78 / 100              │
│                                      │
│   Total does not reconcile          │
│    Expected: ₹12,380                 │
│    Invoice:  ₹12,480                 │
│                                      │
│ ⚠ Tax structure is inconsistent     │
│                                      │
│ ⚠ Unusual discount detected          │
│                                      │
│ ✓ GSTIN is structurally valid        │
│                                      │
│ ⚠ Possible duplicate invoice number  │
└──────────────────────────────────────┘
```

Potential risk signals:

- GSTIN and invoice information inconsistencies
- Tax calculation mismatches
- Line-item and total mismatches
- Unexpected tax structure
- Unusual discounts
- Duplicate invoice numbers
- Supplier-pattern deviations

*VYOM+ doesn't accuse. It surfaces risk with evidence.*

#### 7. Invoice DNA

For repeat suppliers, VYOM+ can build a lightweight **Invoice DNA** from previously verified invoices.

```
             INVOICE DNA

Supplier Identity       ██████████
Layout Pattern          █████████░
Tax Pattern             ██████████
Document Quality        ███████░░░
Extraction Reliability  █████████░
```

When another invoice from the same supplier arrives:

```
SUPPLIER PATTERN MATCH: 94%

✓ Same GSTIN
✓ Similar invoice layout
✓ Similar tax structure

⚠ Invoice number pattern changed
⚠ Total is unusually high
```

This does not prove fraud. It provides historical context that helps identify invoices that deserve closer inspection.

```
Verified Invoices
       ↓
Supplier Profile
       ↓
Historical Patterns
       ↓
New Invoice
       ↓
Pattern Comparison
       ↓
Normal / Unusual
```

---

# 5. Objectives

1. **Extract** structured, accounting-ready data from handwritten, photographed, scanned, and digital GST invoices.
2. **Verify** every important field using two independent readers and deterministic GST and arithmetic rules.
3. **Recover** failed fields through targeted re-reading of the relevant invoice region.
4. **Explain** each decision with a clear status, reason, and supporting evidence.
5. **Escalate selectively**, sending only unresolved fields to a human reviewer.
6. **Protect privacy** by running locally by default, so sensitive invoice data stays on the user's machine.
7. **Measure trust**, not just accuracy: when VYOM+ says "Verified," how often is the field actually correct?

### The Three Possible Outcomes

Every important field ends in one of three states.

| Status | Meaning |
| --- | --- |
|   **Verified** | Passed applicable validation and reader agreement |
|  **Warning** | Passed hard checks but contains an uncertainty or anomaly |
|   **Needs Review** | Could not be established reliably after recovery |

This prevents the system from silently converting uncertainty into false certainty.

---

# 6. Target Users / Use Case

| User | Need |
| --- | --- |
| **Small Businesses** | Receive supplier invoices as photos, scans, or handwritten documents. |
| **Accountants** | Need structured data without manually rechecking every field. |
| **CA Firms** | Process invoices from multiple businesses and need faster review. |
| **Finance Teams** | Need reliable structured invoice data before it enters downstream accounting workflows. |

### Supported Inputs

- Handwritten invoices
- Phone photographs
- Scanned invoices
- Digital PDFs
- JPG
- PNG
- Spreadsheet-based invoice data

The main focus is **messy GST invoice verification**, not simply supporting as many file extensions as possible.

### Primary Use Case

A small business receives a blurry phone photo of a handwritten supplier invoice. VYOM+ reads it, verifies the GSTIN and tax math, flags the one total that does not reconcile, shows exactly why, and lets the accountant fix only that field.

---

# 7. Open-Source AI Technology Selected

| Technology | Role |
| --- | --- |
| **PaddleOCR** | Open-source OCR for text recognition and spatial coordinates |
| **Qwen2.5-VL** | Open-source vision-language model for visual understanding, layout, handwriting, and field extraction |
| **Ollama** | Open-source local runtime for running Qwen2.5-VL on the user's machine |

---

# 8. Why This Technology Was Selected

**PaddleOCR**
- Returns text together with bounding-box coordinates, which is essential for the Evidence Graph and for cropping regions during targeted re-reads.
- Works as a reader that is architecturally independent from the vision model.

**Qwen2.5-VL**
- Understands layout, handwriting, and document structure, which plain OCR struggles with.
- Can extract named fields directly from the image.
- Open-source and runnable locally.

**Ollama**
- Makes local model execution simple, which supports the privacy-first design.
- Keeps invoice data on the user's machine by default.

**Two independent readers**
- The two systems fail differently, so disagreement between them is a meaningful risk signal that a single model cannot provide.

**Open-source and local-first**
- Invoices contain sensitive business information. Local processing removes the need to send them to a third-party service.

---

# 9. AI's Role in the System

AI is used for **perception**. It is not trusted to decide what is correct.

| Responsibility | Owner |
| --- | --- |
| Reading text and coordinates | PaddleOCR |
| Understanding layout, handwriting, fields | Qwen2.5-VL |
| Targeted re-read of a cropped region | Qwen2.5-VL / PaddleOCR |
| GST and arithmetic correctness | Deterministic rule engine |
| Final status per field | Cross-check + rule engine |
| Unresolved decisions | Human reviewer |

The model reads. The rule engine verifies. The system **never invents a correction** just to make an invoice pass.

---

# 10. System Architecture

```
                    INVOICE
                       │
                       ↓
                 File Detection
                       │
          ┌────────────┴────────────┐
          ↓                         ↓
   Structured Input          PDF / Image Input
          │                         │
          │                  Image Preparation
          │                         │
          │             ┌───────────┴───────────┐
          │             ↓                       ↓
          │        PaddleOCR              Qwen2.5-VL
          │             │                       │
          │             └───────────┬───────────┘
          │                         ↓
          └──────────────────→ Cross-Check
                                  ↓
                         GST + Math Validation
                                  ↓
                           Failed Field?
                            /          \
                          No            Yes
                          ↓              ↓
                       Verify      Targeted Re-read
                                         ↓
                                  Validate Again
                                         ↓
                                  /             \
                               Pass             Fail
                                ↓                ↓
                             Verify        Human Review
                                │                │
                                └───────┬────────┘
                                        ↓
                              Evidence + Risk Report
                                        ↓
                              JSON / CSV / UI Result
```

### Privacy-First Architecture

The default architecture keeps invoice data on the user's machine and runs the vision model through Ollama.

```
Invoice
   ↓
Local Processing
   ├── OCR
   ├── Vision Model
   ├── Validation
   └── Evidence
   ↓
Local Result
```

Cloud GPU execution may be used as an **optional development/demo configuration** and is not required by the core architecture.

---

# 11. Component-Level Architecture

| Component | Responsibility |
| --- | --- |
| **File Router** | Detects the input type (structured data vs. PDF/image) and routes it. |
| **Image Preparation** | Renders PDF pages, then pre-processes images (e.g., deskew, denoise) with OpenCV / pypdfium2. |
| **OCR Reader** | PaddleOCR extracts text and bounding boxes. |
| **Vision Reader** | Qwen2.5-VL (via Ollama) extracts fields, layout, and handwriting. |
| **Cross-Checker** | Compares both readings field by field; disagreement becomes a risk signal. |
| **GST Validator** | GSTIN structure, state code, check digit, PAN pattern, date validity. |
| **Math Validator** | Quantity × rate, line totals, CGST / SGST / IGST consistency, total reconciliation. |
| **Recovery Engine** | Locates the relevant region, crops it, and runs a targeted re-read. |
| **Status Engine** | Assigns Verified / Warning / Needs Review to each field. |
| **Evidence Builder** | Records page, region, both readings, and validation results per field. |
| **Health and Risk Reporter** | Produces Invoice Health and Invoice Risk Radar from measurable signals. |
| **Output Layer** | Writes JSON / CSV and serves the Streamlit review UI. |

---

# 12. Data / Information Flow

1. **Upload**: the user uploads an invoice (image, PDF, or spreadsheet).
2. **Detect**: the file type is identified and routed.
3. **Prepare**: PDFs are rendered to images; images are pre-processed.
4. **Read**: PaddleOCR and Qwen2.5-VL each extract the invoice independently.
5. **Cross-check**: results are compared field by field.
6. **Validate**: GST and arithmetic rules run on the merged result.
7. **Recover**: failed fields are cropped and re-read, then validated again.
8. **Decide**: each field becomes Verified, Warning, or Needs Review.
9. **Explain**: evidence, health, and risk reports are generated.
10. **Output**: structured JSON / CSV and a side-by-side review UI.

```
Image / PDF  →  Readers  →  Cross-Check  →  Rules  →  Recovery  →  Status  →  Evidence  →  JSON / CSV / UI
```

---

# 13. Agentic Workflow

VYOM+ uses a bounded, rule-guided loop rather than free-form autonomy.

```
AI reads invoice
      ↓
Validation fails
      ↓
Find relevant invoice region
      ↓
Crop the region
      ↓
Targeted re-read
      ↓
Validate again
      ↓
Pass → Verified      Still fail → Human Review
```

| Step | Agent behavior |
| --- | --- |
| **Perceive** | Two readers extract the invoice. |
| **Check** | The rule engine detects which field failed. |
| **Act** | The system picks the relevant region and re-reads it. |
| **Re-check** | The same rules validate the new reading. |
| **Stop** | If evidence is still insufficient, it stops and asks a human. |

**The key behavior:** the system does not guess or fabricate a fix. *A safe system knows when to stop.*

---

# 14. Technology Stack

| Component | Technology | Purpose |
| --- | --- | --- |
| Language | Python | Core application |
| Interface | Streamlit | Interactive invoice review |
| OCR | PaddleOCR | Text + coordinates |
| Vision | Qwen2.5-VL | Layout, handwriting, field extraction |
| Local Runtime | Ollama | Local model execution |
| Image Processing | OpenCV | Pre-processing |
| PDF Processing | pdfplumber / pypdfium2 | PDF extraction and rendering |
| Data | pandas | Structured data handling |
| Validation | Python rules | GST + arithmetic verification |
| Output | JSON / CSV | Structured results |

### Trust Layer

The core differentiator is not a single model. It is the combination of:

**OCR + Vision + Cross-Check + Rules + Recovery + Evidence**

---

# 15. Expected Features

### Core

- Invoice upload
- File-type routing
- OCR extraction
- Vision-model extraction
- Cross-reader comparison
- GST validation
- Arithmetic validation
- Targeted field re-reading
- Field-level status (Verified / Warning / Needs Review)
- Structured JSON output
- CSV output
- Side-by-side invoice/result review

### Advanced

- Evidence Graph
- Invoice Health
- Invoice Risk Radar
- Benchmarking
- Invoice DNA

> **Implementation status:** Advanced features are marked according to their **actual implementation status at submission.**
>
> | Feature | Status |
> | --- | --- |
> | Evidence Graph | _To be updated at submission_ |
> | Invoice Health | _To be updated at submission_ |
> | Invoice Risk Radar | _To be updated at submission_ |
> | Benchmarking | _To be updated at submission_ |
> | Invoice DNA | _To be updated at submission_ |

### Example Result

A messy handwritten invoice might produce:

```
┌─────────────────────────────────────────────┐
│                 VYOM+ RESULT                │
├─────────────────────────────────────────────┤
│                                             │
│ Invoice No.       INV-0231                │
│ Date              14/09/2026              │
│ Seller GSTIN      27ABCDE...              │
│ Buyer GSTIN       27XYZ...                │
│                                             │
│ Item Amount       ₹10,000                 │
│ CGST              ₹900                   │
│ SGST              ₹900                  │
│                                             │
│ Total             ₹11,800                 │
│                                             │
│ ⚠ Total does not reconcile by ₹100         │
│                                             │
│        INVOICE HEALTH: 82/100               │
│        RISK LEVEL: HIGH                     │
│                                             │
└─────────────────────────────────────────────┘
```

The user can inspect the original invoice, view the evidence, trigger a targeted re-read, or correct the unresolved field.

---

# 16. Implementation Approach

1. **Input layer**: Streamlit upload with file-type detection and routing.
2. **Preparation**: render PDFs with pypdfium2 / pdfplumber; pre-process images with OpenCV.
3. **Dual extraction**: run PaddleOCR and Qwen2.5-VL (through Ollama) independently on the same invoice.
4. **Cross-check**: normalize both outputs into a common schema and compare field by field.
5. **Rule engine**: implement GST and arithmetic validators in plain Python.
6. **Recovery loop**: on failure, use PaddleOCR coordinates to crop the region, re-read it, and re-validate.
7. **Status and evidence**: assign field status and attach evidence (page, region, both readings, checks).
8. **Reporting**: compute Invoice Health and Risk Radar from measurable signals.
9. **Output and UI**: export JSON / CSV and show a side-by-side invoice and result view in Streamlit.
10. **Benchmark**: evaluate on a manually verified test set.

### Demo Flow

The demo is built around a difficult invoice, not a perfect digital PDF.

| Step | Action |
| --- | --- |
| **1. Upload** | Upload a handwritten or poor-quality invoice. |
| **2. Read** | Two reading systems independently extract the invoice. |
| **3. Verify** | GST and arithmetic rules validate the result. |
| **4. Detect** | A suspicious field is identified. |
| **5. Explain** | The UI shows exactly why the field is suspicious. |
| **6. Recover** | VYOM+ re-reads only the relevant region. |
| **7. Decide** | Enough evidence → **Verified**. Uncertainty remains → **Human Review**. |

### Benchmark Plan

VYOM+ is evaluated against a manually verified test set containing:

- Clean printed invoices
- Digital PDFs
- Poor photographs
- Skewed images
- Blurred images
- Handwritten invoices

Each field is compared against ground truth.

| Metric | Question it answers |
| --- | --- |
| **Field Accuracy** | How often are extracted values correct? |
| **Trust Accuracy** | When VYOM+ says a field is Verified, how often is it actually correct? |
| **Human Review Rate** | What percentage of fields require human attention? |
| **Recovery Rate** | How often does targeted re-reading resolve an initially failed field? |
| **Processing Time** | How long does the system take per invoice? |

These metrics show not just whether VYOM+ can read invoices, but whether it can **safely automate invoice verification.**

---

# 17. Expected Final Output

For every invoice, VYOM+ produces:

- **Structured JSON** with each field's value, status, and evidence
- **CSV** of accounting-ready data
- **Field-level status**:  Verified,  Warning,  Needs Review
- **Evidence** per important field (region, both readings, validation results)
- **Invoice Health** report *(where implemented)*
- **Invoice Risk Radar** report *(where implemented)*
- **Side-by-side review UI** showing the original invoice next to the extracted result

```
MESSY INVOICE
      ↓
AI EXTRACTION
      ↓
VERIFICATION
      ↓
EVIDENCE
      ↓
RISK ANALYSIS
      ↓
┌────────┴────────┐
↓                 ↓
TRUSTED        UNCERTAIN
↓                 ↓
ACCOUNTING     HUMAN
FLOW           REVIEW
```

---

# 18. Future Scope / Scalability

| Direction | Description |
| --- | --- |
| **Supplier Intelligence** | Build historical supplier profiles from verified invoices (Invoice DNA). |
| **Accounting Integration** | Send verified structured data into accounting workflows. |
| **Batch Processing** | Process large invoice collections with selective review. |
| **Multilingual Handwriting** | Improve support for regional Indian languages. |
| **Learning From Corrections** | Use reviewer corrections to improve repeat-supplier extraction. |
| **Related Documents** | Extend the trust layer to credit notes, purchase orders, and other financial documents. |

### The Bigger Vision

Today's invoice automation asks: *"Can we extract this document?"*

VYOM+ asks: **"Can we safely trust what we extracted?"**

The long-term goal is not another OCR engine. It is an evidence-backed trust layer between messy financial documents and automated accounting. Over time, VYOM+ can move from one-time invoice verification toward supplier intelligence.

---

# 19. Open-Source Dependencies / Components

| Dependency | Use |
| --- | --- |
| [PaddleOCR](https://github.com/PaddlePaddle/PaddleOCR) | OCR text and coordinates |
| [Qwen2.5-VL](https://github.com/QwenLM/Qwen2.5-VL) | Vision-language extraction |
| [Ollama](https://github.com/ollama/ollama) | Local model runtime |
| [Streamlit](https://github.com/streamlit/streamlit) | Review interface |
| [OpenCV](https://github.com/opencv/opencv) | Image pre-processing |
| [pdfplumber](https://github.com/jsvine/pdfplumber) | PDF text extraction |
| [pypdfium2](https://github.com/pypdfium2-team/pypdfium2) | PDF page rendering |
| [pandas](https://github.com/pandas-dev/pandas) | Structured data handling |
| Python standard library | GST and arithmetic rule engine |

---

# 20. Expected Challenges and Mitigation

VYOM+ does not claim that every invoice can be fully automated.

| Challenge | Mitigation |
| --- | --- |
| Extremely poor handwriting | Dual readers, targeted re-read, then human review instead of guessing |
| Severe blur | Image pre-processing, document-quality signal in Invoice Health, human review if unresolved |
| Missing fields | Mark as Needs Review; never fabricate values |
| Damaged documents | Region-level re-read; escalate if evidence is insufficient |
| Ambiguous tax information | Rule engine flags inconsistency; status set to Warning or Needs Review |
| Unusual layouts | Vision model handles layout; cross-check exposes disagreement |
| Model and OCR disagreement | Treated as a risk signal, not silently resolved |
| Local hardware limits | Optional cloud GPU configuration for development/demo |
| Sensitive business data | Local-first architecture by default |

In these cases, the correct behavior is not to guess. It is to request human review.

**A safe system knows when to stop.**

---

## Team

**Team Phantom**

- Aniruddha Chaudhary
- Aryan Waghchoure
- Divyanshu Chede
- Yash Lohiya

**Event:** Hacktober Fest — Open Source AI Hackathon

---

# VYOM+

**Don't just read an invoice. Know whether you can trust it.**

> **AI proposes. Rules verify. Evidence explains. Humans resolve only what remains uncertain.**

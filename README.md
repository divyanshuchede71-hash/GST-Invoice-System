# VYOM+

## 1. Project Name

**VYOM+: The AI Trust Layer for GST Invoices**

*Don't just read an invoice. Know whether you can trust it.*

| | |
| --- | --- |
| Team | Team Phantom |
| Members | Aniruddha Chaudhary, Aryan Waghchoure, Divyanshu Chede, Yash Lohiya |
| Event | Hacktober Fest: Open Source AI Hackathon |

---

## 2. Problem Statement

Small businesses and accounting teams in India deal with GST invoices that arrive in every possible form: handwritten bills, photographs taken on a phone, scans of varying quality, and digital PDFs. Reading these correctly is harder than it looks. One wrong digit in a GSTIN, or a misread tax amount, quantity, or total, can turn into an accounting error that someone has to find and fix later.

Most existing tools treat this as an OCR problem and answer a single question: what text is visible on the page?

```text
+---------+     +-----+     +------+     +-------------------+
| Invoice | --> | OCR | --> | Text | --> | Accounting System |
+---------+     +-----+     +------+     +-------------------+
```

The trouble is that reading the text correctly does not mean the financial record is correct. A value can be perfectly legible and still lead to a wrong invoice total, a wrong tax calculation, or a wrong accounting entry. Traditional OCR returns values and leaves it to the user to decide whether they are right, which usually means rechecking every field by hand.

The real cost of invoice automation is not in reading characters. It is in dealing with the consequences of being wrong.

| What goes wrong | Why it matters |
| --- | --- |
| A GSTIN digit is misread | The supplier cannot be identified correctly and input tax credit may be affected. |
| A tax amount is misread | The tax calculation no longer matches the invoice. |
| A quantity or rate is misread | The line-item amount and the total are wrong. |
| The total does not match the line items | The accounting entry is wrong from the start. |

---

## 3. Project Overview

VYOM+ turns messy invoices into structured, accounting-ready data, and then checks that data before it reaches any accounting workflow.

Where a typical OCR tool asks "what does this invoice say?", VYOM+ asks a different question: which of the extracted values are actually safe to trust?

The system works in four stages.

| Stage | What happens |
| --- | --- |
| Read | PaddleOCR and Qwen2.5-VL read the invoice independently of each other. |
| Verify | Their outputs are cross-checked and then tested against deterministic GST and arithmetic rules. |
| Recover | If a field fails, VYOM+ goes back to that part of the invoice and reads it again, focusing only on that region. |
| Explain | Every important field gets a status of Verified, Warning, or Needs Review, along with the reason and the evidence behind it. |

A core principle of the project is that uncertainty should be shown openly rather than hidden. Instead of saying "I think this is correct," VYOM+ tells the user the value, the evidence for it, the checks it passed, and what is still uncertain.

---

## 4. Proposed Solution

We propose a trust layer that sits between messy financial documents and automated accounting. It does not replace the OCR step. It builds on it by adding a second independent reader, a rule engine, a recovery step, and an evidence trail.

```text
Invoice
   |
   +-----------------+
   |                 |
   v                 v
PaddleOCR        Qwen2.5-VL
   |                 |
   +--------+--------+
            |
            v
       Cross-Check
            |
            v
 GST + Math Validation
            |
            v
    Any field failed?
      |           |
     No          Yes
      |           |
      v           v
  VERIFIED   Targeted Re-read
      ^           |
      |           v
      |    Still uncertain?
      |      |           |
      |     No          Yes
      |      |           |
      +------+           v
                   HUMAN REVIEW
```

### Traditional OCR compared with VYOM+

| Traditional OCR | VYOM+ |
| --- | --- |
| Reads text | Uses two independent readers |
| Returns values | Cross-checks the important ones |
| Leaves correctness to the user | Validates GST rules and arithmetic |
| No recovery | Re-reads suspicious regions |
| No explanation | Shows the evidence behind each decision |
| Everything goes to a human, or nothing does | Only unresolved fields go to a human |

### What makes the approach different

**Two readers, one verdict.** PaddleOCR provides text recognition and spatial coordinates. Qwen2.5-VL provides visual understanding, layout awareness, handwriting reading, and field extraction. The first reading from each is fully independent. When they disagree on an important field, we treat that disagreement as a risk signal.

**Rules verify what the AI reads.** The AI is used only for perception. Whether the extracted information is consistent is decided by deterministic rules. In short, the model reads and the rule engine verifies.

| Rule group | Checks performed |
| --- | --- |
| GSTIN | Structure, state code, check digit, and the PAN pattern inside the GSTIN |
| Line items | Quantity times rate, and line-item totals |
| Tax | Tax calculations, CGST and SGST consistency, and IGST consistency where applicable |
| Invoice totals | Reconciliation of the invoice total against the line items and taxes |
| Dates | Date validity |

**Progressive verification.** We do not send every uncertain field straight to a human. When a field fails validation, the system finds the relevant region, crops it, reads it again, and validates the result. Only if the field still fails does it go to human review. This keeps human involvement selective. The full loop is shown in the diagram above and in Section 13.

**Evidence graph.** VYOM+ does not just return a value. For each important field it keeps the evidence behind the decision.

| Evidence kept for each field | Description |
| --- | --- |
| Value | The final value presented to the user |
| Status | Verified, Warning, or Needs Review |
| Page and region | Where on the invoice the value was found |
| OCR reading | What PaddleOCR read in that region |
| Vision reading | What Qwen2.5-VL read in that region |
| Validation results | Which GST and arithmetic checks passed or failed |

The chain of evidence runs from the invoice region to the final decision.

```text
Invoice Region
      |
      v
Extracted Value
      |
      v
Independent Reading
      |
      v
Validation
      |
      v
Decision
```

In the interface, this appears as a short explanation next to each field. For example:

| Field | Value | Status | Why |
| --- | --- | --- | --- |
| GSTIN | 27ABCDE1234F1Z5 | Verified | OCR and vision readings agree. The check digit is valid. The state code is valid. |
| Total | ₹12,480 | Needs Review | Calculated total is ₹12,380 but the invoice total is ₹12,480, a difference of ₹100. |

The user is not asked to trust the AI blindly. They can see why a value was trusted, or why it was not.

**Invoice health.** Rather than showing an unexplained confidence number, VYOM+ produces an Invoice Health report built from signals we can actually measure: agreement between the readers, GST validation, arithmetic validation, agreement after re-reading, document quality, and supplier history where it is available. The score explains the result. It is not an AI guess.

| Signal | Example score |
| --- | --- |
| Extraction reliability | 9 / 10 |
| GST validation | 10 / 10 |
| Arithmetic consistency | 8 / 10 |
| Cross-reader agreement | 9 / 10 |
| Document quality | 6 / 10 |
| **Overall invoice health** | **82 / 100** |

The report also lists the reasons in plain language. In this example, the GSTIN is structurally valid, both readers agree on it, and the tax structure is consistent, but the image is blurry and the total does not reconcile, so two fields need review.

**Invoice risk radar.** VYOM+ points out inconsistencies on an invoice that deserve a closer look. It does not claim to prove fraud. Every signal comes with its evidence.

| Risk signal | Example finding | Severity |
| --- | --- | --- |
| Total mismatch | Expected ₹12,380 but the invoice says ₹12,480 | High |
| Tax structure | Tax structure is inconsistent with the supplier and buyer states | Medium |
| Discount | Unusual discount detected | Medium |
| Invoice number | Possible duplicate invoice number | Medium |
| GSTIN | GSTIN is structurally valid | Pass |

Other signals include line items that do not add up to the total and deviations from a supplier's usual pattern.

**Invoice DNA.** For suppliers who send invoices repeatedly, VYOM+ builds a lightweight profile from earlier verified invoices. When a new invoice arrives, it is compared against that profile. A change in pattern does not prove anything, but it gives useful context about which invoices deserve closer inspection.

```text
Verified Invoices
       |
       v
Supplier Profile
       |
       v
Historical Patterns
       |
       v
New Invoice
       |
       v
Pattern Comparison
       |
       v
Normal / Unusual
```

| Profile element | What it captures |
| --- | --- |
| Supplier identity | GSTIN and supplier details |
| Layout pattern | How the invoice is usually laid out |
| Tax pattern | The usual tax structure |
| Invoice number pattern | How invoice numbers are normally formed |
| Document quality | Typical quality of this supplier's invoices |
| Extraction reliability | How reliably this supplier's invoices have been read before |

---

## 5. Objectives

1. Extract structured, accounting-ready data from handwritten, photographed, scanned, and digital GST invoices.
2. Verify every important field using two independent readers and deterministic GST and arithmetic rules.
3. Recover from failed validations by re-reading only the relevant region of the invoice.
4. Explain each decision with a clear status, a reason, and the available evidence.
5. Involve humans only where it is needed, by escalating just the fields that remain uncertain.
6. Keep invoice data private by running locally by default.
7. Measure trust, not only accuracy. When VYOM+ marks a field as Verified, we want to know how often that field is actually correct.

Every important field ends in one of three states.

| Status | Meaning |
| --- | --- |
| Verified | Passed the applicable validation and reader agreement. |
| Warning | Passed the hard checks, but contains an uncertainty or anomaly. |
| Needs Review | Could not be established reliably, even after recovery. |

This stops the system from quietly turning uncertainty into false certainty.

---

## 6. Target Users / Use Case

| User | What they need |
| --- | --- |
| Small businesses | Supplier invoices often arrive as photos, scans, or handwritten documents, and there is no easy way to check them. |
| Accountants | Structured data without having to recheck every field manually. |
| CA firms | Faster, more focused review across invoices from many clients. |
| Finance teams | Reliable structured invoice data before it enters downstream accounting workflows. |

### Supported inputs

| Input type | Notes |
| --- | --- |
| Handwritten invoices | A main focus of the project |
| Phone photographs | Includes skewed and uneven lighting |
| Scanned invoices | Varying scan quality |
| Digital PDFs | Text-based and image-based |
| JPG and PNG | Common image formats |
| Spreadsheet-based invoice data | Structured input that skips the image steps |

Our focus is on messy GST invoice verification rather than on supporting as many file types as possible.

### Typical use case

A small business receives a blurry phone photo of a handwritten supplier invoice. VYOM+ reads it with two independent readers, checks the GSTIN and the tax arithmetic, and notices that the total does not match the sum of the line items. It shows exactly why, tries a targeted re-read of the total, and if the problem remains, hands only that field to the accountant to correct. The rest of the invoice does not need to be rechecked.

---

## 7. Open-Source AI Technology Selected

| Technology | Role in VYOM+ |
| --- | --- |
| PaddleOCR | Open-source OCR engine for text recognition and spatial coordinates |
| Qwen2.5-VL | Open-source vision-language model for layout understanding, handwriting, and field extraction |
| Ollama | Open-source runtime used to run Qwen2.5-VL locally |

---

## 8. Why This Technology Was Selected

| Choice | Reason |
| --- | --- |
| PaddleOCR | It returns text together with bounding-box coordinates. The coordinates let us crop the exact region of an invoice for a targeted re-read and point to the evidence for a value in the interface. It is also a reader that is genuinely separate from the vision model. |
| Qwen2.5-VL | It understands the page as a whole, not only as lines of text. It copes better with layout, handwriting, and invoice structure, and it can extract named fields directly from the image. It is open source and can run on a local machine. |
| Ollama | It makes it simple to run the vision model locally, which supports our privacy-first approach. |
| Two readers | Two different systems tend to fail in different ways, so disagreement between them tells us something that a single model cannot. |
| Open source and local-first | Invoices contain sensitive business information. Running locally means that information does not need to be sent to an outside service. |

---

## 9. AI's Role in the System

In VYOM+, AI is used for perception, not for judgment. It reads the invoice, but it is not trusted to decide whether what it read is correct.

| Responsibility | Handled by |
| --- | --- |
| Reading text and coordinates | PaddleOCR |
| Understanding layout, handwriting, and fields | Qwen2.5-VL |
| Re-reading a cropped region | Qwen2.5-VL and PaddleOCR |
| Checking GST and arithmetic correctness | Deterministic rule engine |
| Deciding the status of each field | Cross-check together with the rule engine |
| Resolving what remains uncertain | A human reviewer |

The system never invents a correction just to make an invoice pass. If the evidence is not enough, it says so.

---

## 10. System Architecture

The overall flow, from upload to final output, is shown below.

```text
                         INVOICE
                            |
                      File Detection
                            |
              +-------------+-------------+
              |                           |
       Structured Input           PDF / Image Input
              |                           |
              |                   Image Preparation
              |                           |
              |               +-----------+-----------+
              |               |                       |
              |           PaddleOCR              Qwen2.5-VL
              |               |                       |
              |               +-----------+-----------+
              |                           |
              +-------------------->  Cross-Check
                                          |
                                 GST + Math Validation
                                          |
                                    Failed field?
                                     /         \
                                   No          Yes
                                   |            |
                                Verify    Targeted Re-read
                                                |
                                         Validate again
                                                |
                                          /           \
                                       Pass           Fail
                                        |               |
                                     Verify       Human Review
                                        |               |
                                        +-------+-------+
                                                |
                                     Evidence + Risk Report
                                                |
                                       JSON / CSV / UI Result
```

### Privacy-first design

By default, everything runs on the user's own machine: OCR, the vision model (through Ollama), validation, and evidence generation. The invoice does not leave the machine.

```text
Invoice
   |
   v
+------------------------------------------+
| Local processing on the user's machine   |
|                                          |
|   PaddleOCR                              |
|   Qwen2.5-VL (through Ollama)            |
|   Validation                             |
|   Evidence                               |
+------------------------------------------+
   |
   v
Local Result
```

Running on a cloud GPU is an optional configuration that we may use for development and demos. It is not required by the core architecture.

---

## 11. Component-Level Architecture

```text
INPUT LAYER
  File router
       |
       v
  Image preparation
       |
       v
READING LAYER
  PaddleOCR (OCR reader)        Qwen2.5-VL (vision reader)
       |                                  |
       +--------------+---------+---------+
                      |
                      v
VERIFICATION LAYER
  Cross-checker
       |
       v
  GST validator + Math validator
       |
       v
  Status engine ----------------------------+
       |                                    |
       | all fields decided                 | failed field
       v                                    v
OUTPUT LAYER                          RECOVERY LAYER
  Evidence builder                    Recovery engine
       |                              (crops the region, re-reads it,
       v                               then sends it back to the
  Health and risk reporter             cross-checker)
       |
       v
  Streamlit UI, JSON and CSV
```

| Component | What it does |
| --- | --- |
| File router | Detects the input type (structured data, or PDF and image) and sends it down the right path. |
| Image preparation | Renders PDF pages and pre-processes images using OpenCV and pypdfium2. |
| OCR reader | Uses PaddleOCR to extract text and bounding boxes. |
| Vision reader | Uses Qwen2.5-VL through Ollama to extract fields, layout, and handwriting. |
| Cross-checker | Compares the two readings field by field. A disagreement is recorded as a risk signal. |
| GST validator | Checks GSTIN structure, state code, check digit, the PAN pattern, and date validity. |
| Math validator | Checks quantity times rate, line totals, CGST, SGST, and IGST consistency, and total reconciliation. |
| Status engine | Assigns Verified, Warning, or Needs Review to each field. |
| Recovery engine | Finds the relevant region, crops it, and runs a targeted re-read. The result goes back through the cross-checker. |
| Evidence builder | Records the page, region, both readings, and the validation results for each field. |
| Health and risk reporter | Builds the Invoice Health and Invoice Risk Radar reports from measurable signals. |
| Output layer | Produces the Streamlit review interface and the JSON and CSV results. |

---

## 12. Data / Information Flow

```text
User uploads invoice
        |
        v
Streamlit UI: detect file type and route it
        |
        v
Preparation: render PDF pages and clean images
        |
        +---------------------+
        |                     |
        v                     v
   PaddleOCR             Qwen2.5-VL
 text + coordinates     fields + layout
        |                     |
        +----------+----------+
                   |
                   v
Rule engine: cross-check, GST and math validation
                   |
                   v
      Any field failed?  --- Yes --->  Crop the region, re-read it with
                   |                   both readers, validate again
                   No                  |
                   |                   |
                   +<------------------+
                   |
                   v
Field statuses and evidence
                   |
                   v
Side-by-side review, JSON and CSV
```

| Step | What happens |
| --- | --- |
| 1. Upload | The user uploads an invoice as an image, a PDF, or a spreadsheet. |
| 2. Detection | The file type is identified and the file is routed accordingly. |
| 3. Preparation | PDFs are rendered to images, and images are pre-processed. |
| 4. Reading | PaddleOCR and Qwen2.5-VL each extract the invoice on their own. |
| 5. Cross-check | The two results are compared field by field. |
| 6. Validation | GST and arithmetic rules are applied to the combined result. |
| 7. Recovery | Fields that failed are cropped and re-read, then validated again. |
| 8. Decision | Each field is marked Verified, Warning, or Needs Review. |
| 9. Explanation | Evidence, health, and risk reports are generated. |
| 10. Output | The result is available as JSON, CSV, and a side-by-side review screen. |

---

## 13. Agentic Workflow

VYOM+ includes a small, controlled loop that behaves in an agent-like way, but it is deliberately bounded. It is guided by the rule engine rather than left to act freely.

```text
Perceive:  two readers extract the invoice
        |
        v
Check:     rule engine finds the failed field
        |
        v
Act:       crop the region and re-read it
        |
        v
Re-check:  validate the new reading
        |
        v
Enough evidence?
   |          |
  Yes         No
   |          |
   v          v
Verified   Stop and ask a human
```

| Step | What the system does |
| --- | --- |
| Perceive | Two readers extract the invoice. |
| Check | The rule engine identifies which field failed. |
| Act | The system picks the relevant region of the invoice and re-reads it. |
| Re-check | The same rules validate the new reading. |
| Stop | If the evidence is still not sufficient, the system stops and asks a human. |

The important part is the last step. The system does not guess and does not make up a fix. It knows when to stop.

---

## 14. Technology Stack

| Component | Technology | Purpose |
| --- | --- | --- |
| Language | Python | Core application |
| Interface | Streamlit | Interactive invoice review |
| OCR | PaddleOCR | Text and coordinates |
| Vision | Qwen2.5-VL | Layout, handwriting, and field extraction |
| Local runtime | Ollama | Running models locally |
| Image processing | OpenCV | Pre-processing |
| PDF processing | pdfplumber, pypdfium2 | PDF extraction and rendering |
| Data handling | pandas | Structured data |
| Validation | Python rules | GST and arithmetic verification |
| Output | JSON, CSV | Structured results |

The main strength of the project is not any single model. It is the combination of OCR, vision, cross-checking, rules, recovery, and evidence working together.

---

## 15. Expected Features

| Feature | Description | Priority |
| --- | --- | --- |
| Invoice upload | Upload images, PDFs, and spreadsheets | Core |
| File-type routing | Send each file down the right path | Core |
| OCR extraction | Text and coordinates from PaddleOCR | Core |
| Vision-model extraction | Fields, layout, and handwriting from Qwen2.5-VL | Core |
| Cross-reader comparison | Field-by-field comparison of the two readings | Core |
| GST validation | GSTIN structure, state code, check digit, PAN pattern, dates | Core |
| Arithmetic validation | Line items, taxes, and total reconciliation | Core |
| Targeted field re-reading | Crop and re-read only the failed region | Core |
| Field-level status | Verified, Warning, or Needs Review for each field | Core |
| JSON and CSV output | Structured, accounting-ready results | Core |
| Side-by-side review | Original invoice next to the extracted result | Core |
| Evidence Graph | Evidence kept for each important field | Additional |
| Invoice Health | Explained reliability score | Additional |
| Invoice Risk Radar | Inconsistencies surfaced with evidence | Additional |
| Benchmarking | Evaluation on a manually verified test set | Additional |
| Invoice DNA | Supplier pattern profiles from verified invoices | Additional |

The additional features are part of the proposed design. The core features come first in the build order, and the additional ones follow once the core pipeline is working.

### Example result

For a messy handwritten invoice, the output might look like this.

| Field | Value | Status |
| --- | --- | --- |
| Invoice No. | INV-0231 | Verified |
| Date | 14/09/2026 | Verified |
| Seller GSTIN | 27ABCDE... | Verified |
| Buyer GSTIN | 27XYZ... | Verified |
| Item amount | ₹10,000 | Verified |
| CGST | ₹900 | Verified |
| SGST | ₹900 | Verified |
| Total | ₹11,800 | Needs Review |

The total does not reconcile and is off by ₹100. The invoice health is 82 out of 100 and the risk level is High. From here, the user can look at the original invoice, view the evidence, trigger a targeted re-read, or correct the unresolved field directly.

---

## 16. Implementation Approach

| Phase | What we plan to do | Main tools |
| --- | --- | --- |
| 1. Input layer | Build the upload screen with file-type detection and routing. | Streamlit |
| 2. Preparation | Render PDFs and pre-process images. | pypdfium2, pdfplumber, OpenCV |
| 3. Dual extraction | Run both readers independently on the same invoice. | PaddleOCR, Qwen2.5-VL, Ollama |
| 4. Cross-check | Convert both outputs to a common schema and compare field by field. | Python, pandas |
| 5. Rule engine | Write the GST and arithmetic validators. | Python |
| 6. Recovery loop | Use PaddleOCR coordinates to crop failed regions, re-read, and validate again. | OpenCV, PaddleOCR, Qwen2.5-VL |
| 7. Status and evidence | Assign a status to each field and attach its evidence. | Python |
| 8. Reporting | Compute Invoice Health and Risk Radar from measurable signals. | Python, pandas |
| 9. Output and interface | Export JSON and CSV and show results beside the invoice. | Streamlit, pandas |
| 10. Benchmarking | Evaluate the system on a manually verified test set. | Python, pandas |

### Demo flow

The demo is planned around a difficult invoice rather than a clean digital PDF.

```text
1. Upload a poor-quality invoice
        |
        v
2. Two readers extract it independently
        |
        v
3. Rules verify the result
        |
        v
4. A suspicious field is detected
        |
        v
5. The interface explains why
        |
        v
6. Targeted re-read of that region
        |
        v
7. Enough evidence?
     |          |
    Yes         No
     |          |
     v          v
 Verified   Human Review
```

The key moment in the demo is that the system does not invent a correction just to make the invoice pass. It recognizes when it does not have enough evidence.

### Benchmark plan

VYOM+ will be evaluated against a manually verified test set that includes clean printed invoices, digital PDFs, poor photographs, skewed images, blurred images, and handwritten invoices. Each extracted field is compared against ground truth.

| Metric | What it tells us |
| --- | --- |
| Field accuracy | How often the extracted values are correct. |
| Trust accuracy | When VYOM+ says a field is Verified, how often it is actually correct. |
| Human review rate | The share of fields that need human attention. |
| Recovery rate | How often a targeted re-read fixes a field that initially failed. |
| Processing time | How long one invoice takes. |

Together, these show not only whether VYOM+ can read invoices, but whether it can do so safely.

---

## 17. Expected Final Output

For each invoice, VYOM+ is expected to produce the following.

| Output | Description |
| --- | --- |
| Structured JSON | Each field's value, status, and evidence |
| CSV file | Accounting-ready data |
| Field-level status | Verified, Warning, or Needs Review |
| Evidence | Region, both readings, and validation results for each important field |
| Invoice Health report | Explained reliability score, where implemented |
| Invoice Risk Radar report | Inconsistencies with evidence, where implemented |
| Side-by-side review screen | Original invoice next to the extracted result |

```text
Messy Invoice
      |
      v
AI Extraction
      |
      v
Verification
      |
      v
Evidence
      |
      v
Risk Analysis
      |
   +--+-----------+
   |              |
   v              v
Trusted       Uncertain
   |              |
   v              v
Accounting Flow  Human Review
```

---

## 18. Future Scope / Scalability

| Direction | Description |
| --- | --- |
| Supplier intelligence | Build historical profiles of suppliers from verified invoices, extending the Invoice DNA idea. |
| Accounting integration | Send verified structured data directly into accounting workflows. |
| Batch processing | Handle large collections of invoices, with review limited to the fields that need it. |
| Multilingual handwriting | Improve support for regional Indian languages. |
| Learning from corrections | Use reviewer corrections to improve extraction for repeat suppliers. |
| Related documents | Extend the trust layer to credit notes, purchase orders, and other financial documents. |

The longer-term aim is not to build another OCR engine. It is to provide an evidence-backed trust layer between messy financial documents and automated accounting. Most invoice automation today asks whether a document can be extracted. We want to answer a more useful question: can we safely trust what we extracted?

---

## 19. Open-Source Dependencies / Components

| Dependency | Use in the project |
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

## 20. Expected Challenges and Mitigation

VYOM+ does not claim that every invoice can be fully automated. Some will stay difficult, and for those the right behavior is to ask for human review instead of guessing.

| Challenge | How we plan to handle it |
| --- | --- |
| Extremely poor handwriting | Use both readers, try a targeted re-read, then send to human review if still unclear. |
| Severe blur | Pre-process the image, reflect document quality in the Invoice Health report, and escalate if unresolved. |
| Missing fields | Mark the field as Needs Review and never fill in a made-up value. |
| Damaged documents | Re-read at the region level, and escalate if the evidence is not enough. |
| Ambiguous tax information | Let the rule engine flag the inconsistency and set the status to Warning or Needs Review. |
| Unusual layouts | Rely on the vision model for layout, and use the cross-check to expose disagreement. |
| Disagreement between OCR and the vision model | Treat it as a risk signal and do not resolve it silently. |
| Limited local hardware | Offer an optional cloud GPU configuration for development and demos. |
| Sensitive business data | Run locally by default so invoices stay on the user's machine. |

A safe system knows when to stop.

---

**VYOM+**
*Don't just read an invoice. Know whether you can trust it.*

AI proposes. Rules verify. Evidence explains. Humans resolve only what remains uncertain.

# VYOM+

## 1. Project Name

**VYOM+: The AI Trust Layer for GST Invoices**

*Don't just read an invoice. Know whether you can trust it.*

Team Phantom
Aniruddha Chaudhary, Aryan Waghchoure, Divyanshu Chede, Yash Lohiya

Built for the Hacktober Fest Open Source AI Hackathon.

---

## 2. Problem Statement

Small businesses and accounting teams in India deal with GST invoices that arrive in every possible form: handwritten bills, photographs taken on a phone, scans of varying quality, and digital PDFs. Reading these correctly is harder than it looks. One wrong digit in a GSTIN, or a misread tax amount, quantity, or total, can turn into an accounting error that someone has to find and fix later.

Most existing tools approach this as an OCR problem. They answer a single question: what text is visible on the page?

```
Invoice  ->  OCR  ->  Text  ->  Accounting System
```

The trouble is that reading the text correctly does not mean the financial record is correct. A value can be perfectly legible and still lead to a wrong invoice total, a wrong tax calculation, or a wrong accounting entry. Traditional OCR returns values and leaves it to the user to decide whether they are right, which usually means rechecking every field by hand.

The real cost of invoice automation is not in reading characters. It is in dealing with the consequences of being wrong.

---

## 3. Project Overview

VYOM+ turns messy invoices into structured, accounting-ready data, and then checks that data before it reaches any accounting workflow.

Where a typical OCR tool asks "what does this invoice say?", VYOM+ asks a different question: which of the extracted values are actually safe to trust?

The system works in four stages:

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

```
                         Invoice
                            |
              +-------------+-------------+
              |                           |
          PaddleOCR                  Vision Model
              |                           |
              +-------------+-------------+
                            |
                       Cross-Check
                            |
                   GST + Math Validation
                            |
                      Anything wrong?
                       /           \
                     No            Yes
                     |              |
                  Verify     Targeted Re-read
                                    |
                            Still uncertain?
                              /          \
                            No           Yes
                            |             |
                         Verify      Human Review
```

The comparison with a traditional OCR pipeline looks like this:

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

**Rules verify what the AI reads.** The AI is used only for perception. Whether the extracted information is consistent is decided by deterministic rules. These cover GSTIN structure, state code, check digit, and the PAN pattern inside the GSTIN, as well as quantity times rate, tax calculations, CGST and SGST consistency, IGST consistency where applicable, line-item totals, invoice total reconciliation, and date validity. In short, the model reads and the rule engine verifies.

**Progressive verification.** We do not send every uncertain field straight to a human. When a field fails validation, the system finds the relevant region, crops it, reads it again, and validates the result. Only if the field still fails does it go to human review. This keeps human involvement selective.

```
AI reads invoice
      |
Validation fails
      |
Find relevant invoice region
      |
Crop the region
      |
Targeted re-read
      |
Validate again
      |
  +---+-----------+
  |               |
 Pass          Still fails
  |               |
Verify       Human Review
```

**Evidence graph.** VYOM+ does not just return a value. It can keep the evidence behind each decision. A plain extractor would return something like this:

```json
{
  "gstin": "27ABCDE1234F1Z5",
  "total": 1180
}
```

VYOM+ can instead represent the same field along with its proof:

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

In the interface, this appears as a short explanation next to each field:

```
GSTIN: 27ABCDE1234F1Z5
Status: Verified
  - OCR and vision readings agree
  - GSTIN check digit is valid
  - State code is valid

Total: Rs. 12,480
Status: Needs Review
  - Calculated total:  Rs. 12,380
  - Invoice total:     Rs. 12,480
  - Difference:        Rs. 100
```

The chain of evidence runs from the invoice region, to the extracted value, to the independent reading, to validation, and finally to the decision. The user is not asked to trust the AI blindly. They can see why a value was trusted, or why it was not.

**Invoice health.** Rather than showing an unexplained confidence number, VYOM+ can produce an Invoice Health report built from signals we can actually measure: agreement between the readers, GST validation, arithmetic validation, agreement after re-reading, document quality, and supplier history where it is available. The score is meant to explain the result, not to be an AI guess.

```
VYOM+ INVOICE HEALTH: 82 / 100

Extraction Reliability    9/10
GST Validation            10/10
Arithmetic Consistency    8/10
Cross-Reader Agreement    9/10
Document Quality          6/10

2 fields need review.

Why:
  - GSTIN is structurally valid
  - Both readers agree on the GSTIN
  - Tax structure is consistent
  - The invoice image is blurry
  - The total does not reconcile
```

**Invoice risk radar.** VYOM+ can point out inconsistencies on an invoice that deserve a closer look. It does not claim to prove fraud. It surfaces signals such as tax calculation mismatches, line items that do not add up to the total, an unexpected tax structure, unusual discounts, duplicate invoice numbers, and deviations from a supplier's usual pattern, always with the evidence attached.

**Invoice DNA.** For suppliers who send invoices repeatedly, VYOM+ can build a lightweight profile from earlier verified invoices. When a new invoice arrives, it is compared against that profile, covering things like the GSTIN, the layout, the tax pattern, and the invoice number format. A change in pattern does not prove anything, but it gives useful context about which invoices deserve closer inspection.

```
Verified invoices -> Supplier profile -> Historical patterns
        -> New invoice -> Pattern comparison -> Normal / Unusual
```

---

## 5. Objectives

1. Extract structured, accounting-ready data from handwritten, photographed, scanned, and digital GST invoices.
2. Verify every important field using two independent readers and deterministic GST and arithmetic rules.
3. Recover from failed validations by re-reading only the relevant region of the invoice.
4. Explain each decision with a clear status, a reason, and the available evidence.
5. Involve humans only where it is needed, by escalating just the fields that remain uncertain.
6. Keep invoice data private by running locally by default.
7. Measure trust, not only accuracy. When VYOM+ marks a field as Verified, we want to know how often that field is actually correct.

Every important field ends in one of three states:

| Status | Meaning |
| --- | --- |
| Verified | Passed the applicable validation and reader agreement. |
| Warning | Passed the hard checks, but contains an uncertainty or anomaly. |
| Needs Review | Could not be established reliably, even after recovery. |

This stops the system from quietly turning uncertainty into false certainty.

---

## 6. Target Users / Use Case

**Small businesses** often receive supplier invoices as photos, scans, or handwritten documents and have no easy way to check them.

**Accountants** need structured data without having to recheck every field manually.

**CA firms** process invoices from many clients and need faster, more focused review.

**Finance teams** need reliable structured invoice data before it enters downstream accounting workflows.

### Supported inputs

- Handwritten invoices
- Phone photographs
- Scanned invoices
- Digital PDFs
- JPG and PNG images
- Spreadsheet-based invoice data

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

**PaddleOCR** returns text together with bounding-box coordinates. This matters for two reasons: the coordinates let us crop the exact region of an invoice for a targeted re-read, and they let us point to the evidence for a value in the interface. It also works as a reader that is genuinely separate from the vision model.

**Qwen2.5-VL** understands the page as a whole, not only as lines of text. It copes better with layout, handwriting, and the structure of an invoice, and it can extract named fields directly from the image. It is open source and can be run on a local machine.

**Ollama** makes it simple to run the vision model locally, which supports our privacy-first approach.

**Using two readers** is a deliberate choice. Two different systems tend to fail in different ways, so disagreement between them tells us something that a single model cannot.

**Staying open source and local-first** matters because invoices contain sensitive business information. Running locally means that information does not need to be sent to an outside service.

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

```
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

```
Invoice -> Local processing (OCR, Vision Model, Validation, Evidence) -> Local result
```

Running on a cloud GPU is an optional configuration that we may use for development and demos. It is not required by the core architecture.

---

## 11. Component-Level Architecture

| Component | What it does |
| --- | --- |
| File router | Detects the input type (structured data, or PDF/image) and sends it down the right path. |
| Image preparation | Renders PDF pages and pre-processes images using OpenCV and pypdfium2. |
| OCR reader | Uses PaddleOCR to extract text and bounding boxes. |
| Vision reader | Uses Qwen2.5-VL through Ollama to extract fields, layout, and handwriting. |
| Cross-checker | Compares the two readings field by field. A disagreement is recorded as a risk signal. |
| GST validator | Checks GSTIN structure, state code, check digit, the PAN pattern, and date validity. |
| Math validator | Checks quantity times rate, line totals, CGST, SGST, and IGST consistency, and total reconciliation. |
| Recovery engine | Finds the relevant region, crops it, and runs a targeted re-read. |
| Status engine | Assigns Verified, Warning, or Needs Review to each field. |
| Evidence builder | Records the page, region, both readings, and the validation results for each field. |
| Health and risk reporter | Builds the Invoice Health and Invoice Risk Radar reports from measurable signals. |
| Output layer | Produces JSON and CSV results and the Streamlit review interface. |

---

## 12. Data / Information Flow

1. **Upload.** The user uploads an invoice as an image, a PDF, or a spreadsheet.
2. **Detection.** The file type is identified and the file is routed accordingly.
3. **Preparation.** PDFs are rendered to images, and images are pre-processed.
4. **Reading.** PaddleOCR and Qwen2.5-VL each extract the invoice on their own.
5. **Cross-check.** The two results are compared field by field.
6. **Validation.** GST and arithmetic rules are applied to the combined result.
7. **Recovery.** Fields that failed are cropped and re-read, then validated again.
8. **Decision.** Each field is marked Verified, Warning, or Needs Review.
9. **Explanation.** Evidence, health, and risk reports are generated.
10. **Output.** The result is available as JSON, CSV, and a side-by-side review screen.

```
Image / PDF -> Readers -> Cross-Check -> Rules -> Recovery -> Status -> Evidence -> JSON / CSV / UI
```

---

## 13. Agentic Workflow

VYOM+ includes a small, controlled loop that behaves in an agent-like way, but it is deliberately bounded. It is guided by the rule engine rather than left to act freely.

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

**Core features**

- Invoice upload
- File-type routing
- OCR extraction
- Vision-model extraction
- Cross-reader comparison
- GST validation
- Arithmetic validation
- Targeted field re-reading
- Field-level status
- Structured JSON output
- CSV output
- Side-by-side invoice and result review

**Additional features**

- Evidence Graph
- Invoice Health
- Invoice Risk Radar
- Benchmarking
- Invoice DNA

The status of the additional features will be marked according to what is actually implemented at the time of submission.

| Feature | Implementation status |
| --- | --- |
| Evidence Graph | To be updated at submission |
| Invoice Health | To be updated at submission |
| Invoice Risk Radar | To be updated at submission |
| Benchmarking | To be updated at submission |
| Invoice DNA | To be updated at submission |

### Example result

For a messy handwritten invoice, the output might look like this:

```
VYOM+ RESULT

Invoice No.      INV-0231        Verified
Date             14/09/2026      Verified
Seller GSTIN     27ABCDE...      Verified
Buyer GSTIN      27XYZ...        Verified

Item Amount      Rs. 10,000      Verified
CGST             Rs. 900         Verified
SGST             Rs. 900         Verified

Total            Rs. 11,800      Needs Review
  The total does not reconcile. It is off by Rs. 100.

Invoice Health:  82 / 100
Risk Level:      High
```

From here, the user can look at the original invoice, view the evidence, trigger a targeted re-read, or correct the unresolved field directly.

---

## 16. Implementation Approach

1. **Input layer.** A Streamlit upload screen with file-type detection and routing.
2. **Preparation.** PDFs are rendered with pypdfium2 and pdfplumber, and images are pre-processed with OpenCV.
3. **Dual extraction.** PaddleOCR and Qwen2.5-VL (through Ollama) read the same invoice independently.
4. **Cross-check.** Both outputs are converted to a common schema and compared field by field.
5. **Rule engine.** GST and arithmetic validators are written in plain Python.
6. **Recovery loop.** When a field fails, the PaddleOCR coordinates are used to crop the region, which is then re-read and validated again.
7. **Status and evidence.** Each field gets a status, with the page, region, both readings, and the checks attached.
8. **Reporting.** Invoice Health and Risk Radar are computed from measurable signals.
9. **Output and interface.** Results are exported as JSON and CSV, and shown side by side with the invoice in Streamlit.
10. **Benchmarking.** The system is evaluated on a manually verified test set.

### Demo flow

The demo is built around a difficult invoice rather than a clean digital PDF.

1. **Upload** a handwritten or poor-quality invoice.
2. **Read.** Two systems extract the invoice independently.
3. **Verify.** GST and arithmetic rules check the result.
4. **Detect.** A suspicious field is identified.
5. **Explain.** The interface shows exactly why the field looks suspicious.
6. **Recover.** VYOM+ re-reads only the relevant region.
7. **Decide.** If the evidence is enough, the field becomes Verified. If doubt remains, it goes to Human Review.

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

For each invoice, VYOM+ is expected to produce:

- Structured JSON containing each field's value, status, and evidence
- A CSV file with accounting-ready data
- A field-level status of Verified, Warning, or Needs Review
- Evidence for each important field, including the region, both readings, and the validation results
- An Invoice Health report, where implemented
- An Invoice Risk Radar report, where implemented
- A side-by-side review screen showing the original invoice next to the extracted result

```
Messy invoice -> AI extraction -> Verification -> Evidence -> Risk analysis
                                                                  |
                                                    +-------------+-------------+
                                                    |                           |
                                                 Trusted                    Uncertain
                                                    |                           |
                                            Accounting flow               Human review
```

---

## 18. Future Scope / Scalability

- **Supplier intelligence.** Build historical profiles of suppliers from verified invoices, extending the Invoice DNA idea.
- **Accounting integration.** Send verified structured data directly into accounting workflows.
- **Batch processing.** Handle large collections of invoices, with review limited to the fields that need it.
- **Multilingual handwriting.** Improve support for regional Indian languages.
- **Learning from corrections.** Use reviewer corrections to improve extraction for repeat suppliers.
- **Related documents.** Extend the trust layer to credit notes, purchase orders, and other financial documents.

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

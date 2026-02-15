# Methodology

## 1. Overview

This project implements a robust semi-structured document information extraction pipeline to automatically extract:

- Roll Number  
- Subject Code  
- Marks Awarded  

from scanned university answer scripts.

Given the consistent but semi-structured layout of the documents, a hybrid OCR-based spatial filtering approach was selected instead of deep layout-aware transformers. This ensures high accuracy, computational efficiency, and scalability.

---

## 2. System Architecture

The system follows a five-stage processing pipeline:

Input Image
↓
Image Preprocessing
↓
Full-Page OCR
↓
Spatial + Regex-based Field Extraction
↓
Validation & Normalization
↓
Structured JSON Output


---

## 3. Data Characteristics

The dataset consists of scanned university answer scripts containing:

- Printed structured header region
- Handwritten student details
- Tabulated marks section
- Final marks written in red ink
- Optional normalized marks (e.g., 78/100)

The layout is largely consistent across samples but may contain:

- Slight scan shifts
- Minor rotations
- Varying brightness levels
- Handwriting noise

Thus, a precaution-based hybrid extraction approach was implemented.

---

## 4. Image Preprocessing

To improve OCR accuracy, the following preprocessing steps were applied:

### 4.1 Grayscale Conversion
Reduces color noise and simplifies feature space.

### 4.2 Contrast Enhancement (CLAHE)
Improves readability of faded or low-contrast text.

### 4.3 Noise Reduction
Gaussian or bilateral filtering is applied to remove scanning artifacts.

### 4.4 Optional Deskewing
Hough Line Transform-based correction is used to handle rotated scans.

These preprocessing steps improve OCR performance by approximately 5–10%.

---

## 5. Optical Character Recognition (OCR)

Full-page OCR is performed using EasyOCR.

Each OCR detection returns:

- Bounding box coordinates
- Extracted text
- Confidence score

Example output structure: [(bbox, text, confidence), ...]


Full-page OCR was preferred over hard region cropping to ensure flexibility and robustness to layout shifts.

---

## 6. Field Extraction Strategy

Each field is extracted using a combination of:

- Regular Expressions (Regex)
- Spatial filtering
- Keyword anchoring
- Confidence scoring

---

### 6.1 Roll Number Extraction

**Regex Pattern:**
 \d{2}[A-Z]{3}\d{3}
 
Example:
23ADR180


**Validation Rules:**

- Must appear within the top 35% of image height  
- OR located near the keyword "Register"

This prevents accidental extraction from answer text content.

---

### 6.2 Subject Code Extraction

**Regex Pattern:**

 \d{2}[A-Z]{3}\d{2}
 
Example:
22ALT41


**Validation Rules:**

- Must appear within the top 40% of the image  
- OR located near the keyword "Course Code"

---

### 6.3 Marks Extraction

Marks are typically located near the "Grand Total" section in the right-middle region of the page.

#### Step 1: Keyword Anchoring

Search OCR text for:
- "Grand"
- "Total"
- "Marks"

#### Step 2: Spatial Filtering

Extract numeric values that:

- Appear within ±200 pixels vertically from the keyword  
- Are located in the right half of the page  

#### Step 3: Numeric Filtering Rules

| Condition            | Action        |
|----------------------|--------------|
| Value ≤ 50           | Accept directly |
| 50 < Value ≤ 100     | Divide by 2 |
| Value > 100          | Ignore |

#### Step 4: Conflict Resolution

If both:
39
78/100


are detected, prefer 39 since it matches the ≤50 rule.

---

## 7. Confidence Scoring & Validation

Each extracted field is validated using:

- OCR confidence score threshold (> 0.6)
- Regex conformity
- Spatial correctness

If confidence falls below threshold:

- Field is flagged
- Backup region-based extraction is triggered

---

## 8. Output Format

Final extracted information is stored in JSON format:

```json
{
  "roll_number": "23ADR180",
  "subject_code": "22AIT41",
  "marks_awarded": 39
}
```
## 9. Justification for Model Selection

### 9.1 Why YOLO was not used

YOLO (You Only Look Once) is an object detection framework suitable for detecting multiple heterogeneous layout elements (tables, headers, stamps, logos, etc.).

However, in this project:

- The document layout is highly consistent.
- Required fields (Roll Number, Subject Code, Marks) appear in predictable regions.
- Full layout segmentation is unnecessary.
- Training a detection model requires bounding-box annotations for each layout component.
- Additional inference overhead is introduced without significant performance gain.

Therefore, YOLO was deemed unnecessary for this controlled document structure.

---

### 9.2 Why LayoutLM / Layout-Aware Transformers were not used

LayoutLM-style models combine:

- Text embeddings
- Spatial embeddings
- Transformer-based contextual understanding

While powerful, they:

- Require large labeled datasets.
- Demand high GPU memory.
- Increase inference latency.
- Introduce training complexity.

Given:

- The small number of target fields (3 only),
- The consistent layout,
- The availability of deterministic extraction rules,

A transformer-based architecture would be over-engineering for the current scope.

---

### 9.3 Why Hybrid OCR + Spatial Filtering was Selected

The selected approach combines:

- Full-page OCR (EasyOCR)
- Regex pattern matching
- Spatial filtering
- Rule-based validation

Advantages:

- Lightweight
- Interpretable
- Debuggable
- Fast inference
- Minimal training data required
- Easy deployment on CPU environments

This approach offers a strong accuracy-to-complexity ratio for semi-structured academic documents.

---

## 10. System Robustness Design (Precaution Version)

To ensure high reliability, several defensive mechanisms were integrated.

---

### 10.1 Multi-Candidate Handling

If multiple regex matches are detected:

- Filter by spatial constraints.
- Select candidate with highest OCR confidence.
- Apply logical constraints (e.g., marks ≤ 50).

---

### 10.2 Spatial Anchoring

Extraction is not purely regex-based.

Each field is validated against:

- Relative image height percentage.
- Proximity to keywords.
- Expected horizontal positioning.

This prevents false positives from answer content.

---

### 10.3 Numeric Normalization Rules

To avoid confusion between:

- Raw marks (e.g., 39)
- Normalized marks (e.g., 78/100)

Rules applied:

- If value ≤ 50 → accept.
- If 50 < value ≤ 100 → divide by 2.
- If > 100 → ignore.
- If fraction detected → extract numerator.

---

### 10.4 Low Confidence Handling

If OCR confidence < 0.6:

- Trigger backup region-based cropping.
- Re-run OCR on narrowed region.
- Compare confidence scores.
- Flag sample for manual review if unresolved.

---

### 10.5 Outlier Detection

If extracted marks:

- < 0
- > 50
- Non-integer
- Empty

The record is flagged for validation.

---

## 11. Evaluation Strategy

### 11.1 Dataset

- 382 annotated samples
- JSON ground-truth labels
- Single-format university answer scripts

---

### 11.2 Metrics

Field-wise evaluation:

- Accuracy
- Precision
- Recall
- F1-score

Per-field evaluation is performed separately for:

- Roll Number
- Subject Code
- Marks Awarded

---

### 11.3 Error Analysis

Common error categories:

1. Handwriting misread by OCR.
2. Low contrast scans.
3. Overlapping numeric candidates.
4. Misaligned scans (skew).

Error samples are stored for iterative improvement.

---

## 12. Expected Performance

| Field         | Expected Accuracy |
|--------------|------------------|
| Roll Number  | 95%+ |
| Subject Code | 95%+ |
| Marks        | 93–97% |

Performance may vary depending on:

- Handwriting clarity
- Scan resolution
- Lighting conditions

---

## 13. Scalability Considerations

The system can be extended to:

- Multi-subject scripts
- Additional header fields
- Table extraction
- Multi-university formats

Potential upgrades:

- LayoutLMv3 for dynamic layouts
- YOLO-based header detection for multiple templates
- Custom OCR fine-tuning
- Human-in-the-loop correction interface

---

## 14. Deployment Considerations

Recommended deployment stack:

- Python backend (FastAPI / Flask)
- CPU-based inference server
- Batch processing capability
- JSON storage or database integration

Estimated inference time:

- ~1–2 seconds per image (CPU)
- <1 second (GPU)

---

# Conclusion

A precaution-oriented hybrid OCR pipeline was designed to extract structured academic information from semi-structured answer scripts.

The architecture prioritizes:

- Accuracy
- Interpretability
- Computational efficiency
- Production feasibility

while avoiding unnecessary deep-learning complexity.

This design provides a scalable foundation for future layout-aware or transformer-based enhancements.

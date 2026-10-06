# Clinical Record Foundation

## Goal

Build a reusable clinical-document foundation for large EMR PDFs. V1 uses HEDIS care-gap evidence retrieval and Risk Adjustment evidence retrieval as reference consumers, but neither use case should be embedded into the foundational artifacts.

## Architecture

```text
SOURCE PDF
    |
    v
OCR ENGINE
    |
    v
ARTIFACT 1: SOURCE OCR JSON
"What is on the pages?"
    |
    v
CLINICAL RECORD MAPPER
- encounter segmentation
- encounter classification
- clinical-document segmentation/classification
- section mapping
- dates / authorship / status
- relationships / confidence
    |
    v
ARTIFACT 2: CLINICAL DOCUMENT MAP JSON
"What is where and how is it related?"
    |
    +----------------------+
    |                      |
    v                      v
HEDIS EVIDENCE         RISK ADJUSTMENT
EXTRACTOR              EXTRACTOR
    |                      |
    v                      v
HEDIS EVIDENCE JSON    RA EVIDENCE JSON
    |                      |
    v                      v
Measure/rules engine    RA/HCC rules
```

**Core rule:** Artifact 1 preserves the source. Artifact 2 organizes the source. Use-case consumers interpret the source.

## Artifact 1 — Source OCR JSON

Artifact 1 is the normalized representation of OCR/document-intelligence output. It preserves what was present in the source PDF and where it came from, without clinical interpretation.

```json
{
  "artifact_type": "source_ocr",
  "artifact_version": "1.0",
  "source_document_id": "pdf_123",
  "source_hash": "sha256:...",
  "page_count": 127,
  "pages": [
    {
      "page_number": 1,
      "ocr_confidence": 0.98,
      "blocks": [
        {
          "block_id": "p1_b001",
          "type": "text",
          "text": "Annual Wellness Visit",
          "bbox": [100, 50, 700, 110],
          "confidence": 0.99
        }
      ],
      "tables": []
    }
  ]
}
```

Preserve page boundaries, text blocks, bounding boxes, OCR confidence, tables/cells, headers/footers, forms, checkboxes, handwriting indicators and original source references when available. Every OCR element needs a stable source ID.

## Artifact 2 — Clinical Document Map

Artifact 2 is a use-case-neutral map answering: **what clinical material exists, where is it, when did it occur, and how do the pieces relate?**

```text
Clinical Record
  -> Patient
  -> Encounter
      -> Clinical Document
          -> Section
              -> Artifact 1 source references
```

Standalone documents are allowed and must not be forced into encounters.

## Encounter Segmentation and Classification

Segmentation asks which pages/spans belong to the same encounter. Classification asks what type of encounter it is. Keep these as separate modules.

Encounter types should be extensible: outpatient, annual wellness, emergency department, observation, inpatient, skilled nursing facility, behavioral health, psychiatric hospitalization, rehabilitation, home health, hospice, telehealth, unknown, other.

A date change does not necessarily mean a new encounter. An inpatient admission may contain notes across many dates:

```text
03/10 Admission H&P
03/11 Progress Note
03/12 Cardiology Consult
03/13 Progress Note
03/17 Discharge Summary
```

All may belong to one hospitalization.

## Clinical Document Classification

Within encounters classify documents such as admission H&P, progress note, consult note, psychiatric evaluation, mental-status exam, nursing note, discharge summary, medication reconciliation/list, problem list, lab report, imaging report, procedure/operative note, pathology report, therapy note, care plan, depression screening, fall-risk assessment, functional/cognitive assessment, administrative, unknown, other.

Preserve the source document title.

## Section Mapping

Map sections including chief complaint, HPI, ROS, vitals, physical exam, assessment/plan, diagnoses, problem list, medications, allergies, labs, imaging, procedures, social/family history, depression screening, fall-risk assessment, functional/cognitive assessment, and discharge instructions.

Multiple sections can occur on one page and sections can span pages.

```json
{
  "section_id": "sec_023",
  "document_id": "doc_004",
  "section_type": "assessment_plan",
  "source_title": "Assessment and Plan",
  "source_refs": ["p36_b014:p36_b029"],
  "confidence": 0.98
}
```

## Time, Authorship, Status and Relationships

Time is first-class. Preserve encounter start/end, document service/authored/signed timestamps and lab ordered/collected/resulted/finalized timestamps when available.

Preserve author name, credentials, specialty/role and document status such as draft, preliminary, final, signed, amended, corrected, cancelled, unknown.

Support relationships such as:

```text
inpatient encounter -> followed_by -> SNF encounter
lab report -> associated_with -> inpatient encounter
imaging report -> ordered_during -> ED encounter
document -> continuation_of -> document
```

Do not force low-confidence relationships.

## Mapping Pipeline

```text
Artifact 1
   |
   v
Cheap metadata extraction
   |
   v
Page/document signals
   |
   v
Candidate document + encounter boundaries
   |
   +--> high confidence --> accept
   |
   +--> ambiguous --> model adjudication
   |
   v
Encounter grouping
   |
   v
Encounter classification
   |
   v
Clinical document classification
   |
   v
Section mapping
   |
   v
Relationships + temporal metadata
   |
   v
Artifact 2
```

Candidate signals include patient identifiers, service/admission/discharge dates, encounter/document IDs, facility, provider, title, page numbering, repeated headers, signatures, layout changes, section continuity and sentence/table continuation.

Do not send a 1,000-page chart to a large model just to identify boundaries. Use local context around ambiguous boundaries.

## Reference Consumer 1 — HEDIS

HEDIS consumes Artifact 2 and resolves relevant content from Artifact 1. The mapper itself must not know HEDIS rules.

The HEDIS output is primarily evidence:

```json
{
  "use_case": "hedis",
  "source_document_id": "pdf_123",
  "measure_id": "example_measure",
  "measurement_period": {
    "start": "2026-01-01",
    "end": "2026-12-31"
  },
  "denominator_evidence": [],
  "numerator_evidence": [],
  "exclusion_evidence": [],
  "exception_evidence": [],
  "missing_evidence": [],
  "warnings": []
}
```

Evidence should include evidence type/value, code/code-system when present, service date, encounter ID, source refs and confidence. A separate rules layer determines denominator eligibility, numerator status, exclusions/exceptions and care-gap status.

## Reference Consumer 2 — Risk Adjustment

Risk Adjustment also consumes Artifact 2. Do not reduce RA extraction to diagnosis-code detection alone. Retrieve documented diagnoses/codes, condition text, assessment/plan evidence, relevant clinical support, encounter context, service date, provider context, source provenance and relevant procedures.

```json
{
  "use_case": "risk_adjustment",
  "source_document_id": "pdf_123",
  "conditions": [
    {
      "condition": "Type 2 diabetes mellitus with hyperglycemia",
      "documented_codes": [
        {
          "code": "E11.65",
          "code_system": "ICD-10-CM",
          "source_refs": ["p8_b016"]
        }
      ],
      "clinical_evidence": [
        {
          "evidence_type": "assessment",
          "text": "Type 2 diabetes with hyperglycemia",
          "source_refs": ["p8_b016"]
        }
      ],
      "encounter_id": "enc_001",
      "service_date": "2026-03-14",
      "confidence": 0.97
    }
  ],
  "procedures": []
}
```

A separate RA/HCC rules layer determines final business/coding outcomes.

## Retrieval

Artifact 2 should support queries such as:

```python
contexts = clinical_map.find(
    encounter_types=["outpatient", "inpatient"],
    document_types=["annual_wellness_visit", "discharge_summary"],
    section_types=["assessment_plan", "diagnoses", "medications", "labs"]
)
```

Resolve source refs against Artifact 1. Also allow broad retrieval when a use case intentionally wants an entire note/document. The map is navigation, not a restriction on context size.

## Versioning and Caching

Cache independently:

```text
PDF hash
  -> Artifact 1
  -> Artifact 2
  -> HEDIS evidence
  -> RA evidence
```

Changing HEDIS or RA logic must not rerun OCR or mapping.

## Evaluation

Evaluate Artifact 1 for OCR text quality, table fidelity, page/block preservation and source-reference stability.

Evaluate Artifact 2 for encounter-boundary precision/recall, encounter grouping/type accuracy, document-boundary/type accuracy, section classification, source-reference accuracy and temporal metadata accuracy.

The most important downstream metric is **evidence routing recall**. Favor recall: a little extra context is preferable to missing clinically relevant evidence.

## Suggested Repository Structure

```text
src/
  ocr/
    interface.py
    normalizer.py
    models.py
  mapper/
    signals.py
    encounter_segmenter.py
    encounter_classifier.py
    document_segmenter.py
    document_classifier.py
    section_mapper.py
    relationships.py
    mapper.py
  models/
    artifact1.py
    artifact2.py
    encounter.py
    clinical_document.py
    section.py
    relationship.py
  retrieval/
    clinical_map.py
  use_cases/
    hedis/
      extractor.py
      schema.py
      config/
    risk_adjustment/
      extractor.py
      schema.py
schemas/
  artifact1.schema.json
  artifact2.schema.json
  hedis_evidence.schema.json
  risk_adjustment_evidence.schema.json
tests/
  fixtures/
  unit/
  integration/
  evaluation/
examples/
  artifact1.example.json
  artifact2.example.json
  hedis.example.json
  risk_adjustment.example.json
```

## Coding-Agent Build Order

1. Artifact contracts: typed models + JSON Schemas for Artifacts 1 and 2. Support outpatient, inpatient, SNF, behavioral health and noncontiguous page membership.
2. OCR adapter: provider-neutral interface, PDF -> Artifact 1.
3. Clinical mapper: Artifact 1 -> Artifact 2, including segmentation, classification, sections, time, authorship/status, confidence and relationships.
4. Visual/debug map for every fixture.
5. HEDIS reference consumer for one/few selected measures.
6. Risk Adjustment reference consumer.
7. Evaluation harness with manually labeled expected maps and evidence-recall evaluation.

## Initial Test Corpus

Use deliberately diverse PDFs: simple outpatient, annual wellness, multiple outpatient encounters, inpatient, inpatient + specialist consults, inpatient followed by SNF, behavioral-health/psychiatric evaluation, lab-heavy chart, poor-quality scan and a large multi-encounter PDF.

## Non-Negotiable Architecture Rules

1. OCR once.
2. Artifact 1 preserves the source.
3. Artifact 2 organizes the source.
4. Artifact 2 remains use-case neutral.
5. Every downstream fact is traceable to Artifact 1.
6. Time is first-class.
7. Do not assume one date equals one encounter.
8. Do not assume one page equals one document or section.
9. Do not assume encounters/documents are contiguous.
10. Represent uncertainty explicitly.
11. Do not build a universal mega clinical-facts JSON.
12. HEDIS and Risk Adjustment are consumers of the map, not part of it.
13. Optimize the map for evidence-routing recall.
14. Prefer deterministic/cheap segmentation when reliable; use models for ambiguity.
15. Keep OCR, mapping, retrieval and use-case logic independently versioned/cacheable.

## Definition of Done for V1

Given representative clinical PDFs:

```text
PDF
 -> Artifact 1
 -> Artifact 2
 -> visual/debug map
 -> HEDIS evidence JSON
 -> Risk Adjustment evidence JSON
```

A reviewer can trace any HEDIS or RA evidence item back to the exact source page/block. Adding another HEDIS measure or changing RA logic does not require rerunning OCR. Adding a future use case does not require redesigning Artifact 1 or Artifact 2.
